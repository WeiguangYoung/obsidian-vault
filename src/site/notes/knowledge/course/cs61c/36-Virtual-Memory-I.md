---
{"dg-publish":true,"permalink":"/knowledge/course/cs61c/36-Virtual-Memory-I/","title":"CS61C L36 · Virtual Memory I — 虚拟地址、分页与地址翻译","tags":["CS61C","virtual memory","分页","地址翻译","page table","VPN","PPN","课程笔记"],"dg-note-properties":{"title":"CS61C L36 · Virtual Memory I — 虚拟地址、分页与地址翻译","tags":["CS61C","virtual memory","分页","地址翻译","page table","VPN","PPN","课程笔记"],"date":"2026-11-16","source":"https://cs61c.org/fa26/"}}
---


# CS61C FA26 · L36 Virtual Memory I

> UC Berkeley · Great Ideas in Computer Architecture
> 授课：Dan Garcia · 课件 35-Virtual Memory I（43 页；本笔记记为 L36）

---

## 一、为什么需要 Virtual Memory

### 1.1 ⭐⭐ 三个动机

> **「给每个进程一种『拥有整块地址空间』的错觉。」**

| # | 问题 | 后果 |
|---|---|---|
| 1 | **主存比程序地址空间小** 怎么办？ | 32 位地址空间上限 4 GiB；访问超出 `0x0FFF FFFF` 的地址会 **crash** |
| 2 | **两个程序访问同一地址** 怎么办？ | 若所有进程都能访问任意 32 位地址，彼此可以 **corrupt/crash** |
| 3 | 需要**保护（isolation）** | 进程之间必须互相看不见 |

```
程序 A 眼里的地址空间          实际的 DRAM
┌──────────────┐            ┌──────────────┐
│ 0x00000000   │            │ 只放得下「工作集」│
│    ...       │   ←��射 →   │  (working set)│
│ 0xFFFFFFFF   │            └──────────────┘
└──────────────┘              偶尔换出到 disk
```

### 1.2 Virtual Memory 的定义与收益

> **「Virtual memory 是存储层次里的下一层：给每个进程一个完整、专属的地址空间幻觉；底层只把工作集（working set）的页面放在主存，其余放在 disk。」**

| 收益 | 说明 |
|---|---|
| **Demand paging** | 可以运行比 DRAM 还大的程序 |
| **保护与共享** | OS 可以在进程间共享内存，也能彼此隔离 |
| **隐藏机器差异** | 程序不需要关心实际内存大小 |
| 历史注脚 | 历史上 virtual memory 比 cache 还早出现 |

⭐ **「今天，virtual memory 对保护/隔离的意义已经大于对内存管理的意义。」**

### 1.3 ⚠ 暂时假定没有 cache

> 为了讲清楚翻译流程，本讲**先假设 cache 不存在**，翻译都由软件/硬件翻译器完成。cache 与 **TLB** 的整合放到**下一讲**。

```
Processor chip  ├── (assume this doesn't exist: cache)
DRAM chip       ├── DDR3/4/5, HBM/HBM2/3
SSD / HDD       └── 存储层次的最外层
```

---

## 二、虚拟地址空间与物理地址空间

### 2.1 两个地址空间

```
进程 1: VA 0x0000...0xFFFF           进程 2: VA 0x0000...0xFFFF
        （各自都以为自己拥有全部）           （地址完全冲突，互不干扰）
                    \                      /
                     \                    /
                   ┌────────────────────┐
                   │  Translator /      │   ← OS + 硬件
                   │  Memory Manager    │
                   └─────────┬──────────┘
                             ▼
              Physical Address Space 0x00000000 .. 0x07FF FFFF
              （每个物理地址唯一，只有 OS 知道映射）
```

> **「虚拟地址空间 = 用户程序所知道的一组地址；物理地址空间 = 映射到真实内存位置的一组地址，对用户程序隐藏。」**

### 2.2 谁管理哪一层（[from Caches I] 的延伸）

| 层次 | 管理者 |
|---|---|
| registers ↔ memory | 编译器 / 汇编程序员 |
| cache ↔ main memory | **cache controller 硬件** |
| main memory ↔ disk | **操作系统（virtual memory）**，虚拟→物理翻译由硬件协助（**TLB**） |
| 文件 | 程序员 |

