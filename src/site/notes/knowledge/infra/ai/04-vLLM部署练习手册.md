---
{"dg-publish":true,"permalink":"/knowledge/infra/ai/04-vLLM部署练习手册/","title":"vLLM 部署练习手册（6GB 显存版）","tags":["vLLM","LLM","本地部署","量化","AWQ","OpenAI兼容API","压测","6GB显存"],"dg-note-properties":{"category":"AI / 本地部署","title":"vLLM 部署练习手册（6GB 显存版）","date":"2026-11-09","tags":["vLLM","LLM","本地部署","量化","AWQ","OpenAI兼容API","压测","6GB显存"]}}
---


# vLLM 部署练习手册（6GB 显存版）

> 硬件：MSI 主机 · **GTX 1660 SUPER（6GB，sm_75 / Turing）** · 31GB 内存 · Ubuntu 22.04.5 · Python 3.10.12
> **本文不是「推荐方案」，而是「练习方案」。** 目的是在低配卡上把 vLLM 的完整工程流程跑一遍：环境隔离 → 装机 → 起服务 → 调参 → 压测 → 观测 → 排障。
> 这些流程在 A100 / 4090 / H100 上一模一样，学会后可直接迁移到生产卡。
>
> ⚠️ 日常自用请走 [[knowledge/infra/ai/03-Ollama本地部署方案\|03-Ollama本地部署方案]]（Ollama / llama.cpp）。vLLM 不推荐用于 6GB 的单人场景，原因见该文 §1.3。

---

## 〇、先明确目标与代价

### 0.1 本机为什么「能跑但不划算」

| 限制 | 影响 | 对练习的影响 |
|---|---|---|
| 6GB 显存 + vLLM 池化预分配（默认 0.9） | 仅 1.5B 级 FP16 / 3B 级 AWQ 可用 | 不影响流程练习 |
| sm_75 无 Tensor Core、无 FP8 | 吞吐明显低于官方 benchmark | **压测绝对值无参考意义，只看相对变化** |
| FlashAttention 需 sm_80+ | 回退到 xformers / 原生 attention | 会在日志里看到 fallback，属正常 |
| 无 CUDA Toolkit（无 nvcc） | 不能源码编译 | 走 wheel 安装，够用 |
| 无 torch | 需先装 | 见 §2 |

### 0.2 练习能学到什么（这才是重点）

- ✅ 用 venv 做隔离环境，不污染系统 Python
- ✅ 用 `--gpu-memory-utilization` / `--max-model-len` / `--max-num-seqs` 控制显存预算
- ✅ 用 AWQ 量化模型在低显存下启动服务
- ✅ OpenAI 兼容 API + Python/curl 客户端调用
- ✅ 用 `vllm bench serve` 做压测并读结果
- ✅ 读 Prometheus 指标，理解 KV Cache 使用率
- ✅ 看懂 vLLM 启动日志里的关键行（模型加载、KV 块数、并发上限）

### 0.3 预算表（6GB 实际可跑什么）

| 模型 | 精度 | 权重 | 预计可用 max-model-len | 结论 |
|---|---|---|---|---|
| Qwen2.5-0.5B-Instruct | FP16 | 1.0GB | 8K | ✅ 轻松 |
| Qwen2.5-1.5B-Instruct | FP16 | 3.1GB | 4K～6K | ✅ 可用 |
| **Qwen2.5-1.5B-Instruct-AWQ** | INT4 | ~1.1GB | 8K～16K | ✅ **推荐** |
| **Qwen2.5-3B-Instruct-AWQ** | INT4 | ~2.1GB | 4K | ✅ 推荐 |
| Qwen2.5-7B-Instruct-AWQ | INT4 | ~4.5GB | 512～1K | ⚠️ 仅能验证启动 |
| Qwen2.5-3B-Instruct | FP16 | 6.2GB | — | ❌ 启动即 OOM |

