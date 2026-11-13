---
{"dg-publish":true,"permalink":"/knowledge/infra/ai/03-Ollama本地部署方案/","title":"Ollama 本地大模型部署方案（6GB 显存）","tags":["LLM","本地部署","Ollama","量化","推理","6GB显存","GTX1660"],"dg-note-properties":{"category":"AI / 本地部署","title":"Ollama 本地大模型部署方案（6GB 显存）","date":"2026-11-09","tags":["LLM","本地部署","Ollama","量化","推理","6GB显存","GTX1660"]}}
---


# Ollama 本地大模型部署方案（6GB 显存）

> 目标机型：MSI 主机 · **GTX 1660 SUPER（6GB，Turing sm_75）** · i7 级 CPU · 31GB 内存
> 一句话结论：**6GB 卡的最优解是 3B～4B 的 Q4_K_M 模型 + 8K 上下文**；7B 只能上 Q4 且要砍上下文；13B 必须 CPU 卸载。
> 选型主线：**权重决定能不能装下，KV Cache 决定能聊多长，带宽决定有多快。**
> 工具主线：**用 Ollama 装、用 Ollama 拉模型、用 Ollama 起服务、用 Ollama 管显存** —— 本篇全部围绕它展开。
>
> ✅ **本方案已于 2026-11-09 在本机落地运行**：Ollama v0.40.2 + `qwen2.5:3b-instruct-q4_K_M`，**实测 84～92 tok/s、显存 2.3GB/6GB、100% GPU**。实测数据见 §4.3，安装踩坑见 §2.1。

---

## 〇、先算清楚：6GB 到底能装什么

### 0.1 权重占用表

| 精度 | 每参数字节 | 6GB 理论可放 | 实际建议 |
|---|---|---|---|
| FP16 | 2.0 | ≈3B | 1.5B～3B |
| INT8 / FP8 | 1.0 | ≈6B | 3B～4B |
| **Q4_K_M（INT4）** | **~0.55** | **≈10B** | **3B～7B（主力）** |
| Q3_K_M | ~0.45 | ≈13B | 7B～13B（紧张） |
| Q5_K_M | ~0.68 | ≈8B | 3B～4B（质量更好） |

> Q4_K_M 是 llama.cpp 的「甜点」量化：K-quant 混合精度，关键层用 Q6_K/Q5_K，其余 Q4_K，质量损失远小于纯 Q4_0。

### 0.2 但权重不是全部

```
可用显存 (6000MB 级)
├── 模型权重         Q4 3B ≈ 2.0GB / Q4 7B ≈ 4.4GB
├── KV Cache         随上下文线性增长 ← 真正的杀手
├── 计算缓冲区       0.3～0.8GB（llama.cpp 的 compute buffer，随 batch 增长）
├── CUDA context     0.2～0.3GB
└── 桌面窗口占用     实测 210MB（Xorg/浏览器会再吃掉一些）
```

**KV Cache 估算公式**

```
KV = 2 × 层数 × hidden × 上下文长度 × bytes_per_element / 并行数
```

| 模型 | 层数 | hidden | 4K 上下文 FP16 KV | 8K 上下文 FP16 KV |
|---|---|---|---|---|
| Qwen2.5-1.5B (28L GQA) | 28 | 1536 | ~0.1GB | ~0.2GB |
| Qwen2.5-3B (36L GQA) | 36 | 2048 | ~0.2GB | ~0.4GB |
| Qwen2.5-7B (28L GQA) | 28 | 3584 | ~0.4GB | ~0.8GB |
| Llama-3.1-8B (32L GQA) | 32 | 4096 | ~0.5GB | ~1.0GB |

> GQA（分组查询注意力）把 KV head 数降到 1/4～1/8，这是 7B 能在 6GB 上勉强跑长上下文的唯一原因。老架构（MHA）模型同尺寸 KV 要 ×4～8。

### 0.3 显存预算实例：Qwen2.5-7B-Q4_K_M

