---
{"dg-publish":true,"permalink":"/knowledge/devops/README/","title":"DevOps 知识体系","tags":["DevOps","嵌入式","云原生","AIOps","知识体系","Linux","容器","Kubernetes","CI-CD","可观测性"],"dg-note-properties":{"title":"DevOps 知识体系","date":"2026-07-22","updated":"2026-11-06","tags":["DevOps","嵌入式","云原生","AIOps","知识体系","Linux","容器","Kubernetes","CI-CD","可观测性"]}}
---


# 🧭 DevOps 知识体系

> 从 Linux 基础到 AIOps 的完整技术地图：**Linux/网络 → 容器编排 → CI/CD → 制品与配置 → 质量与安全 → 可观测性 → AIOps**。
> 每章给「该学什么 + 核心认知」，落地实操见 [[knowledge/devops/practices/README\|🛠️ 实战演练]]。

返回 [[首页\|🏠 首页]]

---

## 📑 章节索引

| 章节 |
|:--|
| 一、Linux 网络与脚本基础 |
| 二、容器与编排 |
| 三、CI/CD 持续集成交付 |
| 四、制品与配置管理 |
| 五、质量与安全 |
| 六、可观测性 |
| 七、AI 赋能 DevOps（AIOps） |

> 行业方向（汽车 / 机器人）见 [[knowledge/devops/industry/00-行业对比\|🏭 行业对比]]。

---

## 一、Linux 网络与脚本基础

> DevOps 的地基：Linux 系统管理、Shell 自动化、网络与安全基础，以及自动化脚本语言选型。

### 1.1 Linux 系统管理

- **文件系统与磁盘** Linux 里"一切皆文件"——普通文件、目录、设备、套接字，都通过统一的文件接口访问。日常运维第一件事，是搞清楚**磁盘空间去哪了**。

`df` 和 `du` 经常成对使用，但看的不是一回事：`df` 展示的是**文件系统层面**的块占用（整个分区用了多少），`du` 统计的是**目录里文件实际占用的空间**。两者对不上时，最常见的原因是——**有文件被删除了，但进程还开着它的句柄**。这类文件对 `du` 不可见（目录项没了），却实实在在占着块（inode 还挂着），`df` 会如实反映。定位方法是 `lsof +L1`（列出 link count 为 0 的打开文件），释放要么重启持有它的进程，要么 `: > /proc/<pid>/fd/<fd>` 把内容清空。

| 命令 | 作用 |
|:----|:------|
| `df -h` | 各文件系统的使用率 |
| `du -sh /var/log` | 指定目录总占用 |
| `ls -la` | 列出文件（含隐藏、权限） |
| `lsof -p PID` | 进程打开的文件 |
| `find / -name "*.log" -mtime -1` | 找最近一天修改的日志 |

- **inode 与链接** 理解 inode 是理解 Linux 文件系统的钥匙。**inode 存的是文件的"身份"**——权限、属主、大小、时间戳、指向数据块的指针，唯独**不存文件名**。文件名只是目录里一条"名字 → inode 号"的映射。

由此引出两种链接：**硬链接**（`ln src dst`）是给同一个 inode 挂第二个名字，两个名字地位平等，删掉一个另一个照常，且不能跨文件系统（inode 号只在单个文件系统内唯一）；**软链接**（`ln -s src dst`）则是单独的一个文件，内容就是"指向的路径"，所以能跨文件系统、能指向目录、也可能"悬空"（目标没了）。

```bash
ls -i file            # 查 inode 号
stat file             # 看元数据详情
find / -inum 12345    # 按 inode 反查文件
```

这也解释了一个经典现象：磁盘"满了"删文件却释放不出空间——因为 inode 还被进程引用，`du` 看不见，`df` 却减不掉。

- **权限管理** Linux 的权限模型围绕"三种身份 × 三种操作"展开：身份是**属主（owner）/ 属组（group）/ 其他（other）**，操作是**读（r=4）/ 写（w=2）/ 执行（x=1）**。用 `ls -l` 看到的 `-rwxr-xr--`，就是三组三位的组合。

```
-rwxr-xr--  1 user group  4096 Jul 19 10:00 app
 │││││││││
 │└─ rwx (owner) ─── rwx = 7
 │   └─ r-x (group) ── r-x = 5
 │       └─ r-- (other) ─ r-- = 4
```

所以 `chmod 755` 的含义一目了然：属主可读写执行（7），属组和其他人只读+执行（5）。日常还要注意：**目录的 x 位含义是"能否进入/遍历"**，没有 x 的目录即使有 r 也进不去。

| 命令 | 作用 |
|:----|:------|
| `chmod 755 file` | 改权限（数字或 `u+x` 符号法） |
| `chown user:group file` | 改属主和属组 |
| `sudo -u appuser cmd` | 以指定用户执行 |
| `visudo` | 安全编辑 sudoers（带语法校验） |

- **进程与信号** 进程是运行中的程序。`ps aux` 看快照，`top`/`htop` 看实时。**进程状态**是排障时的关键线索：`R` 运行中、`S` 可中断睡眠（在等事件，可被唤醒）、`D` 不可中断睡眠（通常在等磁盘/网络 IO，**连 kill -9 都杀不掉**）、`Z` 僵尸（已退出但父进程还没回收它的退出码）、`T` 已停止。

**信号**是进程间通信的一种。最常用的两个：`SIGTERM`（15）是"请你优雅退出"，进程能捕获它做清理（关连接、刷缓冲）；`SIGKILL`（9）是"立即处决"，内核直接干掉，进程无法捕获，因此**可能丢数据**。所以规范是：先 `kill`（默认发 15），等一会儿不行再 `kill -9`。

| 命令 | 作用 |
|:----|:------|
| `ps aux` | 所有进程快照 |
| `top` / `htop` | 实时资源监控 |
| `kill PID` | 发 SIGTERM（15，可捕获） |
| `kill -9 PID` | 发 SIGKILL（9，不可捕获） |
| `systemctl status/restart svc` | systemd 管理服务 |

- **systemd 服务管理** systemd 是现代 Linux 的"1 号进程"，管着系统启动和服务生命周期。它用 **unit 文件**描述服务，并支持依赖关系（`After=`/`Requires=`），因此能按拓扑顺序拉起服务。日常最常踩的坑：**改完 unit 文件必须 `daemon-reload`**，否则 systemd 用的还是旧配置。

```bash
systemctl daemon-reload          # 改 unit 文件后必须执行
systemctl enable --now svc       # 开机自启并立即启动
systemctl list-dependencies svc  # 看依赖树
journalctl -u svc --since "10 min ago"  # 看服务日志
```

- **排查常用** ```bash
ps aux --sort=-%cpu | head -10   # CPU 占用最高的进程
ps aux --sort=-%mem | head -10   # 内存占用最高的进程
ss -tlnp | grep :8080            # 谁占着 8080 端口
iostat -x 1                      # 磁盘 IO 展开视图（%util 高说明盘是瓶颈）
free -h                          # 内存与 swap
```

## 1.2 Shell 自动化

Shell 是运维的"胶水语言"：把一个个小工具用管道串起来，快速完成一次性任务。三类核心工具值得记牢。

**grep 负责"找"**：`grep -r` 递归搜索，`grep -v` 反向过滤（排除匹配行）。**awk 负责"取列和算"**：它按行处理、按列切分，`$1` 是第一列、`$NF` 是最后一列、`$0` 是整行，还能带条件（`awk '$3 > 80 {print $1}'`）。**sed 负责"改"**：`s/old/new/g` 是全局替换，`-i` 原地修改。

```bash
# grep：过滤
grep -r "ERROR" /var/log/     # 递归搜日志里的 ERROR
grep -v "DEBUG" app.log       # 排除 DEBUG 行

# awk：按列处理
awk '{print $1, $NF}' file            # 打印第 1 列和最后一列
df -h | awk '$5+0 > 80 {print $1}'    # 磁盘使用率超 80% 的分区

# sed：替换
sed 's/old/new/g' file        # 全局替换
sed -i 's/debug//g' file      # 原地删除所有 debug
```

**定时任务用 cron**，格式是"分 时 日 月 周"五段。常见误区：cron 执行时**没有交互式 shell 的环境变量**（PATH 很短），所以脚本里最好用绝对路径、显式声明环境。另外 cron 的"周"和"日"是**或**的关系，同时指定容易踩坑。

```bash
# 分 时 日 月 周 命令
0 2 * * * /scripts/backup.sh     # 每天凌晨 2 点
*/5 * * * * /scripts/health.sh   # 每 5 分钟
```

写健壮的脚本，开头加 `set -euo pipefail` 是基本功：`-e` 遇错退出、`-u` 未定义变量报错、`-o pipefail` 让管道里任一环失败即失败。

## 1.3 网络基础

- **TCP/IP 分层** 网络通信被抽象成四层，每层只操心自己的事（分层解耦）：

```
┌─────────────┐
│   应用层     │  HTTP / HTTPS / DNS / SSH
├─────────────┤
│   传输层     │  TCP / UDP
├─────────────┤
│   网络层     │  IP / ICMP
├─────────────┤
│   链路层     │  Ethernet / ARP
└─────────────┘
```

**TCP vs UDP** 是这里的核心分水岭：TCP 面向连接、保证可靠有序（有握手、确认、重传、流量控制），代价是开销大；UDP 无连接、不保证到达和顺序，胜在轻快，适合 DNS 查询、视频流、游戏这类"丢一两包无所谓、要快"的场景。

- **DNS 解析** DNS 把好记的域名翻译成 IP。最需要搞清楚的是**解析链路**：先查本地 hosts，再问本地 DNS 缓存/递归服务器，递归服务器若没缓存，就从**根域名服务器 → 顶级域（.com）→ 权威域名服务器**逐级问下去，最后拿到结果并缓存。`TTL` 决定了这条记录能被缓存多久——**TTL 短**适合需要快速切换（如切流量、故障转移），**TTL 长**则减少查询压力。

| 记录 | 含义 |
|:----|:------|
| A | 域名 → IPv4 |
| AAAA | 域名 → IPv6 |
| CNAME | 域名 → 另一个域名（别名） |
| MX | 邮件服务器 |

```bash
dig example.com +trace    # 从根开始跟踪解析全过程
nslookup example.com      # 简单查询
cat /etc/resolv.conf      # 本机 DNS 配置
```

- **负载均衡** 负载均衡按**工作层次**分两类。**L4** 只看 IP 和端口做转发，不解析应用层内容，性能高、通用，典型是 Nginx Stream、云 SLB；**L7** 会解析 HTTP，能按域名、路径、Header 路由，灵活但开销大，典型是 Nginx、K8s Ingress。

```
用户 → DNS → L4 LB:443 → Nginx（TLS 终结）→ L7 路由 → 后端 Pod
```

实际架构里常见的分工是：L4 把流量引进来，L7 在里层做精细路由，TLS 往往在 L7 这层终结。

- **TLS 握手** HTTPS 的秘密在于"**握手用非对称、传输用对称**"这套组合拳。原因很实际：非对称加密（RSA/ECDHE）安全但慢，对称加密（AES）快但双方得先有同一把密钥——于是用非对称的方式，安全地协商出一把对称的会话密钥，之后的数据都用这把会话密钥加密。

```
Client Hello（支持的加密套件 + 随机数）
  → Server Hello（选定的套件 + 随机数 + 证书）
  → 客户端验证证书链（CA 签名 / 有效期 / 域名 SAN）
  → 密钥交换（ECDHE）
  → 双方各自导出会话密钥 → 加密通信
```

这里有三个常被追问的点：

**前向保密（PFS）**——如果用 RSA 做密钥交换，服务器私钥一旦泄露，攻击者可以解密**历史上所有**截获的流量；换成 ECDHE（临时椭圆曲线），每次会话的密钥都是临时产生的，私钥泄露也解不开旧流量。

**证书链**——信任是逐级传递的：站点证书由中间 CA 签发，中间 CA 由根 CA 签发，而根 CA 早已内置在浏览器/系统里。服务端必须下发**完整证书链**，否则某些客户端（不缓存中间证书的）会因拼不出链而报错。

**验证什么**——客户端要校验证书是否由受信任 CA 签发、是否过期、证书里的域名（SAN）是否与访问的域名匹配。

- **网络排障** 排障讲究"从下往上、逐层确认"：先 `ping` 通不通（网络层），再 `telnet host 443` 看端口开没开（传输层），然后 `curl -v` 看 HTTP 层细节，必要时 `tcpdump` 抓包看真实报文。

```bash
ping -c 4 host            # 连通性（ICMP）
traceroute host           # 路由逐跳
curl -v https://host      # HTTP 链路诊断
ss -tlnp                  # 本机监听端口
ss -tnp state established # 已建立的连接
tcpdump -i eth0 port 80   # 抓包
```

- **TCP 三次握手与四次挥手** ```text
握手：CLIENT → SYN → SERVER
     CLIENT ← SYN+ACK ← SERVER
     CLIENT → ACK → SERVER

挥手：主动方 → FIN →  被动方
     主动方 ← ACK ←  被动方     （半关闭：一方不再发，但仍能收）
     主动方 ← FIN ←  被动方
     主动方 → ACK →  被动方 → TIME_WAIT（等待 2MSL）
```

**为什么握手是三次而不是两次？** 两次的话，服务端收到一个 SYN 就建立连接，但这个 SYN 可能是**网络里滞留的旧请求**，服务端会白白占着资源等一个不会来的客户端。第三次 ACK 让服务端确认"对方确实收到了我的 SYN+ACK，且确实想连接"。

