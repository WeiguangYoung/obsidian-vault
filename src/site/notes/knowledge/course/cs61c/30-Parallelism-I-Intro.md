---
{"dg-publish":true,"permalink":"/knowledge/course/cs61c/30-Parallelism-I-Intro/","title":"CS61C L30 · Parallelism I — 超标量、能耗墙、Flynn 分类与 Amdahl 定律","tags":["CS61C","superscalar","超标量","energy","Dennard","Flynn","Amdahl","dgemm","课程笔记"],"dg-note-properties":{"title":"CS61C L30 · Parallelism I — 超标量、能耗墙、Flynn 分类与 Amdahl 定律","tags":["CS61C","superscalar","超标量","energy","Dennard","Flynn","Amdahl","dgemm","课程笔记"],"date":"2026-11-16","source":"https://cs61c.org/fa26/"}}
---


# CS61C FA26 · L30 Parallelism I: Introduction

> UC Berkeley · Great Ideas in Computer Architecture
> 授课：Dan Garcia · 课件 30 · Parallelism I: Introduction（48 页）

---

## 一、开篇：流水线之后还能怎么快？

上一讲收尾时，我们的 5 级流水线 datapath 已经把 instruction throughput 从「每个时钟一条」往上推了一步，但 hazards（结构、数据、控制）会让流水线被迫 stall，直接拖低吞吐：

```
时钟 →
指令1  IF  ID  EX  MEM  WB
指令2      IF  ID  --  EX  MEM  WB     ← data hazard，插 bubble
指令3          IF  --  ID  EX  MEM  WB
                              ↑
                     1/throughput（每 N 个时钟完成一条）
```

**Hazards 造成的 stall 会负面影响 instruction throughput 与任务性能。** 这一讲要回答的是：既然提频越来越难，CPU 设计者手上还剩哪些牌？

---

## 二、⭐ 三条继续提速的老路

slide 4 列了三条路，各自都有明确的天花板：

| # | 手段 | 收益 | 代价 / 限制 |
|---|---|---|---|
| 1 | **提高时钟频率** | 单位时间执行更多指令 | 受**工艺技术**与**功耗散热**限制 |
| 2 | **加深流水线** | 「重叠」执行，比如 10 级、15 级；每级工作量变少 → 时钟周期更短、单级功耗更低 | **三类 hazard 的可能性全部变大** → 更多 stall → **CPI > 1**（文字抽提丢了 `>`，按上下文还原） |
| 3 | **设计 superscalar 处理器** | 每周期发射多条指令 | 电路复杂度上升；桌面、笔记本、手机里常是「几个超标量核 + 若干简单 5 级流水线核」混搭 |

> ⭐ **「深流水线不是免费的 —— 级数越深，分支预测错、数据相关传播的代价就以「脉冲 × 级数」的方式放大。」**

---

## 三、⭐⭐ Superscalar 处理器（超标量）

### 3.1 Multiple-issue：一个周期发射多条

> **「Multiple-issue：每个时钟周期开始多条指令。」**

结构上做了三件事：

1. **多个执行单元**（多个 ALU、FP 单元、load/store 单元……）并行执行指令；
2. **每个执行单元有自己的流水线**；
3. 指令先进入各自的 **reservation station**（保留站），谁的操作数齐了谁先走。

```
                 ┌──────────────────────────┐
                 │ Instruction fetch & decode│   （一次取/译码多条）
                 └────────────┬─────────────┘
        ┌──────────┬──────────┼──────────┬──────────┐
   ┌────▼───┐ ┌────▼───┐      …     ┌────▼───┐ ┌────▼────┐
   │Reserv. │ │Reserv. │            │Reserv. │ │Reserv.  │
   │station │ │station │            │station │ │station  │
   └────┬───┘ └────┬───┘            └────┬───┘ └────┬────┘
        ▼          ▼                     ▼          ▼
   ┌────────┐ ┌────────┐           ┌────────┐ ┌─────────┐
   │Integer │ │Integer │    …      │ Float  │ │Load/    │
   │ ALU    │ │ ALU    │           │ Point  │ │Store    │
   └────────┘ └────────┘           └────────┘ └─────────┘
```