| 项 | 占用 |
|---|---|
| 权重 | 4.4GB |
| KV @ 4096 ctx | 0.4GB |
| compute buffer + CUDA ctx | ~0.8GB |
| **合计** | **~5.6GB / 6.0GB** |

→ 只剩 400MB 余量，**4K 上下文是 7B-Q4 在 6GB 上的红线**；再长会 OOM 或掉到共享内存（速度暴跌 5-10 倍）。

---

## 一、模型选型

### 1.1 推荐矩阵（6GB 实操）

| 场景 | 推荐模型 | 量化 | 权重 | 建议 ctx | 预期速度 |
|---|---|---|---|---|---|
| **中文对话/写作（主力）** | **Qwen2.5-3B-Instruct** | Q4_K_M | 2.0GB | 8K | 35～45 tok/s |
| 中文更强、能忍慢一点 | Qwen2.5-7B-Instruct | Q4_K_M | 4.4GB | 4K | 18～25 tok/s |
| 代码补全 | Qwen2.5-Coder-3B / 7B-Q4 | Q4_K_M | 2.0 / 4.4GB | 4K | 35 / 18 tok/s |
| 英文通用 | Llama-3.2-3B-Instruct | Q4_K_M | 2.0GB | 8K | 35～45 tok/s |
| 多语言/图像理解 | Gemma-3-4B-it | Q4_K_M | 2.5GB | 8K | 30～40 tok/s |
| 极速/低占用 | Qwen2.5-1.5B-Instruct | Q4_K_M | 1.0GB | 16K | 60～80 tok/s |
| **质量优先（能忍慢）** | Qwen2.5-7B-Instruct | Q5_K_M + CPU 卸载 | 5.0GB+ | 4K | 8～15 tok/s |

> 速度基于 GTX 1660 SUPER 实测参数量级（336 GB/s 显存带宽、无 Tensor Core FP16 加速优势，走 FP16 CUDA Core 路径）。

### 1.2 部署工具选型

| 方案 | 6GB 适用性 | 原因 |
|---|---|---|
| **Ollama** | ✅ 首选 | 按显存自动切层，`num_ctx` 可控，零配置 |
| **llama.cpp (llama-server)** | ⭕ 备选 | 手调粒度最细，`-ngl` 逐层卸载；无现成安装包，需从源码编译 |
| vLLM | ❌ 不建议 | 显存池化预分配 + Turing 无 Tensor Core，详见 §1.3 |
| TensorRT-LLM | ❌ 不建议 | 需要编译 engine，Turing 支持弱，收益不如投入 |
| LM Studio | ⚠️ 可用 | 图形化，适合试用；底层同为 llama.cpp |

**结论：本方案以 Ollama 为唯一落地工具（已实际运行）；llama.cpp 仅在需要逐层调 `-ngl` 时才考虑，需自行编译；vLLM 不采用（见 §1.3）。**

### 1.3 为什么不推荐 vLLM（2026-11-09 实测评估，决定弃用）

> 评估结论：**能跑，但不值得跑。** 本机不采用 vLLM，理由按影响从大到小排列。

**① 显存预分配模型与 6GB 冲突（最致命）**

vLLM 启动即按 `gpu_memory_utilization`（默认 **0.9**）一次性切走 KV Cache 池：

```
KV 池 = 总显存 × 0.9 − 权重 − 激活 − CUDA context
6GB × 0.9 = 5.4GB 被预占
```

| 模型（FP16 权重） | 权重 | 剩余给 KV | 结果 |
|---|---|---|---|
| Qwen2.5-1.5B | 3.1GB | ~2.3GB | 勉强启动 |
| Qwen2.5-3B | 6.2GB | 负数 | **启动即 OOM** |

→ 6GB 上只能碰 1.5B 级别，实际能力还不如 Ollama 跑 3B～4B。

**② Turing（sm_75）拿不到 vLLM 的核心红利**

vLLM 的价值来自 PagedAttention + FlashAttention + Tensor Core + FP8。本机 GTX 1660 SUPER：

