---
{"dg-publish":true,"permalink":"/knowledge/course/cs61c/32-Parallelism-III-TLP-OpenMP/","title":"CS61C L32 · Parallelism III — 线程级并行与 OpenMP","tags":["CS61C","TLP","OpenMP","thread","线程","并行计算","课程笔记"],"dg-note-properties":{"title":"CS61C L32 · Parallelism III — 线程级并行与 OpenMP","tags":["CS61C","TLP","OpenMP","thread","线程","并行计算","课程笔记"],"date":"2026-11-16","source":"https://cs61c.org/fa26/"}}
---


# CS61C FA26 · L32 Parallelism III: Thread-Level Parallelism / OpenMP

> UC Berkeley · Great Ideas in Computer Architecture
> 授课：Dan Garcia · 课件 32-Parallelism III: TLP / OpenMP（33 页）

---

## 一、动机：为什么需要 TLP（Thread-Level Parallelism）

### 1.1 时钟频率不再提升之后

前几讲讲的 ILP（instruction-level parallelism：流水线、超标量）已经榨得差不多了，而单核主频在 2000 年代中期停止增长。于是硬件厂商换了个方向：**与其让一条指令流跑得更快，不如让多条指令流同时跑。**

> **「Idea: What if we had multiple CPUs that we could execute instructions at the same time?」**

```
CPU Core 1                      CPU Core 2
  …                               …
addi …                          sw …
 lw …                           sub …
  …                               …
```

slide 提出的核心问题：

> **「Is it possible to split a single program into multiple parts and run them on different CPU cores?」**

⭐ 这就是 **TLP** —— 把「一个程序」拆成「多条并发执行的指令流」，分配到多个核上。TLP 与 ILP 的区别：

| | ILP | TLP |
|---|---|---|
| 并行来源 | 单条指令流内部的无关指令 | 多条独立的指令流 |
| 谁负责挖掘 | 编译器 + 硬件（乱序、流水线） | 程序员（写 thread 代码） |
| 硬件需求 | 流水线、多发射 | 多个 core |
| 典型技术 | 流水线、超标量、乱序 | thread、OpenMP、pthread |

---

## 二、Thread 是什么

### 2.1 定义

> **「A Thread (short for threads of execution) is a single stream of instructions.」**

每个 thread 拥有：

| 组成 | 是否共享 | 说明 |
|---|---|---|
| **Memory（heap、global variables）** | ⭐ **共享** | 同进程内所有 thread 看到同一块地址空间 |
| **Registers（含 stack pointer）** | 私有 | 每个 thread 自己一套寄存器 |
| **Program Counter (PC)** | 私有 | 每个 thread 各自知道「下一条执行哪」 |
| **Stack** | 私有 | 每次调用有自己的栈帧（slide 18 提到） |

> **「Multithreading is the use of multiple threads of execution within a single program (process).」**

⭐ 关键：「同一进程内」意味着 **地址空间是共享的** —— 这是后面的所有麻烦（以及所有便利）的根源。

### 2.2 Concurrency vs Parallelism

这是本课反复强调的一对概念：

```
Concurrency:                          Parallelism:
                                      (aka multi-processing)
Thread 1  ──┬──┬──┬──┬──             Thread 1  ──────────
Thread 2  ──┴──┴──┴──┴──             Thread 2  ──────────
            时间 →                               时间 →
```

> **「Concurrent: Making progress on multiple tasks at once (not necessarily at the same time)」**
> **「Parallel: Running at the same time (implies concurrent)」**

| 概念 | 中文 | 含义 | 需要多核？ |
|---|---|---|---|
| **Concurrency** | 并发 | 多个任务在一段时间内都在推进（可能交替） | 不需要 |
| **Parallelism** | 并行 | 多个任务在**同一时刻**真正同时跑 | 需要 |

⭐ 并行蕴含并发，反过来不成立。

### 2.3 Fork-Join 模型

> **「Fork-Join model: A program / process can split, or fork itself into separate threads, which can (in theory) execute in parallel.」**

三步：

1. **Sequential**：主线程顺序执行，直到第一个 parallel region。
2. **Fork**：主线程创建一队（team）子线程。
3. **Join**：子线程完成各自任务后同步、终止，只剩主线程继续。

```
        ┌──────┐
主线程 ─┤ fork ├─┬─ subthread 0 ─┐
        └──────┘ ├─ subthread 1 ─┤
                 └─ subthread 2 ─┴─ join ── 主线程继续
```

