---
{"dg-publish":true,"permalink":"/knowledge/course/cs61c/37-Virtual-Memory-II/","title":"CS61C L37 · Virtual Memory II — TLB、多级页表与地址翻译实战","tags":["CS61C","TLB","地址翻译","page table","page fault","多级页表","课程笔记"],"dg-note-properties":{"title":"CS61C L37 · Virtual Memory II — TLB、多级页表与地址翻译实战","tags":["CS61C","TLB","地址翻译","page table","page fault","多级页表","课程笔记"],"date":"2026-11-16","source":"https://cs61c.org/fa26/"}}
---


# CS61C FA26 · L37 Virtual Memory II

> UC Berkeley · Great Ideas in Computer Architecture
> 授课：Dan Garcia · 课件 36-Virtual Memory II（43 页）

---

## 一、⭐⭐ 开场 warmup：八道判断题

> 前提：**32-bit virtual address space**、**16 GiB DRAM**、**4 KiB pages**。

先把关键数字列出来，后面每一问都要用：

```
虚拟地址 32 位  →  VA 范围 2^32 = 4 GiB
页面大小 4 KiB  →  offset = log2(4096) = 12 位
物理内存 16 GiB →  PA 需要 log2(16 × 2^30) = 34 位
```

### 1.1 地址位划分图 ⭐⭐

```
32 位虚拟地址 VA
┌────────────────────────────────┬──────────────┐
│            VPN (20 位)         │ offset (12)  │
└────────────────────────────────┴──────────────┘
 bit 31                      bit 12          bit 0

34 位物理地址 PA
┌──────────────────────────────────────┬──────────────┐
│               PPN (22 位)            │ offset (12)  │
└──────────────────────────────────────┴──────────────┘
 bit 33                            bit 12          bit 0
                      ↑
              offset 完全一样 —— 页内偏移不变
```

### 1.2 逐题解答

| # | 命题 | 答案 | 推导 |
|---|---|---|---|
| 1 | 有 2^32 / 2^12 = 2^20 个 VPN | **True** | VPN = 32 − 12 = **20 位**，2^20 = 1 M 个虚拟页 |
| 2 | 有 2^34 / 2^12 = 2^22 个 PPN | **True** | 16 GiB = 2^34 B；PPN = 34 − 12 = **22 位** |
| 3 | 虚拟地址低 12 位是 page offset | **True** | log2(4 KiB) = 12 |
| 4 | Virtual page size = Physical page size = 4 KiB | **True** | 页大小由硬件统一规定，两边必须一致，否则 offset 无法直接复用 |
| 5 | # page table entries = # virtual page numbers | **True** | 单级页表「每虚拟页一项」，2^20 项 |
| 6 | 2^20 项 × 4 B = 4 MiB | **True** | 2^20 × 2^2 = 2^22 B = 4 MiB —— **这就是单级页表的痛点** |
| 7 | Page tables are stored in memory | **True** | 页表在 DRAM 里，所以查页表本身就是一次内存访问 |
| 8 | 无 cache 时每条 lw/sw 需要 2 次内存访问 | **True** | ①读 PTE（page table walk）②读写真实数据 |

> **「注意 2^34 = 16 GiB 这个换算：1 GiB = 2^30 B，所以 16 GiB = 2^4 × 2^30 = 2^34 B。**

📌 抽提文本里 `232 / 212 = 220` 与 `24230 / 212 = 222` 是上标丢失的结果，应为 `2^32 / 2^12 = 2^20`、`2^34 / 2^12 = 2^22`。

---

## 二、⭐ 复习：现代虚拟内存与 demand paging

### 2.1 三大目的

```
Modern VM = 用 address translation 提供「大、私有、统一」的存储幻觉
   ├─ 大     → Demand Paging：程序可以比 DRAM 还大
   ├─ 私有   → Protection：每进程独立地址空间
   └─ 统一   → 掩盖不同机器配置差异
代价         → 每次内存引用都要做 address translation（页表）
```

### 2.2 为什么不能一口气把所有页都装进内存

