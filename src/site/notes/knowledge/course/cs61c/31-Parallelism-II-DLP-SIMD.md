---
{"dg-publish":true,"permalink":"/knowledge/course/cs61c/31-Parallelism-II-DLP-SIMD/","title":"CS61C L31 · Parallelism II — 数据级并行与 SIMD","tags":["CS61C","SIMD","DLP","AVX","SSE","intrinsics","vector","数据级并行","课程笔记"],"dg-note-properties":{"title":"CS61C L31 · Parallelism II — 数据级并行与 SIMD","tags":["CS61C","SIMD","DLP","AVX","SSE","intrinsics","vector","数据级并行","课程笔记"],"date":"2026-11-16","source":"https://cs61c.org/fa26/"}}
---


# CS61C FA26 · L31 Parallelism II: Data-Level Parallelism, SIMD

> UC Berkeley · Great Ideas in Computer Architecture
> 授课：Head TA Cassie Jeng · 课件 30 · Parallelism II: DLP, SIMD（48 页）

---

## 一、开头复习：Amdahl 与四个象限

### 1.1 ⭐ Amdahl's Law（复习）

```
                1
Speedup = ─────────────────────
          (1 − F) + F / S
```

**例子**：程序 **1/3** 的时间可加速 **2 倍** → 整体加速?

```
Speedup = 1 / ( (1 − 1/3) + (1/3)/2 )
        = 1 / ( 2/3 + 1/6 )
        = 1 / 0.8333
        ≈ 1.2×
```

> ⭐⭐ **「并行的性能收益，被「必须串行执行的软件比例」封死。」**

### 1.2 Great Idea #4：串行尾巴永远存在

> **「合理假设：即使并行化，总有一部分程序必须串行跑 —— 例如协调并行工作本身。」**

```
[Sequential] ── [Parallel 工工工工工工] ── [Sequential]
```

### 1.3 课堂练习：128 核要 10 倍

| 前提 | 值 |
|---|---|
| 处理器数 | 128 |
| 并行部分可提速 | 128× |
| 目标总加速 | 10× |
| 求：串行比例 S | ? |

```
10 = 1 / ( S + (1 − S)/128 )
S + (1 − S)/128 = 0.1
127S = 11.8
S ≈ 0.0929 → 约 9%
```

答案 **D. 9%**。

> ⭐ **「并行/硬件不能解决一切。你仍然得挑选「真正能利用你架构」的任务。」**

### 1.4 Flynn 分类（复习）

| | Data = Single | Data = Multiple |
|---|---|---|
| **Instruction = Single** | **SISD** | **SIMD** |
| **Instruction = Multiple** | **MISD** | **MIMD** |

> **「SIMD、MIMD 是今天最常遇到的两种 —— 通常同时存在于同一个系统里。」**

- **SPMD（Single Program, Multiple Data）**：今天最常见的并行编程风格，同一个程序跑在 MIMD 的所有处理器上（后面 TLP 讲）。

### 1.5 ⭐ 操作的时间代价（粗估，slide 12）

| 操作 | 代价 | 备注 |
|---|---|---|
| **I/O 操作** | **极慢（数百万 cycles）** | 文件读写、`printf`、stdout/stdin |
| **内存操作** | 比文件快约 **100×** | RAM 访问；靠 cache 可降到 **3–100 个时钟周期** |
| **分支与跳转** | 额外时钟周期 | control hazard；（可用循环展开缓解） |
| **算术运算** | 最快（理想 **1 cycle**） | 受 data hazard 影响 |

> ⭐⭐ **「内存比算术慢 3–200 倍 —— 记住这个比例，它决定了后面 SIMD 真正的收益来源。」**

---

## 二、⭐⭐ 数据级并行（DLP）

### 2.1 定义

> **「Data-Level Parallelism (DLP)：一个操作作用于多条数据流。」**

典型例子：**逐元素向量乘法**。滤波器里，用系数向量乘数据向量：

```python
y = [c * x for c, x in zip(cs, xs)]
```

