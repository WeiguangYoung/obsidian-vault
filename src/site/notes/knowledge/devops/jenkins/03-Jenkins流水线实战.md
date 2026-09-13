---
{"dg-publish":true,"permalink":"/knowledge/devops/jenkins/03-Jenkins流水线实战/","tags":["Jenkins","Pipeline","Jenkinsfile","CI/CD","排障","实战记录"],"dg-note-properties":{"date":"2026-09-05","tags":["Jenkins","Pipeline","Jenkinsfile","CI/CD","排障","实战记录"]}}
---


# Jenkins 流水线实战（问题排查与解决 · hello-jenkins）

> Jenkins 任务：`hello-jenkins`（仓库 codeup.aliyun.com/WeiguangYoung/hello-jenkins，dev 分支）
> 环境：Jenkins Master（Docker 8080/50000）+ 三类 agent —— ①docker/linux ②`win-agent-01`（label `windows`）③k3s 动态 pod（label `k8s-linux`，双节点跨 VPC：server 47.99 杭州 / agent 101.133 上海）
> 每条问题按 **现象 → 排查过程 → 根因 → 解决** 记录，按类别归档，持续补充。

---

## 一、DNS / 网络类

### 1.1 k8s pod clone 报 `Could not resolve host`（上游 DNS 递归超时）

**现象**：动态 agent 构建第一步 checkout 就挂，pod 内 git 解析 codeup.aliyun.com 稳定超时；宿主机自己解析正常。

**排查过程**：
1. 不重启构建，往 jenkins ns 放**诊断 pod** 复现/对照（关键手法）。
2. 对照发现三条链路各自表现：agent 宿主机解析 OK（走 VPC 内网 DNS）、server 宿主机解析 OK、但 **pod 走集群 CoreDNS(10.43.0.10) 解析公网稳定超时**（curl 连续 10 次失败）→ 瓶颈锁定在 CoreDNS 转发链。
3. 查 CoreDNS Corefile：`forward . /etc/resolv.conf` = 转发到 server 的 systemd-resolved，即 agent pod → server CoreDNS → systemd-resolved → 公网，**跨 VPC 递归链不稳**。
4. 拉日志障碍：跨 VPC 时从 server 拉 agent 上 pod 日志全 502（6443 代理到 agent kubelet 不通）→ 让诊断 pod 把结果写 `/dev/termination-log`，从 API 读 pod status 绕开。

**根因**：三层 —— pod 落在跨 VPC 的 agent 节点 + CoreDNS 递归链不稳 + 误以为"宿主 OK 则 pod OK"（两条解析路径必须分开查）。

**解决**（server 上 cluster-admin 操作）：
1. 备份 CoreDNS cm 后，上游改 `forward . 223.5.5.5 223.6.6.6`（阿里同厂商 DNS 秒回；实测腾讯 119.29.29.29 跨厂商超时，别选）。
2. rollout restart 卡死（新 coredns pod 漂到 agent、就绪探针 503）→ 给 coredns deployment 加 **nodeSelector 固定回 server**。
3. 重跑构建，clone 成功。

**教训**：busybox 解析偶发成功 vs glibc 稳定失败——**别信单次结果**；系统级 Deployment（CoreDNS）必须 nodeSelector 固定，别让它漂。

### 1.2 解析失败复发：pod 根本到不了 CoreDNS（`dnsPolicy: Default` 根治）

**现象**：上游 DNS 修好后，落到 agent 节点的构建 pod 又报 `Could not resolve host`——症状一样，断点变了。

**排查过程**：
1. 查 CoreDNS 现状：pod Running、ConfigMap 里 `forward . 223.5.5.5` 还在 → 上游没坏。
2. 在 agent 宿主机验证：`getent hosts codeup.aliyun.com` 秒回、https 可达 302 → **宿主机 VPC 内网 DNS 完全正常**。
3. 从 agent 宿主机探测集群 DNS：`/dev/tcp/10.43.0.10/53` **不可达** → 断点在"agent 节点 pod → 跨 VPC overlay → server 上的 CoreDNS"这条 UDP/53 链路。
4. 定性：这正是 §1.1 里把 CoreDNS nodeSelector 固定回 server 的**副作用**——CoreDNS 只活在一个节点，另一个节点的 pod 访问它要穿跨 VPC 的 flannel overlay，而这条 overlay 链路本身不通/不稳。

**根因**：pod 默认 `dnsPolicy: ClusterFirst` 强走集群 CoreDNS；跨 VPC 双节点集群里，单点 CoreDNS + 不稳的 overlay = 随机解析失败。

**解决**：给 Jenkins k8s pod template 的 Raw YAML（Overrides 合并策略）注入：
```yaml
spec:
  dnsPolicy: Default   # pod 直接用所在节点 /etc/resolv.conf（VPC 内网 DNS），不进集群 CoreDNS
```
改 config.xml 后重启 Jenkins 生效。验证：构建 pod 内 clone 成功、解析失败归零。
**前提**：构建 pod 不需要解析集群内 service 名（本环境 Jenkins 走公网 IP，成立）；若以后需要集群内 DNS，考虑 node-local-dns 或每节点跑 CoreDNS。
**后续（2026-09-05）**：元凶——跨 VPC overlay 不通——已由**沪杭 VPC 对等连接**根治（实测 agent 节点 pod 走 10.43.0.10 解析公网 5/5 OK），集群 DNS 恢复可用；`dnsPolicy: Default` 变通保留不碍事（走节点 VPC DNS 解公网更快），等哪天需要集群内 service 名时再摘。配置过程见 k3s部署实战.md「VPC 对等连接打通实录」。

### 1.3 Windows 节点日志中文乱码

**现象**：bat 输出的中文/日期在 Jenkins 日志里变 `ϵͳҲ`、`锟斤拷`。
**根因**：cmd 输出 GBK，Jenkins 按 UTF-8 渲染。
**解决**：bat 开头 `chcp 65001`，或日志内容用英文 ASCII。

---

## 二、Jenkinsfile 语法类（Groovy 陷阱）

