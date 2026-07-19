---
{"dg-publish":true,"permalink":"/knowledge/tools/FunClip部署方案/","tags":["funclip","docker","cuda","部署","AI","视频剪辑","竖屏"],"dg-note-properties":{"date":"2026-07-12","tags":["funclip","docker","cuda","部署","AI","视频剪辑","竖屏"]}}
---


# FunClip 一体化部署方案

> **FunClip** 是阿里达摩院 FunASR 开源的视频智能剪辑工具，支持语音识别、字幕生成、说话人分离、LLM 智能裁剪。

**适用场景：** 宿主机 i5‑11400 + GTX1660S‑6G，Ubuntu(WSL2)，Docker 容器内运行 FunClip，将横屏日语视频自动转为 9:16 竖屏短视频。

---

## 🐳 Dockerfile

```dockerfile
FROM nvidia/cuda:11.8.0-base-ubuntu22.04

ENV TZ=Asia/Tokyo
ENV DEBIAN_FRONTEND=noninteractive
WORKDIR /workspace

# 更换阿里apt源、安装python3.10以及系统依赖
RUN sed -i 's/archive.ubuntu.com/mirrors.aliyun.com/g' /etc/apt/sources.list && \
 sed -i 's/security.ubuntu.com/mirrors.aliyun.com/g' /etc/apt/sources.list && \
 apt update && apt install -y python3.10 python3-pip python3.10-dev ffmpeg imagemagick && rm -rf /var/lib/apt/lists/* && \
 update-alternatives --install /usr/bin/python3 python3 /usr/bin/python3.10 1

# 设置pip清华源
COPY requirements.txt ./
RUN python3 -m pip config set global.index-url https://pypi.tuna.tsinghua.edu.cn/simple && \
 python3 -m pip install torch torchvision torchaudio --index-url https://download.pytorch.org/whl/cu118 && \
 python3 -m pip install -r requirements.txt

# 复制全部代码
COPY . ./

EXPOSE 7860
CMD ["python3","funclip/launch.py","--server-name=0.0.0.0"]
```

## 🐙 docker-compose.yml（推荐）

```yaml
version: "3.8"

services:
  funclip:
    build: .
    runtime: nvidia
    ports:
      - "7860:7860"
    volumes:
      - ./videos:/workspace/videos
      - ./output:/workspace/output
      - ./model_cache:/root/.cache/modelscope
    environment:
      TZ: Asia/Tokyo
      NVIDIA_VISIBLE_DEVICES: 0
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: all
              capabilities: [gpu]
    restart: unless-stopped
```

## 🛠️ 构建 & 运行

```bash
# 进入工作目录
cd /home/yang/services/funclip

# 构建镜像
docker compose build

# 启动服务
docker compose up -d

# 查看容器状态
docker compose ps
```

## ✅ 确认容器正常运行

### 1. 查看容器状态

状态为 `Up` 代表正常运行。

### 2. 确认 GPU 可用（非常重要）

```bash
docker exec -it funclip-funclip-1 python3 -c "import torch;print(torch.cuda.is_available())"
```

返回 `True` ✅ GPU 正常启用；  
如果返回 `False`，检查宿主机 `nvidia-container-toolkit` 是否安装正确。

### 3. 打开 Web 管理页面

浏览器访问：`http://宿主机IP:7860`

---

## ⚙️ 页面配置（只需配置一次）

进入页面上方的【设置】标签页，分三大模块配置。

### 模块 1：本地 GPU‑ASR 配置（日语识别）

| 配置项 | 值 |
|:------|:---|
| 语音识别引擎 | `Local‑FunASR/SenseVoice` |
| Model ID | `iic/SenseVoiceSmall`（显存占用 ~500MB） |
| 备选模型 | `iic/paraformer‑ja‑large‑asr`（峰值显存 4.2GB） |
| 识别语种 | **日语(ja)**，关闭自动语种检测 |
| 推理设备 | `cuda` |
| 勾选 | ✅ 输出带时间戳字幕、自动标点补全 |

