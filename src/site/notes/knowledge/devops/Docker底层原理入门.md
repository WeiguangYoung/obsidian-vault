---
{"dg-publish":true,"permalink":"/knowledge/devops/Docker底层原理入门/","tags":["Docker","容器","Linux","Namespace","Cgroup","底层原理"],"dg-note-properties":{"date":"2026-07-06","tags":["Docker","容器","Linux","Namespace","Cgroup","底层原理"]}}
---


# 🐳 Docker 底层原理入门 — 不只是会用，而是理解它

## 容器到底是什么？

很多人把 Docker 容器说成「轻量级虚拟机」——**这个说法不完全对，但也不完全错**。

更准确的说法是：

> **容器 = 一个被隔离的 Linux 进程**

Docker 没有发明什么新的「虚拟化技术」，它只是把 Linux 内核已有的几个机制组合起来，做了一套好用的上层工具。

---

## Docker 的底层四大支柱

```
┌──────────────────────────────────────────────┐
│                  Docker CLI/API               │
├──────────────────────────────────────────────┤
│            containerd (容器运行时管理)          │
├──────────────────────────────────────────────┤
│    runc (OCI 容器运行时，真正的容器启动器)       │
├────────────────┬──────────────┬───────────────┤
│  Namespace     │   Cgroup     │  UnionFS      │
│  (隔离)         │  (资源限制)   │  (分层镜像)    │
├────────────────┴──────────────┴───────────────┤
│              Linux Kernel                      │
└──────────────────────────────────────────────┘
```

### 1️⃣ Namespace — 实现「隔离」

**Namespace 做的事情**：让进程只能看到自己的世界，看不到外面。

Linux 有 8 种 Namespace，Docker 容器用了其中 6~7 种：

| Namespace             | 隔离什么    | 容器中的体现                       |
|:--------------------|:------|:---------------------------|
| **PID**               | 进程编号    | 容器里看到的是 PID 1，外面可能是 PID 4231 |
| **Network**           | 网络栈     | 容器有自己的 eth0、IP、端口            |
| **Mount**             | 文件系统挂载点 | 容器看到的是自己的文件系统，看不到宿主机         |
| **UTS**               | 主机名和域名  | 容器有自己的 hostname              |
| **IPC**               | 进程间通信   | 容器内进程不能跨容器通3信                |
| **User**              | 用户 ID   | 容器里的 root 在外面可能只是个普通用户       |
| **Cgroup**            | 资源视图    | 让容器看到受限的 CPU/内存视图            |
| **Time** (Linux 5.6+) | 系统时间    | 每个容器可以有独立的时间偏移               |

**关键理解**：Namespace 是 Linux 内核的一个属性，创建进程时可以指定用哪些 Namespace。`clone()` 系统调用是这个机制的入口：

```c
// 底层原理：clone() 系统调用创建新 Namespace
clone(child_func, stack, 
      CLONE_NEWPID | CLONE_NEWNET | CLONE_NEWNS | CLONE_NEWUTS,
      args);
```

容器本质上就是调用 `clone()` 时带上了 `CLONE_NEW*` 标志的普通进程。

### 2️⃣ Cgroup — 实现「限制」

Namespace 解决的是「看不见」的问题，Cgroup 解决的是「不能抢」的问题。

**Cgroup (Control Groups)**：限制、记录、隔离进程组的物理资源使用。

```
/sys/fs/cgroup/
├── cpu/              ← CPU 时间配额
├── memory/           ← 内存使用上限
├── blkio/            ← 磁盘 I/O
├── cpuset/           ← CPU 亲和性（绑定到某颗核）
├── pids/             ← 进程数上限
├── net_prio/         ← 网络优先级
└── freezer/          ← 暂停/恢复进程组
```

Docker 容器启动时会在这些子系统下创建自己的 cgroup 目录：

```bash
/sys/fs/cgroup/cpu/docker/<container-id>/cpu.cfs_quota_us
/sys/fs/cgroup/memory/docker/<container-id>/memory.limit_in_bytes
```

**举例**：限制容器只能用 0.5 个 CPU 核心 + 256MB 内存

