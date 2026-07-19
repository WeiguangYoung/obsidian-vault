---
{"dg-publish":true,"permalink":"/knowledge/devops/kubernetes硬核实战/11-Pod网络路由/","tags":["Kubernetes","K8s","硬核实战"],"dg-note-properties":{"date":"2026-07-19","tags":["Kubernetes","K8s","硬核实战"]}}
---


# 配置 Pod 网络路由

Pod 从所在节点的 Pod CIDR 范围获得 IP 地址。目前由于缺少路由，不同节点上的 Pod 之间无法通信。

本实验为每个 Worker 节点创建路由，将节点的 Pod CIDR 范围映射到节点的内部 IP。

> 实现 K8s 网络模型[还有其他方式](https://kubernetes.io/docs/concepts/cluster-administration/networking/#how-to-achieve-this)。

## 配置路由表

提取各节点的 IP 和 Pod CIDR：

```bash
{
  SERVER_IP=$(grep server machines.txt | cut -d " " -f 1)
  NODE_0_IP=$(grep node-0 machines.txt | cut -d " " -f 1)
  NODE_0_SUBNET=$(grep node-0 machines.txt | cut -d " " -f 4)
  NODE_1_IP=$(grep node-1 machines.txt | cut -d " " -f 1)
  NODE_1_SUBNET=$(grep node-1 machines.txt | cut -d " " -f 4)
}
```

在各节点上添加路由：

```bash
ssh root@server <<EOF
  ip route add ${NODE_0_SUBNET} via ${NODE_0_IP}
  ip route add ${NODE_1_SUBNET} via ${NODE_1_IP}
EOF
```

```bash
ssh root@node-0 <<EOF
  ip route add ${NODE_1_SUBNET} via ${NODE_1_IP}
EOF
```

```bash
ssh root@node-1 <<EOF
  ip route add ${NODE_0_SUBNET} via ${NODE_0_IP}
EOF
```

## 验证

查看各节点的路由表：

```bash
ssh root@server ip route
```

```text
default via XXX.XXX.XXX.XXX dev ens160 
10.200.0.0/24 via XXX.XXX.XXX.XXX dev ens160 
10.200.1.0/24 via XXX.XXX.XXX.XXX dev ens160 
...
```

```bash
ssh root@node-0 ip route
```

```text
default via XXX.XXX.XXX.XXX dev ens160 
10.200.1.0/24 via XXX.XXX.XXX.XXX dev ens160 
...
```

```bash
ssh root@node-1 ip route
```

```text
default via XXX.XXX.XXX.XXX dev ens160 
10.200.0.0/24 via XXX.XXX.XXX.XXX dev ens160 
...
```

下一步：[冒烟测试](12-smoke-test.md)