```c
for (int i = 0; i < n; i++) {
  y[i] = c[i] * x[i];
}
```

```
   cs:  [c0][c1][c2][c3]
          ×    ×    ×    ×
   xs:  [x0][x1][x2][x3]
          =    =    =    =
    y:  [y0][y1][y2][y3]
```

### 2.2 SIMD 为什么能利用 DLP（slide 15）

> **「SIMD 架构为 DLP 提供性能提升：」**

1. **一条指令为整个操作取指、译码**（省掉 N 次 fetch/decode 的 instruction bandwidth）；
2. **乘法是已知独立的**（不需要在 lane 之间检查 hazard）；
3. **内存访问上也有流水线/并发**（一次 load 拿到连续多个元素）。

### 2.3 应用与实现

| 应用 | 实现 |
|---|---|
| 科学计算（**Matlab, NumPy**） | **x86 Intel Intrinsics** |
| 图形与视频处理（**Photoshop**） | **ARM**（NEON） |
| 大数据 / **Deep learning** | **RISC-V vector extensions**（见 CS152） |
| 游戏 | **显卡（GPU）** |

> **「SIMD 指令通常通过某套 ISA 的扩展来访问，例如 Intel x86。」**

### 2.4 ⭐ 历史第一块 SIMD 扩展：MIT Lincoln Labs TX-2（1957）

原文引述（slide 17）：

> 「应该强调，程序员可以**同时**使用多个 arithmetic elements [AEs]……(18,18) 耦合给出两个完整、独立的 18-bit 算术单元，它们由正在执行的指令**分开但同时**控制；(9,9,9,9) 耦合给出四个 9-bit 算术单元。」

> ⭐ **「SIMD 的思想比 CPU 本身还老 —— 1957 年就有了。」**

---

## 三、⭐⭐ SIMD 工作原理

### 3.1 一句话

> **「取一条指令，做多条指令的工作。」**

```
标量（SISD）：
  a0 ┐         b0 ┐
  a1 │ ×3 遍    b1 │ ×3 遍      →  c0, c1, c2, c3（4 条 add）
  a2 │         b2 │
  a3 ┘         b3 ┘

SIMD：
  ┌────┬────┬────┬────┐        ┌────┬────┬────┬────┐
  │ a0 │ a1 │ a2 │ a3 │        │ b0 │ b1 │ b2 │ b3 │
  └────┴────┴────┴────┘        └────┴────┴────┴────┘
        └────────── + ──────────────┘
  ┌────┬────┬────┬────┐
  │ c0 │ c1 │ c2 │ c3 │    ← 一条 add 搞定
  └────┴────┴────┴────┘
```

要点：

- **源操作数与目标寄存器都足够宽**，能装下多个值（例如四个 64-bit 浮点数）；
- **同一个操作（如 OP = add）作用到寄存器内所有操作数上**。

### 3.2 向量寄存器

```
__m256d（256-bit）
┌────────┬────────┬────────┬────────┐
│ double │ double │ double │ double │
└────────┴────────┴────────┴────────┘
     ← 普通寄存器只能装一个 double →
```

### 3.3 ⭐ Intrinsics

> **「Intrinsics 是提供汇编语言访问能力的 C 函数/过程。用 intrinsics，可以间接地用汇编指令编程。」**

- **SIMD 扩展汇编指令（Intel AVX / SSE）与 intrinsic 一一对应**。
- slide 原话：「不用写汇编了，我们用 intrinsics 在 C 这种更高层语言里写。」

### 3.4 SIMD 操作伪码（256-bit，装四个 double）

| SIMD 伪函数 | 说明 |
|---|---|
| `vector vec_load(double *A)` | 把地址 A 起的四个 double 载入向量 |
| `void vec_store(double *dst, vector src)` | 把 src 存到 dst |
| `vector vec_setnum(double num)` | 造一个每个元素都等于 num 的向量 |
| `vector vec_add(vector A, vector B)` | 逐元素相加 |

