---
{"dg-publish":true,"permalink":"/knowledge/devops/jenkins/02-Jenkins核心引擎/","tags":["Jenkins","CI/CD","DevOps","核心引擎","Controller","Agent"],"dg-note-properties":{"date":"2026-09-16","tags":["Jenkins","CI/CD","DevOps","核心引擎","Controller","Agent"]}}
---


# 🧩 Jenkins 核心引擎

> Jenkins 功能模块全景 · 第二章。Jenkins 的运行时骨架：一个控制器 + 若干执行节点。

---

## 一、Controller（控制器，原 Master）

Jenkins 的中枢，本身**不跑构建**（除非配置了内置节点），负责：

- 任务调度与队列管理
- 配置存储（`$JENKINS_HOME`）
- Web UI 与 REST API
- 权限与安全
- 插件生命周期
- 构建日志与结果收集

**关键点**：Controller 是**单点**。它挂了整个 Jenkins 停摆，所以生产环境要备份 `JENKINS_HOME`、必要时做冷备/主备。

## 二、Agent / Node（代理节点，原 Slave）

实际执行构建的 Worker。Controller 把任务分派给空闲的 Agent，Agent 跑完把结果和产物回传。

**接入方式**（决定 Agent 怎么连上 Controller）：

| 方式 | 说明 | 适用 |
|:-----|:-----|:-----|
| **SSH** | Controller 主动 SSH 到 Agent 启动进程 | Agent 有固定 IP、可被 Controller 访问 |
| **JNLP / WebSocket** | Agent 主动连 Controller（`-secret` 认证） | **内网 NAT 只有出方向**的场景（WebSocket 单端口复用最省事） |
| **K8s Pod** | Controller 通过 kubernetes-plugin 动态创建 Pod 当 Agent | 云原生、按需伸缩、用完即销 |
| **Docker Agent** | 以容器形式拉起 Agent | 环境隔离、可复现 |
| **Permanent Agent** | 长期在线的固定节点 | 需要特殊环境（Windows、硬件台架） |

**Agent 配置项**：远程工作目录、标签（label）、启动方式、并发执行数、环境变量、工具路径。

## 三、Executor（执行器 / 执行槽）

Agent 上的**并发槽位**。一个 Agent 可配多个 Executor，代表它能同时跑几个构建。

- Agent 的「总执行能力」= Executor 数量
- Controller 自身也默认有 2 个 Executor（**生产环境建议设为 0**，让 Controller 专心调度）
- 队列里等待的任务，本质是在等一个空闲 Executor

## 四、三者的关系

```
        ┌──────────────────┐
        │    Controller    │  调度 / 配置 / UI / API
        │  (master, 单点)   │
        └────────┬─────────┘
                 │ 分派任务
     ┌───────────┼───────────┐
     ▼           ▼           ▼
 ┌────────┐  ┌────────┐  ┌────────┐
 │ Agent1 │  │ Agent2 │  │ Agent3 │   执行构建
 │ [E][E] │  │ [E]    │  │ [E][E] │   E = Executor
 └────────┘  └────────┘  └────────┘
   SSH         JNLP/WS     K8s Pod
```

- **Controller**：调度者，1 个
- **Agent**：执行者，N 个（可跨机器、跨平台、跨网络）
- **Executor**：并发度，Agent 内部再分槽

## 五、串起来看

```
Job(任务)  →  Pipeline(流程)  →  Stage/Step(阶段步骤)  →  Agent(执行载体)  →  Executor(并发槽)
   声明        编排                具体干活                  谁来干               同时干几件
```

> 💡 **排障提示**：构建一直排队 → 看是不是 Agent 掉线或 Executor 被占满；构建报 "no space on node" → 看 Agent 工作区磁盘；动态 Agent 拉不起来 → 看 Cloud 配置与 Pod 模板。

---

> 🦐 虾管家 · 2026-09-16