### 2.1 Windows 反斜杠未转义 → Groovy 编译失败

**现象**：流水线还没跑就 `unexpected char: '\'` FAILURE。
**根因**：Jenkinsfile 是 Groovy，双引号串里 `\` 是转义符，Windows 域账户写法 `%USERDOMAIN%\%USERNAME` 直接踩雷。
**解决**：
```groovy
// ❌ bat "echo %USERDOMAIN%\%USERNAME"
bat "echo %USERDOMAIN%\\%USERNAME"   // ✅ 双引号用 \\，或去掉反斜杠
```

### 2.2 单引号变量不展开 → 归档"静默失败"（SUCCESS 假象）

**现象**：构建 SUCCESS，但 Artifacts 列表里没有 cppcheck 报告；日志里有 `doesn't match anything ... Configuration error?` 却不判失败。
**排查过程**：扫描成功日志时发现 archiveArtifacts 模式 `${ARTIFACT_DIR}/...` 匹配告警 → 对比产物目录实际文件存在 → 确认是**路径没展开**而非产物缺失；同时注意到 MISRA 报告能发布，是因为 publishCppcheck 用了 `**/misra-report.xml` 递归 glob 兜底。
**根因**：`archiveArtifacts 'ARTIFACT_DIR}/...'` 传的是**单引号 Groovy 字符串**（不展开变量）；且 archiveArtifacts 匹配不到时**只告警不失败**。
**解决**：
- 改双引号 `"$ARTIFACT_DIR/..."` 或直接写死字面路径
- 插件类报告步骤用 `**/xxx.xml` 递归 glob 兜底
- 教训：**SUCCESS ≠ 全都干了，验收必须核对 Artifacts 实际列表**
- **门禁落地（2026-09-07，commit 83d3871）**：k8s-linux 与 windows 两处产物 `archiveArtifacts` 均改 `allowEmptyArchive: false`，Windows stage 另加 `if not exist build\UDS_S32K144_Bootloader.elf (exit /b 1)` 前置检查——空归档从"只告警"升级为"直接失败"，彻底封死本节这种 SUCCESS 假象。#120 实测两平台 Artifacts 齐、门禁全过。

### 2.3 插件 DSL 步骤名写错

**现象**：`NoSuchMethodError: No such DSL method 'cppCheck'`。
**根因**：静态检查插件提供的步骤是 `publishCppcheck`，不存在 `cppCheck`。
**解决**：报错信息本身就列出了全部可用 steps/symbols 清单，按图索骥改名字。

### 2.4 declarative `agent { docker { ... } }` 块里的两个语法坑（#130）

**现象**：#130 秒失败，一行真代码没跑：`WorkflowScript: 56: expecting '}', found ','`。
**根因**：Linux 腿从 `dockerfile{}` 改成 `docker{}` 时按 Groovy 方法参数习惯写了逗号——但 declarative 的 agent 描述块是**符号 DSL 上下文不是表达式上下文**，多个属性（image/label/registryUrl/registryCredentialsId）必须用**分号**分隔。
**解决**（@15de62a）：
```groovy
// ❌ agent { docker { image "...", label '...', registryUrl '...' } }
agent { docker { image "${BUILD_IMAGE_REPO}:${BUILD_IMAGE_TAG}"; label 'k8s-linux'; registryUrl 'https://registry.cn-hangzhou.aliyuncs.com'; registryCredentialsId 'acr-cred' } }   // ✅ 分号
```
**连带注意**：`image` 要插值 env 变量必须用**双引号**（单引号不展开，同 §2.2 的坑换个马甲）；改完 #131 一次跑绿。

---

## 三、构建环境 / Agent 能力类

### 3.1 Docker agent 容器里没有 git → exit 127

**现象**：`script.sh: 1: git: not found`（127 = 命令不存在）。
**排查过程**：exit 127 直接指向命令缺失 → 进构建镜像确认无 git。
**根因**：自己写的构建镜像只装了交叉工具链，没装 git/coreutils。
**解决**：Dockerfile apt 补装 git + coreutils。
**教训**：**别假设 agent 环境自带什么**，流水线用到的每个命令都要在镜像里有。

### 3.2 k8s pod 里 `dockerfile` agent 报 `docker: not found`（完整解决线）

**现象**：`agent { dockerfile { label 'k8s-linux' } }` 在构建 pod 内执行 `docker build` 报 127。

**排查过程**：
1. 定性：jnlp 容器（inbound-agent 镜像）里没有 docker CLI——但这不是主要矛盾。
2. 深挖：k3s **只有 containerd，没有 docker daemon**，节点上根本没有 `/var/run/docker.sock` → 判断"装个 CLI 也没用，挂 socket 更没得挂"，此为**死路结论**。
3. 方案比选：① kaniko（无 daemon 构建，改造成本低但要换 stage 写法）② 预构建镜像 ③ 构建回 master docker agent ④ **给 k3s 节点直接装 docker**。
4. 最终选 ④（节点级一次性解决）：agent 节点是 Alibaba Cloud Linux 3（RHEL 系，**没有 apt**），用 `dnf` + 阿里云 docker-ce 源装 Docker CE 26.1.3，`systemctl enable --now docker`。
5. 装完直连 Docker Hub 超时 → 参照 k3s registries.yaml 同款方案给 `/etc/docker/daemon.json` 配国内镜像加速（daocloud + 1ms.run），hello-world 验证通过；server 节点 docker 同步补齐 daemon.json。
6. 改 Jenkins pod template Raw YAML 注入挂载（Overrides）：
```yaml
spec:
  containers:
    - name: jnlp
      volumeMounts:
        - { mountPath: /usr/bin/docker,     name: docker-cli, readOnly: true }
        - { mountPath: /var/run/docker.sock, name: docker-sock }
  volumes:
    - { name: docker-cli,  hostPath: { path: /usr/bin/docker } }
    - { name: docker-sock, hostPath: { path: /var/run/docker.sock } }
```
7. 重启 Jenkins 后首次验证构建**仍然报 docker: not found**——二次排查发现是**旧模板创建的 agent pod 还在被复用**（pod spec 里只有 workspace 卷、没有 docker 挂载）；等旧 pod 回收后新 pod 从新模板创建，挂载生效。
8. 再次构建：`docker build` 三步全过、镜像 build 成功——但**表面 SUCCESS 实为僵尸构建**（后续 stage 永远卡死），见 §3.4