### 3.5 ⭐⭐ 课堂判断题（slide 22/24，[for home]）

```c
// SIMD with vectors
vector C = vec_add(A, B);

// SISD（scalar）
double C[] = {0, 0, 0, 0};
for (int i = 0; i < 4; i++) {
  C[i] = A[i] + B[i];
}
```

**哪些关于向量架构的说法为真？（多选）**

| 选项 | 判定 | 理由 |
|---|---|---|
| A. 只有一次指令取指（`vec_add`） | ✅ | 一条指令覆盖四个加法 |
| B. 硬件不必检查 C 各元素之间的 data hazard | ✅ | 四个 lane 天然独立 |
| C. Load/store 更快，因为 A 的元素相邻 | ✅ | 一次大块 load，利用内存带宽 |
| D. 由于省了 instruction bandwidth、hazard 检查、内存带宽，向量架构在功耗/能量上优于标量 | ✅ | 能效优势 |
| E. 以上都不对 | ❌ | —— |

> ⭐ **「A、B、C、D 全选 —— 而这四条恰好就是 SIMD 的四个收益来源。」**

---

## 四、⭐⭐ SIMD 矩阵乘（dgemm）

### 4.1 标量版回顾（slide 26）

用 k 迭代构造 C 的元素：

- A 第 i 行的「列」（column-major 下连续的 N 个）；
- B 第 j 列的各「行」。

```
      j                                j
k ┌───┐       ┌───────┐          ┌───────┐
i │ A │   x   │   B   │    =     │   C   │
  └───┘       └───────┘          └───────┘
（C 的一个元素 = 长度 N 的点积）
```

```c
double Cij = 0;              // i = 0, j = 0
for (int k = 0; k < N; k++) {
  Cij += A[i+k*N] * B[k+j*N];
}
```

### 4.2 一次算四个 i（slide 27–30）

思路：**固定 j，一次算 C 的 4 个连续 i**（因为 column-major 下 C[i..i+3][j] 在内存里连续）。

六步顺序：

| 步 | 计算对象 |
|---|---|
| 1/6 | **i = 0, j = 0**（4 个元素） |
| 2/6 | **i = 4, j = 0** |
| 3/6 | **i = 0, j = 1** |
| 4/6 | **i = 4, j = 1** |
| 5/6 | 内层循环体（伪码） |
| 6/6 | 完整函数 |

```
A（column-major）
      j=0
     ┌──────────┐
i=0  │ a  a  a  a │  ← vec_load 一次取 4 个连续 double
i=4  │ a  a  a  a │
     └──────────┘
              B[k+j*N] 是个标量 → vec_setnum 广播成 4 份
                    │
     ┌──────────────▼───────────────┐
     │ b  b  b  b  （同一个值 ×4）  │
     └──────────────────────────────┘
```

### 4.3 ⭐⭐ 内层循环体（slide 31）

```c
// 256-bit vector registers capable      // reference
// of holding four 64-bit doubles        // scalar code
Cij = vec_add(                            // Cij += \
         Cij,                             //     A[i+k*N] * B[k+j*N]
         vec_mul(                         //     or equivalently
              vec_load(A+i+k*N),          //     *(A+i+k*N) * (B[k+j*N]))
              vec_setnum(B[k+j*N])));
```

> ⭐ **「四件事：load 一列 A → 广播一个标量 B → 逐元素乘 → 累加到 Cij。」**

### 4.4 完整函数（slide 32，P&H Figure 2.43）

```c
 1   // 256-bit vector registers capable        // reference
 2   // of holding four 64-bit doubles          // scalar code
 3   void dgemm_vector(int N, double *A,
 4                    double *B, double *C) {
 5     for (int i = 0; i < N; i+= 4) {          // for (int i = 0; i < N; i++)
 6       for (int j = 0; j < N; j++) {
 7         vector Cij = vec_setnum(0);          // double Cij = 0
 8         for (int k = 0; k < N; k++) {
 9           Cij = vec_add(                     // Cij += \
10                    Cij,                      //      A[i+k*N] * B[k+j*N]
11                    vec_mul(                  //      or equivalently
12                         vec_load(A+i+k*N),   //      *(A+i+k*N) * (B[k+j*N]))
13                         vec_setnum(B[k+j*N])));
14         }
15         vec_store(C+i+j*N, Cij);       // C[i+j*N] = Cij
16       }
17     }
18   }
```

