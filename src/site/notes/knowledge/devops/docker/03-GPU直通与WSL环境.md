---
{"dg-publish":true,"permalink":"/knowledge/devops/docker/03-GPU直通与WSL环境/","title":"Docker GPU 直通与 Windows WSL 环境","tags":["docker","GPU","NVIDIA","WSL","Windows","Docker-Desktop","容器"],"dg-note-properties":{"category":"Docker / 容器 / DevOps","title":"Docker GPU 直通与 Windows WSL 环境","date":"2026-11-10","tags":["docker","GPU","NVIDIA","WSL","Windows","Docker-Desktop","容器"]}}
---


# Docker GPU 直通与 Windows WSL 环境

> 两块内容合一：Linux 上让容器用上 NVIDIA 显卡，Windows 上用 WSL 2 跑 Docker Desktop 并解决 C 盘爆满。
> 一句话主线：**GPU 直通的关键不是 `--gpus` 参数，而是宿主机有没有装对 `nvidia-container-toolkit`；WSL 的关键不是装，而是把它的虚拟磁盘从 C 盘挪走。**

---

# 第一部分 · Docker 使用 NVIDIA 显卡

## 一、前提检查

装之前先确认三件事，缺一个后面都会卡：

```bash
# 1. 宿主机显卡驱动正常（能跑出 GPU 信息）
nvidia-smi

# 2. 驱动版本与 CUDA 版本对应
nvidia-smi | grep "CUDA Version"

# 3. Docker 已装
docker --version
```

> **关键认知**：容器里**不需要装驱动**，用的是宿主机的驱动 + 内核模块。容器里只需要 CUDA runtime / cuDNN 这些用户态库。

---

## 二、安装 nvidia-container-toolkit（新版，推荐）

老教程还在教装 `nvidia-container-runtime` + 手改 `daemon.json`，现在**不需要了**——`nvidia-ctk` 一条命令帮你配好。

### 2.1 加源

```bash
# 官方源（国内可能慢）
curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey \
  | sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg

curl -s -L https://nvidia.github.io/libnvidia-container/stable/deb/nvidia-container-toolkit.list \
  | sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' \
  | sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list
```

### 2.2 安装 + 配置

```bash
sudo apt-get update
sudo apt-get install -y nvidia-container-toolkit

# 关键：让 nvidia-ctk 自动写 daemon.json
sudo nvidia-ctk runtime configure --runtime=docker

# 重启
sudo systemctl restart docker
```

### 2.3 验证

```bash
# 1. daemon.json 里应该有 runtimes.nvidia
cat /etc/docker/daemon.json

# 2. 跑个 GPU 容器
docker run --rm --gpus all nvidia/cuda:12.4.0-base-ubuntu22.04 nvidia-smi
```

跑出 GPU 信息就算成功。

---

## 三、老版方式（了解即可，不推荐）

老教程的流程，遇到老环境才用：

```bash
# 1. 加 nvidia-container-runtime 源
distribution=$(. /etc/os-release; echo $ID$VERSION_ID)
curl -s -L https://nvidia.github.io/nvidia-container-runtime/gpgkey | sudo apt-key add -
curl -s -L https://nvidia.github.io/nvidia-container-runtime/$distribution/nvidia-container-runtime.list \
  | sudo tee /etc/apt/sources.list.d/nvidia-container-runtime.list

# 2. 装
sudo apt-get update && sudo apt-get install -y nvidia-container-runtime

# 3. 手动在 daemon.json 里加 runtime
```

```json
{
  "runtimes": {
    "nvidia": {
      "path": "/usr/bin/nvidia-container-runtime",
      "runtimeArgs": []
    }
  }
}
```

```bash
sudo systemctl daemon-reload && sudo systemctl restart docker
```

> 缺点：`nvidia-container-runtime` 是单层封装，不像 toolkit 那样带 `nvidia-ctk` 自动配置和 CDI 支持。**新环境一律用 toolkit。**

---

## 四、怎么用

### 4.1 命令行

```bash
# 全部 GPU
docker run -it --gpus all <镜像> bash

# 指定数量
docker run -it --gpus 2 <镜像> bash

# 指定具体卡（多卡机器常用）
docker run -it --gpus '"device=0,1"' <镜像> bash

# 指定能力（默认只暴露 compute + utility）
docker run -it --gpus 'all,"capabilities=compute,utility,graphics"' <镜像> bash
```

### 4.2 docker compose

```yaml
services:
  trainer:
    image: pytorch/pytorch:2.4.0-cuda12.1-cudnn9-runtime
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: all          # 或 1 / 2
              capabilities: [gpu]
```

> **注意**：`runtime: nvidia` 是老写法（配 runtimes.nvidia 时用），新写法建议用 `deploy.resources.reservations.devices`。
> **另一点**：`deploy` 段在 `docker compose` v2 的单机模式下**也生效**了，不再需要 swarm。

### 4.3 环境变量方式（最简单的兼容写法）