- **无 Tensor Core**：FP16 只能走 CUDA Core，矩阵运算无加速
- **FP8 不支持**：Turing 之后（sm_89+）才有
- **FlashAttention 需回退**：FA2 要求 sm_80+，sm_75 只能走 xformers/原生实现

结果：**工程优势（连续批处理、分页 KV、并发调度）还在，但速度优势被抹平**，单用户场景比 llama.cpp 更慢。

**③ 必须依赖预量化模型（AWQ/GPTQ）**

| 模型 | 权重 | 6GB 可用 ctx |
|---|---|---|
| Qwen2.5-1.5B-AWQ | ~1.1GB | 4K～8K ✅ |
| Qwen2.5-3B-AWQ | ~2.1GB | 4K ✅ |
| Qwen2.5-7B-AWQ | ~4.5GB | **512～1K ❌ 不可用** |

llama.cpp 的 K-quant（Q4_K_M/Q5_K_M）质量和显存效率更好，且**不需要预先找好量化版本**。

**④ 本机安装成本高**

实测前置条件：Python 3.10.12 ✅，但 **无 CUDA Toolkit（无 nvcc）、无 torch**。Turing + CUDA 12 组合需手动编译或找兼容 wheel，依赖数 GB、耗时数小时。

**⑤ 并发能力在单机上没有意义**

vLLM 真正的杀手锏是**多请求高并发下的吞吐**（连续批处理把 GPU 填满）。6GB 单卡、本地单人使用场景下，并发永远是 1，这项优势直接用不上。

**⑥ 对照表**

| 维度 | llama.cpp / Ollama | vLLM @ 6GB sm_75 |
|---|---|---|
| 3B 可用性 | ✅ 开箱（Q4_K_M） | ⚠️ 需 AWQ |
| 7B 可用性 | ✅ 4K 上下文 | ❌ 512 上下文 |
| 单用户速度 | 更快（量化更激进、无池化开销） | 更慢（无 Tensor Core） |
| 多用户并发 | 弱 | 强（但本机用不上） |
| 显存利用率 | 按层 `-ngl` 灵活卸载 | 池化预分配，死板 |
| 安装成本 | 1 条命令 | 数小时 + 数 GB |
| 模型格式要求 | 任意 GGUF | 必须 AWQ/GPTQ/FP8 |

**⑦ 什么情况下仍值得用 vLLM（重新评估的触发条件）**

1. 显存 **≥24GB**（A100 / 4090 / L40S）—— PagedAttention 与连续批处理才真正收益
2. 需要**多用户并发 / 对外 API 服务**—— 吞吐优先于单请求延迟
3. 目标是**练习生产级推理服务流程**（指标、压测、模型热更新），而非本地自用

> 一句话：**vLLM 是为「大显存 + 高并发」设计的，6GB 单机属于它的反面场景。**

---

## 二、安装与首次部署（Ollama）

### 2.1 安装

> **本机实际安装记录（2026-11-09）**：`ollama.com/install.sh` 会 307 跳转到 GitHub Release，而本机到 GitHub 全链路不通（403/504/超时），**官方脚本不可用**。
> 实际可行路径是 **snap**（Snap Store 可访问），但 snapd 自身下载 1.62GB 包极慢（0.3%/分钟）。
> 最终解法：**用 curl 直接拉 snap 包并分段并行下载，再 `snap install --dangerous` 本地安装**。