改动只有三处：`i += 4`；`Cij` 变成 `vector`；内层用 `vec_load/vec_setnum/vec_mul/vec_add`。

---

## 五、⭐⭐ Intel Intrinsics（AVX）

### 5.1 AVX 版 dgemm（slide 34）

```c
 1   // AVX with 256-bit wide registers
 2   // operate on 4 doubles in parallel
 3   void dgemm_avx(int N, double *A,
 4                    double *B, double *C) {
 5     for (int i = 0; i < N; i+= 4) {
 6       for (int j = 0; j < N; j++) {
 7         __m256d Cij = {0, 0, 0, 0};
 8         for (int k = 0; k < N; k++) {
 9           Cij = _mm256_add_pd(
10                     Cij,
11                     _mm256_mul_pd(
12                          _mm256_load_pd(A+i+k*N),
13                          _mm256_broadcast_sd(B+k+j*N)));
14         }
15         _mm256_store_pd(C+i+j*N, Cij);
16       }
17     }
18   }
```

**与向量伪码一一对应：**

| 伪码 | AVX intrinsic |
|---|---|
| `vector` | `__m256d` |
| `vec_add` | `_mm256_add_pd` |
| `vec_mul` | `_mm256_mul_pd` |
| `vec_load` | `_mm256_load_pd` |
| `vec_setnum` | `_mm256_broadcast_sd` |
| `vec_store` | `_mm256_store_pd` |

> ⭐ **「读懂一个，就读懂另一个 —— 只是名字长。」**

### 5.2 ⭐⭐ 基准数据（slide 35）

| N | Python GFLOPs | C (scalar) | **C + AVX** |
|---|---|---|---|
| 32 | 0.0054 | 1.30 | **4.56** |
| 160 | 0.0055 | 1.30 | **5.47** |
| 480 | 0.0054 | 1.32 | **5.27** |
| 960 | 0.0053 | 0.91 | **3.64** |

**结论：SIMD 比标量快约 4 倍。**

> ⭐ **「……但仍远低于这台 CPU 的峰值 25 GFLOPs —— 为什么？」**

可能原因（slide 原列）：

- **Cache**（N=960 时矩阵放不下，掉到 3.64）；
- **Pipeline hazards**；
- 详见 CS61C Fall 2021 Lecture 21。

```
峰值 25 GFLOPs ───────────────────────────────────
AVX  实测 ~5.5 GFLOPs  ██████░░░░░░░░░░░░░  约 20%
```

### 5.3 Intrinsics 定位

> **「Intrinsics 是往汇编里塞 C 函数的一种方式，包括 SSE 指令；与 AVX 指令一一对应。」**

> ⭐ **「本课用 Intel AVX intrinsics 在 C 里写 SIMD，而不是直接写汇编。」**

### 5.4 数据类型与寄存器

**C 类型同时指定了寄存器宽度和数据类型：**

| 类型 | 宽度 | 装什么 |
|---|---|---|
| `__m128` | 128-bit | float |
| `__m128d` | 128-bit | double |
| `__m128i` | 128-bit | 整数 |
| `__m256` | 256-bit | float |
| `__m256d` | 256-bit | double |
| `__m256i` | 256-bit | 32-bit 整数 |

（slide 46 明确列出：`__m256`/`__m256d`/`__m256i` 与 `__m128`/`__m128d`/`__m128i`。）

> ⭐ **「x86 对 float、double、整数用的是不同寄存器集合。」**

### 5.5 ⭐⭐ 命名格式

```
_mm<register width>_<instruction>_<data type>
```

例：