```bash
docker run --rm --gpus all \
  -e NVIDIA_VISIBLE_DEVICES=all \
  -e NVIDIA_DRIVER_CAPABILITIES=compute,utility \
  nvidia/cuda:12.4.0-base-ubuntu22.04 nvidia-smi
```

---

## 五、典型报错

| 报错 | 根因 | 解法 |
|:--|:--|:--|
| `could not select device driver "" with capabilities: [[gpu]]` | **没装 toolkit**（最常见）| `apt install nvidia-container-toolkit` |
| `Unknown runtime specified nvidia` | `daemon.json` 里没 `runtimes.nvidia` | 跑 `nvidia-ctk runtime configure --runtime=docker` |
| `nvidia-smi: command not found`（容器内）| CUDA 镜像的 `base` 版不含驱动工具 | 换 `devel` 镜像，或确认 `NVIDIA_DRIVER_CAPABILITIES` 含 `utility` |
| `CUDA driver version is insufficient` | 宿主机驱动太旧 | 升级宿主驱动，**不是**在容器里装 |
| `Failed to initialize NVML: Unknown Error` | 驱动与内核模块版本不匹配 | 重启宿主机；或 `nvidia-smi` 本身报错先修宿主机 |

### 关键排查顺序

```
容器里 nvidia-smi 失败
  ↓
① 宿主机 nvidia-smi 能不能跑？      不能 → 修宿主机驱动
  ↓ 能
② docker info 里有没有 nvidia runtime？  没有 → 配 daemon.json
  ↓ 有
③ 用 --gpus all 启动还是报 device driver？  是 → 装 toolkit
  ↓ 否
④ 容器内报 CUDA version insufficient？  → 宿主机驱动版本太低
```

> **90% 的情况是第 ③ 步：装的是 `nvidia-container-runtime` 而不是 `nvidia-container-toolkit`，或者装了没重启 Docker。**

---

## 六、验证一张表

| 检查点 | 命令 | 期望 |
|:--|:--|:--|
| 宿主机驱动 | `nvidia-smi` | 输出 GPU 表 |
| 驱动 CUDA 版本 | `nvidia-smi \| grep CUDA` | ≥ 镜像要求的版本 |
| toolkit 已装 | `dpkg -l \| grep nvidia-container-toolkit` | 有 package |
| runtime 已配 | `docker info \| grep -i runtime` | 含 `nvidia` |
| 容器能跑 | `docker run --rm --gpus all nvidia/cuda:12.4.0-base-ubuntu22.04 nvidia-smi` | 输出 GPU 表 |
| compose 能跑 | `docker compose up` | 同上 |

---

# 第二部分 · Windows WSL 与 Docker Desktop

## 七、安装 WSL

### 7.1 一条命令装完（Win10 2004+ / Win11）

```powershell
# 管理员 PowerShell
wsl --install
```

默认装 Ubuntu + WSL 2。装完重启。

### 7.2 手动选版本

```powershell
# 看可用的发行版
wsl --list --online          # 或 wsl -l -o

# 设置默认版本（推荐 WSL 2）
wsl --set-default-version 2

# 装指定发行版
wsl --install -d Ubuntu-22.04
```

### 7.3 WSL 1 与 WSL 2 的选择

| | WSL 1 | WSL 2 |
|:--|:--|:--|
| 内核 | 翻译层（syscall 转换）| **真实 Linux 内核（轻量 VM）** |
| 文件性能（Linux 侧）| 慢 | **快** |
| 文件性能（访问 Windows 盘）| 快 | 慢（跨 VM 边界）|
| Docker | ❌ 不支持 | ✅ **必须用 WSL 2** |
| 内存占用 | 低 | 高（默认可用宿主一半）|
| 端口转发 | 直接 | 自动（localhost）|

> **结论**：跑 Docker Desktop 必须 WSL 2，没得选。

---

## 八、Docker Desktop 与 WSL 集成

### 8.1 装完后的三个发行版

Docker Desktop 启用 WSL 2 后端后，会多出两个发行版：

```powershell
wsl -l -v
```

```
  NAME                    STATE      VERSION
* Ubuntu-22.04            Running    2
  docker-desktop          Stopped    2       ← Docker 引擎本体
  docker-desktop-data     Stopped    2       ← 镜像/容器数据（占空间大户）
```

> **`docker-desktop-data` 会吃掉几十 GB**——所有镜像、容器层、卷都在这里。**默认位置在 C 盘，这是 C 盘爆满的头号原因。**

### 8.2 设置里启用

Docker Desktop → Settings → Resources → WSL Integration → 勾选要接入的发行版。

勾选后，在该 WSL 发行版里直接 `docker` 命令可用（不用另装）。

---

## 九、把 WSL 虚拟磁盘从 C 盘迁走

这是 Windows 用 Docker 的**必做项**。

### 9.1 三个都要迁