`hello world` 这类程序，虚拟地址空间里绝大部分页从来不会被访问。若启动时把每页都从 disk 读进 DRAM，纯粹浪费。

> **「Demand Paging：只在该页真正被请求时才把它从 disk 载入内存。程序启动时，页表里每一项都指向 Disk。」**

### 2.3 翻译步骤回顾

1. 程序给出虚拟地址 **VA**
2. OS/硬件翻译成 **PA**：
   - a. 从 VA 中抽出 **VPN**
   - b. 在页表中查出 **PPN**
   - c. 拼出 PA = **PPN ‖ offset**（offset 与虚拟地址完全相同）
3. 若物理页不在内存 → **page fault**，OS 从 disk 载入
4. 按 PA 读内存，取回数据

**第 2 步 = 第 1 次内存访问（page table walk）；第 4 步 = 第 2 次内存访问。**

---

## 三、⭐⭐ 单级页表太贵：从痛点走向多级

### 3.1 单级页表的问题

| 项目 | 数值 |
|---|---|
| 虚拟页数 | 2^20（32 位 VA，4 KiB 页） |
| PTE 大小 | 4 B（PPN + 状态位） |
| 单进程页表 | 2^20 × 4 B = **4 MiB** |
| 进程数 | 每进程一份 |

一个进程光页表就吃掉 4 MiB，几十个进程就是上百 MiB —— 而且其中绝大多数 PTE 的 valid 位都是 0（该页根本没用过）。

> **「多级页表（multi-level page table）的核心思想：只为真正用到的区域分配二级页表。没用到的一整段，在页目录里记一个 invalid 即可。」**

### 3.2 树状结构 ⭐⭐

```
                 一级页表 = Page Directory（页目录）
                 ┌────┬────┬────┬─────┬────┐
                 │[0] │[1] │[2] │ ... │[n] │
                 └─┬──┴─┬──┴─┬──┴─────┴────┘
                   │    │    │
        valid=0 ───┘    │    └─── valid=1 ──┐
      （整段未用，      │                    │
        不分配下级）    │ valid=0             ▼
                       │              二级页表（Second-level PT）
                       │              ┌────┬────┬────┐
                       ▼              │[0] │[1] │... │
                    不分配            └─┬──┴─┬──┴────┘
                                        │    │
                                        ▼    ▼
                                      PPN   PPN
                                     （+valid/dirty/prot 等）
```

- **叶子**：存放 PPN 与状态位
- **内部节点**：只有指向下一级页表的指针 + 一个 valid 位
- 稀疏是常态：`Code` / `Heap` / `Stack` 三段各自只需要一两张二级页表，中间巨大的空洞不占任何空间

### 3.3 地址如何拆成多段索引

32 位 VA 与 12 位 offset 固定，剩下的 20 位 VPN 再切成两段：

```
一级索引 = VPN 高位、二级索引 = VPN 低位
┌───────────────┬────────────────┬──────────────┐
│  Page Dir Idx │  Page Table Idx│  offset (12) │
│   10 位        │     10 位       │              │
└───────────────┴────────────────┴──────────────┘
 bit 31     bit 22          bit 12           bit 0
```

**怎么定每级取几位？** 目标是让每张页表正好放进**一个页**：

```
一个页 4 KiB = 2^12 B；每个 PTE 4 B = 2^2 B
一张页表能装 2^12 / 2^2 = 2^10 = 1024 个 PTE → 索引 10 位
VPN 共 20 位 → 20 / 10 = 2 级，每级 10 位
```

三级则每级 20/3 不整除 → 例如 8/8/4 或 10/8/2 之类的拆法；位数取法看具体设计，原则始终是「一张页表 ≤ 一页」。

### 3.4 翻译过程（每级一次内存访问）

```
VA → [PD idx]  → 查 Page Directory（第 1 次访存）
              → 得到二级页表基址
   → [PT idx]  → 查二级页表（第 2 次访存）
              → 得到 PPN
   → PA = PPN ‖ offset → 读数据（第 3 次访存）
```

