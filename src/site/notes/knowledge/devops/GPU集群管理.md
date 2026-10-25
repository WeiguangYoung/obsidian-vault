---
{"dg-publish":true,"permalink":"/knowledge/devops/GPU集群管理/","tags":["GPU","CUDA","Nvidia","TensorRT","Triton","推理","K8s","AI基础设施"],"dg-note-properties":{"date":"2026-10-25","tags":["GPU","CUDA","Nvidia","TensorRT","Triton","推理","K8s","AI基础设施"]}}
---


# GPU 集群管理：Nvidia Driver / CUDA / TensorRT / Triton

> 面向 AI 训练与推理场景的 GPU 基础设施全景：从底层驱动到上层推理服务的四层栈，以及版本兼容、容器化、K8s 调度、可观测性与排障要点。
> 一句话主线：**Driver 决定能跑多新的 CUDA；CUDA 决定能跑多新的算子库；TensorRT 决定单模型多快；Triton 决定多模型怎么统一上线。**

---

## 〇、全景：四层栈与各组件定位

```
┌─────────────────────────────────────────────────────────┐
│  应用 / 框架层    PyTorch · TensorFlow · vLLM · ...        │
├─────────────────────────────────────────────────────────┤
│  推理服务层       Triton Inference Server                  │  ← 多模型 / 多后端 / 动态批处理
├─────────────────────────────────────────────────────────┤
│  推理加速层       TensorRT · cuDNN · cuBLAS · NCCL          │  ← 单模型极致优化
├─────────────────────────────────────────────────────────┤
│  计算层           CUDA Runtime / CUDA Driver API           │  ← 算子运行时
├─────────────────────────────────────────────────────────┤
│  驱动层           Nvidia Driver (nvidia.ko) + nvidia-smi   │  ← 内核态，版本决定天花板
├─────────────────────────────────────────────────────────┤
│  硬件层           GPU（A100/H100/L40S…）· NVLink · MIG      │
└─────────────────────────────────────────────────────────┘
```

| 组件 | 定位 | 一句话 |
|---|---|---|
| **Nvidia Driver** | 内核态驱动 + 用户态库 | 最底层，决定「这块卡能被用到什么程度」 |
| **CUDA** | 并行计算平台与编程模型 | 算子运行时，装机时最先被「版本地狱」缠上的东西 |
| **TensorRT** | 推理专用优化器/运行时 | 把训练好的模型「编译」成极致推理引擎 |
| **Triton** | 通用推理服务框架 | 把一堆模型统一管起来、并发跑起来、可观测起来 |

---

## 一、Nvidia Driver

### 1.1 作用与组成

驱动是内核态模块 + 用户态接口的组合，负责让 OS 认识 GPU、分配显存、调度 kernel。

- **内核模块**：`nvidia`（核心）、`nvidia_uvm`（统一内存/GPU 直连宿主内存，CUDA 必需）、`nvidia_modeset`（显示）、`nvidia_drm`（DRM/KMS，多为显示用途）
- **用户态**：`libcuda.so`（Driver API，`nvidia-smi` 也依赖它）、`nvidia-smi`、`nvidia-persistenced`

```bash
# 查看驱动版本 / GPU 型号 / 显存 / 温度 / 功耗 / 进程占用
nvidia-smi

# 持续观察（1s 一次）
nvidia-smi -l 1

# 查看内核模块
lsmod | grep nvidia

# 查看卡上所有进程
nvidia-smi --query-compute-apps=pid,process_name,used_memory --format=csv
```

> ⚠️ 关键认知：`nvidia-smi` 头部那个 `CUDA Version: 12.4` **不是已安装的 CUDA Toolkit 版本**，而是**该驱动能支持的最高 CUDA Runtime（Driver API）版本**。实际用哪版 CUDA 取决于容器/环境里的 Toolkit。

### 1.2 安装方式