### 3.2 动态调度与「乱序执行」

> **「Dynamic pipeline scheduling 实现 out-of-order 执行：fetch/decode 单元在硬件里重排指令，以减少 hazard 的影响。」**
>
> **「Commit unit 再按「程序取指顺序」把结果写回。」**

也就是说：**执行可以乱，提交必须有序**。这样「乱序」对程序员完全透明，异常和中断的语义仍然是顺序程序的语义。

### 3.3 ⭐ CPI < 1，IPC > 1

- 对 superscalar 来说 **CPI < 1**，因为 **IPC > 1**（一个周期可以完成多条指令）。
  （原文抽提成 `CPI  1` / `IPC  1`，丢失的比较符按上下文还原为 `<` 与 `>`。）
- CPI 要在各种 benchmark 上测：数据压缩、代码编译、视频解码、网络仿真……

**ARM Cortex-A53 例子（2 GHz，dual-issue）：**

| 指标 | 值 |
|---|---|
| 频率 | 2 GHz |
| dual-issue | 每周期最多 2 条 |
| 峰值吞吐 | **4 BIPS**（billion instructions per second） |
| 峰值 CPI | **0.5** |

```
峰值：4 GIPS ÷ 2 GHz = 2 IPC → CPI = 0.5
实际：dependencies / hazards 把 CPI 拉回 1 以上
```

slide 7 的基准图（P&H 图 4.78）里，横轴是按代价排序的各 benchmark，纵轴 CPI；作者标注了一句很妙的话：「minimum cost flow (NP-hard; see CS170)」—— 把指令调度排序问题类比成 NP-hard 的最小费用流。

> ⭐ **「实际 CPI > 1：指令依赖与流水线 hazard 会吃掉超标量带来的收益。峰值是广告，实测才是真相。」**

---

## 四、⭐⭐ 能耗：为什么提频走不下去了

### 4.1 灯泡类比（slide 9）

| 灯泡 | 功率 | 亮度 |
|---|---|---|
| 白炽灯（incandescent） | **60 W**（大量发热） | 参考亮度 |
| LED | **9.5 W** | 同等亮度 |

> ⭐ **「同等亮度，功率只有约 1/6 —— 大量能量被变成了热，而不是光。」**

**需要大量功率的 CPU 就必须大量散热。** 这就是提频路线的物理天花板。

### 4.2 最小化「每个任务的能量」

> **「Energy Efficiency：优化执行一个程序所需能量，就是最小化每次执行程序消耗的能量。」**

```
                              Energy per task
任务总能量 =（功率）×（执行时间）≈  ( 1/CPI × f ) × ( 指令数 × CPI / f )
                                 ≈  指令数  →  所以真正要压的是每条的能耗
```

单条指令的能量随电容与电压变化：

```
E ∝ C · V²        （充放电部分）
P ∝ C · V² · f    （功率，即单位时间的能量）
```

- **C = capacitance**：取决于硬件工艺，以及处理器特性（比如核数）；
- **V = supply voltage**，例如 1 V。

> **「要提高每任务能效，就要降低电容和电压。」**

### 4.3 Power-constrained systems（slide 12）

功率约束系统的性能式子可以粗略理解为「CPU 时间 × 功率」：

```
Power ≈ C·V²·f ，  Performance ≈ IPC × f
```

两类约束：

1. **功率受限**：同一功率预算下要跑得更快 → 必须提高能效；
2. **能量受限**（电池）：必须提高能效来延长续航。

> ⭐⭐ **「能效是所有计算设备的关键指标 —— 不只是手机，数据中心同样被电费和散热卡死。」**

---

## 五、⭐ Dennard scaling 及其终结

### 5.1 1974：Dennard 的观察

> **「Dennard scaling：1974 年 Robert Dennard 观察到，给定面积硅片的 power density（功率密度）保持不变。」**

逻辑链：

```
缩小晶体管尺寸
   → 电容 C 降低 + 电压 V 降低
   → 功率 ∝ C·V²·f 同比例下降
```

