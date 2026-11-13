---
{"dg-publish":true,"permalink":"/knowledge/devops/docker/06-Rancher部署与K8s运维/","title":"Rancher 部署单节点 K8s 与运维","tags":["rancher","k8s","kubernetes","容器","运维","图形化"],"dg-note-properties":{"category":"Docker / 容器 / DevOps","title":"Rancher 部署单节点 K8s 与运维","date":"2026-11-10","tags":["rancher","k8s","kubernetes","容器","运维","图形化"]}}
---


# Rancher 部署单节点 K8s 与运维

> Rancher 的价值是**把 K8s 的日常运维图形化**——建集群、装监控、看日志、管权限都不用敲 kubectl。
> 一句话主线：**Rancher 本身只是个管理面（跑在 Docker 或已有 K8s 上），真正的工作集群是它下发 Agent 建出来的；搞不清这两层，后面所有问题都会搞混。**

---

## 一、先分清两件事

```
┌─────────────────────────────────────────────┐
│  Rancher Server（管理面）                     │
│  - 跑在 Docker / K8s 上                      │
│  - Web UI、认证、多集群统一管理               │
│  - 不跑你的业务                               │
└──────────────┬──────────────────────────────┘
               │ 下发 cluster-agent
               ▼
┌─────────────────────────────────────────────┐
│  下游集群（工作负载）                         │
│  - RKE2 / K3s / 自建 K8s                     │
│  - 跑你的业务 Pod                             │
└─────────────────────────────────────────────┘
```

**常见误解**：以为装了 Rancher 就有 K8s 了。**不对**——Rancher Server 只是个管理界面，K8s 要另外由它创建（或接入已有的）。

---

## 二、前置条件

Rancher 对宿主机要求不低（它是管理面，但也要跑不少组件）：

| 项 | 最低 | 建议 |
|:--|:--|:--|
| CPU | 2 核 | 4 核 |
| 内存 | 4GB | 8GB |
| 磁盘 | 20GB | 50GB+ |
| 系统 | 支持 Docker 的 Linux | Ubuntu 22.04 / RHEL 9 |

**本机（2 核 1.9GB 内存）跑不动 Rancher Server**——光启动就要 3GB 以上。

---

## 三、系统预检（踩坑重灾区）

### 3.1 关掉防火墙

Rancher / K8s 需要大量端口互通，防火墙最容易导致"装上去但节点 NotReady"。

```bash
# UFW
sudo apt install -y ufw
sudo ufw disable
sudo ufw status

# firewalld
sudo systemctl stop firewalld
sudo systemctl disable firewalld
```

### 3.2 放开 iptables

```bash
sudo iptables -P INPUT ACCEPT
sudo iptables -P FORWARD ACCEPT
sudo iptables -P OUTPUT ACCEPT
sudo iptables -F
```

> ⚠️ **这是最关键的一步**。`FORWARD` 链默认 DROP 时，Pod 之间的流量全部被挡，表现是「节点 Ready、Pod 起得来、但服务互相访问不了」。

### 3.3 关 swap

```bash
sudo swapoff -a
sudo sed -i '/swap/s/^/#/' /etc/fstab        # 永久关闭
```

### 3.4 内核模块与参数

```bash
sudo modprobe br_netfilter
sudo modprobe overlay

cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
br_netfilter
overlay
EOF

cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

sudo sysctl --system
```

| 参数 | 为什么必须 |
|:--|:--|
| `bridge-nf-call-iptables` | 让 iptables 能处理 bridge 流量，否则 Service 不通 |
| `ip_forward` | Pod 跨节点通信的基础 |

### 3.5 检查端口

```bash
# Rancher Server 需要
ss -lntp | grep -E "80|443"

# 下游集群（RKE2）需要
ss -lntp | grep -E "6443|9345|10250"
```

---

## 四、安装 Rancher Server

### 4.1 创建挂载目录

```bash
sudo mkdir -p /docker_volume/rancher_home/rancher
sudo mkdir -p /docker_volume/rancher_home/auditlog
```

### 4.2 启动容器

```bash
docker run -d \
  --restart=unless-stopped \
  --name rancher \
  -p 8080:80 \
  -p 8443:443 \
  -v /docker_volume/rancher_home/rancher:/var/lib/rancher \
  -v /docker_volume/rancher_home/auditlog:/var/log/auditlog \
  rancher/rancher:stable
```

> **老教程里的命令不完整的原因**：原始记录在 `-v` 参数后断了，实际必须给镜像名。

### 4.3 等初始化

```bash
# 看日志（首次启动会引导 etcd，5~10 分钟）
docker logs -f rancher

# 关键日志：出现这行表示就绪
# "Rancher startup complete"
```

### 4.4 获取初始密码

```bash
docker logs rancher 2>&1 | grep "Bootstrap Password:"
```

或（新版）

```bash
kubectl get secret --namespace cattle-system bootstrap-secret \
  -o go-template='{{.data.bootstrapPassword|base64decode}}{{"\n"}}'
```

### 4.5 首次登录

访问 `https://<IP>:8443` → 输入 bootstrap 密码 → **设置新密码** → 设置 Server URL。

> **Server URL 必须是外部可访问的地址**，下游集群的 Agent 要用它回连。写 `localhost` 会导致后面所有下游集群都连不上。

---

## 五、用 Rancher 创建下游集群

### 5.1 创建流程