| 方式 | 命令 | 适用 |
|---|---|---|
| 发行版仓库 | `apt install nvidia-driver-550`（Ubuntu） | 简单，版本受仓库限制 |
| 官方 runfile | `bash NVIDIA-Linux-x86_64-550.xx.run` | 指定精确版本、离线环境 |
| 包管理器（仓库） | `dnf install nvidia-driver` / NVIDIA 官方 repo | RHEL/CentOS 系 |
| **容器化（推荐生产）** | **NVIDIA GPU Operator**（K8s） | 集群批量、自动维护驱动 |
| 云镜像 | 云厂商预装（含 DCGM） | 云上 ECS/CCE |

生产集群强烈推荐**用 GPU Operator 或预制镜像统一管理**，避免逐台手工装驱动导致版本漂移。

### 1.3 驱动分支

- **Production Branch（生产分支）**：稳定，长期维护，生产首选
- **New Feature Branch**：新功能，稳定性次之
- **Data Center 分支 vs Desktop 分支**：数据中心卡（A100/H100/L40S）用 DC 驱动；消费卡（RTX）用 Game Ready / Studio
- **不装图形界面（headless）**：服务器上通常 `--no-opengl-files`，避免与 Nouveau 冲突

### 1.4 驱动版本决定 CUDA 天花板

驱动与 CUDA 是「**向下兼容**」关系：新驱动能跑旧 CUDA，旧驱动跑不了新 CUDA。

| 驱动版本（≥） | 支持的最高 CUDA |
|---|---|
| 535.x | 12.2 |
| 545.x | 12.3 |
| 550.x | 12.4 |
| 555.x | 12.5 |
| 560.x | 12.6 |
| 570.x | 12.8 |

**Minor Version Compatibility（CUDA 11+）**：同一大版本内（如 11.x），较新的 Toolkit 可以在只支持 11.0 的驱动上运行（不依赖新驱动特有能力时）。大版本之间（11 → 12）则必须升级驱动。

### 1.5 常见坑

- **Nouveau 冲突**：Ubuntu 默认开源驱动会抢卡 → 安装前 `blacklist nouveau`
- **Secure Boot**：开启时会拒绝未签名内核模块 → 需签名或关 Secure Boot
- **`nvidia_uvm` 未加载**：CUDA 报 `Unable to determine the device handle` → 检查模块与权限
- **内核升级后驱动失效**：需 **DKMS** 自动重编译内核模块，否则重启后 `nvidia-smi` 报 "Driver/library version mismatch"
- **`nvidia-smi` 卡住**：常见于 Xid 错误、卡掉线（用 `nvidia-smi -q`、dmesg 查 Xid）

---

## 二、CUDA

### 2.1 组成

CUDA Toolkit 不止一个 `nvcc`：

| 组成 | 说明 |
|---|---|
| **CUDA Runtime**（`libcudart`） | 应用最常链接的运行时，`cudaMalloc` 等 |
| **CUDA Driver API**（`libcuda`） | 更底层，随驱动安装 |
| **nvcc / nvrtc** | 编译器（离线编译 / 运行时编译） |
| **数学库** | cuBLAS、cuFFT、cuSPARSE、cuDNN、cuRAND… |
| **NCCL** | 多卡/多机集合通信（分布式训练命脉） |
| **工具** | `cuda-gdb`、`nsight`、`compute-sanitizer`、`nvprof`（已弃用→`nsys`/`ncu`） |

```bash
# 已安装 toolkit 版本
nvcc --version
# 或
cat /usr/local/cuda/version.json
# 运行时版本（Python）
python -c "import torch; print(torch.version.cuda)"
```

### 2.2 三个"CUDA 版本"别搞混

| 版本 | 来源 | 含义 |
|---|---|---|
| **Driver API 版本** | `nvidia-smi` 头部 | 驱动支持的最高 CUDA |
| **Runtime 版本** | `cudart` / 框架报告 | 应用实际运行的 CUDA 运行时 |
| **Toolkit 版本** | `nvcc --version` | 编译器/工具链版本 |

三者可不同，只要满足：Toolkit/Runtime ≤ Driver 支持上限。

### 2.3 兼容性规则

