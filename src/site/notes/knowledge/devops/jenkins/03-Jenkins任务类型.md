---
{"dg-publish":true,"permalink":"/knowledge/devops/jenkins/03-Jenkins任务类型/","tags":["Jenkins","CI/CD","DevOps","Job","Pipeline","任务类型"],"dg-note-properties":{"date":"2026-09-16","tags":["Jenkins","CI/CD","DevOps","Job","Pipeline","任务类型"]}}
---


# 🧩 Jenkins 任务类型（Job）

> Jenkins 功能模块全景 · 第三章。Jenkins 里「一个任务」有哪几种形态，各自适合什么场景。

---

## 一、Freestyle Project（自由风格）

最基础的任务类型，全部通过 **Web UI 点点点**配置：

- 源码管理（Git/SVN）
- 构建触发器
- 构建步骤（执行 Shell、调用 Maven、跑 Windows 批处理…）
- 构建后操作（归档产物、发通知、触发下游）

**优点**：上手快、无门槛。
**缺点**：配置不进版本库、复杂流程难以维护、无法复用。
**适用**：一次性任务、简单脚本、快速验证。

## 二、Pipeline（流水线）

把整个 CI/CD 流程写成**代码（Jenkinsfile）**，存进 Git 随代码版本化。

| 风格 | 特点 |
|:-----|:-----|
| **Declarative**（声明式） | `pipeline { }` 结构化语法，有固定骨架（agent/stages/steps/post），易读易维护，**推荐** |
| **Scripted**（脚本式） | 纯 Groovy `node { }`，灵活度最高、可写逻辑，但可读性差 |

**优点**：配置即代码、可评审、可复用、可断点重放（Replay）。
**适用**：绝大多数生产流水线。

## 三、Multibranch Pipeline（多分支流水线）

**自动发现**仓库里的分支和 PR，**每个分支自动生成一条流水线**。

- 分支创建/删除 → 流水线自动增删
- 支持分支来源（Branch Source）插件：GitHub / GitLab / Gitea / Bitbucket
- PR 会自动触发构建并把结果回写（配合 `Jenkinsfile` 里的 `when { changeRequest() }`）

**适用**：多分支并行开发、MR/PR 门禁——**车载多分支项目就是靠它**。

## 四、Organization Folder（组织文件夹）

比 Multibranch 更高一层：**按组织/用户扫描其下所有仓库**，自动为每个仓库建 Multibranch 流水线。

- 插件：GitHub Organization / Bitbucket Team / GitLab Group
- 适合：一个组织下几十上百个仓库统一纳管

## 五、Folder（文件夹）

纯**组织结构**用途，不是构建类型：

- 给任务分组、树状展示
- 权限可按 Folder 隔离
- 环境变量/凭证可在 Folder 级别共享

## 六、Matrix Project（矩阵项目）

**一维/多维参数组合**，自动拆成多个并行子任务。

- 例：`{OS: [linux, windows]} × {JDK: [8, 11, 17]}` = 6 个组合并行跑
- 适合：多平台 × 多版本兼容性测试

## 七、对比速查

| 类型 | 配置方式 | 版本化 | 自动发现 | 典型场景 |
|:-----|:-----|:---:|:---:|:-----|
| Freestyle | UI | ❌ | ❌ | 简单/一次性任务 |
| Pipeline | Jenkinsfile | ✅ | ❌ | 标准 CI/CD |
| Multibranch | Jenkinsfile | ✅ | ✅ 分支/PR | 多分支开发、MR 门禁 |
| Organization | Jenkinsfile | ✅ | ✅ 全组织 | 大规模仓库纳管 |
| Folder | UI/API | ❌ | ❌ | 分组与权限隔离 |
| Matrix | Jenkinsfile | ✅ | ❌ | 多平台多版本矩阵 |

## 八、怎么选（决策顺序）

```
只是跑个脚本？            → Freestyle
要把流程代码化？          → Pipeline
有多个分支要分别构建？    → Multibranch Pipeline
有几十个仓库要统一管理？  → Organization Folder
要按平台/版本做矩阵？     → Matrix
只是想分类归档？          → Folder
```

> 💡 **实践共识**：能上 Pipeline 就别用 Freestyle。Freestyle 只留给「临时跑一下」的场景，否则配置散在 UI 里，迁移和复用都是灾难。

---

> 🦐 虾管家 · 2026-09-16
