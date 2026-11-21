---
{"dg-publish":true,"permalink":"/knowledge/course/cs61c-labs/Lab07/","title":"CS61C FA26 · Lab 07 · 并行编程 SIMD 与 OpenMP","tags":["CS61C","lab","lab07","SIMD","OpenMP","并行","AVX","性能优化","课程笔记"],"dg-note-properties":{"title":"CS61C FA26 · Lab 07 · 并行编程 SIMD 与 OpenMP","tags":["CS61C","lab","lab07","SIMD","OpenMP","并行","AVX","性能优化","课程笔记"],"date":"2026-11-20","source":"https://cs61c.org/fa26/labs/lab07/"}}
---


# Lab 07 · 并行编程 SIMD 与 OpenMP

> UC Berkeley · Great Ideas in Computer Architecture · Fall 2026
> 代码仓库：`github.com/61c-teach/fa26-lab-starter`
> 本地路径：`/home/yang/repos/fa26-lab-starter/lab07/`

返回 [[knowledge/course/cs61c-labs/index\|🧪 Labs 总览]] · [[knowledge/course/cs61c/index\|📚 CS61C 目录]]

---

## 一、本 Lab 概览

课程覆盖三类并行，本 lab 做前两类：

| 类型 | 缩写 | 并行粒度 | 本 lab | 课程讲次 |
|---|---|---|---|---|
| **数据级并行** | DLP | 一条指令算多个数据（SIMD） | ✅ Exercise 1 | [[knowledge/course/cs61c/31-Parallelism-II-DLP-SIMD\|L31]] |
| **线程级并行** | TLP | 多个线程跑同/不同代码 | ✅ Exercise 2 | [[knowledge/course/cs61c/32-Parallelism-III-TLP-OpenMP\|L32]] |
| 进程级并行 | MLP / MPI | 多进程跨机器 | ❌ 后面 lab | L33+ |

> ⚠️ **必须在 hive 机器上完成**（需 AVX2 + 多核）；本地 CPU 不支持 AVX 会报 `target-specific option mismatch`。
> 拉代码：`git pull origin main`，再 `git pull starter main`。**截止**：Nov 24, 11:59:59 PM PT。

---

## 二、文件清单

> ✅ **已核对**（2026-11-20）：上游 `61c-teach/fa26-lab-starter` 的 `main` 已发布 lab07（commit `878c972 "Fa26 lab07 (#9)"`），本地已 pull 到。以下清单**与仓库实际文件一致**，不再是 FA24 版推测。

| 文件 | 状态 | 说明 |
|---|---|---|
| `ex1.c` | ⬜ 待实现 | `sum` / `sum_unrolled` / `sum_simd` / `sum_simd_unrolled` 四个函数 |
| `ex1.h` | 勿改 | `NUM_ELEMS = (1<<16)+10`（65546）、`OUTER_ITERATIONS = 1<<14` |
| `ex1_test.c` | 勿改 | `main`，测正确性 + 计算加速比 |
| `ex2.c` | ⬜ 待实现 | `dotp_naive` / `dotp_critical` / `dotp_reduction` / `dotp_manual_reduction` |
| `ex2.h` | 勿改 | 含 `<omp.h>`，`ARRAY_SIZE = 5000000`、`REPEAT = 60` |
| `ex2_test.c` | 勿改 | 扫描线程数、测时间、验证结果 |
| `Makefile` | 勿改 | 提供 `make ex1` / `make ex2` / `make openmp_example` |
| `openmp_example.c` | 示例 | Hello World，用于理解 `#pragma omp parallel` |

**Makefile 关键编译选项**（FA24 版）：

```text
CFLAGS_SIMD = -std=c99 -Wall -O0 -c
CFLAGS_OMP  = -std=gnu99 -fopenmp -c -g
LDFLAGS_OMP = -fopenmp
```

> ⭐ `<x86intrin.h>` 已声明全部 SSE/AVX intrinsic，**不加 `-mavx` 也能编译**；但若直接写 `__m256` 或依赖自动向量化，就需要 `-mavx` / `-mavx2` / `-msse4.1`。
> ⭐ `-O0` 是为了**公平比较**（naive 不被自动向量化，否则 SIMD 版不会更快）；`#pragma omp` 需 `-fopenmp`（编译和链接都要）。

---

## 三、Exercise 1 前置：什么是 SIMD 与 DLP

### 3.1 数据级并行（DLP）