### 2.3 OS 的三项职责

1. 把虚拟地址映射到物理地址；
2. 同时使用 memory 与 disk（disk 更大更慢，用来制造「内存更大」的错觉）；
3. **保护**：进程间隔离。

> 背后仍是 **spatial / temporal locality**：只把「很可能会访问」的数据留在内存里，从 disk 取数要尽量少。

---

## 三、分页（Paged Memory）

### 3.1 为什么是「页」

> **「disk 的一次访问会把整页（page）载入内存。」**

- 页必须**足够大**，才能摊销 disk 的高延迟；
- 现代 OS 典型页大小：**4 KiB**；
- 4 KiB = 2¹² bytes → 页内偏移需要 **12 bits**。

```
虚拟地址空间（paged）
0xF…F  ┌──────────┐
       │  stack   │
       ├──────────┤
       │          │
       ├──────────┤
       │  heap    │
       ├──────────┤
       │  data    │  ← Page Number 1
       ├──────────┤
       │  text    │  ← Page Number 0
0x0…0  └──────────┘
```

### 3.2 ⭐ 地址位划分（核心图）

```
Virtual address (e.g. 32 bits)
┌────────────────────┬──────────────┐
│   VPN (20 bits)    │ offset (12b) │
└────────────────────┴──────────────┘
        │                     │
        │ 页表翻译             │ 原样保留
        ▼                     ▼
┌────────────────────┬──────────────┐
│   PPN (36 bits)    │ offset (12b) │
└────────────────────┴──────────────┘
Physical address (e.g. 48 bits)
```

> **「Virtual 与 physical page 大小相同，所以页内偏移位数完全相同，offset 直接照抄。」**

⭐ 翻译只改「页号」，不动「页内偏移」—— 这是分页的全部魔法。

---

## 四、⭐ 地址翻译流程

### 4.1 Page Table 是什么

> **「page table 把某个进程的虚拟地址翻译成物理地址。」**

| 属性 | 内容 |
|---|---|
| 一个进程一张 | 每个进程拥有自己的 page table |
| 条目对应什么 | **每个 VPN 一条** |
| 条目的内容 | 若页在内存 → 给出 **PPN**；否则 → 标记为 disk，OS 触发 **page fault** |

> ⚠ **「page table 不是 cache！它是查找表（lookup table），不含数据，但它有『状态位 + PPN』。」**
> 若一定要类比 cache：**「no tags; index is VPN」**。

```
Page table（每进程一份，本质是个大数组）
 index(VPN)        status   PPN
 ┌─────────┐     ┌──────┬──────┐
 │ 0x00000 │ ──▶ │  V   │  0   │
 │ 0x06000 │ ──▶ │  V   │  2   │
 │   ...   │     │      │ disk │
 │ 0xFFFFF │ ──▶ │  V   │  1   │
 └─────────┘     └──────┴──────┘
```

**PTE（page table entry）字段（概念层面）：**

| 字段 | 含义 |
|---|---|
| **PPN** | 物理页号（在内存时有效） |
| **valid bit** | 该页是否在内存（0 → 要 page fault） |
| **dirty bit** | 是否被修改过（换出时需写回 disk） |
| **reference / accessed bit** | 是否被访问过（供替换算法参考，如 LRU 近似） |
| 其他（下一讲） | 权限位等保护信息 |

### 4.2 命中路径：页在内存中

```
1. 程序执行 load，给出 virtual address(VA)
        │
2. 提取 VPN ──▶ 查 page table ──▶ 得到 PPN
        │
3. 组 PA = PPN ‖ offset(来自 VA)
        │
4. 用 PA 读 DRAM，把数据送到目标寄存器
```

**示例（课件数值，32 位虚拟地址、4 个 4 KiB 物理页）：**

| 步骤 | 细节 | 结果 |
|---|---|---|
| 1 | `Load byte @ 0xFFFF F004 to t0` | — |
| 2 | VPN = `0xFFFFF`（前 20 位）→ 页表说 PPN = `1` | 页在内存 ✅ |
| 3 | PA = `1 ‖ F004` = **`0x0000 1F004`** | 无 page fault |
| 4 | 读 PA 处的字节 | 返回给 t0 |

