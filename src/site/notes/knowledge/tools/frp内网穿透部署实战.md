---
{"dg-publish":true,"permalink":"/knowledge/tools/frp内网穿透部署实战/","tags":["DevOps","frp","内网穿透","Docker","反向代理"],"dg-note-properties":{"date":"2026-09-18","updated":"2026-09-18","tags":["DevOps","frp","内网穿透","Docker","反向代理"]}}
---


# frp 内网穿透部署实战

> frp（Fast Reverse Proxy）把没有公网 IP 的内网机器通过「服务端 + 客户端」隧道暴露到公网。
> 本文记录两种部署方式：**二进制 + systemd（原生）** 与 **Docker Compose（容器）**，并附服务端与客户端配置。
> 版本对应：v0.52+ 使用 TOML 配置，之前使用 INI 配置。

## 一、架构与角色

```
内网机器（无公网 IP）                公网服务器（有公网 IP）
┌──────────────┐   隧道（frpc 主动拨出）  ┌──────────────┐
│   frpc       │ ───────────────────────▶ │   frps       │
│  客户端       │                          │  服务端       │
│  本地 22/80.. │ ◀─────────────────────── │  监听 7000    │
└──────────────┘     外部访问者连 frps 的   └──────────────┘
                     映射端口 → 转到内网
```

- **frps**：部署在有公网 IP 的服务器，监听 `bindPort`（默认 7000）等待客户端接入。
- **frpc**：部署在内网机器，**主动向外拨出**建立长连接，把本地端口映射到服务端的 `remotePort`。
- 关键点：内网机器**只出站**，无需开放入站端口 —— 这也是它能穿透家宽 NAT 的原因。

## 二、服务端部署（frps）

### 2.1 配置（TOML，v0.52+）

```toml
bindPort = 7000
vhostHTTPPort = 8080

# 隧道鉴权：客户端必须带相同 token 才能登录
auth.method = "token"
auth.token  = "<自定义长随机串>"

# Dashboard 管理面板
webServer.addr     = "0.0.0.0"
webServer.port     = 7500
webServer.user     = "admin"
webServer.password = "<面板密码>"

log.to    = "console"
log.level = "info"
```

### 2.2 Docker Compose 方式（推荐）

```yaml
services:
  frps:
    image: snowdreamtech/frps:latest
    network_mode: host          # 必须 host，否则映射端口不通
    volumes:
      - ./:/etc/frp             # 当前目录挂进去，容器读 /etc/frp/frps.toml
    restart: always
```

启动与验证：

```bash
mkdir -p /root/services/frp && cd /root/services/frp
# 写入上面的 frps.toml
docker compose -f docker-compose-frps.yaml up -d
docker logs --tail 20 frp-frps-1
ss -tulnp | grep -E '7000|7500|8080'
```

正常日志：

```
frps uses config file: /etc/frp/frps.toml
frps tcp listen on 0.0.0.0:7000
dashboard listen on 0.0.0.0:7500
frps started successfully
```

> **易错点**：镜像 tag 从 `0.51.3` 升到 `latest`（0.71.0）后，**配置格式从 INI 变为 TOML**。若仍挂载 `frps.ini`，容器日志会报 `open /etc/frp/frps.toml: no such file or directory` 并退出。升级时务必改写 `frps.toml`。

### 2.3 二进制 + systemd 方式

```ini
# /etc/systemd/system/frps.service
[Unit]
Description=frps server
After=network-online.target
Wants=network-online.target

[Service]
ExecStart=/usr/local/bin/frps -c /etc/frp/frps.toml
Restart=on-failure
RestartSec=10

[Install]
WantedBy=multi-user.target
```

```bash
systemctl daemon-reload && systemctl enable --now frps
systemctl status frps
```

### 2.4 安全组 / 防火墙

公网服务器需放行：

| 端口 | 协议 | 用途 |
|:----:|:----:|:-----|
| 7000 | TCP | frpc 接入（**必须**）|
| 7500 | TCP | Dashboard（可选，建议只放内网）|
| 映射端口 | TCP | 各隧道暴露的端口（如 5222/5223）|

> 云厂商安全组与主机防火墙（firewalld/ufw）需**分别**放行，只改一边会出现「本机 curl 通、外网连不上」。

## 三、客户端部署（frpc）

### 3.1 配置（TOML）

```toml
serverAddr = "<frps 公网 IP>"
serverPort = 7000

auth.method = "token"
auth.token  = "<与服务端完全一致的 token>"

log.to    = "console"
log.level = "info"

# 一条隧道 = 一个 [[proxies]] 块
[[proxies]]
name       = "ssh"
type       = "tcp"
localIP    = "127.0.0.1"
localPort  = 22          # 本机被映射的服务端口
remotePort = 5223        # 暴露到 frps 的端口
```

> **多隧道**：再加一个 `[[proxies]]` 块即可，例如把 k3s API（6443）、Web（30080）一起映射。

### 3.2 方式一：二进制 + 用户级 systemd（原生）

适合个人机器、不想装 Docker 的场景。用 **user systemd + linger**，即使用户不登录服务也常驻。

```
~/.config/frp/frpc.toml          # 配置
~/bin/frpc                        # 二进制
~/.config/systemd/user/frpc.service
```

```ini
[Unit]
Description=frp client
After=network-online.target
Wants=network-online.target

[Service]
ExecStart=/home/<user>/bin/frpc -c /home/<user>/.config/frp/frpc.toml
Restart=on-failure
RestartSec=10
WorkingDirectory=/home/<user>

[Install]
WantedBy=default.target
```