一条指令流里，**相邻多个操作彼此独立、操作同类数据**，就存在 DLP：

```c
for (int i = 0; i < n; i++)
    c[i] = a[i] + b[i];   // 第 i 次与第 i+1 次完全无关，可同时做
```

### 3.2 为什么 SIMD 能加速

标量指令一次只处理一个数，4 个元素要 16 条指令：

```text
load a0 │ load b0 │ add │ store c0
load a1 │ load b1 │ add │ store c1
load a2 │ load b2 │ add │ store c2
load a3 │ load b3 │ add │ store c3     ← 16 条指令
```

SIMD（Single Instruction, Multiple Data）把这 4 个数装进一个 128-bit 寄存器，**1 条向量加法**搞定：

```text
load [a0 a1 a2 a3] │ load [b0 b1 b2 b3] │ vadd │ store [c0 c1 c2 c3]
```

> ⭐ **加速来源**：指令条数 ÷4（load/store 次数 ÷4），**但时间不一定 ÷4** —— 这就是实测只有 3–5 倍而非 4 倍整的原因。

### 3.3 Intrinsics 是什么

**Intrinsic** 是「实现由编译器负责」的函数：你写函数调用，编译器直接内联成对应机器指令。比手写汇编可读可移植，比普通 C 能精确指定指令。命名规律 `_mm<位宽>_<指令>_<数据类型>`，如 `_mm_add_epi32` = 128-bit + add + packed 32-bit int；AVX 版前缀 `_mm256_`。

### 3.4 AVX / SSE intrinsics 速查表

| 函数 | 作用 |
|---|---|
| `_mm_setzero_si128()` | 返回 `{0,0,0,0}`，初始化累加器（**每轮循环都要重置**） |
| `_mm_set1_epi32(x)` | 返回 `{x,x,x,x}`，**广播**同一个数到 4 条 lane |
| `_mm_set_epi32(e3,e2,e1,e0)` | 返回 `{e0,e1,e2,e3}`（⚠️ 参数**逆序**） |
| `_mm_loadu_si128((__m128i*)p)` | 从 `p` 读 16 字节（**u = unaligned**，不要求对齐） |
| `_mm_load_si128((__m128i*)p)` | 同上但要求 16 字节对齐，否则**段错误** |
| `_mm_storeu_si128((__m128i*)p, v)` | 把 `v` 写回 `p`（寄存器「落地」到内存） |
| `_mm_add_epi32(a,b)` | 逐元素 `a[i]+b[i]`，32 位，**会回绕** |
| `_mm_cmpgt_epi32(a,b)` | 逐元素 `a[i]>b[i] ? 0xFFFFFFFF : 0`，生成**掩码** |
| `_mm_and_si128(a,b)` | 逐元素按位与，用掩码「筛选」：`x & mask` |
| `_mm_extract_epi32(v,i)` | 取第 `i` 条 lane 的值（SSE4.1，需 `-msse4.1`） |
| `_mm256_*` 系列 | 256-bit 版本，8 个 int32（需 `-mavx` / `-mavx2`） |

> ⭐ 关键认知：`_mm_cmpgt_epi32` 产生的是 **全 1 / 全 0 的掩码**，不是 1 和 0。所以「条件加法」= `and` 再 `add` —— 不满足条件的 lane 被 AND 成 0，加上去等于没加。

### 3.5 编译选项

```bash
gcc -std=c99 -O2 -mavx2 -c ex1.c -o ex1.o
```

| 选项 | 含义 |
|---|---|
| `-O0` | 不优化（本 lab 默认，为了**公平比较**） |
| `-O2` | 开启自动向量化等优化 |
| `-mavx` / `-mavx2` | 允许 AVX（256-bit 浮点/部分整型）/ AVX2（256-bit 整型） |
| `-msse4.1` | 允许 SSE4.1（`_mm_extract_epi32` 等） |

> ⚠️ `-mavx2` 编译出的二进制在只支持 SSE 的机器上会 **SIGILL**。
> ⚠️ **别偷偷改 `-O2`** —— naive 被自动向量化后加速比测试就失效了，autograder 也用自己的参数。

### 3.6 循环展开（Loop Unrolling）

`sum_unrolled()` 把 `sum()` 的内层循环**一次处理 4 个元素**，再用 tail 循环处理余数：

