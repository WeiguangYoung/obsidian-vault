---
{"dg-publish":true,"permalink":"/knowledge/devops/practices/02-Jenkins流水线实战/","tags":["Jenkins","Pipeline","Jenkinsfile","CI/CD","排障","实战记录"],"dg-note-properties":{"date":"2026-09-05","tags":["Jenkins","Pipeline","Jenkinsfile","CI/CD","排障","实战记录"]}}
---


# Jenkins 流水线实战（问题排查与解决 · hello-jenkins）

> Jenkins 任务：`hello-jenkins`（仓库 codeup.aliyun.com/example-org/hello-jenkins，dev 分支）
> 环境：Jenkins Master（Docker 8080/50000）+ 三类 agent —— ①docker/linux ②`win-agent-01`（label `windows`）③k3s 动态 pod（label `k8s-linux`，双节点跨 VPC：server 203.0.113.11 杭州 / agent 203.0.113.13 上海）
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

### 2.5 `sh ''' '''` 里的 shell 变量，`post` 块插值取不到 → MissingPropertyException

**现象**：sherpa-lite #43，前面几个 stage（编 .so、打 APK×2）全绿，到 `UT + 覆盖率` 的 **post 块**整体 FAILURE：
```
groovy.lang.MissingPropertyException: No such property: DEMO for class: groovy.lang.Binding
	at WorkflowScript.run(WorkflowScript:294)   ← UT post 块
	at WorkflowScript.run(WorkflowScript:305)   ← publishHTML 的 reportDir
```
后面的 stage 被 skip，构建红。

**根因**：`DEMO=android/SherpaOnnxJavaDemo` 写在 `sh ''' '''` **脚本体内部**，那只是 shell 进程内的变量，sh 一退出就消失。而 post 块里的 `"${DEMO}/app/build/..."` 是 **Groovy 字符串插值**，Groovy 在自己的 binding 里找不到 `DEMO` → 抛异常。

**两个命名空间必须分清**：
| 位置 | 归属 | 谁能看见 |
|---|---|---|
| `sh ''' DEMO=xxx '''` 内部 | shell 变量 | 只有这次 sh 进程 |
| `post` / `steps` 里的 `"${DEMO}"` | Groovy 插值 | 只在 Groovy binding 里找 |

`sh ''' '''` 是两者之间的墙，变量穿不过去。跨 stage / 跨 post 块共享的值，必须声明在 **`environment`**（或 pipeline 顶层 `def`）。

**为什么之前没炸**：早期 UT stage 没有覆盖率报告，post 块里没有 `${DEMO}` 插值；后来加了 `publishHTML`/`xunit` 才引入。要同时满足「走到该 post 块」+「插值真被求值」才暴露——#43 是第一次两个条件都成立。

**为什么这次才发现（假绿掩盖）**：UT 主体 `gradlew testDebugUnitTest jacocoTestReport` 输出 `25 actionable tasks: 25 up-to-date`——Gradle 缓存全命中、根本没真跑，于是 stage「看起来」过了，一路走到 post 才撞上作用域 bug。

**解决**（sherpa-lite #44，commit `dd5c6d9c`）：把 `DEMO` 从 shell 层提到 Groovy `environment` 层：
```groovy
environment {
    DEMO_KOTLIN = 'android/SherpaOnnx'
    DEMO_JAVA   = 'android/SherpaOnnxJavaDemo'
}
// sh 里读环境变量，一个值两界可见：
sh '''
    DEMO="${DEMO_JAVA}"
    cd "${DEMO}" && ./gradlew ... jacocoTestReport
'''
post { always {
    publishHTML(target: [reportDir: "${DEMO_JAVA}/app/build/reports/jacoco/jacocoTestReport/html", ...])
} }
```

**教训**：
- **写 Jenkinsfile 先问一句「这行是 Groovy 还是 shell」**——混了就静默塌，且往往在 `post`/收尾阶段才炸，此时主任务已绿，极易误判
- 与 §2.2（单引号不展开）是同一类错误的不同变体：**都是「Groovy 字符串的世界」和「shell 的世界」之间的边界没搞清**
- 顺带：`25 actionable tasks: 25 up-to-date` 这种**全缓存命中**的「测试通过」含金量存疑，需 `--rerun-tasks` 或 `clean` 才可信

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

### 3.8 k8s 动态 agent 全面失效：跨云 JNLP + 工作目录属主 + docker.sock 权限三重坑（2026-09-16）

**背景**：固定节点 `intranet-agent-01`（家中 Linux 机）/`win-agent-01`（家中 Windows 机）双双离线后，任务首次真正落到 k8s 动态 agent 兜底路径，一启用即全线卡死——**这条兜底路径此前从未经过实战，配置"看着对"但一用就爆**。

**故障表象**（Bootloader PR-2 / hello-jenkins #188 / sherpa-lite #35-#38 同源）：

| 表象 | 实际含义 |
|---|---|
| `Still waiting to schedule task` + `All nodes of label 'docker-linux' are offline` | 固定节点离线，任务无限排队 |
| Pod 起来秒挂 `Containers are terminated with exit codes: {jnlp=1}` | agent 进程启动即退出 |
| Pod 卡 `ContainerCreating`（落 home-node） | 家宽拉 docker.io 镜像抖动 |

**根因链（三层，逐个钉死）**：

1. **跨云 JNLP 连不上**：模板 `webSocket=false` → agent 走**传统 JNLP/TCP 50000 端口**回连腾讯云主控；跨云（阿里云→腾讯云）该链路不稳，Jenkins 侧日志满是 `Connection #xxx failed: null`。**验证法**：在同款 Pod 里手动 `java -jar agent.jar -webSocket` → `WebSocket connection open / Connected` 秒连；不加 `-webSocket` 才失败。
2. **工作目录属主错位**：HostPath workspace `/home/jenkins/agent` 由 k3s 以 **root** 身份创建，容器却以 `uid=1000(jenkins)` 运行 → agent entrypoint 写工作目录被拒 → `exit 1`。**判据**：`ls -ld` 看属主是 root 而容器内 `id` 是 1000。（手动测试用 `-jar /usr/share/jenkins/agent.jar` 会绕开工作目录、误判"能连"，必须看 entrypoint 路径。）
3. **docker.sock 权限不足**：`docker pull` 报 `permission denied ... dial unix /var/run/docker.sock`——socket 属 `root:docker(119) 0660`，容器内 uid 1000 **不在 docker 组**。宿主机 `yang`(uid 1000) 在 docker 组所以宿主机没事，但容器里没有该组。

**修复（六项，全部落模板/节点）**：

| # | 项 | 值 | 解决 |
|---|---|---|---|
| 1 | pod template `webSocket` | `false → true` | 走 8080 而非 50000，避开跨云 JNLP |
| 2 | `slaveConnectTimeout` | `1000 → 60000` | 合理超时（原 1s 不够、后调 300s 太长） |
| 3 | 工作目录属主 | `chown -R 1000:1000 /home/jenkins/agent`（节点侧） | jenkins 用户可写 |
| 4 | pod template Raw YAML 加 `initContainers` | `busybox` 跑 `chown -R 1000:1000` | 自动修权限，防重建回退 |
| 5 | pod template `securityContext.supplementalGroups` | `[119]`（home-node docker 组 GID） | 容器内可访问 docker.sock |
| 6 | pod template `nodeSelector` | `kubernetes.io/hostname: home-node` | 钉住唯一有资源的构建节点，防飘到别处静默失效 |

**最终模板 YAML（Raw YAML Overrides 段）**：

```yaml
spec:
  nodeSelector:
    kubernetes.io/hostname: home-node
  securityContext:
    supplementalGroups: [119]
  dnsPolicy: Default
  initContainers:
    - name: fix-agent-perms
      image: busybox:1.36
      command: ['sh','-c','chown -R 1000:1000 /home/jenkins/agent']
      volumeMounts:
        - { mountPath: /home/jenkins/agent, name: workspace-volume }
  containers:
    - name: jnlp
      volumeMounts:
        - { mountPath: /usr/bin/docker,      name: docker-cli, readOnly: true }
        - { mountPath: /var/run/docker.sock, name: docker-sock }
  volumes:
    - { name: docker-cli,  hostPath: { path: /usr/bin/docker } }
    - { name: docker-sock, hostPath: { path: /var/run/docker.sock } }
```

**验证结果**：agent 端 `Using /home/jenkins/agent/remoting`（目录可写）+ `WebSocket connection open` + `Connected`；GID 测试 Pod 内 `groups=1000(xxx),119` + `docker ps` 无 permission denied；#38 `docker pull` 正常执行（不再 403）。