**根因（分两层）**：
- 表层：jnlp 容器无 docker CLI + k3s 无 docker daemon（原死路判断成立）
- 隐蔽层：**Jenkins 重启 + 模板更新后，已 provision 的旧 agent pod 会带着旧 spec 继续服务**，验证时容易误判"改了没用"

**解决**：节点装 docker + 镜像加速 + pod template hostPath 挂 socket/CLI + 等旧 pod 回收再验证。
**注意**：`chmod 666 docker.sock` 是为让容器内 uid 直连的权宜（socket 即 root 权限，内网环境接受，严格做法是容器 runAsUser 对齐 docker 组 GID）；hostPath 挂 CLI 依赖节点与容器 glibc 兼容（本次 el8 CLI 在 inbound-agent 容器可用）。

### 3.3 Windows 节点不能用 sh / agent label 不匹配

**现象**：Windows 上 `sh` 步骤直接报错；`agent any` 的 job 调度不到 k8s 模板。
**根因**：Windows 只认 `bat`/`powershell`；带 label 的（k8s）模板要求显式匹配 label。
**解决**：`agent { label 'windows' }`；多约束用 `label 'windows && vs2022'`；Linux/Windows 双线的 stage 要么拆两个 job，要么按节点分支 `isUnix()` / `isWindows()` 双实现。

---

### 3.4 dockerfile agent 僵尸构建：emptyDir workspace × docker.sock 挂载错位

**现象**：挂载修好后构建"看似成功"：build.xml 有 SUCCESS 字样但 `duration=0`、日志无 `Finished:` 行、页面永远显示进行中；且 Agent Setup（docker build）之后的第一个 `sh` 就报 `process apparently never started in .../@tmp/durable-xxx`，agent pod 陷入"seems to be removed or offline → is back online"无限循环。

**排查过程**：
1. 判定 SUCCESS 是脏数据：无 Finished 行、无 workflow-completed → 从未正常结束，是真僵尸。
2. 节点 `docker ps` 发现悬挂的构建容器（镜像就是刚 build 出来的 f17a06...）还 Up 着。
3. `docker inspect` 它的挂载：bind 源是宿主路径 `/home/jenkins/agent/workspace/hello-jenkins{,@tmp}`——去宿主一看，**是 docker 自动创建的空目录（0 个文件）**。
4. 对照链路：jnlp pod 里的 workspace 是 **emptyDir**（只存在于 pod 内）；宿主 daemon 收到 Jenkins 发的 `docker run -v /home/jenkins/agent/workspace/...` 时按**宿主文件系统**解析同名路径 → 构建容器挂到空目录 → Jenkins 写进 `@tmp` 的 durable-task 脚本容器里根本不存在 → "process never started" → agent 断连循环。

**根因**：外置 daemon 模式（挂 docker.sock）下，所有 `-v` 源路径由宿主解析；而 k8s pod 的 workspace 默认 emptyDir，宿主与容器"同名不同物"。这是 docker-in-docker via socket 的经典挂载错位。

**解决（定稿）**：❌ **Raw YAML 覆盖不了 workspace 卷**——kubernetes 插件是在 YAML 合并**之后**才补挂默认 emptyDir `workspace-volume`，Overrides 里写同名 hostPath 卷会被插件的默认卷顶掉（此前两轮修复就折在这一步，当时误以为方案有效）。正确字段是 PodTemplate 的 **`workspaceVolume`**，UI 上有 "Workspace Volume" 下拉可选（dropdownDescriptorSelector），config.xml 直改写法：
```xml
<workspaceVolume class="org.csanchez.jenkins.plugins.kubernetes.volumes.workspace.HostPathWorkspaceVolume">
  <hostPath>/home/jenkins/agent</hostPath>
</workspaceVolume>
```

⚠️ **踩坑两连（今天真实踩过）**：
1. **XStream 多态字段必须用 `class` 属性式**。第一版按"嵌套子元素"写（`<workspaceVolume><org.csanchez...HostPathWorkspaceVolume>...</...>`），Jenkins 重启加载时**静默丢弃整个字段**——启动无 SEVERE、config.xml 里该字段消失、pod 依旧 emptyDir（其后两轮验证构建因此继续僵尸）。同文件里 `podRetention class="..."`、`yamlMergeStrategy class="..."` 就是标准范式可对照。**改完必须重启后 grep 确认字段存活**，再跑验证构建。
2. **类名/字段别凭记忆猜**。定稿前从插件 jar 实查：`unzip -l kubernetes.jar | grep -i WorkspaceVolume` 确认实现类全名 `org.csanchez.jenkins.plugins.kubernetes.volumes.workspace.HostPathWorkspaceVolume`（唯一字段 `hostPath`），`javap`/config.jelly 交叉验证。

前置：两节点预建 `mkdir -p /home/jenkins/agent`，且**属主必须是 1000:1000**（jnlp 容器跑 uid 1000；此前僵尸期 docker 在宿主自动建的 `workspace/hello-jenkins{,@tmp}` 是 root 属主，要 `chown -R 1000:1000` 修，光 chmod 777 不够稳）。这样 jnlp pod 写的文件和构建容器 `-v` 的源是同一份宿主真实路径，durable 脚本可见。

**验证（定稿判据）**：`Finished: SUCCESS` + duration 非 0（3m51s）+ 日志零次 `process apparently never started` + 宿主 `agent节点:/home/jenkins/agent/workspace/hello-jenkins/` 真实落盘（源码/build/artifacts，属主 1000）+ Artifacts 有产物 + pod 结束即回收。以上全齐才算真绿。