```c
for (unsigned int i = 0; i < NUM_ELEMS / 4 * 4; i += 4) {
    if (vals[i]     >= 128) sum += vals[i];
    if (vals[i + 1] >= 128) sum += vals[i + 1];
    if (vals[i + 2] >= 128) sum += vals[i + 2];
    if (vals[i + 3] >= 128) sum += vals[i + 3];
}
for (unsigned int i = NUM_ELEMS / 4 * 4; i < NUM_ELEMS; i++)   // tail case
    if (vals[i] >= 128) sum += vals[i];
```

**为什么有用**：循环计数与跳转**摊销到 4 个元素**（分支开销 ÷4）；4 条独立加法可**重排并行发射**，减少数据冒险停顿，提升 ILP。

**与 SIMD 的关系**：方向一致但量级不同。

| | 循环展开 | SIMD |
|---|---|---|
| 谁变宽 | **指令条数**变多，每条还是标量 | **每条指令**变宽（4 lane） |
| 依赖的硬件特性 | ILP（超标量、乱序） | 向量单元 |
| 典型收益 | 1.1–1.3× | 3–5× |

> ⭐ `sum_simd_unrolled` = 「**向量化 × 展开**」：一次迭代处理 `4 lane × 4 = 16` 个元素。
> ⭐ **为什么不全展开**：可读性急降；指令变多 → 程序变大 → **I-cache 放不下**；过度展开超过寄存器数与乱序窗口，收益递减。

---

## ⭐ Exercise 1 · Writing SIMD Code（核心）

### 1.1 题目要求

`sum(vals)` 只累加 **≥128 的元素**，外层重复 `OUTER_ITERATIONS` 次以便计时。两件事：

1. **实现 `sum_simd()`** —— 用 SIMD 向量化内层循环；
2. **复制到 `sum_simd_unrolled()` 并展开 4 次** —— 一次处理 16 个元素。

```bash
make ex1 && ./ex1
```

**期望**：naive ≈ 7 s、`sum_simd` ≈ 1–2 s、展开版再略快。`ex1_test.c` 判定：数值全等（`simd == reference`）、`reft > simdt * 2`（≥2×）、`simdt > simdut`（展开版更快）。

### 1.2 ⭐ 为什么 127 这个向量要预先构造

函数骨架已给出 `__m128i _127 = _mm_set1_epi32(127);`，三个理由：

1. **SIMD 里没有「和立即数比较」的指令** —— `_mm_cmpgt_epi32` 只能比较**两个向量**，所以必须先把常数 127 铺成 `{127,127,127,127}`。
2. **它是不变量** —— 4 条 lane 阈值都是 127，在 `OUTER_ITERATIONS = 2^14` 轮里都不变，放循环外构造一次，省掉 `2^14` 次广播。
3. **「≥128」等价于「>127」** —— 只有严格大于的指令可用；数组元素由 `rand() % 256` 生成（0–255），不必担心负数。

### 1.3 ⭐ 掩码（mask）从哪里来

```c
__m128i tmp  = _mm_loadu_si128((__m128i *)(vals + i));  // 读 4 个元素
__m128i mask = _mm_cmpgt_epi32(tmp, _127);              // 逐 lane: tmp[j] > 127 ?
tmp          = _mm_and_si128(tmp, mask);                // 不满足的 lane 清 0
sum_vec      = _mm_add_epi32(sum_vec, tmp);
```

`_mm_cmpgt_epi32` 产生的是全 1 / 全 0，不是 0/1：

```text
tmp   = { 200,  42,  150, 128 }
_127  = { 127, 127, 127, 127 }
mask  = { -1,   0,   -1,   -1 }   // 0xFFFFFFFF / 0x00000000
tmp&mask = { 200, 0, 150, 128 }
```

> ⭐ **这就是「无分支（branchless）条件加法」**：没有 if 就没有分支预测失败，4 条 lane 同步推进；代价是多一次 ADD。

### 1.4 完整实现 `sum_simd`

