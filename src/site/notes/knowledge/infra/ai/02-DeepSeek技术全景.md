---
{"dg-publish":true,"permalink":"/knowledge/infra/ai/02-DeepSeek技术全景/","title":"DeepSeek 技术架构全景","tags":["DeepSeek","大模型","MoE","MoE架构","推理引擎","Agent","DeepInfer","DeepSeek Harness"],"dg-note-properties":{"category":"AI / 大模型","title":"DeepSeek 技术架构全景","date":"2026-11-01","tags":["DeepSeek","大模型","MoE","MoE架构","推理引擎","Agent","DeepInfer","DeepSeek Harness"]}}
---


# DeepSeek 技术架构全景

> 从「模型」到「引擎」再到「Agent」，DeepSeek 已经是三层全栈：
> **DeepSeek-V4.1-Flash（模型）→ DeepInfer（推理引擎）→ DeepSeek Harness（Agent 框架）**。
> 本篇是总纲，单点深入看各节对应的文档。

## 一、三层栈全景

```
┌─────────────────────────────────────────────┐
│  Agent 层   DeepSeek Harness                  │
├─────────────────────────────────────────────┤
│  引擎层     DeepInfer（推理服务）              │
│             算子: FlashMLA / DeepGEMM / DeepEP│
│             解码: DSpark                       │
├─────────────────────────────────────────────┤
│  模型层     DeepSeek-V4.1-Flash               │
│             MoE + CSA2 + Engram               │
└─────────────────────────────────────────────┘
```

> 有意思的是：这三层**互相咬合**。模型的结构（Engram、稀疏注意力）决定了引擎必须定制算子；引擎的调度特性又反过来影响 Agent 层的并发设计。

## 二、模型层：DeepSeek-V4.1-Flash

> 详见本篇「二、模型层」

### 2.1 关键规格

| 指标 | 数值 |
|:----|:----|
| 主干参数 | **552B** |
| Engram 条件记忆参数 | 196B |
| 含视觉编码器总计 | 763B |
| 架构 | **MoE + 因果编解码非对称结构**（20 层 encoder + 20 层 decoder） |
| 激活参数 | 输入 **8B** / 输出 **16B** |
| 每层专家 | 1 共享专家 + 384 路由专家，每 token 激活 6 个 |
| 上下文 | **1M tokens** |
| 最大输出 | 384K tokens |
| 模态 | 原生多模态（文本 + 图像 → 文本） |

### 2.2 三个结构亮点

1. **非对称激活**：prefill 只激活 8B，decode 激活 16B —— 长输入的算力开销被压下来
2. **CSA2（压缩稀疏注意力 v2）**：Full / Reindex / Reuse 三种模式跨层共享 KV，**KV cache 降到约 890 字节/token**（约为上一代 1/4）
3. **Engram 条件记忆**：196B 参数的 n-gram 查找，**把「记忆」从「计算」里剥离**，稀疏访问不占主算力

> 一句话：**用结构换成本** —— 不靠堆算力，靠「什么时候只算一点点」。

## 三、引擎层：DeepInfer

> 详见本篇「三、引擎层」

### 3.1 定位

- **2026-10-20 开源，Apache 2.0**
- 不是 Demo，是**支撑 V4.1-Flash 线上服务的同款框架**
- 基于 vLLM 深度改造，参考 SGLang / TensorRT-LLM，**调度与采样核心已重写**

### 3.2 内置技术栈

| 组件 | 作用 |
|:----|:----|
| **FlashMLA** | 多头潜在注意力算子 |
| **DeepGEMM** | 高性能矩阵计算库 |
| **DeepEP** | 专家并行通信库（MoE 的 all-to-all） |
| **DSpark** | 推测解码，动态验证（比固定验证少验 ~30% 候选） |
| **SWA Bounded Replay** | 长文本稀疏注意力，减少 prefill 计算 |

### 3.3 硬件门槛与开源边界

| 项 | 说明 |
|:----|:----|
| 支持硬件 | NVIDIA **SM100**（CUDA 13）/ 华为**昇腾 950**（CANN 9.2） |
| 软件要求 | Linux、GCC ≥ 13、Python 3.12 |
| **未开源** | KV Cache 系统、Prefill/Decode 分离部署、跨副本负载均衡 |

> **关键认知**：开源的是**离线推理核心**，不是「开箱即用的推理服务器」。想搭线上服务，KV Cache 与 PD 分离得自己实现。

### 3.4 性能参考