**僵尸清理手法**：这种 agent 断连循环的卡死构建，Jenkins 页面上 Abort 往往也没反应（agent 已失联），**直接 `kubectl delete pod` 掉对应 agent pod** 即可强制结束（状态转 ABORTED）。

**副作用/注意**：
- 同节点所有构建 pod 共享同一宿主目录——同一 job 并发构建会互踩（需要时加 `disableConcurrentBuilds()` 或 `ws()` 隔离子目录）。
- hostPath 不随 pod 销毁，工作区文件残留，定期 `cleanWs()` 或手工清理。

### 3.5 dockerfile agent 冷构建：层缓存节点本地 × 跨节点调度 = 超时 ABORTED（#117/#118）

**现象**：#117/#118（2026-09-06 20:28 / 22:10）连挂，日志无执行报错，结尾 `Cancelling nested steps due to timeout` + `Timeout has been exceeded`；掐断时 apt 正下到 `Get:219 libstdc++-arm-none-eabi-newlib [439 MB]`，且日志**零 `Using cache`**（整个 Dockerfile 层从头构建）。

**时间线**：

| 构建 | 结果 | 用时 | 层缓存 | 说明 |
|---|---|---|---|---|
| #115/#116 | SUCCESS | ~2.2min | 命中 | apt 步骤 `Using cache` |
| #117/#118 | ABORTED | ~30min | **未命中** | 冷 apt 下 681MB，30min timeout 掐死（仅下载就要 ~24min @473kB/s） |
| #119 | SUCCESS | 43min | 未命中 | timeout 提到 90min（commit 2346778）后冷缓存完整跑通 |
| #120 | SUCCESS | 8.4min | 命中 | 新产物门禁 + Windows Size Check 验证通过 |

**根因**：`dockerfile` agent（挂 docker.sock，见 §3.2/§3.4）走的是**节点本地 dockerd**，`debian:latest` 基础镜像和各 RUN 层缓存都落在节点上，**不跟着 pod 走**。k3s 双节点集群里构建 pod 换节点 = 缓存全丢 → apt 重下 681MB 工具链，30min 超时必爆。"同一份代码，2.2min 和 43min 差一个数量级"就是这个症状。90min timeout 是止血不是根治。

**根治方向（待办，见文档末尾）**：
1. 工具链镜像一次构建推 ACR/Harbor，流水线改成拉现成镜像（弃用每次 docker build）——k3s 侧讨论见 k3s部署实战.md「构建 Pod 跨节点调度与节点本地 Docker 缓存」
2. builder pod 钉到单节点（nodeSelector），层缓存稳定命中——代价是构建吞吐绑死一台 + §3.4 并发互踩问题

**与 §5.1 的区分**：ABORTED 有两种——排队取消（agent 供给不出来）和 **timeout 掐死**（日志有 `Cancelling nested steps due to timeout`），后者的掐断位置直接指出慢在哪个 step。

**Windows 线同事项**（commit 83d3871）：新增 `size_check_win.sh` stage——`arm-none-eabi-size` 校验 Flash ≤512KB / RAM ≤64KB，超限 exit 1 判失败；`.elf` 缺失在 copy 前就显式报错（堵住此前“copy 找不到 + 空归档静默绿”的链）。#120 实测：Flash 51308B（9%）/ RAM 6708B（10%）。

### 3.6 ACR 工具链镜像路线落地：从“每次现 build”到“拉现成镜像”，缓存抽奖降级为分钟级 pull（09-07 晚，#125–#133）

**定位**：§3.5 根治选项 1 落地，一个晚上跑通“推”和“拉”两半。

**提交链 × 构建实录**：

| 构建 | 北京时间 | 结果 | 验证内容 |
|---|---|---|---|
| #125 | 17:49 | ABORTED 5min（手动） | Build & Push 首跑，stage label 修正 k3s→k8s-linux（@70ce737） |
| #126 | 17:57 | ABORTED 33min（手动掐） | **公网 push 2.94GB@~500KB/s 无望**→ 定性：ECS 公网上传限速（下载不限、上传限，apt 快 push 慢的原因） |
| #127 | 18:42 | SUCCESS 26min | @b4e4233 **VPC 内网双推**：`registry-vpc.cn-hangzhou.aliyuncs.com` 推 2.94GB 仅 5.25min（~9.5MB/s，阿里云骨干不经沪杭对等链路），随后公网域名再 push 全层 `Layer already exists` 秒完；curl `401` 探测做可达性门禁 |
| #128 | 19:21 | SUCCESS 25min | @cc05f29 PUSH_IMAGE 开关引入（Build & Push skipped，日常构建不推） |
| #129 | 19:24 | SUCCESS 48min | @bfd135e PUSH_IMAGE=true 但落上海→vpc 探测失败→**回落公网 push 耗时 36min**（v0.1 首次入库 ACR；回落路径可用但慢） |
| #130 | 19:51 | FAILURE 秒挂 | declarative `docker{}` 逗号分隔语法错（@15de62a 修复，见 §2.4） |
| #131 | 20:02 | SUCCESS 18min | @cdf2d58 **Linux 腿改拉 ACR 镜像首跑通**：`agent { docker { image "..."; registryUrl ...; registryCredentialsId 'acr-cred' } }`——dockerd 拉私有仓凭证由 Jenkins 凭证下发，无需节点手工 login，与 containerd/registries.yaml 无关 |
| #132 | 20:19 | SUCCESS 16min | @64cd873 tag 钉定 v0.1；落上海本地 inspect 命中零 pull |
| #133 | 20:25 | SUCCESS 25min | 落杭州无 v0.1 → **从 ACR pull 2.94GB 约 4.5min**，pull 路线首次实测；冷节点代价从“build 55min/超时”降到分钟级 |

