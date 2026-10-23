---
{"dg-publish":true,"permalink":"/knowledge/devops/practices/01-Jenkins部署实战/","dg-note-properties":{}}
---


# Jenkins 部署实战：从装 Master 到三种 Agent 接入的完整主线

---

> 📌 **主线导览** —— 本文档以"把一个应用的构建跑起来"为实施主线推进：
>
> **阶段 1 · 部署 Master**：① Docker Compose 拉起 Master → 环境依赖 → 组件/插件 → 部署期踩坑
> **阶段 2 · 接入 Agent（三种渠道）**：① 内网 Linux（JNLP+WS）→ ② Windows → ③ K8s 动态 agent（k3s）。三者可并行，均为"Master 之外谁能跑构建"。
> **运维实录**：上线后遇到的真实故障与修复（k8s agent clone CodeUp DNS 排障）。
> 各阶段"遇到的问题/坑"就近附在对应环节，方便边做边看。

---


## 阶段 1 · 部署 Master（Docker Compose）

compose 文件：本机 `/root/.openclaw/workspace/docker-compose-jenkins.yml`

```yaml
services:
  jenkins:
    image: jenkins/jenkins:lts
    container_name: jenkins
    restart: unless-stopped
    ports:
      - "8080:8080"     # Web
      - "50000:50000"   # inbound agent 回连端口
    environment:
      - TZ=Asia/Shanghai
    group_add:
      - "992"          # 宿主机 docker 组 GID，配合挂 docker.sock 实现容器内操作宿主 docker
    volumes:
      - /root/jenkins_home:/var/jenkins_home
      - /var/run/docker.sock:/var/run/docker.sock
      - /usr/bin/docker:/usr/bin/docker:ro
    deploy:
      resources:
        limits:
          cpus: "1"
          memory: 1G
```

```bash
cd /root/.openclaw/workspace
docker compose -f docker-compose-jenkins.yml up -d      # 安装/更新
docker exec -it jenkins bash                            # 进容器
cat /root/jenkins_home/secrets/initialAdminPassword     # 初始密码（已改则无效）
```

## 阶段 1 附 · Master 环境依赖

| 依赖 | 用途 | 备注 |
|---|---|---|
| Docker + docker.sock 挂载 | Pipeline 内 build/run 容器 | 需 `group_add` 对齐宿主机 docker 组 GID（本机 992），并只读挂入 docker CLI |
| 端口 8080 / 50000 | Web / agent 回连 | 腾讯云安全组已放行，外网实测可达（`http://203.0.113.12:8080`） |
| 内存限额 1G | Java master | 512M 会 OOM，至少 1G |
| 容器内 Debian 换国内源 | apt 装工具 | deb.debian.org 境内极慢；现源 mirrors.aliyun.com（Debian Trixie 13） |
| 容器内已装工具 | 直接当构建环境 | python3(3.13)/pip3/venv、gcc、g++ |
| Agent 机器 **JDK 21**（当前版本 agent 硬性要求，JDK ≤17 拒跑） | 命令行/WebSocket 方式起 agent | MSI 方式自带 JRE 可免装；实测 2.555.3 + win-agent-01 ✅ |

## 阶段 1 附 · Master 组件与插件清单

- **Master**：`jenkins/jenkins:lts`（2.555.x），启动参数固定 `-Djenkins.model.Jenkins.slaveAgentPort=50000`
- **插件**（现有）：credentials / credentials-binding / plain-credentials / ssh-credentials / **kubernetes**（含 kubernetes-client-api、kubernetes-credentials 依赖，9-4 装毕；`.tmp` 残留已消失即安装完整）
- 注：本版本（2.555.3）**无需单独 inbound-agent 插件**，Launch method 原生含 "Launch agent by connecting it to the controller"，`/jnlpJars/agent.jar` 由 core 提供
- **安全**：全局安全已开（匿名 API 403）；账号 admin / user / noreply

## 阶段 1 附 · 踩坑记录（Master 部署与 CLI 自动化）