```powershell
# 查看当前列表和状态
wsl -l -v --all

# 全部关掉（关键：不关没法 export）
wsl --shutdown

# ---- 迁移 docker-desktop-data（最重要，占空间最大）----
wsl --export docker-desktop-data D:\Docker\docker-desktop-data.tar
wsl --unregister docker-desktop-data
wsl --import docker-desktop-data D:\Docker\docker-desktop-data\ D:\Docker\docker-desktop-data.tar --version 2

# ---- 迁移 docker-desktop ----
wsl --export docker-desktop D:\Docker\docker-desktop.tar
wsl --unregister docker-desktop
wsl --import docker-desktop D:\Docker\docker-desktop\ D:\Docker\docker-desktop.tar --version 2

# ---- 迁移 Ubuntu（发行版名按自己的改）----
wsl --export Ubuntu-22.04 D:\Docker\Ubuntu.tar
wsl --unregister Ubuntu-22.04
wsl --import Ubuntu-22.04 D:\Docker\Ubuntu\ D:\Docker\Ubuntu.tar --version 2

# 确认
wsl -l -v --all
```

### 9.2 迁完的两个坑

**坑 1：Ubuntu 变成 root 默认用户**

`wsl --import` 不保留原来的默认用户，进去变成 root。

```powershell
# 改回普通用户（发行版名 + 用户名按实际改）
ubuntu2204 config --default-user <用户名>
```

**坑 2：导出的 tar 占空间，确认无误再删**

```powershell
# 确认 Docker Desktop 正常启动、镜像都在之后
Remove-Item D:\Docker\docker-desktop-data.tar
Remove-Item D:\Docker\docker-desktop.tar
Remove-Item D:\Docker\Ubuntu.tar
```

### 9.3 为什么用 export/import 而不是直接改路径

新版 Docker Desktop 设置里可以改 "Disk image location"，但：

- 改了之后只会**影响新数据**，已有数据不会自动搬
- 有些版本改路径后 Docker Desktop 起不来

**export + unregister + import 是唯一可靠的做法**，麻烦但一次到位。

---

## 十、WSL 2 资源限制

WSL 2 默认最多吃宿主机一半内存，且会逐渐膨胀不释放——跑 Docker 时更是。

### 10.1 配置文件

在 `C:\Users\<用户名>\.wslconfig`：

```ini
[wsl2]
memory=8GB              # 最大内存（默认 50%）
processors=4            # 最多用几个核
swap=2GB
localhostForwarding=true

# 关键：空闲多久后回收内存
# 不设这个，WSL 占的内存不会还给你
[experimental]
autoMemoryReclaim=gradual   # 或 dropcache
sparseVhd=true
```

改完执行：

```powershell
wsl --shutdown
# 等 8 秒，再启动
```

### 10.2 压缩虚拟磁盘（释放已占用空间）

删了镜像，`ext4.vhdx` 不会自动变小。

```powershell
# 1. 关 WSL
wsl --shutdown

# 2. 用 diskpart 压缩
diskpart
```

```
select vdisk file="D:\Docker\docker-desktop-data\ext4.vhdx"
attach vdisk readonly
compact vdisk
detach vdisk
exit
```

> **注意**：compact 前必须 `attach readonly`，否则会失败。

---

## 十一、WSL 常见问题

| 问题 | 原因 | 解法 |
|:--|:--|:--|
| `WSL 2 requires an update to its kernel component` | 内核包缺失 | 下载 [WSL2 Linux kernel update](https://aka.ms/wsl2kernel) 安装 |
| 内存被吃光 | 默认无上限回收 | `.wslconfig` 配 `autoMemoryReclaim` |
| C 盘爆满 | docker-desktop-data 在 C 盘 | 迁到 D 盘（第九章）|
| 访问 Windows 文件极慢 | 跨 VM 边界 | 项目代码放 Linux 侧（`/home/`），不要放 `/mnt/c/` |
| 端口访问不了 | localhost 转发未生效 | `.wslconfig` 开 `localhostForwarding=true` |
| 导入后变 root | import 不保留用户 | `ubuntu2204 config --default-user <用户名>` |
| 改了 Windows hosts 无效 | WSL 用独立 hosts | 改 `/etc/hosts` |

### 性能对比（重要）

```
项目放 /mnt/c/Users/xxx/project   →  npm install 要 5 分钟
项目放 /home/xxx/project          →  npm install 要 30 秒
```

**差 10 倍。** WSL 2 跨 VM 访问 Windows 文件系统的开销极大。

---

## 十二、一句话总结

> **GPU 直通：宿主机驱动 + `nvidia-container-toolkit` + 一条 `nvidia-ctk runtime configure`，三步搞定；报 `could not select device driver` 就是 toolkit 没装。**
>
> **WSL：装很简单，麻烦在数据位置——`docker-desktop-data` 必须迁出 C 盘，项目代码必须放 Linux 侧，`.wslconfig` 必须配内存回收。这三条不做，用一个月就会想重装系统。**
