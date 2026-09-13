---
{"dg-publish":true,"permalink":"/knowledge/devops/roadmap/01-Linux网络与脚本基础/","tags":["DevOps","Linux","网络","SSH","DNS","负载均衡","脚本","Python","Go","Groovy"],"dg-note-properties":{"date":"2026-07-19","tags":["DevOps","Linux","网络","SSH","DNS","负载均衡","脚本","Python","Go","Groovy"]}}
---


# 一、Linux、网络与脚本基础

> DevOps 的地基：Linux 系统管理、Shell 自动化、网络与安全基础，以及自动化脚本语言选型。

## 1.1 Linux 系统管理

### 文件系统与磁盘

Linux 里"一切皆文件"——普通文件、目录、设备、套接字，都通过统一的文件接口访问。日常运维第一件事，是搞清楚**磁盘空间去哪了**。

`df` 和 `du` 经常成对使用，但看的不是一回事：`df` 展示的是**文件系统层面**的块占用（整个分区用了多少），`du` 统计的是**目录里文件实际占用的空间**。两者对不上时，最常见的原因是——**有文件被删除了，但进程还开着它的句柄**。这类文件对 `du` 不可见（目录项没了），却实实在在占着块（inode 还挂着），`df` 会如实反映。定位方法是 `lsof +L1`（列出 link count 为 0 的打开文件），释放要么重启持有它的进程，要么 `: > /proc/<pid>/fd/<fd>` 把内容清空。

| 命令 | 作用 |
|:----|:------|
| `df -h` | 各文件系统的使用率 |
| `du -sh /var/log` | 指定目录总占用 |
| `ls -la` | 列出文件（含隐藏、权限） |
| `lsof -p PID` | 进程打开的文件 |
| `find / -name "*.log" -mtime -1` | 找最近一天修改的日志 |

### inode 与链接

理解 inode 是理解 Linux 文件系统的钥匙。**inode 存的是文件的"身份"**——权限、属主、大小、时间戳、指向数据块的指针，唯独**不存文件名**。文件名只是目录里一条"名字 → inode 号"的映射。

由此引出两种链接：**硬链接**（`ln src dst`）是给同一个 inode 挂第二个名字，两个名字地位平等，删掉一个另一个照常，且不能跨文件系统（inode 号只在单个文件系统内唯一）；**软链接**（`ln -s src dst`）则是单独的一个文件，内容就是"指向的路径"，所以能跨文件系统、能指向目录、也可能"悬空"（目标没了）。

```bash
ls -i file            # 查 inode 号
stat file             # 看元数据详情
find / -inum 12345    # 按 inode 反查文件
```

这也解释了一个经典现象：磁盘"满了"删文件却释放不出空间——因为 inode 还被进程引用，`du` 看不见，`df` 却减不掉。

### 权限管理

Linux 的权限模型围绕"三种身份 × 三种操作"展开：身份是**属主（owner）/ 属组（group）/ 其他（other）**，操作是**读（r=4）/ 写（w=2）/ 执行（x=1）**。用 `ls -l` 看到的 `-rwxr-xr--`，就是三组三位的组合。

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

### 进程与信号

进程是运行中的程序。`ps aux` 看快照，`top`/`htop` 看实时。**进程状态**是排障时的关键线索：`R` 运行中、`S` 可中断睡眠（在等事件，可被唤醒）、`D` 不可中断睡眠（通常在等磁盘/网络 IO，**连 kill -9 都杀不掉**）、`Z` 僵尸（已退出但父进程还没回收它的退出码）、`T` 已停止。

**信号**是进程间通信的一种。最常用的两个：`SIGTERM`（15）是"请你优雅退出"，进程能捕获它做清理（关连接、刷缓冲）；`SIGKILL`（9）是"立即处决"，内核直接干掉，进程无法捕获，因此**可能丢数据**。所以规范是：先 `kill`（默认发 15），等一会儿不行再 `kill -9`。

| 命令 | 作用 |
|:----|:------|
| `ps aux` | 所有进程快照 |
| `top` / `htop` | 实时资源监控 |
| `kill PID` | 发 SIGTERM（15，可捕获） |
| `kill -9 PID` | 发 SIGKILL（9，不可捕获） |
| `systemctl status/restart svc` | systemd 管理服务 |

### systemd 服务管理

systemd 是现代 Linux 的"1 号进程"，管着系统启动和服务生命周期。它用 **unit 文件**描述服务，并支持依赖关系（`After=`/`Requires=`），因此能按拓扑顺序拉起服务。日常最常踩的坑：**改完 unit 文件必须 `daemon-reload`**，否则 systemd 用的还是旧配置。