```c
    __m128i _127 = _mm_set1_epi32(127);   // 骨架已给
    long long int result = 0;

    for (unsigned int w = 0; w < OUTER_ITERATIONS; w++) {
        __m128i sum_vec = _mm_setzero_si128();          // ⚠️ 每轮重置！

        for (unsigned int i = 0; i < NUM_ELEMS / 4 * 4; i += 4) {
            __m128i tmp  = _mm_loadu_si128((__m128i *)(vals + i));
            __m128i mask = _mm_cmpgt_epi32(tmp, _127);
            tmp          = _mm_and_si128(tmp, mask);
            sum_vec      = _mm_add_epi32(sum_vec, tmp);
        }

        // 水平求和：store 到临时数组再相加
        int tmp_arr[4];
        _mm_storeu_si128((__m128i *)tmp_arr, sum_vec);
        long long int sum = tmp_arr[0] + tmp_arr[1] + tmp_arr[2] + tmp_arr[3];

        // tail：处理 NUM_ELEMS % 4 个剩余元素
        for (unsigned int i = NUM_ELEMS / 4 * 4; i < NUM_ELEMS; i++) {
            if (vals[i] >= 128) {
                sum += vals[i];
            }
        }
        result += sum;
    }
    return result;   // 后续 clock 与打印由骨架负责
```

### 1.5 三个设计决策的权衡

**(a) `sum_vec` 放在哪一层循环？**

| 位置 | 结果 |
|---|---|
| 外层循环**外**（只初始化一次） | ❌ 累积 `2^14 × 2^16 × 255`，int32（31 位有效）早溢出成负数 |
| **每轮外层循环内重置** | ✅ **正确做法** |
| 内层循环内每次重置 | ❌ 只剩最后一次迭代的和 |

**(b) store 放在哪一层？**（题目提示：内层循环内不要 store）

- store 在**内层循环里**：每次迭代多写 16 字节 → 比不做 SIMD 还慢；
- store 在**最外层循环之后**：`sum_vec` 累积 `2^14` 轮的和 → **int32 溢出**；
- ✅ **每轮外层循环结束后 store 一次**：`2^14` 次写可忽略，又不会溢出 —— **够稀疏以摊销开销，够频繁以避免溢出。**

**(c) 水平求和（horizontal sum）**

`sum_vec` 里是 4 条并行累加的部分和 `{s0, s1, s2, s3}`，**必须再加一次**：`sum = s0 + s1 + s2 + s3` —— 少这一步，结果只剩总量的 1/4。

两种写法：store 到 `int tmp_arr[4]` 再标量相加（本 lab 推荐，SSE2 即可）；或 `_mm_extract_epi32(sum_vec, 0..3)` 求和（SSE4.1，需 `-msse4.1`）。

### 1.6 `sum_simd_unrolled`：向量化 + 展开 4 次

内层每次处理 16 个元素（4 个向量），tail 从 `NUM_ELEMS / 16 * 16` 开始。

```c
    __m128i _127 = _mm_set1_epi32(127);
    long long int result = 0;

    for (unsigned int w = 0; w < OUTER_ITERATIONS; w++) {
        __m128i sum_vec = _mm_setzero_si128();

        for (unsigned int i = 0; i < NUM_ELEMS / 16 * 16; i += 16) {
            /* 4 段相同的 4 行代码，偏移分别是 +0 / +4 / +8 / +12 */
            __m128i tmp  = _mm_loadu_si128((__m128i *)(vals + i));
            __m128i mask = _mm_cmpgt_epi32(tmp, _127);
            sum_vec      = _mm_add_epi32(sum_vec, _mm_and_si128(tmp, mask));

            tmp  = _mm_loadu_si128((__m128i *)(vals + i + 4));
            mask = _mm_cmpgt_epi32(tmp, _127);
            sum_vec = _mm_add_epi32(sum_vec, _mm_and_si128(tmp, mask));

            tmp  = _mm_loadu_si128((__m128i *)(vals + i + 8));
            mask = _mm_cmpgt_epi32(tmp, _127);
            sum_vec = _mm_add_epi32(sum_vec, _mm_and_si128(tmp, mask));

            tmp  = _mm_loadu_si128((__m128i *)(vals + i + 12));
            mask = _mm_cmpgt_epi32(tmp, _127);
            sum_vec = _mm_add_epi32(sum_vec, _mm_and_si128(tmp, mask));
        }

        int tmp_arr[4];
        _mm_storeu_si128((__m128i *)tmp_arr, sum_vec);
        long long int sum = tmp_arr[0] + tmp_arr[1] + tmp_arr[2] + tmp_arr[3];

        // tail：从 NUM_ELEMS / 16 * 16 开始，逐元素处理
        for (unsigned int i = NUM_ELEMS / 16 * 16; i < NUM_ELEMS; i++) {
            if (vals[i] >= 128) sum += vals[i];
        }
        result += sum;
    }
    return result;
}
```