### 4.3 ⭐⭐ 未命中路径：page fault

当 VPN 对应的是 `disk` 而不是 PPN：

```
1. load VA
2. 查页表 → 发现该页「在 disk 上」，valid = 0
3. ⚠ Page Fault！
   OS 介入：
     a. 在 DRAM 里找一个空闲物理页（或驱逐一页，dirty 需先写回）
     b. 从 disk 把整页载入（~ a million cycles later）
     c. 更新页表：PPN = 2，valid = 1
4. 用新的 PA 访问内存，把数据返回给程序
```

**课件具体数值对照：**

| VPN | 操作前 PPN | 操作后 PPN |
|---|---|---|
| `0x40000` | disk | disk |
| `0x60000` | disk | **2**（page fault 后分配） |
| `0xFFFFF` | 1 | 1 |

> ⭐ **1 次 page fault ≈ 百万级 cycles —— 这相当于 disk 访问延迟，是存储层次里最贵的 miss。**

---

## 五、课堂练习（Practice）

### 5.1 翻译题

**题**：给定上表的页表，虚拟地址 `0x00003450`、4 KiB 页，翻译出的物理地址是？

- A. `0x00003450`
- B. `0x0000250`
- C. `0x00503450`
- D. `0x0F543450`
- E. `0x0F54450`
- F. Disk/Other

**答案：`0x0F54450`**（课件显示为 **E**）。

**推导**：

```
0x00003450
┌───────────┬──────┐
│ VPN       │ 450  │   ← 低 12 位是 offset
└───────────┴──────┘
offset 部分 = 0x450（保留原样）
VPN 部分 = 0x00003 → 页表查出 PPN = 0x0F5
PA = 0x0F5 ‖ 0x450 = 0x0F54450
```

> ⭐ 所以 E 正确：`0x0F5` + `0x450` = **`0x0F54450`**。干扰项 D `0x0F543450` 是把 PPN 当成 3 位半拼错位置 —— 牢记 **offset 12 位不参与翻译**。

### 5.2 位数计算题 ⭐

**Setup**：32 位机器，**8 GiB RAM**，页大小 **16 KiB**。求：

| # | 问题 | 答案 | 推导 |
|---|---|---|---|
| 1 | **page offset** 位数 | **14 bits** | `log2(16 KiB) = log2(2⁴ × 2¹⁰) = 14` |
| 2 | **VPN** 位数 | **18 bits** | `32 − 14 = 18` |
| 3 | **PPN** 位数 | **19 bits** | `log2(8 GiB) − 14 = log2(2³ × 2³⁰) − 14 = 33 − 14 = 19` |

```
虚拟地址（32 位）:  [ VPN: 18 ][ offset: 14 ]
物理地址（33 位）:  [ PPN: 19 ][ offset: 14 ]   ← 8 GiB = 2^33 B
```

> ⭐ 关键：**offset 位数只由页大小决定；VPN 由虚拟地址宽度决定；PPN 由物理内存大小决定**（注意 PPN 可以和地址宽度无关，因为物理地址 = PPN ‖ offset = 19 + 14 = 33 位 > 32 位虚拟地址宽度）。

### 5.3 页表大小题 ⭐

**追问**：每张 PTE 4 B，这张 page table 有多大（MiB）？

```
2^18 个条目 × 4 B/entry
= 2^18 × 2^2 B
= 2^20 B
= 1 MiB
```

> 答案：**1 MiB**（每 VPN 一条，条目数 = VPN 数）。

### 5.4 二级页表 / 多级页表如何省空间

课件本节（Page Table Details I）只给出「4 MiB 太大、放内存里」这个出发点，**二级页表的具体结构与推导留给后续讲**。这里只记下问题的成因，不展开未讲内容：

| 现象 | 数值 |
|---|---|
| 32 位虚拟地址、**4 KiB 页** → VPN = 20 bits | 2²⁰ 条 PTE |
| 每条约 4 B + 状态位 | 4 MiB |
| 占 4 GiB 内存的比例 | 约 **0.1%** |
| 对 cache 而言 | **太大，放不进 cache** |