```bash
# 本质上就是写这几个文件
echo 50000 > /sys/fs/cgroup/cpu/docker/<cid>/cpu.cfs_quota_us
echo 268435456 > /sys/fs/cgroup/memory/docker/<cid>/memory.limit_in_bytes
```

> **关键认识**：Docker `--memory` 和 `--cpus` 参数，底层就是写 cgroup 文件。没什么魔法。

### 3️⃣ UnionFS — 实现「分层镜像」

这是 Docker 最巧妙的设计之一。

**UnionFS（联合文件系统）**：把多个目录"叠加"成一个目录。

```
容器可写层 (Container Layer)     ← 容器运行时的修改都写在这
  ┃ Merge
镜像层 N (Image Layer N)         ← apt install 新包
  ┃ ...
镜像层 2 (Image Layer 2)         ← 修改配置文件
镜像层 1 (Image Layer 1)         ← FROM ubuntu:22.04
镜像层 0 (Image Layer 0)         ← ubuntu:22.04 基础层
```

**关键机制**：
- **Copy-on-Write (写时复制)**：读文件时从最下层往上找，到找到为止；写文件时先把文件复制到可写层再改
- **层复用**：多个容器共享相同的镜像层，不重复存储

常见 UnionFS 实现：

| 驱动 | 特点 | 适用场景 |
|:----|:-----|:--------|
| **overlay2** ⭐ | 当前默认，性能好 | 生产环境首选 |
| **aufs** | 早期默认，已淘汰 | 历史兼容 |
| **devicemapper** | 基于设备映射器 | CentOS 历史版本 |
| **btrfs/zfs** | 基于快照 | 特定存储后端 |

```bash
# 查看当前 Docker 使用的存储驱动
docker info | grep "Storage Driver"
# Storage Driver: overlay2
```

`overlay2` 的底层原理：

```
/var/lib/docker/overlay2/
├── <layer-hash>/diff/   ← 这一层的实际文件
│   └── etc/nginx/nginx.conf
├── <layer-hash>/link    ← 指向下一层的链接
└── <container-id>/
    └── diff/            ← 容器可写层
```

**overlay 挂载命令**（Docker 在后台就是这么做的）：

```bash
mount -t overlay overlay \
  -o lowerdir=/lower,upperdir=/upper,workdir=/work \
  /merged
```

### 4️⃣ 容器网络 — veth pair + bridge

```
宿主机网络
┌─────────────────────────────────────────────┐
│  docker0 (bridge, 172.17.0.1)               │
│     ├─ vethXXX ──► 容器 eth0 (172.17.0.2)    │
│     ├─ vethYYY ──► 容器 eth0 (172.17.0.3)    │
│     └─ ...                                   │
│  NAT (iptables MASQUERADE) → 外网访问         │
└─────────────────────────────────────────────┘
```

**核心：veth pair** — 一根虚拟网线，一头在宿主机，一头在容器的 Network Namespace 里。

```bash
# 底层大致流程
ip link add veth0 type veth peer name eth0  # 创建 veth pair
ip link set eth0 netns <container-ns>         # 一头放进容器
ip link set veth0 master docker0              # 一头桥接到 docker0
```

---

## docker run 的背后发生了什么？

```bash
docker run -it --rm -m 256m --cpus 0.5 nginx bash
```

这条命令执行时，幕后：

```
1. Docker CLI → Docker Daemon (REST API over Unix Socket)
2. Daemon 检查本地有没有 nginx 镜像
   ├─ 没有 → pull 镜像（拉取各层）
   └─ 有 → 复用已有层
3. Daemon 调用 containerd
4. containerd 用 runc 创建容器
5. runc 调用 Linux 内核：
   ├─ clone(CLONE_NEWPID|CLONE_NEWNET|CLONE_NEWNS|...) → 创建 Namespace
   ├─ mount overlay2 → 准备容器文件系统
   ├─ echo 256M → cgroup memory 限制
   ├─ echo 50000 → cgroup cpu 限制
   ├─ ip link add veth... → 创建网络接口
   └─ pivot_root → 切换根文件系统
6. chroot / 切换进去 → exec bash
```