| # | 问题 | 现象 | 处理/结论 |
|---|---|---|---|
| 1 | CLI 自动化被凭证卡住 | 匿名 403；无 API Token 无法用 CLI 建节点/装插件 | 待生成 API Token（Manage Jenkins→用户→Configure）；或全部走 UI |
| 2 | 宿主机无 java | `jenkins-cli.jar` 本机跑不了 `java: command not found` | `docker cp` 进容器内执行（容器自带 JRE） |
| 3 | CLI WebSocket 握手失败 | 容器内跑 CLI `whoami` 报 `DeploymentException: Handshake error` | CLI 默认走 WebSocket 协议被拒；改从 UI 操作，或启用 TCP agent 协议后再试 |
| 4 | 容器内查版本报错 | `docker exec jenkins jenkins --version` → executable not found | 镜像内该入口不在 PATH；用 `curl http://localhost:8080/api/json` 看 version 字段 |
| 5 | 容器内存上限 | 初期 512M 偏紧 | 已上调 1G；并发多了再扩 |
| 6 | 首次 agent 连不上 | inbound 连接后停留"等待" | 需管理员在节点页点 **Trust** 批准公钥（新节点必经步骤，不是故障） |
| 7 | agent 要求 JDK 21 | 新版 agent.jar 在 JDK ≤17 上拒绝启动 | Windows 机装 JDK 21 后接入成功（win-agent-01 在线）；MSI 方式自带 JRE 可避开此问题 |

---

## 阶段 2 · 节点接入 ①：内网 Linux 节点 agent-01（JNLP+WebSocket）

**节点档案**：

| 项 | 值 |
|---|---|
| 节点名 | `intranet-agent-01` |
| Labels | `agent-01` |
| 机器 | 内网 Linux（NAT 后，公网出口 203.0.113.20） |
| 连接方式 | **JNLPLauncher（inbound）+ WebSocket** |
| 通信协议版本 | Remoting 3383 |
| 远程目录 | `/home/jenkins/agent` |
| 执行器 | 2 |
| 状态 | 接入时验证在线 ✅；**当前在线状态未核对** |

**为什么选 WebSocket**：agent 在内网 NAT 后，只有到 master 8080 的**出方向**通路，master 无法反连它的 50000——`-webSocket` 让 agent 全程用 8080 单端口双向复用，穿透无需任何打洞配置。

**接入过程**：
1. UI 建节点：Nodes → New Node → `intranet-agent-01`，Launch method 选 `Launch agent by connecting it to the controller`，Labels `agent-01`，remote root `/home/jenkins/agent`，executors 2 → 保存，复制页面 secret
2. agent 机器拉 jar + 以 WebSocket 启动：
```bash
mkdir -p /home/jenkins/agent && cd /home/jenkins/agent
curl -O http://203.0.113.12:8080/jnlpJars/agent.jar
java -jar agent.jar -url http://203.0.113.12:8080/ \
  -name "intranet-agent-01" -secret "<SECRET>" \
  -workDir "/home/jenkins/agent" -webSocket
```
3. 节点页点 **Trust** 批准 → 变绿 ✅
4. 常驻化：systemd 服务（`Restart=always`）包装上面的启动命令（日志 `agent.log`）

**Pipeline 用法**：`agent { label 'agent-01' }`

---

## 阶段 2 · 节点接入 ②：Windows 节点

### ②.1 创建节点（Jenkins 侧）

Manage Jenkins → **Nodes → New Node**

| 配置项 | 值 |
|---|---|
| 节点名称 | `win-agent-01` |
| 类型 | Permanent Agent |
| Labels | `windows`（按实际能力再加，如 `java` `dotnet`） |
| Remote root directory | `C:\Jenkins` |
| Launch method | **Launch agent by connecting it to the controller** |
| Usage | *Only build jobs with label expressions matching this node* |

保存后进节点页 → 记录页面上的 **secret**（方式 B/C 使用）。

### ②.2 Windows 机器先自检

```powershell
Test-NetConnection 203.0.113.12 -Port 8080
Test-NetConnection 203.0.113.12 -Port 50000
java -version          # 方式 B/C 需要；⚠️ 必须是 JDK 21（当前版本 agent 硬性要求）；方式 A 免
```