### 2.4 Thread vs Core

| | Thread | Core |
|---|---|---|
| 本质 | 一条指令流（软件概念） | 真正取指执行的硬件 |
| 数量关系 | 可以 >> core 数 | 有限（Hive 机器：6 核 × 2 = 12 个硬件线程） |
| 谁决定映射 | ⭐ **OS** 决定「哪个 thread 在哪个 core 上、按什么顺序跑」 | 被动执行 |

⭐ **「The operating system decides how its CPU cores execute its threads (the order isn't fixed)」** —— 顺序**没有保证**，这是线程编程非确定性的第一个来源。

### 2.5 只有一个核时的情形

slide 问：只有一个 CPU core，两条指令流能同时跑吗？

> **「Answer: It takes turns – one core can only run one instruction at a time, but the OS switches between both threads to make it look like it's running in parallel」**

两个可能的交替顺序：

```
可能的顺序 1：           可能的顺序 2：
  addi (T1)               addi (T1)
  add  (T2)               add  (T2)
   lw  (T1)                sw  (T2)
   sw  (T2)                lw  (T1)
```

多线程（3 条以上）时更明显：

```
CPU Core（单核）
  addi T1
  addi T2      ← 每一条都必须一个一个来
   lw  T1
  addi T3
   lw  T2
   lw  T3
```

> **「Instructions on a single core must run one at a time.」**

于是 slide 追问：**「Do multiple threads on one core make the program run faster?」** —— 答案通常是**不会**（只有 context switch 带来的额外开销），这正是 concurrency ≠ speedup 的直观例子。

### 2.6 多个核时

```
CPU Core 1                    Thread 1    Thread 2    Thread 3
  addi …                       addi …      addi …      addi …
   lw …                         lw …        lw …        lw …
  addi …                         …           …           …
CPU Core 2
  addi …                    ← 不同 core 上的指令可以真正同时执行
   lw …
   lw …
```

> **「Instructions on different cores can run at the same time.」**

但 slide 又追问：**「Do multiple threads on multiple cores make the program run faster?」** —— 这才进入本讲后半部分的答案：**只有当工作被正确切分、且没有数据竞争与同步开销时，才会变快。**

---

## 三、OpenMP 入门

### 3.1 并行编程语言很多，没有万能解

> **「There is no universal solution to parallel programming.」**

slide 列了一整页语言名（Ada、Chapel、Cilk、Clojure、CUDA、Erlang、Go、Haskell、Julia、MPI 家族、Scala、SISAL、VHDL…），强调：**需求极度依赖问题领域** —— 科学计算 / ML、web server、I/O 各不相同。OpenMP 的定位是：

> **「OpenMP: threads, fork-join model」**
> **「Different models for different applications (e.g., message passing for process-level parallelism, with concurrent independent tasks)」**

### 3.2 ⭐⭐ OpenMP 是什么

> **「OpenMP is an API (Application Programming Interface) with language extensions for C, C++, and Fortran.」**

| 特性 | 内容 |
|---|---|
| 全称 | **Open Multi-Processing** |
| 模型 | shared-memory parallelism，fork-join |
| 可移植性 | portable、standardized |
| 头文件 | `#include <omp.h>` |
| 编译 | `gcc -fopenmp foo.c` |

**两个核心主题（本讲 + 下一讲）：**

| 主题 | 讲次 |
|---|---|
| Shared vs. private variables | **今天** |
| 并行化与工作共享（parallelization & work sharing） | **今天** |
| 同步（synchronization） | **下一讲（L33）** |

### 3.3 Pragma 机制

> **「Pragmas are a preprocessor mechanism C provides for language extensions.」**

编译器的约定：**不认识 pragma 的编译器必须忽略它**。这带来一个巨大好处：

> ⭐ **「Even with embedded pragmas, can run on sequential computer!」**

即同一份源码，`gcc foo.c` 编出来是串行程序，`gcc -fopenmp foo.c` 编出来是并行程序 —— 可移植性极佳。61C 不讲的常见 pragma：structure packing、symbol aliasing、浮点异常模式。

### 3.4 最基本的构造：`#pragma omp parallel`

```c
#pragma omp parallel        // 开始一个 parallel region：fork 出一队线程
{                           // 花括号内的代码会被「每个线程各执行一遍」
  /* code goes here */
}                           // 大括号结束时隐式 join，子线程退出
```

> **「Each thread runs the instructions within the block」**
> **「Thread scheduling is non-deterministic」**

### 3.5 线程数与 intrinsic 函数

> **「OpenMP creates as many threads as specified in the environment variable OMP_NUM_THREADS.」**

| 项目 | 说明 |
|---|---|
| 环境变量 | `OMP_NUM_THREADS`，设成「你想用的最大线程数」 |
| 默认值 | 一般是 (物理核数) × (每核线程数) |
| Hive 机器 | `lscpu` 查看，**6 × 2 = 12** 个硬件线程 |
| 本质 | OpenMP thread 是 **OS (software) thread**，由 OS 复用到硬件线程上 |

⭐ **OpenMP intrinsic 函数表：**

| Intrinsic | Description |
|---|---|
| `omp_set_num_threads(x);` | Set number of threads to x.（程序内设定线程数） |
| `num_th = omp_get_num_threads();` | Get number of threads.（当前 parallel region 里的线程总数） |
| `tid = omp_get_thread_num();` | Get Thread ID number.（当前线程编号，0 ~ n-1） |

---

## 四、⭐ Shared vs. Private 变量（本讲重点之一）

> **「Shared variables: all threads read/write the same variable.」**
> **「Private variables: Each thread has its own copy of the variable.」**

```c
int var1, var2;                  // 声明在 parallel region 之外
char *var3 = malloc(…);
#pragma omp parallel private(var2)   // 显式声明 var2 为私有
{
  int var4;                      // 声明在 parallel region 之内
  // var1 shared (default)       ← 外部声明的全局/局部 → 默认共享
  // var2 private                ← private() 子句强制私有，每个线程一份副本
  // var3 shared (heap)          ← 指针本身共享，指向的堆内存也共享
  // var4 private (thread's stack) ← 块内声明 → 在每个线程自己的栈帧上
  …
}
```

**判定规则表：**

| 变量声明位置 | 默认属性 | 原因 |
|---|---|---|
| parallel region **之外** | **Shared** | 同一块地址空间 |
| **Heap** 分配（`malloc`） | **Shared** | 堆对所有线程可见 |
| **Static** 变量 | **Shared** | 全局存储区 |
| parallel region **之内** | **Private** | 每个线程有独立栈帧 |
| `private(x)` 子句 | **Private** | 显式覆盖默认 |

> ⭐ **「Heap memory is shared across threads.」**

```
           ┌────────────── 进程地址空间 ──────────────┐
Thread 0   │  stack0 │  regs0 │  PC0  │              │
Thread 1   │  stack1 │  regs1 │  PC1  │   heap  ────┼── 共享
Thread 2   │  stack2 │  regs2 │  PC2  │   globals ──┼── 共享
           └──────────────────────────────────────────┘
                私有 ←────────────┼──────────→ 共享
```

⚠️ 注意 `private(var2)` 只是「每个线程有一份自己的 var2」，它**没有初值**（如果需要初值要用 `firstprivate`，61C 未展开）。这一点在写归约时会咬人（见第 6 节）。

---

## 五、Threaded Hello World

### 5.1 Take 1：`#pragma omp parallel`

```c
 1  #include <stdio.h>
 2  #include <omp.h>                       // 提供 OpenMP API（线程号/线程数）
 3  int main() {
 4    /* Fork team of threads with private variable tid */
 5    #pragma omp parallel               // fork：每个线程都进入下面的块
 6    {
 7      int tid = omp_get_thread_num();  // 取自己的线程编号（0..n-1），块内声明→私有
 8      printf("Hello World from thread = %d\n", tid);
 9
10      /* Only main thread does this */
11      if (tid == 0) {                  // 只有 0 号线程（主线程）执行这段
12        printf("Number of threads = %d\n",
13               omp_get_num_threads()); // 打印本 region 的线程总数
14      }
15    } /* All threads join main and terminate */  // 块结束 → 隐式 join
16    return 0;
17  }
```

**逐点拆解：**

- 第 5 行：**parallel region 的入口**，被每个子线程各执行一次（连同一行代码，n 个线程共 n 次）。
- 第 7 行：`omp_get_thread_num()` 保证每个线程拿到**不同的 tid**。
- 第 11 行：`if (tid == 0)` 是 OpenMP 里常用的「让主线程做汇总/打印」的写法。
- 输出顺序**不确定** —— 谁先打印由 OS 调度决定。

编译与运行：

```bash
gcc -fopenmp hello.c -o hello
OMP_NUM_THREADS=4 ./hello    # 4 个线程
```

### 5.2 Take 2：加一个共享变量（⚠️ 埋下 race condition）

```c
 1  #include <stdio.h>
 2  #include <omp.h>
 3
 4  /* Fork team of threads with private variable tid, shared variable x */
 5  int main() {
 6    int x = 0;                        // 在 region 外声明 → 共享变量
 7    #pragma omp parallel
 8    {
 9      int tid = omp_get_thread_num(); // 私有
10      x++;                            // ⚠ 对共享变量做读-改-写，没有同步！
11
12      printf("Hello World from thread = %d, x = %d\n", tid, x);
13      /* Only main thread does this */
14      if (tid == 0) {
15        printf("Number of threads = %d\n", omp_get_num_threads());
16      }
17    } /* All threads join main and terminate */   ⚠ race condition!
18    return 0;
19  }
```

⭐ slide 在同一页就打了两个警告标记：**「⚠ race condition!」** + **「(more next time) ⚠」** —— 下一讲 L33 整讲都在讲这件事。

### 5.3 OS 与 Context Switch（插播）

> **「Context switches: The OS can choose whichever threads it wants to run, and change threads at any time」**

context switch 时要换掉的内容：

- Registers
- Stack pointer、Program Counter
- （**不换**）内存：同一程序内的线程共享地址空间

现实：

> **「On most modern computers, number of active threads >> number of available cores, so most threads are idle at any given time」**
> ⭐ **「Synchronizing use of shared resources is hard!」**

这个问题正是 L33 的全部内容。

### 5.4 关于线程的一句名言

> 「Although threads seem to be a small step from sequential computation, in fact, they represent a huge step. They discard the most essential and appealing properties of sequential computation: understandability, predictability, and determinism. Threads, as a model of computation, are wildly nondeterministic, and the job of the programmer becomes one of pruning that nondeterminism.」
> —— Edward A. Lee, *The Problem with Threads*, UC Berkeley Tech Report UCB/EECS-2006-1 (2006)

⭐ 一句话总结：**线程把「确定性」丢掉了**，程序员的工作就是把这个非确定性重新剪掉。

---

## 六、⭐ 循环工作共享（For Loop Work-Sharing）

### 6.1 问题定义

> **「Problem: You have to do some work over an array of 2^27 numbers, with 12 people. How do you split the work?」**

前提假设（slide 明确列出）：

| 假设 | 含义 |
|---|---|
| 必须在跑代码**之前**决定切分 | 划分方案是静态写死在源码里的 |
| 数组每个元素**互相独立** | 可以任意顺序做 |
| 各线程速度**基本相等** | 每人分到约 11 million 个元素 |

原始串行代码：

```c
#define LENGTH (1 << 27)                 // 2^27 ≈ 1.34 亿个元素
int main(...) {
  char *arr = malloc(sizeof(char) * LENGTH);   // 堆分配 → 所有线程共享
  /* how to parallelize this loop? */
  for (int i = 0; i < LENGTH; i++) {
    arr[i] = …;                          // 每个元素独立写
  }
}
```

### 6.2 四种写法对比（本讲核心 slide）

#### A. 只 fork，不切分 —— 重复劳动

```c
/* A. */
#pragma omp parallel                    // 只 fork 出 N 个线程
{
  for (int i = 0; i < LENGTH; i++) {    // ⚠ 每个线程都完整跑一遍整个循环
    arr[i] = …;                         // N 个线程做 N 倍的工作
  }
}
```

> slide 批注：**「duplicates work」**

⛔ 这不是并行，是**重复做 N 倍工作**（而且更慢）。

#### B. 手动交错（interleaved）

```c
/* B. */
#pragma omp parallel                    // fork 出 N 个线程
{
  int tid = omp_get_thread_num();       // 我的线程编号
  int num_threads = omp_get_num_threads(); // 总共多少线程
  for (int i = tid;                  // 从「我的编号」开始
       i < LENGTH;
       i+= num_threads) {            // 每次跳 N 步 → 交错访问
    arr[i] = …;
  }
}
```

> slide 批注：**「"interweaves" array access between threads」**

线程 0 处理 0, N, 2N, …；线程 1 处理 1, N+1, 2N+1, …

#### C. 手动分块（chunks）

```c
/* C. */
#pragma omp parallel
{
  int tid = omp_get_thread_num();
  int num_threads = omp_get_num_threads();
  int thread_start = tid * LENGTH / num_threads;    // 我的起始下标
  int thread_end   = (tid+1) * LENGTH / num_threads; // 我的结束下标（不含）
  for (int i = thread_start; i < thread_end; i++) {  // 连续的一块
    arr[i] = …;
  }
}
```

> slide 批注：**「"chunks" array」**（注意：slide 这一版把 pragma 误印成 `#pragma omp parallel sections`，语义上应为 `parallel`，此处按上下文还原）

线程 0 → [0, L/N)，线程 1 → [L/N, 2L/N)，……

#### D. 用 `#pragma omp for` —— OpenMP 帮你切

```c
/* D. */
#pragma omp parallel                    // fork 出线程队
{
  #pragma omp for                        // 工作共享：把下面的循环迭代分给各线程
  for (int i = 0; i < LENGTH; i++) {
    arr[i] = …;
  }
}
```

> slide 批注：**「Like C, but planned via OpenMP」**

### 6.3 四者对比表

| 写法 | 工作切分 | 是否重复工作 | 访存模式 | 相对效果 |
|---|---|---|---|---|
| **A** 只 `parallel` | 无 | ⛔ **N 倍重复** | 每个线程全量扫 | 最慢 |
| **B** 交错手动 | 按 tid 步进 | 无 | interleaved | 快，但 stride 大（缓存不友好） |
| **C** 分块手动 | 连续区间 | 无 | chunk（顺序） | 快，缓存友好 |
| **D** `#pragma omp for` | 编译器代劳 | 无 | 由 `schedule` 决定 | ⭐ 推荐，且可调调度 |

⭐ 结论：**「Simply creating threads duplicates the work. We must divide iterations among threads.」**

### 6.4 ⭐ `#pragma omp for` 的语法规则

```c
#pragma omp parallel                    // 先有 parallel region
{
  #pragma omp for                        // 再在内部声明工作共享
  for (int i = 0; i < LENGTH; i++) {
    arr[i] = …;
  }
}
```

**等价简写**（当 parallel region 里只有这一个循环）：

```c
#pragma omp parallel for                 // 二合一：fork + 切分循环
for (int i = 0; i < LENGTH; i++) {
  arr[i] = …;
}
```

**硬性规则：**

| 规则 | 说明 |
|---|---|
| 必须写在已有 parallel region **内部** | 或者用 `parallel for` 合一 |
| 循环形状必须**简单** | OpenMP-aware 编译器要能静态判断迭代次数 |
| 迭代次数必须在进入循环前**已知** | 运行时系统要能算「每个线程分几次」 |
| ⛔ **不允许提前退出** | 不能用 `break`、`return`、`exit`、`goto` |

### 6.5 调度策略 `schedule(...)`

`#pragma omp for` 后可以加 `schedule` 子句控制「迭代怎么分给线程」：

| 调度 | 写法 | 分配方式 | 适用场景 |
|---|---|---|---|
| **static** | `schedule(static[, chunk])` | 编译期预先切成固定块，线程各自领走 | 迭代耗时**均匀**、开销最小 |
| **dynamic** | `schedule(dynamic[, chunk])` | 每个线程做完当前块后**动态领取**下一块 | 迭代耗时**不均**（负载不均衡） |
| **guided** | `schedule(guided[, chunk])` | 块大小从大到小递减（动态领取） | 折中：减少领取次数又保持均衡 |

```
static（4 线程 / 12 迭代）：
  T0: 0 1 2      T1: 3 4 5      T2: 6 7 8      T3: 9 10 11

dynamic（chunk=1，谁先做完谁领下一个）：
  时间 →  T0:0   T2:1   T0:2   T1:3   T0:4   T2:5 …
```

⭐ `dynamic` 负载均衡好但领取开销大；`static` 开销小但可能某个线程拖尾；`guided` 是二者的折中。

### 6.6 计时：`clock()` vs `omp_get_wtime()`

> **「`clock()` from `<time.h>` measures active CPU time.」** —— 所有线程 CPU 时间**之和**（注意：并行 4 线程跑 1 秒 wall time，可能显示 4 秒 CPU time！）
> **「`omp_get_wtime()` measures wall time (in seconds) with respect to one thread」**

```c
#include <omp.h>
int main(...) {
  double start, end;
  start = omp_get_wtime();              // 记录墙上时间起点
  /* ... work to be timed ... */        // 要测的并行工作
  end = omp_get_wtime();                // 记录墙上时间终点
  printf("Work took %f seconds\n", end - start);  // wall time 差值 = 真实耗时
}
```

⭐ **测并行加速比必须用 `omp_get_wtime()`**，用 `clock()` 会把「并行」测成「更慢」。

---

## 七、归约与数据竞争（预告 L33）

### 7.1 朴素写法为什么错

假设要算数组和：

```c
int sum = 0;                            // 共享（region 外声明）
#pragma omp parallel for
for (int i = 0; i < N; i++) {
  sum += arr[i];                        // ⚠ 多个线程同时读-改-写同一个 sum → 数据竞争
}
```

`sum += arr[i]` 在机器层面是 **load → add → store** 三步。两个线程交替执行时：

```
T0: load sum (=10)
T1: load sum (=10)        ← T1 也读到 10
T0: add arr[i]  → 11
T0: store sum = 11
T1: add arr[j]  → 12
T1: store sum = 12        ← 覆盖掉 T0 的 11，丢了一次加法
```

➡️ 结果**不确定**，这就是 **race condition**（L33 第一章）。

### 7.2 ⭐ `reduction` 子句：正确解法

```c
int sum = 0;                            // 共享变量
#pragma omp parallel for reduction(+:sum)  // 每个线程有 sum 的私有副本（初值 0）
for (int i = 0; i < N; i++) {
  sum += arr[i];                        // 各线程只更新自己的副本，不冲突
}                                       // region 结束时，所有副本用 + 合并回 sum
```

**归约的语义拆解：**

| 阶段 | 发生什么 |
|---|---|
| fork 时 | 为每个线程创建一个 `sum` 的**私有副本**，用加法单位元 `0` 初始化 |
| 循环体内 | 每个线程只累加自己的副本 ✔ 无竞争 |
| join 时 | 各副本按 `+` 依次合并到共享 `sum` |

支持的归约操作符：`+ - * & | ^ && || max min`（slide 未细列，此处按 OpenMP 标准惯例，考试以老师标注为准）。

⭐ 对比表：

| 写法 | 正确性 | 性能 |
|---|---|---|
| `sum += arr[i]` 直接写 | ❌ race condition，结果不确定 | 看似快，实则错 |
| `#pragma omp critical { sum += arr[i]; }` | ✔ 正确 | 每次迭代都要加锁 → **串行化**，极慢 |
| `reduction(+:sum)` | ✔ 正确 | 每个线程一份副本 → 几乎线性扩展 |

---

## 八、案例：矩阵乘法并行化

把 `C = A × B` 的三重循环并行化。**关键洞察**：外层 `i` 循环的每次迭代写的是 `C` 的不同行，**行与行之间独立**。

```c
#define N 1024
double A[N][N], B[N][N], C[N][N];

#pragma omp parallel for schedule(static)     // 按行切分，静态调度
for (int i = 0; i < N; i++) {                 // ← 每个线程负责若干行 i
  for (int j = 0; j < N; j++) {               // j 串行
    double s = 0.0;                           // 块内声明 → 每个线程私有
    for (int k = 0; k < N; k++) {             // k 串行
      s += A[i][k] * B[k][j];                 // 读 A 的整行 + B 的整列
    }
    C[i][j] = s;                              // 只写自己在第 i 行上的元素
  }
}
```

**为什么只并行外层：**

| 并行层 | 是否安全 | 说明 |
|---|---|---|
| `i`（外层） | ✔ 安全 | 每个线程写 `C` 的不同行 |
| `j`（中层） | ✔ 安全但粒度太细 | 每次要 fork/领任务，开销占比过高 |
| `k`（内层） | ⛔ 不安全 | 同一 `s` 被多个线程累加 → 需要 `reduction(+:s)`，且 `s` 声明在循环内做不到 |

⭐ 矩阵乘是「**弱扩展性**」代表：`O(N³)` 工作量，但数据量 `O(N²)`，缓存行为对性能影响极大（回忆 L26–L29）。

---

## 九、加速比实测与 Amdahl 呼应

把前面的循环实跑（12 个硬件线程的 Hive 机器），典型观测：

| 版本 | 时间 | 加速比 | 说明 |
|---|---|---|---|
| A. 只 `parallel`（重复工作） | ≈ N × T_seq | **< 1（比串行慢）** | 做了 12 倍工作 |
| B. 手动交错 | 明显快 | 数倍 | 但 stride 大，缓存不友好 |
| C. 手动分块 | 更快 | 更接近线性 | 连续访问 |
| D. `#pragma omp for` | 与 C 相当 | 接近线性 | OpenMP 的 `schedule(static)` 就等价于分块 |

**为什么做不到 12×：**

⭐ 呼应 Amdahl's Law（L23）：

```
               1
加速比 = ─────────────────────
        (1 - p) + p / n
```

| 因素 | 影响 |
|---|---|
| **串行部分（1-p）** | fork/join、`printf`、初始化 → 上限 |
| **同步开销** | 子句 `critical`、`barrier` 的等待 |
| **false sharing** | 相邻线程写同一 cache line（L33 详谈）|
| **内存带宽** | 多核一起抢 DRAM，带宽成为瓶颈 |
| **负载不均** | static 调度拖尾 |

> ⭐ **总结 slide 的原话：「Parallelism does not guarantee speedup. Work distribution, memory access patterns, and threading overhead matter.」**

---

## 十、本讲小结（slide 33 原文要点）

| # | 要点 |
|---|---|
| 1 | Multithreading：每个 thread 有自己的 registers、PC、stack，但**共享内存** |
| 2 | Concurrency vs. parallelism：并发是「时间段重叠」，并行是「真正同时」 |
| 3 | OpenMP 用 compiler directive 写 shared-memory 并行，例如 `#pragma omp parallel` |
| 4 | Shared vs. private：全局变量与堆共享；局部变量私有；**未同步访问共享数据 → race condition** |
| 5 | 并行化循环：**只 fork 会重复工作**，必须分迭代（手动或 `#pragma omp parallel for`） |
| 6 | 性能：并行**不保证**加速；工作分配、内存访问模式、线程开销都决定结果 |

---

## 术语速查

| 术语 | 中文 | 说明 |
|---|---|---|
| **TLP** | 线程级并行 | Thread-Level Parallelism，多线程并行 |
| **ILP** | 指令级并行 | 流水线/超标量，单流内并行 |
| **thread** | 线程 | 一条指令流，私有 reg/PC/stack，共享内存 |
| **process** | 进程 | 独立地址空间，thread 的容器 |
| **multithreading** | 多线程 | 一个进程内多条指令流 |
| **concurrency** | 并发 | 时间段内推进，可能交替 |
| **parallelism** | 并行 | 同一时刻真正同时 |
| **fork-join** | 分叉-汇合 | 主线程分叉子线程，结束后汇合 |
| **core** | 核心 | 实际执行指令的硬件 |
| **context switch** | 上下文切换 | OS 换线程，换寄存器/SP/PC |
| **OMP_NUM_THREADS** | 线程数环境变量 | 决定 OpenMP 创建多少线程 |
| **pragma** | 编译指示 | 预处理器扩展，不认识的编译器会忽略 |
| **parallel region** | 并行区 | `#pragma omp parallel` 的花括号块 |
| **work-sharing** | 工作共享 | 把循环迭代分给线程队 |
| **schedule(static)** | 静态调度 | 预先固定切块 |
| **schedule(dynamic)** | 动态调度 | 做完再领下一块 |
| **schedule(guided)** | 引导调度 | 块大小递减的动态调度 |
| **reduction** | 归约 | 各线程私有副本，结束时合并 |
| **race condition** | 数据竞争 | 多线程无同步访问同一内存，结果不确定 |
| **shared variable** | 共享变量 | 所有线程同一份 |
| **private variable** | 私有变量 | 每线程一份副本 |
| **omp_get_thread_num** | 取线程号 | 返回 0..n-1 |
| **omp_get_num_threads** | 取线程总数 | 当前 region 线程数 |
| **omp_get_wtime** | 取墙上时间 | 秒，用于测并行耗时 |
| **duplicates work** | 重复工作 | 只 fork 不切分的后果 |
| **speedup** | 加速比 | T_seq / T_par |

---

## 相关

- 课程导航：[[knowledge/course/cs61c/index\|CS61C 目录导航]]
- 上一讲：[[knowledge/course/cs61c/30-Parallelism-I-Intro\|L30 Parallelism I — 并行导论]]
- 下一讲：[[knowledge/course/cs61c/33-Parallelism-IV-Concurrency\|L33 Parallelism IV — 并发与同步]]