```bash
# 方案一：官方脚本（GitHub 可访问时）
curl -fsSL https://ollama.com/install.sh | sh

# 方案二：snap（推荐，国内可用）
sudo snap install ollama

# 方案三：本机实际采用（GitHub 不通 + snap 慢时）
# 1) 取真实 CDN 直链
URL=$(curl -sSI "https://api.snapcraft.io/api/v1/snaps/download/dKf8qnwq79UYj4TGLmf7pvptuMnBPqQT_141.snap" \
      | grep -i '^location:' | sed 's/^[Ll]ocation: //' | tr -d '\r')
# 2) 六段并行下载（关键：必须加 --http1.1，HTTP/2 多路复用会被 CDN 限流到 0）
TOTAL=1618587648; CHUNK=$((TOTAL/6))
for i in 0 1 2 3 4 5; do
  S=$((i*CHUNK)); E=$((S+CHUNK-1))
  [ $i -eq 5 ] && E=$((TOTAL-1))
  curl -sS --http1.1 -r $S-$E -o /tmp/p$i "$URL" &
done; wait
cat /tmp/p0 /tmp/p1 /tmp/p2 /tmp/p3 /tmp/p4 /tmp/p5 > /tmp/ollama.snap
# 3) 本地安装
sudo snap install --dangerous /tmp/ollama.snap

# 验证
ollama --version          # 实测 v0.40.2
snap services ollama      # ollama.listener active

# 关键：让模型落在数据盘而不是系统盘（可选）
sudo systemctl edit ollama
# 写入：
# [Service]
# Environment="OLLAMA_MODELS=/data1/ollama/models"
sudo systemctl daemon-reload && sudo systemctl restart ollama
```

> 默认路径 `/usr/share/ollama/.ollama/models`；本机 `/data1` 是 COS 挂载的网络盘，**不建议放这里**（随机读会拖慢加载），建议 `/home/yang/ollama-models`。

### 2.2 拉取模型

```bash
# 主力：3B Q4（约 2.0GB）
ollama pull qwen2.5:3b-instruct-q4_K_M

# 备选：7B Q4（约 4.4GB，上下文要砍）
ollama pull qwen2.5:7b-instruct-q4_K_M

# 代码
ollama pull qwen2.5-coder:3b
```

### 2.3 跑起来

```bash
# 交互式
ollama run qwen2.5:3b-instruct-q4_K_M

# 显式指定上下文（关键！）
ollama run qwen2.5:3b-instruct-q4_K_M --num-ctx 8192
```

**7B 的正确姿势（必须限上下文）**

```bash
OLLAMA_KV_CACHE_TYPE=q8_0 ollama run qwen2.5:7b-instruct-q4_K_M --num-ctx 4096
```

- `OLLAMA_KV_CACHE_TYPE=q8_0`：KV Cache 量化到 8bit，显存减半，质量几乎无损 ← **6GB 跑 7B 的救命开关**
- `--num-ctx 4096`：不要用默认 2048 以上还想要 8K，会直接爆

### 2.4 推荐 Modelfile（一次设定，永久生效）

```dockerfile
# 保存为 Modelfile.qwen3b
FROM qwen2.5:3b-instruct-q4_K_M

PARAMETER num_ctx 8192
PARAMETER num_gpu 99          # 99 = 全部层上 GPU
PARAMETER num_batch 256       # 批大小，显存紧张时降到 128
PARAMETER temperature 0.7
PARAMETER top_p 0.9
PARAMETER repeat_penalty 1.1
```

```bash
ollama create qwen3b-local -f Modelfile.qwen3b
ollama run qwen3b-local
```

### 2.5 OpenAI 兼容 API（接任意客户端）

Ollama 自带 `:11434` 的 OpenAI 兼容端点：

```bash
curl http://127.0.0.1:11434/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "qwen3b-local",
    "messages": [{"role": "user", "content": "用一句话解释 PagedAttention"}],
    "temperature": 0.7
  }'
```

可直接对接：Cherry Studio / Open WebUI / Continue（VSCode）/ Dify / LangChain。

### 2.6 关键环境变量

| 变量 | 推荐值 | 作用 |
|---|---|---|
| `OLLAMA_HOST` | `127.0.0.1:11434` | 监听地址（默认只本机，安全） |
| `OLLAMA_KEEP_ALIVE` | `30m` | 模型驻留时间，避免反复加载 |
| `OLLAMA_NUM_PARALLEL` | `1` | **6GB 必须 1**，并行数>1 会让 KV 翻倍 |
| `OLLAMA_MAX_LOADED_MODELS` | `1` | 同上，防止两个模型抢显存 |
| `OLLAMA_KV_CACHE_TYPE` | `f16`（3B）/ `q8_0`（7B） | KV 精度，省显存利器 |
| `CUDA_VISIBLE_DEVICES` | `0` | 只有一张卡，无所谓 |