**挥手为什么是四次？** 因为 TCP 是全双工的，一方说"我不发了"（FIN）之后，另一方可能还有数据要发，所以 ACK 和它自己的 FIN 不能合并，于是比握手多一次。

**TIME_WAIT 为什么要等 2MSL？** 两个原因：一是确保自己最后的 ACK 能到达对方（对方没收到会重发 FIN，得有人应答）；二是让本次连接里滞留的旧报文在网络中消散，避免干扰下一个同四元组的新连接。

**大量 TIME_WAIT / CLOSE_WAIT 怎么排查？** 一句话：TIME_WAIT 集中在**主动关闭方**，多了可以做 `tcp_tw_reuse` 复用；而 **CLOSE_WAIT 是"对方发了 FIN，我还没 close()"**——这是应用层没正确关闭连接，属于代码 bug，得改程序而不是调内核参数。

```bash
ss -s                                      # 连接状态汇总
ss -tn state time-wait | wc -l             # TIME_WAIT 数量
netstat -n | awk '/^tcp/ {++S[$NF]} END {for(a in S) print a, S[a]}'
```

### 1.4 安全基础

- **SSH 免密与别名** SSH 用**非对称密钥**做认证：你把公钥放到服务器的 `authorized_keys`，登录时用私钥证明身份。`ed25519` 比 RSA 更短更安全，是当前推荐。`~/.ssh/config` 里配别名能省去每次敲一长串参数。

```bash
ssh-keygen -t ed25519          # 生成密钥对
ssh-copy-id user@host          # 把公钥推到目标机

# ~/.ssh/config 别名
Host prod
    HostName 10.0.1.100
    User deploy
    IdentityFile ~/.ssh/prod_key
```

- **证书与防火墙** ```bash
# 查证书有效期
openssl s_client -connect host:443 2>/dev/null | openssl x509 -noout -dates

# 自签证书（仅测试）
openssl req -x509 -newkey rsa:4096 -keyout key.pem -out cert.pem -days 365 -nodes

# iptables 基本操作
iptables -L -n                                   # 查看规则
iptables -A INPUT -p tcp --dport 443 -j ACCEPT   # 放行 443

# firewalld（更上层、更易用）
firewall-cmd --add-port=8080/tcp --permanent
firewall-cmd --reload
```

- **安全加固 Checklist** - [ ] 禁用 root SSH 登录，只用密钥认证
- [ ] 防火墙只开放必要端口
- [ ] 定期 `yum/apt update` 打安全补丁
- [ ] 启用审计日志（`auditd`）
- [ ] SELinux / AppArmor 保持 enforcing

## 1.5 脚本语言选型

不同语言在 DevOps 里的定位，是**按场景分的**，没有万能。选择的核心是看任务性质：一次性粘合 → Shell；复杂工具/API 调用/数据处理 → Python；写 K8s Operator 或高性能 CLI → Go；写 Jenkins Pipeline → Groovy。

| 语言         | 适合                         | 不适合             |
| :--------- | :------------------------- | :-------------- |
| **Python** | CI/CD 工具、API 封装、数据处理、运维自动化 | 高性能服务、系统级工具     |
| **Shell**  | 构建脚本、系统巡检、一次性操作            | 复杂业务逻辑、跨平台      |
| **Go**     | K8s Operator、CLI 工具、中间件    | 快速原型验证          |
| **Groovy** | Jenkins Pipeline           | 任何非 Jenkins 的场景 |

- **Python** Python 是 DevOps 脚本的"万能胶"，生态最全。典型场景包括写构建/发布脚本、批量运维、调用各家 API（K8s / 云厂商 / GitLab）、处理日志和监控数据。调用 K8s API 时，官方 `kubernetes` 库是最直接的方式：

```python
from kubernetes import client, config

config.load_kube_config()                     # 读 ~/.kube/config
v1 = client.CoreV1Api()
pods = v1.list_namespaced_pod(namespace='production')
for pod in pods.items:
    print(f"{pod.metadata.name}: {pod.status.phase}")
```

常用库：`requests`（HTTP）、`click`/`typer`（CLI 框架）、`Jinja2`（模板渲染，生成配置文件）、`pandas`（数据分析）。

- **Go** Go 是"云原生时代的母语"——K8s、Docker、etcd、Prometheus 全是 Go 写的。它的强项是**编译成单二进制、并发模型简洁、性能接近 C**。用 `client-go` 操作 K8s 是标准做法：

```go
import "k8s.io/client-go/kubernetes"

clientset, _ := kubernetes.NewForConfig(config)
pods, _ := clientset.CoreV1().Pods("default").List(ctx, metav1.ListOptions{})
for _, pod := range pods.Items {
    fmt.Println(pod.Name)
}
```

- **Groovy** Groovy 在这里几乎是"为 Jenkins Pipeline 而生"的语言——它运行在 JVM 上，语法灵活，是写 Jenkinsfile 和 Jenkins Shared Library 的语言。

```groovy
// Jenkins 共享库示例
def call(Map config) {
    pipeline {
        agent any
        stages {
            stage('Build') {
                steps { sh config.buildCmd }
            }
        }
        post {
            failure {
                emailext body: '${BUILD_URL}', subject: 'Build Failed', to: config.owner
            }
        }
    }
}
```

---

## 二、容器与编排

> 从"一个容器怎么跑起来"，到"成百上千个容器怎么被调度、编排、治理"。

## 2.1 Docker

> 底层原理（namespaces / cgroups / UnionFS）见「Docker 底层原理入门」。

- **容器到底"轻"在哪** 要理解容器，先要和虚拟机对比。虚拟机是"**造一台完整的假电脑**"：它模拟硬件、装一个完整的 Guest OS，隔离最彻底，但启动要分钟级、体积 GB 起、资源开销大。容器则完全不同——它不模拟硬件，而是**直接借用宿主机的内核**，只是通过内核提供的几项机制"假装"自己独占一台机器。因此容器启动是秒/毫秒级、体积 MB 级、几乎没有额外资源开销。

| 维度 | 容器 | 虚拟机 |
|:----|:------|:------|
| 隔离级别 | 进程级（共享内核） | 硬件级（独立 Guest OS） |
| 启动速度 | 秒 / 毫秒级 | 分钟级 |
| 体积 | MB 级 | GB 级 |
| 隔离强度 | 较弱（共享内核） | 强 |
| 资源开销 | 低 | 高 |

**这套"假装"由三样东西合起来实现**（Docker 的本质）：

- **namespaces**——让容器"看不见别人"：PID、网络、挂载、用户等命名空间隔离，容器里 `ps` 只能看到自己的进程。
- **cgroups**——让容器"用不超过配额"：限制 CPU、内存、IO 的用量。
- **UnionFS（联合文件系统）**——让容器"有自己的一套文件"：把只读的镜像层和可写的容器层叠成一个统一的根文件系统。

代价是隔离没那么硬（共享内核意味着内核漏洞可能被穿透），所以高安全场景会叠加 gVisor、Kata 这类"沙箱容器"来补强。

- **Dockerfile 与镜像分层** 镜像不是一坨整体，而是**一层层叠出来的只读层**。Dockerfile 里每条会改文件系统的指令（`RUN`/`COPY`/`ADD`）都产生一层。这个设计带来两个直接好处：**层可被多个镜像共享**（磁盘省），**未变的层可复用缓存**（构建快）。

```dockerfile
FROM node:18-alpine AS builder       # 构建阶段
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production

FROM node:18-alpine                  # 运行阶段（重新起一层干净基础）
WORKDIR /app
COPY --from=builder /app/node_modules ./node_modules
COPY . .
EXPOSE 3000
CMD ["node", "server.js"]
```

几个要点：

- **多阶段构建**：把"编译需要但运行不需要"的东西（编译器、源码、构建缓存）留在 builder 阶段，最终镜像只拷产物，体积能小一个数量级。
- **层缓存利用**：`COPY package*.json` 再 `RUN npm ci`，是为了"依赖没变就不重装"——只要改了业务代码而不动依赖，前面的层直接命中缓存。
- **Alpine 基础镜像**：体积小、攻击面小（用 musl 而非 glibc，个别依赖可能有兼容问题）。
- **.dockerignore**：把 `node_modules`、`.git` 排除在构建上下文外，既快又不误拷。

- **docker-compose** 单机跑多个容器时，`docker-compose` 用一份 YAML 描述"一整套服务"：谁依赖谁、端口怎么映射、卷怎么挂。

```yaml
version: "3.8"
services:
  app:
    build: .
    ports: ["3000:3000"]
    environment:
      DB_HOST: db          # 用服务名当主机名，compose 内置 DNS
    depends_on: [db]
  db:
    image: postgres:16-alpine
    volumes: [pgdata:/var/lib/postgresql/data]
volumes:
  pgdata:
```

`depends_on` 只保证**启动顺序**，不保证 db "已经能接受连接"——真要用健康检查（`healthcheck`）来等就绪。

- **镜像安全扫描** 镜像里可能带着有漏洞的基础库，所以推送前扫描是标配：

```bash
trivy image myapp:v1.2.3        # 输出按严重级分层的漏洞列表
```

（CI 里通常集成成"扫到 HIGH/CRITICAL 就阻断推送"），扫描可以作为构建后阶段，也可以 `docker build` 时内置。

## 2.2 Kubernetes

Kubernetes（K8s）解决的是"**一个容器好跑，一千个容器怎么管**"的问题。它的核心哲学只有一句话：**声明式 API + 控制器循环**。你不去命令它"启动 3 个副本"，而是"声明"期望状态（我要 3 个副本、镜像是什么），然后一堆**控制器（Controller）**在后台不断比对"实际状态"和"期望状态"，不满足就动手补齐——这个过程叫 **Reconcile（调谐）**。所以哪怕一个 Pod 崩了，控制器也会自动再拉一个起来，这就是"自愈"的来源。

- **核心资源** | 资源 | 作用 | 一句话理解 |
|:----|:------|:------|
| **Pod** | 最小调度单位 | 一个或多个共享网络/存储的容器 |
| **Deployment** | 无状态应用 | 管副本数、滚动更新、回滚 |
| **StatefulSet** | 有状态应用 | 有稳定网络标识和独立存储（数据库、MQ） |
| **Service** | 服务发现与负载均衡 | 给一组 Pod 一个稳定入口 |
| **Ingress** | HTTP 路由 | 域名/路径 → Service（七层） |
| **ConfigMap / Secret** | 配置 / 敏感配置 | 把配置从镜像里剥离出来 |
| **PVC** | 存储申请 | 声明"我要一块持久盘" |

- **从 Deployment 看 K8s 的层次** ```yaml
apiVersion: apps/v1
kind: Deployment
metadata: { name: myapp }
spec:
  replicas: 3
  selector:
    matchLabels: { app: myapp }     # 认领带此标签的 Pod
  template:                          # Pod 模板：Pod 长什么样
    metadata:
      labels: { app: myapp }
    spec:
      containers:
      - name: app
        image: myapp:v1.2.3
        ports: [{ containerPort: 3000 }]
        resources:
          requests: { cpu: 100m, memory: 128Mi }
          limits:   { cpu: 500m, memory: 256Mi }
        livenessProbe:
          httpGet: { path: /health, port: 3000 }
        readinessProbe:
          httpGet: { path: /ready,  port: 3000 }
```

层次关系是：**Deployment 管 ReplicaSet，ReplicaSet 管 Pod**。Deployment 每次更新镜像，就会生成一个新的 ReplicaSet，旧 RS 保留着以便回滚。

- **Service 与 Ingress：从"临时 IP"到"稳定入口"** Pod 的 IP 是临时的——重建就变。所以需要一个稳定的访问入口：**Service**。它通过**标签选择器**找到一组 Pod，并给它们一个稳定的虚拟 IP（ClusterIP），请求进来后自动负载均衡到后端 Pod。**Ingress** 则更上层，处理"从集群外进来的 HTTP 流量"：按域名和路径路由到不同 Service，通常由 Nginx、Traefik 这类 Ingress Controller 实现。

```yaml
# Service：给 myapp 的 Pod 一个稳定入口
apiVersion: v1
kind: Service
metadata: { name: myapp }
spec:
  selector: { app: myapp }
  ports: [{ port: 80, targetPort: 3000 }]
---
# Ingress：app.example.com/ → myapp:80
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata: { name: myapp }
spec:
  ingressClassName: nginx
  rules:
  - host: app.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service: { name: myapp, port: { number: 80 } }
```

### 2.3 调度与资源

**调度（Scheduling）** 决定"这个 Pod 该放到哪个节点"。流程分两步：**过滤（Filter）**——先剔除所有不满足条件的节点（资源不够、亲和不符、污点没容忍）；**打分（Score）**——给剩下的节点打分，选最优的绑定上去。

- **requests 与 limits：调度与上限** 这两个参数容易混，但作用完全不同：

- **requests** 是"**保证量**"——调度器按它来找节点。一个节点上所有 Pod 的 requests 总和不能超过节点容量，所以 requests 决定了 Pod **能不能**被调度上去。
- **limits** 是"**上限**"——运行时不能超。**超 CPU 会被限流（throttle）**（进程变慢但不死）；**超内存会被 Kill（OOMKilled）**（内存无法"限流"，只能杀）。