### ②.3 启动 agent（三选一）

**方式 A · MSI 服务（推荐：开机自启、断线自恢复）**
1. 下载 `http://203.0.113.12:8080/jnlpJars/jenkins-agent.msi`
2. 向导填：master URL `http://203.0.113.12:8080`、Agent name `win-agent-01`、Work dir `C:\Jenkins`、（认证开启时）用户名+API Token
3. 装完自动注册服务 `Jenkins Agent` 并启动
4. 回节点页点 **Trust** → 节点变绿 ✅

**方式 B · 命令行（调试用，需 JDK）**
```powershell
mkdir C:\Jenkins; cd C:\Jenkins
Invoke-WebRequest http://203.0.113.12:8080/jnlpJars/agent.jar -OutFile agent.jar
java -jar agent.jar -url http://203.0.113.12:8080/ -name "win-agent-01" -secret "<节点页SECRET>" -workDir "C:\Jenkins"
```

**方式 C · WebSocket（50000 被封时，仅用 8080；同 agent-01 场景）**
方式 B 命令末尾追加 `-webSocket`。

> 建议：B 跑通看到 "Connected" 后，再切 A 做常驻。

### ②.4 验证

```groovy
pipeline {
  agent { label 'windows' }
  steps { bat 'echo agent OK & ver' }
}
```

### ②.5 通用避坑

| 坑 | 对策 |
|---|---|
| 节点一直离线/等待 | 节点页日志 + `Test-NetConnection` 逐端口排查 |
| 办公网封 50000 | 方式 C `-webSocket` |
| 构建乱码/路径含中文 | workspace 纯英文路径 |
| 服务崩溃不自恢复 | 服务恢复选项设"失败时重新启动"；日志 `C:\Jenkins\jenkins-agent.log` |
| Windows 更新重启 | MSI 服务默认自启可扛；更新时段避开构建 |

---

## 阶段 2 · 节点接入 ③：K8s 动态 agent（k3s 集群）

**拓扑**：Jenkins master（腾讯云 203.0.113.12）→ k3s 集群（server 杭州 `203.0.113.11:6443` + agent 上海 `203.0.113.13`，v1.36.4，2 节点 4C/3.5G）。构建时动态起 agent Pod、结束自动销毁；agent 经公网 8080/50000 回连 master。

**链路实测**：master→6443 ✅（401=API 正常）；k3s→203.0.113.12:8080/50000 ✅ 回程通。

**k3s 侧已完成（无需重做）**：
- `jenkins` namespace + ServiceAccount `jenkins-agent` + 最小权限 Role（pods/pods:exec/pods:log/pvc + secrets:get）
- 镜像加速 registries.yaml 已配（境内拉 docker.io 必配，详见 k3s部署实战.md）

**凭证位置备查**：

| 凭证                        | 存放位置                                                     | 说明                                                                                                              |
| ------------------------- | -------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------- |
| SA token（Jenkins Cloud 用） | 腾讯云机 `/root/ci-secrets/k3s-jenkins-sa-token.txt`（600 权限） | JWT，**2027-09 过期**；到期在阿里云机执行 `kubectl -n jenkins create token jenkins-agent --duration=8760h` 重生成并更新 Jenkins 凭证 |
| 服务器 CA                    | 腾讯云机 `/root/ci-secrets/k3s-server-ca.crt`（PEM）           | CN=k3s-server-ca，有效期至 2036-09                                                                                   |
| kubeconfig（kubectl 直用）    | 阿里云机 `/etc/rancher/k3s/k3s.yaml`（644）                    | server 本机 kubectl 免配置                                                                                           |
| 集群加入 token（扩容节点用）         | 阿里云机 `/var/lib/rancher/k3s/server/node-token`            | 新 agent 节点安装时传 `K3S_TOKEN`                                                                                      |

> ⚠️ 以上文件只存服务器本地，全文内容不录入任何笔记库/聊天。

**Jenkins 侧步骤**：
1. ✅ 装 **Kubernetes 插件**（4547.x，含 kubernetes-client-api/credentials 依赖）
2. ✅ 存凭证：Secret text，ID **`k3s-sa-token`**
3. ✅ Cloud 已保存（实参核对无误）：