---

## 三、日常运维：常用命令（Ollama 落地后）

> 本机已按 Ollama 落地运行，以下命令可直接复制使用。

### 3.1 启动 / 停止 / 状态

| 操作 | 命令 |
|---|---|
| 查看服务状态 | `snap services ollama` |
| 启动服务 | `sudo snap start ollama` |
| 停止服务 | `sudo snap stop ollama` |
| 重启服务 | `sudo snap restart ollama` |
| 查看版本 | `ollama --version` |
| 健康检查 | `curl -s http://127.0.0.1:11434/api/version` |

```bash
$ snap services ollama
Service          Startup  Current  Notes
ollama.listener  enabled  active   -

$ ollama --version
ollama version is 0.40.2
```

### 3.2 四种启动模型的方式

**① 交互式聊天**（退出即卸载）

```bash
ollama run qwen2.5:3b-instruct-q4_K_M
# 出现 >>> 提示符后直接提问，/bye 退出
```

**② 单次问答**

```bash
ollama run qwen2.5:3b-instruct-q4_K_M "用一句话解释什么是量化"
```

**③ 常驻显存 + API 服务**（客户端调用推荐）

```bash
# 预加载进显存，保持 1 小时
curl -s http://127.0.0.1:11434/api/generate -d '{
  "model":"qwen2.5:3b-instruct-q4_K_M",
  "prompt":"","keep_alive":"1h"
}'

# 确认已在 GPU 上（PROCESSOR 必须显示 100% GPU）
ollama ps
# NAME                        SIZE    PROCESSOR   CONTEXT  RUNNER
# qwen2.5:3b-instruct-q4_K_M  2.2 GB  100% GPU    4096     llamacpp
```

**④ 卸载出显存（不删模型）**

```bash
ollama stop qwen2.5:3b-instruct-q4_K_M
```

### 3.3 模型管理

```bash
ollama list                              # 列出已装模型
ollama pull qwen2.5:7b-instruct-q4_K_M   # 下载新模型
ollama rm <model>                        # 删除模型（释放磁盘）
ollama show qwen2.5:3b-instruct-q4_K_M   # 查看模型参数与模板
```

### 3.4 显存占用对照（本机实测）

| 状态 | 显存占用 | 说明 |
|---|---|---|
| 服务空闲（仅桌面） | **210 MiB** | 模型已卸载，Ollama 服务常驻但不占显存 |
| 3B-Q4 加载，ctx 4096 | **~2350 MiB** | 日常状态 |
| 3B-Q4 加载，ctx 8192 | ~2600 MiB（推算） | 长上下文 |
| 3B-Q4 + 长对话 | 最高 ~3.5 GB | KV Cache 随对话增长 |

> **关键认知**：Ollama **不是**常驻显存服务。没有请求且超过 `keep_alive`（默认 5 分钟）后，模型会自动卸载，显存回落到 210 MiB。这是特性不是 bug —— 省下显存给别的程序用。

### 3.5 让模型常驻

| 方式 | 做法 |
|---|---|
| 单次请求指定 | 请求体加 `"keep_alive":"2h"`（`-1` = 永不卸载） |
| 全局默认 | 服务环境变量 `OLLAMA_KEEP_ALIVE=30m` |
| Modelfile 内固定 | `PARAMETER keep_alive 1h` |

---

## 四、验证与基准测试

### 4.1 冒烟测试

```bash
# Ollama 侧延迟/吞吐
curl -s http://127.0.0.1:11434/api/generate -d '{
  "model":"qwen3b-local","prompt":"写一句关于秋天的诗","stream":false
}' | jq '.eval_count, .eval_duration'

# tok/s = eval_count / (eval_duration / 1e9)
```

### 4.2 显存观测