```c
__m256i _mm256_add_epi32 (__m256i a, __m256i b);
        │       │      └─ 32-bit integer
        │       └──────── add
        └──────────────── 256-bit 寄存器

    int │ int │ int │ int │ int │ int │ int │ int   （8 个 32-bit）
```

```c
__m128 _mm_load_ps (float const* mem_addr);
       │      └─ load packed single-precision
       └──────── 128-bit 寄存器

    float │ float │ float │ float    （4 个 32-bit float）
```

> **「从内存 load 128 bit（4 个 packed single-precision 32-bit float）到 dst。`mem_addr` 必须按 16-byte 边界对齐。」**

也可以写成 `_<register size>_<instruction>_<component_type>` 的形式。

### 5.6 ⭐ 伪码 ↔ intrinsic 对照表（slide 39）

| SIMD 伪码 | Intel intrinsic (SSE / AVX) | 说明 |
|---|---|---|
| `vec_load(int32_t *A)` | `__m128i _mm_loadu_si128(__m128i *p)` | 从地址 A load 四个 int 到向量 |
| `vec_store(int32_t *dst, vector src)` | `void _mm_storeu_si128(__m128i *p, __m128i a)` | 把 src 存到 dst |
| `vec_setnum(int32_t num)` | n/a（无直接对应） | 每个元素都等于 num |
| `vec_setnum(0)` | `__m128i _mm_setzero_si128()` | 全零向量 |
| `vec_setnum(*p)` | `__m256d _mm256_broadcast_sd(double const *p)` | 从内存广播一个 double 到所有元素 |
| `vec_add(A, B)` | `__m128i _mm_add_epi32(__m128i a, __m128i b)` | 逐元素相加 |

### 5.7 ⭐ Intel x86 SIMD 演进（slides 40–42）

| 扩展 | 年份 / 平台 | 宽度 |
|---|---|---|
| **MMX**（MultiMedia eXtension） | 1997 Pentium II 起 | 64-bit |
| **SSE**（Streaming SIMD Extension） | 1999 Pentium III 起 | **128-bit** |
| **AVX**（Advanced Vector eXtension） | 2011 Sandy Bridge 起 | **256-bit** |
| **AVX-512** | 2016 Xeon Phi 起 | **512-bit** |
| **AMX**（Advanced Matrix eXtension） | 2023 Xeon 起 | 矩阵块 |

要点：

- 命名随功能变化：**每隔几年新指令、更宽的寄存器、更多并行度**；
- **Intel 架构总体向后兼容**；
- **更宽的位扩展会与大版本寄存器重叠**以保持兼容：

```
AVX-512 寄存器 ┌────────────────────────────────┐
AVX     寄存器 ├───────────────┬────────────────┤
SSE     寄存器 ├───────┬───────┤                │
              128 bit   256 bit   512 bit
```

slide 41 标注 hive 机器支持 **SSE / AVX**（CS 61C hive machines）。

---

## 六、⭐⭐ SIMD 指令的收益与局限

### 6.1 收益从哪来（slide 44）

> **「与其一次对一个数做数学，我们可以一次对好几个数做，在一个时钟周期内。」**

- 称为 **SIMD instructions** 或 **vector instructions**；
- 用专门的 **vector registers**，**128 / 256 / 512 bit**；
- 作为基础 ISA 的扩展 —— 不同系统支持不同 SIMD 指令集。

> ⭐⭐ **「一般来说，大部分加速不是来自「一次做四次数学」，而是来自「一次做一大块内存 load/store」。」**
> 回忆：**内存操作比算术操作慢 3–200 倍。**

```
标量： load a0 │ load b0 │ add │ load a1 │ load b1 │ add …   （内存占主导）
SIMD： load [a0..a3] │ load [b0..b3] │ add×1              （load 次数 ÷4）
```

### 6.2 ⭐ 局限（slide 45，逐条）