与 Moore's Law 合在一起：

| 定律 | 内容 |
|---|---|
| **Moore's Law** | 芯片晶体管数每 2 年翻倍 |
| **Dennard scaling** | 翻倍可以做到**不改变单位面积的功耗** |
| **合起来 →** | **每焦耳性能每 1.5 年翻一倍** |

### 5.2 CMOS 的能量都花在哪（slide 14）

```
       ┌────┐        CMOS 反相器符号
 ──────┤▽○ ├──────
       └────┘
```

| 去向 | 占比 |
|---|---|
| **Leakage（漏电）** | **30%** |
| **Charging capacitors（给电容充放电）** | **70%** |

> ⭐ **「即使在『不工作』的时候，也有 30% 的能量从漏电里漏掉了。」**

### 5.3 终结：power wall

近几年产业界**不再怎么降低供电电压**了，原因：

- 继续降压会让 **power leakage** 变大 —— 晶体管不能完全关断（更像「调光开关」而不是「开/关开关」）；
- 于是形成 **power wall**（功耗墙），并且越来越严重；
- 同时晶体管尺寸、电容也不再像以前那样代际缩小 —— 需要走向 **3D 工艺**。

### 5.4 Power density 曲线（slide 17，S. Borkar / Intel，circa 2000 的预测）

```
功率密度 (W/cm²)
10000 ┤                                    ✕ Sun's surface
      │                              ╱     ✕ Rocket nozzle
 1000 ┤                        ╱╱  ✕ Nuclear reactor
      │                   ╱╱
  100 ┤             ╱╱
      │        ╱                       ┌─ Core 2（现实被压平）
   10 ┤    ╱                       ● P6 ● Pentium
      │ 4004 8086 286 386 486         ↑Hot plate
    1 ┤ 8008 8080 8085
      └───┬────┬────┬────┬────┬────┬────
        1970 1980 1990 2000 2010  年
```

如果按老趋势继续，芯片表面功率密度会追上**太阳表面**。现实是曲线在 2005 年前后被「压平」了 —— 这就是 slide 标题里的 **「2005 Sea Change」**（海变 / 时代转折）。

配套的 slide 18–21 是 Moore's Law 相关曲线、处理器性能趋势，以及 **Apple Silicon** 的新闻配图（The Eclectic Light Company, 2024）：单线程性能不再疯涨，而多核、大缓存、能效比成为主战场。

---

## 六、⭐⭐ Flynn's Taxonomy（Flynn 分类，1966）

### 6.1 为什么必须并行（slide 23）

- **CPU 时钟频率不再增长**：更先进的散热对多数应用太贵/不现实，电费也受不了；
- **并行处理是唯一的出路**；
- 类比：**最高空速受经济性限制，那就用更多、更大的飞机提高 throughput**（……以及更小的座位）。

本模块聚焦：**如何让一个程序/任务跑得更快？**

### 6.2 四象限

用两个维度切分：**Instruction Streams**（单/多）× **Data Streams**（单/多）：

| | Data Stream = Single | Data Stream = Multiple |
|---|---|---|
| **Instruction = Single** | **SISD** | **SIMD** |
| **Instruction = Multiple** | **MISD** | **MIMD** |

> ⭐ **「SIMD 与 MIMD 是今天最常见两种并行 —— 而且通常同时存在于同一个系统里。」**

逐个看：

| 类别 | 定义 | 例子 | 备注 |
|---|---|---|---|
| **SISD** | 指令流与数据流都不利用并行 | 传统单处理器；**我们的 RISC-V 处理器**；**superscalar**（因为编程模型是顺序的） | 「SISD 就是 CS61C 到目前为止做的一切」 |
| **SIMD** | 一条指令流作用于多条数据流，适合天然可并行的运算 | **Intel SIMD 指令扩展**；**NVIDIA GPU** | 读作「sim-dee」，下一讲细讲 |
| **MISD** | 多条指令流作用于同一数据流 | 今天**没有**；历史上某些 array processors | 应用极少，CS61C 不覆盖 |
| **MIMD** | 多个自主处理器同时执行不同指令、处理不同数据 | **多核**；** Warehouse Scale Computers** | 读作「mim-dee」，下一讲之后讲（TLP） |