- **QoS 与驱逐** 你会不会在某天发现 Pod 被莫名赶走？这跟 **QoS（服务质量等级）** 有关。K8s 按 requests/limits 的设置给 Pod 分三级，节点资源紧张时**按等级从低到高驱逐**：

| QoS 等级 | 条件 | 驱逐优先级 |
|:----|:------|:------|
| **Guaranteed** | 每个容器 requests == limits | 最低（最后逐） |
| **Burstable** | 设置了但两者不等 | 中 |
| **BestEffort** | 完全没设置 | 最高（最先逐） |

所以生产的关键服务，要么设 Guaranteed，要么给足 requests，别让它在资源紧张时第一个被牺牲。

- **探针：判断 Pod"活着"和"能干活"** K8s 用三种探针来监控容器健康，它们的**失败后果不同**，用错会带来麻烦：

| 探针 | 判断 | 失败后果 |
|:----|:------|:------|
| **livenessProbe** | 进程还活着吗 | 重启容器 |
| **readinessProbe** | 准备好接流量了吗 | 从 Service 摘除（**不重启**） |
| **startupProbe** | 启动完成了吗 | 完成前不执行 liveness |

关键区别：**liveness 失败会重启**，所以它只该判断"进程是不是死了"。如果拿它去判断"依赖的数据库连不上"，就会导致 Pod 反复重启雪崩——这种情况应该用 readiness（摘流量，等依赖恢复后再接）。

### 2.4 网络与存储

这三块是 K8s 里"插件化"的部分，K8s 本身只定接口，实现交给你选：

| 模块 | 说明 | 常用 |
|:----|:------|:------|
| **CNI**（网络） | 给 Pod 分配 IP、打通跨节点通信 | Calico（网络策略强）/ Flannel（简单） |
| **CSI**（存储） | 把云盘/NFS 等挂给 Pod | 各云厂商 CSI 插件 |
| **NetworkPolicy** | 控制 Pod 间谁能访问谁 | 需 CNI 支持（Calico 可以） |

### 2.5 Helm

装一个复杂应用，往往要写一堆 YAML（Deployment、Service、Ingress、ConfigMap……）。**Helm 是把这些 YAML "模板化 + 参数化 + 版本化"的工具**，解决三个痛点：一套模板适配多环境（values 分层）、一次安装/升级/回滚（Release 版本化）、应用可打包分发（Chart）。

```bash
helm create myapp              # 生成 Chart 骨架
# myapp/
# ├── Chart.yaml                # 元信息
# ├── values.yaml               # 默认参数
# └── templates/                # 模板（deployment.yaml 等）

helm install myapp ./myapp \
  -f values-base.yaml \
  -f values-prod.yaml          # 后者覆盖前者
```

| 概念 | 说明 |
|:----|:------|
| **Chart** | Helm 包，含模板 + 默认 values |
| **values 分层** | base → env(dev/test/prod) → override，逐层覆盖 |
| **Release** | 一次安装的实例（同名可升级、可回滚） |
| **Repo** | Chart 的托管分发（Harbor / Chart Museum） |

### 2.6 多集群

当一个集群扛不住（跨地域、容灾、隔离）时，就需要多集群：

| 场景 | 方案 |
|:----|:------|
| 跨地域灾备 | 多集群部署，DNS/GSLB 分流 |
| 跨集群发布 | ArgoCD + ApplicationSet 统一下发 |
| 集群联邦 | KubeFed 跨集群资源同步 |
| 混合云 | 部分自建 K8s，部分公有云 |

```
                 ┌──────────┐
                 │  DNS/GSLB│
                 └────┬─────┘
           ┌──────────┴──────────┐
           ▼                     ▼
   ┌──────────────┐    ┌──────────────┐
   │  集群A (上海) │    │  集群B (新加坡) │
   │  ArgoCD同步   │    │  ArgoCD同步   │
   └──────────────┘    └──────────────┘
```

### 2.7 云平台

公有云把 K8s 打包成"托管服务"，省掉自己维护控制面的成本。

- **主流云厂商** | 云厂商 | K8s 服务 | 特点 |
|:----|:------|:------|
| 阿里云 | ACK | 国内份额第一，生态全 |
| 腾讯云 | TKE | 游戏/社交场景有优势 |
| 火山引擎 | VKE | 字节生态，性价比高 |
| AWS | EKS | 全球份额第一，服务最丰富 |
| Azure | AKS | 企业 Microsoft 生态 |
| GCP | GKE | K8s 原生体验最好 |

- **阿里云 ACK 核心服务** | 服务 | 说明 |
|:----|:------|
| **ACK** | 托管 K8s，支持专有版 / 托管版 / 边缘版 |
| **ACR** | 容器镜像服务（对标 Harbor，企业版支持扫描） |
| **SLB** | 负载均衡（对接 Ingress / LoadBalancer Service） |
| **OSS** | 对象存储（静态资源、日志归档） |
| **NAS** | 文件存储（多 Pod 共享） |
| **SLS** | 日志服务（采集 + 检索 + 告警） |
| **CMS** | 云监控（可对接 Prometheus） |

- **AWS EKS 核心服务** | 服务 | 说明 |
|:----|:------|
| **EKS** | 托管 K8s 控制面 |
| **ECR** | 容器镜像仓库 |
| **ELB** | 负载均衡（ALB Ingress Controller） |
| **S3** | 对象存储 |
| **CloudWatch** | 监控 + 日志 |
| **Route53** | DNS（多集群流量调度） |

- **多云管理** ```
                 ┌──────────┐
                 │  DNS/GSLB│
                 └────┬─────┘
           ┌──────────┴──────────┐
           ▼                     ▼
   ┌──────────────┐    ┌──────────────┐
   │  阿里云上海    │    │  AWS 新加坡   │
   │  ACK + OSS   │    │  EKS + S3    │
   └──────────────┘    └──────────────┘
```

| 维度 | 策略 |
|:----|:------|
| 流量调度 | DNS/GSLB 按地域分流，就近接入 |
| 数据同步 | 异步复制或数据库多地域 |
| CI/CD | ArgoCD + ApplicationSet 统一管理 |
| 监控 | 多集群统一 Prometheus / Grafana |
| 成本 | 按需选不同厂商的实例与计费 |
| 灾备 | 主备或双活，明确 RPO/RTO |

- **云原生服务对照** | 类型 | 厂商方案 | 开源方案 |
|:----|:------|:------|
| 对象存储 | OSS / S3 / COS | MinIO |
| 负载均衡 | SLB / ELB / CLB | Nginx / HAProxy |
| CDN | 全站加速 / CloudFront | 自建不推荐 |
| Serverless | 函数计算 / Lambda | Knative / OpenFaaS |
| 消息队列 | RocketMQ / SQS | Kafka / RabbitMQ |
| 数据库 | RDS / Aurora | 自建 MySQL / PostgreSQL |

---

## 三、CI/CD 持续集成交付

> 从"代码提交"到"部署上线"这条流水线，怎么自动化、怎么控风险。

## 3.1 流水线平台

CI/CD 平台是这条流水线的"发动机"。选型时主要看两点：**代码托管在哪**（决定集成成本），**流程有多复杂**（决定灵活性需求）。

| 平台 | 特点 | 适用场景 |
|:----|:------|:------|
| **Jenkins** | 最灵活，插件生态丰富，Pipeline as Code (Groovy) | 自建平台、复杂流程、嵌入式/汽车行业 |
| **GitLab CI** | 与 GitLab 深度集成，.gitlab-ci.yml 声明式配置 | 源码在 GitLab、中小团队 |
| **GitHub Actions** | GitHub 原生，Marketplace 生态好 | 开源项目、GitHub 为主的技术栈 |

Jenkins 的优势是"什么都能干"——尤其适合需要自定义构建环境（比如交叉编译、License 绑定的 Windows IDE）的场景，这也是嵌入式/汽车行业偏爱它的原因。

- **CI / CD / CD：三个词的区别** 这三个缩写常被混淆，其实是一条链上的三个位置：

- **CI（持续集成）**：开发者频繁把代码合入主干，每次合入都**自动构建 + 跑测试**，目的是"尽早发现集成问题"——不要等到发布前才发现三周前的改动冲突了。
- **CD（持续交付，Continuous Delivery）**：在 CI 基础上，保证**任何时刻都能一键发布**——但最后一步上线要**人工审批**。
- **CD（持续部署，Continuous Deployment）**：把最后一步也自动化，**过了门禁就自动发到生产**。

一句话记忆：**Delivery 与 Deployment 的区别，就在"最后一步是否人工"**。

- **Pipeline as Code** 把流水线定义写进代码仓库（Jenkinsfile / .gitlab-ci.yml），而不是在界面上点。好处很直接：**可评审**（走 Code Review）、**可追溯**（跟着代码版本走）、**可复用**（抽成模板）。

```groovy
// Jenkins 声明式 Pipeline
pipeline {
    agent any
    stages {
        stage('Checkout') { steps { git url: '...' } }
        stage('Build')    { steps { sh 'make' } }
        stage('Test')     { steps { sh './run_tests.sh' } }
        stage('Deploy')   { steps { sh 'kubectl apply -f deploy/' } }
    }
}
```

```yaml
# GitLab CI
stages: [build, test, deploy]

build:
  stage: build
  script: make
  artifacts:
    paths: [build/]        # 产物传给后续 stage

test:
  stage: test
  script: ./run_tests.sh

deploy:
  stage: deploy
  script: kubectl apply -f deploy/
  only: [main]             # 只在 main 分支执行
```

## 3.2 流水线设计

一条标准的 CI/CD 流水线，本质是**一串"检查点"**——每一步都是一道关，过不了就拦下：

```
代码提交 → 编译 → 静态检查 → 单元测试 → 镜像构建 → 部署到测试 → 验收测试 → 发布
```

| 阶段 | 内容 | 失败处理 |
|:----|:------|:------|
| 编译 | make / mvn / go build | 阻断，通知开发者 |
| 静态检查 | SonarQube / Cppcheck / lint | 设置质量门禁阈值 |
| 单元测试 | GTest / pytest / JUnit | 必须 100% 通过 |
| 镜像构建 | docker build + push | 阻断 |
| 部署测试 | 部署到 test 环境 | 人工介入 |
| 验收测试 | E2E / 冒烟测试 | 阻断发布 |
| 发布 | 金丝雀 / 滚动更新 | 自动回滚或人工决策 |

设计时有两条原则值得记住：**越早的阶段越快**（编译/静态检查要秒级，不能拖到开发者不耐烦）；**越靠近发布的阶段越谨慎**（增加人工审批、灰度、回滚预案）。

## 3.3 多环境管理

为什么要分多个环境？因为**风险要分级消化**：谁也不想让一个未经验证的改动直接面对真实用户。

| 环境 | 用途 | 数据 | 稳定性要求 |
|:----|:------|:------|:---:|
| **dev** | 开发调试、快速验证 | 造数/脱敏 | 低 |
| **test** | 集成测试、功能验证 | 造数/脱敏 | 中 |
| **staging** | 预发布、生产镜像验证 | 脱敏生产数据 | 高 |
| **production** | 真实用户服务 | 真实数据 | 最高 |

代码是**逐级晋级**的——在 dev 通过、去 test 集成、到 staging 压测、最后上 prod：

```
dev（开发自测）→ 提交 MR 触发 CI
  ↓
test（集成测试 + 自动化回归）
  ↓ 通过 → 晋级
staging（预发布验证 + 压测）
  ↓ 通过 → 发布审批
production（金丝雀 → 全量）
```

- **环境一致性的三条铁律** - **代码一致**：**同一个镜像**贯穿所有环境——绝不"在每个环境重新构建一次"，否则测的不是同一个东西。
- **配置分离**：环境差异只体现在 ConfigMap / Secret，镜像不变。
- **基础设施一致**：集群配置、节点规格尽量对齐，减少"在测试好好的，上生产就崩"。

```yaml
apiVersion: apps/v1
kind: Deployment
spec:
  template:
    spec:
      containers:
      - name: app
        image: myapp:v1.2.3           # 所有环境同一镜像
        envFrom:
        - configMapRef:
            name: app-config-{{env}}   # 按环境注入不同配置
```

- **环境隔离** 环境之间必须隔离，否则一个环境的操作会污染另一个：

| 维度 | 隔离方式 |
|:----|:------|
| 网络隔离 | 不同 Namespace / VPC / 网段 |
| 权限隔离 | RBAC：dev 只能操作 dev |
| 资源隔离 | 不同节点池 / 资源配额 |
| 数据隔离 | 独立数据库，禁止跨环境访问 |

- **多环境流水线** 用 `when` 控制"什么分支部署到什么环境"，是最常见的多环境编排方式：

```groovy
pipeline {
    stages {
        stage('Build') { steps { sh 'docker build' } }
        stage('Deploy Dev') { steps { sh 'kubectl apply -f dev/' } }
        stage('Deploy Test') {
            when { branch 'develop' }
            steps { sh 'kubectl apply -f test/' }
        }
        stage('Deploy Staging') {
            when { branch 'release/*' }
            steps { sh 'kubectl apply -f staging/' }
        }
        stage('Deploy Prod') {
            when { branch 'main' }
            input { message '确认发布到生产？' }    // 人工审批闸门
            steps { sh 'kubectl apply -f prod/' }
        }
    }
}
```

## 3.4 发布策略

发布策略要回答的核心问题：**怎么把新版本交给用户，同时把风险降到最低**。不同策略在这条"风险-成本"轴上各有位置。

