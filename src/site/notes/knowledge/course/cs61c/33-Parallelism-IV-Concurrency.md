---
{"dg-publish":true,"permalink":"/knowledge/course/cs61c/33-Parallelism-IV-Concurrency/","title":"CS61C L33 · Parallelism IV — 并发与同步","tags":["CS61C","concurrency","race condition","mutex","atomic","lock","同步","课程笔记"],"dg-note-properties":{"title":"CS61C L33 · Parallelism IV — 并发与同步","tags":["CS61C","concurrency","race condition","mutex","atomic","lock","同步","课程笔记"],"date":"2026-11-16","source":"https://cs61c.org/fa26/"}}
---


# CS61C FA26 · L33 Parallelism IV: Concurrency and Synchronization

> UC Berkeley · Great Ideas in Computer Architecture
> 授课：Dan Garcia · 课件 32-Parallelism IV: Concurrency（39 页）

---

## 一、回顾：Take 2 埋的雷

> **「Fork team of threads with private variable tid, shared variable x」**

```c
 1  #include <stdio.h>
 2  #include <omp.h>
 3
 4  /* Fork team of threads with private variable tid, shared variable x */
 5  int main() {
 6    int x = 0;                        // 共享变量（region 外声明）
 7    #pragma omp parallel
 8    {
 9      int tid = omp_get_thread_num(); // 私有
10      x++;                            // 对共享变量读-改-写，无同步
11
12      printf("Hello World from thread = %d, x = %d\n", tid, x);
13      /* Only main thread does this */
14      if (tid == 0) {
15        printf("Number of threads = %d\n", omp_get_num_threads());
16      }
17    } /* All threads join main and terminate */
18    return 0;
19  }
```

⭐ 上一讲结论是「⚠ race condition!」，本讲要回答：**到底是哪一步坏了，怎么修，修的成本是多少。**

---

## 二、⭐⭐ Race Condition（数据竞争）

### 2.1 slide 的课堂提问

```c
int x = 0;
#pragma omp parallel
{
  x = x + 1;
}
```

> **「Suppose we run the below code on 4 threads. What are possible values of x after running this code?」**
> A. 0　B. 1　C. 2　D. 3　E. 4　F. 5 or more

⭐ 正确答案是 **B、C、D、E（1、2、3、4）** —— 这是本讲第一个反直觉点：4 个线程各加 1，结果**不一定是 4**。

### 2.2 ⭐⭐ 定义

> **「Two memory accesses form a race condition (aka data race) if:」**
> - **「They are from different threads to the same location」**
> - **「At least one is a write」**
> - **「They occur one after another.」**

三个条件**同时**满足才是 race condition：

| 条件 | 反例（不构成 race） |
|---|---|
| 不同线程 | 同一线程内的两次访问 → 顺序确定 |
| 同一内存位置 | 两个线程写不同数组元素 → 安全 |
| 至少一个是写 | 两个线程都只读 → 安全 |
| 交错发生 | 硬件保证原子 → 安全 |

⚠ 注意最后一条的微妙处：**「They occur one after another」** 指的是「执行在时间上有交叠/相邻」，而不是「顺序确定」。

### 2.3 根本原因：`x++` 不是原子操作

> **「Remember thread model: shared memory.」**
> **「For a given thread, these two operations don't necessarily happen together…!」**

```
x++ 在机器层面 = 三步：
  ┌─────────────────────────────────────┐
  │ 1. load   ：把 x 从内存读进寄存器   │
  │ 2. add    ：寄存器加 1              │
  │ 3. store  ：把寄存器写回内存        │
  └─────────────────────────────────────┘
   ⚠ 只有这三步「整体不被打断」，x++ 才是原子的
```

对应 RISC-V 汇编：

```asm
# 四个线程各执行一遍下面这段
int x = 0;              sw  x0, 0(sp)     # x = 0，初始化
#pragma omp parallel
{
   x = x + 1;           lw  t0, 0(sp)      # ① load：读 x
                        addi t0, t0, 1     # ② add ：加 1
                        sw  t0, 0(sp)      # ③ store：写回 x
}
```

