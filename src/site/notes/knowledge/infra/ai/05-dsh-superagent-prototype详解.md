---
{"dg-publish":true,"permalink":"/knowledge/infra/ai/05-dsh-superagent-prototype详解/","title":"dsh-superagent-prototype 详解","tags":["DeepSeek","Agent","dsh-superagent","研究原型","GitHub自动化","PR机器人","DSH"],"dg-note-properties":{"category":"AI / Agent","title":"dsh-superagent-prototype 详解","date":"2026-11-09","tags":["DeepSeek","Agent","dsh-superagent","研究原型","GitHub自动化","PR机器人","DSH"]}}
---


# dsh-superagent-prototype 详解

> 官方描述：**"Towards a More Autonomous Agent System"**
> 一句话定位：**一个只服务单个 GitHub 仓库的「主动型编码 Agent」，以 DeepSeek Harness 插件包的形式发布。**
>
> ⚠️ 官方 README 首句即声明：*"Research prototype for reference only, **not an official DeepSeek product**."* —— 研究参考原型，非官方产品，非生产可用。
>
> 📌 **别和 DeepSeek Harness（`deepseek-harness`）搞混**：那是通用 agent 底座产品（npx 即跑），本项目是建在它上面的**一个具体应用原型**。

---

## 一、基本档案

| 项 | 内容 |
|:--|:--|
| 仓库 | `github.com/deepseek-ai/dsh-superagent-prototype` |
| 全名 | `dsh-superagent`（README 标题） |
| 描述 | Towards a More Autonomous Agent System |
| 语言 | **TypeScript** |
| 许可 | MIT |
| Stars | 约 **21,500** |
| 创建 | 2026-10-31（研究原型，发布即开源） |
| 最后推送 | 2026-11-03 |
| 默认分支 | `main` |
| 形态 | **DeepSeek Harness 的插件 bundle**（不是一个独立可执行程序） |
| 场景 | **单个 GitHub 仓库** + 可选飞书（Feishu/Lark） |
| 官网 | 无（文档即 `docs/` 目录） |

> 有意思的背景（官方自述）：*"This code grew out of our team's day-to-day development work, with much of the engineering done by coding agents."* —— **这个项目本身大量代码是 coding agent 写的**。

---

## 二、它到底做什么

### 2.1 一段话说明

它**监听 GitHub（可选加飞书）**。当**被允许的人**向 Agent 提需求时，它会：

1. 开一个 **work item（Ticket）**
2. 在**自己的 agent session 和自己的目录**里干活
3. 回到**原来的对话**里答复 —— 比如提一个 Pull Request

再配一个 **Web 面板**，只显示「需要人处理的事」。

### 2.2 与传统 coding agent 的区别

| | 传统 coding agent | dsh-superagent |
|:--|:--|:--|
| 触发方式 | 你在终端/IDE 里敲 | **GitHub 评论 / @提及 / 飞书消息** |
| 运行位置 | 你的终端 | **常驻服务 + 独立 profile** |
| 产出形式 | 打印在屏幕 | **PR / 评论 / 消息回原对话** |
| 隔离 | 单一 session | **每个 Ticket 独立 session + 独立目录** |
| 谁去看结果 | 你 | **团队看 GitHub / 面板** |

### 2.3 核心信条（000-philosophy.md）

> **Take as little human attention as possible, and give as much value as possible.**
> 尽可能少占用人的注意力，尽可能多交付价值。

这条原则贯穿全项目，具体落地为：

| 对 Agent 的要求 | 说明 |
|:--|:--|
| **只在被请求或规则允许时行动** | 未授权的主动工作只在「记录模式」下运行 |
| **只在需要人行动或能用上结果时才说话** | 确认一次 → 干活时静默 → 出结果/需决策/失败时才回复 |
| **一次只问一个具体问题** | 结构化提问，避免淹没在长报告里 |
| **按影响停下，而不是按工作量停下** | 局部改动做到可评审为止；需要设计决策就停下来问 |
| **如实汇报** | 草稿标未完成，失败就说失败，不绕过限制 |

**设计层面**也体现这条原则：概念要少、设置要少、面板只显示需要人的事、文档按序阅读。

---

## 三、安装方式（官方"让 agent 带你装"）

### 3.1 官方推荐：让 coding agent 读 SETUP.md