| 策略 | 原理 | 风险 | 回滚速度 | 资源成本 | 适用场景 |
|:----|:------|:---:|:---:|:---:|:------|
| **滚动更新** | 逐个替换实例，分批完成 | 低 | 快 | 低 | 无状态服务（大部分场景） |
| **蓝绿部署** | 新旧两套完整环境，切流量 | 极低 | 秒级 | 高（双倍） | 有状态服务、大版本变更 |
| **金丝雀发布** | 小比例流量先验证，逐步放量 | 中 | 快 | 中 | 需要真实流量验证 |
| **A/B 测试** | 按用户特征分流，对比效果 | 低 | 快 | 中 | 产品决策、功能验证 |
| **影子发布** | 镜像生产流量但不影响用户 | 零 | 无需回滚 | 中 | 新架构验证、压测 |

- **滚动更新** K8s 默认策略：**逐个替换**旧实例，用 `maxSurge`（允许超出多少）和 `maxUnavailable`（允许少多少）控制节奏。优点是不需要额外资源、平滑；缺点是新旧版本会**短暂共存**，对版本兼容性有要求。

```
v1  v1  v1  v1          v1  v2  v1  v1         v2  v2  v2  v2
[ ] [ ] [ ] [ ]   →   [ ] [ ] [ ] [ ]   →   [ ] [ ] [ ] [ ]
```

- **蓝绿部署** 维护两套**完整环境**：老的 Blue 在跑，新的 Green 部署好待命，验证 OK 后**一次性把流量切过去**。回滚就是切回来，秒级。代价是**双倍资源**，而且要提前处理数据库这类有状态组件的兼容。

```
┌─────────────┐      切换 DNS/LB       ┌─────────────┐
│   🔵 Blue   │ ──────────────────→   │   🟢 Green  │
│   v1 运行中  │                        │   v2 就绪    │
└─────────────┘                        └─────────────┘
```

- **金丝雀发布** 名字来自"矿工带金丝雀下井试毒"。做法是**先放一小撮真实流量给新版本**，观察指标没问题再逐步放量：

```
10% 用户 → v2（观察 15 分钟）→ ✅
50% 用户 → v2（观察 15 分钟）→ ✅
100% 用户 → v2
```

关键观察指标：**错误率、延迟 P99、业务指标**（如下单成功率）。工具常用 Istio / Linkerd（流量分割）、Argo Rollouts。

- **怎么选：决策树** ```
需要按用户分流量验证产品效果？
  ├─ 是 → A/B 测试
  └─ 否 → 需要零风险切换？
            ├─ 是 → 蓝绿部署
            └─ 否 → 需要真实流量逐步验证？
                      ├─ 是 → 金丝雀发布
                      └─ 否 → 滚动更新（默认选择）
```

- **回滚策略** 再完善的发布也要有退路：

| 场景 | 回滚方式 |
|:----|:------|
| 滚动更新 | `kubectl rollout undo` |
| 蓝绿部署 | LB 切回 Blue |
| 金丝雀 | 流量权重调回 0%，或 rollout undo |
| 数据库变更 | **反向迁移脚本（必须有！）** |

数据库变更是最容易被忽略的回滚难点——代码能秒回，但改过的表结构/数据得有**反向迁移**才能回去。

- **发布 Checklist** - [ ] Code Review 通过
- [ ] CI 流水线全部绿灯
- [ ] 监控仪表盘已配置
- [ ] 告警规则已检查
- [ ] 回滚方案已确认
- [ ] 数据库迁移已准备（含反向）
- [ ] 通知已发出

### 3.5 GitOps

GitOps 的核心思想：**把 Git 当作集群状态的"唯一事实源"**。集群里跑什么，全写在一份 Git 仓库里；有个 Agent（如 ArgoCD）盯着这份仓库，发现集群实际状态和 Git 里声明的期望状态不一致，就自动把它拉回来。

| 工具 | 特点 |
|:----|:------|
| **ArgoCD** | K8s 原生，Web UI 强大，自动同步 |
| **Flux CD** | 轻量，Helm 集成好，Git 原语操作 |

```
开发者 Push 代码 → CI 构建镜像 → 更新部署仓库的 YAML → ArgoCD 检测变更 → 自动同步到集群
```

**四大原则**：声明式（描述期望状态）、版本化且不可变（存 Git、可追溯）、拉取式自动同步（控制器主动拉）、持续调谐（自动纠偏，即 Drift Detection）。

GitOps 与传统 CI/CD 最大的区别在**推送方向**：

| 模式 | 机制 | 优点 | 缺点 |
|:----|:------|:------|:------|
| **Push**（传统） | CI 直接 `kubectl apply` | 简单直观 | CI 得持有集群凭证，安全面大 |
| **Pull**（GitOps） | 集群内 Agent 拉 Git | 凭证不出集群，可审计、可回滚 | 需额外部署控制器 |

**为什么 Pull 更安全？** 传统模式下，CI 系统必须拿到集群的管理员凭证才能部署——一旦 CI 被攻破，集群就裸奔了。GitOps 反过来：部署凭证只存在集群内部的 Agent 手里，CI 只负责往 Git 推代码，攻击面小得多。

### 3.6 Repo — 多仓库管理

一个大型嵌入式项目，往往拆成**几十甚至上百个 Git 仓库**（BSP 层、中间件层、各应用层各自独立）。一个个 `git clone`/`git pull` 显然不现实——**Repo** 就是 Google 为 Android/AOSP 开发的多仓库管理工具，专门解决这个问题。

它的核心是一份 **manifest（XML 清单）**：把所有仓库的地址、分支、路径写在一起，之后 `repo sync` 一键同步全部，`repo start` 在全部仓库创建同名分支，`repo upload` 批量提交评审。

| 概念 | 说明 |
|:----|:------|
| **manifest** | XML 清单，定义所有仓库的地址、分支、路径 |
| **default.xml** | 默认清单，描述整个项目的组成 |
| **repo sync** | 一键同步所有仓库到指定分支/标签 |
| **repo start** | 在所有相关仓库创建主题分支 |
| **repo upload** | 批量推送到 Gerrit 评审 |

```xml
<?xml version="1.0" encoding="UTF-8"?>
<manifest>
  <remote name="origin" fetch="ssh://git@gerrit.example.com:29418/" />
  <default revision="main" remote="origin" sync-j="4" />   <!-- 并行 4 路同步 -->

  <project path="bsp/mcu-a"      name="platform/bsp/mcu-a"      revision="v2.1.0" />
  <project path="bsp/mcu-b"      name="platform/bsp/mcu-b"      revision="v2.1.0" />
  <project path="middleware/someip" name="middleware/someip" />
  <project path="apps/cluster"   name="apps/cluster" />
  <project path="apps/ivi"       name="apps/ivi" />
  <project path="tools/build"    name="tools/build" />
</manifest>
```

```bash
repo init -u ssh://git@gerrit/project/manifest.git -b main  # 初始化（拉取 manifest）
repo sync                                                    # 同步所有仓库
repo start feature-xxx --all                                 # 全部仓库开分支
repo status                                                  # 查看全部变更
repo upload                                                  # 批量提交 Gerrit 评审
repo forall -c 'git clean -fdx'                              # 对所有仓库执行命令
```

和 Git submodule 的区别：submodule 是"每个仓库各自记录子仓库的指针"，逐个更新很繁琐；Repo 是"用一份清单统一管理"——**适合"仓库多且需要整体协同"的场景**（这也是汽车/AOSP 类项目的标配）。

### 3.7 Gerrit — 代码评审系统

Gerrit 和 GitLab MR 最大的不同在**评审粒度**：GitLab 评审的是"一个分支/MR"，而 Gerrit 评审的是**单个 Commit**。每个 Commit 进 Gerrit 后是一个独立的"Change"，靠 `Change-Id` 跟踪。

| 维度 | Gerrit | GitLab MR |
|:----|:------|:----------|
| 评审粒度 | 单个 Commit | 整个分支/MR |
| 工作流 | Commit → Review → amend 重提 → +2 → Submit | Push 分支 → MR → Review → Merge |
| 权限模型 | 细粒度：+1 / +2 分权 | 相对粗粒度 |
| 适用场景 | 大型开源项目（AOSP）、汽车行业 | 通用企业 |
| 学习曲线 | 陡峭 | 平缓 |

Gerrit 的权限模型值得留意：**Code-Review +1** 表示"我看过，认可但不够格合并"，**+2** 才是"批准合并"；CI 回写的是 **Verified +1/-1**（验证通过与否）。两套独立的分数，把人评审和机器验证分开。

```
开发者 git commit → repo upload
    ↓
Gerrit 接收为一个 Change
    ↓
CI 触发验证（基于 refs/changes/xx/yy/zz 这个特殊 ref）
    ↓
CI 回写 Verified +1 / -1
    ↓
人工 Review（Code-Review +2）
    ↓
Submit → 合入目标分支
```

CI 通过 Jenkins 的 Gerrit Trigger 插件监听变更：

```groovy
pipeline {
    agent any
    triggers {
        gerrit(
            project: 'platform/bsp/mcu-a',
            branch: 'refs/heads/main',
            triggerOn: [
                patchsetCreated(approvalCategory: 'Code-Review', approvalValue: '+1')
            ]
        )
    }
    stages {
        stage('Verify') {
            steps {
                sh 'cmake -B build && cmake --build build'
                sh 'ctest --test-dir build --output-on-failure'
            }
        }
    }
}
```

### 3.8 KubeSphere — 容器平台

KubeSphere 是在原生 K8s 之上**加了一层"好用"**的平台。原生 K8s 功能强但门槛高（RBAC 要手写、监控日志要自己搭、流水线要自己装 Jenkins），KubeSphere 把这些打包成了开箱即用的一套。

| 模块 | 功能 |
|:----|:------|
| **多租户管理** | 企业空间 → 项目 → 资源三级隔离，比原生 RBAC 友好 |
| **DevOps 流水线** | 内置 Jenkins 引擎，可视化编排 |
| **灰度发布** | 蓝绿 / 金丝雀 / 流量镜像一键配置 |
| **微服务治理** | 基于 Istio / Spring Cloud 的服务网格 |
| **应用商店** | 基于 Helm 的应用模板市场 |
| **可观测性** | 内置 Prometheus + Grafana + EFK 日志 |

```
拉取代码 → 单元测试 → 代码分析 → 镜像构建 → 推送仓库 → 部署到开发环境 → 部署到生产
```

KubeSphere 把 Jenkins Pipeline 封装成可视化编排，支持凭证管理、构建节点池、流水线模板——**降低了维护 Jenkins 的成本**。

| 场景 | KubeSphere 的优势 |
|:----|:------|
| 团队权限管理 | 企业空间/项目/角色体系，无需手写 RBAC |
| 流水线 | 开箱即用的 DevOps 引擎 |
| 日志/监控 | 内置可观测性，省去自建 EFK/Prometheus |
| 应用发布 | 灰度/蓝绿/金丝雀一键配置 |

### 3.9 多技术栈统一流水线设计

现实中的团队往往要同时维护 C++ / Java / Go / Python 多种技术栈，如果每个栈一套流水线，维护成本会失控。统一流水线的思路是**抽公共、分差异**：

```
              ┌─────────────────────┐
              │    代码仓库层        │
              │ Git / Repo / Gerrit │
              └─────────┬───────────┘
                        │
        ┌───────────────┼───────────────┐
        ▼               ▼               ▼
  ┌──────────┐  ┌──────────┐  ┌──────────┐
  │ C++ 构建 │  │ Java构建 │  │ Go/Py    │
  │ CMake    │  │ Maven    │  │ go build │
  │ Conan    │  │ Gradle   │  │ pip      │
  │ 交叉编译 │  │ Spring   │  │          │
  └────┬─────┘  └────┬─────┘  └────┬─────┘
       │             │             │
       └─────────────┼─────────────┘
                     ▼
           ┌──────────────────┐
           │    质量门禁       │      ← 公共环节（统一）
           │ SonarQube/Trivy  │
           └────────┬─────────┘
                    ▼
           ┌──────────────────┐
           │  制品 & 部署      │      ← 公共环节（统一）
           │ Nexus + K8s      │
           └──────────────────┘
```

**公共环节**（拉代码、质量门禁、制品归档、部署）抽成 Shared Library，各语言栈只写自己的"构建 + 测试"差异部分——这样加一个新项目，只需填几个参数即可复用整条流水线。

---

## 四、制品与配置管理

> 代码写完之后，怎么管分支、怎么存产物、怎么管配置和密钥。

### 4.1 代码管理

- **分支策略：三种主流玩法** 分支策略的选择，本质是在"**发布节奏的确定性**"和"**集成的频繁度**"之间做取舍。

**Git Flow** 是最"重"的一套：有 `main`（生产）、`develop`（集成分支）、`feature/*`（功能）、`release/*`（发布准备）、`hotfix/*`（紧急修复）五类分支。它适合**有明确版本发布周期**的传统项目——缺点是分支多、合并路径长、容易"合并地狱"。

**Trunk-Based** 是最"轻"的：所有人往一条主干（main）上提交，分支存活时间很短（几小时到一两天）。它要求**自动化测试足够强**（否则不敢直接合主干），但配合持续部署最顺。

**GitHub Flow** 介于两者之间：`main` + 短生命周期的 feature 分支，提 PR、合并即部署。简单项目、SaaS 产品常用。

