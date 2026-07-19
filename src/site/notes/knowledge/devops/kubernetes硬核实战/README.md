---
{"dg-publish":true,"permalink":"/knowledge/devops/kubernetes硬核实战/README/","tags":["Kubernetes","K8s","硬核实战"],"dg-note-properties":{"date":"2026-07-19","tags":["Kubernetes","K8s","硬核实战"]}}
---


# Kubernetes The Hard Way

> 来源：https://github.com/kelseyhightower/kubernetes-the-hard-way
> 不使用 kubeadm，纯手动搭建 K8s 集群，吃透底层原理

## 📖 教程目录

| 序号 | 章节 | 说明 |
|:---:|:-----|:------|
| 1 | [[knowledge/devops/kubernetes硬核实战/01-环境准备\|01-环境准备]] | 机器要求（Debian 12，4台） |
| 2 | [[knowledge/devops/kubernetes硬核实战/02-跳板机配置\|02-跳板机配置]] | 安装工具、下载二进制 |
| 3 | [[knowledge/devops/kubernetes硬核实战/03-计算资源准备\|03-计算资源准备]] | SSH、主机名、/etc/hosts |
| 4 | [[knowledge/devops/kubernetes硬核实战/04-CA与TLS证书\|04-CA与TLS证书]] | OpenSSL 搭建 PKI |
| 5 | [[knowledge/devops/kubernetes硬核实战/05-K8s认证配置\|05-K8s认证配置]] | 各组件 kubeconfig |
| 6 | [[knowledge/devops/kubernetes硬核实战/06-数据加密密钥\|06-数据加密密钥]] | 静态数据加密 |
| 7 | [[knowledge/devops/kubernetes硬核实战/07-引导etcd集群\|07-引导etcd集群]] | 单节点 etcd |
| 8 | [[knowledge/devops/kubernetes硬核实战/08-引导控制平面\|08-引导控制平面]] | API Server、Scheduler、Controller Manager |
| 9 | [[knowledge/devops/kubernetes硬核实战/09-引导Worker节点\|09-引导Worker节点]] | containerd、kubelet、kube-proxy、CNI |
| 10 | [[knowledge/devops/kubernetes硬核实战/10-配置kubectl\|10-配置kubectl]] | 远程集群访问 |
| 11 | [[knowledge/devops/kubernetes硬核实战/11-Pod网络路由\|11-Pod网络路由]] | 跨节点 Pod 通信 |
| 12 | [[knowledge/devops/kubernetes硬核实战/12-冒烟测试\|12-冒烟测试]] | 加密、部署、Service 验证 |
| 13 | [[knowledge/devops/kubernetes硬核实战/13-清理环境\|13-清理环境]] | 删除资源 |

## 📁 参考文件

- `configs/` — 各组件配置文件（CNI、containerd、kubelet、scheduler 等）
- `units/` — systemd 服务单元文件