| 字段 | 值 | 状态 |
|---|---|---|
| Name | `k3s` | ✅ |
| Kubernetes URL | `https://203.0.113.11:6443` | ✅ |
| Server Certificate | 粘贴 `k3s-server-ca.crt`（PEM 全文） | ✅ |
| Credentials | `k3s-sa-token` | ✅（API 实测可 list pods） |
| Namespace | `jenkins` | ✅ |
| Jenkins URL | `http://203.0.113.12:8080` | ✅ master 在集群外，必须公网地址 |
| Tunnel | 留空 | ✅ 50000 已通 |

> 📌 **Test Connection 按钮位置**：新版插件（4547.x）里它**不在 Cloud 主行**，在 **Credentials 下拉框正下方**的小蓝字。
> 📌 **点了报 403 不必慌**：它要 `list nodes` 权限，当初给 SA 的是最小权限没带这个；**不影响建 pod 跑 job**。要消除报错在 server 上给 Role 补一行：`resources: [nodes]  verbs: [get, list, watch]`。
> 📌 更真实的连通性验收不是按钮，而是下面步骤 5 的验证 job。

4. **Pod Template（当前缺的就这步）**：同 Cloud 页 → Pod Templates → **Add Pod Template**，两层配置：

   **① Pod 层**

   | 字段 | 值 | 说明 |
   |---|---|---|
   | Name | `k8s-linux` | 仅显示名 |
   | **Labels** | `k8s-linux` | **必填！** pipeline 里 `node('k8s-linux')` 只匹配这个字段（见下方实查经验 ❗） |
   | Namespace | 留空 | 继承 Cloud 的 jenkins |
   | Node selector | 留空 | 调度器自由挑（杭州/上海都可落）；固定上海填 `kubernetes.io/hostname=i-xxxxxxxx0002` |
   | Usage | **Only build jobs with label expressions matching this node** | 防误吃普通 job |
   | Build timeout (min) | `30` | 防僵尸 |
   | **Idle minutes** | `5` | 空转 5 分钟自动回收 |
   | Delete pod after finish | 不勾 | 配合 idle=5 足够 |
   | Kubernetes Service Account | 留空 | pod 自身用集群默认 SA 即可（RBAC 管的是 Jenkins 建 pod 的权限） |
   | **Workspace Volume** | 默认 EmptyDir；挂 docker.sock 外置构建（dockerfile agent）必改 **Host path = `/home/jenkins/agent`** | emptyDir + 宿主 docker `-v` 同名不同物 → 僵尸构建（详录 流水线实战.md §3.4）；UI 下拉选 HostPath；config.xml 直改用 `class` 属性式，嵌套子元素写法会被重启静默丢弃 |

   **② Container 层（至少一个；资源限额藏在每个容器的 Advanced 里，最容易漏）**

   | 字段 | jnlp 容器 | 可选 build 容器（例 maven） |
   |---|---|---|
   | Name | **`jnlp`**（一字不差，插件默认连接容器名） | `maven` 随意 |
   | Docker image | `jenkins/inbound-agent:latest-jdk21`（⚠️ JDK 21 同 Windows 线要求） | `maven:3.9-eclipse-temurin-21` |
   | Commands to run / Args / Shell / Yaml | **全部留空**（填了会覆盖插件默认启动命令，pod 秒退） | 留空；若容器秒退再补 `cat` + `/dev/null` |
   | CPU request / limit | `200m` / `500m` | `200m` / `1` |
   | Memory request / limit | `256Mi` / `512Mi` | `512Mi` / `1Gi` |

   > 集群合计才 4C/3.5G，**limits 不填=不设防**，一个失控构建就能把节点压到 kubelet 驱逐一切，必设。首次建议只建 jnlp 一个容器打通链路，工具链容器后续加。
   >
   > 📌 **实查经验**（直接看 `$JENKINS_HOME/config.xml` 的 `<templates>` 段即可验配置是否真保存，无需凭证）：
   > - **UI 建模板后最容易漏的一步**：Containers 区域要再点一次 **Add Container**——只填 Pod 层不建容器，存出来就是 `<containers/>` 空壳，job 永远 Pending
   > - 资源限额四项（`resourceRequestCpu/Memory`、`resourceLimitCpu/Memory`）全空 = Advanced 没展开，jnlp 轻负载勉强能跑，**加工具链容器前必须补**
   > - ❗ **Labels 栏必填，模板名不会隐式当 label**（实测于插件 4547.x）：只填 Name 不填 Labels，job 会永远卡在队列报 `There are no nodes with the label 'k8s-linux'`，且 **Jenkins 日志和集群 events 双双静默**（插件压根不调创建 API），极难从报错定位——遇到这个报错第一反应就是查 `<labels>` 是否保存进 config.xml
   > - idleMinutes=0 + Cloud 级 podRetention=Never 也能用完即删，只是没"空转回收"兼容位；单模板实验期可不纠结
   > - **requests 可留空**：K8s 规则缺省自动抬到 limit 同值，pod 成 Guaranteed QoS；jnlp 这种轻连接容器一口价即可，工具链容器再显式区分 request/limit
   > - **单位笔误插件不拦**：手滑写成 `1024Ni` 这种非法单位能直接保存进 config.xml，直到建 pod 才被 API 拒——保存后复查一眼 `<resourceLimitMemory>` 是习惯（合法后缀：E/P/T/G/M/K ± i）