| 策略 | 特点 | 适用 |
|:----|:------|:------|
| **Git Flow** | main / develop / feature / release / hotfix | 版本发布周期明确的传统项目 |
| **Trunk-Based** | 全部提交到 main，短分支快速合并 | 持续部署、微服务 |
| **GitHub Flow** | main + feature 分支，PR 合并即上线 | 简单项目、SaaS |

- **Code Review 流程** Code Review 不只是"找人看一眼代码"，它是**知识共享 + 质量兜底**的机制。典型流程是：开 feature 分支 → 提交 MR/PR → CI 自动跑（编译 + 测试 + 扫描）→ 人工 Review 通过 → 合并 → 自动部署。关键点是**CI 先过、人再看**——机器能查的（能不能编译、测试过不过）别浪费人的时间。

```
开发者创建 Feature 分支 → 提交 MR/PR
    → CI 自动跑（编译 + 测试 + 扫描）
    → 至少 1 人 Review + Approve
    → 合并到目标分支
    → 自动部署
```

### 4.2 制品仓库

**为什么需要制品仓库？** 因为"构建产物"和"开源依赖"都需要一个**统一、可信、可追溯**的存放点。它能带来四件事：团队共享（不用人人往外网拉）、加速构建（代理缓存）、离线可用（外网断了也能构建）、版本唯一可追溯（哪个镜像对应哪个 commit）。

- **制品类型与对应仓库** 不同形态的产物，习惯上放在不同的仓库里：

| 制品类型 | 仓库 | 说明 |
|:----|:------|:------|
| **Docker 镜像** | Harbor / Docker Hub | 容器镜像专用，支持安全扫描、镜像复制 |
| **Java jar/war** | Nexus / JFrog Artifactory | Maven/Gradle 依赖 + 构建产物 |
| **npm / pip / Go mod** | Nexus / Artifactory | 前端/脚本语言依赖代理缓存 |
| **通用二进制** | Nexus Raw / Artifactory Generic | .bin / .elf / .hex / .zip / .tar.gz |
| **Helm Chart** | Harbor / Chart Museum | K8s 应用包管理 |

- **代理 / 宿主 / 聚合：Nexus 的三种仓库** 理解 Nexus 的关键是这三种仓库类型的配合：

- **Proxy（代理仓）**：它不存你的东西，而是**代理远程上游**（Maven Central、PyPI、npm）。第一次拉某依赖时它去上游下载并缓存，之后团队再拉就走本地缓存——**既加速，又在外网断开时仍可用**。
- **Hosted（宿主仓）**：真正**存放你自己制品**的地方（内部的 releases / snapshots）。
- **Group（聚合仓）**：把上面几个仓库**聚合成一个 URL**，对外只暴露一个地址。客户端配上它，就能同时拿到"代理来的开源依赖"和"宿主仓里的私有包"。

| 类型 | 角色 | 举例 |
|:----|:------|:------|
| **Proxy** | 缓存远程上游，加速 + 断网可用 | 代理 Maven Central / PyPI |
| **Hosted** | 存放私有制品（可写） | 内部 releases / snapshots |
| **Group** | 多个仓库聚合成单一 URL | maven-public = proxy + releases + snapshots |

- **Harbor — 容器镜像仓库** Harbor 是"给企业用的镜像仓库"，相比裸的 Docker Registry，多了几样关键能力：

| 功能 | 说明 |
|:----|:------|
| **镜像管理** | Push / Pull / Tag / 删除 / 保留策略 |
| **安全扫描** | 内置 Trivy，推送即扫漏洞 |
| **访问控制** | 项目级 RBAC，支持 LDAP/OIDC 认证 |
| **镜像复制** | 跨数据中心 / 跨云同步（配合多集群） |
| **垃圾回收** | 清理未引用的镜像层，省空间 |

```bash
docker build -t myapp:v1.2.3 .
docker tag  myapp:v1.2.3 harbor.example.com/prod/myapp:v1.2.3
docker push harbor.example.com/prod/myapp:v1.2.3
```

**镜像保留策略**很实用：保留最近 30 天 + 最近 10 个版本，生产标签永久保留——否则仓库很快会被历史镜像撑爆。

- **版本管理：语义化版本** 版本号不是随便编的，`v<MAJOR>.<MINOR>.<PATCH>` 每一段都有含义：

```
v1.2.3          → 正式版本
v1.2.3-rc1      → 候选版本（Release Candidate）
v1.2.3-SNAPSHOT → 开发快照
```

- **MAJOR**：有**不兼容**的变更时 +1（用的人可能要改代码）
- **MINOR**：**向后兼容**地新增功能时 +1
- **PATCH**：**向后兼容**地修 bug 时 +1

| | Release | Snapshot |
|:----|:---:|:---:|
| 版本 | 固定，不可变 | 可变，可覆盖 |
| 仓库 | releases | snapshots |
| 场景 | 正式上线 | 开发联调 |

**为什么 Release 必须不可变？** 因为下游依赖它。如果 `v1.2.3` 的内容能被悄悄改掉，那"我测的是 v1.2.3"就失去了意义——所以正式版本一旦发布就不能覆盖，要改只能发新版本。

- **制品元数据与安全** 一个好的制品，除了文件本身，还要带上**溯源信息**：Git Commit、CI 任务号、构建时间、校验和。

```
制品名: myapp-v1.2.3.jar
Git Commit:  a1b2c3d
CI Job:      build #1234
构建时间:    2026-07-19 10:00:00
签名:        myapp-v1.2.3.jar.sha256
```

制品安全措施：

| 措施 | 说明 |
|:----|:------|
| **访问控制** | 读/写分离（CI 写、部署读） |
| **镜像扫描** | 推送时自动扫描，高危阻止部署 |
| **签名校验** | 制品签名 + 部署时验证，防篡改 |
| **网络隔离** | 仓库不直接暴露公网 |
| **审计日志** | 记录所有 Pull/Push |

- **CI/CD 集成** ```groovy
pipeline {
    environment {
        HARBOR = 'harbor.example.com'
        NEXUS  = 'nexus.example.com'
    }
    stages {
        stage('Build') {
            steps {
                sh 'mvn clean package'
                sh 'docker build -t ${HARBOR}/prod/myapp:${BUILD_TAG} .'
            }
        }
        stage('Push') {
            steps {
                sh 'mvn deploy'
                sh "docker push ${HARBOR}/prod/myapp:${BUILD_TAG}"
            }
        }
        stage('Deploy') {
            steps {
                sh "kubectl set image deployment/myapp myapp=${HARBOR}/prod/myapp:${BUILD_TAG}"
            }
        }
    }
}
```

## 4.3 配置管理

配置管理的核心目标：**把"会变的东西"从"不变的东西"里剥离出来**——镜像保持一样，环境差异靠配置注入。

| 工具 | 特点 |
|:----|:------|
| **ConfigMap / Secret** | K8s 原生，适合容器化环境 |
| **Apollo / Nacos** | 配置中心，支持动态刷新、灰度、权限管理 |
| **Vault** | 密钥管理，动态生成数据库凭证、加密存储 |

- **各种方案怎么选** | 方案 | 存储 | 动态刷新 | 适用 |
|:----|:------|:---:|:------|
| **环境变量** | 进程环境 | 需重启 | 简单应用 |
| **配置文件** | 挂载卷 | 需重启 / watch | 传统应用 |
| **ConfigMap/Secret** | K8s 对象 | 挂载可热更新 | 容器化 |
| **配置中心** | 独立服务 | 支持 | 微服务、多环境 |
| **Vault** | 加密后端 | 支持 | 密钥/证书/动态凭据 |

三条原则：敏感信息用 Secret（别硬编码）、按环境分层（base → dev → staging → prod）、配置变更要留审计。

- **密钥泄漏的应急处理** 万一发现密钥泄漏了，按这四步走：

1. **立即轮换（rotate）**：不管是不是真的泄露，先换掉——密钥的价值会随时间衰减。
2. **排查暴露面**：`git log -S "secret"` 翻历史、扫镜像层、查 CI 日志，找出**所有**泄漏点。
3. **上线扫描门禁**：用 gitleaks/trufflehog 在提交时拦截。注意**默认规则有盲区**——裸的长 hex 串、以及不扫 `.git/config`（remote URL 里可能内嵌 token）。
4. **复盘**：迁移到 Vault / CI 凭证，禁止明文入库。

> 验证过程见「Jenkins 流水线实战」中 gitleaks 盲区一节。

## 4.4 项目管理

| 工具 | 用途 |
|:----|:------|
| **Jira** | 需求、Bug、版本规划 |
| **Confluence** | 技术文档、设计决策 |
| **飞书文档** | 团队协作、轻量文档 |

## 4.5 Java 构建体系（Maven / Gradle）

Java 有两大构建工具，理解它们的差别有助于选型。

- **Maven：约定优于配置** Maven 的哲学是"**约定优于配置**"——你按它规定的目录结构放代码，就不用写太多配置。它用 `pom.xml` 描述项目，用 **GAV 坐标（GroupId:ArtifactId:Version）** 唯一定位一个制品。Maven 还有一套固定的**生命周期**：clean → compile → test → package → verify → install → deploy，执行某个阶段会自动执行它之前的所有阶段。

| 概念 | 说明 |
|:----|:------|
| **pom.xml** | 项目对象模型，定义依赖、插件、构建配置 |
| **坐标 GAV** | GroupId:ArtifactId:Version 唯一标识制品 |
| **生命周期** | clean → compile → test → package → verify → install → deploy |
| **依赖管理** | 传递依赖、依赖范围（compile/test/provided）、冲突仲裁 |
| **多模块** | 父子 POM 聚合构建，统一管理版本 |

- **Gradle：灵活 + 快** Gradle 更像"**可编程的构建工具**"：构建脚本本身就是代码（Groovy/Kotlin DSL）。它的基本单位是 **Task**，比 Maven 的固定阶段灵活得多。最大的性能优势是**增量构建**——只跑真正需要重跑的 Task。`gradlew`（Wrapper）能锁定 Gradle 版本，团队成员不用各自装。

| 概念 | 说明 |
|:----|:------|
| **build.gradle(.kts)** | Groovy / Kotlin DSL 构建脚本 |
| **Task** | 构建的基本单元，比 Maven 阶段灵活 |
| **依赖配置** | implementation / api / compileOnly / testImplementation |
| **增量构建** | 自动跳过未变更的 Task（比 Maven 快） |
| **Gradle Wrapper** | gradlew 脚本，锁定版本，无需预装 |

- **CI/CD 中的 Java 构建** ```groovy
stage('Build Java Service') {
    steps {
        sh 'mvn clean package -DskipTests'      // Maven
        // sh './gradlew clean build -x test'   // Gradle
    }
}
stage('Test') {
    steps {
        sh 'mvn test'                   // Maven
        // sh './gradlew test'          // Gradle
    }
    post {
        always {
            junit '**/target/surefire-reports/*.xml'   // Maven
            // junit '**/build/test-results/**/*.xml'  // Gradle
        }
    }
}
stage('Publish') {
    steps {
        // Maven: 发布到 Nexus
        sh 'mvn deploy -DaltDeploymentRepository=nexus::default::https://nexus.example.com/repository/maven-releases/'
        // Gradle: 用 maven-publish 插件
        // sh './gradlew publish'
    }
}
```

### 4.6 C++ 依赖管理（Conan / vcpkg）

C++ 长期没有统一的依赖管理方案，这是它相比其他语言的"历史欠账"。传统做法是 vendoring（把源码拷进项目）或 Git Submodule，问题是**版本漂移、更新困难、二进制不兼容**。现代方案（Conan / vcpkg）借鉴了其他语言包管理器的思路：**声明式依赖 + 版本锁定 + 二进制缓存**。

| 方式 | 工具 | 问题 |
|:----|:------|:------|
| 传统 | vendor 目录、Git Submodule | 版本漂移、更新困难、二进制不兼容 |
| 现代 | Conan / vcpkg | 声明式依赖、版本锁定、二进制缓存 |

- **Conan** Conan 用 `conanfile.py` 声明依赖，并支持 **profile**（描述目标平台/编译器）——这对交叉编译特别有用。

```python
# conanfile.py
from conan import ConanFile

class MyProject(ConanFile):
    name = "my-app"
    version = "1.0"
    settings = "os", "compiler", "build_type", "arch"

    def requirements(self):
        self.requires("boost/1.84.0")
        self.requires("protobuf/3.21.12")
        self.requires("gtest/1.14.0")

    def build(self):
        cmake = CMake(self)
        cmake.configure()
        cmake.build()
```

```bash
# CI 中使用（交叉编译到 ARM 裸机）
conan install . --profile=arm-none-eabi-gcc --build=missing
cmake --preset conan-release
cmake --build --preset conan-release
```

- **vcpkg** 微软维护，与 CMake 原生集成。它有两种模式：**经典模式**（全局安装到 vcpkg 目录）和 **Manifest 模式**（项目级 `vcpkg.json` 声明依赖，可跟着代码走）——CI 里推荐后者，因为依赖清单可版本化。

```json
// vcpkg.json
{
  "name": "my-app",
  "version": "1.0",
  "dependencies": ["boost-asio", "protobuf", "gtest"]
}
```

```bash
cmake -B build -DCMAKE_TOOLCHAIN_FILE=$VCPKG_ROOT/scripts/buildsystems/vcpkg.cmake
cmake --build build
```

