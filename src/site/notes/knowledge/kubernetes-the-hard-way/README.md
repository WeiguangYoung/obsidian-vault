---
{"dg-publish":true,"permalink":"/knowledge/kubernetes-the-hard-way/README/","tags":["Kubernetes","K8s","硬核实战"],"dg-note-properties":{"date":"2026-07-19","tags":["Kubernetes","K8s","硬核实战"]}}
---


# Kubernetes The Hard Way

> 来源：https://github.com/kelseyhightower/kubernetes-the-hard-way
> 不使用 kubeadm，纯手动搭建 K8s 集群，吃透底层原理

## 📖 教程目录

| 序号 | 章节 | 说明 |
|:---:|:-----|:------|
| 1 | [[knowledge/kubernetes-the-hard-way/01-prerequisites\|环境准备]] | 机器要求（Debian 12，4台） |
| 2 | [[knowledge/kubernetes-the-hard-way/02-jumpbox\|配置跳板机]] | 安装工具、下载二进制 |
| 3 | [[knowledge/kubernetes-the-hard-way/03-compute-resources\|准备计算资源]] | SSH、主机名、/etc/hosts |
| 4 | [[knowledge/kubernetes-the-hard-way/04-certificate-authority\|CA 与 TLS 证书]] | OpenSSL 搭建 PKI |
| 5 | [[knowledge/kubernetes-the-hard-way/05-kubernetes-configuration-files\|K8s 认证配置]] | 各组件 kubeconfig |
| 6 | [[knowledge/kubernetes-the-hard-way/06-data-encryption-keys\|数据加密密钥]] | 静态数据加密 |
| 7 | [[knowledge/kubernetes-the-hard-way/07-bootstrapping-etcd\|引导 etcd 集群]] | 单节点 etcd |
| 8 | [[knowledge/kubernetes-the-hard-way/08-bootstrapping-kubernetes-controllers\|引导控制平面]] | API Server、Scheduler、Controller Manager |
| 9 | [[knowledge/kubernetes-the-hard-way/09-bootstrapping-kubernetes-workers\|引导 Worker 节点]] | containerd、kubelet、kube-proxy、CNI |
| 10 | [[knowledge/kubernetes-the-hard-way/10-configuring-kubectl\|配置 kubectl]] | 远程集群访问 |
| 11 | [[knowledge/kubernetes-the-hard-way/11-pod-network-routes\|Pod 网络路由]] | 跨节点 Pod 通信 |
| 12 | [[knowledge/kubernetes-the-hard-way/12-smoke-test\|冒烟测试]] | 加密、部署、Service 验证 |
| 13 | [[knowledge/kubernetes-the-hard-way/13-cleanup\|清理环境]] | 删除资源 |

## 📁 参考文件

- `configs/` — 各组件配置文件（CNI、containerd、kubelet、scheduler 等）
- `units/` — systemd 服务单元文件