```bash
# 关键：开启 linger，否则用户会话结束服务被回收、也不能开机自启
loginctl enable-linger <user>

systemctl --user daemon-reload
systemctl --user enable --now frpc
systemctl --user status frpc        # 注意：是 --user，不用 sudo
```

> **坑**：`systemd --user` 与系统级 `systemd` 是两套；管理命令必须带 `--user`。`linger=yes` 是"无人登录也运行 + 开机自启"的命门。

### 3.3 方式二：Docker Compose（容器）

```yaml
services:
  frpc:
    image: snowdreamtech/frpc:latest
    container_name: frpc
    network_mode: host          # 必须 host：localIP=127.0.0.1 才指向宿主机
    restart: always
    environment:
      - TZ=Asia/Shanghai
    volumes:
      - ./frpc.toml:/etc/frp/frpc.toml:ro
```

```bash
docker compose -f docker-compose-frpc.yaml up -d
docker logs -f frpc
```

**离线部署**（目标机拉不到 docker.io 时）：

```bash
docker save snowdreamtech/frpc:latest -o frpc-alpine.tar
# 拷贝到目标机
docker load -i frpc-alpine.tar
```

> **切换注意**：若目标机已有原生 frpc，先停掉再起容器，否则两者抢同一端口/同一隧道：
> `systemctl --user disable --now frpc`
>
> `network_mode: host` 是必须的 —— frp 客户端要访问宿主机的 `127.0.0.1:22`，桥接下拿到的是容器自己的回环。

### 3.4 k3s / Kubernetes 方式（可选）

ConfigMap 存 `frpc.toml` + Deployment（`hostNetwork: true`）：

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: frpc-config
data:
  frpc.toml: |
    serverAddr = "<frps 公网 IP>"
    serverPort = 7000
    auth.method = "token"
    auth.token  = "<token>"
    [[proxies]]
    name       = "ssh"
    type       = "tcp"
    localIP    = "127.0.0.1"
    localPort  = 22
    remotePort = 5223
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frpc
spec:
  replicas: 1
  strategy:
    type: Recreate
  selector:
    matchLabels: { app: frpc }
  template:
    metadata:
      labels: { app: frpc }
    spec:
      hostNetwork: true
      dnsPolicy: ClusterFirstWithHostNet
      tolerations:
        - operator: Exists
      containers:
        - name: frpc
          image: snowdreamtech/frpc:latest
          args: ["-c", "/etc/frp/frpc.toml"]
          volumeMounts:
            - { name: config, mountPath: /etc/frp, readOnly: true }
      volumes:
        - name: config
          configMap: { name: frpc-config }
```

> **hostNetwork 的副作用**：`127.0.0.1` 指向**所在节点的宿主机**，所以这实际是「把节点本机端口映射出去」。若要映射集群内某个 Service，应去掉 `hostNetwork`，用 Service DNS 名作 `localIP`。

## 四、验证与排障

### 4.1 连通性验证

```bash
# 客户端侧
docker logs --tail 20 frpc
#   login to server success, get run id [...]     ← token 校验通过
#   proxy added: [ssh] / start proxy success       ← 隧道注册成功

# 外部侧（从任意有公网的机器）
ssh -p 5223 <user>@<frps 公网 IP>
```

### 4.2 常见错误

| 现象 | 原因 | 处理 |
|:-----|:-----|:-----|
| `open /etc/frp/frps.toml: no such file or directory` | 镜像已升级到 TOML 版，仍挂 INI | 改写 `frps.toml` |
| `token in login doesn't match token from configuration` | 客户端 token 与服务端不一致 | 两端改一致后重启 |
| `kex_exchange_identification: Connection closed` | 隧道/家宽瞬时抖动 | 直接重试，非配置问题 |
| 本机 curl 通、外网连不上 | 只改了主机防火墙，没放安全组（或反之）| 安全组 + 防火墙都放行 |
| 容器 `network_mode` 非 host 时连不上本机 22 | 桥接下 127.0.0.1 是容器自身 | 用 `network_mode: host` |
| `version` is obsolete 警告 | compose 文件顶层 `version:` 已废弃 | 删掉该行（不影响运行）|

### 4.3 配置格式速查

| frp 版本 | 配置文件 | 格式 |
|:--------:|:---------|:-----|
| ≥ 0.52.0 | `frps.toml` / `frpc.toml` | TOML |
| < 0.52.0 | `frps.ini` / `frpc.ini` | INI（`[common]` 段）|

## 五、选型建议

| 场景 | 推荐方式 |
|:-----|:---------|
| 个人机器、少依赖 | 二进制 + 用户级 systemd（`linger`）|
| 统一用容器管理 / 多实例 | Docker Compose（离线可 `docker save/load`）|
| 已在 K8s 环境、要跟着集群走 | Deployment + `hostNetwork` |
| 生产高可用 | 服务端多副本 + 前置 LB，客户端配多 `serverAddr`（frp 支持 failover）|

## 六、安全提醒

1. **务必设置 `auth.token`**，否则任何人扫到 7000 端口即可注册隧道，甚至做内网跳板。
2. Dashboard（7500）**不要暴露公网**，或至少改默认账密 + 加 IP 白名单。
3. 映射端口按需开放，避免把数据库、管理后台等敏感服务直接压到公网。
4. token、公网 IP、端口等**真实资产信息**不要写进公开文档。

---

## 📎 相关文档

- [[knowledge/devops/practices/04-k3s部署实战\|k3s 部署实战]] —— 内网节点组网（WireGuard）与集群网络