### 6.3 「Single Program, Multiple Data」

> **「SPMD：今天最常见的并行编程风格 —— 同一个程序跑在 MIMD 的所有处理器上。」**

slide 25 用「New-school Machine Structures」回顾了五层并行在课程里的位置：

| 层级 | 例子 | 状态 |
|---|---|---|
| Parallel **Requests** | 搜索 "Cats" | 未讲 |
| Parallel **Threads** | 分配到各核：查询、广告 | 未讲 |
| Parallel **Instructions** | 5 条流水线指令 | ✅ 已讲（但一次一条） |
| Parallel **Data** | 4 对 word 相加 | 未讲（→ L31） |
| **Hardware descriptions** | 所有门同时工作 | ✅ 已讲 |

---

## 七、⭐⭐ Amdahl's Law（「令人心碎定律」）

### 7.1 公式

> **「收益递减定律的定量版本。」**

```
                1
Speedup = ─────────────────────
          (1 − F) + F / S
```

- **F** = 可被加速部分所占比例（parallel portion）；
- **S** = 该部分的加速倍数（enhancement）；
- **(1 − F)** = 只能串行的部分（serial portion）。

等价写法（用在 N 核上，S = N）：

```
Speedup(N) = 1 / [ (1 − F) + F / N ]
```

### 7.2 时间轴直觉（slide 34）

```
串行部分（不变）
┌──────────┬────────────────────────────────────────┐
│  Serial  │   Parallel（用 N 个核切成 N 份）        │
└──────────┴────────────────────────────────────────┘
   1 个核    1   2   3   4   5   ...   N
             ↑只有这一段被压缩，前面那段纹丝不动
```

> ⭐⭐⭐ **「并行的性能收益，被「必须串行执行的软件比例」封死。」**

### 7.3 三个例子，全部要会算

| 情形 | 计算 | 结果 |
|---|---|---|
| 程序 1/3 的时间可加速 **2 倍** | `1 / (1/3 + (1/3)/2) = 1 / (1/3 + 1/6)` | **1.2×** |
| 某增强加快 **20 倍**，但只有 **25%** 时间能用 | `1 / (0.75 + 0.25/20)` | **1.31×** |
| 同上，只有 **15%** 时间能用 | `1 / (0.85 + 0.15/20)` | **1.17×** |

### 7.4 ⭐⭐ 100 核想线性加速 → 必须零串行

> **「Amdahl's Law 告诉我们：要用 100 个处理器达到线性加速，原始计算里不能有任何一部分是串行的。」**

要 **90 倍**加速（不是 100 倍）呢？串行比例必须 **≤ 0.1%**：

```
1 / (0.001 + 0.999/100) = 90.99×
```

### 7.5 Great Idea #4：并行依然有串行尾巴（slide 36）

> **「合理假设：即使并行化，总有一部分程序必须串行跑 —— 例如协调并行工作本身。」**

```
[Sequential] ── [Parallel 工工工工工工] ── [Sequential]
                  ↑ 协调（同步、归约、分发任务）
```

### 7.6 ⭐ 课堂练习：128 核要 10 倍

> **「假设有 128 个处理器，能把程序的并行部分提速 128 倍。想要从该并行架构上拿到 10 倍加速，程序里最多有多少比例 S 是串行的？」**
> A. 50%  B. 15%  C. 10%  **D. 9%**  E. 其他

解：

```
10 = 1 / ( S + (1 − S)/128 )
S + (1 − S)/128 = 0.1
128S + 1 − S = 12.8
127S = 11.8
S ≈ 0.0929  →  约 9%
```

> ⭐⭐ **「并行/硬件不能解决一切。你仍然得挑选「真正能利用你架构」的任务。」**

---

## 八、单线程优化回顾：dgemm（slide 39–41）

`dgemm` 的单线程优化三件套（本讲只列提纲，细节在往届课）：