```bash
git clone https://github.com/deepseek-ai/dsh-superagent-prototype
cd dsh-superagent-prototype
cat SETUP.md
```

然后对你的 coding agent 说：

> **Read SETUP.md in this repository and set up dsh-superagent with me.**

SETUP.md 的设计非常克制（本身就在践行 philosophy）：

- **一次只问一个问题**，等回答再问下一个
- 每个问题给一个「一个词就能接受」的推荐答案
- 每封信 < 300 词
- **收集完所有答案才一次性给出计划并执行**
- **绝不在聊天里索要密钥** —— 需要人亲手做的步骤，给出一条可复制到终端执行的命令

### 3.2 前置依赖

| 组件 | 版本要求 |
|:--|:--|
| Node.js | 22.19+ 或 24+ |
| pnpm | 任意近期版本 |
| git | 2.32+ |
| Python | 3.8+ |
| tmux | 必需 |
| GitHub CLI（`gh`） | **2.48+** |
| **DeepSeek Harness** | 版本需在 `package.json` 的 `engines.dsh` 范围内 |
| 模型 provider | 必须先在 DSH 里配好一个 |
| 飞书（可选） | `lark-cli`（tested 1.0.90）+ 一个带 bot 的应用 |

### 3.3 SETUP 会问的 9 个问题

**来源（Triggers 从哪来）**
1. 只接 GitHub，还是也接飞书？〔默认：只 GitHub〕
2. 哪个仓库（`owner/name`）？
3. Agent 用哪个 GitHub 账号？〔**必须是专用账号，绝不能用你自己的**——它写的一切都署这个名〕
4. （若接飞书）哪几个群？

**策略（谁可以提需求）**
5. GitHub 上哪些 login 可以提？〔默认：只有你自己〕
6. （若接飞书）谁能提？〔默认：第一个群的所有人〕

**执行（Agent 拿 Ticket 干什么）**
7. 从哪个本地 clone 检出？〔默认：`~/src/<name>` 新建一个〕
8. 提交作者署名谁？〔默认：Agent 账号〕
9. 要不要开启 PR 自动合并？〔**默认不要**——先让人来合〕

### 3.4 SETUP 不问、直接用的默认值

| 项 | 默认值 |
|:--|:--|
| Profile 名 | `superagent`（DSH Web profile 的副本，独立于你日常用的 profile） |
| 端口 | **3082** |
| 状态目录 | `<home>/.dsh/state/superagent`（绝对路径） |
| 自愈检查 | 每小时从你自己的 profile 检查一次，不在跑就修复 |
| 分支前缀 | `agent/` |
| **Accept 模式** | **`observe`（只记录主动工作，不真的做）** |

> `observe` 模式很关键：**先让它"空转"一段时间**，你看它本来会做什么，觉得对了再放宽。

---

## 四、使用形式

### 4.1 日常使用流程

```
① 团队成员在 GitHub 上 @Agent 或评论提需求
        ↓
② Source 收到 Trigger → Eval 判断「要不要接这个活」
        ↓
③ 接受 → 创建 Ticket（带 Mandate 授权范围）
        ↓
④ Assignment 分给 Executor（独立 session，名字 worker-<ticket id>）
        ↓
⑤ Executor 在自己的目录里干活，提交分支、开 PR
        ↓
⑥ 结果通过 Channel 回到原对话
        ↓
⑦ Web 面板显示「需要人处理的事」
```

### 4.2 人的三种介入点

| 场景 | 面板标签 |
|:--|:--|
| Agent 有问题要问 | **Awaiting answer** |
| 产出待评审 | **Awaiting review** |
| 卡住 / 失败 | **Blocked · Failed** |
| 输入投递失败 | **Input undelivered** |

其余状态（Queued / Working / Waiting for a Trigger）**不需要人**，健康状态在面板上不占空间。

### 4.3 停止与恢复

- 人发 **stop** → 只有被 stop 的 Ticket 停止
- 人发新消息 → **blocked / failed** 的 Ticket 自动回到 Executor 继续
- Ticket 结束后人再在 Thread 里说话 → 重新变 open（除非被 stop）

---

## 五、架构：领域模型驱动

### 5.1 工作流（官方 mermaid 原图重绘）