- **CUDA 保证向前兼容（Forward Compatibility）**：为旧 CUDA 编译的程序可在新驱动上运行
- **不保证反向（Backward）**：新 CUDA 程序跑不了旧驱动
- **Minor Version Compatibility**：CUDA 11/12 内部小版本可跨（11.x 程序跑在 11.0 驱动上）
- **PTX JIT**：若编译时嵌入了 PTX（默认包含），新驱动可在运行时把 PTX 编译成对应算力的 SASS → 提升向前兼容性；但会牺牲启动性能

### 2.4 环境变量

```bash
# 只暴露指定卡（多用户/多任务隔离）
CUDA_VISIBLE_DEVICES=0,1
# 按 UUID 更稳（避免序号漂移）
CUDA_VISIBLE_DEVICES=GPU-xxxxxxxx-....
# 强制禁用某卡
CUDA_VISIBLE_DEVICES=""
```

### 2.5 容器内跑 CUDA

容器本身不含驱动（驱动在内核态，由宿主提供），但需要用 **nvidia-container-toolkit** 把设备与用户态库注入容器：

```bash
# 安装 nvidia-container-toolkit 后
docker run --rm --gpus all nvidia/cuda:12.4.0-base-ubuntu22.04 nvidia-smi
docker run --rm --gpus '"device=0,1"' ...
```

docker-compose 写法：

```yaml
services:
  train:
    image: nvidia/cuda:12.4.0-runtime
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: all
              capabilities: [gpu]
```

> 老教程里的 `runtime: nvidia`（旧 nvidia-docker2）已过时；新方式统一走 `--gpus`。

---

## 三、K8s 上的 GPU 支撑

### 3.1 核心组件

| 组件 | 作用 |
|---|---|
| **NVIDIA GPU Operator** | 一站式：驱动、container-toolkit、device-plugin、DCGM、NFD 全套编排 |
| **nvidia-device-plugin** | 把 GPU 暴露成 K8s 可调度资源 `nvidia.com/gpu` |
| **Node Feature Discovery (NFD)** | 探测节点硬件特征、打标签（如 `nvidia.com/gpu.present=true`） |
| **DCGM-Exporter** | 采集 GPU 指标给 Prometheus |
| **MIG Manager** | 把 A100/H100 切成多个 MIG 实例 |

### 3.2 调度与配额

```yaml
resources:
  limits:
    nvidia.com/gpu: 1        # 关键：GPU 只能写在 limits，requests 需 == limits
```

- GPU 是**不可压缩资源**，K8s 不接受 `limits` 与 `requests` 不等
- 分配粒度默认整卡；细粒度用 **MIG** 或 **时间片（time-slicing，非隔离）**
- nodeSelector / 污点：让训练/推理任务落到对应 GPU 池

```yaml
nodeSelector:
  nvidia.com/gpu.product: NVIDIA-A100-SXM4-80GB
```

### 3.3 类别

- **整卡独占**：默认，隔离最好
- **MIG**：A100 最多切 7 份，H100 可更多，硬件级隔离（推荐多租户）
- **Time-Slicing**：软切分、无隔离，适合推理高密度，慎用于训练

---

## 四、TensorRT（推理加速）

### 4.1 定位

NVIDIA 的**推理专用优化器 + 运行时**：把训练好的模型（多为 ONNX）「编译」成针对**特定 GPU 架构 + 特定精度**高度优化的 **Engine**，换来数倍于原生框架的推理吞吐/延迟。

核心优化手段：层/张量融合、精度量化（FP16/INT8）、kernel 自动调优（autotuning）、显存复用、动态 shape 支持。

### 4.2 工作流

```
训练产出 → 导出 ONNX → trtexec/polygraphy 构建 Engine → TensorRT Runtime 加载推理
                          （build 阶段，耗时，与硬件绑定）
```

```bash
# 命令行构建 engine（FP16）
trtexec --onnx=model.onnx --fp16 --saveEngine=model.plan \
        --minShapes=input:1x3x224x224 \
        --optShapes=input:8x3x224x224 \
        --maxShapes=input:32x3x224x224

# 直接跑 benchmark
trtexec --loadEngine=model.plan --shapes=input:8x3x224x224 --iterations=100
```

### 4.3 精度模式

