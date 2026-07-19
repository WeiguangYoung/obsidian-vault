---
{"dg-publish":true,"permalink":"/knowledge/devops/GitHub优质DevOps开源项目/","tags":["DevOps","开源项目","学习资源","教程"],"dg-note-properties":{"date":"2026-07-19","tags":["DevOps","开源项目","学习资源","教程"]}}
---


# GitHub 优质 DevOps 开源项目

> 分 4 大类：路线教程、实操习题、完整 CI/CD 流水线、国产 DevOps 平台。

---

## 一、系统学习路线 / 教程仓库

### 1. 90DaysOfDevOps（最推荐新手）

> https://github.com/MichaelCade/90DaysOfDevOps

- 90 天分段学习计划：Linux → Docker → K8s → CI/CD → 监控 → 云平台
- 每天配套实操 demo、命令、配置文件
- GitHub Actions、Terraform、Prometheus、ArgoCD 实战案例
- 29.6k ⭐，3 年持续更新（2022/2023/2024）

### 2. developer-roadmap（DevOps 学习图谱）

> https://github.com/kamranahmedse/developer-roadmap

- 可视化 DevOps 技能树，标注每个阶段要学的工具和知识点
- 适合搭建完整学习框架，查漏补缺

### 3. kubernetes-the-hard-way（K8s 底层硬核实战）

> https://github.com/kelseyhightower/kubernetes-the-hard-way

- 不使用 kubeadm，纯手动一步步搭建 K8s 集群
- 吃透 K8s 底层原理，云原生 DevOps 必学

---

## 二、习题 / 面试实操仓库

### devops-exercises

> https://github.com/bregman-arie/devops-exercises
> 83k ⭐，行业公认 DevOps 题库

- 覆盖 Linux、Docker、Jenkins、K8s、Terraform、Ansible、Prometheus、云厂商全场景
- 理论问答 + 实操动手题，自带参考答案
- 自学自测、面试突击都能用

---

## 三、完整 CI/CD 流水线实战项目

### 1. Jenkins-Zero-To-Hero（Jenkins + GitOps）

> https://github.com/iamvickyav/Jenkins-Zero-To-Hero

```
代码提交 → Jenkins 构建测试 → Docker 打包镜像
    → Harbor 仓库 → ArgoCD GitOps 部署 K8s
```

提供全部 Jenkinsfile、K8s yaml、ArgoCD 配置，一键复现。

### 2. GitHub-Actions + ArgoCD GitOps（轻量化云原生）

> https://github.com/devopsboys/eks-gitops-demo

纯云原生无 Jenkins 方案：
- GitHub Actions 自动构建、镜像安全扫描、推送镜像
- ArgoCD 监听 Git 仓库自动发布到 K8s

### 3. Tekton Pipeline Demo（K8s 原生 CI）

谷歌开源云原生 CI 框架，提供完整 Tekton 任务和流水线模板，适合 K8s 集群内自建 CI。

---

## 四、国产开源 DevOps 平台

### 1. Spug — 轻量持续发布平台

> https://github.com/openspug/spug

- 主机管理、自动化发布、任务编排、监控告警
- 代码简洁易读，适合学习中小型企业发布系统设计

### 2. CoDo / OpenDevOps — 一站式多云运维平台

> https://github.com/opendevops/opendevops

- 完整 CMDB、CI/CD、权限审计、定时任务、监控大盘
- 大型企业 DevOps 架构参考

### 3. JumpServer — 堡垒机

> https://github.com/jumpserver/jumpserver

- 运维安全 4A 体系核心组件
- 权限管控、操作审计、Web 终端
- DevOps 安全必备

---

## 五、配套工具合集

### HariSekhon/DevOps-Bash-tools

> https://github.com/HariSekhon/DevOps-Bash-tools

上千条运维自动化 Shell 脚本，覆盖 K8s、AWS、Docker、数据库日常操作，直接复用提升效率。

---

## 快速选型建议

| 场景 | 推荐 |
|:----|:------|
| 零基础入门 | 90DaysOfDevOps |
| 刷题巩固 / 面试 | devops-exercises |
| 学习企业 CI/CD 流水线 | Jenkins-Zero-To-Hero |
| 吃透 K8s 底层 | kubernetes-the-hard-way |
| 自研 DevOps 平台 | Spug / CoDo |