> ⚠️ 更激进：用 4 个独立累加器 `sum_vec0..3` 各累加一个向量，最后合并 —— 打断 `sum_vec` 的**数据冒险**链，但多占 3 个寄存器、水平求和多做 3 次。本 lab 收益很小。

### 1.7 常见 bug 清单

| # | Bug | 现象 | 修法 |
|---|---|---|---|
| 1 | **求和条件写错**（如 `> 128`、`>= 127`、比较方向反） | 数值对不上 | 必须是 `>= 128`（即 `> 127`） |
| 2 | **tail 忘了条件或起点写错** | 结果偏大 / 越界 | 起点 `NUM_ELEMS / 4 * 4`（展开版 `/16*16`），且 tail 也要 `if (vals[i] >= 128)` |
| 3 | **`sum_vec` 未初始化 / 放到最外层循环外** | 乱数 / 溢出成负数 | `_mm_setzero_si128()`，且每轮 `w` 重置 |
| 4 | **忘了水平求和** | 结果只有 1/4 的量级 | 最后把 4 条 lane 相加 |
| 5 | **store 到 `long long int[4]`** | 数错乱 | 缓冲区必须是 `int[4]`（一次只写 16 字节） |
| 6 | **`_mm_load_si128` 地址未对齐** | 段错误 SIGSEGV | 用 `_mm_loadu_si128`（`int vals[]` 不保证 16 字节对齐） |
| 7 | **完全没有 tail** | `65546 % 4 == 2`，漏 2 个元素 | 补 tail 循环 |

> ⭐ `NUM_ELEMS = 65546`，`% 4 = 2`、`% 16 = 10` —— **tail 一定会触发**，不是「理论上可能」。

### 1.8 调试手段

```bash
gcc -std=c99 -O2 -mavx2 -S ex1.c -o - | grep -E "vpaddd|vpcmpgtd|vpand|vmovdqu"
```

逐 lane 观察：<https://piotte13.github.io/SIMD-Visualiser/#/>；看汇编：<https://godbolt.org/>。

---

## 四、OpenMP 前置知识

### 4.1 OpenMP 是什么

**Open** specification for **M**ulti-**P**rocessing —— C 语言的**编译指示（compiler directive）**框架，不是标准库也不是新语言。`#pragma` = 给编译器的提示（不认识就忽略）；`omp` = 命名空间；`parallel` = 构造类型；`{ ... }` ⚠️ **左花括号必须另起一行**。

```c
#pragma omp parallel
{
    int thread_id = omp_get_thread_num();      // 块内声明 → 每线程私有
    printf("hello world from thread %d\n", thread_id);
}
```

```bash
make openmp_example
OMP_NUM_THREADS=4 ./openmp_example    # 指定线程数
```

> ⭐ 输出顺序**每次都不同**：OpenMP 不保证执行顺序。块内声明 = private，块外声明 = shared（可用 `private()` / `shared()` 覆盖）。

### 4.2 Fork-Join 模型

```text
主线程 ──fork──> [T0 │ T1 │ T2 │ T3] ──join(隐式 barrier)──> 主线程
```

> ⭐ 并行区结束时有个**隐式同步点** —— 所有线程到齐才继续。这就是 `dotp_manual_reduction` 里 `#pragma omp critical` 能安全加在 `#pragma omp for` 之后的原因。

### 4.3 变量作用域：shared vs private

| 位置 | 默认 |
|---|---|
| 并行块**外**声明 | **shared**（共享） |
| 并行块**内**声明 | **private**（私有） |
| `#pragma omp for` 的循环变量 `i` | **private**（自动） |

> ⭐ 本 lab 的核心陷阱：`global_sum` 在并行区**外**声明 → shared → 多线程同时 `+=` → **竞态（data race）**。

### 4.4 三种归约写法（概览）

| 写法 | 关键 pragma | 上锁次数 |
|---|---|---|
| ① critical | `parallel for` + 循环体内 `critical` | `n`（每元素） |
| ② reduction | `parallel for reduction(+:sum)` | 0（编译器树形合并） |
| ③ manual | `parallel` 包 `for`，线程内 `local_sum`，末尾 `critical` | 线程数 |

完整代码见下面 Exercise 2 §2.2。

### 4.5 ⭐ 为什么 critical 慢、reduction 快