> **省空间的基本思路（由上面数字推出）：把「一整张 2²⁰ 条的直线数组」拆成多级 —— 只为**真正用到的**那部分虚拟地址范围分配下级页表，因此稀疏地址空间的进程可以省下大量内存。**

---

## 六、Page Table 的现实问题

### 6.1 存在哪里

> **「page table 存在内存（DRAM）里，不放在 cache 里。」**

因为 4 MiB 远超任何 cache 的容量，塞进去会把数据挤光。

### 6.2 ⭐ 性能代价：两条内存访问

```
load/sw (cache miss 时)
  │
  ├─① 读 page table（在主存里）→ 得到 PPN
  │
  └─② 用 PA 读/写真正的数据页（也在主存里）
```

> ⚠ **「每次 lw/sw 在 cache miss 时会有两次（很慢的）内存访问。」**

**缓解手段（课件列出的两条）：**

| 手段 | 效果 |
|---|---|
| **以 cache block 为单位在 DRAM 与 cache 间传输**（而不是单字） | 摊薄开销 |
| **给常用的页表项做缓存 → TLB（Translation Lookaside Buffer）** | 下一讲 |

### 6.3 TLB 预告

> **「TLB = 地址翻译的高速缓存」，缓存的是 **PTE（VPN → PPN + 状态位）**，不是数据。**

```
        cpu 发出 VA
              │
        ┌─────▼─────┐  hit   ┌────────────────┐
        │    TLB    │───────▶│ 直接用 PPN 拼 PA │
        └─────┬─────┘        └────────────────┘
              │ miss
              ▼
        走 page table（慢路径）→ 回填 TLB
```

> TLB 命中/未命中的完整路径、TLB 与 cache 的交互、**VIPT / PIPT** 等问题，本讲 slide 未展开，留到**下一讲**。

---

## 七、本讲结论（And in Conclusion）

- OS 管理多个进程共享的 CPU、内存、I/O 等资源。
- 每个进程跑在 **virtual memory** 里；OS 通过 **page table** 管理 **virtual ↔ physical** 的翻译。
- 遗留问题（课件列出的 open questions）：
  1. OS 如何做 **context switch**？
  2. 页不在内存里怎么办（page fault 细节）？
  3. **write-back 还是 write-through**？
  4. **如何把 cache 与 virtual memory 结合起来**（→ TLB）？

---

## 术语速查

| 术语 | 中文 | 说明 |
|---|---|---|
| **virtual memory** | 虚拟内存 | 让每个进程以为拥有全部地址空间 |
| **virtual address (VA)** | 虚拟地址 | 程序发出的地址 |
| **physical address (PA)** | 物理地址 | 真正落到 DRAM 的地址 |
| **virtual address space** | 虚拟地址空间 | 每个进程一份，可重复 |
| **physical address space** | 物理地址空间 | 唯一，对程序隐藏 |
| **paging** | 分页 | 以页为单位换入换出 |
| **page** | 页 | 典型 4 KiB |
| **page offset** | 页内偏移 | 翻译时原样保留 |
| **VPN** | 虚拟页号 | 虚拟地址高位 |
| **PPN** | 物理页号 | 由 page table 查得 |
| **page table** | 页表 | 每进程一张的查找表，不是 cache |
| **PTE** | 页表项 | 含 PPN 与状态位 |
| **valid bit** | 有效位 | 页是否在内存 |
| **dirty bit** | 脏位 | 是否被修改过，换出要写回 |
| **reference bit** | 引用位 | 是否被访问，供替换策略 |
| **page fault** | 缺页 | 页在 disk 上，需载入（百万级 cycles） |
| **demand paging** | 按需调页 | 只在实际访问时才载入 |
| **working set** | 工作集 | 当前阶段活跃的页面集合 |
| **MMU / memory manager** | 内存管理单元 | 执行 VA→PA 翻译 |
| **TLB** | 转址旁路缓存 | 缓存 PTE，下一讲 |
| **isolation / protection** | 隔离 / 保护 | 进程间互不可见 |

---

## 相关

- 课程导航：[[knowledge/course/cs61c/index\|CS61C 目录导航]]
- 上一讲：[[knowledge/course/cs61c/34-Parallelism-V-MIMD\|L34 Parallelism V: MIMD]]
- 下一讲：Virtual Memory II（TLB、Cache 与 VM 的整合）