| 指标 | 数值 |
|:----|:----|
| 官方日均处理 | 155 万亿 token |
| 官方首字延迟 | ~400ms |
| 单卡解码峰值 | 26,000 token/s |
| 第三方对比（GPUStack，8 卡） | 吞吐比 vLLM **+25.2%**，首字延迟约为 vLLM 2 倍 |

## 四、Agent 层：DeepSeek Harness

> 详细内容见「附三、DeepSeek Harness 详解」。

### 4.1 定位

**DeepSeek Harness**（`deepseek-harness`）是 DeepSeek 开源的**通用 agent harness**，架构理念 **"Everything is a Plugin"**，构建在 [Cordis](https://github.com/cordiverse/cordis) 之上。

### 4.2 要点速览

- **npx 一条命令启动**：`npx @deepseek-ai/dsh web` → Web UI 跑在 `127.0.0.1:3080`
- **8 种 entry mode**：web / headless / acp / sdk / sdk-minimal / tui / 自定义 profile / plugin 管理
- **profile 机制**：插件 bundle 有序叠加，配置放 `$DSH_HOME/profiles/<name>`
- **核心概念**：Everything is a Plugin —— 连 Web UI、CLI、SDK 本身就是 profile

---

## 五、三层之间的关系（这是重点）

| 层 | 优化目标 | 手段 | 对下层的依赖 |
|:----|:----|:----|:----|
| 模型 | 单位 token 成本 | 结构稀疏（Engram / CSA2） | — |
| 引擎 | 吞吐与延迟 | 定制算子 + 动态推测解码 | 依赖模型结构（非通用框架） |
| Agent | 人的时间 | 事件驱动 + 常驻 + 并行 | 依赖引擎的并发与首字延迟 |

**串起来看**：模型把「每次计算变少」→ 引擎把「每次计算变快、批得更大」→ Agent 把「人不用等」。
三者合起来就是 DeepSeek 对外讲的**「同等能力下更便宜」**。

## 六、对求职 / 工作的实际价值

结合你的方向（DevOps / 平台 / AI 基础设施）：

| 论文/项目 | 可讲的点 |
|:----|:----|
| **V4.1-Flash 架构** | 非对称激活、KV cache 压缩、MoE 专家并行 —— 面试常问「大模型推理为什么贵」 |
| **DeepInfer** | 推理服务的部署形态、PD 分离、KV Cache 管理、昇腾适配 |
| **DeepSeek Harness** | 插件化 agent 底座（TypeScript/Cordis）、profile 机制、8 种 entry mode |
| **推理加速与部署** | 结合 [[knowledge/infra/ai/01-模型推理加速与部署\|模型推理加速与部署]]，讲调度与显存优化 |

> 一句话包装：**「我不只会部署模型，我理解模型结构如何决定推理成本和调度策略」** —— 这是 AI 平台岗最稀缺的能力。

## 附一、DeepSeek-V4.1-Flash 模型 详解

> DeepSeek 全新模型结构系列里**尺寸最小**的一档，但靠新预训练方式 + 更大规模 RL 后训练，
> 在多项 Agent 基准上**超过了包括 V4 Pro 在内的旗舰模型**，同时更快更便宜。
> 上层引擎是 DeepInfer，见「三、引擎层」。

### 一、规格速览

| 指标 | 数值 |
|:----|:----|
| 主干参数 | **552B** |
| Engram 条件记忆参数 | 196B |
| 含视觉编码器总计 | **763B** |
| 架构 | **MoE + 因果编解码（Causal-Encoder-Decoder）非对称结构** |
| 层数 | 40 层 = 20 层 encoder + 20 层 decoder |
| 每层专家 | 1 个共享专家 + 384 个路由专家，每 token 激活 **6** 个 |
| 激活参数 | prefill **8B** / decode **16B** |
| 上下文 | **1M tokens** |
| 最大输出 | 384K tokens |
| 模态 | 原生多模态（文本 + 图像 → 文本） |
| 开源协议 | MIT（HuggingFace / ModelScope） |
| 首发 | 2026-09-10，最新 **V4.1-Flash-1017**（2026-10-17） |

### 二、为什么是「非对称」的编解码结构

传统 decoder-only 模型：**不管输入多长，每层都用同一套参数算**。
V4.1-Flash 把模型切成两段：

```
输入 tokens ──► [20 层 Encoder, 激活 8B] ──► [20 层 Decoder, 激活 16B] ──► 输出
                    prefill 阶段                  decode 阶段
                    吃大量 token                  一次只出 1 个 token
```

**动机**：

| 阶段 | 特点 | 优化方向 |
|:----|:----|:----|
| Prefill（读输入） | 一次处理几十万 token，**算力密集** | 参数激活越少越省 |
| Decode（生成输出） | 每步只出 1 token，**显存带宽密集** | 参数激活多点换取质量 |

> 所以输入侧 8B、输出侧 16B —— **在长上下文场景里，输入往往占 90%+ 的 token，省这里最值钱**。

### 三、CSA2：压缩稀疏注意力 v2

长上下文最大的敌人是 **KV Cache** —— 1M 上下文下，传统做法光缓存就能吃掉几十 GB 显存。

#### 3.1 三种注意力模式

| 模式 | 作用 |
|:----|:----|
| **Full** | 完整注意力，用于关键层 |
| **Reindex** | 重建索引，跨层复用已有 KV 结构 |
| **Reuse** | 直接复用上层算好的 KV，跳过重算 |

**核心思想：跨层共享 KV** —— 不再是每层各存一份，而是层层之间「接力」，把冗余压掉。

#### 3.2 效果

| 指标 | 数值 |
|:----|:----|
| 全局 KV cache | **约 890 字节 / token** |
| 相对上一代 | 约为 **1/4** |
| KV 精度 | **FP4** 主缓存 |

> FP4 + 跨层复用，是 1M 上下文能跑得起的前提。

### 四、Engram：把「记忆」从「计算」里剥离

这是本代最有意思的架构创新。**Engram = 记忆痕迹**（神经科学术语）。

| | 传统 MoE | Engram |
|:----|:----|:----|
| 参数用途 | 全部用于「计算」 | 一部分专门做「记忆查找」 |
| 规模 | — | 196B 参数做 **n-gram 查找** |
| 访问方式 | 计算密集 | **稀疏查表**，几乎不吃主算力 |
| 与主模型关系 | — | 条件触发：需要时查，不需要时不查 |

**收益**：模型可以存下巨量「事实性记忆」，但**不增加主干的推理成本**。
本质上是一次「**存储与计算解耦**」—— 和计算机体系结构里 cache/主存分离的思路同源。

### 五、DSpark 推测解码

推测解码（Speculative Decoding）：小模型先草拟几个 token，大模型并行验证，验证通过就一次性接收。

V4.1-Flash 用的是**半自回归草稿 + 置信度调度验证**：

- 不是固定数量的草稿，而是按置信度动态决定验多少
- 比固定验证方式**少验约 30% 的候选**

> 结果：decode 阶段的实际速度明显提升，而输出质量无损（验证不通过就回退）。

### 六、多模态与训练配方

| 项 | 说明 |
|:----|:----|
| 视觉编码器 | 已计入 763B 总参数 |
| 输入 | 文本 + 图像 |
| 输出 | 文本 |
| 训练 | 新预训练方式 + 更大规模 RL 后训练 |

> 「尺寸最小 + 超越旗舰」这件事本身说明：**这一代的收益主要来自训练配方和结构，而不是堆参数**。

### 七、对推理成本的直接影响

| 环节 | 传统做法 | V4.1-Flash |
|:----|:----|:----|
| 长输入 prefill | 全参数计算 | 只激活 8B |
| KV cache | 每层独立、FP16/FP8 | 跨层复用 + FP4，约 890 B/token |
| decode | 逐 token 自回归 | DSpark 动态推测解码 |
| 事实记忆 | 靠参数「硬记」 | Engram 稀疏查表 |

**串起来**：省算力（8B 激活）→ 省显存（890 B/token）→ 省时间（DSpark）→ 省参数（Engram）。
这就是官方「同等能力更便宜」的技术底子。

### 八、和 V4 Pro / V4-Flash 怎么选

| 场景 | 建议 |
|:----|:----|
| Agent / 工具调用 / 长文档 | **V4.1-Flash**（基准超 V4 Pro，成本低） |
| 需要极致推理深度 | V4 Pro（部分复杂推理仍占优） |
| 私有化部署 | V4.1-Flash + DeepInfer（官方同源） |
| 老显卡 / 消费级卡 | **都不行**，硬件门槛是 SM100 / 昇腾 950 |

### 九、常见误解澄清

| 误解 | 事实 |
|:----|:----|
| 「Flash 就是小模型」 | 553B 主干 + 196B Engram，总量比很多「大模型」都大；**小的是激活量（8B/16B）** |
| 「MoE 就是省显存」 | MoE 省的是**算力**，显存要装全部专家，**反而更吃显存** |
| 「1M 上下文 = 能记住 1M」 | 上下文窗口 ≠ 有效记忆，超长上下文仍有注意力稀释问题 |
| 「开源了就能自建服务」 | 权重开源，但 DeepInfer 的 KV Cache / PD 分离**未开源** |

### 十、一句话总结

> **结构换成本**：输入 8B 激活 + KV 压到 890 B/token + Engram 稀疏记忆 + DSpark 推测解码，
> 让「最小尺寸的模型」拿到了「超越旗舰的效果」。

## 附二、DeepInfer 推理引擎 详解

> DeepSeek 2026-10-20 开源的**生产级推理引擎**，Apache 2.0。
> 不是 Demo —— 它就是你调 DeepSeek API 时背后跑的那套。
> 模型侧见「二、模型层」。

### 一、基本档案

| 项 | 内容 |
|:----|:----|
| 开源时间 | 2026-10-20 |
| 许可证 | **Apache 2.0**（可商用、可改） |
| 仓库 | `github.com/deepseek-ai/DeepInfer` |
| 配套 | `deepseek-recipe`（MIT，token 编解码组件，Rust 实现 + Python 绑定） |
| 支持模型 | `deepseek-ai/DeepSeek-V4.1-Flash-1017` |
| 热度 | 发布数日 star 破万，HN 榜首 |

### 二、它和 vLLM / SGLang 什么关系

**基于 vLLM 深度改造**，参考了 SGLang 与 TensorRT-LLM 的设计，但：

| 模块 | 处理 |
|:----|:----|
| 调度器 | **重写** |
| 采样计算 | **重写** |
| 模型实现 | 为 V4.1-Flash 结构**定制** |
| 算子 | 全用自家库（FlashMLA / DeepGEMM / DeepEP） |

> **一句话**：长得像 vLLM，骨子里是「为 DeepSeek 模型量身定做」。
> 它**不是通用推理框架** —— 换个模型基本用不了，这是它和 vLLM 最大的区别。

### 三、内置算子全家桶

| 组件 | 全称/作用 | 解决什么 |
|:----|:----|:----|
| **FlashMLA** | Multi-head Latent Attention 算子 | MLA 结构的高效计算与 KV 压缩 |
| **DeepGEMM** | 高性能矩阵乘库 | FP8/FP4 GEMM，Hopper/Blackwell 优化 |
| **DeepEP** | Expert Parallel 通信库 | MoE 的 all-to-all 通信，低延迟高吞吐 |
| **DSpark** | 推测解码 | 半自回归草稿 + 置信度调度验证，少验 ~30% |
| **SWA Bounded Replay** | 滑动窗口注意力 | 长文本 prefill 计算量削减 |
| **TileKernels / DeepSelect** | TileLang 定制算子 | 专家选择、分块计算 |

> 这套东西原本是分散开源的（FlashMLA、DeepGEMM、DeepEP 都单独发过），DeepInfer 把它们**缝成了一个可运行的完整引擎**。

### 四、硬件与软件门槛（很硬）

| 项 | 要求 |
|:----|:----|
| GPU | **NVIDIA SM100**（Blackwell 之后的新一代）+ CUDA 13 |
| NPU | **华为昇腾 950** + CANN 9.2 |
| 系统 | Linux |
| 编译 | GCC ≥ 13 |
| Python | 3.12 |

**不支持**：V100 / A100 / H100 / 消费级 RTX —— 不是性能问题，是**算子按 SM100 指令集写的**。

> **昇腾 950 支持是重点**：MoE 推理在国产算力上终于有了成熟的官方栈。

### 五、性能数据

#### 5.1 官方线上服务

| 指标 | 数值 |
|:----|:----|
| 日均处理 | **155 万亿 token** |
| 平均首字延迟 | **~400ms** |
| 单卡解码峰值 | **26,000 token/s** |

#### 5.2 第三方对比（GPUStack，8 卡 SM100，100 请求 / 16 并发）

| 指标 | DeepInfer | vLLM | 差异 |
|:----|:----|:----|:----|
| 总吞吐 | 1392 tok/s | 1112 tok/s | **+25.2%** |
| 首字延迟 | 4.28s | 2.04s | 约 **2 倍慢** |

> 注意：双方都未充分调优，DeepInfer 外包了一层社区接口。
> **吞吐赢了、首字延迟输了** —— 这很符合 PD 未分离的离线推理特征。

### 六、开源边界（最关键的一节）

DeepSeek 明确说明**三项没开源**：

| 未开源组件 | 原因 | 后果 |
|:----|:----|:----|
| **KV Cache 系统** | 与内部基础设施深度耦合 | 没法直接搭在线服务 |
| **Prefill/Decode 分离部署** | 同上 | 缺服务层核心架构 |
| **跨副本负载均衡** | 同上 | 多机调度得自己写 |

**实际能做什么**：

```bash
# 当前只有离线批量推理入口
python examples/offline_inference.py
```

**实际做不了**：一键起一个像官方 API 那样的在线服务。

> 官方说法是「开源的是**离线推理核心**」。想要线上能力，KV Cache 管理、PD 分离、多副本调度这三块得自己补 —— 或者用 vLLM/SGLang 顶上。

### 七、PD 分离为什么重要（补背景）

```
传统耦合部署：
  同一批 GPU 既做 prefill（算力密集）又做 decode（带宽密集）
  → 两种负载互相干扰，资源利用率都不高

PD 分离：
  Prefill 集群（少量大卡，跑长输入）
       ↓ KV Cache 传输
  Decode 集群（大量卡，持续出 token）
  → 各跑各的最优形态，整体吞吐更高
```

这是当前大模型推理服务的主流架构（vLLM、SGLang、Mooncake 都在做）。
**DeepInfer 没开源这部分，等于把「服务化」的核心留在了内部。**

### 八、适用场景判断

| 推荐用 | 不建议用 |
|:----|:----|
| 私有化部署 **V4.1-Flash** 的团队 | 需要多模型混合部署 |
| 追求与官方 API 一致的成本/性能 | 手上只有 V100/A100/H100 |
| 昇腾 950 上的 MoE 推理 | 想「一键起在线 API」 |
| 研究生产级推理优化 | 需要通用 LLM 推理框架（选 vLLM/SGLang） |

### 九、对 DevOps / 平台岗的意义

| 视角 | 说明 |
|:----|:----|
| **部署形态** | 推理引擎 = 一个需要 NPU/GPU 直通、超大显存、高速互联的服务 |
| **K8s 落地** | 设备插件（nvidia-device-plugin / ascend-device-plugin）+ 大页内存 + RDMA |
| **可观测性** | 首字延迟、TPOT、KV cache 命中率、吞吐是核心指标 |
| **容量规划** | 155 万亿 token/天 这种量级，靠的是 PD 分离 + 批处理 |
| **国产替代** | 昇腾 950 的 CANN 9.2 适配，是当前国产 AI 基建的关键拼图 |

> 结合 [[knowledge/infra/ai/01-模型推理加速与部署\|模型推理加速与部署]] 一起看：驱动 → CUDA/CANN → 算子库 → 推理引擎，四层栈的第四层就是它。

### 十、一句话总结

> **DeepInfer = vLLM 的壳 + DeepSeek 自研算子 + V4.1-Flash 定制**，
> 开源了「算得快」，没开源「服务得好」；硬件门槛把绝大多数人挡在门外，昇腾 950 支持是最大增量。

## 附三、DeepSeek Harness（`deepseek-harness`）详解

> 2026-11-09 核查官方仓库与文档。

### 一、基本档案

| 项 | 内容 |
|:--|:--|
| 仓库 | `github.com/deepseek-ai/deepseek-harness` |
| 定位 | 开源 **agent harness**，架构理念 **"Everything is a Plugin"** |
| 底座 | [Cordis](https://github.com/cordiverse/cordis)，设计见论文 arXiv:2608.25512 |
| 语言 | TypeScript |
| 许可 | MIT |
| Stars | 约 **277,000** |
| 创建 | 2026-08-13（持续活跃，最后推送 2026-11-04） |
| 官网 | `deepseek.com/harness` |
| 文档站 | `deepseek-harness.github.io/deepseek-harness/` |
| 状态 | **developer preview**，官方明示 **会有破坏兼容的变更** |
| 应用载体 | `cli` · `web` · `desktop` · `desktop-host` · `plugin-market-generator` |

### 二、安装形式

**方式一：npx（推荐，只需 Node.js）**

```bash
npx @deepseek-ai/dsh web
```

- 默认在 `http://127.0.0.1:3080` 起 Web UI 并自动打开浏览器
- `--no-open`：只起服务不开浏览器（SSH 场景用）
- `--port 8080`：换端口（属于 web app 的参数，不是 launcher 的）

**方式二：源码**

```bash
git clone https://github.com/deepseek-ai/deepseek-harness.git
cd deepseek-harness
pnpm install
pnpm run build      # 产出构建物
pnpm dsh web        # 用构建物启动，不重新编译
```

> 开发模式：`pnpm run dev:web`（构建 + 起服务 + 监听源码改动自动重建）。

**方式三：Python SDK / 桌面端**
Python runtime wheel 打包同一个 `dsh` 命令；桌面端是 Electron 载体，有自己的 `desktop` profile。

### 三、使用形式：8 种 entry mode

`dsh` 是**唯一受支持的 Node 应用启动器**。SDK 与 ACP 都是 profile，不是独立的 bin。

| 命令 | 用途 |
|:--|:--|
| `dsh web` | 启动 Web profile（图形界面） |
| `dsh --profile headless "job"` | 跑一个全新持久会话，打印最终答案后退出 |
| `dsh --profile acp` | 通过 ACP stdio 服务自动化客户端，直到断开 |
| `dsh --profile sdk` | 通过 JSON-RPC stdio 服务 SDK 客户端 |
| `dsh --profile sdk-minimal` | 同上，但用独立的极简 agent 树 |
| `dsh <name>` / `dsh --profile <name>` | 启动 `$DSH_HOME/profiles/<name>` 下的具名 profile |
| `dsh --profile <name> --from-default-profile <template>` | 用出厂模板重建一个自定义 profile 再启动 |
| `dsh plugin --profile <name> <pnpm args>` | 在 profile 目录里转发 pnpm 管理插件 |

常用示例：

```bash
dsh --profile web --port 8080        # --port 属于 web app
dsh --profile headless "run the tests"
dsh --help                           # launcher 自己的帮助
dsh --dump-config                    # 不启动，先看合成后的配置树
```

> **参数归属规则**：launcher 只解析自己的 flag，第一个不认识的 token 之后全部交给 profile 内的 app 插件。

### 四、核心概念：Profile

Profile = **插件 bundle 的有序叠加层**，目录结构：

```
$DSH_HOME/profiles/<name>/
├── package.json        # dsh.profile.bundles 有序列表 + 外部插件依赖
└── cordis.patch.yml    # 用户自己的补丁层
```

**合成顺序**（在空根树之上叠加）：

```
① 每个 bundle 的 patch（按 dsh.profile.bundles 顺序）
② profile 自己的 cordis.patch.yml
③ $DSH_HOME/cordis.patch.yml（home 级）
④ --patch 命令行覆盖
```

**内置 bundle**：`dsh-base` · `dsh-web-app` · `dsh-headless` · `dsh-sdk-app` · `dsh-sdk-minimal` · `dsh-acp-app`

**自动初始化**：`web` / `headless` / `sdk` / `sdk-minimal` / `acp` 首次使用时从出厂模板自动创建。`desktop` 名字被 Electron 占用，CLI 拒绝为其启动。

**HMR**：profile 里启用 `dsh-hmr` 后，会监听 manifest 与两处 patch 文件，串行重载；未启用则重启生效。

### 五、Web UI 使用流程

```
① 启动  dsh web
② Settings → Models  填入 DeepSeek API Key（platform.deepseek.com）
   → 保存即生效，无需重启
③ Choose workspace   选一个项目目录
   —— 未选 workspace 时，输入框不可用
④ 开会话，发任务：
   "Summarize this repository and identify its main packages."
```

Agent 能力：读写 workspace 文件、执行命令、委派子任务、维护计划。
**受权限策略约束的操作会先问你。**

### 六、安全提醒（官方 SAFETY.md）

| 要点 | 官方原话要点 |
|:--|:--|
| 实验状态 | 未经安全审计，**不得视为安全或生产可用** |
| 能力范围 | 会执行模型生成的代码和命令，加载第三方插件，访问网络/进程/凭据/文件 |
| 风险 | 错误的模型输出、缺陷、误配置、恶意输入、不可信插件都可能损坏主机、删改文件、泄露数据 |
| 沙箱局限 | 沙箱/审批/权限控制可降低风险，但**不保证隔离**，不能作为唯一安全措施 |
| 建议 | 最小权限运行；优先用一次性 VM / 容器 / 专用环境；备份文件；审阅插件与配置 |

---

## 相关文档

- [[knowledge/infra/ai/01-模型推理加速与部署\|模型推理加速与部署]]
- [[knowledge/infra/ai/03-Ollama本地部署方案\|Ollama 本地部署方案]]（6GB 显存落地）
- [[knowledge/infra/ai/04-vLLM部署练习手册\|vLLM 部署练习手册]]

