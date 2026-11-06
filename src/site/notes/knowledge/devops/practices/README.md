---
{"dg-publish":true,"permalink":"/knowledge/devops/practices/README/","title":"实战演练 导航","tags":["导航","索引","实战演练"],"dg-note-properties":{"title":"实战演练 导航","tags":["导航","索引","实战演练"]}}
---


# 🛠️ 实战演练 · 目录导航

> 落地实战：Jenkins（部署/迁移/排障）→ CI 工程化（版本规范 + GitLab CI + ci-lib）→ k3s 集群 → 网关与告警 → 构建系统（CMake/Bazel）→ 制品与 Harbor → AIOps 排障
> 全部按「问题 → 根因 → 解决」组织，可直接照做复现。

返回 [[首页\|🏠 首页]]

---

## 📊 目录概览

| 类别 | 文档 |
|:--|:--|
| **CI/CD 主线** | 01 Jenkins 部署迁移 · 02 Jenkins 排障 · 03 CI 工程化 |
| **集群运维** | 04 k3s 集群 · 05 网关与告警 |
| **构建与制品** | 06 CMake/Bazel · 07 制品管理与 Harbor |
| **专项** | 08 AIOps 排障 |

---

## 文档索引

| # | 文档 | 说明 |
|:--|:-----|:-----|
| 01 | [[knowledge/devops/practices/01-Jenkins部署与迁移实战\|01-Jenkins部署与迁移实战]] | Master + 三种 Agent 接入（Linux/Windows/K8s），附迁移 k3s 完整方案 |
| 02 | [[knowledge/devops/practices/02-Jenkins流水线实战\|02-Jenkins流水线实战]] | **排障案例库**：50+ 条按问题分类，DNS/语法/Agent/Shell/状态/配置六大类 |
| 03 | [[knowledge/devops/practices/03-CI工程化实战\|03-CI工程化实战]] | 版本发布规范（SemVer/GitFlow/门禁）+ GitLab CI 双平台对比 + ci-lib 共享库 |
| 04 | [[knowledge/devops/practices/04-k3s集群部署实战\|04-k3s集群部署实战]] | 三节点跨地域集群 + WireGuard 统一平面 + 监控栈 + 问题速查 19 条 |
| 05 | [[knowledge/devops/practices/05-网关与告警接入实战\|05-网关与告警接入实战]] | APISIX 网关迁移（含 RCE 事件）+ Alertmanager 告警出口 + 节点安全审计 |
| 06 | [[knowledge/devops/practices/06-构建系统实战-CMake与Bazel\|06-构建系统实战-CMake与Bazel]] | CMake 七大能力实战（交叉编译/链接脚本/闭源库）+ Bazel 改造（含远程缓存、toolchain） |
| 07 | [[knowledge/devops/practices/07-制品管理与Harbor实战\|07-制品管理与Harbor实战]] | 制品管理全景（工具选型/命名规范/生命周期）+ Harbor 实战（RBAC/漏洞门禁/GC/CI 集成） |
| 08 | [[knowledge/devops/practices/08-AI-Agent辅助CI排障实战\|08-AI-Agent辅助CI排障实战]] | 把 CI 失败日志喂给 LLM 做根因定位，含 RAG 设计与实测 |

---

## 🔍 按症状找文档

| 症状 | 去哪查 |
|:--|:--|
| Jenkins 装不起来 / Agent 接不上 | 01 |
| `Could not resolve host` / 镜像拉不下来 | 02 §1 |
| `MissingPropertyException` / Groovy 语法错 | 02 §2 |
| `docker: not found` / agent 起不来 | 02 §3 |
| 构建 SUCCESS 但产物不对 / 门禁假绿 | 02 §4 · §5 |
| 配置改了不生效 / 重启后消失 | 02 §6 |
| 版本号乱 / Tag 没规范 | 03 第一部分 |
| Jenkins 与 GitLab CI 怎么选 | 03 第二部分 |
| 集群节点 NotReady / 网络不通 | 04 问题速查 |
| 家宽节点 `kubectl logs` 502 | 04 家宽节点接入 |
| 告警发不出去 / 企微收不到 | 05 第二部分 |
| CMake 交叉编译 / 链接脚本 / 闭源库 | 06 第一部分 |
| 构建慢 / 缓存不共享 / 想换 Bazel | 06 第二部分 |
| 镜像仓库怎么选 / Harbor 怎么配 | 07 第一部分 |
| 镜像扫描不生效 / 漏洞门禁拦不住 | 07 第二部分 |
| CI 失败日志看不懂 | 08 |