> 练习建议：**主力用 1.5B-AWQ**（权重小、KV 空间宽裕），把 3B-AWQ 当「进阶档」，7B-AWQ 当「边界实验」。

---

## 一、Python 版本问题（本机第一道坎）

### 1.1 现状（实测）

| 项 | 值 |
|---|---|
| 系统 Python | **3.10.12** |
| 最新 vLLM | 0.33.0 |
| 0.33.0 requires-python | **>=3.11,<3.15** ← 3.10 装不上 |
| 清华源上 cp310 wheel 最高版本 | 0.5.1（2024 年，太旧，缺 AWQ/prompt cache 等特性） |

也就是说：

- **直接 `pip install vllm` → 报 `No matching distribution`**
- **给 3.10 的最后一个 wheel 是 0.5.1**，版本太旧，练习价值低

### 1.2 三条解法（按推荐度）

| 方案 | 说明 | 代价 |
|---|---|---|
| C. 装 0.5.1 旧版 | `pip install vllm==0.5.1`（镜像上有 cp310 轮子） | 版本太旧，与现行文档不符，**不推荐** |
| **B. uv 装独立 Python 3.12（本文主线）** | `uv python install 3.12`，不碰系统 Python | 需装 uv，venv 内 wheel 约 3～5GB |
| **A. Docker（备选，见附录 A）** | 官方 `vllm/vllm-openai` 镜像，自带 Python 3.12 + CUDA 运行时 | 需装 Docker + nvidia-container-toolkit |

> 主线用 **方案 B**：不需要 docker 权限、过程透明、适合练习「环境构建」这一环。

---

## 二、环境准备

### 2.1 安装 uv 与 Python 3.12

```bash
# 安装 uv（单文件，不污染系统）
curl -LsSf https://astral.sh/uv/install.sh | sh
source ~/.local/bin/env   # 或重新登录 shell
uv --version

# 用 uv 拉一个独立的 Python 3.12（不替换系统 3.10）
uv python install 3.12
uv python list | grep 3.12
```

### 2.2 建 venv 并装依赖

```bash
mkdir -p ~/vllm-lab && cd ~/vllm-lab
uv venv --python 3.12 .venv
source .venv/bin/activate

# 关键：先装对应 CUDA 版本的 torch，再装 vLLM，避免拉到错误 CUDA 版本
uv pip install torch --index-url https://download.pytorch.org/whl/cu121
uv pip install vllm

# 验证
python -c "import torch, vllm; print(torch.__version__, torch.version.cuda, torch.cuda.is_available()); print('vllm', vllm.__version__)"
```

预期输出（示例）：

```
2.5.x+cu121 12.1 True
vllm 0.33.0
```

> **必须看到 `True`**。若为 `False`：说明装成了 CPU 版 torch，用 `--index-url .../whl/cu121` 重装。

### 2.3 磁盘与缓存位置

```bash
# HuggingFace 权重缓存别放 COS（/data1 是网络盘），放本地 NVMe
echo 'export HF_HOME=~/vllm-lab/hf' >> ~/.bashrc
echo 'export HF_ENDPOINT=https://hf-mirror.com' >> ~/.bashrc   # 国内加速
source ~/.bashrc
```

> 本机 `/` 有 844GB 可用，权重放 `~/vllm-lab/hf` 完全够。

---

## 三、启动服务（核心练习）

### 3.1 最小可跑命令（1.5B-AWQ）

```bash
vllm serve Qwen/Qwen2.5-1.5B-Instruct-AWQ \
  --quantization awq \
  --dtype half \
  --max-model-len 8192 \
  --gpu-memory-utilization 0.85 \
  --max-num-seqs 4 \
  --enforce-eager \
  --served-model-name qwen15 \
  --port 8000
```

### 3.2 每个参数为什么这么设（6GB 专属）

