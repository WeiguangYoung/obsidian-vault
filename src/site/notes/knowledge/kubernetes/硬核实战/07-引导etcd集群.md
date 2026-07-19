---
{"dg-publish":true,"permalink":"/knowledge/kubernetes/硬核实战/07-引导etcd集群/","tags":["Kubernetes","K8s","硬核实战"],"dg-note-properties":{"date":"2026-07-19","tags":["Kubernetes","K8s","硬核实战"]}}
---


# 引导 etcd 集群

Kubernetes 各组件为无状态设计，集群状态存储在 etcd 中。本实验将引导一个单节点 etcd 集群。

## 准备工作

将 etcd 二进制文件和 systemd unit 文件拷贝到 server 节点：

```bash
scp \
  downloads/controller/etcd \
  downloads/client/etcdctl \
  units/etcd.service \
  root@server:~/
```

登录 server 节点：

```bash
ssh root@server
```

## 引导 etcd 集群

### 安装 etcd 二进制文件

```bash
{
  mv etcd etcdctl /usr/local/bin/
}
```

### 配置 etcd 服务

```bash
{
  mkdir -p /etc/etcd /var/lib/etcd
  chmod 700 /var/lib/etcd
  cp ca.crt kube-api-server.key kube-api-server.crt \
    /etc/etcd/
}
```

每个 etcd 成员必须在集群中有唯一名称。此处使用主机名作为 etcd 名称。

部署 systemd unit 文件：

```bash
mv etcd.service /etc/systemd/system/
```

### 启动 etcd

```bash
{
  systemctl daemon-reload
  systemctl enable etcd
  systemctl start etcd
}
```

## 验证

列出集群成员：

```bash
etcdctl member list
```

```text
6702b0a34e2cfd39, started, controller, http://127.0.0.1:2380, http://127.0.0.1:2379, false
```

下一步：[引导 K8s 控制平面](08-bootstrapping-kubernetes-controllers.md)