**关键经验**：
- **同名 tag 本地命中 = 不再 pull**：docker 插件先 `docker inspect`，查到本地有就直接用——tag 复用会拉不到新镜像（#127 推过 :latest 再 #131 用 :latest 就埋着这个坑，#132 起因即把 tag 改 v0.1 才逼出“真正从仓库拉”的验证）。结论：工具链镜像用**递增版本 tag**（v0.1→v0.2），改 Dockerfile 后手动 PUSH_IMAGE=true 跑一次再关回。
- **根治后瓶颈已经转移**（#132 实测）：MISRA-C 7.9min > agent 启动 6.6min（调度+checkout，冷 pull 另加 ~4.5min）> Build/Cppcheck/UT 各 0.3min；Windows 腿三段合计 ~30s。单阶段最慢现在是 **MISRA（cppcheck --addon 单线程扫全量 Sources）**。
- 同日 Dockerfile 两笔加固改变层哈希、顺带解释了当天几次“莫名”冷建：@e8716cd 基座锁 `debian:12-slim`（可复现构建，浮动 latest 是定时炸弹）、@7569abf apt 源换 ECS 内网 `mirrors.cloud.aliyuncs.com`——冷建耗时从 55min+ 降到 ~10min（上海实测 532MB/19s @28MB/s）；该内网源仅同地域可达，跨地域回退公网的行为与 §3.5 带宽数字对得上。注意：改这两笔会使所有节点层缓存作废，应配合递增 BUILD_IMAGE_TAG 重推 ACR。

### 3.7 `--no-install-recommends` 瘦身的隐藏炸弹：gcc 的 libc6-dev 是 Recommends，宿主机 UT 编译找不到 `string.h`（#136）

**现象**：#136（09-08 00:25，@d7993fa）FAILURE，5.5min。Windows 腿全绿（固件链接 + Size Check Flash 9%/RAM 10% 过），Linux 腿交叉编译也正常，唯独 UT 阶段炸：

```
/usr/include/CUnit/CUnit.h:53:10: fatal error: string.h: No such file or directory
make: *** [Makefile:128: build/ut/test/test_autolibc.o] Error 1
```

**根因链**（§3.6 那晚 P0 瘦身 @53cf16b 的副作用）：

1. Dockerfile 加 `--no-install-recommends` 后，`gcc` 的依赖树里 **`libc6-dev`（`/usr/include/string.h`、`stdio.h` 等 C 标准库头文件的载体）挂在 Recommends 而非 Depends**（`apt-cache show gcc`: `Recommends: libc6-dev | libc-dev`，debian:12-slim 容器内实测确认），于是被裁掉。
2. UT 用**宿主机 gcc**（Makefile `UT_CC := gcc`）编译链接 CUnit 测试 → CUnit 头在（`libcunit1-dev` 显式装了），但 CUnit 自己 `#include <string.h>` 时找不到标准头 → 炸在第三方头文件里，报错位置有迷惑性。
3. **arm 交叉编译线不受影响**：newlib 头文件来自显式安装的 `libnewlib-arm-none-eabi`，且交叉工具链不依赖 host libc6-dev——所以"编译全绿、只有 UT 挂"，一半流水线掩盖了镜像的残缺。
4. 旁证：v0.1（带 recommends）#128–#133 UT 全绿；#136 是瘦身镜像 v0.2 的**首次 UT 执行**，一跑即爆。镜像"瘦"没瘦坏，只有 UT 这一条宿主机 gcc 路径会暴露。

**修复**（@72bdd87，v0.2→v0.3 重推）：安装列表显式加 `libc6-dev`。`apt-get -s` 模拟实测：修复后 `libc6-dev + linux-libc-dev` 进安装集，体积回弹仅 MB 级（libc6-dev 本体 <4MB），不会重回 2.9GB 坏层。备选方案：`gcc` 换成 `build-essential`（gcc/libc6-dev/make 全是硬 Depends，语义上免疫 recommends 裁剪，但会多拽入 dpkg-dev 一族）。

**通用教训**：
- `--no-install-recommends` 是**逐包语义**的：对某些包纯赚（cppcheck 们的 X11/文档 recommends），对个别包是拆台（gcc↔libc6-dev、python3-pip↔部分 wheel 构建依赖这类"recommends 实为功能必需"）。瘦身后用**镜像本身跑一遍全量 CI**验证，别只看 build 阶段绿。
- 判据速查：`apt-cache show <pkg> | grep Recommends` 看被裁的是什么；只要编译报 `No such file or directory` 找的是**标准库头**，第一反应就是 libc6-dev（或交叉场景的 newlib）没进镜像。
- 巧合加成：#135（用户 23:38 手动掐，ABORTED 属 §5.1 正常语义）虽死，但 v0.2 的 apt 层已在该节点建完——#136 `docker build` 12 秒 `Using cache` 命中，瘦身目标（apt 层下载时间）当天即验证达成。

---

## 四、脚本逻辑 / Shell 兼容性类

### 4.1 `sha256sum build/*.bin` 无匹配返回非零 → 构建失败

**现象**：编译明明成功，post 阶段 exit 1 翻车。
**排查过程**：报错定位到 post success 块 → 手工执行发现 `build/UDS_S32K144_Bootloader.bin` 不存在（产物只有 .hex/.elf），glob 匹配不到文件时 sha256sum 返回非零。
**根因**：① 产物文件名假设错误 ② **post/success 块里的报错同样让整个构建 FAILURE**。
**解决**：产物先 `ls` 核对真实文件名；无匹配兜底 `|| true` 或改条件判断。

### 4.2 `type build\build.log` 报"系统找不到指定的文件"

**现象**：Windows 构建 stage 末尾读日志文件 exit 1，连续两次复现。
**排查过程**：看 stage 内命令序列——前面全是 `echo` 打屏，`mkdir build` 后没有任何写文件动作 → 读了一个从未生成的文件。
**根因**：纯 echo 只打 stdout **不落盘**；占位构建没有真正生成 build.log。
**解决**：**先写后读**——逐行 `echo xxx >> build\build.log` 保证文件存在再 type。推广：任何"读产物"的步骤，前面必须有真写动作。

### 4.3 CRLF 脚本在 Linux agent 上炸

