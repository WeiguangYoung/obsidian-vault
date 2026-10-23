---
{"dg-publish":true,"permalink":"/knowledge/devops/practices/06-AI-Agent辅助CI排障实战/","tags":["DevOps","AIOps","LLM","CI/CD","Jenkins","排障","RAG","实战"],"dg-note-properties":{"date":"2026-09-22","tags":["DevOps","AIOps","LLM","CI/CD","Jenkins","排障","RAG","实战"]}}
---


# AI Agent 辅助 CI 排障实战：从架构到 Demo

> 构建失败日志动辄几千行，人工定位根因靠经验、靠感觉。这篇文章记录一个**能跑起来的最小可用方案**：
> 把 Jenkins 失败日志喂给 LLM，自动产出「根因 + 分析过程 + 修复方案 + 分类 + 相似历史案例」。
> 不是概念科普，是**架构取舍 + 踩坑记录 + 实测数据**。

---

## 一、问题：CI 失败日志为什么难查

先看一组真实数据。一个 Java 前后端同仓项目（Halo）的一次失败构建，控制台日志：

```
365,618 字节
3,056 行
```

而真正的失败信息只有**最后 3 行**：

```
Step 1/17 : FROM eclipse-temurin:21-jre AS builder
Get "https://registry-1.docker.io/v2/": net/http: request canceled while waiting for connection
ERROR: script returned exit code 1
```

中间那 3,000 行是什么？是编译 warning、是前端 pnpm 装包、是单元测试日志、是 `UP-TO-DATE`。

**问题的本质**：
- **信噪比极低**——错一行藏在几千行噪音里
- **表象与根因分离**——`exit code 1` 是表象，真因在上面几行；`BUILD FAILED` 永远是最后一行，但它不告诉你为什么
- **依赖经验**——"Docker Hub 被墙""Node 版本不够"这些坑，踩过才知道
- **重复劳动**——同一个坑，换个项目又会碰到，但知识没沉淀

---

## 二、业界怎么做（先看别人踩过的路）

| 厂商 | 方案 | 核心思路 |
|:---|:---|:---|
| **字节跳动** | 单 Agent 下钻式诊断 | 表象现象 → 采集日志/指标 → 知识推理 → 生成检查步骤 → 验证定位 |
| **微软 RCACopilot** | LLM + 向量检索历史故障 | 历史故障库嵌入 → LLM 判断是否旧故障 |
| **华为云** | 多 Agent 协同 | 主管 Agent → 调度检测/根因/分析/修复 Agent |
| **清华** | ChatTS / SparseRCA | 指标问答大模型、无监督根因分析 |

**共同范式**：

```
自然语言输入 → Agent 采集数据 → LLM 推理 → 根因 + 修复建议
```

**关键差异点**在「Agent 采集数据」这一步：是只喂日志，还是同时给指标、给历史案例、给代码 diff。给的上下文越准，诊断越准。

---

## 三、架构设计：三段式流水线

我做的 demo（`aiops-log-doctor`）采用**极简三段式**：

```
Jenkins 失败日志
    ↓
① 日志压缩       ← 本地正则粗筛，省 token、去噪声
    ↓
② 案例检索(RAG)  ← 从历史故障库召回相似案例
    ↓
③ LLM 诊断       ← 结构化输出根因/修复
    ↓
根因 + 分析过程 + 修复方案 + 分类 + 相似案例
```

### 3.1 为什么先本地压缩

**直接丢全量日志给 LLM 又贵又不准**——上下文塞满噪声，模型反而抓不住重点。

压缩策略：**保留「错误信号」所在行 + 上下文窗口，丢弃环境噪音**。

```python
# 命中即保留（大小写不敏感）
ERROR_SIGNALS = [
    r"\bERROR\b", r"\bFATAL\b", r"\bfailed\b",
    r"\bexception\b", r"\bcaused by\b", r"\bwhat went wrong\b",
    r"\bBUILD FAILED\b", r"\bExecution failed for task\b",
    r"error:", r"exit code [1-9]", r"script returned exit code",
    r"\bTimeout\b", r"\bCould not (resolve|GET|download)\b",
]

# 命中即丢弃（噪音）
NOISE_SIGNALS = [
    r"^\s*\d+%\s",                      # 进度条
    r"^\s*[a-f0-9]{12}:\s+(Pulling|Downloading|...)",  # 拉镜像层
    r"^\s*>\s+Task .*(UP-TO-DATE|SKIPPED)",            # Gradle 跳过
    r"^\s*(at |\s+at )[a-z]+\.[a-z]",   # 堆栈帧
]
```