### 模块 2：自定义 FFmpeg 滤镜（导出直接 9:16 竖屏 — 方案 1 核心）

找到 **自定义FFmpeg视频滤镜**，完整填入：

```
scale=iw*min(1080/iw,1920/ih):ih*min(1080/iw,1920/ih),pad=1080:1920:(1080-iw*min(1080/iw,1920/ih))/2:(1920-ih*min(1080/iw,1920/ih))/2,split[v1][v2];[v1]crop=1080:1920,boxblur=8:8[bg];[v2]overlay=(1080-w)/2:(1920-h)/2
```

**参数说明：**
- 输出分辨率固定 **1080×1920**（短视频竖屏标准）
- 画面主体自动缩放居中，空余区域高斯模糊填充背景
- `boxblur=8` 控制模糊强度，数值越大越模糊

### 模块 3：LLM 智能切片配置（DeepSeek）

| 配置项 | 值 |
|:------|:---|
| 接口类型 | OpenAI 兼容接口 |
| API‑Key | DeepSeek 官网申请的密钥 |
| API‑Base‑URL | `https://api.deepseek.com/v1` |
| 模型名称 | `deepseek-v4-flash` |
| Temperature | `0.1`（降低随机性） |

**日语提示词（完整粘贴）：**

```
あなたは動画切り出し専門アシスタントです。提供された日本語字幕に基づき、動画を適切な長さで分割してください。
ルール：
1. 各クリップの長さは15秒～180秒の範囲に収め、無言区間・無駄な雑談を削除する。
2. 字幕に存在しない架空の時間や文章を生成してはいけない。
3. 出力はJSON形式だけにし、キーはstart(秒数),end(秒数),score(0～10),summary(15文字以内の要約)。
説明文、Markdown、不要な記号を一切出力しないでください。
```

> 💡 最后点击【保存设置】。首次保存时容器会下载 SenseVoiceSmall 模型到宿主机 `model_cache` 文件夹，后续启动无需重复下载。

---

## 🎬 日常完整操作流程

> 放入横屏视频 → GPU 生成字幕 → AI 自动分段 → 导出即为 9:16 竖屏成品

| 步骤 | 操作 |
|:---|:-----|
| **①** | 将横屏日语视频放到 `./videos` 目录 |
| **②** | 切换到【剪辑】页面，点击加载本地视频 |
| **③** | 点击【语音识别】，GPU 加速 ASR，30 分钟视频约 2-4 分钟完成 |
| **④** | （可选）校对字幕中识别错误的短句 |
| **⑤** | 点击【LLM 智能段落选取】，DeepSeek 自动划分高光片段 |
| **⑥** | 勾选需要导出的片段，点击批量渲染导出 |
| **⑦** | 成品在 `./output` 目录，mp4 已自动转为 9:16 竖屏，可直接发布 |

---

## 🛡️ 日常运维命令

```bash
# 查看日志（排错）
docker compose logs -f funclip

# 重启容器
docker compose restart funclip

# 停止服务
docker compose down

# 进入容器内部调试
docker exec -it funclip-funclip-1 bash
```

---

## 💻 硬件负载参考

| 硬件 | 负载情况 |
|:---|:---------|
| GPU GTX1660S‑6G | SenseVoiceSmall ~500MB；paraformer‑ja‑large 峰值 4.2GB |
| CPU i5‑11400 | FFmpeg 解码 + 应用逻辑，多核负载适中 |
| 内存 16GB | 常驻 3‑6GB，完全够用 |

---

## 🔧 可选优化

### 滤镜替换 — 黑色背景版

不想要模糊效果，可用纯黑背景：

```
scale=1080:-1,pad=1080:1920:(1080-ow)/2:(1920-oh)/2:black
```

### 全自动方案

丢视频到 `videos` 目录即自动完成识别、切片、导出竖屏，可编写目录监听脚本实现，按需补充。
