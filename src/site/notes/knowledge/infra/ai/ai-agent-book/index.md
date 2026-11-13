---
{"dg-publish":true,"permalink":"/knowledge/infra/ai/ai-agent-book/index/","title":"《深入理解 AI Agent》精读 · 目录","tags":["导航","索引","AI","Agent","读书笔记"],"dg-note-properties":{"title":"《深入理解 AI Agent》精读 · 目录","tags":["导航","索引","AI","Agent","读书笔记"]}}
---


# 📖 《深入理解 AI Agent》精读

> 李博杰《深入理解 AI Agent：设计原理与工程实践》（*AI Agents in Depth*）
> 全书核心公式：**Agent = LLM + 上下文 + 工具**

返回 [[knowledge/infra/ai/index\|🧠 AI 导航]] · [[首页\|🏠 首页]]

---

## 关于这本书

| 项 | 内容 |
|:--|:--|
| 书名 | 《深入理解 AI Agent：设计原理与工程实践》 |
| 作者 | 李博杰（@bojieli） |
| 仓库 | `github.com/bojieli/ai-agent-book` |
| 在线阅读 | `bojieli.github.io/ai-agent-book` |
| 许可 | Apache 2.0，全文 Markdown 开源 |
| 规模 | 10 章正文 + 109 个实验 + 15 种语言 |

**姊妹篇**：《深入理解 AI Infra：量化分析与系统设计》（`github.com/bojieli/ai-infra-book`）

> 本整理基于 **2.0 版**书稿。2.0 版把原第四章「异步交互」与原第九章「多模态 Agent」合并为新的第六章，后续章节顺延。

---

## 章节索引

| # | 文档 | 主题 | 核心一句话 | 实验数 |
|:--:|:--|:--|:--|:--:|
| 01 | [[knowledge/infra/ai/ai-agent-book/01-AI-Agent-入门\|01-AI-Agent-入门]] | AI Agent 入门 | Agent = LLM + 上下文 + 工具；Harness 工程才是竞争力 | 4 |
| 02 | [[knowledge/infra/ai/ai-agent-book/02-上下文工程\|02-上下文工程]] | 上下文工程 | 上下文决定能力上限：KV Cache、提示工程、Agent Skills、上下文压缩 | 10 |
| 03 | [[knowledge/infra/ai/ai-agent-book/03-用户记忆和知识库\|03-用户记忆和知识库]] | 用户记忆和知识库 | 跨会话记住用户、接入外部知识：用户记忆、RAG、结构化索引、知识图谱 | 12 |
| 04 | [[knowledge/infra/ai/ai-agent-book/04-工具\|04-工具]] | 工具 | 工具是 Agent 的双手：MCP 协议、感知/执行/协作三类工具与主动工具发现 | 5 |
| 05 | [[knowledge/infra/ai/ai-agent-book/05-Coding-Agent-与通用-Agent\|05-Coding-Agent-与通用-Agent]] | Coding Agent 与通用 Agent | 代码是「能创造新工具的工具」，生产级 Coding Agent 全景 | 16 |
| 06 | [[knowledge/infra/ai/ai-agent-book/06-交互-观察与动作空间的扩展\|06-交互-观察与动作空间的扩展]] | 交互：观察与动作空间的扩展 | 从模态与时序两个维度扩展观察与动作空间：异步与事件驱动、语音、Computer Use、机器人操作 | 14 |
| 07 | [[knowledge/infra/ai/ai-agent-book/07-Agent-的评估\|07-Agent-的评估]] | Agent 的评估 | 把表现变成可比较信号：评估环境、指标、统计显著性、评估驱动选型 | 14 |
| 08 | [[knowledge/infra/ai/ai-agent-book/08-模型后训练\|08-模型后训练]] | 模型后训练 | 预训练/SFT/RL 三阶段：何时选 SFT、何时选 RL，工具调用内化、样本效率 | 19 |
| 09 | [[knowledge/infra/ai/ai-agent-book/09-Agent-的持续进化\|09-Agent-的持续进化]] | Agent 的持续进化 | 从运行轨迹获得学习信号，更新知识、指令、程序与参数 | 9 |
| 10 | [[knowledge/infra/ai/ai-agent-book/10-多-Agent-协作\|10-多-Agent-协作]] | 多 Agent 协作 | 群体智能高于个体：协作框架、上下文共享/隔离、涌现的「Agent 社会」 | 6 |

> 合计 **10 章 / 109 个实验**

---

## 全书主线

```
                 Agent = LLM + 上下文 + 工具
                            │
        ┌───────────────────┼───────────────────┐
        │                   │                   │
     【大脑】            【眼睛】            【手脚】
      LLM                上下文              工具
        │                   │                   │
   Ch8 后训练           Ch2 上下文工程        Ch4 工具
   Ch9 持续进化         Ch3 记忆与知识库      Ch6 交互扩展
                            │
                        Ch10 多 Agent
                            │
                     Ch7 评估（贯穿全书）
```

## 两个贯穿全书的概念

| 概念 | 含义 |
|:--|:--|
| **Harness** | Agent 边界内、模型之外的运行与治理层：上下文管理 + 工具接口 + 约束 + 验证 + 纠正 |
| **ReAct 循环** | 想 → 做 → 看 → 想，不断重复直到任务完成 |