| 局限 | 说明 |
|---|---|
| **只能用它自带的那套指令** | 每条指令需要自己的电路，所以只能使用 CPU 附带的指令集；**RISC-V 没有标准向量库**，所以课上用 x86 的向量操作（实际影响不大，算术语法和 RISC-V 很像） |
| **仍只有一个 PC** | **无法向量化 branch / jump 指令**；Lab 里讨论了用类似 `slt` 的办法绕过去 |
| **指令集有限** | 不能对向量的不同分量做不同数学运算；且只能方便地 load **连续块**的内存 |
| **大向量电路贵** | 实现成本高，cycles/instruction 常**高于**普通指令 |

> ⭐ **「一个 PC 决定了 SIMD 只能是「整齐划一」的并行 —— 分支、非连续访问、混合运算都是它的软肋。」**

### 6.3 ⭐⭐ 常见错误（slide 48）

| # | 错误 | 正确做法 |
|---|---|---|
| 1 | **想直接访问 SIMD 向量里的 32-bit 片段**（比如强行 typecast） | 寄存器不同于内存，必须做**显式 load/store** |
| 2 | **用未对齐地址调用 `_mm_load` / `_mm_store`** | 用 `loadu` / `storeu`（较慢），或让地址对齐；malloc 用 **`aligned_alloc`**；局部变量用 attribute（slide 有示例） |
| 3 | **忘了尾块（tail case）** | 若数据长度不是向量宽的倍数，最后几个元素要**逐个**处理 |
| 4 | **用了太多向量 / 开了一个大向量数组** | 编译器会反复把 SIMD 向量 load/store 到 stack，**严重拖慢**代码 |

```c
// 对齐示例（示意）
double A[1024] __attribute__((aligned(32)));   // 32 字节对齐
double *p = aligned_alloc(32, N * sizeof(double));
```

> ⭐⭐ **「四条错误里，两条是「寄存器 ≠ 内存」，一条是「长度不等于 4 的倍数」，一条是「别拿寄存器当数组用」。」**

---

## 七、术语速查

| 术语 | 中文 | 说明 |
|---|---|---|
| **DLP** | 数据级并行 | 一个操作作用于多条数据 |
| **SIMD** | 单指令多数据 | 一条指令处理多个元素 |
| **MIMD** | 多指令多数据 | 多核、WSC |
| **SISD / MISD** | 单指令单数据 / 多指令单数据 | Flynn 另外两象限 |
| **SPMD** | 单程序多数据 | 常见并行编程风格 |
| **vector register** | 向量寄存器 | 128 / 256 / 512 bit |
| **vector instruction** | 向量指令 | SIMD 指令的别称 |
| **intrinsics** | 内建函数 | C 函数，与汇编一一对应 |
| **AVX** | 高级向量扩展 | 256-bit |
| **AVX-512 / AMX** | —— | 512-bit / 矩阵扩展 |
| **SSE** | 流式 SIMD 扩展 | 128-bit |
| **MMX** | 多媒体扩展 | 1997，64-bit |
| **NEON** | ARM 的 SIMD | ARM 平台实现 |
| **`__m256d`** | 256-bit double 寄存器 | 装 4 个 double |
| **`_mm256_add_pd`** | 256-bit 双精度加 | packed double add |
| **`_mm256_broadcast_sd`** | 广播标量 double | 对应 `vec_setnum` |
| **`_mm_load_ps` / `loadu`** | packed 对齐 / 非对齐 load | `u` = unaligned |
| **alignment** | 对齐 | 16 / 32 字节边界 |
| **tail case** | 尾块处理 | 长度非倍数时的余数 |
| **loop unrolling** | 循环展开 | 缓解分支 / 控制 hazard |
| **GFLOPs** | 每秒十亿浮点运算 | 性能指标 |
| **Amdahl's Law** | 阿姆达尔定律 | 串行部分是天花板 |

---

## 相关

- 课程导航：[[knowledge/course/cs61c/index\|CS61C 目录导航]]
- 上一讲：[[knowledge/course/cs61c/30-Parallelism-I-Intro\|L30 Parallelism I — Introduction]]