整个过程不超过 **100ms**。这就是容器「快」的根本原因——它只是进程，不是操作系统。

---

## Docker ≠ 虚拟化

| 对比 | 容器 (Docker) | 虚拟机 (VM) |
|:----|:--------------|:-----------|
| **启动速度** | 毫秒级 | 秒~分钟级 |
| **内核** | 共享宿主机内核 | 每个 VM 独立内核 |
| **隔离粒度** | 进程级 (Namespace) | 硬件级虚拟化 |
| **资源开销** | 几乎为零 | 每个 VM 独占内存/CPU |
| **镜像大小** | MB~几百 MB | GB 级 |
| **文件系统** | 分层 UnionFS | 完整磁盘镜像 |
| **跨平台** | 必须同内核架构 | 可在不同 OS 上运行 |

> **Docker 本质：进程隔离 + 资源限制 + 分层文件系统**
>
> 不是虚拟化，而是内核特性用户态化。

---

## 常见面试追问

### Q: 容器里执行 top 为什么看到的是宿主机进程？

因为默认没有隔离 `/proc` 文件系统。需要 `--pid=host` 才看得到外面。

Docker 的做法：用 `lxcfs` 或自定义 `/proc` 挂载来伪造 `/proc` 视图。

### Q: 为什么容器里不能 systemctl 启动服务？

因为容器里没有 PID 1 的 init 系统（systemd）。PID 1 是容器主进程（比如 bash 或 nginx）。

### Q: --privileged 是什么意思？

去掉所有 Namespace 隔离，容器能看到所有宿主机设备、内核参数。**安全风险极大**，相当于给了宿主机 root 权限。

### Q: 容器进程是直接在宿主机内核上跑的吗？

**是的。** 容器里的所有系统调用（`open()`、`read()`、`write()`）都由宿主机内核处理。

这就是为什么 Linux 容器不能在 Windows 上原生运行（需要 WSL2 的 Linux 内核）。

---

## 实战：手动搭一个「容器」

下面用 Linux 命令模拟 Docker 的底层操作（理解原理用，别在生产跑）：

```bash
#!/bin/bash
# ===== 手动模拟容器的底层操作 =====

# 1. 创建隔离的根文件系统
mkdir -p /mycontainer/rootfs
docker export $(docker create busybox) | tar -C /mycontainer/rootfs -xf-

# 2. 准备 overlay2 分层
mkdir -p /mycontainer/{lower,upper,work,merged}
mount -t overlay overlay \
  -o lowerdir=/mycontainer/rootfs,upperdir=/mycontainer/upper,workdir=/mycontainer/work \
  /mycontainer/merged

# 3. 创建新的 Namespace 并 chroot
unshare --mount --pid --net --uts --ipc --fork \
  chroot /mycontainer/merged /bin/sh

# 4. 在另一个终端限制资源
echo 50000 > /sys/fs/cgroup/cpu/mycontainer/cpu.cfs_quota_us
echo 268435456 > /sys/fs/cgroup/memory/mycontainer/memory.limit_in_bytes
```

> 上面就是 Docker 底层干的事（当然 Docker 要复杂和严谨得多）。

---

## 推荐学习资源

| 资源 | 类型 | 适合 |
|:----|:-----|:----|
| 《自己动手写 Docker》 | 书 | 彻底理解容器原理 |
| Docker 官方文档 | 文档 | 查询最佳实践 |
| `man namespaces` / `man cgroups` | Linux man 手册 | 权威参考 |
| Kernel 源码 `kernel/fork.c` | 源码 | 硬核原理党 |
| 深入理解 Linux 网络 | 书 | 网络 Namespace + veth |

---

---

## ⚡ Docker 优化实战

掌握了底层原理，优化就有依据了。下面从 **镜像**、**构建**、**运行时**、**网络**、**存储** 五个维度讲优化。

### 1️⃣ 镜像优化 — 缩小体积

**镜像大的本质**：每一层都包含了不必要的文件（缓存、日志、临时文件）。