5. 验证 job（新建 Pipeline 类型 job，直接贴下面的 Jenkinsfile）：
```groovy
pipeline {
  agent { label 'k8s-linux' }
  stages {
    stage('Where am I') {
      steps {
        sh '''
          echo "== hostname =="      ; hostname
          echo "== os =="            ; cat /etc/os-release | head -2
          echo "== dns (集群内证明)" ; cat /etc/resolv.conf
          echo "== java (JDK21?)"    ; java -version
          echo "== 资源限额 =="      ; cat /sys/fs/cgroup/memory.max 2>/dev/null || cat /sys/fs/cgroup/memory/memory.limit_in_bytes
          df -h / | tail -1
        '''
      }
    }
    stage('Works?') {
      steps { sh 'echo "hello from k3s dynamic agent" > ok.txt && cat ok.txt' }
    }
  }
  post { always { echo "workspace: ${env.WORKSPACE}" } }
}
```
**通过标准**（逐项核构建日志）：
- `resolv.conf` 出现 `nameserver 10.43.0.10` + `svc.cluster.local` → 确认在 k3s pod 内
- `java -version` 报 **21.x** → inbound-agent JDK21 镜像就位
- memory 限额打出 `536870912`（=512Mi）→ 模板 limits 真实生效
- 构建绿 ✅，结束后 `kubectl -n jenkins get pods` 为空（用完即删）

> ⚠️ 默认生成的 "Hello World" pipeline 不能用：它的 `agent any` 不带标签，不会匹配 k8s 模板（label 模板 Usage=Only… 时），白测。

**pod 清理时机**（当前实配）：Cloud 级 `podRetention=Never` → 构建结束（成败中止皆然）插件**秒级删 pod**；channel 卡死极端情况由 retentionTimeout=5min 兜底强删。模板 `idleMinutes=0` 此模式下是摆设；若想"热 pod 复用省冷启动"，改 idleMinutes=5 即可（空转 5 分钟无新构建才回收）。

> 排查提示：job 失败且集群 events 空（`kubectl get events -n jenkins`）= Jenkins 压根没调创建 API，查模板 label/Usage 匹配；有创建事件但卡 Pending = 看 pod events 区分镜像/调度/回连问题；events 保留 1 小时，事后取证来得及。

6. 进阶：Pod 模板 YAML 化写进 Jenkinsfile（`agent { kubernetes { yaml '...' } }`），依赖缓存挂 local-path PVC 复用。