```
Source ──Trigger──► Thread ──batch──► Eval ──accept(带 Mandate)──► Ticket
  │                    │                                              │
  │                    └──Subscription fan-out──────────────────────►│
  │                                                                   │
  └◄──────────────────────── Channel ────────────────────────────────┘
                                                                      │
                                                          Assignment  │
                                                                      ▼
                                                                  Executor
```

### 5.2 六个核心概念

| 概念 | 定义 | 关键规则 |
|:--|:--|:--|
| **Source** | 事件流。可复用实现 + 实例配置 | 持有读位点和自己的 bot 身份；**重启不重放事件** |
| **Trigger** | 一个不可变事件 | 属于一个或多个 Thread；**同一 Trigger 接两次只产出一个 Ticket**（幂等） |
| **Thread** | 按时间排的 Trigger 序列 | **同一时刻只能跑一个 Eval**；可有多个 Ticket |
| **Policy** | 带版本的评估规则 | 规则变了，**正在跑的 Eval 必须作废** |
| **Eval** | 对一批 Trigger 的判断（gate + assessment 两层） | **无状态**；输出 Decision |
| **Mandate** | accept 授予的权限范围 | **不可变**；扩大范围 = 新 Ticket |
| **Ticket** | 授权工作单元 | **只有 Eval 能创建**；有自己的状态机，不跟随 Executor |

### 5.3 两层授权（002-authorization.md）

```
人提及 Agent（mention / direct）
        ↓
   [Gate] 检查发件人是否在白名单 + 是否显式请求
        ↓
   [Eval] 语义判断要不要接
        ↓
   [Mandate] 授予具体范围
```

关键规则：
- **消息、评论、被引用的文档都只是"输入"**，不能改变权限
- **Bot 只能产出"推断候选"，永远不能算显式请求**
- **订阅投递（Subscription）绕过 gate，但不授予新权限**
- Executor 拥有某条消息时，主 Session **只能 stop，不能 accept/attach/ignore/requeue**

### 5.4 执行的五个阶段（003-execution.md）

| Stage | 条件 | 面板标签 |
|:--|:--|:--|
| person / delivery | 输入投递失败，2s 后重试，指数退避到 1 分钟 | Input undelivered |
| person / answer | 有待回答的问题 | Awaiting answer |
| person / review | 最新 Report 是 `ready` | Awaiting review |
| person / blocked, failed | Report 是 `blocked`/`failed` 且此后无人发消息 | Blocked · Failed |
| working / queued | 无 Assignment 或待处理 | Queued |
| idle | Report 是 `waiting`，或 Assignment 结束 | Waiting for a Trigger |
| working | 其余 | Working |

**从上往下匹配，第一条命中即生效。**

### 5.5 Executor 细节

- Session 命名：`worker-<ticket id>`
- 目录：`<stateDir>/work/<ticket short name>`
- **以 operator 的 OS 用户身份执行命令，无需审批**
- 预设 `dsh-superagent-worker`：前台 shell 命令 5 秒后转后台，这样 worker 能在下一步收到新消息
- 主 Session 用 `dsh-superagent-router`：同样的 5 秒限制，工具集更小
- **后台任务上限 30 分钟**：到点 shell job 被杀，Ticket 报 waiting（subagent 允许继续）

---

## 六、技术架构（007-architecture.md）

### 6.1 组件图

```
Source plugins ─┐
Session tools  ─┼──► Engine ──► Store (Repositories)
Management HTTP┘      │ ├──► Policy
                      │ ├──► SourceRegistry
                      │ ├──► MasterAssessor ──decide──► Engine
                      │ ├──► DshExecutor
                      │ └──► Outbox
Settings ──policyChanged──► Engine
Effects ◄── observer/executor/sessions/outbox
```

### 6.2 四个关键模块

| 模块 | 职责 |
|:--|:--|
| **Runtime**（`src/app/runtime.ts`） | 组合根，不含领域规则；锁状态目录、开 Store、构建组件；启动注册管理路由、跑 Setup、装 hooks、恢复中断的 Assignment |
| **Passes**（`src/app/passes.ts`） | **每 2 秒跑一次 pass**：先到期 effects，再 `Engine.evaluateDue` |
| **Outbox**（`src/app/outbox.ts`） | 唯一写回执的地方；`send` 在一个事务里存回执 + 排队回复 effect |
| **Notices**（`src/app/notices.ts`） | 领域事件/worker 活动 → 主 Session 消息；`reportRelay` 立即转发问题，普通进度留在 Ticket |