**现象**：sh 脚本报 Bad substitution 之类怪错。
**根因**：Windows 上提交的 .sh 带 CRLF，Linux 下 `$'\r'` 污染变量/语法。
**解决**：仓库加 `.gitattributes` 强制 `*.sh text eol=lf`。

---

## 五、构建状态语义类

### 5.1 ABORTED ≠ 故障

**现象**：接 k8s agent 初期连续多次 ABORTED，日志 `Queue task was cancelled` / `Aborted by xxx`。
**排查过程**：看 build.xml 状态是 ABORTED 不是 FAILURE；日志里没有执行报错，只有排队取消 → 是 agent provision 不出来、任务在队列里被人为取消。
**根因**：pod template 未建好/label 不匹配时 job 无限排队，人等不及手动 abort。
**解决**：修 agent 供给本身（模板、label、RBAC），**别把 ABORTED 当构建 bug 排**。

### 5.2 logRotator 导致历史构建"消失"

**现象**：只能看到最近 20 次构建，早期的找不回。
**根因**：Jenkinsfile `buildDiscarder(logRotator(numToKeepStr: '20'))` 自动清理，被删记录不可恢复。
**解决**：改大保留数（`numToKeepStr: '100', artifactNumToKeepStr: '50'`）。注意**改的是仓库里的 Jenkinsfile**（commit/push），工作区副本会被重新 checkout 覆盖；另留意仓库不同分支各有一份配置时容易改错版本。

### 5.3 失败通知邮件发不出去

**现象**：`SMTP connection error while sending email. Retrying once more in 10 seconds.`
**根因**：email-ext 全局 SMTP 未配置/不通。
**解决**：Manage Jenkins → System → Extended E-mail 配 SMTP（QQ 邮箱需授权码 + 465 SSL）；配置通知迭代教训：**失败通知先保证 console echo 兜底，email 是增强**（当时走过 email→echo→echo+email 三轮）。

---

## 六、配置生效与安全注意

- **config.xml 直改流程**：停容器 → 改 → XML 校验 → 起容器（运行中改会被内存配置覆盖回写）；改前备份。
- **config.xml 多态字段用 `class` 属性式**（`<field class="全限定类名">`），嵌套子元素写法会被 XStream **静默丢弃**且不报错——重启后 grep 确认字段仍在才算保存成功（实证见 §3.4）。
- **Jenkins 重启后旧 agent pod 复用陷阱**见 §3.2 排查第 7 步——验证模板变更务必确认新 pod 的 spec。
- Codeup 个人访问令牌曾**明文内嵌**在本地克隆的 remote URL（`git remote -v` 可见）→ 建议轮换，改用 Jenkins 凭证 / credential helper。
- **gitleaks 默认规则有两个实证盲区**（09-07 接入时扫描确认，no leaks found 但真实存在）：① 无上下文裸 hex 长串不报——Jenkins agent secret（64位 hex）明文躺在 `Documentation/WIN_AGENT_SETUP.md` 入库（git log -S 命中 @4850aca）；② 只扫工作树不扫 `.git/config`——Codeup remote URL 内嵌完整 token 看不见。结论：默认规则≠安全审计，需自定义规则补齐这两类；已泄露两处待 rotate + 改占位/清洗（见待办）。
- **Jenkins API Token 未生成**，CLI 自动化受限（匿名 403），节点/配置操作目前走 config.xml 直改 + 重启。
- docker.sock 挂进构建 pod = 容器内可拿节点 root，内网可接受，公网/多租户禁用。

---

## 七、三腿并行重构 + MISRA 报告三大修复闭环（2026-09-08 下午）

> 早上一路从旧双腿（k8s-linux 编译 / intranet 检查 / windows）重构到三腿，并连续清掉三个 MISRA/publishCppcheck 的坑。提交链（dev）：`dd233a1 → 4448322 → 5351b61 → 280e4aa → baf610d → cd1a927 → a024a8d`。构建 #144–#155 全程实证。

### 7.1 三腿并行重构：消除重复 Build、慢活/快活分工

**演进**（`dd233a1 → 5351b61`）：
- 之前代码检查/UT/Coverage 挂在 `intranet-agent-01` 腿并让它自包含 `make build`，与固件编译腿重复编译、浪费。
- 机制约束：**parallel 各腿分配独立 workspace/agent，产物互不可见**；要 build 产物的腿只能自包含或 stash（后者有顺序/体积代价）。
- **最终结构**（`5351b61`）：
  ```
  Build & Push build image   (PUSH_IMAGE=false → beforeAgent 整段跳过)
  并行 3 腿：
  ├─ linux-build   (linux + docker)：Versioning→Build→Size Check→Unit Test→Coverage
  ├─ codecheck     (linux + docker)：Cppcheck→MISRA（纯静态，不依赖 build 产物）
  └─ windows-build (label windows)：Versioning→Secret Scan→Build→Size Check→Deploy
  ```
- 腿名统一小写。`-j 8` 只加在**无 addon 的普通 Cppcheck** 上，**MISRA 不加**（见 §7.3）。

### 7.2 `when`+`beforeAgent`：关闭镜像推送时连 agent 都不调度

**现象**：`PUSH_IMAGE=false`，但 `Build & Push build image` 仍 checkout / 占用等待。
**根因**：Declarative 只要 stage 声明了 `agent`，会**先分配 agent + checkout，再判 `when`**；`when` 拦不住前面的 checkout 开销（日志卡在 `git fetch --tags --force` 拉全部分支）。
**解决**（`280e4aa`）：`when { beforeAgent true; expression { env.PUSH_IMAGE=='true' } }` —— 关闭时**连 agent 分配 + checkout 一起跳过**，消除干等。前一条 149 失败正是 controller 重启中断 checkout（`SynchronousResumeNotSupportedException`，构建 #149 FAILURE 240s），属 §5 语义；顺带 `beforeAgent` 后此类空跑更少。

### 7.3 MISRA 加 `-j` 无效、不加：addon 单线程