**已知坑**：
- agent pod 卡 ContainerCreating → 十有八九镜像加速没配好（新节点记得带 registries.yaml）
- **Raw YAML 盖不掉 workspace 卷**：插件在 YAML 合并**之后**才补挂默认 emptyDir `workspace-volume`，Overrides 里写同名 hostPath 无效 → 必须用模板字段 `workspaceVolume`；且 config.xml 手写要多态 `class` 属性式（`<workspaceVolume class="...HostPathWorkspaceVolume">`），嵌套子元素写法会被 XStream **静默丢弃**（无 SEVERE、重启后字段消失、pod 依旧 emptyDir）→ 改完重启后 grep 确认字段存活再跑验证构建（全记录 流水线实战.md §3.4）
- pod 起来了但 job 卡 Pending → Cloud 的 Jenkins URL 不对，pod 连不回 master（localhost/内网地址必死，必须公网 `203.0.113.12:8080`）
- 上海节点 agent 日志每 ~35s 刷一条 `remotedialer ... 172.23.174.21 timeout` → 当年是跨 VPC 预期回落行为非故障；**2026-09-05 已根治**（沪杭 VPC 对等 + agent 切内网回连，噪音清零，实录见 k3s部署实战.md「VPC 对等连接打通实录」）
- token 过期表现为 pod 创建 403 → 重新 create token + 更新 Jenkins 凭证

---



## 运维实录 · k8s 动态 agent clone CodeUp 失败（Could not resolve host）→ 修复全记录

> 2026-09-05 实测 · 现象：#92 FAILURE → 修复后 #93 SUCCESS
> 根因链条 + 修复步骤逐条可复现，作为 k8s agent 出网/DNS 排障的样板。

### R.1 现象与初步定位

- **job `hello-jenkins` 连续失败**：#92 FAILURE，其余 ABORTED
- #92 尾部报错（`builds/92/log` 直接看，无需凭证）：
  ```
  ERROR: Error cloning remote repo 'origin'
  stderr: fatal: unable to access 'https://codeup.aliyun.com/...': Could not resolve host: codeup.aliyun.com
  ERROR: Maximum checkout retry attempts reached, aborting
  Finished: FAILURE
  ```
- 初判是"构建 agent pod DNS 解析不了公网域名 codeup"（Git 仓库在阿里云云效 CodeUp）。

### R.2 分清三台机器与两个网络域（关键前提）

| 角色 | IP | 归属 | 网络 |
|:-----|:-----|:-----|:-----|
| Jenkins master | 203.0.113.12（腾讯云，即本机 VM-0-7 环境） | 腾讯云 | 公网可达 203.0.113.11:6443 |
| k3s server | 203.0.113.11 | 阿里云杭州 172.23.174.21 | 跑 CoreDNS / API |
| k3s agent | 203.0.113.13 | 阿里云上海 172.28.89.44 | 跑构建 pod（跨 VPC） |

> ⚠️ **server 与 agent 不同 VPC、内网互 ping 不通**（文档早就记录）——这直接导致后续**从 server 拉 agent 上 pod 的日志/exec 全部 502**，排障要绕开这条路。

### R.3 用集群内诊断 pod 复现并钉死根因（不需要重启构建）

Jenkins 侧留了 `k3s-sa-token`（本机 `/root/ci-secrets/`），用它直接打 k8s API，往 `jenkins` 命名空间放诊断 pod，等效于"看构建会怎样"：

1. **建 busybox pod 测 DNS** → `nslookup codeup.aliyun.com` 竟解析成功（203.0.113.14）
2. **建 curl pod 连续测** → `Resolving timed out after 3000ms` 连续 10 次全挂
   - 结论：**不是 pod DNS 坏了，是"时好时坏"的间歇性失败**——第一次正好赶上能解析。

**关键判断**：真正要区分的是"agent 节点自身网络" vs "集群 CoreDNS 转发"，于是从宿主机各侧直查：

| 位置 | 解析 codeup | 结论 |
|:-----|:-----|:------|
| agent 宿主机（100.100.2.136 上游） | ✅ 秒回 | agent 出网 OK |
| server 宿主机（127.0.0.53） | ✅ 秒回 | server 出网 OK |
| **agent 宿主机直查集群 CoreDNS(10.43.0.10)** | ❌ 5 次全超时 | **CoreDNS 这条转发链是瓶颈** |

### R.4 根因：CoreDNS 上游 = server 的 systemd-resolved，跨 VPC 链路不稳