**critical 的问题**：`#pragma omp critical` 是**互斥锁**，每次 `global_sum += tmp` 都要「抢锁 → 读 → 加 → 写 → 放锁」。500 万元素 = 500 万次抢锁，**同一时刻只有一个线程能进临界区**，其余全在等锁 —— 并行度被压平成串行，锁竞争本身还有开销。

> 所以 `ex2_test.c` 里 `dotp_critical` 的扫描循环写死 `for (int i = 1; i <= 1; i++)` —— **只测 1 个线程**，因为多线程会显著更慢。

**reduction 的做法**：编译器给**每线程一个私有副本**，最后一次性合并：

```text
T0: local0 = 0; for (i ∈ T0 区间) local0 += x[i]*y[i];
T1: local1 = 0; for (i ∈ T1 区间) local1 += x[i]*y[i];
global_sum = local0 + local1 + ...      // 树形/原子合并，只发生一次
```

| | critical | reduction | manual |
|---|---|---|---|
| 锁次数 | `n`（每元素） | 1（每线程） | 1（每线程） |
| 并行度 | 压成 1 | **满** | **满** |
| 多线程表现 | **越开越慢** | 接近线性 | 接近线性 |

> ⭐ 三者**数值相同**（浮点累加顺序不同，误差在 `0.001` 阈值内），差别全在性能。

### 4.6 常用子句速查

| 子句 | 作用 |
|---|---|
| `#pragma omp parallel for` | 建线程 + 切分循环，一步到位 |
| `#pragma omp for` | 必须写在已有 `parallel` 块里，只切分循环 |
| `reduction(op:var)` | 每线程私有副本 + 结束时用 `op` 合并 |
| `#pragma omp critical` | 互斥，一次一个线程（需 `{}`） |
| `#pragma omp atomic` | 只保护**单条**读-改-写语句，比 critical 轻 |
| `schedule(static\|dynamic\|guided[,chunk])` | 迭代分配方式 |
| `private(v)` / `shared(v)` | 覆盖默认作用域 |

> ⭐ `atomic` 只保护**一条**读-改-写语句（`lock add` 之类），比 `critical` 轻。
> ⭐ `schedule`：`static` 预先切块、开销最小，适合均匀迭代；`dynamic` 做完再领、负载均衡但领取开销大；`guided` 块递减折中。本 lab 默认 `static` 足够。

### 4.7 线程数与计时

```bash
OMP_NUM_THREADS=8 ./ex2      # 环境变量
```

```c
omp_set_num_threads(8);      // 代码设置（ex2_test.c 的方式）
int t = omp_get_max_threads();
```

> ⚠️ `ex2_test.c` 用 `omp_set_num_threads(i)` 逐级扫线程数，别在函数里写 `num_threads(...)` 覆盖它。
> ⭐ 计时必须用 `omp_get_wtime()`（**墙上时间**）；`clock()` 统计**所有线程 CPU 时间之和** —— 4 线程跑 1 s 显示 ~4 s，会得出「线程越多越慢」的错误结论。

---

## ⭐ Exercise 2 · OpenMP Dot Product（核心）

### 2.1 问题：点积的归约挑战

`dotp_naive` 是串行版 `sum += x[i] * y[i]`。并行化时**乘法完全可并行**，难点全在 **`global_sum +=`**：

```text
T0: 读 global_sum=5 → 加 3.2 → 写 8.2
T1: 读 global_sum=5 → 加 4.7 → 写 9.7   ← T0 的写入被覆盖（丢失更新）
```

> ⭐ **数据竞争（data race）**：结果取决于线程执行顺序，**每次都不同**。

### 2.2 三个待实现函数

```bash
make ex2 && ./ex2
```

| 函数 | 要求 | 关键写法 |
|---|---|---|
| `dotp_critical` | `critical` 保护归约 | `parallel for` + 循环体内 `critical { ... }` |
| `dotp_reduction` | `reduction` 子句 | `parallel for reduction(+:global_sum)` |
| `dotp_manual_reduction` | **不许用** `reduction` | `parallel` 包 `for`，线程内 `local_sum`，末尾 `critical` 合并 |

**① `dotp_critical`**：

```c
double dotp_critical(double* x, double* y, int arr_size) {
    double global_sum = 0.0;
    #pragma omp parallel for
    for (int i = 0; i < arr_size; i++) {
        double tmp = x[i] * y[i];       // 乘法放在临界区外
        #pragma omp critical
        {
            global_sum += tmp;
        }
    }
    return global_sum;
}
```

