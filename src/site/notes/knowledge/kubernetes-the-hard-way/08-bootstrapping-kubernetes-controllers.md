---
{"dg-publish":true,"permalink":"/knowledge/kubernetes-the-hard-way/08-bootstrapping-kubernetes-controllers/","tags":["Kubernetes","K8s","硬核实战"],"dg-note-properties":{"date":"2026-07-19","tags":["Kubernetes","K8s","硬核实战"]}}
---


# 引导 K8s 控制平面

本实验引导 Kubernetes 控制平面，在 `server` 节点上安装以下组件：API Server、Scheduler、Controller Manager。

## 准备工作

在 `jumpbox` 上将二进制文件和 systemd unit 文件拷贝到 server：

```bash
scp \
  downloads/controller/kube-apiserver \
  downloads/controller/kube-controller-manager \
  downloads/controller/kube-scheduler \
  downloads/client/kubectl \
  units/kube-apiserver.service \
  units/kube-controller-manager.service \
  units/kube-scheduler.service \
  configs/kube-scheduler.yaml \
  configs/kube-apiserver-to-kubelet.yaml \
  root@server:~/
```

登录 server：

```bash
ssh root@server
```

## 部署控制平面

创建 K8s 配置目录：

```bash
mkdir -p /etc/kubernetes/config
```

### 安装控制面二进制文件

```bash
{
  mv kube-apiserver \
    kube-controller-manager \
    kube-scheduler kubectl \
    /usr/local/bin/
}
```

### 配置 API Server

```bash
{
  mkdir -p /var/lib/kubernetes/

  mv ca.crt ca.key \
    kube-api-server.key kube-api-server.crt \
    service-accounts.key service-accounts.crt \
    encryption-config.yaml \
    /var/lib/kubernetes/
}
```

部署 API Server 的 systemd unit：

```bash
mv kube-apiserver.service \
  /etc/systemd/system/kube-apiserver.service
```

### 配置 Controller Manager

```bash
mv kube-controller-manager.kubeconfig /var/lib/kubernetes/
```

部署 systemd unit：

```bash
mv kube-controller-manager.service /etc/systemd/system/
```

### 配置 Scheduler

```bash
mv kube-scheduler.kubeconfig /var/lib/kubernetes/
mv kube-scheduler.yaml /etc/kubernetes/config/
```

部署 systemd unit：

```bash
mv kube-scheduler.service /etc/systemd/system/
```

### 启动控制面服务

```bash
{
  systemctl daemon-reload

  systemctl enable kube-apiserver \
    kube-controller-manager kube-scheduler

  systemctl start kube-apiserver \
    kube-controller-manager kube-scheduler
}
```

> API Server 完全初始化约需 10 秒。

检查各组件状态：

```bash
systemctl is-active kube-apiserver
systemctl status kube-apiserver
```

查看日志：

```bash
journalctl -u kube-apiserver
```

### 验证

```bash
kubectl cluster-info \
  --kubeconfig admin.kubeconfig
```

```text
Kubernetes control plane is running at https://127.0.0.1:6443
```

## Kubelet 鉴权 RBAC

配置 RBAC 权限，使 API Server 可以访问各 Worker 节点上的 Kubelet API（用于获取指标、日志和执行命令）。

> 本教程将 Kubelet 的 `--authorization-mode` 设为 `Webhook`，通过 SubjectAccessReview API 进行鉴权。

以下命令在 server 上执行：

```bash
kubectl apply -f kube-apiserver-to-kubelet.yaml \
  --kubeconfig admin.kubeconfig
```

### 整体验证

从 `jumpbox` 访问 API Server：

```bash
curl --cacert ca.crt \
  https://server.kubernetes.local:6443/version
```

```text
{
  "major": "1",
  "minor": "32",
  "gitVersion": "v1.32.3",
  ...
  "platform": "linux/arm64"
}
```

下一步：[引导 Worker 节点](09-bootstrapping-kubernetes-workers.md)
