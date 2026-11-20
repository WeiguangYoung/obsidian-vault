---
{"dg-publish":true,"permalink":"/knowledge/course/cs61c/34-Parallelism-V-MIMD/","title":"CS61C L34 · Parallelism V — 多核架构与缓存一致性","tags":["CS61C","multicore","硬件线程","SMT","cache coherence","snooping","MSI","课程笔记"],"dg-note-properties":{"title":"CS61C L34 · Parallelism V — 多核架构与缓存一致性","tags":["CS61C","multicore","硬件线程","SMT","cache coherence","snooping","MSI","课程笔记"],"date":"2026-11-16","source":"https://cs61c.org/fa26/"}}
---


# CS61C FA26 · L34 Parallelism V: MIMD Architectures

> UC Berkeley · Great Ideas in Computer Architecture
> 授课：Dan Garcia · 课件 34-Parallelism V MIMD Architectures（37 页）

---

## 一、并行计算机的谱系

### 1.1 从仓库级到多核

课件用一张图把「并行计算机」按规模铺开：

```
warehouse-scale computers   ← 数据中心级别（数千台机器）
        ↑
    multicore               ← 单芯片多核（本讲主题）
        ↑
      （示例：Apple M14）
```

> **「multicore processor 在单个集成电路（single integrated circuit）里包含多个处理器，每个叫一个 core。」**

### 1.2 多核的执行模型

> **「Execution Model」**

| 编号 | 模型 | 细节 |
|---|---|---|
| (1) | 每个处理器执行**独立的指令流** | 单独拥有高层缓存（L1、L2） |
| (2) | 所有处理器访问**同一份共享内存** | 共享 DRAM，可能共享 L3 cache；通过 store/load 同一地址通信 |

**典型片上布局：**

```
┌──────────────────────────────────────────┐
│                Processor chip            │
│  ┌────────┐  ┌────────┐      ┌────────┐  │
│  │ Core 0 │  │ Core 1 │ ...  │ Core N │  │
│  │ L1$    │  │ L1$    │      │ L1$    │  │  ← 私有
│  │ L2$    │  │ L2$    │      │ L2$    │  │  ← 私有
│  └────────┘  └────────┘      └────────┘  │
│        └──────────┬──────────────┘       │
│              Shared L3$                  │  ← 共享
│        └──────────┬──────────────┘       │
└───────────────────┼──────────────────────┘
                    │  Memory bus
              ┌─────┴─────┐
              │   DRAM    │
              └───────────┘
```

⭐ **「L1/L2 私有，L3 共享，DRAM 共享」—— 分层缓存把成本摊开，也让一致性成为问题。**

### 1.3 多核的两种使用场景

| 场景 | 目标 | 例子 |
|---|---|---|
| **Parallel-processing program** | 缩短**单个程序**的运行时间 | 多线程程序（multithreaded program） |
| **Process-level parallelism**（job-level） | 为互相独立的作业提供高 **throughput** | 操作系统 + 多个程序并发运行 |

> 课件注明：process-level parallelism **不在 CS61C 范围（not covered）**。

### 1.4 ⭐ 三个关键设计问题（Key Design Questions）

| # | 问题 | 课件的回答 |
|---|---|---|
| 1 | 支持多少个 core？ | 看目标工作负载；多数系统 = 约束内最好的单核；功耗敏感（手机）= 最好的核 + 最省电的核混合 |
| 2 | 处理器之间如何协调/通信？ | 通过内存中的共享变量 + load/store；用 **synchronization primitives**（如 **locks**）保证一次只有一个处理器访问 |
| 3 | 处理器之间如何共享数据？ | 通过 **shared-memory multiprocessor（SMP）** |

---

## 二、硬件线程 vs 软件线程

### 2.1 复习：什么是线程 [review]

> **「Thread = 单一指令流（a single stream of instructions）。」**

每个线程拥有：

| 私有 | 共享 |
|---|---|
| 自己的 registers（含 stack pointer） | heap |
| 自己的 **program counter（PC）** | global variables |
| 自己的栈 | 其他线程的内存 |

> 同一进程内的线程可以 **concurrent**（并发，逻辑上同时推进），有时还能 **parallel**（真正同时执行，需要多个硬件线程）。

### 2.2 ⭐⭐ 硬件线程 与 软件线程

```
软件线程（OS 层面）              硬件线程（core 层面）
[SW T0] [SW T1] [SW T2] ...  →  [HW T0] [HW T1]  ← core 提供的执行槽位
                                      ↓
                             只有映射上去的才真正在跑
```

> **「每个 core 提供一个（或多个）hardware thread，负责真正执行指令。OS 把多个 software thread **multiplex（复用）**到可用的 hardware thread 上。」**