> ⭐ `tmp` 在临界区外算好 —— 临界区越短越好，别把乘法和取内存都塞进去。
> ⚠️ `global_sum` 必须是 shared（声明在并行区外 ✅）；`tmp` 在循环体内声明，自动 private ✅。

**② `dotp_reduction`**：

```c
double dotp_reduction(double* x, double* y, int arr_size) {
    double global_sum = 0.0;
    #pragma omp parallel for reduction(+:global_sum)
    for (int i = 0; i < arr_size; i++) {
        global_sum += x[i] * y[i];
    }
    return global_sum;
}
```

> ⭐ `reduction(+:global_sum)` 的语义：**循环体里对 `global_sum` 的读写全部转到线程私有副本上**，循环结束后用 `+` 把所有副本合并回原变量。
> ⚠️ 用了 `reduction` 就**不要**再手写 `critical`，否则双重保护，反而慢。

**③ `dotp_manual_reduction`**：

```c
double dotp_manual_reduction(double* x, double* y, int arr_size) {
    double global_sum = 0.0;
    #pragma omp parallel
    {
        double local_sum = 0;           // 每线程私有（声明在并行块内）
        #pragma omp for
        for (int i = 0; i < arr_size; i++) {
            local_sum += x[i] * y[i];
        }
        #pragma omp critical
        {
            global_sum += local_sum;    // 每线程只上锁 1 次
        }
    }
    return global_sum;
}
```

> ⭐ 三个关键点：① `parallel` 在外、`for` 在内 —— 只有这样才能在 `for` **之后**再加 `critical`；② `local_sum` 声明在并行块内 → 天然 private，**不用写 `private()`**；③ 上锁次数 = 线程数而非 `arr_size`。
> ⚠️ 写成 `#pragma omp parallel for` 就不行了：`local_sum` 随并行区结束丢失，且 `for` 之后不能跟同块代码。

### 2.3 结果与判定

`ex2_test.c` 检查数值误差 `<= 0.001`，并要求 reduction / manual 的最快时间满足 `min * 2.5 <= naive`（≥2.5×）。

```text
Naive: 1 thread took X seconds
Critical: 1 thread(s) took Y seconds     ← 与 naive 相当或更慢
Reduction Optimized: 2 thread(s) ...     ← 逐级变快
Congrats! All dotp tests passed
```

> ⚠️ 加速比受核数限制（hive 一般 4–8 核）；线程数超过物理核数后加速比回落（上下文切换开销）。

### 2.4 Exercise 2 常见 bug

| # | Bug | 现象 | 修法 |
|---|---|---|---|
| 1 | 只写 `#pragma omp for`，忘了外层 `parallel` | 编译错 / 单线程 | `for` 必须在 `parallel` 块里 |
| 2 | `critical` 块没写花括号 | 编译错 | `#pragma omp critical` 后必须 `{}` |
| 3 | 把整个循环体放进 `critical` | 比 naive 还慢 | 只把 `global_sum +=` 放进去 |
| 4 | `reduction` 变量名写错 | 结果不增长 / 编译错 | 变量名必须与源码一致 |
| 5 | manual 版写成 `parallel for` | 无法在循环后加 `critical` | 拆成 `parallel` + `for` |
| 6 | `local_sum` 声明在并行块**外** | shared → 竞态 | 移到 `{ }` 内部 |
| 7 | 用 `clock()` 测时 / 忘 `-fopenmp` | 多线程显示更慢 / pragma 被忽略 | 用 `omp_get_wtime()`；编译链接都加 `-fopenmp` |

---

## 五、SIMD vs OpenMP 对照

| | SIMD（DLP） | OpenMP（TLP） |
|---|---|---|
| 并行单位 | 一条指令内的 4/8 条 lane | 线程（各有自己的 PC） |
| 一条指令能做什么 | 只能「整齐划一」的同类运算 | 每个线程可跑不同代码 |
| 处理分支 | 困难（用掩码绕） | 容易（各线程独立） |
| 典型加速 | 3–5×（受 lane 数限制） | ~核数（受 Amdahl + 临界区限制） |
| 主要陷阱 | 对齐、tail、忘记水平求和 / 重置 | 数据竞争、过多 critical、false sharing |
| 组合方式 | 常与展开一起用 | 常与 SIMD 叠加（内层向量化，外层切线程） |