| 精度 | 说明 | 权衡 |
|---|---|---|
| FP32 | 高精度，慢 | 基准 |
| FP16 | 主流，速度约 2x | 精度损失极小，推荐 |
| INT8 | 最快，需 **calibration**（量化校准） | 精度需验证，需校准集 |
| BF16 | A100/H100 友好 | 兼顾范围与速度 |

INT8 校准流程：准备代表性校准集 → 生成 calibration cache → build engine；cache 可跨构建复用。

### 4.4 版本强绑定（关键坑）

TensorRT 的 **Engine 不可移植**：

- **绑定 TensorRT 版本**：换 TRT 版本必须重新 build
- **绑定 GPU 架构（SM 版本）**：A100(sm80) 上 build 的 engine 跑不了 H100(sm90)
- **绑定平台**：一般 x86/ARM 不同
- **版本-Toolkit 绑定**：TRT 与 CUDA/cuDNN 版本严格对应（见下表）

| TensorRT | CUDA | cuDNN |
|---|---|---|
| 8.6 | 12.x | 8.9 |
| 10.0 | 12.4 | 9.x |
| 10.3 | 12.5 | 9.x |
| 10.5 | 12.6 | 9.x |

> 生产实践：**engine 应在目标 GPU 上、用目标版本现场构建**（首次启动预热），而不是分发预构建 engine。

---

## 五、Triton Inference Server（推理服务化）

### 5.1 定位

NVIDIA 开源的**通用推理服务框架**：一个进程统一托管多模型、多后端，内置动态批处理、并发执行、模型版本管理、ENSEMBLE 编排，自带 Prometheus 指标。

### 5.2 支持的 Backend

`tensorrt` · `onnxruntime` · `pytorch(libtorch)` · `python` · `openvino` · `fil` · `dali` · `vllm`（后起，LLM 场景）等。

### 5.3 模型仓库结构

```
model_repository/
├── resnet50/
│   ├── config.pbtxt        # 模型配置（必需，或自动推断）
│   └── 1/                  # 版本目录（数字）
│       └── model.plan
├── bert_onnx/
│   ├── config.pbtxt
│   └── 1/
│       └── model.onnx
└── ensemble_pipeline/
    ├── config.pbtxt        # platform: ensemble
    └── 1/                  # 空目录
```

### 5.4 `config.pbtxt` 关键字段

```protobuf
name: "resnet50"
platform: "tensorrt_plan"          # 或 backend: "onnxruntime"
max_batch_size: 32

input  { name: "input"  data_type: TYPE_FP16  dims: [3, 224, 224] }
output { name: "output" data_type: TYPE_FP16  dims: [1000] }

dynamic_batching {                   # 动态批处理：攒批提升吞吐
  preferred_batch_size: [8, 16]
  max_queue_delay_microseconds: 5000
}

instance_group [{                    # 每卡并发实例数
  count: 2
  kind: KIND_GPU
  gpus: [0]
}]
```

### 5.5 三大核心能力

| 能力 | 说明 |
|---|---|
| **Dynamic Batching** | 把多个请求攒成一个 batch 推理，大幅提吞吐（代价：单请求延迟↑） |
| **Concurrent Model Execution / Instance Groups** | 同一模型多实例并行；多模型可共享 GPU |
| **Model Ensemble** | 多模型 DAG 编排（预处理→推理→后处理）在一个请求内串起来 |
| **Sequence Batching** | 有状态模型（如对话、流式）按序列维护状态 |

### 5.6 端口与客户端

| 端口 | 协议 |
|---|---|
| 8000 | HTTP（REST） |
| 8001 | gRPC |
| 8002 | Prometheus 指标 |

```bash
# 启动
tritonserver --model-repository=/models --log-verbose=1

# 客户端（Python）
import tritonclient.http as httpclient
c = httpclient.InferenceServerClient(url="localhost:8000")
```

### 5.7 部署形态

- **Docker**：`nvcr.io/nvidia/tritonserver:24.xx-py3`
- **K8s**：官方 Helm chart / 自建 Deployment + Service（GPU 资源按 §3.2 声明）
- 生产常配：多副本 + HPA（按 QPS/GPU 利用率）、模型仓库来自对象存储/PVC、Triton 的 **model control API** 支持热更新模型

