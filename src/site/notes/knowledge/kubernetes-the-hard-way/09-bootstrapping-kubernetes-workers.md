---
{"dg-publish":true,"permalink":"/knowledge/kubernetes-the-hard-way/09-bootstrapping-kubernetes-workers/","tags":["Kubernetes","K8s","硬核实战"],"dg-note-properties":{"date":"2026-07-19","tags":["Kubernetes","K8s","硬核实战"]}}
---


# 引导 Worker 节点

本实验引导两个 K8s Worker 节点，安装以下组件：runc、CNI 网络插件、containerd、kubelet、kube-proxy。

## 准备工作

以下命令在 `jumpbox` 上执行。

将配置文件和二进制文件分发到各 Worker：

```bash
for HOST in node-0 node-1; do
  SUBNET=$(grep ${HOST} machines.txt | cut -d " " -f 4)
  sed "s|SUBNET|$SUBNET|g" \
    configs/10-bridge.conf > 10-bridge.conf

  sed "s|SUBNET|$SUBNET|g" \
    configs/kubelet-config.yaml > kubelet-config.yaml

  scp 10-bridge.conf kubelet-config.yaml \
  root@${HOST}:~/
done
```

```bash
for HOST in node-0 node-1; do
  scp \
    downloads/worker/* \
    downloads/client/kubectl \
    configs/99-loopback.conf \
    configs/containerd-config.toml \
    configs/kube-proxy-config.yaml \
    units/containerd.service \
    units/kubelet.service \
    units/kube-proxy.service \
    root@${HOST}:~/
done
```

```bash
for HOST in node-0 node-1; do
  scp \
    downloads/cni-plugins/* \
    root@${HOST}:~/cni-plugins/
done
```

登录各 Worker 节点：

```bash
ssh root@node-0
```

## 部署 Worker 节点

安装系统依赖：

```bash
{
  apt-get update
  apt-get -y install socat conntrack ipset kmod
}
```

> `socat` 为 `kubectl port-forward` 提供支持。

禁用 Swap：

Kubernetes 对 swap 支持有限，因为启用了 swap 难以保证 Pod 内存使用。

```bash
swapon --show
```

如无输出，说明 swap 已禁用。如有输出，立即禁用：

```bash
swapoff -a
```

> 为确保重启后 swap 保持关闭，请查阅 Linux 发行版文档。

创建所需目录：

```bash
mkdir -p \
  /etc/cni/net.d \
  /opt/cni/bin \
  /var/lib/kubelet \
  /var/lib/kube-proxy \
  /var/lib/kubernetes \
  /var/run/kubernetes
```

安装 Worker 二进制文件：

```bash
{
  mv crictl kube-proxy kubelet runc \
    /usr/local/bin/
  mv containerd containerd-shim-runc-v2 containerd-stress /bin/
  mv cni-plugins/* /opt/cni/bin/
}
```

### 配置 CNI 网络

```bash
mv 10-bridge.conf 99-loopback.conf /etc/cni/net.d/
```

加载 `br-netfilter` 内核模块，确保 CNI bridge 的流量被 iptables 处理：

```bash
{
  modprobe br-netfilter
  echo "br-netfilter" >> /etc/modules-load.d/modules.conf
}
```

```bash
{
  echo "net.bridge.bridge-nf-call-iptables = 1" \
    >> /etc/sysctl.d/kubernetes.conf
  echo "net.bridge.bridge-nf-call-ip6tables = 1" \
    >> /etc/sysctl.d/kubernetes.conf
  sysctl -p /etc/sysctl.d/kubernetes.conf
}
```

### 配置 containerd

```bash
{
  mkdir -p /etc/containerd/
  mv containerd-config.toml /etc/containerd/config.toml
  mv containerd.service /etc/systemd/system/
}
```

### 配置 kubelet

```bash
{
  mv kubelet-config.yaml /var/lib/kubelet/
  mv kubelet.service /etc/systemd/system/
}
```

### 配置 kube-proxy

```bash
{
  mv kube-proxy-config.yaml /var/lib/kube-proxy/
  mv kube-proxy.service /etc/systemd/system/
}
```

### 启动 Worker 服务

```bash
{
  systemctl daemon-reload
  systemctl enable containerd kubelet kube-proxy
  systemctl start containerd kubelet kube-proxy
}
```

检查 kubelet 状态：

```bash
systemctl is-active kubelet
```

```text
active
```

> ⚠️ 务必在 `node-0` 和 `node-1` 上都完成以上步骤。

## 验证

从 `jumpbox` 查看注册的节点：

```bash
ssh root@server \
  "kubectl get nodes \
  --kubeconfig admin.kubeconfig"
```

```
NAME     STATUS   ROLES    AGE    VERSION
node-0   Ready    <none>   1m     v1.32.3
node-1   Ready    <none>   10s    v1.32.3
```

下一步：[配置 kubectl 远程访问](10-configuring-kubectl.md)