**通用教训**：
- **兜底路径必须定期演练**——"配好了"不等于"能用"，长期不启用的配置等于没有配置。固定 agent 一挂，隐藏缺陷集中爆发。
- **跨云 JNLP 优先 WebSocket**：`-secret`/网络都不是问题，50000 端口跨云 TCP 才是；WebSocket 复用 8080 天然穿透。
- **容器内 uid 与节点目录/设备属主必须对齐**：HostPath 由 k3s root 创建 → 权限必错位，用 initContainer 自动纠偏比手工 chown 可靠。
- **socket 类设备授权用 `supplementalGroups` 而非 `chmod 666`**：前者精确给组、后者等于给容器 root。
- **改 pod template 后必须重启 Jenkins + 清旧 pod**：已 provision 的旧 agent 带旧 spec 继续服务，容易误判"改了没用"（同 §3.2 第 7 步）。

**当时残留（业务层，另排）**：`静态检查 (cppcheck)` / `(ktlint Kotlin)` 分支仍失败——属工具缺失/代码问题，与基础设施无关。

---

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
- **Gitea Server 全局连接**：`org.jenkinsci.plugin.gitea.servers.GiteaServers.xml` → serverUrl=`http://203.0.113.13:30080`、credentialsId=`gitea-cred`、**`manageHooks:true`**（Gitea 插件自动给仓库挂/管 webhook —— 这就是"即使触发器区只有周期扫描，webhook 也能即时触发 MR"的关键）
- **Job：`Bootloader` = Multibranch Pipeline（多分支）**，Branch Source 指向 Gitea `http://203.0.113.13:30080/example-org/hello-jenkins.git`，scriptPath=`Jenkinsfile`
- 凭证：gitea 连它用 `gitea-cred`（UsernamePassword）；既有的 `gitee-cred`/`gitee-token` 等是旧 Gitee 的

### ⚠️ 多分支触发要搞清的层
- Multibranch 的 Scan Repository Triggers **默认只有"周期扫描"**，没有现成 PR trigger 可选 —— 这是正常的
- 但 Gitea 插件对**多分支**的 webhook 即时触发是另一条路：靠 `manageHooks` 自动挂的 webhook，PR 事件打到 `/gitea-webhook/post` → Jenkins 识别仓库 → 触发多分支**重扫**。**不需要在周期扫描里折腾**
- 想深度用 PR/mr 类型的分支发现（把每个 PR 变独立分支的完整形态），需把 Branch Source 从通用 `GitSCMSource` 换成 Gitea 的 `GiteaSCMSource`（gitea.jpi 内置 `BranchDiscoveryTrait`/`ForkPullRequestDiscoveryTrait`）——当前 Bootloader 用通用 Git 源已有 PR-1 出现，够用，暂不换

### Gitea 侧 webhook 配置
- 仓库 → Settings → Webhooks → Gitea，type=`gitea`、content=`json`、**Target URL=`http://203.0.113.12:8080/gitea-webhook/post`**
- 事件：MR/PR 相关勾 `pull_request`（push 提醒另勾，按需）；`branch_filter=*`
- 网络：Gitea(203.0.113.13)→ Jenkins(203.0.113.12:8080) 三层 curl 均 403（匿名根路径限制），但 `/gitea-webhook/post` 匿名端点可达，可触发

### ✅ 验证（用户报喜的里程碑）
- Gitea PR #1 每次 reopened/closed/UPDATED，Jenkins logs 立即有：
  ```
  Pull request #1 ... UPDATED event from 203.0.113.13 ⇒ http://203.0.113.12:8080/gitea-webhook/post ... processed in 0.4s
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

---

## 十、sherpa-lite 流水线 #43→#54 十二连败排障实录（2026-09-17 ~ 09-21）

> 任务：`sherpa-lite`（Gitea `example-org/sherpa-lite`，main 分支，多分支流水线）
> 目标：sherpa-onnx Android arm64 一体化构建（.so + Kotlin/Java 两个 demo APK + UT + 三腿静态检查）
> 结局：**#54 首次全绿**（`构建结束: SUCCESS`），12 个构建连环排障闭环。
> 本文按 **现象 → 排查 → 根因 → 解决** 记录，坑都可复用。

### 10.0 总览对照表（一图看清 12 个构建）

| # | 提交 | 结果 | 真因 | 修复 |
|---|---|---|---|---|
| #43 | e8ebfe39 前 | ❌ | Groovy/shell 变量作用域（`No such property: DEMO`） | DEMO 提升为 `environment` 常量 |
| #44 | dd5c6d9c | ❌ | 作用域修好，但 `publishHTML does not exist` | — |
| #45 | 8584e3a5 | ❌ | UT 缓存假绿（25 up-to-date），jacocoTestReport SKIPPED | 加 `--rerun-tasks` |
| #46 | 66f5bda6 | ✅ | 治标：报告缺失不阻断 | `publishHTML allowMissing: true` |
| #47 | e8ebfe39 | ❌ | checkout `dubious ownership` + 三支并行抢锁 | 三支各独立 `ws()`（后被证伪） |
| #48 | 0dc45a0c | ❌ | `ws()` 在 built-in 节点造出不存在的路径 → exit -2 | 回退 `ws()` |
| #49 | d4f7bb94 | ❌ | Java APK 依赖下载 DNS 解析失败（家宽节点） | — |
| #50 | 6d20d788 | ❌ | `dubious ownership` 只在三支里有兜底，上游 stage 没覆盖 | — |
| #51 | — | ⚠️ABORTED | webhook 抢占（`disableConcurrentBuilds`） | 非问题 |
| #52 | d4f7bb94 | ❌ | ktlint 2738 条暴涨（glob 扩展）；`COUNT` 换行坑 | 阈值 44→100 / 161→3000 / 补 500 |
| #53 | c3814317 | ❌ | `androidLint` DSL 方法不存在（插件未装） | 暂时停用 recordIssues |
| **#54** | 05eaa735 | ✅ | — | **SUCCESS** |

---

### 10.1 【核心坑】Jenkins master 位置定位

**现象**：以为 Jenkins 在杭州 k3s 上，ssh 去 `203.0.113.11` 找半天没有。

**排查**：
```bash
kubectl get ns                    # jenkins 命名空间存在但 No resources found
kubectl get all -A | grep -i jenkins   # 无结果（agent 是按需动态起 Pod）
```

**根因**：**Jenkins master 就在本机**（腾讯云 `203.0.113.12` = VM-0-7），容器名 `jenkins`。

**解决**：
```bash
docker exec jenkins cat /var/jenkins_home/jobs/sherpa-lite/branches/main/builds/<N>/log
```
**教训**：查 Jenkins 先 `docker ps | grep jenkins` 看本机，别默认它在远端集群。

---

### 10.2 【坑】Git `dubious ownership` 导致 checkout 失败

**现象**：
```
fatal: detected dubious ownership in repository at '/home/jenkins/agent/workspace/sherpa-lite_main/.git'
fatal: Could not read from remote repository.
ERROR: Checkout failed
ERROR: Maximum checkout retry attempts reached, aborting
```

**排查过程（三次迭代才修对）**：
1. **#47 初次**：以为加 `git config --global --add safe.directory "*"` 到三支静态检查里就行 → 但**上游 stage（编译 .so / 打 APK / UT）在 checkout 时就炸了**，根本走不到静态检查
2. **#47 误判**：同判三支"并行共用 workspace 抢锁"，加了独立 `ws("sc-xxx")` → **#48 证明这是错的**（下面 10.3）
3. **#50 定位**：`dubious ownership` 其实来自 **CMake 拉 eigen 时调 git**（warning 级，非致命）+ **顶层真正 checkout 失败**

**根因**：agent 容器执行用户与 workspace `.git` 属主不一致，Git 新版安全检查（`safe.directory`）拦截。

**解决（#50，最终版）**：新增**所有 stage 之前的全局预处理 stage**，写入 `~/.gitconfig`：
```groovy
stage("环境预处理 (git safe.directory)") {
    agent { label "docker-linux" }
    steps {
        sh '''
            set -x
            git config --global --add safe.directory "*" || true
            git config --global --add safe.directory "${WORKSPACE}/.git" || true
            git config --global --add safe.directory "${WORKSPACE}" || true
            git config --global --list || true
        '''
    }
}
```
**关键**：safe.directory 必须覆盖到**所有** stage，不能只写在静态检查那一支。

**辨析**：日志里 `编译 .so` 阶段也报同一条 `dubious ownership`，但那是 CMake `FetchContent` 拉 eigen 时顺手 `git describe` 探测版本，**探测失败不影响 eigen 本体（已从 tarball 解压）**，CMake 会继续 → **属 warning 级，不是真凶**。

---

### 10.3 【坑】`ws()` 在 built-in 节点造出不存在的路径 → `process apparently never started`

**现象**（#48）：
```
08:19:22 process apparently never started in /home/jenkins/agent/sc-cppcheck@tmp/durable-91a3ab8d
(running Jenkins temporarily with -Dorg.jenkinsci.plugins.durabletask.BourneShellScript.LAUNCH_DIAGNOSTICS=true might make the problem clearer)
script returned exit code -2
```

**排查**：Jenkins UI 显示 `Took 19 分 on built-in` —— **构建被调度到 built-in（本机 Jenkins 容器）节点**，而该节点上**没有 `/home/jenkins/agent` 这个挂载**（那是 k8s agent 的 remoteFS）。

**根因**：`ws("sc-cppcheck")` 会创建 `/home/jenkins/agent/sc-cppcheck`，但在 built-in 节点上该路径不存在 → **shell 进程起不到来** → 5 分钟后超时 `exit -2`。

**辨析**：`exit -2` 是"进程没起来"的哨兵值，**不是业务失败**（业务失败是 `exit 1`）。

**解决（#48）**：
1. 立即回退 `ws()`（去包裹，保留 `git safe.directory`）
2. 正解是后续用**全局预处理 stage**（10.2）根治属主问题，而非靠 `ws()` 隔离

**拓展**：`docker inspect -f . <image>` 这步**通过**（0.49 秒）说明镜像在的 → 排除"镜像 No such object"的误判。

---

### 10.4 【坑】k8s 动态 Pod 落在 Lenovo（家宽节点）→ DNS 解析失败

**现象**（#49）：
```
> Could not resolve com.github.k2-fsa.sherpa-onnx:sherpa-onnx:v1.13.7
  > Could not GET 'https://maven.aliyun.com/repository/public/...'
  > Could not GET 'https://jitpack.io/...'
     > jitpack.io: Temporary failure in name resolution    ← 关键