```bash
# 实时看
nvidia-smi --query-gpu=memory.used,utilization.gpu,temperature.gpu --format=csv -l 1

# 只看本进程
nvidia-smi --query-compute-apps=pid,process_name,used_memory --format=csv
```

**判定标准（6GB 卡）**

| 现象 | 含义 | 处理 |
|---|---|---|
| 显存稳定 5.0～5.8GB | 健康 | — |
| 显存 6.0GB + 利用率跌到 20% | 已溢出到共享内存（走 PCIe） | 减 `-ngl` / 减 `num_ctx` |
| `CUDA error: out of memory` | 超了 | 同上，或换 q8_0 KV |

### 4.3 实测性能基线（GTX 1660 SUPER，2026-11-09 本机真实数据）

> ⚠️ 以下为**本机实测**，非推算。环境：Ollama v0.40.2（snap 安装）+ `qwen2.5:3b-instruct-q4_K_M` + `num_ctx 4096`，模型 100% 驻留 GPU。

| 场景 | 生成速度 | TTFT | 显存占用 | 输出 token |
|---|---|---|---|---|
| 短问候（"你好"） | **91.9 tok/s** | 27 ms | 2351 MiB | 9 |
| 长文本（200 字说明文） | **84.4 tok/s** | 148 ms | 2351 MiB | 258 |

**实测结论**

- 实测 **84～92 tok/s**，明显好于原本推算的 35～45 tok/s —— Turing 的 FP16 CUDA Core 路径对小模型效率比预期高
- TTFT 27～148ms，交互流畅
- 显存只占 **2.3GB / 6GB**，余 3.7GB，可放心把 `num_ctx` 提到 8192 甚至 16384
- `ollama ps` 显示 `100% GPU`、runner `llamacpp`，无 CPU 卸载

**其他尺寸推算（未实测，按参数量与带宽外推）**

| 模型 | 量化 | ctx | 生成速度（推算） |
|---|---|---|---|
| Qwen2.5-1.5B | Q4_K_M | 8K | 120～150 tok/s |
| Qwen2.5-7B | Q4_K_M | 4K | 30～40 tok/s |
| Qwen2.5-14B | Q4_K_M | 2K（部分卸载） | 5～10 tok/s |

---

## 五、排障

| 现象 | 原因 | 处理 |
|---|---|---|
| `ollama run` 很慢且显存没占满 | 层没全上 GPU | 设 `num_gpu 99`；检查 `nvidia-smi` 是否看到 ollama 进程 |
| 跑着跑着变卡 | 上下文涨、KV 溢出到内存 | 限 `num_ctx`；`OLLAMA_KV_CACHE_TYPE=q8_0` |
| `CUDA out of memory` | 权重+KV+缓冲超限 | 先降 `num_ctx` 一半；再降 `num_batch`；再换更小模型 |
| 首 token 很慢（>2s） | prompt 太长 + compute buffer 大 | 降 `num_batch` 到 128；避免超长 system prompt |
| 中文输出乱码/夹英文 | 模板或参数量太小 | 换 Qwen 系；temperature 降到 0.6～0.7 |
| 桌面卡顿 | 显存吃光，合成器掉帧 | 开 `nvidia-smi` 看；给桌面留 500MB |
| 模型加载要 30s+ | 模型在 COS 网络盘上 | 移到本地 NVMe（`/home/yang`） |
| 重启后模型没了 | 没设 `OLLAMA_KEEP_ALIVE` 或服务未自启 | `systemctl enable ollama` |

---

## 六、进阶：让 6GB 更值

| 手段 | 收益 | 代价 |
|---|---|---|
| **KV Cache 量化（q8_0）** | 显存省 ~50% | 极轻微质量损失 |
| **GQA 模型优先** | 同尺寸 KV 减 4～8 倍 | 选型时就得挑对 |
| **Speculative decoding** | 提速 1.3～2x | 要一对投机模型，配置复杂 |
| **前缀缓存（prompt cache）** | 重复 system prompt 秒回 | Ollama 自动做，无需配置 |
| **定期重启服务** | 防止碎片累积 | `sudo snap restart ollama`，牺牲冷启动时间 |
| **换成 4bit 更激进的 Q3_K_M** | 能跑 13B | 质量明显下降 |