⭐ **`lw` 和 `sw` 之间可以被别的线程插入。** 这就是 L33 全讲的核心事实。

### 2.4 Data race ≠ Data hazard

| | Data hazard | Data race |
|---|---|---|
| 英文 | data hazard | race condition / data race |
| 发生在 | **单条指令流内部** | **多条指令流之间** |
| 原因 | 相邻指令有数据依赖，流水线来不及 | 缺少同步，谁先访问内存不确定 |
| 解决 | forwarding、stall（L24） | 加锁/原子指令 |
| 是否可能无 ILP 也出错 | 不会 | ⭐ **会** |

> **「Not a data hazard…! Here, even with no ILP, can have nondeterministic results. This results from lack of synchronization on which thread accesses memory first.」**

---

## 三、⭐ 数据竞争的三种经典交错

前提：**非流水线、单周期数据通路** → 每条指令都是**原子**的（instruction atomic），即「指令执行中不会被插队」。即便如此，**指令之间**就可以插队。

### 3.1 Case 1：全部串行（最理想的顺序）

| 步骤 | 动作 | 事件 |
|---|---|---|
| 1 | Orange load | read x: **0** |
| 2 | Orange store | write x: **1** |
| 3 | Green load | read x: **1** |
| 4 | Green store | write x: **2** |
| 5 | Blue load | read x: **2** |
| 6 | Blue store | write x: **3** |
| 7 | Purple load | read x: **3** |
| 8 | Purple store | write x: **4** |

> **Final value of x: 4**

### 3.2 Case 2：完美交错（最坏情况）

| 步骤 | 动作 | 事件 |
|---|---|---|
| 1 | Orange load | read x: **0** |
| 2 | Green load | read x: **0** |
| 3 | Blue load | read x: **0** |
| 4 | Purple load | read x: **0** |
| 5 | Orange store | write x: **1** |
| 6 | Green store | write x: **1** |
| 7 | Blue store | write x: **1** |
| 8 | Purple store | write x: **1** |

> **Final value of x: 1**

⭐⭐ 四个人都读了 0，都算成 1，都写回 1 —— 三次加法凭空消失。

```
时间 →
T_orange  load(0) ─────────────────────── store(1)
T_green       load(0) ───────────────── store(1)
T_blue            load(0) ──────────── store(1)
T_purple              load(0) ─────── store(1)
                       ↑ 四个 load 挤在一起，导致结果 = 1
```

### 3.3 Case 3：Orange 最后写

| 步骤 | 动作 | 事件 |
|---|---|---|
| 1 | Orange load | read x: **0** |
| 2 | Green load | read x: **0** |
| 3 | Green store | write x: **1** |
| 4 | Blue load | read x: **1** |
| 5 | Blue store | write x: **2** |
| 6 | Purple load | read x: **2** |
| 7 | Purple store | write x: **3** |
| 8 | Orange store | write x: **1**（用陈旧的 0+1） |

> **Final value of x: 1**

⭐ Orange 的 store 用了「很早以前读到的 0 加上 1」，把后来的 3 覆盖了 —— 这叫**丢失更新（lost update）**。

### 3.4 ⭐ 可能取值的完整分析

> **「105* possible instruction orderings to generate final value of x!」**

slide 给出的边界分析：

| 边界 | 理由 |
|---|---|
| **不能超过 4** | 总共只有 4 条 `addi +1` 指令 |
| **不能低于 1** | 最小的 load 值是 0，所以最小的最后一次 store 值是 1 |
| **1 到 4 之间都能取到** | 通过精心安排交错 |

> ⭐ **「Can get any value between 1 and 4」**

**正确性的形式化定义：**

> **「Formally, a multithreaded program is only considered correct if ANY interlacing of threads yield the same result.」**