> ⭐ **级数越多，page table walk 的内存访问次数越多** —— 这正是下一节 TLB 存在的理由。

---

## 四、⭐ 页表项 PTE 的字段

> **「一个 PTE 至少要有：PPN（若在主存）+ 若干状态位。」** 本节讨论三个状态位。

```
┌──────────┬───────┬───────┬────────┬─────────┐
│   PPN    │ Valid │ Dirty │ Write  │  (其他) │
│          │  bit  │  bit  │ Protect│         │
└──────────┴───────┴───────┴────────┴─────────┘
```

### 4.1 Valid bit —— demand paging 的开关

| 状态 | 含义 | 动作 |
|---|---|---|
| valid = 1 | 页在 DRAM 中 | 直接按进程要求读写 |
| valid = 0 | 页只在 disk 上 | 触发 **page fault** exception |

page fault 的处理流程：

1. OS 介入，为请求页在 DRAM 中分配一个物理页
2. 若 DRAM 已满 → 先选一页**替换**：把被踢出的页写回 disk
3. 从 disk 读入请求页
4. 更新 PTE（新 PPN、valid = 1）
5. 重新执行那条指令

### 4.2 Dirty bit —— 相当于主存的写策略位

> **「主存就是 disk 的 cache。」**

- write-through：写同时写 disk —— 太慢，**不用**
- **write-back：只有被替换时才写回 disk —— 所有 VM 系统都这么做**

dirty bit 记录「这一页有未同步的写」：

| dirty | 替换时 |
|---|---|
| 1 | 把该页写回 disk |
| 0 | 直接丢弃，不写 disk |

### 4.3 Write protection bit —— 安全的共享

- OS 目标：进程间**隔离**内存，同时允许**安全共享**
- 共享手段：多个进程的 VPN 指向**同一个 PPN**（典型例子：`stdlib` 之类的库代码）
- write protection bit = 1 → 该页「受保护」，写它会触发 OS exception
- 常见于程序代码段、系统数据

### 4.4 每进程一份页表

```
Process A ──> Page Table A ──> 物理页 {0, 2, 5}
Process B ──> Page Table B ──> 物理页 {7, 9}

同一 VPN 在不同进程可映射到不同 PPN → 隔离
同一 PPN 出现在两张页表 → 共享
```

OS 负责记录当前哪个进程在跑（切换页表基址寄存器）。

---

## 五、⭐⭐ TLB：Translation Lookaside Buffer

### 5.1 为什么要 TLB

- 理想：地址翻译 **≤ 1 个周期**，且空间省
- 现实：每条指令 / 每次数据访问都要翻译
- page table walk = 访问内存中的页表，单级 1 次、二级 2 次访存

> **「TLB 是 VPN→PPN 映射的 cache。它缓存的是『页表项』，不是程序数据。」**

一句课堂原话式的比喻：

> **「TLB = 我最近见过的那些 VPN/PPN 映射。」**（3–128 条）

```
                ┌─────────── CPU ───────────┐
                │                           │
                │   VPN ──► [  TLB  ] ──► PPN
                │             ↑ miss
                │             └── Page Tables (in DRAM)
                └───────────────────────────┘
       TLB 离 CPU 和 cache 都非常近
```

### 5.2 条目结构

| 字段 | 含义 |
|---|---|
| **Tag / VPN** | 虚拟页号（有的设计带 ASID） |
| **PPN** | 物理页号 |
| **属性位** | valid、dirty、write-protect 等的副本 |

### 5.3 设计参数

| 参数 | 典型值 |
|---|---|
| 条目数 | 3–128 |
| 相联度 | Fully associative（或 2-way） |
| 替换策略 | FIFO、Random（**不是 LRU**） |

### 5.4 TLB Reach ⭐

```
TLB Reach = 条目数 × 页大小
```

例：3 条 × 4 KiB = **12 KiB = 12288 个虚拟地址**可被立即翻译。

> **「TLB hit → 不需要 page table walk，省掉翻译的那次内存访问。」**

---

## 六、⭐⭐ 地址翻译三条路径对比