#### ① 多阶段构建 (Multi-stage Build)

```dockerfile
# ❌ 坏做法：编译工具留在最终镜像里
FROM golang:1.21 AS builder
WORKDIR /app
COPY . .
RUN go build -o myapp .

# 最后镜像 = golang 完整环境 + 编译产物 = 800MB+


# ✅ 好做法：多阶段构建
FROM golang:1.21 AS builder
WORKDIR /app
COPY . .
RUN go build -o myapp .

FROM alpine:3.19
RUN apk add --no-cache ca-certificates
COPY --from=builder /app/myapp /usr/local/bin/myapp
CMD ["myapp"]

# 最终镜像 = alpine + 二进制 = 15MB
```

**核心原理**：第一阶段用完整环境编译，第二阶段只要结果。充分利用 UnionFS 的分层合并能力。

#### ② 基础镜像选型

| 镜像 | 大小（约） | 适用场景 |
|:----|:---------:|:--------|
| `alpine` | 5MB | 编译型语言二进制部署 |
| `slim` (如 `node:slim`) | ~40MB | 解释型语言精简运行 |
| `distroless` (Google) | ~10MB | 极致安全，只有运行环境 |
| `ubuntu:22.04` | ~70MB | 广泛兼容 |
| `golang:1.21`（完整） | ~800MB | ❌ 不要作为最终镜像 |

**原则**：运行时不需要的工具（编译器、包管理器、build-essential）一个都不要留。

#### ③ 层合并 (Layer Squashing)

每一行 `RUN` 就是一层。层数多 → 构建慢、拉取慢。

```dockerfile
# ❌ 坏做法：每步一层
RUN apt-get update
RUN apt-get install -y curl wget git
RUN apt-get clean
RUN rm -rf /var/lib/apt/lists/*
# 共 4 层

# ✅ 好做法：合并 RUN
RUN apt-get update && \
    apt-get install -y curl wget git && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*
# 共 1 层
```

**原理**：每层是一个 overlay2 diff 目录，层越多，容器启动时查找文件的开销越大（逐层查找）。

#### ④ 依赖缓存分离

利用 Docker 构建缓存机制，**不变的部分放前面，变化的部分放后面**：

```dockerfile
# ✅ 先复制依赖描述文件（很少变）
COPY package.json package-lock.json ./
RUN npm ci --only=production

# 再复制源码（经常变）
COPY . .
```

**原理**：Docker 构建时逐层检查缓存，如果一层没变就复用缓存。`COPY . .` 经常变，所以应该尽量往后放。

#### ⑤ .dockerignore

```dockerignore
.git
node_modules
target/
*.log
.env
.gitignore
Dockerfile
docker-compose*.yml
```

**为什么重要**：`COPY . .` 会先把所有文件传给 Docker daemon（build context），不加 ignore 会把 `.git`（可能有几百 MB）也传过去。

---

### 2️⃣ 构建优化 — 更快的 CI/CD

#### ① 构建缓存命中

```dockerfile
# 依赖锁定文件先复制 → 安装 → 再复制源码
# 这样只要 package.json 不变，npm ci 这层永远走缓存
COPY package.json yarn.lock ./
RUN yarn install --frozen-lockfile
COPY . .
RUN yarn build
```

#### ② BuildKit — 新一代构建引擎

```bash
# 启用 BuildKit（Docker 23.0+ 默认已启用）
export DOCKER_BUILDKIT=1

# BuildKit 带来的优化：
# - 并发构建（并行解析 Dockerfile 指令）
# - 只推送有变化的层
# - --cache-from 远程缓存复用
# - 秘密挂载（--secret，不用把密钥写入镜像）
```

#### ③ 远程缓存

在 CI/CD 中可以利用远程缓存加速构建：

```bash
# Pull 远程缓存 → 构建 → Push 新缓存
docker buildx build \
  --cache-from=type=registry,ref=myregistry/myapp:cache \
  --cache-to=type=registry,ref=myregistry/myapp:cache,mode=max \
  -t myapp:latest .
```

#### ④ 并发与层数平衡

合并 RUN 可以减少层数，但过度合并也影响 BuildKit 的并行能力：