BUILD FAILED
```

**排查**：
```bash
kubectl get pods -n jenkins -o wide
# k8s-linux-4vzg6  10.42.2.183  home-node     ← Pod 落在 Lenovo
```
**根因**：Jenkins k8s cloud 的 Pod Template **没做 node 亲和** → Pod 随机调度，**可能落 Lenovo**。而 Lenovo 是**家宽节点**（无公网 IP、走 WG overlay、MTU 1420→1370），**家宽 DNS 抖动**（UDP 小包易丢）→ `Temporary failure in name resolution`。

**为什么 #46 能过**：Gradle 依赖有本地缓存，`UP-TO-DATE` 时不联网；但 Java demo 的 Jacoco + JitPack 依赖**从没成功缓存过**（#47/#48 没跑到那一步）→ 每次现拉 → 一遇抖动就炸。

**解决**：改 Jenkins k8s Pod Template 的 YAML（`/var/jenkins_home/config.xml`），显式指定 DNS：
```yaml
dnsPolicy: None
dnsConfig:
  nameservers:
    - 223.5.5.5      # 阿里公共 DNS
    - 119.29.29.29   # DNSPod
  searches:
    - svc.cluster.local
    - cluster.local
  options:
    - name: ndots
      value: "5"
```
改完 **`docker restart jenkins`** 让配置生效（光改 config.xml 不 reload 无效）。

**关键认知**：`--network=host` 的容器加 `--dns` **不生效**（Docker 行为）→ 必须在 **Pod 层面**配 dnsConfig。且 `intranet-agent-01` 与 Lenovo 在同一局域网（家宽），**共享同一条家宽出口** → DNS 修复对两者都有价值。

---

### 10.5 【坑】质量门禁阈值过紧 + glob 扩展导致暴涨

**现象**（#52）：
```
--- ktlint 报告摘要：2738 条 ---
❌ ktlint 发现 2738 条问题（阈值 161），构建阻断
```

**根因**：ktlint 的 glob 从 `android/SherpaOnnx/**/*.kt` 扩到 `android/**/*.kt`（多扫了 Java demo 的 Kotlin 源码），**存量从 161 暴涨到 2738**，旧基线完全失效。

**解决**：按实测存量调大门禁阈值，留 buffer：
```groovy
// 基线（实测，按下调大留 buffer，避免存量误伤构建）
CPPCHECK_MAX_ERRORS = '100'   // 实测 44
KTLINT_MAX_ERRORS   = '3000'  // 实测 2738
LINT_MAX_ISSUES     = '500'   // Android Lint 实测 0
```
**心得**：质量门禁要**先测存量、再定基线**，不能拍脑袋写 0/161。逐步收紧才是正路。

---

### 10.6 【坑】`grep -c || echo 0` 在 shell 里产生换行 → `Illegal number`

**现象**（#52 Android Lint 支）：
```
+ COUNT=$(grep -c "<issue " ... || echo 0)
+ echo 0
+ COUNT=0
+ [ 0
0 -gt 500 ]                     ← 数字被拆成两行
script.sh.copy: 23: [: Illegal number: 0
```

**根因**：`grep -c` 匹配 **0 条**时**输出 `0` 但退出码 1** → 触发 `|| echo 0` → 两个 `0` 都进了命令替换 → `COUNT="0\n0"` → `[ 0\n0 -gt 500 ]` 语法错误 → `set -e` 判死。

**为什么 ktlint/cppcheck 那两支没炸**：它们匹配数 > 0（2738、44），grep 退出码 0，不触发 `echo 0`。

**解决**：改用 `wc -l`（永远输出单个数字、退出码 0，无坑）：
```bash
# 改前（有坑）
COUNT=$(grep -c "<issue " file 2>/dev/null || echo 0)
# 改后（无坑）
COUNT=$(grep -o "<issue " file 2>/dev/null | wc -l)
```
**教训**：`set -e` 环境下，`grep -c` + `|| echo` 兜底是经典双输出坑，一律用 `grep -o | wc -l`。

---

### 10.7 【坑】`recordIssues(tool: androidLint(...))` → DSL 方法不存在

**现象**（#53）：
```
java.lang.NoSuchMethodError: No such DSL method 'androidLint' found among steps
  [acceptGiteeMR, addGiteeMRComment, archive, bat, build, catchError, checkout, ...]
```
**且**脚本已打印 `✅ Android Lint 通过（0 ≤ 500）`，归档也成功，但分支仍被判 `Failed`。

**排查**：
```bash
unzip -l /var/jenkins_home/plugins/warnings-ng.jpi | grep -i android
# icons/android-lint-24x24.png   ← 只有图标，没有 parser 类
cat .../warnings-ng/META-INF/MANIFEST.MF   # 13.10258（2026-08，很新）
```
**根因**：`androidLint` parser 属 **Warnings NG 的独立可选插件**（`android-lint`），当前 Jenkins **没装**。装了 `warnings-ng` + `analysis-model-api` 不等于装了它。

**解决（#53，临时）**：注释掉 `recordIssues(tool: androidLint(...))`，保留门禁脚本 + `archiveArtifacts` 归档 → 只是少了 Jenkins UI 趋势图。
```groovy
// TODO: androidLint parser 属独立插件, 当前 Jenkins 未装 -> NoSuchMethodError
// recordIssues(tool: androidLint(pattern: 'codecheck/lint-java-report.xml', ...), ...)
```
**后续**：装 `android-lint` 插件后恢复。

---

### 10.8 【坑】多分支构建被 webhook 抢占 → ABORTED

**现象**（#51）：`Finished: ABORTED`，日志里同一分钟内出现两次 `Commit message`。

**根因**：`options { disableConcurrentBuilds() }` → 推新提交触发新构建时，**正在跑的构建被掐掉让位**。

**辨析**：`ABORTED` ≠ `FAILURE`，是**并发策略的正常表现**，不是 bug。查日志时别误判。

---

### 10.9 最终状态（#54 全绿）

```
构建结束: SUCCESS（see artifacts: .so tar.gz + debug apk）
```
| Stage | 结果 |
|---|---|
| 环境预处理 (git safe.directory) | ✅ |
| 编译 .so (arm64-v8a) | ✅ |
| 打 APK (Kotlin) | ✅ |
| 打 APK (Java) | ✅ |
| UT + 覆盖率 | ✅ |
| 静态检查 cppcheck | ✅ 44 ≤ 100 |
| 静态检查 ktlint | ✅ 2738 ≤ 3000 |
| 静态检查 Android Lint | ✅ 0 ≤ 500 |
**归档产物**：`output/{java,kotlin}/app-debug.apk`、`ci-artifacts/sherpa-onnx-android-libs-arm64-v8a.tar.gz`、`codecheck/{cppcheck,ktlint,lint-java}-report.xml`

### 10.10 遗留技术债
- [ ] **androidLint 插件安装** → 恢复 `recordIssues(tool: androidLint(...))`（TODO 已留注释块）
- [ ] **`jacocoTestReport` 恒 SKIPPED**：`app/build.gradle` 的 `debug { testCoverageEnabled true }` 与手动注册的 `JacocoReport` task 冲突，AGP 把后者 skip。正解：去掉 `testCoverageEnabled`，或改用 AGP 自带 `createDebugUnitTestCoverageReport`。当前 `allowMissing: true` 止血，**覆盖率报告实际没产出**
- [ ] **APK 备份**（`/data1/backups/sherpa-lite/`）：三次失败（PTY 破坏二进制 / scp 被杀 / 双进程并发写）。正解：`rsync --partial --progress --append` 后台单实例 + 双侧 `md5sum` 校验
- [ ] **构建节点网络**：`intranet-agent-01` 与 k8s 的 Lenovo **同处家宽局域网**，DNS 抖动是共性风险；考虑是否把构建节点迁到云上

### 10.11 Gitea 推送备忘
- Gitea：k3s Pod（上海 `203.0.113.13`），NodePort **30080**，`http://203.0.113.13:30080/example-org/sherpa-lite.git`
- 推送执行机：杭州 `203.0.113.11`（有 git 2.34）
- 认证：账号 `example-org`，密码 `<GITEA_PASS>`（非交互 push 用内嵌密码 URL；曾出现 `remote: Failed to authenticate user` —— 实为 shell 转义把密码写坏，非 token 失效）
- 推送即触发 webhook → 构建

---

## 十一、Halo 流水线"从零到出运行镜像"全流程（2026-09-22）

> 任务：`halo`（Gitea `example-org/halo`，main 分支，Multibranch）
> 目标：Java（Gradle）前后端同仓项目，拉取 → 编译+单测 → 打包 JAR → 打运行镜像 → 推 ACR
> 结局：**#14 全绿**（`Finished: SUCCESS`），并产出可直接部署的运行镜像；随后部署到 k3s（见 `04-k3s部署实战.md` §Halo 部署实战）。
> 与 sherpa-lite 的差别：这是**首个"jar 进运行镜像"的完整交付链路**（sherpa 是 Android 产物），且用了 `withDockerRegistry` + `withDockerContainer` 这一对**已定稿的正确姿势**。

### 11.0 总览对照表（构建 #4 ~ #14）

| # | 结果 | 真因 | 修复 |
|---|---|---|---|
| #4~#5 | ❌ | 私有镜像拉取 `manifest not found` / 容器内 `gradlew: No such file` | 换 `withDockerRegistry` + `withDockerContainer`（照抄 sherpa-lite） |
| #5 | ❌ | `error: Promise.withResolvers is not a function` | Node 20 → 22（Halo 前端 vite 7 要求 Node≥22） |
| #5~#6 | ❌ | 老问题复现（镜像没换新） | 确认 ACR 上 digest，重建推镜像 |
| #6 | ❌ | `:api:spotlessJavaCheck` → `NoClassDefFoundError: ImmutableRangeMap$Builder` | `-x spotlessCheck -x spotlessJavaCheck -x spotlessMiscCheck` |
| #9 | ❌ | `Task 'spotlessKotlinCheck' not found` | **halo 无 kotlin，`-x` 不存在的任务会直接报错** → 去掉 |
| #11 | ✅ | — | 编译+单测+打包 JAR 首次全绿（打镜像 stage 暂 `when` 跳过） |
| #12 | ❌ | `gradle-9.7.0-bin.zip ... timeout` | `gradle-wrapper.properties` distributionUrl 换**腾讯镜像** |
| #13 | ❌ | `Step 1/17 FROM eclipse-temurin:21-jre` → `registry-1.docker.io` 超时 | **Dockerfile FROM 换 daocloud 镜像** |
| **#14** | ✅ | — | **SUCCESS，运行镜像已推 ACR** |

### 11.1 【核心】两把钥匙：`withDockerRegistry` + `withDockerContainer`

**坑 A：私有镜像拉取报 `manifest not found`（或静默失败）**
- 现象：`agent { docker { image '...:latest', registryCredentialsId: 'acr-cred' } }` 写法，docker 插件**认不出私有仓凭证** → 拉不到构建镜像
- **正解（sherpa-lite 已验证）**：改用 `withDockerRegistry` 包裹：
```groovy
withDockerRegistry(credentialsId: 'acr-cred', url: 'https://registry.cn-hangzhou.aliyuncs.com') {
    sh 'docker pull registry.cn-hangzhou.aliyuncs.com/example-org/halo-build:latest'
    // ... 构建
}
```

**坑 B：容器内 `./gradlew: No such file or directory`（exit 127）**
- 现象：手动 `docker run -v ${WORKSPACE}:/src -w /src ...`，容器里看不到源码；且 `__cacert_entrypoint.sh: /src/gradlew: No such file`
- **根因（关键认知）**：**构建跑在 `intranet-agent-01` 上，`${WORKSPACE}` = 那台机器的 `/home/jenkins/agent/workspace/halo_main`**；手工 `-v` 挂载路径与 Jenkins 实际 workspace 机制错位
- **正解**：改用 `withDockerContainer`（Jenkins docker-workflow 插件自动处理 workspace 挂载，无需手写 `-v`）：
```groovy
withDockerContainer(args: '-u 0:0 -v /var/run/docker.sock:/var/run/docker.sock', image: '...halo-build:latest') {
    sh './gradlew --no-daemon -I /root/.gradle/init.gradle clean check ...'
}
```
**教训**：Jenkins 里跑容器化构建，**永远用 `withDockerContainer`，不要手写 `docker run -v`**。

### 11.2 【坑】Node 版本：`Promise.withResolvers is not a function`

- 现象：`> Task :ui:buildPackages FAILED`，前端报 `error: Promise.withResolvers is not a function` / `[ELIFECYCLE] Command failed with exit code 1`
- 根因：`Promise.withResolvers()` 是 **Node 22+** API；构建镜像里是 Node 20；Halo 前端（vite-plus / vite 7）**要求 Node≥22**
- **解决**：改 `Dockerfile.build` 的 `setup_20.x` → `setup_22.x`，重建推 ACR
- **辨析**：改了镜像重建后 **#6 仍报同一错**，是因为 **agent 拉到的是本地缓存的旧镜像**——`docker pull` 对同 tag 会命中本地。**验证法**：进容器 `node -v`，或看 pull 日志的 digest 是否变化

### 11.3 【坑】`-x spotlessKotlinCheck` → `Task not found`

- 现象（#9）：`Task 'spotlessKotlinCheck' not found in root project 'halo' and its subprojects.` → `BUILD FAILED`
- 根因：**Gradle 对 `-x` 一个不存在的任务会直接报错**（不是忽略）。Halo 是纯 Java（无 Kotlin）
- **halo 实际存在的 spotless 任务**（从日志 grep 确认）：`spotlessJavaCheck` / `spotlessMiscCheck` / `spotlessFlexmarkCheck` / `spotlessJsonCheck` / `spotlessSqlCheck` / `spotlessWebResourcesCheck`
- **解决**：只排除真实存在的 → `-x spotlessCheck -x spotlessJavaCheck -x spotlessMiscCheck`
- 为什么不跑 spotless：`:api:spotlessJavaCheck` 报 `NoClassDefFoundError: com/google/common/collect/ImmutableRangeMap$Builder`（**palantir-java-format 依赖的 Guava 传递依赖在国内 Maven 镜像缺失**），CI 阶段先跳过，格式检查留给 pre-commit

### 11.4 【坑】Gradle wrapper 下载超时（services.gradle.org 被墙）

- 现象（#12）：`Downloading https://services.gradle.org/distributions/gradle-9.7.0-bin.zip` → `Attempt 1/1 failed ... timeout (10000ms)` / `java.net.SocketTimeoutException`
- 根因：Gradle 发行包源在墙外，家宽/国内网络抖动随机失败
- **解决**：改 `gradle/wrapper/gradle-wrapper.properties`：
```properties
distributionUrl=https\://mirrors.cloud.tencent.com/gradle/gradle-9.7.0-bin.zip
networkTimeout=60000
retries=3
retryBackOffMs=1000
```
- **辨析**：`#11 能过、#12 挂` 是缓存问题——首次下载成功后缓存在 `.gradle/wrapper/dists/`，一旦缓存失效就重新下载撞超时。**换国内镜像源是一劳永逸**（跟 sherpa-lite 的 maven 镜像同款思路）

### 11.5 【核心坑】Docker Hub 被墙 → `docker build` 拉基础镜像超时

- 现象（#13）：
```
Step 1/17 : FROM eclipse-temurin:21-jre AS builder
Get "https://registry-1.docker.io/v2/": net/http: request canceled while waiting for connection (Client.Timeout exceeded while awaiting headers)
```
- **根因**：仓库自带 Dockerfile 里 `FROM eclipse-temurin:21-jre` 走 **Docker Hub**，境内不可达
- **关键辨析**：`withDockerRegistry` **只管 push 凭证，不影响 FROM 拉基础镜像**——FROM 是 docker daemon 直连 registry-1.docker.io，绕不开
- **解决**：改仓库 `Dockerfile`（两处 FROM）用 daocloud 镜像：
```dockerfile
FROM docker.m.daocloud.io/library/eclipse-temurin:21-jre AS builder
...
FROM docker.m.daocloud.io/library/eclipse-temurin:21-jre
```
- 同时修好另外两个挡路问题：
  1. **`.dockerignore` 加 `.gradle` 等**（原文件只排了 `ui/.github/.git`）——否则 `docker build` 扫描 context 撞到 `.gradle/configuration-cache/xxx` 无法 stat，直接 `checking context: can't stat ...` 失败
  2. **`--build-arg JAR_FILE=application/build/libs/halo-*.jar` 显式传**——Dockerfile 里 `ARG JAR_FILE=/application/build/libs/*.jar` 是绝对路径默认值，构建时要显式覆盖

### 11.6 最终状态（#14 全绿，运行镜像已出）

| Stage | 结果 |
|---|---|
| 环境预处理 (git safe.directory) | ✅ |
| 构建 + 单测（`gradlew clean check`）| ✅ 4m39s |
| 打包 JAR（`halo-2.27.0-SNAPSHOT.jar` 106M）| ✅ 1m53s |
| 打镜像 + 推 ACR（17 步 Dockerfile）| ✅ |

**产出镜像**：`registry.cn-hangzhou.aliyuncs.com/example-org/halo:latest`（同时打 commit SHA tag），digest `sha256:93d49d6b...`

### 11.7 流水线骨架（已定稿，可复用）

```groovy
// 环境预处理 → 构建+单测 → 打包JAR → 打镜像+推ACR
stage('构建 + 单测') {
  agent { label 'docker-linux' }
  steps {
    withDockerRegistry(credentialsId: 'acr-cred', url: 'https://registry.cn-hangzhou.aliyuncs.com') {
      sh 'docker pull registry.cn-hangzhou.aliyuncs.com/example-org/halo-build:latest'
      withDockerContainer(args: '-u 0:0 -v /var/run/docker.sock:/var/run/docker.sock',
                          image: 'registry.cn-hangzhou.aliyuncs.com/example-org/halo-build:latest') {
        sh '''
          ./gradlew --no-daemon -I /root/.gradle/init.gradle clean check \\
            -x spotlessCheck -x spotlessJavaCheck -x spotlessMiscCheck \\
            --configuration-cache --configuration-cache-problems=warn
        '''
      }
    }
  }
}
```

**打镜像 stage 骨架**：
```groovy
stage('打镜像 + 推 ACR') {
  agent { label 'docker-linux' }
  steps {
    withDockerRegistry(credentialsId: 'acr-cred', url: 'https://registry.cn-hangzhou.aliyuncs.com') {
      sh '''
        set -ex
        TAG=$(git rev-parse --short HEAD)
        IMG="${ACR_REGISTRY}/${ACR_NAMESPACE}/${IMAGE_NAME}"
        docker build --build-arg JAR_FILE=application/build/libs/halo-*.jar -t "${IMG}:${TAG}" -t "${IMG}:latest" .
        docker push "${IMG}:${TAG}"
        docker push "${IMG}:latest"
      '''
    }
  }
}
```

### 11.8 遗留技术债
- [ ] **spotless 依赖根因**：palantir-java-format 的 Guava `ImmutableRangeMap$Builder` 在国内 Maven 镜像缺失 → 可选：补 `repo.spring.io` 等源，或 CI 永跳
- [ ] **打镜像 stage 可用 `when { expression { env.PUSH_IMAGE == '1' } }` 条件化**（编译验证时跳过，省时间）
- [ ] **Jenkins Prometheus 插件**：本期已装 `prometheus 860.v532442b_44e9a_`，端点 `/prometheus/`（**注意末尾斜杠**，不带会 302），匿名 HTTP 200，301 指标；scrape 见 §12

---

## 十二、Jenkins 构建指标接入 Prometheus（可观测性 P0 · 2026-09-22）

> 目标：把 Jenkins 构建指标（成功率/耗时/队列/节点状态）接进 kps 的 Prometheus，Grafana 出看板。

### 12.1 插件安装与端点

- 插件：**Prometheus metrics plugin**（ID `prometheus`，装好版本 `860.v532442b_44e9a_`）
- 开启：`Manage Jenkins → System → Prometheus → Expose Prometheus metrics`
- 端点：`http://203.0.113.12:8080/prometheus/`
  - **坑**：不带末尾斜杠访问 `/prometheus` 会 **302 跳 `/prometheus/`**；Prometheus `metrics_path` 必须写 `/prometheus/`
  - 匿名访问直接 **HTTP 200**（无需额外放权），共 **301 个** `default_jenkins_*` 指标

### 12.2 关键指标

```promql
up{job="jenkins"}                                                   # 存活
default_jenkins_uptime
default_jenkins_nodes_online{node="intranet-agent-01"}               # 节点在线（掉线可告警）
default_jenkins_builds_success_build_count{jenkins_job="halo/main"}   # 成功数
default_jenkins_builds_failed_build_count{jenkins_job="halo/main"}    # 失败数
default_jenkins_builds_duration_milliseconds_summary_count           # 构建次数
default_jenkins_builds_duration_milliseconds_summary_sum             # 构建总耗时(ms)
default_jenkins_queue_size_value                                      # 队列积压
default_jenkins_executors_available                                   # 可用执行器
```

**成功率 / 平均耗时（用表达式拼）**：
```promql
# 成功率
sum(default_jenkins_builds_success_build_count)
/ sum(default_jenkins_builds_success_build_count + default_jenkins_builds_failed_build_count) * 100
# 平均耗时(ms)
default_jenkins_builds_duration_milliseconds_summary_sum / default_jenkins_builds_duration_milliseconds_summary_count
```

### 12.3 接入 Prometheus（kps 用 `ScrapeConfig` CRD）

**现状**：监控是 kube-prometheus-stack（Operator），Prometheus CR `kps-prometheus`，`scrapeConfigSelector.matchLabels: {release: kps}`。**Jenkins 在集群外**（腾讯云 203.0.113.12），ServiceMonitor 抓不到 → 用 **`ScrapeConfig` CR + staticConfigs**。

```yaml
apiVersion: monitoring.coreos.com/v1alpha1
kind: ScrapeConfig
metadata:
  name: jenkins
  namespace: monitoring
  labels:
    release: kps            # 必须匹配 Prometheus CR 的 selector，否则不被采纳
spec:
  metricsPath: /prometheus/  # 末尾斜杠！
  scheme: HTTP
  staticConfigs:
    - targets: ['203.0.113.12:8080']
      labels: { job: jenkins, env: external }
  relabelings:
    - sourceLabels: [__address__]
      targetLabel: instance
      replacement: jenkins-main
```

- `kubectl apply` 后 Operator 自动生成 `job_name: scrapeConfig/monitoring/jenkins`
- **验证**：`kubectl port-forward -n monitoring svc/kps-prometheus 19090:9090` → `curl 'http://127.0.0.1:19090/api/v1/targets'`，找 `job=jenkins` → **health: up**
- **坑**：target 刚加时 `health: unknown`（还没到第一轮 30s 抓取周期），**等一轮即 up**；Prometheus Pod 在 home-node 节点，经公网抓腾讯云 Jenkins 正常

### 12.4 现状
- ✅ Prometheus / Grafana / Alertmanager / Loki+Promtail / **Jenkins 指标** 全部就位（可观测性 P0 收尾）
- ✅ **Grafana 构建看板**（2026-09-27 建：uid `jenkins-ci-overview`，「Jenkins CI 效能看板」14 面板——概览 / 各 Job 现状 / 趋势；数据走 `default_jenkins_builds_*` 全套）
- ✅ **告警出口**（2026-09-27：Alertmanager → 上海转发器 → 企微群，含核心告警规则，详见 `07-Alertmanager告警出口接入实战.md`）
- 顺带发现 `default_jenkins_nodes_online{node="win-agent-01"}=0`（**Windows agent 离线**）

### 12.5 构建看板面板构成（2026-09-27）

| 区块 | 面板 |
|---|---|
| ① 概览 | 任务数 / 健康任务(≥50分) / 执行器忙·总 / 队列长度 / 插件失败数 / Jenkins 状态 |
| ② 各 Job 现状 | 健康分条形图（红黄绿阈值）+ 明细表（Job/最近结果/平均耗时/成功/失败/总构建） |
| ③ 趋势 | 累计构建数 / 平均构建耗时 / 累计成功率（每 job 一条线） |

> 建法：Grafana Dashboard API（`/api/dashboards/db`）导入 JSON，不动 Jenkins 任何配置。
> ⚠️ 本环境未装 image renderer 插件 → `/render` 端点出不了图，截图靠 headless Chromium 登录抓。

**Jenkins 任务健康分算法**（实测反推，用于 `JenkinsJobUnhealthy` 告警阈值）：最近 ≤5 次构建平均分，`SUCCESS=100 / FAILURE=0 / ABORTED=0`；不足 5 次按实际条数平均。→ 一次失败 -20 分。

---

## 十三、Jenkins Shared Library 落地（2026-09-27）

> 目标：把散在 4 套 Jenkinsfile 里的重复逻辑收敛成**唯一逻辑源**，各仓库 Jenkinsfile 保持"瘦"，差异走配置——对齐车载项目「仓库声明 → pipeline.yml 配置 → Shared Library 逻辑源」三层结构。

### 13.1 现状分析（为什么做）

| 仓库 | Jenkinsfile 行数 | 技术栈 |
|:---|:---|:---|
| hello-jenkins | 342 | C/C++ 嵌入式（Windows+Linux 双腿） |
| sherpa-lite | 453 | Android NDK（C++ + Kotlin） |
| halo | 127 | Java/Gradle + Docker |
| UDS_* ×4 | — | C 嵌入式（复用 hello 模板） |

**重复度**（同一段逻辑各写一遍）：`options{timestamps/timeout/buildDiscarder}` 3 处 · `agent{docker{...acr-cred}}` 3+ · `withDockerRegistry` 3+（halo 内 4 次）· ACR 构建镜像+推 2 · Versioning 2 · Codecheck(cppcheck+MISRA) 2 · Coverage 2 · `post{cleanup{docker prune}}` 3。
→ **每套约 40~50% 可标准化**，改一处要改 N 处。

### 13.2 库设计

```
jenkins-shared-library/       ← Gitea 私有库
├── vars/          # 全局步骤（Jenkinsfile 直接按名字调用）
├── src/           # Groovy 类（复杂逻辑、可单测）
├── resources/     # 非 Groovy 资源（模板、脚本）
└── README.md
```

目标形态（后续）：`@Library('ci-lib') _` + `standardPipeline(stack:'c-embedded', pipelineYml:'pipeline.yml')`

### 13.3 落地步骤

**① 建库（Gitea API）**
```bash
curl -u '<user>:<pass>' -X POST "http://203.0.113.13:30080/api/v1/user/repos" \
  -H "Content-Type: application/json" \
  -d '{"name":"jenkins-shared-library","private":true,"default_branch":"main"}'
```

**② 首步骤 `vars/ciInfo.groovy`**（冒烟：打印库版本 + 运行节点/Job）；配套 `src/com/zwg/ci/LibInfo.groovy`（版本 + 技术栈常量）。

**③ Jenkins 挂载（⭐ 关键坑）**

> ⚠️ **全局库配置不在 `config.xml`！** `GlobalLibraries` 是 `GlobalConfiguration`，持久化在独立文件
> `$JENKINS_HOME/org.jenkinsci.plugins.workflow.libs.GlobalLibraries.xml`。
> 往 `config.xml` 写 `<globalLibraries>` → **Jenkins 重启时静默丢弃、不留任何日志**。

**可靠配法：脚本控制台（Groovy）构造 + `save()`**：
```groovy
// Jenkins 脚本控制台 /scriptText（需 crumb + session cookie）
import org.jenkinsci.plugins.workflow.libs.*
import hudson.plugins.git.*
def scm = new GitSCM(
    [new UserRemoteConfig('<lib-git-url>', null, null, 'gitea-cred')],
    [new BranchSpec('*/main')], false, [], null, null, [])
def lc = new LibraryConfiguration('ci-lib', new SCMRetriever(scm))
lc.setDefaultVersion('main'); lc.setImplicit(false); lc.setAllowVersionOverride(true)
def gl = GlobalLibraries.get(); gl.setLibraries([lc]); gl.save()
```

> 插件：新版是 **`pipeline-groovy-lib`**（Long-Name *Pipeline: Groovy Libraries*）；`workflow-cps-global-lib` 是旧名/弃用版，别装它。本环境早已装好，无需额外安装。

**④ 冒烟验证**：建 job `ci-lib-smoke`（`@Library('ci-lib') _` + `ciInfo()`）→ 构建 **SUCCESS**，日志见自动 clone 库到 `@libs/<hash>`：
```
║  Jenkins Shared Library 已加载（冒烟验证）
║  库 : jenkins-shared-library @ 0.1.0
║  节点 : intranet-agent-01
```

### 13.4 关键坑与认知

1. **配置位置**：全局库配置**不在 `config.xml`**，在独立的 `GlobalLibraries.xml`。写错位置**静默丢弃、无日志**——排查全靠文件对比（备份里曾写入、当前文件没有）。
2. **插件名**：`pipeline-groovy-lib`（新）/ `workflow-cps-global-lib`（旧·弃用）。
3. **配置手法**：手写 XML 极易被静默拒 → 优先脚本控制台构造后 `save()`，让 Jenkins 自己序列化。
4. **认证细节**：`/scriptText` 要 `crumb` + **同 session cookie**（`curl -c/-b`），否则 403；用的是 Jenkins 账号密码（非云主机密码）。
5. **验证要看能力**：`@Library` 加载成功 ≠ 库能用；建最小 job 实跑一次才算真通。

### 13.5 后续路线

| 步 | 内容 | 状态 |
|:--:|:---|:--:|
| 1 | 建库 + 挂载 + 冒烟（`ciInfo`） | ✅ 2026-09-27 |
| 2 | 抽 `acrImage`（ACR 推镜+VPC 回落）/ `versioning` + halo 真身改造 | ✅ 2026-09-28 |
| 3 | 抽 `codeCheck` / `coverage` + `ci-lib-steps-test` 验证 | ✅ 2026-09-28 |
| 4 | `standardPipeline` 统一骨架（options/agent/post） | ⬜ |
| 5 | 各仓库改造为「瘦 Jenkinsfile + pipeline.yml」 | ⬜ |

> 首个真实改造对象：**halo**（116 行，最简）。

---

## 十四、Shared Library 第 2 步：`versioning` + `acrImage` 抽象 & halo 真身改造（2026-09-28）

> 目标：把 `versioning` / `acrImage` 两个步骤补到「能覆盖真实仓库」的程度，然后拿 **halo** 做首个真身改造，验证库可用。

### 14.1 实现流程

**① 侦察现状**

- halo 原始 `Jenkinsfile` 116 行，**ACR 登录 / 推镜 / 版本号提取全手写内联**。
- 发现昨晚（09-27 18:22）已建 `ci-lib-steps-test` job 专门测这两步，翻它的构建结果：
  - `versioning` **完全成功**（正确写出 `ci_version.h`）；
  - 失败点仅 `acrImage`（build-only）：测试 Dockerfile `FROM debian:12-slim` 拉不到 → **`registry-1.docker.io` 连接超时**（agent 到 Docker Hub 不通），**与库代码无关**。
- halo 的 `Dockerfile` / `Dockerfile.build` 基础镜像全部走 **ACR 私库**（`registry.cn-hangzhou.aliyuncs.com/example-org/eclipse-temurin`）→ 天然绕开 Docker Hub 墙 → **halo 才是验证 `acrImage` 的正确靶子**。

**② 定位 halo 与库的差异（决定增强点）**

| 差异 | 影响 | 增强 |
|:---|:---|:---|
| 版本号来自 `gradle.properties`（2.27.0-SNAPSHOT），**不是** `git describe` | `versioning` 原来只认 git tag | 加 `versionFile` + `versionPattern`（默认 gradle 正则，自动去 `-SNAPSHOT`） |
| 运行镜像需 `--build-arg JAR_FILE=application/build/libs/halo-*.jar` | `acrImage` 原来不支持构建参数 | 加 `buildArgs`（透传，支持 glob） |
| 想要语义 tag + 唯一 tag 双打 | 原来只有 commit-SHA | 加 `tagTemplate`（`sha`/`v`/自定义 `{sha}`） |

**③ 改库（v0.2.0）→ 推 Gitea `main`（`4a7a72d`）**

- `vars/versioning.groovy`：版本来源优先级 `versionOverride` > `versionFile`+`versionPattern` > 兜底 `git describe`。
- `vars/acrImage.groovy`：新增 `buildArgs` / `tagTemplate` / `pull`（预拉基础镜像避免 FROM 超时），保留 VPC 端点探测 + 内网回落。
- `src/com/zwg/ci/LibInfo.groovy`：版本 `0.1.0` → `0.2.0`。

**④ 改 halo `Jenkinsfile` → 推 `main`（`f1e472e`）**

- 顶部 `@Library('ci-lib') _`；
- 版本号：`versioning(versionFile: 'gradle.properties')` → `env.VERSION`；
- 两处「docker build + push」→ `acrImage(...)`（编译镜像；运行镜像带 `buildArgs: ['JAR_FILE=...']` + `tagTemplate: 'v'`）；
- **116 行内联 → 声明式瘦身**，逻辑收敛到库。

**⑤ 触发构建**（推 main 未自动触发 webhook，见 14.2）→ **build #32**。

### 14.2 遇到的问题

1. **【坑】推送 main 后 webhook 没自动触发** —— 无新 build、队列也空。
   → 手动触发：`crumb` + `POST /job/halo/job/main/build`（`201`）。
2. **【预期内】VPC 端点探测不可达** —— Jenkins 跑在腾讯云本机（非杭州节点），`registry-vpc` 端点探测非 401 → 回落公网 push。日志：`ACR: VPC 不可达（非杭州节点?），回落公网 push（慢）`，符合设计，未报错，耗时约 47s。
3. **【耗时】归档 + 指纹拖长构建** —— 产物 `halo-2.27.0-SNAPSHOT.jar` 106M，`archiveArtifacts` 指纹化约 2.5min；总耗时 15m57s。

### 14.3 验证结果（build #32 全绿）

| 阶段 | 结果 | 证据 |
|:---|:--:|:---|
| 环境预处理 + 版本 | ✅ | `versioning: version=2.27.0 branch=main`；显示名 `#32 [main] 2.27.0` |
| 构建编译镜像 + 推 | ✅ | `acrImage` 构 `halo-build`，tags `[2.27.0, f1e472e]`，已推送 |
| 编译 + 单测 + check | ✅ | `BUILD SUCCESSFUL in 10m 58s`；产物 `halo-2.27.0-SNAPSHOT.jar` 106M |
| 打运行镜像 + 推 | ✅ | `docker build --build-arg JAR_FILE=application/build/libs/halo-*.jar`；tags `[2.27.0, vf1e472e]`，两 tag 均推送成功 |

最终 `result: SUCCESS`，`duration 956844ms`（≈15m57s）。

### 14.4 总结

- **库侧 v0.2.0**：`versioning` 支持文件取版本；`acrImage` 支持 `buildArgs` / `tagTemplate` / `pull`。
- **首个真身改造完成**：halo Jenkinsfile 从「内联 116 行」变为「声明式 + 库调用」，改造后 32 insertions / 48 deletions。
- **能力都在真身跑通**：`buildArgs`（glob `JAR_FILE=`）、`tagTemplate: 'v'`、VPC 回落探测 —— 不再是单元测试级别的验证。
- **认知**：`ci-lib-steps-test` 的失败是「环境问题」（Docker Hub 不通），**不是库问题**；选验证靶子要对齐基础镜像来源（ACR 私库仓库 = 天然免疫）。
- **下一步（第 3 步）**：抽 `codeCheck` / `coverage`。

---

## 十五、Shared Library 第 3 步：`codeCheck` + `coverage` 抽象（2026-09-28）

> 目标：把 C/JVM 两个技术栈里长得一样的「静态检查」与「覆盖率」样板收敛到库，对齐 hello-jenkins 与 sherpa-lite。

### 15.1 抽象前的差异分析

| 维度 | hello-jenkins | sherpa-lite | 收敛点 |
|:---|:---|:---|:---|
| codecheck 工具 | cppcheck（C99）+ MISRA（cppcheck addon） | cppcheck（C++17）+ ktlint（Kotlin）+ Android Lint | 都是「跑工具 → 出 XML → 计数 → recordIssues → 归档」 |
| 门禁 | 无（仅出报告） | **有**（问题数 > 阈值 → exit 1 阻断） | `maxErrors` 可选启用 |
| 报告解析器 | `cppCheck` | `cppCheck` / `checkStyle` / `androidLint` | `parser` 参数映射 |
| coverage | `make coverage` → publishHTML | gradle jacoco + UT → publishHTML | 都是「跑生成命令 → publishHTML → 归档」 |
| UT 结果 | xunit CUnit | xunit JUnit | `junit` 可选参数 |

### 15.2 库设计（v0.3.0）

- **`vars/codeCheck.groovy`**：参数 `id`/`name`/`run`/`report`/`parser`/`countPattern`/`maxErrors`/`archive`。
  顺序：**run → recordIssues（先记录，保证门禁失败时趋势仍在）→ 计数门禁 → 归档**。工具调用本身由调用方传入（`run`），库只收敛「样板」。
- **`vars/coverage.groovy`**：参数 `run`/`reportDir`/`reportFiles`/`reportName`/`junit`/`artifacts`/`allowMissing`。
  顺序：**run → publishHTML → 可选 xunit(JUnit) → 归档**。统一用 publishHTML（沿用两仓库既有约定，不用 recordCoverage）。

### 15.3 遇到的问题（重要）

**【坑】Groovy 块注释被 `**/` 提前闭合**

- 现象：`coverage()` 调用处报 `CpsCompilationErrorsException: unexpected token: *`（指向库文件 `coverage.groovy:18`）；stage 能起容器但脚本编译失败、整条构建红。
- 根因：`coverage.groovy` 的 Javadoc 示例里写了 `junit: "...test-results/**/*.xml"` —— 其中 **`**/`（`*` + `/`）被 Groovy 当成块注释结束符**，导致 `/** ... */` 在示例行提前闭合，后面参数说明文字被当作代码解析。
- 修复：将示例里的 `**/*.xml` 转义为 `**\/*.xml`（`codeCheck.groovy` 当初就正确转义了，所以只有 coverage 中招）。
- **教训**：共享库 `.groovy` 的文档注释里，任何 `*/` 序列（包括 glob 的 `**/`）都会被当作注释结束 —— 写通配例子必须转义。

### 15.4 验证（`ci-lib-steps-test` #3 全绿）

把 `ci-lib-steps-test` job（原本测 versioning+acrImage）扩展出两个 stage（钉在 hello-jenkins 真身上）：

| Stage | 结果 | 证据 |
|:---|:--:|:---|
| versioning | ✅ | `version=d8a165e branch=main` |
| codeCheck (cppcheck) | ✅ | `报告摘要：0 条` → `codeCheck OK: id=cppcheck count=0` |
| coverage (make coverage) | ✅ | `=== Coverage report: build/coverage/html/index.html ===` → `coverage OK` |

### 15.5 总结

- 库v0.3.1（`a688784`）：新增 `codeCheck` / `coverage` 两个通用包装步骤，共 5 个步骤（`ciInfo`/`versioning`/`acrImage`/`codeCheck`/`coverage`）。
- **关键认知**：库注释里的 glob 示例必须转义 `*/`；失败先排除环境（Docker Hub）再怀疑库代码（同第 2 步的教训延续）。
- **下一步（第 4 步）**：`standardPipeline` 统一骨架（options/agent/post）。

---

## 十六、补 codecheck：B 类小仓接入（2026-09-28）

> 背景：扫描全部 9 个仓后，三个「单 `编译` stage」的一开始就没有静态检查 —— `UDS_S32K144_APP` / `UDS_S32K144_FlashDriver` / `UDS_SecurityAccess`。本次给它们补上 codecheck，并顺手接入 `ci-lib`。

### 16.1 侦察结论（决定可行性）

- 三仓都用 `registry.cn-hangzhou.aliyuncs.com/example-org/s32k144-toolchain:v1/v2`；该镜像 Dockerfile（`/tmp/Dockerfile.toolchain`）**已装 `cppcheck`**（还有 lcov/gcovr）→ 不用改镜像即可补检查。
- APP / FlashDriver 的 Makefile 与 `UDS_S32K144_Bootloader` 同模板（宏 `UDS_PROJECT_FOR_APP` / `_FLASHDRIVER`）。
- SecurityAccess 是 C++（g++，含 `Algorithm.cpp`），用 `--language=c++`。

### 16.2 改造内容

| 仓 | 扫描范围 | 关键参数 |
|:---|:---|:---|
| UDS_S32K144_APP | `Sources UDS_DriverConfig UDS_PortingFiles UDS_ProtocolStack` | `-D UDS_PROJECT_FOR_APP` |
| UDS_S32K144_FlashDriver | `Sources` | `-D UDS_PROJECT_FOR_FLASHDRIVER` |
| UDS_SecurityAccess | `UDS_SecurityAccess`（C++） | `--std=c++11` `-D UDSECURITYACCESS_EXPORTS` |

写法：新增 `codecheck` stage（toolchain 镜像内），报告走 `ci-lib` 的 `codeCheck()`（`recordIssues` + 归档），编译 stage 保持原样。

### 16.3 验证（三仓 build #2 全绿）

| 仓 | 结果 | cppcheck 命中 | commit |
|:---|:--:|:--:|:---|
| UDS_S32K144_APP | ✅ SUCCESS | 3 条 | `6dd510e` |
| UDS_S32K144_FlashDriver | ✅ SUCCESS | 3 条 | `d7349e7` |
| UDS_SecurityAccess | ✅ SUCCESS | 0 条 | `86d76e3` |

3 条均 < 阈值（未启用门禁，仅出报告 + 趋势）。

### 16.4 当前落地矩阵（9 仓）

| 仓 | 已有 codecheck | 已接入 ci-lib |
|:---|:--:|:--:|
| halo | ✅(编译内 check) | ✅ |
| UDS_S32K144_APP | ✅(本次补) | ✅ |
| UDS_S32K144_FlashDriver | ✅(本次补) | ✅ |
| UDS_SecurityAccess | ✅(本次补) | ✅ |
| hello-jenkins | ✅ | ❌ |
| UDS_S32K144_Bootloader | ✅ | ❌ |
| sherpa-lite | ✅ | ❌ |
| aiops-log-doctor | ❌(无 Jenkinsfile) | ❌ |

- **下一步**：剩余 3 个有 codecheck 的仓（hello-jenkins / Bootloader / sherpa-lite）的真身改造（抽到 `codeCheck`/`coverage`），随后第 4 步 `standardPipeline`。

---

## 十七、真身改造：3 个「有 codecheck 未接库」的仓（2026-09-28）

> 衔接 §十六 的下一步：把 `hello-jenkins` / `UDS_S32K144_Bootloader` / `sherpa-lite` 三仓改为 `@Library('ci-lib') _`，重复逻辑收敛到库。

| 仓 | 规模 | Jenkinsfile 特点 | commit | 验证 |
|:---|:---|:---|:---|:---|
| hello-jenkins | 350 行 | 单 job（非 Multibranch） | `b764f15` | **#193 SUCCESS** |
| UDS_S32K144_Bootloader | 350 行 | 与 hello-jenkins 同模板 | `f8b5faa` | **#4 SUCCESS** |
| sherpa-lite | 546→422 行 | 最复杂（Android/ASR） | `02cc4c47` | **#63 SUCCESS**（见 §十七.3） |

### 17.1 hello-jenkins / Bootloader（同模板）

- `Versioning` → `versioning(headerFile:'Sources/ci_version.h')`
- `Coverage` → `coverage(run:'make coverage', reportDir:'build/coverage/html', allowMissing:false)`
- codecheck 分支的**双 stage（Cppcheck + MISRA）合并为单 stage**，两处调 `codeCheck(...)`（MISRA 用 `if (env.RUN_MISRA != 'false')` 包住）
- Build&Push / Build / Size Check / Unit Test / windows 腿 原样保留
- Bootloader 与 hello-jenkins 差异仅镜像 repo/tag + VPC 路径

### 17.2 sherpa-lite（最复杂）

- UT+覆盖率 → `coverage(run:..., junit:'...testDebugUnitTest/*.xml', reportDir:'...jacoco/jacocoTestReport/html', ...)`
- 静态检查 3 路（cppcheck/ktlint/Android Lint）→ `codeCheck(...)`，各传 `maxErrors`（门禁）
- **为 sherpa 给 `codeCheck` 加 `record` 开关**（默认 true）：Android Lint 分支因 Jenkins 未装 `androidLint` parser，传 `record:false`（只门禁+归档，不 `recordIssues`）→ 库升 **v0.3.2**（commit `466cc7f`）
- **注意**：Jenkinsfile 的 Groovy 三引号 block 里，glob `**/*` 要转义为 `**\/*`（同 §十五 坑）；shell 变量 `${WORKSPACE}` 在 Groovy 里要写 `\${WORKSPACE}`

### 17.3 验证卡点与最终结果

- 期间撞上 **home-node 家宽节点掉线**（见 §十八），#58~#62 全部卡在 checkout 连不上 Gitea —— **非改造问题**；home-node 恢复后重跑即 **#63 SUCCESS**（~7.5min）：
  - 阶段全绿：编译 .so → 打 APK(Kotlin) → 打 APK(Java) → UT+覆盖率 → 3 路静态检查
  - 门禁：ktlint 2738≤3000 ✅ / cppcheck 44≤100 ✅ / AndroidLint 0≤500 ✅

### 17.4 落地矩阵更新（9 仓）

| 仓 | 已接入 ci-lib |
|:---|:--:|
| halo | ✅ |
| hello-jenkins | ✅（本次） |
| UDS_S32K144_Bootloader | ✅（本次） |
| sherpa-lite | ✅（本次） |
| UDS_S32K144_APP | ✅ |
| UDS_S32K144_FlashDriver | ✅ |
| UDS_SecurityAccess | ✅ |
| aiops-log-doctor | ❌（无 Jenkinsfile） |

> 至此「有 Jenkinsfile 的 8 仓」**全部接入 ci-lib**，只剩无 Jenkinsfile 的 aiops-log-doctor。**下一步：第 4 步 `standardPipeline` 统一骨架。**

---

## 十八、sherpa-lite：模型打包进镜像 + 镜像构建纳入 Jenkins（2026-09-28）

### 18.1 需求①：ASR 模型预置进 APK 镜像

- **动机**：模型 `sherpa-onnx-streaming-zipformer-bilingual-zh-en-2023-02-20`（~511MB）每次构建联网下载，慢且易失败。
- **手法**：基于现有 APK 镜像 `sherpa-build-android-apk:20260916` overlay 加模型层（`/tmp/apkan/Dockerfile.model`，`COPY assets/ → /opt/models/`）→ 得到 `20260928`（1.81GB，digest `sha256:f55b12a3...`），已 push ACR ✅
- **repo 同步**（commit `d3be241f`）：
  - `Dockerfile.build-apk` 加第 5 步（构建时经 `ghfast.top` 下模型固定到 `/opt/models/<model>`）
  - Jenkinsfile `IMG_APK` tag → `20260928`；两处 APK stage 改为**优先 cp 镜像内置 `/opt/models/`，联网下载仅兜底**

### 18.2 需求②：镜像构建也放进 Jenkins

- 在 sherpa-lite Jenkinsfile 加「构建镜像」stage，带参数开关 `params.BUILD_IMAGES`（默认 `false`），复用 ci-lib 的 `acrImage()`
- **坑**：Jenkinsfile 因加参数 → 触发需用 `buildWithParameters`（不是 `build`，否则 400）
- commit `598491eb`；语法校验通过（`PARSE_OK`）

### 18.3 验证与坑

- **卡点（非改造问题）**：#58/#59 卡 Gradle 联网拉 `build-tools 30.0.3` / `maven.aliyun.com` DNS 失败；#60/#61 checkout 拉 Gitea 超时（`Failed to connect to 203.0.113.13:30080 after 130825 ms`）——**根因都是 home-node 掉线**（见 §十八.4）
- **#62**：checkout 已通，构建进行中；home-node 恢复后重跑 → **#63 SUCCESS**
  - 模型：`模型已存在，跳过`（用镜像内置 `/opt/models`，不再联网下载）✅ 改造生效
  - 「构建镜像」stage 按预期跳过（`BUILD_IMAGES=false`）✅

### 18.4 关联事故：home-node 掉线导致「监控+构建」双失能

- **现象**：13:49 收到 `TargetDown` warning（`jenkins` target down），实为**监控栈失能误报**（Jenkins 本机健康、8080 端点 200）
- **根因链**：
  1. `home-node`（家宽 WG `10.88.0.2`）14:02 起掉线，kubelet `Ready=Unknown`，WG 握手停摆
  2. **整个 kps 监控栈被 `nodeSelector: kubernetes.io/hostname: home-node` 钉死在 home-node**（prometheus/alertmanager/loki/grafana/operator 无一例外）→ 节点一死监控当场失能
  3. Prometheus 停抓 → `up{job="jenkins"}==0` → `TargetDown`（for 10m）
  4. **构建同源**：Jenkins k8s agent 的 PodTemplate 也 `nodeSelector: home-node`（在 `~/jenkins_home/config.xml` 的 k3s cloud PodTemplate 里）→ 所有动态 agent 只能落 home-node，静态 agent `intranet-agent-01`（`203.0.113.21` = home-node 家宽出口 IP）也在 home-node → #58~#62 连不上 Gitea / Gradle DNS 失败，**与改造无关**
- **架构缺陷**：监控栈 + 构建 agent **双重重压同一条家宽 WG 节点** = 它一掉既不监控、又发不出告警、还构建不了（此次告警侥幸经上海 `10.88.0.3:9099` relay 投出）
- **待办**（用户未选）：① 把 Prom/AM/Loki/Grafana 迁到杭州 control-plane 或上海节点（治本）；② agent 模板去掉 home-node 硬绑定；③ `TargetDown` 加监控节点自挂注释

### 18.5 经验

- **磁盘**：本轮期间本机曾 93%（仅 4G 空闲），清旧镜像 tag 后 ~92%（4.5G）；后续又清 `disk-alert-sim.bin` 6G + 日志（见 §十九）
- **删除本地 clone/工作副本前**，必须核对 `git merge-base --is-ancestor <HEAD> origin/main` 或比对远端 commit，防丢未推提交