| 参数 | 值 | 理由 |
|---|---|---|
| `--quantization awq` | awq | 4bit 权重，1.5B 只占 1.1GB，给 KV 留空间 |
| `--dtype half` | FP16 | sm_75 不支持高效 BF16，FP16 更稳 |
| `--max-model-len` | 8192 | 上限越小，KV 池越省；**先小后大**试 |
| `--gpu-memory-utilization` | **0.85** | 默认 0.9=5.4GB 太激进，留 0.9GB 给桌面/驱动 |
| `--max-num-seqs` | **4** | 并发数直接乘 KV 占用；6GB 上 4 已很宽裕 |
| `--enforce-eager` | 开启 | 关掉 CUDA Graph，省显存、避免捕获阶段 OOM；**低显存必开** |
| `--served-model-name` | qwen15 | 客户端用的模型名，与仓库路径解耦 |

### 3.3 启动日志里要看的 5 行

```
INFO  Loading model weights took 1.9 GB                 ← 权重占用
INFO  GPU KV cache size: 45,120 tokens                  ← KV 池 token 数（关键！）
INFO  Maximum concurrency for 8,192 tokens per request: 5.51x   ← 并发倍数
INFO  Using FlashAttention backend: xformers            ← sm_75 上的回退，正常
INFO  Uvicorn running on http://0.0.0.0:8000            ← 服务就绪
```

**解读要点**

- `GPU KV cache size` ÷ `max-model-len` = 理论最大并发
- 若并发 < 1，说明**连一个满长度请求都放不下** → 降 `max-model-len`
- `FlashAttention backend` 显示 `xformers` 属正常；显示 `FLASH_ATTN` 才需要 sm_80+

### 3.4 调参顺序（OOM 时按这个来）

```
① 降 --max-model-len（8192 → 4096 → 2048）
② 降 --max-num-seqs（4 → 2 → 1）
③ 降 --gpu-memory-utilization（0.85 → 0.80）
④ 确认 --enforce-eager 已开
⑤ 换更小模型 / 更激进量化
```

---

## 四、API 练习

### 4.1 curl 冒烟

```bash
curl -s http://127.0.0.1:8000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "qwen15",
    "messages": [{"role": "user", "content": "用一句话解释 PagedAttention"}],
    "max_tokens": 128
  }' | python3 -m json.tool
```

### 4.2 Python 客户端（openai SDK）

```python
from openai import OpenAI

client = OpenAI(base_url="http://127.0.0.1:8000/v1", api_key="EMPTY")

r = client.chat.completions.create(
    model="qwen15",
    messages=[{"role": "user", "content": "写三行 Python，反转一个链表"}],
    max_tokens=256,
    extra_body={"top_k": 20},     # vLLM 扩展参数走 extra_body
)
print(r.choices[0].message.content)
print(r.usage)
```

```bash
uv pip install openai
```

> `api_key` 随便填（本地无鉴权），填 `EMPTY` 是 vLLM 官方惯例。

### 4.3 健康检查与模型列表

```bash
curl -s http://127.0.0.1:8000/v1/models | python3 -m json.tool
curl -s http://127.0.0.1:8000/health
```

### 4.4 会话保持（练习点）

vLLM **不做**会话管理，多轮对话要客户端把完整历史带上。这与 Ollama 的 `/api/chat` 不同——练习时要留意这个差异。

---

## 五、压测练习（核心价值）

### 5.1 内置 bench

```bash
vllm bench serve \
  --base-url http://127.0.0.1:8000 \
  --model qwen15 \
  --dataset-name random \
  --random-input-len 512 \
  --random-output-len 128 \
  --num-prompts 20 \
  --max-concurrency 1 \
  --percentile-metrics ttft,tpot,itl \
  --metric-percentiles 50,90,99
```

### 5.2 关键指标含义