**实测效果**：`3,056 行 → 154 行`，压掉 **95%**，成本从"不敢想"降到 **¥0.02/次**。

> **工程要点**：错误行前后各留 N 行上下文（默认 3），因为"真因常在上文"。没命中任何信号时退化为「取尾部 N 行」——错误通常在末尾。

### 3.2 为什么用轻量 RAG 而不是向量库

**这是一个刻意的取舍。**

向量检索（embedding + 向量库）是 RAG 的标准做法，但在这个场景里我**没用**，理由有三：

| 考量 | 说明 |
|:---|:---|
| **没有现成 embedding** | DeepSeek 官方**不提供 embedding 接口**；本地起 BGE 之类模型又太重 |
| **症状词是字面的** | 故障日志的关键词是**报错原文**——`registry-1.docker.io`、`Promise.withResolvers is not a function`、`dubious ownership`。这类字符串**关键词/正则匹配本来就准** |
| **案例库不大** | 几十上百条案例，暴力打分毫秒级，不需要 ANN 索引 |

**打分规则**（简单但有效）：

```
symptoms 关键词命中   → +1.0 / 个
patterns 正则命中     → +2.5 / 个   （正则更精确，权重更高）
总分 ≥ min_score(1.0) 才进候选，取 Top-K(3)
```

案例库用 YAML 维护，加一条案例就是加一段：

```yaml
- id: docker-hub-timeout
  title: Docker Hub 被墙，FROM 拉基础镜像超时
  tags: [网络问题, 环境问题]
  symptoms:
    - "registry-1.docker.io"
    - "Client.Timeout exceeded"
    - "Step 1/N : FROM"
  patterns:
    - 'Get "https://registry-1\.docker\.io/v2/"'
  root_cause: |
    Dockerfile 里 FROM <image> 未带 registry 前缀，默认走 Docker Hub，
    国内构建机不可达，构建在第 1 步就超时。
    注意：docker login 到私有仓不能兜底，因为 FROM 没带 registry 前缀。
  fix: |
    1) 给 docker daemon 配 registry-mirrors
    2) 或改 Dockerfile 的 FROM 显式指向国内镜像
```

**实测**：3 个样例日志，RAG 召回的 **Top-1 全部命中**正确案例（分数 7.5 ~ 11.0）。

> **什么时候该上真·向量库**：案例涨到几千条、或症状描述变得"非字面"（比如需要用自然语言描述业务语义）时，再换 embedding + Milvus/Chroma。**先简单后复杂**。

### 3.3 Prompt 设计：要求 LLM「区分表象与根因」

System Prompt 的核心约束（节选）：

```
诊断时请遵循：
1. 先定位真正的失败点，不要被前面的 warning/噪音带偏
2. 区分表象与根因：例如"exit code 1"是表象，真因在上面几行
3. 结合常见坑：网络（墙/DNS/超时）、版本不匹配、权限、依赖缺失、配置错误
4. 给出可执行的修复步骤，命令要具体

输出严格使用下面的 Markdown 结构：
## 🔍 根因
## 📋 分析过程
## 🔧 修复方案
## 🏷️ 分类
## 📎 相似案例
```

**强约束输出结构**的价值：结果可解析、可比较、便于后续接入自动化（比如按 `分类` 打 tag 路由给不同值班人）。

RAG 召回的案例作为**参考块**注入 User 消息，并明确告知：

```
【历史相似案例（供参考，勿照抄：先确认本次日志是否真与之匹配）】
### 参考案例：Docker Hub 被墙，FROM 拉基础镜像超时
- 根因：...
- 修复：...
```

那句"**勿照抄**"很关键——防止模型"过度自信"地把旧案例的结论硬套在本次日志上。

---

## 四、Demo 实现

### 4.1 项目结构