**判断**（你实测确认）：cppcheck `-j N` 只并行"解析/语法检查"；`--addon=misra` 是**Python 后处理、单线程**，`-j` 对 MISRA 无效，且 `-j`+`--xml`+addon 有输出串扰/错位风险 → **MISRA 不加 `-j`**（`4448322` 曾加 `-j 8` 已撤）。普通 Cppcheck（无 addon）加 `-j` 安全提速。

### 7.4 修复一：publishCppcheck 通配 `**` 扫到多份残留 xml → 条数翻倍 UNSTABLE（#150/#151）

**现象**：#150/#151 UNSTABLE，日志 `total number of issues '19248' exceeds threshold '9999'`。真实 misra-report.xml 只有 **9624** 条，为何插件算 19248？
**根因链**：
1. 三腿重构后 codecheck 写 `artifacts/codecheck/Misra/misra-report.xml`，leg 在 agent 跨构建复用、不清理的 workspace；`linux-build` 与 `codecheck` 都落 `docker-linux` 可跑同一 agent 多 executor workspace。
2. `publishCppcheck pattern: '**/misra-report.xml'` 用宽泛 `**` 从 workspace 根扫描 → **命中 2 份同名 xml** → 插件合并 2×9624 = 19248 > 9999 → UNSTABLE。
3. 分水岭：#148（重构前）`Processing 1 files` SUCCESS；#150/151（重构后）`Processing 2 files` UNSTABLE。
**解决**（`baf610d`）：pattern 收窄为精确路径 `artifacts/codecheck/Misra/misra-report.xml`（同 `Cppcheck/cppcheck-report.xml`），只读本 leg 生成那份。修复后 #152 SUCCESS、处理量回 1 份。
**通用教训**：`publishCppcheck pattern` 别用裸 `**/xxx.xml`，指定相对归档**精确路径**；agent workspace 跨构建残留是常见陷阱。

### 7.5 修复二：suppress 掉 SDK S32K144 寄存器头 94% 噪声