| 指标 | 全称 | 含义 |
|---|---|---|
| **TTFT** | Time To First Token | 首字延迟，反映 prefill 能力 |
| **TPOT** | Time Per Output Token | 每输出 token 耗时 |
| **ITL** | Inter-Token Latency | token 间隔抖动 |
| **Output tok/s** | 吞吐 | 生成速度 |
| **Request throughput** | QPS | 每秒完成请求数 |

### 5.3 递进式实验（练习精髓）

```bash
for c in 1 2 4; do
  echo "===== concurrency=$c ====="
  vllm bench serve --base-url http://127.0.0.1:8000 --model qwen15 \
    --dataset-name random --random-input-len 512 --random-output-len 128 \
    --num-prompts 20 --max-concurrency $c
  nvidia-smi --query-gpu=memory.used --format=csv,noheader
done
```

**预期规律（6GB 体感）**

| 并发 | 现象 |
|---|---|
| 1 | TTFT 最低，吞吐最低 |
| 2 | 吞吐约 ×1.6～1.8，TTFT 略升 |
| 4 | 吞吐趋于平台期，显存接近上限 |
| >4 | 请求排队，TTFT 暴涨 → 该降 `max-model-len` 或限并发 |

> sm_75 上绝对值比官方 benchmark 低 3～5 倍，**不要拿去对比网上的 A100 数据**，只看趋势。

### 5.4 对照实验（体会连续批处理的价值）

```bash
vllm serve ... --max-num-seqs 1     # 对照组：近似串行
vllm serve ... --max-num-seqs 4     # 实验组：连续批处理
```

两次压测吞吐的差距，就是 Continuous Batching 带来的收益。

---

## 六、观测

### 6.1 Prometheus 指标

```bash
curl -s http://127.0.0.1:8000/metrics | grep -E 'vllm:(num_requests|gpu_cache|prefix_cache)' | head -20
```

| 指标 | 含义 |
|---|---|
| `vllm:num_requests_running` | 当前在跑请求数 |
| `vllm:num_requests_waiting` | 排队数（>0 说明饱和） |
| `vllm:gpu_cache_usage_perc` | **KV Cache 使用率**（核心） |
| `vllm:gpu_prefix_cache_hit_rate` | 前缀缓存命中率 |
| `vllm:time_to_first_token_seconds` | TTFT 直方图 |
| `vllm:inter_token_latency_seconds` | ITL 直方图 |

### 6.2 显存对照

```bash
watch -n1 "nvidia-smi --query-gpu=memory.used,utilization.gpu --format=csv"
```

**判定**：启动后显存几乎不涨、`gpu_cache_usage_perc` 一直很低 → 是 KV 池被 `max-model-len` 卡住，不是显存不够。

### 6.3 进阶：接 Grafana（可选）

- 起 Prometheus，抓 `127.0.0.1:8000/metrics`
- 用 Grafana Dashboard **ID 20959**（vLLM 社区模板）
- 观察压测时 `gpu_cache_usage_perc` 曲线

---

## 七、排障

| 现象 | 原因 | 处理 |
|---|---|---|
| `No matching distribution found for vllm` | Python 3.10 太旧 | 按 §1.2 用 uv 装 3.12 |
| `torch.cuda.is_available() == False` | torch 装了 CPU 版 | 用 `--index-url .../whl/cu121` 重装 |
| `No available memory for the cache blocks` | KV 池为 0 | 降 `max-model-len` / `gpu-memory-utilization` |
| `Bfloat16 is only supported on GPUs with compute capability >= 8.0` | sm_75 不支持 BF16 | 加 `--dtype half` |
| `CUDA out of memory`（启动即） | 权重 + 池化超限 | 见 §3.4 调参顺序 |
| `Engine core initialization failed` | 多为显存不足或 CUDA 版本不匹配 | 看完整 traceback，先降显存参数 |
| 启动极慢（几分钟） | 首次下载权重 / 编译 kernel | 有缓存后第二次快 |
| 忘开 `--enforce-eager` 导致 OOM | CUDA Graph 捕获阶段额外占显存 | 低显存**必须**开 |
| 7B-AWQ 起得来但一请求就 OOM | max-model-len 太大 | 降到 512，仅验证流程 |
| 中文输出断在半句 | `max_tokens` 太小 | 调大到 512 |