```
aiops-log-doctor/
├── doctor/
│   ├── extractor.py   # 日志压缩（错误信号 + 上下文窗口）
│   ├── jenkins.py     # Jenkins API 客户端
│   ├── rag.py         # 轻量 RAG 案例检索
│   ├── prompt.py      # 诊断 Prompt 模板
│   └── llm.py         # DeepSeek 官方 API 客户端
├── cases/knowledge.yaml  # 历史故障案例库
├── samples/              # 真实失败日志样例
├── main.py               # CLI 入口
└── requirements.txt      # 仅 requests + pyyaml
```

**依赖极简**：只有 `requests`（调 API）和 `pyyaml`（读案例库）。零重依赖。

### 4.2 用法

```bash
# 1) 连通性自检
python main.py --check

# 2) 🔥 全自动：从 Jenkins 拉最近一次失败构建，直接诊断
python main.py --job halo/main --auto

# 3) 指定构建号
python main.py --job halo/main --auto --build 13

# 4) 本地文件 / 管道
python main.py --file build.log --job halo/main
docker exec jenkins cat .../builds/13/log | python main.py --job halo/main
```

### 4.3 Jenkins 自动拉取

`doctor/jenkins.py` 处理了两件麻烦事：

**① 多分支任务路径解析**——Multibranch 任务的 URL 是 `/job/halo/job/main/`，普通任务是 `/job/hello-jenkins/`：

```python
def _job_path(self, job: str) -> str:
    """halo/main → job/halo/job/main；hello-jenkins → job/hello-jenkins"""
    parts = [p for p in job.split("/") if p]
    return "/".join(f"job/{p}" for p in parts)
```

**② 定位"最近一次失败"**——不是取 lastBuild（可能是成功的），而是遍历构建历史找第一个非 SUCCESS：

```python
def get_last_failed_build(self, job):
    data = self._get(f"{path}/api/json",
                     tree="builds[number,result,url,duration]{0,50}").json()
    for b in data.get("builds", []):
        if b.get("result") and b["result"] != "SUCCESS":
            return BuildInfo(...)   # 找到第一个失败
    return None   # 全是成功 → 调用方退化成最近一次
```

---

## 五、实测：AI 到底诊断得怎么样

用真实的失败日志（不是造的）跑了 4 个案例，**全部命中**：

| 案例 | 压缩 | RAG Top-1 | 花费 |
|:---|:---|:---|:---|
| Docker Hub 被墙 | 35→20 行 | ✅ docker-hub-timeout (9.0) | ¥0.019 |
| Node 版本不兼容 | 23→16 行 | ✅ node-version-too-old (7.5) | ¥0.029 |
| Gradle 分发超时 | 26→20 行 | ✅ gradle-dist-timeout (11.0) | ¥0.015 |
| **全自动（3056 行真日志）** | **3056→154 行** | ✅ docker-hub-timeout | ¥0.022 |

### AI 超出预期的几个表现

**① 自主推理关联**（不是读日志，是"分析"日志）

> "日志里 pnpm 下载到下一行之间静默了 **5 分 17 秒**（01:51:48 → 01:57:05）——和本次失败同属外网慢速下载问题。而且本 commit message 是 `ci: use Tencent mirror for gradle distribution (avoid timeout)`，说明团队在处理超时问题，但只改了 Gradle，Docker 源没同步处理。"

把**时间差**和**commit message** 对照起来推理——这已经超出"文本理解"了。

**② 抵抗误导信息**

日志里有 `@types/node@22.18.0` 这个路径，但 AI 明确指出：

> "路径里的 `@types/node@22.18.0` 只是 TypeScript 类型包，**不代表 Jenkins 实际运行 Node 22**；真正执行的是 `:ui:buildPackages` 调用的 `node`。"

没被表面信息骗到。

**③ 严谨的排除法**

Gradle 超时案例里，AI 主动排除其他可能：

> "`timeout (10000ms)` + `SocketTimeoutException` 是典型的**网络读超时**，不是 404、不是校验失败（校验失败会报 checksum/Corrupted 之类），所以先排除版本号不存在、包损坏。"

**④ 完整的分析过程（可溯源）**

AI 给出的每条结论都引用了具体日志行——这满足了 AIOps 的"**证据可溯源**"底线，人敢信、能复核。