| 技术 | 作用 |
|---|---|
| **Loop unrolling** | 减少循环开销与分支 |
| **Variable caching** | 把热变量留在寄存器里 |
| **Cache blocking** | 分块，让工作集塞进 cache |

参考：Fall 2021 Lecture 21、Spring 2024 Lecture 25/26、Spring 2025 Lecture 17。
slide 41 是「换个挡」的梗图（手动挡换挡）—— 意思是**接下来换赛道：从优化单线程转向并行**。

---

## 九、⭐⭐ 矩阵乘法：性能基准问题

### 9.1 为什么选 dgemm

> **「矩阵乘法是工程、数据与图像处理任务中的基本操作。」**

- 图像滤波、降噪、机器学习……；
- 许多紧密相关的操作；
- `dgemm` = **D**ouble-precision **GE**neral **M**atrix **M**ultiplication；
- 原作在 **FORTRAN**（**按列存储，column-major**）；
- 神经网络每一层都是矩阵-矩阵 / 矩阵-向量乘。

### 9.2 回顾（见 Caches IV）

```
C[i][j] = Σ_k A[i][k] * B[k][j]
```

```c
int sum = 0;               // sizeof(int) = 4
for (int k = 0; k < size; k++) {
  sum += A[i][k] + B[k][j];   // slide 原样（乘法写成加法是缓存版笔误）
}
C[i][j] = sum;
```

> ⭐ **「C 里的 `A[][]` 是 row-major；后面所有 dgemm 例子都用 column-major（FORTRAN 风格）。」**

column-major 下访问是线性的：

```
C[i][j] ↔ C[i + j*N]
A[i][k] ↔ A[i + k*N]
B[k][j] ↔ B[k + j*N]
```

这是后面向量化能「一次 load 四个连续 double」的前提。

### 9.3 Python 版（slide 45）

```python
def dgemm(N, A, B, C):
  for i in range(N):
    for j in range(N):
      C[i+j*N] = 0
      for k in range(N):
        C[i+j*N] += \
            A[i+k*N] * B[k+j*N]
```

**FLOPs** = Floating Point Operations Per Second。dgemm 的运算量是 **2N³**（N³ 次 fadd + N³ 次 fmul）；用运行时间去除即为 FLOPs。

| N | Python MFLOPs | Python GFLOPs |
|---|---|---|
| 32 | 5.4 | **0.0054** |
| 160 | 5.5 | 0.0055 |
| 480 | 5.4 | 0.0054 |
| 960 | 5.3 | 0.0053 |

**注意：Python 的 GFLOPs 几乎不随 N 变化 —— 瓶颈在解释器，不在缓存。**

### 9.4 C 版（slide 46）

```c
void dgemm_scalar(int N, double *A,
                 double *B, double *C) {
  for (int i = 0; i < N; i++) {
    for (int j = 0; j < N; j++) {
      double Cij = 0;
      for (int k = 0; k < N; k++) {
        Cij += \
            A[i+k*N] * B[k+j*N];
      }
      C[i+j*N] = Cij;
    }
  }
}
```

| N | Python GFLOPs | **C GFLOPs** |
|---|---|---|
| 32 | 0.0054 | **1.30** |
| 160 | 0.0055 | 1.30 |
| 480 | 0.0054 | 1.32 |
| 960 | 0.0053 | **0.91**（超出缓存，掉速） |

> ⭐⭐ **「仅仅换成 C，性能就暴涨约 240 倍（0.0054 → 1.30 GFLOPs）—— 解释 vs 编译、类型声明是关键。」**
> 「我们可以到此为止……但让我们做得更好！！」

### 9.5 ⭐ 计时：`clock()` 的坑（slide 47）

```c
#include <time.h>
int main(...) {
  clock_t start = clock();
  // … task code …
  clock_t end = clock();

  // execution time in seconds
  double delta_time = (double)(end - start)/CLOCKS_PER_SEC;
  …
}
```

slide 列出的注意事项：