---

## 八、练习清单（打勾用）

- [ ] 装 uv + Python 3.12，venv 隔离成功
- [ ] torch 显示 `cuda.is_available() = True`
- [ ] 1.5B-AWQ 服务起在 8000
- [ ] 读懂启动日志中的 `GPU KV cache size`
- [ ] curl 与 openai SDK 各调通一次
- [ ] `vllm bench serve` 跑出 TTFT/TPOT/吞吐
- [ ] 并发 1 / 2 / 4 三组对照，画出吞吐曲线
- [ ] 手动制造一次 OOM，并按 §3.4 修复
- [ ] 抓到 `/metrics` 里的 `gpu_cache_usage_perc`
- [ ] 换成 3B-AWQ 再跑一遍，对比显存与速度

---

## 九、附录 A：Docker 方案（更省事的替代）

```bash
# 前置：nvidia-container-toolkit
sudo apt install -y nvidia-container-toolkit
sudo nvidia-ctk runtime configure --runtime=docker
sudo systemctl restart docker

# 起服务（镜像自带 Python 3.12 + CUDA 运行时，无需本机 Toolkit）
docker run --rm --gpus all --ipc=host \
  -v ~/vllm-lab/hf:/root/.cache/huggingface \
  -p 8000:8000 \
  -e HF_ENDPOINT=https://hf-mirror.com \
  vllm/vllm-openai:latest \
  --model Qwen/Qwen2.5-1.5B-Instruct-AWQ \
  --quantization awq --dtype half \
  --max-model-len 8192 --gpu-memory-utilization 0.85 \
  --max-num-seqs 4 --enforce-eager --served-model-name qwen15
```

> 优点：绕过 Python 3.10 限制、环境干净。缺点：镜像 10GB+，需要 docker 权限。

---

## 十、附录 B：迁到生产卡时要改什么

| 项 | 6GB 练习值 | 24GB+ 生产值 |
|---|---|---|
| `--gpu-memory-utilization` | 0.85 | 0.90～0.95 |
| `--max-model-len` | 8192 | 32768 / 模型原生上限 |
| `--max-num-seqs` | 4 | 64～256 |
| `--enforce-eager` | 必开 | 关闭（启用 CUDA Graph 提性能） |
| `--dtype` | half | bfloat16 |
| `--tensor-parallel-size` | 1 | 2 / 4 / 8 |
| 量化 | 必须 AWQ | FP16/BF16 或 FP8 |
| FlashAttention | xformers 回退 | FA2/FA3 原生 |

> **练习的意义就在这里**：参数含义、日志读法、压测方法、指标解读全都不变，只是数值不同。

---

## 十一、一句话总结

- **6GB 上跑 vLLM = 练习工程流程**，不是追求性能
- **本机第一大坎是 Python 3.10**，用 uv 装 3.12 或走 Docker
- **三把钥匙**：`--gpu-memory-utilization 0.85` + `--max-model-len 8192` + `--enforce-eager`
- **必看一行日志**：`GPU KV cache size`，决定你能并发多少、聊多长
- **压测只看趋势不看绝对值**：sm_75 比 A100 慢 3～5 倍是正常的
- 学会后迁到生产卡，**命令几乎不变，只是把数字放大**

> 相关：[[knowledge/infra/ai/01-模型推理加速与部署\|01-模型推理加速与部署]]（四层栈与 Triton）· [[knowledge/infra/ai/02-DeepSeek技术全景\|02-DeepSeek技术全景]]（DeepInfer 与 vLLM 的关系）· [[knowledge/infra/ai/03-Ollama本地部署方案\|03-Ollama本地部署方案]]（本机正式方案）