> 若未来升卡：**12GB（RTX 3060）** 是本地部署的甜点档，可稳跑 7B-Q5 + 16K 上下文；**16GB（4060Ti）** 可上 14B-Q4。

---

## 七、一句话总结

- **6GB 最优解**：Qwen2.5-3B-Q4_K_M + 8K 上下文，35～45 tok/s，日常可用
- **想跑 7B**：Q4_K_M + `num_ctx 4096` + KV 量化 q8_0，是极限
- **想跑 13B**：只能 `-ngl` 部分卸载，5～10 tok/s，能跑不流畅
- **工具选择**：**只用 Ollama**（已落地，实测可用）；llama.cpp 需自行编译，仅在极端调参场景考虑；**别用 vLLM**
- **记忆口诀**：**权重定装不装得下，KV 定能聊多长，带宽定有多快**

---

## 附：本文实测环境

| 项 | 值 |
|---|---|
| GPU | NVIDIA GeForce GTX 1660 SUPER，6144 MiB，sm_75 |
| 驱动 | 595.99.02 |
| OS | Ubuntu 22.04.5 LTS |
| 内存 | 31GB |
| 系统盘 | NVMe 916GB（可用 844GB） |
| **Ollama** | **v0.40.2（snap 安装，`/snap/bin/ollama`）** |
| **已装模型** | **`qwen2.5:3b-instruct-q4_K_M`（1.9GB）** |
| **模型存储** | `/var/snap/ollama/common/models` |
| **服务状态** | `ollama.listener` active，API `:11434` 正常 |
| **实测生成速度** | **84～92 tok/s（100% GPU）** |
| **实测显存占用** | **2351 MiB / 6144 MiB** |

### 验证命令（可直接复现）

```bash
# 1. 服务与版本
ollama --version                          # → 0.40.2
curl -s http://127.0.0.1:11434/api/version  # → {"version":"0.40.2"}

# 2. 模型列表
ollama list
# NAME                          ID              SIZE      MODIFIED
# qwen2.5:3b-instruct-q4_K_M    357c53fb659c    1.9 GB    ...

# 3. 确认跑在 GPU 上（关键：PROCESSOR 必须是 100% GPU）
curl -s http://127.0.0.1:11434/api/generate -d '{
  "model":"qwen2.5:3b-instruct-q4_K_M","prompt":"你好",
  "stream":false,"keep_alive":"30m","options":{"num_ctx":4096,"num_predict":64}}' > /tmp/b.json
ollama ps
# NAME  ... PROCESSOR  CONTEXT  RUNNER
# ...        100% GPU   4096    llamacpp

# 4. 从返回 JSON 算 tok/s
python3 -c "
import json; d=json.load(open('/tmp/b.json'))
ec,ed=d['eval_count'],d['eval_duration']
print('%.1f tok/s' % (ec/(ed/1e9)))"
```

**实测踩坑记录**

| 坑 | 现象 | 解法 |
|---|---|---|
| GitHub 全链路不通 | install.sh 307→GitHub 504 | 改走 snap |
| snapd 下载极慢 | 0.3%/分钟 | curl 直连 CDN + 分片 |
| **HTTP/2 多路复用被限流** | 6 连接全部 0 字节 | **加 `--http1.1`，速度 3.2MB/s** |
| 单连接中途断开 | 分片停在 30%～88% | 断点续传循环重连 |
| 后台 bash 被回收 | pull 进程消失 | 用 `timeout N cmd` 前台跑 |
| `pkill -f 关键词` | 把自己也杀了 | 用 PID 精确 kill |

> 相关：[[knowledge/infra/ai/01-模型推理加速与部署\|01-模型推理加速与部署]]（四层栈全景）· [[knowledge/infra/ai/02-DeepSeek技术全景\|02-DeepSeek技术全景]]（DeepInfer 与 vLLM 对比）