> ⭐ **两者可叠加**：ex1 纯 SIMD，ex2 纯 TLP；真实高性能库（如 BLAS）是「多线程 + SIMD + 分块 + 展开」四件套同时上。

---

## 六、Exercise 3 · 反馈问卷

无需代码，填 <https://forms.gle/EkrZFFE8ZVQNiD4M8> 即可。

---

## 七、提交流程

```bash
cd ~/labs && git add lab07 && git commit -m "Finish lab07" && git push origin main
```

然后到 Gradescope 提交 **Lab 7**。⚠️ 通常只需 `ex1.c` + `ex2.c`（Makefile 若改过也一并提交），测试文件不改。

---

## 八、常见问题（FAQ）

| 症状 | 原因 / 处理 |
|---|---|
| `_mm_set1_epi32` 未定义 | 缺 `#include <x86intrin.h>`（`ex1.h` 已有） |
| `./ex1` 报 `Illegal instruction` | CPU 不支持 AVX/SSE4，或跨机器编译运行；去 hive 机器 |
| `simd` 结果比 `reference` 少一点 | 漏 tail case：`65546 % 4 == 2`，最后 2 个元素没算 |
| `simd` 结果是负数 | `sum_vec` 没在外层循环重置 → int32 溢出 |
| `ex2` 结果不对 | `global_sum` 是 shared 且无保护 → 竞态，加 `reduction` / `critical` |
| `ex2` 极慢 | `critical` 版线程数太多；`ex2_test.c` 已限为 1 线程 |
| 不认 `#pragma omp` | 缺 `-fopenmp`（编译**和**链接都要） |

> 不用 `make` 时：`gcc -std=c99 -O0 -c ex1.c`、`gcc -std=gnu99 -fopenmp -c ex2.c`，链接 ex2 也要 `-fopenmp`。

---

## 九、待补充

| 项 | 原因 |
|---|---|
| ~~FA26 版 `lab07/` 实际文件~~ | ✅ 已核对：`ex1.c/ex1.h/ex1_test.c/ex2.c/ex2.h/ex2_test.c/openmp_example.c/Makefile`，与上文清单完全一致 |
| ~~FA26 `Makefile` 的 `CFLAGS`~~ | ✅ 已核对：`CFLAGS_SIMD=-std=c99 -Wall -O0 -c`、`CFLAGS_OMP=-std=gnu99 -fopenmp -c -g`、`LDFLAGS_OMP=-fopenmp`，**未加 `-mavx`**，与上文一致 |

---

## ✅ 验收清单

| # | 检查 | 命令 | 判据 |
|:--:|---|---|---|
| 1 | SIMD 版通过 | `make ex1 && ./ex1` | 打印 `Test Passed!`；两路 sum 与 reference 一致 |
| 2 | OpenMP 版通过 | `make ex2 && ./ex2` | 同样 `Test Passed!` |
| 3 | 无编译警告 | `make clean && make ex1 ex2` | 无 `-Wunused-variable` 之类的残留 |

> ⚠️ **构建目标是 `make ex1` / `make ex2`，不是 `make ex1_test`**（实测后者会链接失败：`undefined reference to 'sum_unrolled'`）——因为 `ex1_test` 只是测试骨架，没有你的实现。
> ⚠️ starter 原样跑 `./ex1` 会输出 `Test Failed! SIMD unrolled sum 0 doesn't match reference sum 103161741312`——`0` 说明函数还是空壳，属预期。

---

## 相关

- 对应讲次：
  - [[knowledge/course/cs61c/31-Parallelism-II-DLP-SIMD\|L31 数据级并行与 SIMD]]
  - [[knowledge/course/cs61c/32-Parallelism-III-TLP-OpenMP\|L32 线程级并行与 OpenMP]]
- 上一 lab：[[knowledge/course/cs61c-labs/Lab06\|Lab 06 · CPU 搭建与流水线]]
- Labs 总览：[[knowledge/course/cs61c-labs/index\|🧪 Labs]]
- CS61C 目录：[[knowledge/course/cs61c/index\|📚 CS61C]]
- 外部资料：
  - [Intel Intrinsics Guide](https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html)
  - [OpenMP 3.0 Summary Card](https://inst.eecs.berkeley.edu/~cs61c/sp21/resources-pdfs/OpenMP3.0-SummarySpec.pdf)
  - [SIMD Visualiser](https://piotte13.github.io/SIMD-Visualiser/#/)
  - [Godbolt Compiler Explorer](https://godbolt.org/)