例子：读 `0x00004ABC`，页大小 `0x1000`，PTE 4 B。先拆地址：

```
0x00004ABC
offset = 0xABC
VPN    = 0x00004
```

### 6.1 路径 A（Best）：TLB Hit

```
VA = 0x00004ABC
  │
  ├── VPN = 0x00004 ──► TLB 查找 ──► ✅ 命中
  │                                  PPN = 0x82C121D
  │
  └── offset = 0xABC （原样保留）
                              │
                              ▼
              PA = 0x82C121DABC ──► 读数据（1 次访存）
```

- 翻译耗时：**≈ 1 个周期**
- 页表：**完全不访问**，disk：**不访问**

### 6.2 路径 B（Worse）：TLB Miss + Page Table Walk

```
VA = 0x00004ABC
  │
  ├── VPN 4 ──► TLB ──► ❌ miss
  │                 │
  │                 ▼  去主存查页表
  │            Page Directory[?] ──► 二级页表 ──►
  │                 PTE: valid=1, PPN=0x82C121D  ✅
  │                 │
  │                 └──► 回填 TLB（VPN 4 → 0x82C121D）
  │
  └── offset = 0xABC
                              │
                              ▼
                 PA = 0x82C121DABC ──► 读数据
```

- page table walk 代价：**≈ 100 周期**
- disk：**不访问**
- 好处：下次同一个 VPN 就变成路径 A

### 6.3 路径 C（Worst）：Page Fault

```
VA = 0x00004ABC
  │
  ├── VPN 4 ──► TLB ──► ❌ miss
  │                 │
  │                 ▼
  │            PTE: valid = 0（页在 disk 上）❌
  │                 │
  │                 ▼  page fault
  │            OS 分配物理页（可能先踢掉一页，dirty 则写回 disk）
  │            disk ──► DRAM 载入该页
  │            更新 PTE：PPN = 0x82C121D, valid = 1
  │            更新 TLB：VPN 4 → 0x82C121D
  │
  └── offset = 0xABC
                              │
                              ▼
                 PA = 0x82C121DABC ──► 读数据
```

- disk 访问代价：**≈ 数千周期**

### 6.4 汇总表

|  | TLB | Page Table Walk | Disk |
|---|---|---|---|
| **Best** | ✅ Hit | 不访问 | 不访问 |
| **Worse** | ❌ Miss | ✅ PTE valid | 不访问 |
| **Worst** | ❌ Miss | ❌ page fault | ✅ 载入页 |

> **「TLB 里有效 ⇒ 页表里一定有效 ⇒ 数据一定在内存里。」** 三级信息是一层层向下保证的。

---

## 七、⭐ TLB 与上下文切换

- **TLB 每核一个**；页表**每进程一份** —— 两者粒度不匹配
- 若不管：进程 A 的 VPN→PPN 条目会被进程 B 误用（同 VPN 不同 PPN）

两种处理方式：

| 方案 | 做法 | 代价 |
|---|---|---|
| **最简单** | OS **flush TLB**，把全部/部分条目标为 invalid | 切换后全部 miss，一段时间的 page table walk |
| **更复杂** | 每条目记录 **ASID**（Address Space ID / 进程标识） | 命中时还要比对 ASID，但可保留条目 |

```
TLB entry: [ ASID | VPN | PPN | attrs ]
            ↑
   与当前进程 ASID 不匹配 → 视为 miss
```

---

## 八、⭐ Caches vs. Virtual Memory（最后，时间允许才讲）

### 8.1 层次中的单位

| 层次 | 单位 | 典型大小 |
|---|---|---|
| Cache | line / block | 64 B 或 128 B |
| Memory | page | 4 KiB |

### 8.2 关键区别：「cache 有数据副本，页表没有」

> **「Data at each level is a quick-access copy of data at a lower level.」** —— cache 与主存是「副本」关系。
>
> **「A page table translates addresses.」** —— 页表**不是**数据的副本，它只记录「页在哪、状态如何」，用来实现 demand paging。

### 8.3 对照表