- **multiplex** = 把多条通道合并到一条共享通道（如 MUX）。
- 只有被映射到硬件线程上的软件线程在运行，**其余全部在等待**。
- OS 通过 **context switch** 制造「很多线程同时在跑」的假象。

### 2.3 OS 的 context switch

| 阶段 | 动作 |
|---|---|
| 为何切换 | 线程阻塞（等网络/等用户输入）；且线程不应「察觉」自己被暂停 |
| 移走旧线程 | 中断执行（interrupt）→ 把 registers（含 PC）存回内存 |
| 载入新线程 | 从内存装回 registers（含 PC）→ 把控制权交给新线程 |

### 2.4 为什么要在硬件里加线程支持

> 课件原文：处理器资源昂贵，不能闲置。

- cache miss 的 **memory latency** 很高（**~100 cycles**）。
- 而且 **thread context switch 的开销必须远小于 cache miss 延迟**，否则切换还不如等着。
- 硬件做法：**加冗余硬件**（多个 PC、多套 registers 放在同一个 core 里），这样切换时不必真的「save context」。
- 理由：**Moore's Law** —— 晶体管很便宜。

---

## 三、硬件多线程（Hardware Multithreading）

### 3.1 ⭐ SMT（Simultaneous Multithreading）

> **「HW multithreading：在同一个 processor core 里同时保持多个线程『active』，方法是存储多个线程状态（PC、registers）。」**

- 控制逻辑（control logic）决定下一条发射哪条指令；
- 可以混着从**不同线程**里挑指令；
- 对软件而言看起来像 **两个处理器**（hardware thread 0 / hardware thread 1）；
- Intel 的商品名：**Hyper-Threading**。

⭐ **多线程发生在单个 core 内部，不等同于多核。**

### 3.2 实例：Intel i7-8700T（多核 + SMT）

```
$ lscpu
CPU(s):                 12          ← "12 logical CPUs"
Vendor ID:              GenuineIntel
Model name:             Intel(R) Core(TM) i7-8700T CPU @ 2.40GHz
Thread(s) per core:     2
Core(s) per socket:     6           ← "6 physical CPUs"
Socket(s):              1
```

> **6 个物理 core × 每核 2 个线程 = 12 个逻辑 CPU。**

### 3.3 实例：2023 MacBook Pro（无 SMT）

```
% sysctl -a | grep cpu:
hw.ncpu:                     12
hw.activecpu:                12
hw.perflevel1.physicalcpu:    4    ← 1 = Efficient E-cores
hw.perflevel1.logicalcpu:     4
hw.perflevel0.physicalcpu:    8    ← 0 = Performance P-cores
hw.perflevel0.logicalcpu:     8
hw.physicalcpu:              12    ← "12 physical CPUs"
hw.logicalcpu:               12    ← "12 logical CPUs"
```

> ⭐ **Apple 硅：P-core 与 E-core 混合，每核只有 1 个线程（no SMT），物理数 = 逻辑数。**

### 3.4 ⭐ 多线程 vs 多核：性价比对比

| 维度 | **Multithreading（SMT）** | **Multicore** |
|---|---|---|
| 硬件增量 | ~1% more hardware | ~50% more hardware |
| 性能提升 | ~1.10X | ~2X |
| 共享什么 | integer ALU、floating point units、**所有 caches**、memory controller | 外层缓存（如 **L3$**）、memory controller |
| 靠什么提高利用率 | 缩短 OS context switch、掩盖 major stall（如 I-cache miss）的延迟 | 在不同处理器上**并行**执行指令 |
| 性质 | 复制「执行上下文」 | 复制「整个处理器」 |

> 🌟 结论：两者**互补**，现代机器两者都做。

---

## 四、SMP 与缓存一致性问题

### 4.1 SMP 的结构

> **「Shared-memory multiprocessor（SMP）：给多个 core/CPU 提供单一、共享、一致（coherent）的内存。」**

- 也叫 **symmetric multiprocessor**（缩写相同）。
- 课件吐槽：更准确的名字应是 **shared-address multiprocessor**，因为所有处理器共享**同一个物理地址空间**（这一点下一讲 virtual memory 会展开）。

```
   CPU 1        CPU 2    ...   CPU N
     │            │              │
   Cache        Cache          Cache      ← 每个 CPU 私有缓存
     │            │              │
     └────────────┴──────────────┘
                  │
                 Bus                      ← 广播介质
             ┌────┴────┐
          Memory       I/O
```

### 4.2 为什么单处理器也需要缓存

> 内存本身就是性能瓶颈（**memory bottleneck**）—— 缓存用来降低对主存的 **bandwidth** 需求。

但多核之后，缓存带来新麻烦：**同一地址在多个私有缓存里各有一份副本**。

### 4.3 ⭐⭐ 问题现场：Cache (In)Coherence

**初始条件**：`Mem[0x5000] = 20`。