- **CI/CD 最佳实践** | 实践 | 说明 |
|:----|:------|
| **依赖锁定** | conan.lock / vcpkg baseline commit → 可复现构建 |
| **二进制缓存** | 自建 Conan/Artifactory 远端，避免重复编译 |
| **交叉编译 Profile** | ARM GCC / RISC-V / 多工具链预设 |
| **SBOM 生成** | CycloneDX 格式，满足合规要求 |

---

## 五、质量与安全

> 质量不是测出来的，是"卡"出来的；安全不是补出来的，是"左移"进流程的。

### 5.1 静态分析

静态分析的意思是"**不运行代码，只看代码**"就能发现问题。它是质量门禁里成本最低、反馈最快的一环——编译还没出结果，问题就已经报出来了。

先分清几个容易混的概念：**SAST**（静态应用安全测试）扫**源码**，是白盒；**DAST**（动态）打**运行中的应用**，是黑盒；**SCA**（软件成分分析）扫的是**第三方依赖**的成分和漏洞。三者互补，不重叠。

| 工具 | 语言 | 说明 |
|:----|:------|:------|
| **SonarQube** | 多语言 | 综合质量平台，技术债务度量 |
| **Cppcheck** | C/C++ | 内存泄漏、越界、未初始化检测 |
| **ESLint** | JavaScript/TypeScript | 代码风格 + 潜在错误 |
| **Checkstyle** | Java | 代码规范检查 |
| **MISRA-C** | C | 汽车行业安全编码标准 |

- **误报与漏报：静态分析的两难** 静态分析永远做不到"完美"——它总是要在这两个错误之间权衡：

| 概念 | 含义 | 应对 |
|:----|:------|:------|
| **误报（FP）** | 报了但实际没问题 | 抑制规则（suppress）、基线 |
| **漏报（FN）** | 真有问题但没报 | 多工具互补、自定义规则 |

**误报**多了，开发者就会失去信任、直接无视告警（"狼来了"）；**漏报**多了，工具就形同虚设。工程落地的三个要点：**基线化**（存量问题不阻断、只拦新增，否则老项目一上来就被几千条告警劝退）、**分级**（严重级硬门禁、一般级人工确认）、**持续校准**（每次漏检就补一条规则，让工具越用越准）。

- **SonarQube 质量门禁** SonarQube 把"什么样的代码算合格"量化为一张门禁表：

```
覆盖率 < 80%         → ❌ 阻断
新增 Bug > 0         → ❌ 阻断
代码异味 > 阈值       → ⚠️ 警告
安全漏洞 > Critical  → ❌ 阻断
```

注意这里用常是"**新增** Bug"而非"总 Bug"——这正是基线化思路的体现。

```groovy
stage('Static Analysis') {
    steps {
        sh 'cppcheck --enable=all --xml src/ 2> cppcheck.xml'
        sh 'sonar-scanner -Dsonar.projectKey=myapp'
    }
}
```

### 5.2 单元测试

单元测试是"**对最小代码单元（函数/类）的自动化验证**"。它跑得快、定位准——某个测试挂了，基本就锁定在那几行代码。

| 框架 | 语言 | 特点 |
|:----|:------|:------|
| **GTest** | C++ | Google 出品，xUnit 风格 |
| **JUnit** | Java | 最成熟 |
| **pytest** | Python | 简洁，fixture 机制强大 |

- **测试金字塔：为什么单测要多** ```
        ╱╲          E2E / UI 测试（少、慢、贵、脆）
       ╱  ╲
      ╱────╲        集成测试（适中）
     ╱      ╲
    ╱────────╲      单元测试（多、快、便宜、稳）
```

金字塔的含义是**越底层越多**：单测数量最多（快、稳定、定位准），E2E 最少（慢、容易因环境波动失败、只验证关键路径）。如果反过来搞"倒金字塔"（全堆 E2E），结果是 **CI 一跑半小时、偶尔随机挂、挂了还查不出哪儿错**——大家都开始不信任测试。

- **覆盖率：三种口径** "覆盖率"其实有好几种，严格程度递增：

| 指标 | 含义 | 严格度 |
|:----|:------|:---:|
| **行覆盖 (LC)** | 执行到的代码行占比 | 低 |
| **分支覆盖 (BC)** | 每个 if/else 分支是否都走到 | 中 |
| **MC/DC** | 每个条件都能独立影响判定结果 | 高（汽车 ASIL 常用） |

MC/DC 最严——它要求每个条件（如 `a && b` 里的 a 和 b）都能**单独**决定整个判定的真假，因此常用于汽车这类高安全等级场景。

要提醒的是：**覆盖率高不等于质量高**——完全可以写一堆"调用了但没断言"的测试把覆盖率刷上去。但反过来，覆盖率低一定说明测试不足。

```groovy
stage('Unit Test') {
    steps {
        sh 'cmake --build build && cd build && ctest --output-on-failure'
    }
    post {
        always {
            junit 'build/test-results/*.xml'      // 把结果喂给 Jenkins
        }
    }
}
```

覆盖率工具：C/C++ 用 **gcov/lcov**，Java 用 **JaCoCo**，Python 用 **coverage.py**。

## 5.3 安全扫描

按"扫描对象"分，安全扫描有好几类，通常组合使用：

| 扫描类型 | 工具 | 说明 |
|:----|:------|:------|
| **SAST**（静态） | SonarQube / Semgrep | 源码级漏洞检测 |
| **DAST**（动态） | OWASP ZAP | 运行时攻击模拟 |
| **容器扫描** | Trivy / Clair | 镜像漏洞扫描 |
| **依赖扫描** | Snyk / OWASP Dependency-Check | 开源依赖漏洞 |
| **SBOM** | Syft / CycloneDX | 软件物料清单生成 |

```groovy
stage('Security') {
    steps {
        sh 'trivy image --severity HIGH,CRITICAL myapp:${BUILD_TAG}'
        sh 'dependency-check --project myapp --scan ./'
    }
}
```

## 5.4 行业专项

某些行业（尤其汽车）有强制的标准体系，做 CI 时必须把对应的检查固化进流水线：

| 领域 | 标准/工具 | 说明 |
|:----|:------|:------|
| **汽车** | MISRA-C | 嵌入式 C 安全编码规范 |
| **汽车** | ASPICE | 软件过程评估，需求-代码-测试追溯 |
| **功能安全** | ISO 26262 | 道路车辆功能安全标准 |
| **智驾** | HIL / SIL | 硬件在环 / 软件在环测试 |
| **智驾** | CANoe | 车载网络仿真与测试 |

## 5.5 质量门禁流水线设计

把上面这些检查**按顺序串成一条"关卡链"**，就是质量门禁流水线——每一关不过就拦下：

```
代码提交
  ↓
静态分析（SonarQube / Cppcheck）
  ↓ 通过 ← 质量门禁：覆盖率 ≥ 80%，无新增 Bug
单元测试（GTest / pytest）
  ↓ 通过 ← 必须 100%
安全扫描（Trivy / SAST）
  ↓ 通过 ← 无 HIGH/CRITICAL 漏洞
镜像构建 + 推送到测试仓库
  ↓
部署到测试环境
```

## 5.6 安全（DevSecOps）

传统做法是"**开发做完 → 测试 → 最后才做安全**"，结果往往在临近上线时才发现一堆问题，改都来不及。DevSecOps 的主张是：**安全嵌进 DevOps 的每个环节**。

```
传统：  开发 → 测试 → 安全（最后一刻）→ 上线
DevSecOps：安全贯穿全程
```

- **安全左移（Shift Left）** 核心思想：**越早发现问题，修复成本越低**。需求阶段发现是 1x，等到了生产环境可能要 100x（还得考虑数据泄露、公关危机）。

| 阶段 | 安全动作 |
|:----|:------|
| **设计** | 威胁建模（STRIDE）、安全需求评审 |
| **开发** | 安全编码规范、IDE 插件、Commit 扫描 |
| **构建** | SAST、依赖扫描、镜像扫描 |
| **测试** | DAST、渗透测试、模糊测试 |
| **部署** | 镜像签名校验、策略即代码（OPA） |
| **运行** | 运行时安全、入侵检测、WAF |

- **STRIDE 威胁模型** 做威胁建模时，STRIDE 提供了一个"清单式"的思路——按六类威胁逐个对照：

| 威胁 | 含义 | 典型防护 |
|:----|:------|:------|
| **S**poofing | 身份伪造 | 认证（MFA） |
| **T**ampering | 数据篡改 | 完整性校验（签名/MAC） |
| **R**epudiation | 抵赖 | 审计日志 |
| **I**nformation Disclosure | 信息泄露 | 加密 |
| **D**enial of Service | 拒绝服务 | 限流、冗余 |
| **E**levation of Privilege | 权限提升 | 最小权限、RBAC |

- **供应链安全** 现代软件的绝大部分代码来自第三方依赖，"**你依赖的依赖的依赖**"里任何一个出问题，都可能影响你——这就是供应链安全。

**SBOM（软件物料清单）** 的思路是"**先别说有没有漏洞，先把你用了什么列清楚**"：

```bash
syft myapp:v1.2.3 -o cyclonedx-json > sbom.json   # 生成 SBOM
grype sbom.json                                    # 扫 SBOM 里的已知漏洞
```

| 框架 | 说明 |
|:----|:------|
| **SLSA** | Google 提出的供应链安全等级（L1-L4） |
| **Sigstore** | 无密钥签名和验证（Cosign） |
| **in-toto** | 构建全链路的完整性验证 |

**镜像签名**解决的是"我能确认这个镜像就是我构建的那个、没被篡改吗"：

```bash
cosign sign   --key cosign.key myapp:v1.2.3    # 签名
cosign verify --key cosign.pub myapp:v1.2.3    # 验证
# 部署策略：只允许签名镜像
kubectl create secret generic cosign-pub --from-file=cosign.pub
```

依赖扫描工具：`npm audit`（JS）、`pip-audit`（Python）、`OWASP Dependency-Check`（Java）、`trivy fs`（通用）。

- **密钥管理** 密钥管理的理想形态是"**根本不存在长期密钥**"——用 Vault 这样的工具**动态生成临时凭据**：

```
App 启动 → 从 Vault 动态获取数据库凭据（有效期 1h）
  → 到期自动失效，App 重新获取
  → 无需把密码写死在 ConfigMap 里
```

| 功能 | 说明 |
|:----|:------|
| **静态密钥** | KV Store，加密存储 API Key、Token |
| **动态密钥** | 按需生成临时数据库凭据，到期自动回收 |
| **加密即服务** | 应用无需自己管理加密逻辑 |
| **PKI** | 内部 CA，自动签发和轮换 TLS 证书 |

四条原则：**最小权限**（每个密钥只给必需的）、**定期轮换**（设有效期，到期自动换）、**不硬编码**（禁止出现在代码/配置/镜像里）、**审计**（记录所有访问）。

- **访问控制** **RBAC**（基于角色的访问控制）的核心是"**只给人干活需要的权限**"：

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata: { name: pod-reader, namespace: production }
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list"]          # 只读，不给 write/delete
```

**堡垒机**解决的是"人怎么安全地访问服务器"：

| 功能 | 说明 |
|:----|:------|
| 统一入口 | 所有 SSH/RDP 经过堡垒机，不直连 |
| 操作审计 | 命令录制、会话回放 |
| 权限管控 | 谁能在什么时间访问什么机器 |
| 国产推荐 | JumpServer |

身份认证协议：**LDAP**（企业目录/AD）、**OIDC/OAuth2**（现代 SSO）、**SAML**（企业级 SSO）。

- **运行时安全** 代码跑起来之后，还需要防"运行时被搞"：

| 工具 | 场景 |
|:----|:------|
| **Falco** | K8s 运行时异常行为检测（如容器内起 shell） |
| **OPA / Gatekeeper** | 策略即代码，阻止不合规部署 |
| **WAF** | Web 应用防火墙（ModSecurity / Coraza） |
| **Pod Security Standards** | K8s 原生 Pod 安全策略 |

- **合规** 不同行业有各自的合规要求，常见的有：

| 标准 | 适用场景 |
|:----|:------|
| **SOC2** | SaaS 服务安全认证 |
| **ISO 27001** | 信息安全管理系统 |
| **等级保护** | 国内合规强制要求 |
| **PCI DSS** | 支付卡行业数据安全 |
| **GDPR** | 欧盟数据保护法规 |

- **安全 Checklist** - [ ] CI 流水线集成 SAST + 依赖扫描
- [ ] 镜像推送前 Trivy 扫描，高危阻断
- [ ] 所有生产镜像签名验证
- [ ] 密钥使用 Vault 管理，不硬编码
- [ ] 访问使用堡垒机 + RBAC
- [ ] 生产 K8s 启用 Pod Security Standards
- [ ] 定期渗透测试和安全审计

---

## 六、可观测性

> 系统出问题时，你能不能"看懂"它？——这就是可观测性要回答的。

## 6.1 三大支柱

先分清"监控"和"可观测性"。**监控**是"**对已知问题设好告警**"——比如"CPU 超 90% 就报警"，它只能覆盖你**预先想过**的问题。**可观测性**更进一步，是"**通过系统对外输出的信息，去推断你事先没想到的问题**"。前者监控"已知的未知"，后者探索"未知的未知"。

可观测性由三种数据支撑，各自回答不同的问题：

| 支柱 | 回答 | 工具 |
|:----|:------|:------|
| **指标 (Metrics)** | 系统是否正常？ | Prometheus + Grafana |
| **日志 (Logging)** | 出了什么问题？ | ELK / Loki / Splunk |
| **链路 (Tracing)** | 问题出在哪个环节？ | Jaeger / OpenTelemetry |

打个比方：**指标**是仪表盘（车速、油量），**日志**是行车记录仪（具体发生了什么），**链路**是"这单外卖从下单到送达经过哪几个环节"。三者配合，才能完整回答"服务为什么慢了"。

## 6.2 Prometheus — 指标监控

- **架构：拉模型** Prometheus 的架构有一个鲜明的特点——**它是主动去"拉"数据的**，而不是被动等别人"推"：

```
被监控目标（Exporter / App 暴露 /metrics）
    ↓ HTTP Pull（每 15s）