```dockerfile
# 适度拆分，让 BuildKit 能并行处理
RUN apt-get update && apt-get install -y build-essential  # 1层
COPY third-party-libs ./libs                               # 可以跟1并行
RUN make -j$(nproc)                                        # 可以跟1并行
```

---

### 3️⃣ 运行时优化 — 更稳的性能

#### ① 资源限制（永远别忘！）

```bash
# 不设限制 = 一个容器可以打满宿主机的所有资源

docker run -d \
  --memory=512m \            # 硬限制，超了直接 OOM Kill
  --memory-reservation=256m \# 软限制，宿主机压力大时优先回收
  --cpus=0.5 \               # 最多用半个核
  --pids-limit=100 \         # 限制进程数，防 fork bomb
  nginx
```

**底层原理**：限制最终会写入到容器的 cgroup 文件。

#### ② 日志管理

```bash
# 默认 json-file 驱动，不限制大小会把磁盘写爆
docker run -d \
  --log-driver=json-file \
  --log-opt max-size=10m \   # 每个日志文件最大 10MB
  --log-opt max-file=3 \     # 最多保留 3 个轮转文件
  nginx
```

或在 daemon.json 全局配置：

```json
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  }
}
```

#### ③ 重启策略

```bash
docker run -d --restart=unless-stopped nginx
# on-failure    → 非正常退出时重启（可指定最大重试次数）
# always        → 任何退出都重启
# unless-stopped → 除非手动停止，否则一直尝试
```

#### ④ healthcheck

```dockerfile
HEALTHCHECK --interval=30s --timeout=5s --retries=3 \
  CMD curl -f http://localhost:8080/health || exit 1
```

K8s 和编排工具会检查健康状态做自动恢复。没有 healthcheck 的容器，挂了只能在进程层面发现。

---

### 4️⃣ 网络优化 — 更低的延迟

#### ① 网络模式选择

| 模式 | 性能 | 隔离 | 场景 |
|:----|:----:|:----:|:-----|
| `bridge`（默认） | ⭐⭐⭐ | ✅ | 单机容器互访 |
| `host` | ⭐⭐⭐⭐⭐ | ❌ | 极致性能（直接宿主网络栈） |
| `macvlan` | ⭐⭐⭐⭐ | ✅ | 容器需独立 MAC 地址的场景 |
| `overlay` | ⭐⭐ | ✅ | 跨宿主机容器通信（Swarm/K8s） |
| `none` | — | — | 离线沙箱、不需要网络 |

**host 网络原理**：不创建 Network Namespace，容器直接用宿主网络栈。

#### ② 端口映射不要过多

每一对 `-p` 映射就是一条 iptables DNAT 规则。大量端口映射会在宿主机网络栈引入延迟：

```bash
# ❌ 100 个端口 = 100 条 iptables 规则
# ✅ 只映射需要的端口
```

#### ③ 自定义 bridge 网络

```bash
# 默认 docker0 桥接是所有容器共享的
# 不同业务用不同网络隔离：
docker network create --subnet=172.20.0.0/16 app-net
docker run --network=app-net --ip=172.20.0.10 myapp
```

自定义网络提供**内置 DNS 解析**（容器名解析），比 `--link` 方案好得多。

---

### 5️⃣ 存储优化 — 更快的 I/O

#### ① 存储驱动选择

```bash
docker info | grep "Storage Driver"
```

| 驱动 | 性能 | 推荐 |
|:----|:----:|:----:|
| `overlay2` ⭐ | ⭐⭐⭐⭐ | Linux 首选（RHEL 8.4+/Ubuntu） |
| `fuse-overlayfs` | ⭐⭐⭐ | 用户态，rootless 容器 |
| `devicemapper` | ⭐⭐ | ❌ 已淘汰，性能差 |
| `btrfs` | ⭐⭐⭐⭐ | Btrfs 文件系统上的原生方案 |

**overlay2 性能瓶颈**：频繁写大量小文件时，overlay2 的 copy-up 操作（从下层复制到上层再改）有一定开销。

