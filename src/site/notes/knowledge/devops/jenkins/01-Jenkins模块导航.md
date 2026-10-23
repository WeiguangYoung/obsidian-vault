---
{"dg-publish":true,"permalink":"/knowledge/devops/jenkins/01-Jenkins模块导航/","tags":["Jenkins","CI/CD","DevOps","导航","索引"],"dg-note-properties":{"date":"2026-09-16","tags":["Jenkins","CI/CD","DevOps","导航","索引"]}}
---


# 🧩 Jenkins 功能模块全景 · 导航

> Jenkins 本身具备的功能，按模块拆分为独立文档。本文是**总览 + 索引**。

---

## 一句话归纳

Jenkins = **调度器 + 分布式执行 + 插件生态**，核心抽象是四层：

```
Job(任务) → Pipeline(流程) → Stage/Step(阶段步骤) → Agent(执行载体)
```

配套 **Credentials 管密钥、Artifacts 管产物、Triggers 管触发、Plugins 管一切外延**。

## 模块索引

| # | 文档 | 内容 |
|:--|:-----|:-----|
| 02 | [[knowledge/devops/jenkins/02-Jenkins核心引擎\|02-Jenkins核心引擎]] | Controller / Agent / Executor |
| 03 | [[knowledge/devops/jenkins/03-Jenkins任务类型\|03-Jenkins任务类型]] | Freestyle / Pipeline / Multibranch / Organization / Folder / Matrix |
| 04 | [[knowledge/devops/jenkins/04-Jenkins-Pipeline能力\|04-Jenkins-Pipeline能力]] | Jenkinsfile / Stage-Step / Shared Library / 并行 / Agent 指令 / Blue Ocean |
| 05 | [[knowledge/devops/jenkins/05-Jenkins触发机制\|05-Jenkins触发机制]] | Webhook / 轮询 / 定时 / 上游 / 远程 / 参数化 |
| 06 | [[knowledge/devops/jenkins/06-Jenkins构建生态\|06-Jenkins构建生态]] | Credentials / Artifacts / Plugins / 工具自动安装 / Workspace |
| 07 | [[knowledge/devops/jenkins/07-Jenkins质量与反馈\|07-Jenkins质量与反馈]] | 测试报告 / 扫描门禁 / 构建状态 / 通知 / PR 回写 |
| 08 | [[knowledge/devops/jenkins/08-Jenkins进阶与运维\|08-Jenkins进阶与运维]] | JCasC / REST API / CLI / 安全域 / 备份迁移 / 高可用 / 可扩展性 |

## 学习路径建议

```
入门：02 核心引擎 → 03 任务类型 → 04 Pipeline 能力
进阶：04 Pipeline 能力 → 05 触发机制 → 06 构建生态
实战：07 质量与反馈 → 08 进阶与运维
```

## 全景脑图

```
Jenkins
├── 核心引擎        02  Controller / Agent / Executor
├── 任务类型        03  Freestyle / Pipeline / Multibranch / Org / Folder / Matrix
├── Pipeline 能力   04  Jenkinsfile / Stage / SharedLibrary / parallel / agent
├── 触发机制        05  Webhook / 轮询 / cron / 上游 / 远程 / 参数化
├── 构建生态        06  Credentials / Artifacts / Plugins / Tools / Workspace
├── 质量与反馈      07  Test Report / 扫描门禁 / 状态 / 通知 / PR 回写
└── 进阶与运维      08  JCasC / API / CLI / 安全 / 备份 / 高可用 / 扩展
```

---

> 🦐 虾管家 · 2026-09-16