| 步骤 | 动作 | CPU 1 cache | CPU 2 cache | Memory |
|---|---|---|---|---|
| 1 | CPU 1 读 0x5000 | 20 | — | 20 |
| 2 | CPU 2 读 0x5000 | 20 | 20 | 20 |
| 3 | CPU 1 写 40 | **40** | **20（stale!）** | 20 |

```
        CPU 1              CPU 2
      Cache: 40          Cache: 20   ← 过时数据，自己并不知道
          │                  │
          └──────── Bus ─────┘
                 Memory: 20
```

> **「CPU 1 写完，CPU 1 的缓存是最新的，但 CPU 2 的缓存已经 stale，而它自己不知道。」**

⭐ 这引入了新的 miss 类型：**coherence miss（也称 communication miss）** —— 不是容量不够，而是**别的处理器写了共享数据**导致的。

### 4.4 解决方向

> **「CPU 1 的写会使其他 CPU 缓存中的副本失效（invalidate）。」**

---

## 五、Cache Coherence Protocol（[out of scope]）

> ⚠ 课件把整节标为 **[out of scope]**，即**不在 CS61C 考试/作业范围内**。但它是理解「为什么需要一致性」的完整解释，下面按 slide 讲清概念。

### 5.1 总思路：Snooping

> **「任一处​​理器 miss 或写时，通过 bus 通知其他处理器。」**

| 情况 | 处理 |
|---|---|
| 只读 | 多个处理器可以同时持有副本（Shared） |
| 写 | **invalidate（失效）其他所有副本** |

**Snooping 协议**：每个 cache controller「偷听」公共 bus 上的写事务。

```
时间 →
Bus:  ─── [Rd 0x5000] ─── [Rd 0x5000] ─── [Wr 0x5000] ───
CPU1$:    S                 S                 M
CPU2$:    –                 S              I (snoop 到写 → 失效自己)
Mem:     20                20               20 (暂为旧值)
```

⭐ **bus 是广播介质（broadcast medium）—— 一次事务所有控制器都能看到，这是 snooping 可行的硬件前提。**

### 5.2 设计者必须回答的四个问题

1. 一个处理器改变了共享数据的状态时，如何通知别人？
2. 是否每个处理器动作都会让数据改变状态？
3. 谁负责提供最新的数据？
4. 这期间内存里的数据是什么状态？

### 5.3 ⭐⭐ MSI 三态协议

每个 cache block 跟踪一个状态：

| 状态 | 含义 |
|---|---|
| **Modified（M）** | 数据最新，已被改（dirty）；**没有其他缓存有副本**；可直接写；**内存是旧值**（需 write back） |
| **Shared（S）** | 数据最新；**其他缓存可能也有副本** |
| **Invalid（I）** | 不在缓存中（即原有的 **valid flag = 0**） |

**状态迁移图（MSI，单一 cache 视角，PrRd = 处理器读，PrWr = 处理器写，BusRd = 总线读请求，BusRdX = 总线写/独占请求）：**

```
                    PrRd (miss)              PrWr (miss)
          ┌────────────────────┐      ┌────────────────────────┐
          ▼                    │      ▼                        │
    ┌───────────┐  BusRd snoop │ ┌───────────┐  PrWr         ┌───────┐
    │  Invalid  │◀─────────────┴─│  Shared   │──────────────▶│Modif. │
    └───────────┘  (别人写)       └───────────┘  (upgrade)    └───────┘
          │                             ▲                        │
          │ PrWr (miss) → BusRdX        │ BusRd snoop            │ PrRd hit
          └─────────────────────────────┼────(flush,降级 S)──────┘
                                        │
                                   Shared 也可由 Modified
                                   被 snoop 到 BusRd 时降级
```

**用文字版看迁移：**

| 当前状态 | 事件 | 总线动作 | 下一状态 |
|---|---|---|---|
| I | PrRd | BusRd | S（或有他人供数） |
| I | PrWr | BusRdX | M |
| S | PrRd | — | S |
| **S** | **PrWr** | **BusRdX（让其他副本失效）** | **M** |
| M | PrRd / PrWr | — | M |
| **S / M** | **snoop 到 BusRdX** | 交给对方数据 | **I** |
| **M** | **snoop 到 BusRd** | **回写/转发数据** | **S** |

> ⭐ 两条关键迁移：**S→M 需要先原子地「独占」（invalidate 所有 sharer）**；**M 被别人读时必须先降级**。

### 5.4 可选优化：MESI / MOESI

| 状态 | 含义 | 好处 |
|---|---|---|
| **Exclusive（E）** | 最新，无其他副本，**可写**，**内存也是最新** | 块被替换时无需写回内存；read miss 时可直接供数 |
| **Owner（O）** | 最新，其他缓存**可能有**副本（它们必须是 Shared）；本缓存拥有**独占修改权**，必须广播改动 | 允许 **dirty sharing**：修改过的块可在缓存间流转而不用更新主存；O 状态行必须响应 snoop 请求并供数 |