Prometheus Server（时序数据库 TSDB）
    ↓ PromQL 查询
Grafana（可视化）
    ↓ 告警规则
AlertManager（告警分发）
```

被监控方只需暴露一个 `/metrics` 端点，Prometheus 定期去抓，抓到的数据存进自己的时序库（TSDB）。

**为什么用 Pull 而不是 Push？** 好处有三：Prometheus 自己控制采集节奏、能直接感知目标"活着还是挂了"（拉不到就是异常）、调试方便（浏览器直接打开 `/metrics` 就能看）。缺点也很明确——**跑完就消失的短命任务**（如批处理脚本）来不及被拉，这种情况下用 **Pushgateway** 让它把数据先推过去，Prometheus 再从 Pushgateway 拉。

| 模式 | 代表 | 优点 | 缺点 |
|:----|:------|:------|:------|
| **Pull** | Prometheus | 服务发现、目标存活可感知、调试方便 | 短命任务难拉 |
| **Push** | Pushgateway / StatsD | 适合批处理/短任务 | 无法感知 target 存活 |

- **四种指标类型** Prometheus 只有四种指标类型，理解它们的语义很关键：

| 类型 | 说明 | 示例 |
|:----|:------|:------|
| **Counter** | **只增不减**（单调递增） | HTTP 请求总数、错误总数 |
| **Gauge** | 可增可减 | CPU 使用率、内存使用量 |
| **Histogram** | 分桶统计分布（服务端算分位） | 请求延迟 P50/P95/P99 |
| **Summary** | 客户端算分位 | 同上，但客户端计算 |

**Histogram vs Summary** 的区别常被追问：Histogram 把观测值丢进预设的"桶"里计数，因此**多个实例的数据可以聚合**（求和后再算分位）；Summary 是在客户端直接算好分位，**不可跨实例聚合**。所以集群场景更适合 Histogram。

- **PromQL 常用查询** Counter 类型的指标**不能直接看**（它只是个不断变大的数），必须用 `rate()` 等函数转成"速率"：

```promql
# 错误率（原始速率）
rate(http_requests_total{status=~"5.."}[5m])

# 错误率占比（更常用于告警）
sum(rate(http_requests_total{status=~"5.."}[5m])) / sum(rate(http_requests_total[5m]))

# P99 延迟
histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[5m]))

