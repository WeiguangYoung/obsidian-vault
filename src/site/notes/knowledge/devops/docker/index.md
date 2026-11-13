---
{"dg-publish":true,"permalink":"/knowledge/devops/docker/index/","title":"Docker 导航","tags":["导航","索引","Docker"],"dg-note-properties":{"title":"Docker 导航","tags":["导航","索引","Docker"]}}
---


# 🐳 Docker · 目录导航

> 从底层原理到环境搭建、GPU 支持、编排与 K8s 图形化管理

返回 [[首页\|🏠 首页]]

---

## 文档索引

| # | 文档 | 说明 |
|:--|:--|:--|
| 01 | [[knowledge/devops/docker/01-Docker底层原理入门\|01-Docker底层原理入门]] | Namespace / Cgroup / OverlayFS / 网络四大支柱 + 镜像与构建优化 |
| 02 | [[knowledge/devops/docker/02-环境安装与国内源配置\|02-环境安装与国内源配置]] | Ubuntu/CentOS 安装、国内镜像源、data-root 迁移、Portainer |
| 03 | [[knowledge/devops/docker/03-GPU直通与WSL环境\|03-GPU直通与WSL环境]] | nvidia-container-toolkit 配置 + WSL 2 安装与磁盘迁移 |
| 04 | [[knowledge/devops/docker/04-Compose实战与编排规范\|04-Compose实战与编排规范]] | 生产级 compose 模板、健康检查、网络隔离、多环境管理 |
| 05 | [[knowledge/devops/docker/05-Helm安装与Chart实践\|05-Helm安装与Chart实践]] | Helm 安装、values 优先级、Chart 编写与 CI 集成 |
| 06 | [[knowledge/devops/docker/06-Rancher部署与K8s运维\|06-Rancher部署与K8s运维]] | Rancher Server 部署、下游集群创建、常见 NotReady 排查 |

---

## 学习顺序建议

```
入门理解     01 底层原理
     ↓
环境准备     02 安装与源配置  →  03 GPU / WSL（按需）
     ↓
日常使用     04 Compose  →  05 Helm
     ↓
集群管理     06 Rancher
```

> 01 是理解为什么，02 是能跑起来，04/05 是真实项目里用得最多的两样。