**现象**：即便修了 §7.4，publishCppcheck 还要吃 9624 条、分钟级，threshold 被逼到 9999。
**根因**：抽查 misra-report.xml 按顶层目录分布：`SDK 9216` / `Sources 260` / `UDS_* ~140` / `Generated_Code 11`。其中**单个 `SDK/.../S32K144/include/S32K144.h` 就 8712 条 + `S32K144_features.h` 386 条 ≈ 94%** 全是厂商寄存器位操作的 MISRA 噪声。
**解决**（`cd1a927`）：两处 cppcheck（普通 + MISRA）都加 `--suppress=*:*S32K144*.h`（文件级 suppress，格式 = `错误类型:文件名`）。只滤 SDK 寄存器头，不影响对你自己 Sources/*.c 的检查。修复后 #153/#154 构建耗时从 ~407s 骤降到 ~98s/92s。

### 7.6 修复三：删 `|| true`，别让 cppcheck 真故障被静默吞掉

**现象**：MISRA/普通 cppcheck 命令都以 `2> ...xml || true` 结尾。
**根因**：`|| true` 把任何非零退出都吞成成功——找到违规返回非零是正常的（已有 `--error-exitcode=0` 兜底），但若 cppcheck **真崩溃/路径错/报告写失败**也一并静默 → 可能拿到空报告却以为"无违规"，合规静默失效。两开关冗余叠加多一层掩盖。
**解决**（`a024a8d`）：删掉两处 `|| true`，靠 `--error-exitcode=0` 保证"发现违规不 fail"、真故障正常 fail 暴露。

### 7.7 修复效果实测（构建记录）

| 构建 | commit | 结果 | 耗时 | 说明 |
|------|--------|------|------|------|
| #148 | `4448322` | SUCCESS | 585s | 重构前最后绿 |
| #149 | `5351b61` | FAILURE | 240s | controller 重启中断 checkout（非代码） |
| #150 | `5351b61` | UNSTABLE | 959s | `**` 命中 2 份 → 19248 |
| #151 | `280e4aa` | UNSTABLE | 812s | 同上 |
| #152 | `baf610d` | SUCCESS | 407s | 精确 pattern → 条数回落 |
| #153 | `cd1a927` | SUCCESS | 98s | suppress S32K144 → 大提速 |
| #154 | `a024a8d` | SUCCESS | 92s | 删 || true，保持绿 |
| #155 | `0374f01` | SUCCESS | — | 纯文档 commit（未实跑） |

**结论**：精确 pattern（条数归一）→ suppress SDK 头（量级骤降）→ 三连把 codecheck 腿从"近千秒 + 偶发 UNSTABLE"收到**~92s 稳定绿**；MISRA 报告的噪声治理是 publishCppcheck 提速的关键（publish 慢 = 条目多）。

---

## 八、Gitea 接入：MR/PR 触发多分支构建并在 PR 显示结果（2026-09-09）

> 大背景：hello-jenkins 的 Jenkinsfile 被多次重构累到复杂（见 §入门的背景），最终用户选择**换平台** —— Gitee 无法在 PR 显 commit-status（405），改用自建 **Gitea**。部署侧见 `k3s部署实战.md`「Gitea on k3s」章节；本文档聚焦 **Jenkins ↔ Gitea 对接**。

### 动机：为什么从 Gitee 迁走
- 尝尽 Gitee：`POST /api/v5/repos/{*}/statuses/{sha}` 一律 **`405 Not Allowed`** —— **Gitee 平台根本没开放 commit-status API**，Jenkins 想回写"绿勾"无门（只能 PR 评论，非 UI 状态框）
- 迁到支持 commit status + PR checks 的 **Gitea**（自建、境内、轻，适合这台小机）

### Jenkins 侧关键配置（对接现状）
- **Gitea Server 全局连接**：`org.jenkinsci.plugin.gitea.servers.GiteaServers.xml` → serverUrl=`http://101.133.228.245:30080`、credentialsId=`gitea-cred`、**`manageHooks:true`**（Gitea 插件自动给仓库挂/管 webhook —— 这就是"即使触发器区只有周期扫描，webhook 也能即时触发 MR"的关键）
- **Job：`Bootloader` = Multibranch Pipeline（多分支）**，Branch Source 指向 Gitea `http://101.133.228.245:30080/yangweiguang/hello-jenkins.git`，scriptPath=`Jenkinsfile`
- 凭证：gitea 连它用 `gitea-cred`（UsernamePassword）；既有的 `gitee-cred`/`gitee-token` 等是旧 Gitee 的

### ⚠️ 多分支触发要搞清的层
- Multibranch 的 Scan Repository Triggers **默认只有"周期扫描"**，没有现成 PR trigger 可选 —— 这是正常的
- 但 Gitea 插件对**多分支**的 webhook 即时触发是另一条路：靠 `manageHooks` 自动挂的 webhook，PR 事件打到 `/gitea-webhook/post` → Jenkins 识别仓库 → 触发多分支**重扫**。**不需要在周期扫描里折腾**
- 想深度用 PR/mr 类型的分支发现（把每个 PR 变独立分支的完整形态），需把 Branch Source 从通用 `GitSCMSource` 换成 Gitea 的 `GiteaSCMSource`（gitea.jpi 内置 `BranchDiscoveryTrait`/`ForkPullRequestDiscoveryTrait`）——当前 Bootloader 用通用 Git 源已有 PR-1 出现，够用，暂不换

### Gitea 侧 webhook 配置
- 仓库 → Settings → Webhooks → Gitea，type=`gitea`、content=`json`、**Target URL=`http://122.51.10.97:8080/gitea-webhook/post`**
- 事件：MR/PR 相关勾 `pull_request`（push 提醒另勾，按需）；`branch_filter=*`
- 网络：Gitea(101.133)→ Jenkins(122.51.10.97:8080) 三层 curl 均 403（匿名根路径限制），但 `/gitea-webhook/post` 匿名端点可达，可触发

### ✅ 验证（用户报喜的里程碑）
- Gitea PR #1 每次 reopened/closed/UPDATED，Jenkins logs 立即有：
  ```
  Pull request #1 ... UPDATED event from 101.133.228.245 ⇒ http://122.51.10.97:8080/gitea-webhook/post ... processed in 0.4s
  ```
- Bootloader 下出现 `dev` / `main` / `PR-1`，PR-1 构建 SUCCESS —— **MR(PR)触发链路完全正常、webhook 即时驱动**，Gitea PR 页终于能看到 Jenkins 构建结果 🎉

### 待确认/欠账
- 日志仍有 `GiteaChecksPublisher ... HTTP 401 Unauthorized`（回写 PR checks 时）：PR 结果已能看，但想 checks 图标完整全绿，要确认 `gitea-cred` 权限或改用 token 型凭据做发布通道
- `/root/code/hello-jenkins` remote URL 内嵌明文密码 → 建议换 Gitea token / SSH

---

## 九、cppcheck & clang-tidy 安装（OpenCloudOS 9.4）

> 静态检查工具在构建/质检节点上的安装方式。环境：OpenCloudOS 9.4（RHEL 9 系，x86_64），已具备 gcc/g++、cmake、make、yum/dnf。

### 方案一：dnf 在线安装

```bash
sudo dnf install -y epel-release
sudo dnf install -y cppcheck
sudo dnf install -y clang-tools-extra
```

### 方案二：离线 RPM

```bash
# 在外网同架构机器下载
dnf download --resolve cppcheck --destdir=/tmp/cppcheck-rpms/
dnf download --resolve clang-tools-extra --destdir=/tmp/clang-tidy-rpms/

# 传到目标机器后安装
sudo rpm -ivh /tmp/cppcheck-rpms/*.rpm
sudo rpm -ivh /tmp/clang-tidy-rpms/*.rpm
```

### 方案三：源码编译

待补充。

### 验证

```bash
cppcheck --version
clang-tidy --version
```

### 与 Jenkins 流水线的衔接

- **cppcheck** 常用于流水线的 Cppcheck / MISRA-C 质量门禁（见 §7.3–7.6：`-j` 对 MISRA addon 无效、publishCppcheck 通配坑、S32K144 suppress、去掉 `|| true`）
- **clang-tidy** 用于 C/C++ 更细粒度的静态诊断，可按需接入 codecheck 阶段

---

## 待办
- [x] `${ARTIFACT_DIR}` 归档路径写死 + `allowEmptyArchive: false` 门禁（@83d3871，#120 核对两平台 Artifacts 齐）
- [x] 工具链镜像推仓库根治层缓存抽奖（2026-09-07 晚 ACR 路线落地，#125–#133 实测，见 §3.6）
- [x] MISRA-C ~8min 优化：#150/151 定位为 publishCppcheck 扫到多份 xml + SDK S32K144 噪声（非扫描本身）；suppress S32K144*.h 后 #153/#154 codecheck 骤降 ~92s（见 §7.4–7.7，2026-09-08）；剩余可选项：手动分目录并行/收窄扫描目录，待评估
- [ ] 工具链镜像 tag 升级流程化：改 Dockerfile → PUSH_IMAGE=true 跑一次 → BUILD_IMAGE_TAG 递增（防 §3.6 同名 tag 不重拉坑；可考虑 tag 直接用 Dockerfile 内容 hash）；§3.7 的 v0.2→v0.3 已照此流程手动执行
- [x] 瘦身镜像 UT 首跑验证：#137（@72bdd87，v0.3 + libc6-dev）SUCCESS 18min——v0.3 冷建 5.7min→VPC 推 2min→公网秒去重；上海节点 inspect 未命中 pull 仅 1.5min；UT 首跑即绿（string.h 问题解决），Coverage 23.3%，Size Check 过，§3.7 修复闭环（2026-09-08 07:46）
- [ ] email-ext SMTP 配置
- [ ] win-agent-01 与 k8s-linux 两条线 stage 差异收敛（bat/sh 双实现或分支）
- [ ] 考虑 docker.sock 权限收紧（GID 对齐替代 chmod 666）；Codeup token 轮换