### 一个我都没强调的洞察

Docker Hub 案例里，AI 指出：

> "`docker login` 成功反而有迷惑性：登录的是阿里云 ACR，而 `FROM eclipse-temurin:21-jre` 因为没有 registry 前缀，走的仍然是 Docker Hub——**登录 ACR 完全兜不住这个请求**。"

这个"登录的 registry ≠ 拉取的 registry"的关联，我自己写排障记录时都没单独点出来。

---

## 六、踩过的坑（工程细节）

### 6.1 YAML 的冒号陷阱

案例库里的症状词常含冒号，比如 `"Step 1/N : FROM"`。**YAML 里含冒号的字符串必须加引号**，否则被解析成字典：

```yaml
symptoms:
  - "Step 1/N : FROM"     # ✅ 加引号
  - Step 1/N : FROM       # ❌ 会被解析成 dict，后续 .lower() 报错
```

报错是 `'dict' object has no attribute 'lower'`——这个报错信息完全看不出是 YAML 问题。

### 6.2 日志压缩的边界

- **上下文窗口不能太大**：`context=3` 效果好；调到 10 会把大量噪音重新捞回来
- **噪音正则要保守**：误杀关键行比多留噪音更危险。比如别把 `WARN` 一刀切——有些 WARN 就是根因
- **超长时保留尾部**：压缩后仍超 `max_lines` 时，`[-max_lines:]` 保留最新的（越靠近失败点越相关）

### 6.3 Jenkins API 认证

- Jenkins 支持 `user:password`（Basic Auth）和 `user:apiToken` 两种。**密码能用但不安全**，生产建议用 API Token
- 端点 `…/consoleText` 拿纯文本日志，比 `/log`（HTML）好解析
- **首次集成容易踩 401**——先 `curl -u user:pass …/api/json` 确认凭证对

---

## 七、从 Demo 到生产还差什么

这个 demo 是**能跑的验证**，不是生产系统。要落地还得补：

| 方向 | 说明 | 对标 |
|:---|:---|:---|
| **批量扫描** | 扫所有 job 的失败构建，出趋势报告 | 值班场景 |
| **告警联动** | 构建失败 → 自动诊断 → 推飞书/企微 | 实时性 |
| **真·向量库** | 案例涨到几千条时换 embedding + Milvus | 微软 RCACopilot |
| **反馈闭环** | 人工标记诊断"有用/没用"，回流优化 Prompt 和案例库 | 持续变准 |
| **私有化** | 代码/日志敏感时，本地跑 vLLM/Ollama | 数据安全 |
| **多 Agent** | 检测/根因/修复分 Agent 协同 | 华为云方案 |

**落地路线**（从快赢到深水区）：

```
Phase 1  日志诊断单点（本 demo）        → 验证可行性
Phase 2  接 Jenkins/告警，自动触发诊断   → MTTR 降低
Phase 3  案例库沉淀 + 反馈闭环          → 越用越准
Phase 4  私有化 + 多 Agent              → 生产级
```

---

## 八、总结

**几个关键结论**：

1. **AIOps ≠ 把日志丢给大模型**。直接丢 = 又贵又不准。**先压缩、再检索、后推理**才是工程做法。
2. **RAG 要看场景选实现**。症状词是字面报错时，关键词+正则匹配就够，不必上 embedding。**先简单后复杂**。
3. **Prompt 要约束"区分表象与根因"**，并要求证据可溯源——否则 AI 的结论没人敢信。
4. **实测成本极低**：一次诊断 ¥0.02、几秒钟。对比人工查日志的十几分钟，ROI 很明显。
5. **知识要沉淀成案例库**。同一个坑不该踩第二次——这正是"经验"从人脑迁移到系统的过程。

**这个 demo 最大的价值不在代码**，而在于它验证了一条路径：
**把"老师傅的排障经验"变成可检索、可复用、可传承的结构化知识。**

---

> **相关实践**：本文的案例库素材来自一次真实的 CI/CD 攻坚——为 Halo 项目从零搭建 Jenkins 流水线（含 Docker Hub 被墙、Node 版本、Gradle 超时等 9 个坑的完整排障），详见 `02-Jenkins流水线实战.md`。