| 判断 | 结论 |
|---|---|
| 当前程序 | ⛔ **incorrect program**（结果 1~4 都可能） |
| 但如果每个线程处理**独立数据** | ✔ 可保证正确（例如上一讲的 `for` 循环） |

> **「The hardest part of multithreading is understanding and maintaining correctness by synchronizing threads.」**

---

## 四、⭐⭐ 同步：Critical Section 与 Barrier

### 4.1 同步要解决的问题

> **「To enforce multithreaded program correctness, we often need to synchronize threads, i.e., coordinate their execution.」**
> **「Most commonly, know when one task is finished writing so that it is safe for another to read.」**

我们希望的执行形态（四个线程任意顺序，但**每个 load/addi/store 三元组紧密相连**）：

```
load   load   load   load
addi   addi   addi   addi
store  store  store  store
 ↑ 这四段之间可以任意排列（permutation），但段内不许插队
```

⚠ **代价警告：**

> **「Note: If we enforce the above, then execution effectively becomes sequential… (Amdahl's Law)」**

⭐ 同步本质上是**用并行度换正确性**。

### 4.2 定义与两个 pragma

> **「A critical section is a segment of code that must be executed by a single thread at a time, thereby enforcing synchronization.」**

| 指令 | 作用 |
|---|---|
| `#pragma omp barrier` | **「Forces all threads to wait until all threads have hit the barrier」** |
| `#pragma omp critical` | **「Creates a critical section within a parallel code segment; only one thread can run a critical section at a time.」** |

⭐ 关键层次：

> **「This user-level specification (e.g., in OpenMP) relies on hardware synchronization instructions (e.g., in RISC-V)」**

```
程序员的 #pragma omp critical      ← 源程序层（方便）
        ↓ 编译
atomic 指令 / 锁实现               ← 指令层（RISC-V：amoswap、lr/sc）
        ↓ 编译
CPU 缓存一致性 + 总线仲裁          ← 硬件层（保证真的原子）
```

### 4.3 同步版 Hello World

```c
 1  #include <stdio.h>
 2  #include <omp.h>
 3  int main() {
 4    int x = 0;                        // 共享变量
 5    #pragma omp parallel
 6    {
 7      int tid = omp_get_thread_num(); // 私有变量
 8      #pragma omp critical             // 进入临界区：同一时刻只有一个线程能进来
 9      {
10        x++;                          // 读-改-写被保护，不会再丢更新
11        printf("Hello World from thread = %d, x = %d\n", tid, x); // 一并纳入，保证 x 和 tid 打印一致
12      }
13      #pragma omp barrier              // 栅栏：所有线程都到达这里才能继续
14      if (tid == 0) {                  // 主线程在 barrier 之后才打印总数
15        printf("Number of threads = %d\n", omp_get_num_threads());
16      }
17    }
18    return 0;
19  }
```

**逐行要点：**

| 行 | 说明 |
|---|---|
| 8 | `critical` 是**互斥（mutual exclusion）**：只有一个线程能站在花括号里 |
| 10-11 | 把 `x++` 和 `printf` 都放进去 → 打印出来的 x 值是「确切的、连续的」 |
| 13 | `barrier` 保证「所有线程 x++ 都做完了」 |
| 14 | 否则 0 号线程可能在别的线程还没加完时就打印 |

⚠ **放进 critical 的代码越多，串行段越长**（第 11 行的 `printf` 也串行化了）。

### 4.4 ⭐ OpenMP 的局限

> **「OpenMP has very restrictive parallelism. Really only good for parallelizing loops…」**

| 问题 | 后果 |
|---|---|
| critical section 太大 | **「effectively serial program → Amdahl's law quickly rears its ugly head」** |
| critical section 定义不好 | **「can run into deadlock」** |

> ⭐ **「OpenMP is a simple extension to C that (1) abstracts away the challenges of parallel programming, but (2) is therefore limited in performance.」**

---

## 五、⭐ 锁（Lock）—— OpenMP 幕后的东西

> **「How might OpenMP synchronization work, under the surface?」**

### 5.1 买牛奶类比：Attempt 1（无同步）

```
// attempt 1
if milk not in fridge:
  buy milk at store
  put milk in fridge
```

> 如果室友在你出门买奶时也回家了 → **「Result: Two milk cartons! Race condition!」**

⭐ 这就是 `if (x == 0) x = 1;` 之类的 check-then-act 的经典形状。

### 5.2 Attempt 2：冰箱上贴纸条

```
// attempt 2, with note
if note not on fridge:
  if milk not in fridge:
    put note on fridge
    buy milk at store
    put milk in fridge
    take note off fridge
```

看起来是对了，但 slide 给出真实交错：

```
// you                                    // roommate              time
if note not on fridge:
  if milk not in fridge:
                                          if note not on fridge:    ← 同时
                                            if milk not in fridge:  ← 同时
    put note on fridge
    buy milk at store
    put milk in fridge
    take note off fridge
                                            put note on fridge      ← 纸条被覆盖
                                            buy milk at store
                                            put milk in fridge
                                            take note off fridge
```

> ⭐ **「Even when threads execute in parallel, they still sequentially access shared resources.」**

⇒ 结论：**「check 然后 set」必须是原子的**，分两步做就一定有窗口。

### 5.3 正确的解法：Lock

> **「Solution: Use a lock to grant one thread exclusive access to a resource or critical section.」**
> **「Each thread can try to "acquire" the lock, but only one thread can have the lock at a given time.」**
> **「Analogy: bathroom stall locks. One person at a time; others wait.」**
> **「Need all processors to be able to access the lock, so put the lock in a shared memory location」**

**两个操作：**

| 操作 | 语义 |
|---|---|
| **Acquire** | 尝试拿锁；成功就继续，失败就等一下再试（spin） |
| **Release** | 放锁（只有原本持有者才能成功） |

```
// attempt 3, with lock
acquire fridgelock
  if milk not in fridge:
    buy milk at store
    put milk in fridge
release fridgelock
```

实际交错（室友被挡在门外）：

```
// you                                    // roommate
acquire fridgelock
                                          acquire fridgelock    <wait>
  if milk not in fridge:
    buy milk at store
    put milk in fridge
release fridgelock
                                            if milk not in fridge:  ← 此时牛奶已有了
                                              buy milk at store     ← 不再执行
                                              put milk in fridge
                                          release fridgelock
```

⚠ 但：

> **「Locks inherently enforce some serialization of threads. Amdahl's Law strikes again!」**

### 5.4 死锁（Deadlock）

> **「Deadlock: A system state in which no progress is possible because everything is locked waiting for something else.」**

**Dining Philosophers（哲学家就餐，Dijkstra 1965）：**

- 特殊意大利面必须用**两把叉子**吃（左一把、右一把）
- 每位哲学家只能「思考」或「吃饭」
- 如何防止**饿死（starvation）**？

**slide 给出的算法：**

```
Think until left fork available; when it is, pick it up
Think until right fork available; when it is, pick it up
When both forks are held, eat for a fixed amount of time
Then, put the right fork down
Then, put the left fork down
Repeat from the beginning
```

**死锁如何发生：**

```
      P0 ──叉0──┐        全都拿了左叉
      P1 ──叉1──┤        全都在等右叉
      P2 ──叉2──┤   →    永远等不到
      P3 ──叉3──┤
      P4 ──叉4──┘
```

> ⭐ **「All philosophers grab left fork at same time → All are waiting for right fork → All stuck thinking forever」**
> **「Deadlock situations are common when working directly with locks. Be careful!」**

### 5.5 OpenMP 里的 lock

> **「OpenMP also has locks, but we suggest just using the existing critical section directives.」**（参考 openmp.org spec-html/5.0/openmpse31.html）

⭐ 建议：**让 OpenMP 自己管锁**；需要底层 pthreads 的话，去上 CS162。

---

## 六、⭐ 锁的朴素实现（Naive Spin Lock）

### 6.1 设定

| 项 | 值 |
|---|---|
| 锁位置 | 共享内存 `@ 0x5555 0000`，保存在 `s0` |
| **0** | **unlocked / open（开锁）** |
| **1** | **locked / closed（关锁）** |

> **「Technically we call these spin locks.」** —— 拿不到锁就原地空转重试。

### 6.2 ⛔ 朴素实现（错的）

```c
// 汇编写法
acquire:
        li      t1, 1          # t1 = 1（要写入的"锁上"值）
try:    lw      t0, 0(s0)      # ① 读锁状态
        bnez    t0, try        # 若非 0（已锁）→ 回去重试（spin）
lock:   sw      t1, 0(s0)      # ② 写 1，表示我锁上了

release:
        sw      x0, 0(s0)      # 写 0，解锁
```

### 6.3 两个线程同时 acquire 的实际交错

```
# thread 1: acquire                    # thread 2: acquire
        li   t1, 1                                         time →
try:    lw   t0, 0(s0)                                         
                                       try:    lw    t0, 0(s0)    ← T2 也读到 0
        bnez t0, try                                                   
                                               bnez  t0, try       
lock:   sw   t1, 0(s0)                                          ← T1 写 1
                                               lock:  sw   t1, 0(s0) ← T2 也写 1
```

⭐ slide 的诊断：

> **「"Gap" between memory read (lw) + memory write (sw)! Both threads think they can acquire the lock!」**
> **「"Fridge note" situation from before: Exclusive access not guaranteed!」**

```
错误根源图示：
   lw ────────── [?] ────────── sw
              ↑ 这个窗口里另一个线程也能 lw 到 0
```

---

## 七、⭐⭐ RISC-V 原子指令（Atomic Instructions）

### 7.1 为什么硬件必须插手

> **「Hardware support required to prevent an interloper (another thread) from changing the value.」**
> ⭐ **「NB: Still needed even on uniprocessor systems (because OS interrupts trigger thread context switches.)」**

即：**单核也需要** —— 因为 OS 中断随时可能切换线程，`lw` 和 `sw` 之间照样会被打断。

> **「Atomic instruction: all other instructions must happen strictly before or after the read/write」**
> **「i.e., no other access to the memory location allowed between the read and write」**

### 7.2 两条技术路线

| 方案 | 形式 | RISC-V 扩展 |
|---|---|---|
| **方案 1** | 单条原子指令：寄存器与内存交换（`amoswap`、compare-and-swap） | **Zaamo** |
| **方案 2** | 一对指令（一次读 + 一次写）在效果上原子（load-reserved / store-conditional） | **Zalrsc** |

> **「Programmers would not use these atomic instructions directly! Instead, these abstractions are used to enable implementation of parallel libraries, e.g., in assembly.」**

⭐ slide 33 明说：**「(the rest of this section is not explicitly tested, but good to have intuition about)」** —— 后两小节考试不直接考，但要建立直觉。

### 7.3 方案 1：`amoswap`（单条指令交换）

```asm
acquire:
        li            t0, 1        # t0 = 1（想要写入"锁上"）
try:    amoswap.w.aq  t0, t0, (s0) # ⭐ 原子交换：t0 ← 内存旧值，内存 ← 1
                                   #    .aq = acquire 语义（防止后续访存提前）
        bnez          t0, try      # 若旧值是 1（别人已锁）→ spin 重试
locked: # 此时已持有锁（旧值是 0，说明刚刚成功由我锁上）

release:
unlock: amoswap.w.rl  x0, x0, (s0) # ⭐ 原子写 0（x0 恒为 0）解锁
                                   #    .rl = release 语义
```

**关键点：** 交换**本身**就是「读旧值 + 写新值」不可分割 —— 一口气完成，没有 gap。

⭐ 判断逻辑：

- 旧值 = 0 → 我拿到了锁（成功的标志是「换出来的是 0」）
- 旧值 = 1 → 别人拿着，spin 重试

### 7.4 Zaamo：Atomic Memory Operations (AMOs)

> **「Atomic Memory Operations (AMOs) atomically perform an operation on an operand in memory and set the destination register to the original memory value.」**

**指令格式：** `amoINST rd, rs2, (rs1)`

| 步骤 | 动作 |
|---|---|
| 1 | 把 `rs1` 指向的值 load 进 `rd`（**返回的是旧值**） |
| 2 | 对旧值施加操作（操作数来自 `rs2`） |
| 3 | 把结果写回 `rs1` 指向的位置 |

**AMO 家族（R-Type 指令）：**

| 类别 | 操作 |
|---|---|
| 交换 | `swap` |
| 算术 | `add` |
| 位运算 | `and` / `or` / `xor` |
| 比较 | `min` / `max` / `min unsigned` / `max unsigned` |

⭐ 好处：`amoadd` 等可以**一条指令完成「读-算-写」**，不仅用于锁，也用于计数器。

### 7.5 方案 2：Load-Reserved / Store-Conditional

**动机：**

> **「A single atomic memory swap operation is hard to implement. Requires both memory read and write in a single instruction. We know from building the datapath that this is hard/inefficient!」**

（回忆 L20–L22 的数据通路：一条指令要同时完成访存读和写，非常别扭）

**思路：** 用一对指令，**效果上**原子：

| 指令 | 语义 |
|---|---|
| `lr rd, (rs1)` | Load Reserved：load 值到 `rd`，并**在某处登记一个 reservation（预约）** |
| `sc rd, rs2, (rs1)` | Store Conditional：若 reservation 仍有效，把 `rs2` 存入 `rs1` 指向处，并把 `rd` 置 **0（成功）**；若 reservation 已失效（期间该地址被改过），不存任何东西，`rd` 置 **非零（失败）** |

⭐ 精髓：**「if the pair executes successfully, then nothing else has changed the value between each instruction in the pair executing」**

**用 lr/sc 实现锁：**

```asm
        li   t2, 1             # t2 = 1（锁上值）
try:    lr   t1, s1            # ① 读锁状态 + 登记 reservation
        bne  t1, x0, try       # 若已锁（非 0）→ 重试
        sc   t0, s1, t2        # ② 条件写 1；成功 t0=0，失败 t0≠0
        bnez t0, try           # 若写失败（期间被插队）→ 重试
locked: # 成功持锁

        # … critical section …

unlock:
        sw   x0, 0(s1)         # 解锁：直接写 0
```

⭐ 注意与 `amoswap` 的区别：`sc` 会**失败**，失败就回 `try` 重来 —— 这是「乐观重试」。

### 7.6 两种方案对比

| | `amoswap`（Zaamo） | `lr`/`sc`（Zalrsc） |
|---|---|---|
| 指令数 | 1 条 | 2 条 |
| 硬件实现 | 单指令读写，数据通路难 | 拆成两条，但要跟踪 reservation |
| 失败机制 | 无失败，只是拿到旧值 | `sc` 可能失败，需循环重试 |
| 通用性 | 只能交换 | 可实现任意读-改-写（CAS、fetch-add…）|
| 常见用途 | 简单锁 | 复杂同步原语 |

⭐ 两者都**只在汇编层**使用；C 程序员用的是 `atomic` / `mutex` 抽象。

### 7.7 ⭐ `atomic` 的粒度

OpenMP 除了 `critical`，还有 **`#pragma omp atomic`**：

```c
#pragma omp atomic        // 只保护紧跟着的「单条」读-改-写语句
x++;                      // 编译成一条硬件原子指令（如 amoadd）
```

| | `#pragma omp critical` | `#pragma omp atomic` |
|---|---|---|
| 保护范围 | **整个块**（任意多行） | **只有一条语句** |
| 实现 | 锁（acquire/release） | 硬件原子指令（AMO） |
| 开销 | 大（可能阻塞、调度） | 小（无锁） |
| 适用 | 多语句必须整体互斥 | 单个计数器 / 累加操作 |

⭐ 粒度越粗 → 越安全但越慢；`atomic` 是「最小粒度的临界区」。

---

## 八、⭐ False Sharing（伪共享）

虽然 slide 本页未单列，但它是 L32 循环并行化「实测比预期慢」的主要原因之一，且直接联系 L26–L29 的 cache 知识。

```
Cache line（64 B）：
  ┌──────────────────────────────────────┐
  │ x  │ y  │ z  │ …                     │  ← 同一 cache line
  └──────────────────────────────────────┘
     ↑      ↑
   Thread0 Thread1
   写 x    写 y
```

⭐ **现象**：两个线程写的**逻辑上无关**、但**物理上同一 cache line** 的变量。缓存一致性协议（coherence）以 **cache line 为粒度** —— Thread0 写 `x` 会让 Thread1 的整行副本失效，Thread1 必须重新从别的核取（L2/L3/DRAM），即使 `y` 根本没被别人改。

| | True Sharing | **False Sharing** |
|---|---|---|
| 共享的变量 | 真同一个变量 | 不共享，仅同一行 |
| 是否必须同步 | 是 | 否（但硬件不知道） |
| 现象 | 反复 ping-pong | 同样的 ping-pong |
| 修法 | 加锁/归约 | **padding / 对齐到不同 cache line** |

```c
// 坏：counter[0..7] 都在同一 cache line
long counter[8];
#pragma omp parallel for
for (int i = 0; i < 8; i++) counter[i]++;  // 8 个线程互相踢缓存

// 好：每个线程占一行
struct padded { long v; char pad[64 - sizeof(long)]; };
struct padded counter[8];
```

---

## 九、性能实测与调度策略选择

### 9.1 一个典型实测（12 硬件线程）

| 版本 | 相对耗时 | 说明 |
|---|---|---|
| 串行 | 1.0× | 基准 |
| `parallel` 无线程切分（A） | ≈ 12.0× **慢** | 重复工作 |
| `parallel for` 无同步，独立数据 | 接近 1/10× | 理想情形 |
| `critical` 包裹 `x++` | 极慢 | 每次迭代都锁 → 几乎串行 |
| `atomic` 包裹 `x++` | 快很多 | 硬件原子指令，无阻塞 |
| `reduction(+:sum)` | 最快 | 每线程私有副本，结束才合并 |
| 有 false sharing | 明显慢于预期 | 缓存行 ping-pong |

### 9.2 同步手段选择表 ⭐

| 需求 | 首选 | 理由 |
|---|---|---|
| 多个线程累加同一变量 | **`reduction(+:v)`** | 零竞争，结束才合并 |
| 单条计数器语句 | **`#pragma omp atomic`** | 硬件原子指令，开销最小 |
| 多语句必须整体互斥 | **`#pragma omp critical`** | 语义清晰 |
| 所有线程等齐再继续 | **`#pragma omp barrier`** | 全局同步点 |
| 需要更复杂策略（超时、多把锁） | pthread mutex | CS162 内容 |

### 9.3 调度策略（承接 L32）

| 调度 | 均衡性 | 开销 | 何时用 |
|---|---|---|---|
| `static` | 差（迭代不等时） | 最低 | 循环体耗时一致 |
| `dynamic` | 好 | 高（每次领任务） | 循环体耗时差异大 |
| `guided` | 好 | 中（块递减） | 一般情况 |

⭐ 避坑清单：

1. **不要把 I/O（`printf`）放进 critical** —— 串行段立刻吃掉加速比。
2. **优先用 `reduction` 取代 `critical` 累加**。
3. **给每个线程的计数器加 padding**，防 false sharing。
4. **`schedule(dynamic)` 只在负载不均时用**，否则白付开销。
5. **用 `omp_get_wtime()` 测**，别用 `clock()`。

---

## 十、本讲小结（slide 39 原文要点）

| # | 要点 |
|---|---|
| 1 | **Data race**：多线程访问同一位置、至少一个是写、且交错发生 → 结果非确定 |
| 2 | **Synchronization** 协调线程，保证共享资源被正确访问 |
| 3 | **`#pragma omp critical`**：一次只允许一个线程执行临界区 |
| 4 | **`#pragma omp barrier`**：所有线程都到齐才继续 |
| 5 | **Lock** 提供独占：`acquire` 拿锁（否则等），`release` 放锁 |
| 6 | ⚠ **警惕 deadlock** 与串行化带来的性能损失（Amdahl's Law）|
| 7 | **原子指令**提供硬件同步支持 |
| 8 | ⛔ **朴素锁（分离的 load/store）不安全**：两个线程都可能拿到锁 |
| 9 | RISC-V 提供 **`amoswap`** 与 **`lr`/`sc`** 来实现原子操作 |
| 10 | ⭐ **Big Idea**：并行程序必须在**每一种可能的线程交错**下都正确；同步保证正确性，但限制并行性能 |

> ⭐ **「Parallel programs must be correct under every possible interleaving of thread execution.」** —— 这是整个 Parallelism 单元的收束句。

---

## 术语速查

| 术语 | 中文 | 说明 |
|---|---|---|
| **concurrency** | 并发 | 多任务时间上重叠推进 |
| **race condition** | 数据竞争 | 不同线程、同一位置、至少一写、交错 → 结果不定 |
| **data race** | 数据竞争 | race condition 的同义词 |
| **critical section** | 临界区 | 同一时刻只允许一个线程执行的代码段 |
| **mutual exclusion** | 互斥 | 同一时刻只有一个线程进入 |
| **lock** | 锁 | 独占访问抽象，acquire / release |
| **spin lock** | 自旋锁 | 拿不到锁就空转重试 |
| **acquire** | 加锁 | 拿到锁才能继续，否则等待 |
| **release** | 解锁 | 释放，让其他线程可以拿 |
| **deadlock** | 死锁 | 所有线程互相等待，无法推进 |
| **starvation** | 饿死 | 某线程永远拿不到资源 |
| **dining philosophers** | 哲学家就餐 | Dijkstra 1965，死锁经典例子 |
| **barrier** | 栅栏 | 所有线程到齐才继续 |
| **atomic** | 原子 | 不可被中断的操作 |
| **instruction atomic** | 指令原子 | 单条指令执行中不被插队 |
| **AMO** | 原子内存操作 | `amoadd`/`amoswap` 等，返回旧值 |
| **amoswap** | 原子交换 | Zaamo，单指令完成寄存器↔内存交换 |
| **lr / sc** | 加载预约 / 条件存储 | Zalrsc，成对使用，sc 可能失败 |
| **reservation** | 预约 | lr 登记，若被改则 sc 失败 |
| **Zaamo** | RISC-V 原子扩展 | Atomic Memory Operations |
| **Zalrsc** | RISC-V 原子扩展 | Load-Reserved / Store-Conditional |
| **lost update** | 丢失更新 | 后写的覆盖了先写的正确结果 |
| **false sharing** | 伪共享 | 不同变量同处一个 cache line，导致无效化抖动 |
| **cache line** | 缓存行 | 一致性协议的作用粒度（通常 64 B） |
| **Amdahl's Law** | 阿姆达尔定律 | 串行部分限制加速比上限 |
| **interleaving** | 交错 | 多线程指令交织执行的顺序 |
| **nondeterministic** | 非确定 | 同一程序多次运行结果不同 |

---

## 相关

- 课程导航：[[knowledge/course/cs61c/index\|CS61C 目录导航]]
- 上一讲：[[knowledge/course/cs61c/32-Parallelism-III-TLP-OpenMP\|L32 Parallelism III — TLP 与 OpenMP]]
- 下一讲：[[knowledge/course/cs61c/34-Parallelism-V-MIMD\|L34 Parallelism V — SIMD / MIMD]]