```bash
systemctl daemon-reload          # 改 unit 文件后必须执行
systemctl enable --now svc       # 开机自启并立即启动
systemctl list-dependencies svc  # 看依赖树
journalctl -u svc --since "10 min ago"  # 看服务日志
```

### 排查常用

```bash
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

### TCP/IP 分层

网络通信被抽象成四层，每层只操心自己的事（分层解耦）：

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

### DNS 解析

DNS 把好记的域名翻译成 IP。最需要搞清楚的是**解析链路**：先查本地 hosts，再问本地 DNS 缓存/递归服务器，递归服务器若没缓存，就从**根域名服务器 → 顶级域（.com）→ 权威域名服务器**逐级问下去，最后拿到结果并缓存。`TTL` 决定了这条记录能被缓存多久——**TTL 短**适合需要快速切换（如切流量、故障转移），**TTL 长**则减少查询压力。

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

### 负载均衡

负载均衡按**工作层次**分两类。**L4** 只看 IP 和端口做转发，不解析应用层内容，性能高、通用，典型是 Nginx Stream、云 SLB；**L7** 会解析 HTTP，能按域名、路径、Header 路由，灵活但开销大，典型是 Nginx、K8s Ingress。

```
用户 → DNS → L4 LB:443 → Nginx（TLS 终结）→ L7 路由 → 后端 Pod
```

实际架构里常见的分工是：L4 把流量引进来，L7 在里层做精细路由，TLS 往往在 L7 这层终结。

### TLS 握手

HTTPS 的秘密在于"**握手用非对称、传输用对称**"这套组合拳。原因很实际：非对称加密（RSA/ECDHE）安全但慢，对称加密（AES）快但双方得先有同一把密钥——于是用非对称的方式，安全地协商出一把对称的会话密钥，之后的数据都用这把会话密钥加密。

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

### 网络排障

排障讲究"从下往上、逐层确认"：先 `ping` 通不通（网络层），再 `telnet host 443` 看端口开没开（传输层），然后 `curl -v` 看 HTTP 层细节，必要时 `tcpdump` 抓包看真实报文。

```bash
ping -c 4 host            # 连通性（ICMP）
traceroute host           # 路由逐跳
curl -v https://host      # HTTP 链路诊断
ss -tlnp                  # 本机监听端口
ss -tnp state established # 已建立的连接
tcpdump -i eth0 port 80   # 抓包
```

### TCP 三次握手与四次挥手

```text
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

## 1.4 安全基础

### SSH 免密与别名

SSH 用**非对称密钥**做认证：你把公钥放到服务器的 `authorized_keys`，登录时用私钥证明身份。`ed25519` 比 RSA 更短更安全，是当前推荐。`~/.ssh/config` 里配别名能省去每次敲一长串参数。

```bash
ssh-keygen -t ed25519          # 生成密钥对
ssh-copy-id user@host          # 把公钥推到目标机

# ~/.ssh/config 别名
Host prod
    HostName 10.0.1.100
    User deploy
    IdentityFile ~/.ssh/prod_key
```

### 证书与防火墙

```bash
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

### 安全加固 Checklist

- [ ] 禁用 root SSH 登录，只用密钥认证
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

### Python

Python 是 DevOps 脚本的"万能胶"，生态最全。典型场景包括写构建/发布脚本、批量运维、调用各家 API（K8s / 云厂商 / GitLab）、处理日志和监控数据。调用 K8s API 时，官方 `kubernetes` 库是最直接的方式：

```python
from kubernetes import client, config

config.load_kube_config()                     # 读 ~/.kube/config
v1 = client.CoreV1Api()
pods = v1.list_namespaced_pod(namespace='production')
for pod in pods.items:
    print(f"{pod.metadata.name}: {pod.status.phase}")
```

常用库：`requests`（HTTP）、`click`/`typer`（CLI 框架）、`Jinja2`（模板渲染，生成配置文件）、`pandas`（数据分析）。

### Go

Go 是"云原生时代的母语"——K8s、Docker、etcd、Prometheus 全是 Go 写的。它的强项是**编译成单二进制、并发模型简洁、性能接近 C**。用 `client-go` 操作 K8s 是标准做法：

```go
import "k8s.io/client-go/kubernetes"

clientset, _ := kubernetes.NewForConfig(config)
pods, _ := clientset.CoreV1().Pods("default").List(ctx, metav1.ListOptions{})
for _, pod := range pods.Items {
    fmt.Println(pod.Name)
}
```

### Groovy

Groovy 在这里几乎是"为 Jenkins Pipeline 而生"的语言——它运行在 JVM 上，语法灵活，是写 Jenkinsfile 和 Jenkins Shared Library 的语言。

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
