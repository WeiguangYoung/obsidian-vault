---
{"dg-publish":true,"permalink":"/knowledge/kubernetes-the-hard-way/10-configuring-kubectl/","tags":["Kubernetes","K8s","硬核实战"],"dg-note-properties":{"date":"2026-07-19","tags":["Kubernetes","K8s","硬核实战"]}}
---


# 配置 kubectl 远程访问

本实验基于 `admin` 用户凭据，为 `kubectl` 生成远程访问的 kubeconfig 文件。

> 以下命令在 `jumpbox` 上执行。

## Admin 配置文件

首先确认能访问 API Server（依赖之前的 `/etc/hosts` 解析）：

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

生成 admin 用户的 kubeconfig：

```bash
{
  kubectl config set-cluster kubernetes-the-hard-way \
    --certificate-authority=ca.crt \
    --embed-certs=true \
    --server=https://server.kubernetes.local:6443

  kubectl config set-credentials admin \
    --client-certificate=admin.crt \
    --client-key=admin.key

  kubectl config set-context kubernetes-the-hard-way \
    --cluster=kubernetes-the-hard-way \
    --user=admin

  kubectl config use-context kubernetes-the-hard-way
}
```

配置文件将写入默认路径 `~/.kube/config`，此后 `kubectl` 不再需要指定 `--kubeconfig`。

## 验证

查看远程集群版本：

```bash
kubectl version
```

```text
Client Version: v1.32.3
Kustomize Version: v5.5.0
Server Version: v1.32.3
```

列出集群节点：

```bash
kubectl get nodes
```

```
NAME     STATUS   ROLES    AGE    VERSION
node-0   Ready    <none>   10m   v1.32.3
node-1   Ready    <none>   10m   v1.32.3
```

下一步：[配置 Pod 网络路由](11-pod-network-routes.md)