| | Caches | Virtual Memory |
|---|---|---|
| 层次 | Cache ↔ Memory | Memory ↔ Disk |
| 单位 | Line / Block（≈64 B） | Page（≈4096 B） |
| Miss 名字 | cache miss | **page fault** |
| 相联度 | direct-mapped / N-way / fully | **Fully associative**（页可放内存任意处） |
| 替换策略 | LRU 或 random | LRU（最常见）/ FIFO / random |
| 写策略 | write-through 或 write-back | **只有 write-back** |

### 8.4 PIPT：本课假设的 cache 组织

问题：页不在主存时 cache 还能命中吗？内存引用时先访问谁？

> **「Assume physically indexed, physically tagged (PIPT) caches.」**

```
整条路径：
1. VA ──► TLB ──► PPN        （翻译）
2. PA = PPN ‖ offset
3. PA ──► L1 Cache ──► 数据
```

```
     VA ──►[TLB]──► PA ──►[L1 Cache]──► data
            ↑ miss        ↑ miss
         Page Table     L2 / Memory
```

- **关键提醒：`TIO = PA`，而不是 VA** —— index 与 tag 都取**物理地址**的位
- 若页不在内存，cache 里也不可能有该数据（cache 只是主存的子集）
- 其他更复杂设计（VIVT / VIPT）存在于真实硬件，但本课不展开

---

## 九、本讲主线一句话串起来

```
单级页表 4 MiB/进程 太贵
      ↓ 只为用到的区域分配
多级页表：VA = [PD idx | PT idx | offset]，每级 ≤ 1 页
      ↓ 但每次翻译要多次访存
TLB：VPN→PPN 的 cache，3–128 条，全相联，命中即 1 周期
      ↓ 切换进程要让旧条目失效
flush 全清 / ASID 标记
      ↓ 最后把翻译塞回存储层次
PIPT：先翻译拿 PA，再用 PA 访问 cache
```

---

## 术语速查

| 术语 | 中文 | 说明 |
|---|---|---|
| **VPN** | 虚拟页号 | VA 去掉 offset 的部分，32 位 VA/4 KiB 页时为 20 位 |
| **PPN** | 物理页号 | PA 去掉 offset 的部分，16 GiB/4 KiB 页时为 22 位 |
| **VPO / PPO** | 虚拟/物理页内偏移 | 两者**完全相等**，都是 12 位 |
| **PTE** | 页表项 | 存 PPN + 状态位，通常 4 B |
| **page table walk** | 查页表 | 访问内存中的页表，单级 1 次、二级 2 次访存 |
| **multi-level page table** | 多级页表 | 页目录 + 下级页表，只为用到的区域分配 |
| **page directory** | 页目录 | 多级页表的一级索引表 |
| **valid bit** | 有效位 | 1 = 在 DRAM，0 = 在 disk |
| **dirty bit** | 脏位 | 有未同步写，替换时需写回 disk |
| **write protection bit** | 写保护位 | 1 = 受保护，写触发 exception |
| **demand paging** | 按需分页 | 只在请求时载入页 |
| **page fault** | 缺页 | PTE invalid 时触发，OS 从 disk 载入 |
| **page replacement** | 页替换 | 内存满时踢出一页，dirty 则写回 |
| **TLB** | 转址旁路缓存 | VPN→PPN 映射的 cache |
| **TLB Reach** | TLB 覆盖范围 | 条目数 × 页大小 |
| **fully associative** | 全相联 | 条目可放任意位置 |
| **TLB flush** | 清空 TLB | 上下文切换时使条目失效 |
| **ASID** | 地址空间标识 | 给 TLB 条目打进程标签，避免全清 |
| **context switch** | 上下文切换 | OS 换进程运行 |
| **PIPT** | 物理索引物理标记 | 本课假设：先翻译，再用 PA 访问 cache |
| **write-back** | 写回 | VM 唯一使用的写策略 |

---

## 相关

- 上一讲：[[knowledge/course/cs61c/36-Virtual-Memory-I\|L36 Virtual Memory I]]
- 目录导航：[[knowledge/course/cs61c/index\|CS61C 目录导航]]