# CPU 使用率
100 - (avg(rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)

# 内存使用率
(1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes) * 100

# 各 Pod CPU 用量 Top 5
topk(5, sum by (pod) (rate(container_cpu_usage_seconds_total[5m])))
```

**`rate` vs `irate`**：`rate` 取整个时间窗口的平均速率（平滑，适合告警，不容易被瞬时尖峰误触发）；`irate` 只取最近两个数据点（敏感，适合实时观察）。记住：**Counter 必须配 `rate`/`increase` 才可读**。

- **高基数问题（Prometheus 的痛点）** Prometheus 按"标签组合"存时间序列。如果把**用户 ID、请求 ID、带参数的 URL** 这类"千变万化"的值做成标签，时间序列数量会爆炸（这就是**高基数**），内存很快被撑爆。

- **禁忌**：不要把高基数维度（user_id、request_id、带参数的 URL）做 label
- **对策**：用 Recording Rule 预聚合、采样、限制 label 数量
- **排查**：看 `prometheus_tsdb_head_series` 这个指标的时间序列数

## 6.3 Grafana — 可视化

Grafana 是"**把数据画成图**"的层，本身不存数据，只负责查询和展示：

| 要素 | 说明 |
|:----|:------|
| **Data Source** | 对接 Prometheus / Loki / Elasticsearch 等 |
| **Dashboard** | 一组 Panel 的集合，按业务/服务分组 |
| **Panel** | 单个图表（时序图、柱状图、饼图、仪表盘） |
| **Alert** | 基于 Panel 数据设置告警规则 |
| **Variable** | Dashboard 级动态筛选（集群、命名空间、服务名） |

`Variable` 这个功能很实用——一个看板通过下拉框切换"集群/命名空间/服务"，就不用每个维度各做一张。

## 6.4 日志管理

日志回答的是"**刚才到底发生了什么**"。主流有两套方案：

| 方案 | 组成 | 适用 |
|:----|:------|:------|
| **ELK** | Elasticsearch + Logstash + Kibana | 传统部署、海量日志、全文检索强 |
| **Loki** | Loki + Promtail + Grafana | K8s 原生，只索引标签、省资源 |
| **Splunk** | 商业产品 | 企业级合规需求 |

Loki 的思路很讨巧：**它只给日志的标签建索引，不索引日志正文**，因此比 ELK 省很多资源——K8s 环境下和 Prometheus 共用 Grafana，体验统一。

```
应用容器（stdout/stderr）
    ↓ Fluentd / Promtail 采集
日志存储（Elasticsearch / Loki）
    ↓ 检索 + 可视化
Kibana / Grafana
    ↓ 告警规则
AlertManager
```

**日志规范**：用**结构化日志（JSON）**而非纯文本，方便检索解析；关键字段固定（timestamp、level、service、`trace_id`、message）；错误日志必须带堆栈。其中 `trace_id` 尤其重要——它能把一条日志和链路追踪串起来。

```json
{
  "timestamp": "2026-07-19T10:00:00Z",
  "level": "ERROR",
  "service": "order-service",
  "trace_id": "abc123",
  "message": "payment timeout",
  "user_id": "u456"
}
```

## 6.5 链路追踪

微服务下，一个请求会穿过好几个服务。**链路追踪**给每个请求分配一个 **Trace ID**，记录它经过的每个环节和耗时——这样就能回答"这单请求为什么慢：到底卡在支付还是库存"。

| 工具 | 特点 |
|:----|:------|
| **Jaeger** | Uber 开源，UI 友好 |
| **OpenTelemetry** | CNCF 标准，厂商中立的采集 |
| **SkyWalking** | 国内流行，APM 全功能覆盖 |

```
用户请求 → API Gateway → 订单服务 → 支付服务 → 库存服务
               ↑            ↑           ↑           ↑
               └────────────┴───────────┴───────────┘
                        同一 Trace ID 串联
```

## 6.6 告警体系

告警的目标不是"报得越多越好"，而是"**报的每一条都值得人从床上爬起来**"。

- **告警分级** | 级别 | 说明 | 响应要求 | 示例 |
|:----|:------|:------|:------|
| **P0 紧急** | 服务全面不可用 | 5 分钟内响应 | 网站 502 |
| **P1 严重** | 核心功能受损 | 15 分钟内响应 | 支付成功率下降 |
| **P2 一般** | 非核心功能异常 | 1 小时内响应 | 某管理页面报错 |
| **P3 提示** | 潜在风险 | 工作时间处理 | 磁盘 80% 告警 |

- **好告警的标准（治理"告警疲劳"）** 告警太多，人就会麻木、直接无视——这叫**告警疲劳**。一条好告警应该：

| 原则 | 说明 |
|:----|:------|
| **可行动（actionable）** | 收到的人有明确动作可做 |
| **有上下文** | 带 Dashboard 链接、排查手册 |
| **不重复** | 同一根因的衍生告警要抑制/收敛 |
| **阈值合理** | 用 `for:` 消除毛刺；优先用业务指标而非纯技术指标 |

> **反例**：CPU 80% 就告警——CPU 高不等于有问题。更好的做法是告"**错误率上升**"或"**延迟超 SLA**"这类用户真正能感知的信号。

```yaml
groups:
  - name: app-alerts
    rules:
      - alert: HighErrorRate
        expr: rate(http_requests_total{status=~"5.."}[5m]) > 0.05
        for: 5m                    # 持续 5 分钟才告，避免毛刺
        labels: { severity: p0 }
        annotations:
          summary: "{{ $labels.service }} 错误率超过 5%"
```

- **告警治理** - **收敛**：相同告警合并，避免重复通知
- **静默**：维护窗口期抑制告警
- **升级**：P0 未确认则逐级上升（群消息 → 电话 → 上级）
- **值班轮转**：明确每个人负责的时间段

## 6.7 研发效能度量（DORA）

DORA（DevOps Research and Assessment）用**四个指标**来量化"一个团队的交付效能到底怎么样"：

| 指标 | 定义 | Elite 目标 |
|:----|:------|:------|
| **部署频率 (DF)** | 多久部署一次到生产 | 按需 / 每天多次 |
| **变更前置时间 (LT)** | 从 Commit 到上生产的时间 | < 1 小时 |
| **变更失败率 (CFR)** | 导致故障的部署占比 | < 15% |
| **恢复时间 (MTTR)** | 从故障到恢复的时间 | < 1 小时 |

- **四个指标背后的两个维度** ```
吞吐量（Throughput）：部署频率 DF ↑、变更前置时间 LT ↓
稳定性（Stability）： 变更失败率 CFR ↓、恢复时间 MTTR ↓
```

真正重要的是**吞吐与稳定的平衡**——只追"部署频率高"可能换来"故障频发"，那没意义。DORA 的价值在于**用数据暴露瓶颈**：是构建太慢？测试太长？还是回滚太麻烦？

> ⚠️ 注意：DORA 是**团队级诊断工具，不是个人考核指标**。一旦拿来考核个人，就会逼出"拆小部署刷频率""把失败藏起来"这类作弊行为，指标随即失真。

- **数据采集** ```
GitLab API → 取 Commit/MR 时间
Jenkins API → 取构建开始/结束时间
K8s API → 取部署时间、Pod 状态
Jira API → 取故障工单信息

→ 汇总到效能看板（Grafana Dashboard）
```

---

## 七、AI 赋能 DevOps（AIOps）

> 大模型怎么真正用在研发效能上——不是"把日志丢给 AI"，而是把 AI 嵌进流水线的每个环节。

## 7.1 AI Coding 提效

AI 写代码已经成了日常。当前主流工具各有侧重：

| 工具 | 特点 | 适用 |
|:----|:------|:------|
| **GitHub Copilot** | IDE 内嵌，补全 + Chat | 日常开发 |
| **Cursor** | AI-first IDE，基于 VS Code | 复杂重构 |
| **Claude Code** | 终端 Agent，端到端开发 | 自动化任务 |
| **Codex (OpenAI)** | 多步推理代码生成 | API 集成 |

在 DevOps 场景里，AI 的用武之地其实很明确——**那些"格式固定、但有套路"的活儿**：

| 场景 | 用法 |
|:----|:------|
| 编写流水线配置 | AI 辅助生成 .gitlab-ci.yml / Jenkins pipeline |
| Dockerfile 优化 | 分析镜像大小、层缓存、安全建议 |
| 脚本生成 | 生成日志分析脚本、部署脚本 |
| 故障排查 | 贴错误日志 → AI 分析根因并给修复建议 |
| 代码审查 | AI 自动 Review MR/PR，标记潜在问题 |

## 7.2 AIOps 核心理念

AIOps = **用 AI/ML 增强 IT 运维与 DevOps 流程**。它的价值取向很清楚：

| 概念 | 说明 |
|:----|:------|
| **AIOps** | 用 AI/ML 技术增强 IT 运维和 DevOps 流程的自动化与智能化 |
| **核心价值** | 从"被动响应"→"主动预测"；从"人工排查"→"智能诊断" |
| **在 DevOps 的定位** | 不替代现有工具链，而是在流水线每个环节嵌入 AI 能力 |

但这里有个关键认知：**AIOps ≠ 把日志丢给大模型**。工程化落地要守住三条底线：

- **确定性优先**：能用规则解决的（比如"编译失败=退出码非 0"）就别上 AI，AI 只处理规则覆盖不了的模糊问题；
- **证据可溯源**：AI 给的每个结论都要能回到具体的日志行或代码行，否则没人敢信；
- **人工兜底**：AI 只做"建议"，绝不直接阻断生产。

## 7.3 AI 辅助代码评审（AI Code Review）

- **工作流** ```
MR/PR 提交 → AI 自动分析 → 评论标注 → 人工复核 → 合并
```

- **AI 比 Linter 强在哪** 传统 Linter 查的是"格式和语法"，AI 的优势在于**理解语义**：

| 能力 | 描述 |
|:----|:------|
| **代码规范检查** | 识别不符合团队规范的代码（比 Linter 更智能，懂语义） |
| **逻辑缺陷检测** | 潜在空指针、资源泄漏、并发安全问题 |
| **安全漏洞扫描** | SQL 注入、XSS、硬编码密钥等 |
| **性能建议** | 算法复杂度、不必要的内存分配、IO 瓶颈 |
| **最佳实践推荐** | 结合上下文推荐设计模式、重构建议 |

- **落地工具** | 工具 | 特点 |
|:----|:------|
| **GitHub Copilot Code Review** | GitHub 原生集成，PR 自动审查 |
| **CodeRabbit** | AI 逐行评审，支持自定义规则 |
| **Amazon CodeGuru Reviewer** | AWS 生态，Java/Python 深度分析 |
| **自研方案** | LLM API + GitLab Webhook + 定制 Prompt（最灵活） |

- **自研 AI Code Review 架构** ```
GitLab Webhook (MR Event)
    ↓
AI Review Service（触发）
    ↓
拉取 Diff → 分块 → LLM 分析（逐文件）→ 生成 Review Comments
    ↓
GitLab API（发布评论到 MR）
    ↓
人工确认 + 学习反馈（评论的 resolved/ignored 状态回流优化 Prompt）
```

注意架构里的"**学习反馈**"闭环：人工把 AI 评论标记为"已解决"还是"忽略"，这些信号回流去优化 Prompt 和用例库——让 AI 越用越准。

- **关键 Prompt 设计** ```
你是资深 C++/嵌入式软件工程师，请审查以下代码变更：
- 关注内存安全、资源管理、并发正确性
- 遵循 MISRA-C 或团队编码规范
- 标注严重级别：🔴严重/🟡建议/🟢优化
- 给出具体的修复建议和示例代码
```

### 7.4 智能构建故障诊断

- **传统痛点** - 构建失败日志动辄上千行，人工定位根因耗时
- C++ 模板编译错误信息冗长难读（如 STL 错误）
- 依赖冲突、链接错误排查依赖人工经验
- 跨平台交叉编译错误模式多样

- **AI 能做什么** | 能力 | 实现 |
|:----|:------|
| **错误日志智能分类** | LLM 解析日志，自动归类（编译/链接/依赖/环境问题） |
| **根因分析（RCA）** | 多轮对话式排查：日志 → 可能原因 → 验证命令 → 进一步分析 |
| **修复建议生成** | 直接输出 fix 代码或配置变更（如 CMakeLists.txt 修正） |
| **历史故障匹配** | 向量化历史构建失败记录，相似度检索 → 推荐已知修复方案 |

- **日志预处理（工程关键）** **直接丢全量日志给 LLM 又贵又不准**——上下文塞满噪声，模型反而抓不住重点。要点：**截断 + 提取关键段**（错误行前后 N 行）、**去重**（相同错误只留一份）、**结构标记**（阶段/退出码/关键行）。

```python
# 简化：提取错误上下文
lines = log.splitlines()
hits = [i for i, l in enumerate(lines) if 'error:' in l.lower() or 'failed' in l.lower()]
context = []
for i in hits[:20]:
    context += lines[max(0, i-3):i+4]
prompt_log = '\n'.join(dict.fromkeys(context))  # 去重保留顺序
```

- **集成到 Jenkins Pipeline** ```groovy
stage('Build') {
    steps {
        script {
            try {
                sh 'cmake --build build'
            } catch (Exception e) {
                // 捕获构建失败 → 调用 AI 诊断服务
                def diagnosis = aiDiagnose(
                    log: currentBuild.rawBuild.getLog(100),
                    project: env.PROJECT_NAME,
                    branch: env.BRANCH_NAME
                )
                currentBuild.description = "🔴 构建失败 - ${diagnosis.category}"
                error(diagnosis.summary)
            }
        }
    }
}
```

## 7.5 流水线智能优化

| 方向 | 传统方式 | AI 优化 |
|:----|:------|:------|
| **构建加速** | 手动分析瓶颈 | AI 分析构建 Timeline，建议并行化/缓存策略 |
| **测试用例选择** | 全量运行 | 基于代码变更智能选择受影响测试（TIA） |
| **流水线编排** | 固定 Stage 顺序 | AI 分析依赖 DAG，自动优化并行度和顺序 |
| **资源调度** | 固定节点分配 | 根据历史负载预测，动态分配节点 |
| **失败预测** | 事后处理 | 基于代码变更特征预判构建/测试失败概率 |

- **测试影响分析（TIA）原理** TIA 是这里面**最实用、收益最直接**的一项——大型项目全量跑测试可能要几小时，但一次改动其实只影响一小部分测试：

```
MR 变更文件列表
    ↓
代码依赖图谱（调用链 + include 关系）
    ↓
AI 选择：仅运行受影响的测试用例（减少 70-90% 测试时间）
    ↓
小概率补充：随机采样 10% 非相关测试作为安全网
```

最后那句"随机采样做安全网"很重要——依赖图谱不一定完整，完全只跑"相关测试"有漏测风险，所以补一点随机测试兜底。

## 7.6 大模型工程化部署

- **三种架构选型** | 方案 | 适用场景 | 优缺点 |
|:----|:------|:------|
| **API 调用（DeepSeek/GPT/Claude）** | 快速验证、非敏感场景 | 简单但数据出境风险，成本不可控 |
| **私有化部署（vLLM/Ollama）** | 代码等敏感数据场景 | 数据安全，但需 GPU 资源 |
| **混合架构** | 通用走公有云，敏感走私有 | 兼顾成本和安全 |

- **RAG、微调、提示工程怎么选** 三种"让模型懂你的业务"的手段，成本依次递增：

| 手段 | 原理 | 适合 | 成本 |
|:----|:------|:------|:------|
| **提示工程** | 改 prompt / few-shot | 快速迭代、规则明确 | 低 |
| **RAG** | 检索外部知识注入上下文 | 知识频繁变（故障库/文档） | 中 |
| **微调（Fine-tune）** | 用数据继续训练模型 | 风格/领域习惯固化 | 高 |

**选型原则**：知识型需求（内部规范、历史故障）**优先 RAG**——因为知识更新快、需要可溯源、且不用重训；只有当"模型说话风格/领域习惯"本身要固化时，才考虑微调。

- **私有化部署方案** ```
┌────────────────────────────────────┐
│         AI DevOps 引擎              │
│                                    │
│  ┌──────────┐  ┌──────────────┐   │
│  │ 推理服务  │  │ 向量数据库    │   │
│  │ vLLM /   │  │ Milvus /     │   │
│  │ Ollama   │  │ ChromaDB     │   │
│  └──────────┘  └──────────────┘   │
│       │              │             │
│  ┌────┴──────────────┴─────┐      │
│  │     RAG 检索增强生成     │      │
│  │  历史故障库 + 技术文档    │      │
│  └──────────────────────────┘      │
│                                    │
│  能力矩阵:                          │
│  • 代码审查 Agent                  │
│  • 构建诊断 Agent                  │
│  • 文档生成 Agent                  │
│  • 知识问答 Agent                  │
└────────────────────────────────────┘
```

- **关键考量** | 维度         | 要点9                                             |
| :--------- | :---------------------------------------------- |
| **数据安全**   | 代码不进公有云，模型本地推理，审计日志全量留存                         |
| **模型选型**   | DeepSeek-Coder / CodeLlama / Qwen-Coder 等开源代码模型 |
| **RAG 增强** | 内部技术文档、历史故障库、代码规范向量化，增强上下文                      |
| **效果评估**   | 量化指标：修复建议采纳率、诊断准确率、审查误报率                        |

### 7.7 AIOps 落地路线图

落地讲究**从快赢到深水区**，别一上来就啃硬骨头：

```
Phase 1（快赢）: AI 代码评审接入 MR 流程
    → 接入 DeepSeek/GPT API，GitLab Webhook 集成
    → 目标：自动拦截 30% 常见问题

Phase 2（深水区）: 智能构建诊断
    → 构建日志 → LLM 分类 + RCA
    → 目标：MTTR 降低 50%

Phase 3（高阶）: 流水线智能优化 + 私有化
    → TIA 智能测试选择
    → 私有化部署代码大模型
    → 目标：CI 耗时降低 40%
```

### 7.8 开源 CI × LLM 工具地图（2026）

> 现有成熟度差异很大：**代码检视有成型的开源项目，流水线故障定位基本没有能直接用的开源方案**。
> 下面按「能不能今天就用上」分四级。

- **推荐等级说明** | 等级 | 含义 | 建议 |
|:---:|:---|:---|
| ⭐⭐⭐⭐⭐ | 成熟、可直接接入、社区活跃 | **立刻上手** |
| ⭐⭐⭐⭐ | 可用，需少量配置或自研配合 | **值得投入** |
| ⭐⭐⭐ | 有参考价值，但不完整 | 借鉴思路，不直接依赖 |
| ⭐⭐ | 概念好但落地差 / 商业产品开放接口 | 观望 |

- **一、代码检视** | 工具 | 开源程度 | 特点 | 等级 |
|:---|:---|:---|:---:|
| **PR-Agent**（CodiumAI） | **完全开源** | 自动生成 PR 描述 + 审查 + 改进建议；支持 GitHub/GitLab/Gitea；Docker 一条命令起 | ⭐⭐⭐⭐⭐ |
| **Open Code Review**（阿里） | 开源 | 规则引擎 + LLM Agent 混合架构；内置 NPE/线程安全/SQL 注入规则；支持私有化模型 | ⭐⭐⭐⭐ |
| **CodeRabbit** | 部分开源（自托管版） | 逐行评审 + 自定义规则，UI 体验好 | ⭐⭐⭐ |
| **CodeReview BOT** | Action 开源 | 基于 ChatGPT 的 GitHub Action，简单粗暴 | ⭐⭐⭐ |
| **Gerrit LLM 插件** | 插件开源 | 老牌审查工具 + AI 辅助 | ⭐⭐ |

**选型判断**

- **想快速见效**：PR-Agent，接入现有 GitLab 仓库半小时能出第一条评论
- **在乎误报率 / 要私有化**：Open Code Review 的规则层先过滤，LLM 只处理需要推理的部分
- **纯 LLM 通读 diff 的通病**：召回率高但误报多，人工复核成本可能抵消收益

- **二、流水线故障定位** | 方案 | 状态 | 等级 |
|:---|:---|:---:|
| **自研日志分析器**（Python + LLM API） | 完全开源，自己写 | ⭐⭐⭐⭐⭐ |
| **GitLab Duo Root Cause Analysis** | GitLab 商业版内置，社区版没有 | ⭐⭐ |
| **CircleCI MCP Server** | 商业产品的 MCP 接口，绑定 CircleCI | ⭐⭐ |
| **Harness AI DevOps Agent** | 商业产品，少量组件开源 | ⭐⭐ |

**结论：这一类没有能直接用的开源完整方案，自研反而是最优解。**

自研三步就够：

1. 拉日志（Jenkins API `/job/xxx/lastBuild/consoleText`，或 GitLab `jobs/:id/trace`）
2. **预处理**（关键）：截断噪声、提错误行前后 N 行、去重、标注阶段和退出码
3. 结构化 prompt → 输出「根因 / 证据行 / 修复建议」三段

```python
# 核心预处理逻辑（直接丢全量日志又贵又不准）
lines = log.splitlines()
hits = [i for i, l in enumerate(lines) if 'error:' in l.lower() or 'failed' in l.lower()]
context = []
for i in hits[:20]:
    context += lines[max(0, i-3):i+4]
prompt_log = '\n'.join(dict.fromkeys(context))   # 去重保序
```

- **三、流水线优化 / 测试选择** | 方向 | 代表方案 | 开源程度 | 等级 |
|:---|:---|:---|:---:|
| **测试影响分析（TIA）** | 基于代码依赖图选测试，非 LLM | 可自研 | ⭐⭐⭐⭐ |
| **Flaky 测试检测** | XGBoost / 历史统计，非 LLM | 可自研 | ⭐⭐⭐⭐ |
| **构建耗时分析** | Jenkins Build Time Analyzer 等插件 | 开源插件 | ⭐⭐⭐ |
| **流水线编排优化** | 各家 AI 编排框架（CrewAI 等包装） | 多为 prompt 包装 | ⭐⭐ |

> **注意**：TIA 和 Flaky 检测的收益主要来自「代码依赖分析 + 历史数据统计」，**不是靠 LLM**。
> 这两个方向的宣传里经常挂 AI 的名，实际核心是图算法和统计，别被概念带偏。

- **四、我的实操建议** | 优先 | 动作 | 预期收益 |
|:---:|:---|:---|
| 1 | 接 **PR-Agent** 到现有 GitLab 仓库 | 半小时起步，立刻有 AI 审查能力 |
| 2 | 写一个 **Jenkins 日志分析器**（~100 行 Python） | 补上「效能工具开发」缺口，面试可讲 |
| 3 | TIA 先做「简单版」：按文件路径映射测试集 | 省 CI 时间，不需要 LLM |
| 4 | 观察 **Open Code Review** 的混合架构思路 | 借鉴到自己的门禁设计里 |

- **五、几个容易踩的认知坑** 1. **「全链路 AI DevOps 平台」基本都是包装**：把几个 prompt 组合成 Agent 编排，实际效果不超过单点工具
2. **LLM 审查不能替代 Linter 和静态检查**：确定性规则该用规则，LLM 只处理规则覆盖不到的模糊判断
3. **故障诊断的瓶颈在日志预处理，不在模型**：全量日志丢给模型，准确率和成本双输
4. **不要为了「用上 AI」而替换已稳定的流程**：先证明 AI 在某一步比人工好，再引入

---

---

## 相关文档

- [[knowledge/devops/practices/README\|🛠️ 实战演练]] — 落地实操
- [[knowledge/devops/industry/00-行业对比\|🏭 行业对比]] — 行业方向