### 6.3 与 DeepSeek Harness 的关系

**项目本身就是一个 DSH profile**（名字 `superagent`）：

```
$DSH_HOME/profiles/superagent/
├── package.json          # 含 dsh-superagent 依赖
└── cordis.patch.yml      # 配置补丁
```

- 从 DSH 的 **Web profile 模板**创建
- 监听 `127.0.0.1`，面板在 **3082** 端口
- **必须和你日常使用的 profile 分开**：插件会改进程级 `PATH` 和 git 设置，且需要独立重启升级

---

## 七、安全与隔离（重要）

### 7.1 官方明确的风险

| 项 | 说明 |
|:--|:--|
| 预设权限 | `dsh-superagent-worker` 和 `dsh-superagent-router` 都是 **`danger-full-access`** |
| 身份 | 以 **operator 的 OS 用户**运行 |
| 能力 | 可以按路径执行程序、读 operator 的 home 目录 |
| 隔离假象 | **账号 wrapper 只换凭据，不隔离进程** |

### 7.2 官方建议的隔离方式

> **用一个专用 OS 账号，里面只放 Agent 的凭据。**

其他要点：
- 不要把 `dsh-superagent-worker` / `router` 预设用于个人 Session
- 不要禁用它们的配置行，否则无法创建/恢复 Session

### 7.3 操作者责任（000-philosophy.md 结尾）

| 原则 | 含义 |
|:--|:--|
| **从小范围开始** | 短白名单 + 主动工作先只记录 |
| **为少噪音调优** | Agent 吵的时候，把 prompt 改短改严，或暂停它 —— **不要加更多消息去解释噪音** |
| **不要用来刷量** | 批量 PR、广播消息、无意义忙碌对团队是净损失 |
| **对它的发言负责** | 你账号下发的东西要读，错了改 prompt |

> 目标：**在对的时间给对的帮助，其他时间什么都不给。**

---

## 八、文档阅读顺序（官方指定）

`docs/` 下的编号文档**必须按序读**，每篇只依赖前面已引入的概念：

| # | 文档 | 内容 |
|:--|:--|:--|
| 000 | Philosophy | 设计哲学与操作者责任 |
| 001 | Domain model | Source/Trigger/Thread/Eval/Mandate/Ticket 等概念 |
| 002 | Authorization | 两层授权、订阅、handback 规则 |
| 003 | Execution | Ticket 阶段、Executor、后台任务 |
| 004 | Prompts | 提示词设计 |
| 005 | Panel | Web 面板 |
| 006 | Deployment | 安装、配置、升级、排障 |
| 007 | Architecture | 实现映射（类、接口、文件） |
| 008 | Cookbook | 扩展指南（如添加新 Source） |

**开发**：`pnpm check`（typecheck + tests + build）

---

## 九、和 DeepSeek Harness 的区别（再强调一次）

| 维度 | DeepSeek Harness | dsh-superagent-prototype |
|:--|:--|:--|
| 是什么 | 通用 agent **底座产品** | 建在 DSH 上的**一个应用原型** |
| 依赖关系 | 无 | **依赖 DSH**（是它的插件 bundle） |
| 安装 | `npx @deepseek-ai/dsh web` | clone + 按 SETUP.md 配 |
| 配置复杂度 | 低（一条命令） | 高（9 个问题 + operator 手工步骤） |
| 场景 | 通用 | 单个 GitHub 仓库 |
| 目标用户 | 想跑 agent 的人 | 想研究「Agent 怎么替团队值班」的人 |
| Stars | ~277k | ~21.5k |
| 成熟度 | developer preview | research prototype |

> **类比**：Harness 是「操作系统 + 应用框架」，superagent 是「跑在它上面的一个值班机器人应用」。

---

## 十、一句话总结

> **监听 GitHub（可选飞书）→ 白名单的人提需求 → Eval 判断 → 开 Ticket → 独立 Executor 干活 → 开 PR 回原对话 → 面板只显示需要人的事。**
> 核心信条：**尽量少占用人的注意力，尽量多交付价值。**
> 研究原型，别上生产；但「怎么样才算不打扰人」这套设计值得抄。