#### ② 数据卷 vs 容器层

```bash
# ✅ 数据卷：直接挂载宿主机存储，绕过 UnionFS
#    性能好、持久化、可共享
docker run -v /data/logs:/app/logs nginx

# ❌ 写入容器可写层：有 overlay copy-up 开销
#    容器销毁 = 数据丢失
```

**底层区别**：
- 数据卷：直接 `mount --bind` 宿主机目录到容器，没有 overlay
- 容器层：每次写文件 → overlay copy-up（复制到上层再修改）→ 有小文件写密集场景显著变慢

#### ③ tmpfs 挂载

```bash
# 日志/缓存挂到内存，既快又不会撑爆磁盘
docker run --mount type=tmpfs,destination=/app/cache,tmpfs-size=100M nginx
```

**原理**：不走 overlay，直接挂载内存文件系统（`mount -t tmpfs`）。适合 redis、缓存、临时文件。

#### ④ 镜像 GC

```bash
# 定期清理无用的镜像和容器

docker system prune -af          # 全清（警告！会删所有停止的容器）
docker image prune -a --filter "until=24h"  # 只删超过24h的镜像
docker volume prune              # 清理无主数据卷（注意数据备份！）
```

**注意**：`docker system prune -af` 在 CI 环境中很常用，但**不要在本地开发环境随意执行**，会删掉所有未运行的容器和未使用的镜像。

---

### 6️⃣ 安全加固

```bash
# 禁止特权提升（重要！）
docker run --security-opt=no-new-privileges ...

# 只读根文件系统
docker run --read-only --tmpfs /tmp ...

# 丢弃所有 Capability，再按需添加
docker run --cap-drop=ALL --cap-add=NET_BIND_SERVICE ...

# 禁止容器内部挂载
docker run --security-opt=seccomp=default.json ...

# User Namespace 映射（容器root ≠ 宿主机root）
docker run --userns-remap=default ...
```

---

### 7️⃣ 监控与诊断

```bash
# 实时查看资源消耗
docker stats

# 查看容器配置（cgroup、namespace 信息）
docker inspect <container-id>

# 查看日志
docker logs --tail=100 -f <container-id>

# 进入容器查看进程
docker exec -it <container-id> ps aux

# 查看容器 cgroup 实际限制
cat /sys/fs/cgroup/memory/docker/<container-id>/memory.limit_in_bytes

# 查看容器 PID（宿主机视角）
docker inspect -f '{{.State.Pid}}' <container-id>
ls /proc/<pid>/ns/  # 看这个进程的所有 Namespace
```

---

### 📊 优化效果对标

```
镜像大小    1.2GB → 18MB   (多阶段构建 + alpine)
构建时间    5min → 45s     (缓存命中 + BuildKit)
启动时间    3s → 0.3s      (小镜像 + 精简 init)
内存占用    1GB → 128MB    (资源限制 + 基础镜像优化)
磁盘 I/O    高延迟 → 低延迟  (数据卷 + tmpfs)
```

**优化核心原则**：理解底层原理后，每个优化手段都有明确的理论依据，不是靠试出来的。

---

## 一句话总结

> **Docker 不虚拟化硬件，它虚拟化操作系统接口**
>
> 底层就三板斧：**Namespace（隔离） + Cgroup（限制） + OverlayFS（分层）**
> 优化就一句话：**理解底层，对症下药**

## 📂 关联文件

- 长鑫存储-Agent开发 (可在容器中做 Agent 沙箱)：`../job/长鑫存储-Agent开发/长鑫存储-Agent开发工程师.md`
- 智元机器人-DevOps (K8s 依赖容器)：`../job/智元机器人-DevOps平台工程师/智元机器人-DevOps平台工程师.md`
- 奇瑞汽车-CICD (容器化构建环境)：`../job/奇瑞汽车-CICD工程师/奇瑞汽车-CICD工程师.md`
- 比亚迪-Pipeline (K8s Pod = 容器)：`../job/比亚迪-Pipeline开发工程师/比亚迪-Pipeline开发工程师.md`

---

> 🦐 虾管家 · 2026-07-06
