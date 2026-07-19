---
{"dg-publish":true,"permalink":"/knowledge/devops-general/DevOps-01-Linux与网络基础/","tags":["DevOps","Linux","网络","SSH","DNS","负载均衡"],"dg-note-properties":{"date":"2026-07-19","tags":["DevOps","Linux","网络","SSH","DNS","负载均衡"]}}
---


# 一、Linux & 网络基础

> DevOps 的地基。容器、K8s、CI/CD 跑在上面，底层不牢则上层不稳。

## 1.1 Linux 系统管理

### 文件系统

| 命令 | 说明 |
|:----|:------|
| `ls -la` | 列出文件（含隐藏、权限） |
| `du -sh /var/log` | 目录占用 |
| `df -h` | 磁盘使用情况 |
| `lsof -p PID` | 进程打开的文件 |
| `find / -name "*.log" -mtime -1` | 找最近一天修改的日志 |

### 权限管理

```
-rwxr-xr--  1 user group  4096 Jul 19 10:00 app
 │││││││││
 │└─ rwx (owner) ─── rwx
 │   └─ r-x (group) ── r-x
 │       └─ r-- (other) ─ r--
```

| 命令 | 说明 |
|:----|:------|
| `chmod 755 file` | 修改权限 |
| `chown user:group file` | 修改归属 |
| `sudo -u appuser cmd` | 以指定用户执行 |
| `visudo` | 安全编辑 sudoers |

### 进程管理

| 命令 | 说明 |
|:----|:------|
| `ps aux` | 所有进程 |
| `top` / `htop` | 实时监控 |
| `kill -9 PID` | 强制终止 |
| `systemctl status/start/stop/restart` | systemd 管理服务 |

### 排查常用

```bash
# 找到占用 CPU 最高的进程
ps aux --sort=-%cpu | head -10

# 找到占用内存最高的进程
ps aux --sort=-%mem | head -10

# 查看端口占用
ss -tlnp | grep :8080

# 查看磁盘 IO
iostat -x 1
```

## 1.2 Shell 自动化

### 文本处理三剑客

```bash
# grep: 过滤
grep -r "ERROR" /var/log/   | 递归搜索
grep -v "DEBUG" app.log     | 反向过滤

# awk: 列处理
awk '{print $1, $NF}'       | 第1列和最后一列
awk '$3 > 80 {print $1}'    | 第3列大于80的行
df -h | awk '$5+0 > 80'     | 磁盘使用超80%

# sed: 替换
sed 's/old/new/g' file      | 全局替换
sed -i 's/debug//g' file    | 原地删除 debug
```

### 定时任务

```bash
# crontab 格式
# 分 时 日 月 周 命令
0 2 * * * /scripts/backup.sh     # 每天凌晨2点
*/5 * * * * /scripts/health.sh   # 每5分钟
```

## 1.3 网络基础

### TCP/IP 协议栈

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

### DNS

| 概念 | 说明 |
|:----|:------|
| A 记录 | 域名 → IPv4 |
| CNAME | 域名 → 另一个域名 |
| TTL | DNS 缓存时间，短 TTL 利于快速切换 |
| 解析链路 | 本地 hosts → DNS 缓存 → 递归 → 根 → 权威 |

```bash
# 排障命令
nslookup example.com
dig example.com +trace          # 跟踪解析全过程
cat /etc/resolv.conf             # DNS 配置
```

### 负载均衡

| 层级 | 说明 | 实现 |
|:----|:------|:------|
| **L4** | 基于 IP + 端口转发 | Nginx Stream / SLB / ELB NLB |
| **L7** | 基于 HTTP 头 / Path 路由 | Nginx / Ingress / ALB |

```
用户 → DNS → L4 LB:443 → Nginx (TLS 终结) → L7 路由 → 后端 Pod
```

### 常用网络排障

```bash
ping -c 4 host           # 连通性
traceroute host          # 路由跟踪
curl -v https://host     # HTTP 链路诊断
telnet host 443          # 端口是否开放
tcpdump -i eth0 port 80  # 抓包
```

## 1.4 安全基础

### SSH

```bash
# 密钥登录
ssh-keygen -t ed25519
ssh-copy-id user@host

# 别名管理
Host prod
    HostName 10.0.1.100
    User deploy
    IdentityFile ~/.ssh/prod_key
```

### TLS/SSL

```bash
# 检查证书有效期
openssl s_client -connect host:443 2>/dev/null | openssl x509 -noout -dates

# 自签证书（测试用）
openssl req -x509 -newkey rsa:4096 -keyout key.pem -out cert.pem -days 365 -nodes
```

### 防火墙

```bash
# iptables 基础
iptables -L -n              # 查看规则
iptables -A INPUT -p tcp --dport 443 -j ACCEPT

# firewalld
firewall-cmd --list-all
firewall-cmd --add-port=8080/tcp --permanent
```

### 安全加固 Checklist

- [ ] 禁用 root SSH 登录，只用密钥认证
- [ ] 防火墙只开放必要端口
- [ ] 定期 `yum/apt update` 安全补丁
- [ ] 审计日志启用（`auditd`）
- [ ] SELinux / AppArmor 开启 enforcing 模式