- CoreDNS Corefile：`forward . /etc/resolv.conf`，pod 内 `dnsPolicy: Default` → 转发目标是 **server 宿主 /etc/resolv.conf 的 127.0.0.53（systemd-resolved）**
- agent 侧的 pod → 集群 CoreDNS（在 server 上）→ systemd-resolved 递归公网域名，这条链路过跨地域公网回程**不稳定** → codeup clone 偶发 `Could not resolve host`

### R.5 修复步骤（在 server 203.0.113.11 上，cluster-admin）

1. **选可靠上游**：实测阿里公网 DNS 秒回且 codeup 同厂商：
   - ✅ `223.5.5.5` / `223.6.6.6`（阿里）/ `100.100.2.136`（内网）0.00s
   - ❌ `119.29.29.29`（腾讯）跨厂商 timeout——别选
2. **备份 + 改 CoreDNS 上游**（绕过 systemd-resolved）：
   ```bash
   kubectl -n kube-system get cm coredns -o yaml > /root/coredns-cm-backup-$(date +%Y%m%d-%H%M).yaml
   kubectl -n kube-system get cm coredns -o yaml \
     | sed 's|forward \. /etc/resolv.conf|forward . 223.5.5.5 223.6.6.6|' \
     | kubectl apply -f -
   ```
3. **滚动重启 CoreDNS**，但 rollout 卡住——发现新 pod 被调度到 **agent 节点**，且就绪探针 503（跨 VPC，探针/日志都 502 拉不到）。
4. **根治：给 CoreDNS 加 nodeSelector 固定回 server 节点**（原部署时 CoreDNS 本就在 server）：
   ```bash
   kubectl -n kube-system patch deployment coredns --type=strategic \
     -p '{"spec":{"template":{"spec":{"nodeSelector":{"kubernetes.io/hostname":"i-xxxxxxxx0001"}}}}}'
   kubectl -n kube-system rollout restart deployment/coredns
   kubectl -n kube-system rollout status deployment/coredns   # 成功
   ```
   → 新 pod 回到 server，READY 1/1，endpoints 恢复。

### R.6 收尾验证

- `kubectl get nodes`：server(172.23.174.21) / agent(172.28.89.44) 双 Ready
- CoreDNS 固定回 server 节点、就绪、endpoints 有 10.42.0.x 后端
- **Jenkins 重启 #93 构建 → `Cloning repository https://codeup.aliyun.com/... ` 成功，`Finished: SUCCESS`** ✅

> 📌 **经验沉淀**：
> - k8s agent clone 公网仓库报 `Could not resolve host`，先分清"agent 宿主机"和"集群 CoreDNS"两个解析路径，各侧直查再下结论（宿主机 OK ≠ pod OK）。
> - server→agent 跨 VPC 时，**拉 pod 日志 / exec 会 502**（6443 代理到 agent kubelet 走不通），排障优先用"诊断 pod 把结果写进 termination-log，从 API 读 pod status"，绕开 502。
> - CoreDNS 这种系统级 Deployment，**别让它漂到跨 VPC agent 节点**；固定 nodeSelector 到 server 是稳定性关键。
> - 凭据：集群管理 kubeconfig 在 server `/etc/rancher/k3s/k3s.yaml`；Jenkins 侧受限 SA token 本机 `/root/ci-secrets/k3s-jenkins-sa-token.txt`（只够 list pods / 建诊断 pod，改不了 kube-system）。


## 待办

- [ ] Jenkins 管理员 API Token（解锁 CLI 自动化与 MSI 向导凭证）
- [ ] 核对/清理旧节点 `intranet-agent-01` 当前在线状态
- [x] ~~Windows 节点接入~~ ✅ 2026-09-04 完成：win-agent-01 在线（inbound JNLP，JDK 21）
- [ ] Pod Template 配置（第七章步骤 4 照抄）+ 验证 job 跑通 = k8s agent 全线闭环
- [x] ~~k3s 扩容 agent 节点~~ ✅ 2026-09-04 完成：203.0.113.13（上海，跨 VPC 走公网 6443；cgroup v2 已切）