- **MESI** = Modified / Exclusive / Shared / Invalid。
- **MOESI** = 再加 **Owned**（Berkeley Ownership Protocol 的典型形态）。

> **E→ 减少一次写回；O → 减少一次写主存。两者都是纯性能优化，不改变正确性。**

### 5.5 课堂 True/False（连答案）

| 命题 | 答案 | 理由（课件） |
|---|---|---|
| 1. 用 write-through 缓存就不需要 cache coherence 了 | **FALSE** | 你有一份副本，我 write-through 到内存，**你读的时候怎么拿到新值？** |
| 2. 每条 store 指令都必须检查其他缓存的内容 | **FALSE** | 这正是协议存在的意义 —— 靠状态就知道要不要做额外动作 |
| 3. 同一时刻只有一个处理器能缓存某个内存位置 | **FALSE** | 如果全部**都在读**呢？那会非常低效 |

⭐ 三题的共同点：**coherence 解决的是「副本的一致性」，不是「禁止副本」。**

---

## 六、内存一致性模型（consistency）说明

本讲 slide 正文讨论的是 **cache coherence**（缓存层面：**同一地址**的多个副本值是否一致），而非 **memory consistency model**（内存层面：**不同地址**的 load/store 顺序在多处理器下如何被观察到）。

| 概念 | 关注点 | 课件覆盖 |
|---|---|---|
| **Cache coherence** | 同一地址的多个副本 | ✅ 重点（MSI/MOESI） |
| **Memory consistency** | 不同地址的访问顺序 | ❌ 本讲未展开 |

> ⚠ **两讲不要混：coherence 是「值」的问题，consistency 是「顺序」的问题。**

---

## 七、本讲结论（And in Conclusion）

- 多核处理器包含多个独立 core，各自执行自己的指令流，**共享同一物理地址空间**。
- 软件线程 vs 硬件线程：OS 用 **context switch**（保存/恢复 registers 与 PC）把软件线程复用到硬件线程上。
- **硬件多线程（SMT）**：每核多个线程上下文，可以在 stall 期间混发别的线程的指令。
- 多核 vs 多线程：**多线程共享执行资源以掩盖延迟；多核复制处理器以并行执行**。
- **SMP**：多 CPU 共享物理地址空间，私有缓存降低主存流量。
- **Cache coherence**：一个 CPU 写共享数据后，其他 CPU 的副本必须失效或更新。
- **Snooping**：cache controller 监听 bus；**MOESI** 跟踪 Modified/Owned/Exclusive/Shared/Invalid。
- 🌟 **核心takeaway：并行性能不只是「多几个核」——处理器之间必须协调共享数据访问，硬件必须维护缓存一致性。**

---

## 术语速查

| 术语 | 中文 | 说明 |
|---|---|---|
| **multicore** | 多核 | 单芯片内多个 core |
| **core** | 核 | 独立的处理器单元 |
| **MIMD** | 多指令多数据 | 本讲的架构类别 |
| **hardware thread** | 硬件线程 | core 提供的执行槽位 |
| **software thread** | 软件线程 | OS 调度的线程对象 |
| **multiplex** | 多路复用 | OS 把多线程映射到少硬件线程 |
| **context switch** | 上下文切换 | 保存/恢复 registers + PC |
| **SMT** | 同时多线程 | 同核内多线程上下文；Intel 叫 Hyper-Threading |
| **P-core / E-core** | 性能核 / 能效核 | Apple 硅的混合核 |
| **SMP** | 共享内存多处理器 | 共享同一物理地址空间 |
| **bus** | 总线 | 广播介质 |
| **coherence miss** | 一致性缺失 | 被别人写导致的 miss，也叫 communication miss |
| **stale** | 过时 | 缓存里的旧副本 |
| **invalidation** | 失效 | 让别人副本作废 |
| **snooping** | 监听协议 | 控制器监听 bus 事务 |
| **MSI** | 三态协议 | Modified / Shared / Invalid |
| **MESI** | 四态协议 | 加 Exclusive |
| **MOESI** | 五态协议 | 加 Owner，支持 dirty sharing |
| **dirty** | 脏的 | 与内存不一致 |
| **write back** | 写回 | M 态降级时写回内存 |
| **lock** | 锁 | 同步原语，保证互斥访问 |

---

## 相关

- 课程导航：[[knowledge/course/cs61c/index\|CS61C 目录导航]]
- 上一讲：[[knowledge/course/cs61c/29-Caches-IV\|L29 Caches IV]]
- 下一讲：[[knowledge/course/cs61c/36-Virtual-Memory-I\|L36 Virtual Memory I]]