---

## 六、版本矩阵与升级策略

### 6.1 依赖链（从上到下逐层收窄）

```
Driver  ──决定──▶  最高 CUDA  ──决定──▶  库(cuDNN/NCCL)  ──决定──▶  TensorRT  ──决定──▶  Engine
```

升级必须**自底向上、逐层验证**：先升驱动 → 验证 CUDA → 再升 cuDNN/TensorRT → 最后重 build engine。

### 6.2 容器镜像选型

优先用**官方组合镜像**避免手工对齐：

- `nvidia/cuda:<ver>-<cudnn|devel|runtime>-<os>`
- `nvcr.io/nvidia/pytorch:<yy.mm>-py3`（NGC，框架+CUDA+cuDNN+TRT 全对齐）
- `nvcr.io/nvidia/tritonserver:<yy.mm>-py3`

### 6.3 升级 checklist

1. 记录当前四层版本（`nvidia-smi` / `nvcc` / `python -c "import torch..."` / `trtexec --version`）
2. 查官方兼容表，确定目标组合
3. 灰度单节点升级 + 跑基准（吞吐/延迟/精度）
4. Engine 全部重新 build（§4.4）
5. 更新镜像 tag 与 K8s 部署

---

## 七、可观测性

| 手段 | 内容 |
|---|---|
| `nvidia-smi` / `nvidia-smi dmon` | 显存、利用率、温度、功耗、ECC、Xid |
| **DCGM / dcgm-exporter** | 标准 GPU 指标 → Prometheus（`DCGM_FI_DEV_GPU_UTIL`、`DCGM_FI_DEV_FB_USED` 等） |
| Grafana | 官方 GPU 看板（ID 12239 等） |
| Triton metrics | `:8002/metrics`（QPS、延迟分位、批大小、队列） |
| 告警 | 显存打满、温度超阈、Xid 错误、掉卡、ECC 不可纠正错误 |

```bash
# 常用查询
nvidia-smi --query-gpu=index,name,utilization.gpu,memory.used,memory.total,temperature.gpu --format=csv
```

---

## 八、常见问题排查

| 现象 | 可能原因 | 处理 |
|---|---|---|
| `nvidia-smi` 与 CUDA 版本对不上 | 混淆了「驱动支持上限」与「已装 toolkit」 | 分别确认三层版本（§2.2） |
| `Driver/library version mismatch` | 内核升级后模块未重建 | 装 DKMS / 重启 / 重装驱动 |
| CUDA 程序报 `no kernel image is available` | PTX/SASS 与 GPU 算力不匹配 | 重编译并指定 `-arch=sm_XX` |
| TensorRT engine 加载失败 | 版本/架构不匹配 | 在目标卡上重 build |
| 容器里看不到 GPU | 未装 nvidia-container-toolkit / 未传 `--gpus` | 装 toolkit + `--gpus all` |
| K8s Pod Pending（要 GPU） | device-plugin 未就绪 / `nvidia.com/gpu.present` 标签缺失 | 查 device-plugin DaemonSet、NFD 标签 |
| 多卡训练慢 | NCCL 走错网卡 / 无 NVLink | 设 `NCCL_SOCKET_IFNAME`、检查拓扑 `nvidia-smi topo -m` |
| 显存泄漏/碎片 | 长期服务未释放 | 定期重启 / 用 `PYTORCH_CUDA_ALLOC_CONF=expandable_segments:True` |
| `nvidia-smi` 卡死 | Xid 错误 / 卡掉线 | dmesg 查 Xid，必要时重置卡（`nvidia-smi -r`） |

---

## 九、一句话总结

- **Driver** 是地基，决定整个栈的版本天花板，生产用 GPU Operator/镜像统一管
- **CUDA** 是运行时，记住「三个版本」与「向前兼容、不向后兼容」
- **TensorRT** 是单模型加速器，**Engine 与版本/架构强绑定**，现场构建
- **Triton** 是服务化框架，靠**动态批处理 + 多后端 + 多模型**把 GPU 利用率拉满
- 升级一律**自底向上、逐层验证**，容器镜像优先官方组合