- 朴素的 C / Python 用 **row-major**（和 slide 里 dgemm 的 column-major 不一致，要注意数据布局）；
- C 和 Python 都在 matmul 里**堆分配内存**；
- ⭐ **「计时库测的是 execution time（执行时间），不是 runtime（墙钟时间）。」** 在共享机器 / 高负载环境（比如 hive）上差异巨大。

> **「算出 FLOPs 的方法：测运行秒数，再乘以程序里的浮点运算数。」**

---

## 十、⭐ 两个「并行地」观察到的事实（slide 48）

```c
void dgemm_scalar(int N, double *A, double *B, double *C) {
  for (int i = 0; i < N; i++) {
    for (int j = 0; j < N; j++) {
      double Cij = 0;
      for (int k = 0; k < N; k++) {
        Cij += A[i+k*N] * B[k+j*N];
      }
      C[i+j*N] = Cij;
    }
  }
}
```

1. **A 的一行与 B 的一列的 element-wise 乘法可以独立计算** —— 一条 FP 指令就能做点积中的一个「逐元素乘」；
2. **C 的不同元素可以独立计算** —— 可以是分开的 fmul + fadd 序列。

```
C[0][0] │ A 第 0 行 · B 第 0 列  ← 互不相关
C[4][0] │ A 第 4 行 · B 第 0 列      ↓
C[0][1] │ A 第 0 行 · B 第 1 列   这四条可以同时算
C[4][1] │ A 第 4 行 · B 第 1 列
```

> ⭐⭐ **「也许……这就是并行的入口？」** —— 这正是下一讲 SIMD dgemm 的切入点。

---

## 十一、术语速查

| 术语 | 中文 | 说明 |
|---|---|---|
| **superscalar** | 超标量 | 每周期发射多条指令 |
| **multiple-issue** | 多发射 | 超标量核心机制 |
| **dynamic pipeline scheduling** | 动态流水线调度 | 硬件重排指令 |
| **out-of-order execution** | 乱序执行 | 执行乱序、提交有序 |
| **reservation station** | 保留站 | 等待操作数就绪的暂存 |
| **commit unit** | 提交单元 | 按程序顺序写回结果 |
| **CPI** | 每指令周期数 | superscalar 可 < 1 |
| **IPC** | 每周期指令数 | superscalar 可 > 1 |
| **BIPS** | 十亿条指令/秒 | 吞吐单位 |
| **dual-issue** | 双发射 | 每周期最多 2 条 |
| **energy per task** | 每任务能量 | 能效的核心指标 |
| **capacitance C / voltage V** | 电容 / 供电电压 | 能量 ∝ CV² |
| **Dennard scaling** | 登纳德缩放 | 尺寸缩小而功率密度不变 |
| **power wall** | 功耗墙 | 降压实则漏电激增 |
| **leakage** | 漏电 | 约 30% 能量 |
| **power density** | 功率密度 | W/cm² |
| **Flynn's Taxonomy** | Flynn 分类 | SISD/SIMD/MISD/MIMD |
| **SISD / SIMD / MISD / MIMD** | 单/多指令 × 单/多数据 | 四象限 |
| **SPMD** | 单程序多数据 | 常见并行编程风格 |
| **Amdahl's Law** | 阿姆达尔定律 | Speedup = 1/((1−F)+F/S) |
| **serial portion** | 串行部分 | 加速的天花板 |
| **dgemm** | 双精度通用矩阵乘 | 性能基准问题 |
| **column-major** | 列主序 | FORTRAN 的存储方式 |
| **FLOPs** | 每秒浮点运算次数 | dgemm 为 2N³ 次运算 |
| **loop unrolling / cache blocking** | 循环展开 / 缓存分块 | 单线程优化手段 |

---

## 相关

- 课程导航：[[knowledge/course/cs61c/index\|CS61C 目录导航]]
- 上一讲：[[knowledge/course/cs61c/29-Caches-IV\|L29 Caches IV]]
- 下一讲：[[knowledge/course/cs61c/31-Parallelism-II-DLP-SIMD\|L31 Parallelism II — Data-Level Parallelism, SIMD]]