```
Cluster Management → Create → Custom
  ↓
填 Cluster Name
  ↓
选 Kubernetes 版本（RKE2 / K3s）
  ↓
Next → 得到一条注册命令
  ↓
在目标节点上执行这条命令
```

### 5.2 注册命令长这样

```bash
curl --insecure -sfL https://<rancher-url>/v3/import/<token>_<hash>.yaml | kubectl apply -f -
```

**这条命令必须在目标节点上、以 root 执行**，且该节点已经装好 `kubectl`。

### 5.3 节点角色

| 角色 | 作用 | 单节点全选 |
|:--|:--|:--|
| **etcd** | 存集群状态（必须奇数个）| ✅ |
| **controlplane** | API Server / Scheduler / Controller | ✅ |
| **worker** | 跑业务 Pod | ✅ |

单节点测试：三个全勾。生产至少要 3 个 etcd。

### 5.4 等待 Ready

```
Cluster Management → 集群名 → 
  Nodes 页面看状态：Registering → Provisioning → Active
```

首次要 10~20 分钟（拉镜像慢的话更久）。

---

## 六、日常运维

### 6.1 通过 UI 能做什么

| 功能 | 路径 |
|:--|:--|
| 看/管 Pod、Deployment、Service | Cluster → Workload |
| 图形化 kubectl（Web Kubectl）| 右上角 → Kubectl Shell |
| 看日志 | Workload → 点进 Pod → Logs |
| 看监控 | Cluster → Monitoring（需先装 monitoring chart）|
| 装应用 | Apps → Repositories → Charts |
| 管权限 | Cluster → Members / Users & Authentication |
| 看事件 | Cluster → Events |

### 6.2 必备的几个 App

```
Apps → Charts 里搜：
├── Monitoring            （Prometheus + Grafana）
├── Logging               （Fluent Bit + Loki）
├── Cert-Manager          （自动证书）
└── Longhorn              （分布式存储，单节点也能跑）
```

### 6.3 Web Kubectl

右上角 `Kubectl Shell`，直接敲命令，省得配 kubeconfig。

**如果要在本地用 kubectl**：

```
Cluster → 右上角 Download KubeConfig
```

拿到文件放 `~/.kube/config`，注意里面是 Rancher 的代理地址，需要能访问 Rancher Server。

---

## 七、常见报错

| 报错 | 根因 | 解法 |
|:--|:--|:--|
| 节点一直 `Registering` | 节点连不上 Rancher Server | 检查 Server URL、网络、防火墙 |
| 节点 `NotReady` | CNI 没起来 / iptables 挡了 | 看 `kubectl describe node`，检查 FORWARD 链 |
| Pod 之间不通 | `FORWARD` 默认 DROP | `iptables -P FORWARD ACCEPT` |
| Service 访问不通 | 缺 `br_netfilter` | `modprobe br_netfilter` + sysctl |
| `x509: certificate signed by unknown authority` | 用了自签证书 | 加 `--insecure` 或用可信证书 |
| Rancher 容器起不来 | 内存不足 | 至少 4GB，建议 8GB |
| 下游集群 Agent 反复重连 | Server URL 写错 | 重新设置 Server URL（Settings 里可改）|
| UI 卡在 "Waiting for API" | 后端还在初始化 | 等 5~10 分钟，看 `docker logs rancher` |
| 节点加入后磁盘满 | Rancher 存了大量镜像/日志 | 清理或扩盘 |

### 排查顺序

```
节点 NotReady
  ↓
① kubectl describe node <名字>  → 看 Conditions 和 Events
  ↓
② kubectl get pods -A | grep -v Running  → 看谁没起来
  ↓
③ 最常见：calico/flannel Pod CrashLoop  → 看它的日志
  ↓
④ 九成是：iptables FORWARD 被 DROP / 内核模块没加载
```

---

## 八、Rancher vs 直接装 K8s vs K3s

| 方案 | 适用 | 优点 | 缺点 |
|:--|:--|:--|:--|
| **直接装 K8s（kubeadm）** | 学习原理 | 全流程可控，理解深 | 组件多，维护累 |
| **K3s** | 边缘 / 单机 / 小集群 | 单二进制，5 分钟装完，占用低 | 默认裁剪了部分组件 |
| **Rancher + RKE2/K3s** | 多集群管理 | 图形化、统一认证、一键装 App | 管理面本身要资源 |

**选型建议**：

```
就自己一台机器练手           → K3s（最省事）
公司有 3+ 个集群要统一管     → Rancher
要考 CKA / 学原理            → kubeadm
```

> **注意**：Rancher Server 自身要 4~8GB 内存，**在小机器上装 Rancher 反而不如直接装 K3s**。

---

## 九、卸载

```bash
# 1. 先在 UI 里删掉所有下游集群
# 2. 停并删 Rancher 容器
docker stop rancher && docker rm rancher

# 3. 清理数据（可选）
sudo rm -rf /docker_volume/rancher_home

# 4. 清 Docker 里 Rancher 相关的 volume
docker volume ls | grep rancher
```

**下游集群清理**（如果节点上还残留）：

```bash
# RKE2
sudo /usr/local/bin/rke2-uninstall.sh

# K3s
sudo /usr/local/bin/k3s-uninstall.sh
```

---

## 十、一句话总结

> **Rancher = K8s 的管理界面，不是 K8s 本身。** 装之前先把 `iptables FORWARD ACCEPT`、`br_netfilter`、关 swap 这几步做掉——90% 的"节点 NotReady"都是这三样没配。Server URL 一定要填外部可访问地址，否则下游集群永远连不上。
