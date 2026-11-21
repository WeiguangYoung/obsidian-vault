---
{"dg-publish":true,"permalink":"/knowledge/course/cs61c-labs/Lab03/","title":"CS61C FA26 · Lab 03 · RISC-V & Venus","tags":["CS61C","lab","lab03","RISC-V","Venus","实现方案","课程笔记"],"dg-note-properties":{"title":"CS61C FA26 · Lab 03 · RISC-V & Venus","tags":["CS61C","lab","lab03","RISC-V","Venus","实现方案","课程笔记"],"date":"2026-11-14","source":"https://cs61c.org/fa26/labs/lab03/"}}
---


# Lab 03 · RISC-V, Venus

> UC Berkeley · Great Ideas in Computer Architecture · Fall 2026
> 代码仓库：`github.com/61c-teach/fa26-lab-starter`
> 本地路径：`/home/yang/repos/fa26-lab-starter/lab03/`

返回 [[knowledge/course/cs61c-labs/index\|🧪 Labs 总览]] · [[knowledge/course/cs61c/index\|📚 CS61C 目录]]

---

## 环境准备

### Venus 本地运行

```bash
# 下载工具（venus.jar + logisim-evolution.jar）
bash tools/download_tools.sh venus

# 运行单个汇编文件
java -jar tools/venus.jar lab03/ex1_hello.s

# 挂载整个目录到 Web 前端（-dm = debug mode）
java -jar tools/venus.jar . -dm

# 然后在 https://venus.cs61c.org 的终端里执行：
mount local labs
```

> ⚠️ 若提示 `port unable to be bound`，加 `--port 6162` 指定其他端口。

### 常用 ecall 编号

| a0 | 功能 | 参数 |
|---|---|---|
| **1** | **打印整数** | a1 = 值 |
| **4** | 打印字符串 | a1 = 地址 |
| **10** | **退出程序** | — |
| **11** | 打印字符 | a1 = 字符 |
| **17** | **退出（带返回值）** | a1 = 返回值 |

> ⚠️ `utils.s` 里 `malloc`/`free` 用 `li a0 0x3CC`（972 = 内存分配）+ `a6` 区分 1=分配 / 4=释放，这是 Venus 特有的扩展。

---

## 文件清单

| 文件 | 状态 | 说明 |
|---|---|---|
| `ex1_hello.s` | ✅ 已完整 | 示例，无需改 |
| `ex2_answers.txt` | ⬜ 待填 | 调试器 11 题 |
| `ex3_answers.txt` | ⬜ 待填 | Memcheck 8 题 |
| `ex3_memcheck.s` | ✅ 已完整 | malloc + 填零 |
| `ex4_discrete_fn.s` | ⬜ 待实现 | 离散函数 |
| `ex4_discrete_fn_tester.s` | 勿改 | 测试器 |
| `ex5_factorial.s` | ⬜ 待实现 | 阶乘 |
| `fib.s` | ✅ 已完整 | 斐波那契（教程用）|
| `utils.s` | 勿改 | malloc/free/exit |

---

## Exercise 1 · Venus Basics

**目标**：把本地文件系统挂载到 Venus Web 前端，验证编辑双向同步。

**核心步骤**：

```text
① 本地终端：java -jar tools/venus.jar . -dm
② 浏览器打开 https://venus.cs61c.org
③ Web 终端执行：mount local labs
④ 复制本地终端打印的 key，粘贴到浏览器弹窗
⑤ 打开 ex1_hello.s，点运行 → 输出 1234
```

**`ex1_hello.s` 为什么输出 1234**：

```asm
addi a0 x0 1       # ecall 编号 1 = 打印整数
addi a1 x0 1234    # 要打印的值
ecall              # 打印 "1234"
addi a0 x0 17      # ecall 17 = 带返回值退出
addi a1 x0 0       # 返回值 0
ecall
```

> **不需要改任何代码** —— 这道题是环境验证。

---

## Exercise 2 · Using the Venus Debugger

**目标**：用单步执行、断点、寄存器观察来回答 11 道题。

**调试器关键操作**：

| 操作 | 快捷键 / 按钮 |
|---|---|
| **单步执行（Step）** | **F6 或 Step 按钮** |
| **运行到断点** | F5 |
| **设置断点** | 行号左侧点击 |
| **观察寄存器** | 右侧 Registers 面板 |
| **观察内存** | Memory 面板（输入地址） |
| **重置** | Reset 按钮 |

**答题方法（通用套路）**：

```text
① 在关心的指令处打断点
② 运行到断点，读寄存器值
③ 单步一条，再读一次 → 算出「这条指令做了什么」
④ 对照题目问的「值 / 地址 / 变化」填答案
```

**常见问法对照**：

| 问法 | 看哪里 |
|---|---|
| 「a0 的值是多少」 | Registers → a0 |
| 「这条指令后 PC 变成多少」 | Registers → pc |
| 「内存 0x10010000 处是什么」 | Memory → 输入地址 |
| 「执行了几次循环」 | 断点命中次数计数 |
| 「为什么无限循环」 | 检查循环变量是否真的递减 |

### 官方 11 道题（原文照录）

全部基于 `fib.s`，答案写进 `ex2_answers.txt`。注意：**题号未必等于步骤号**。

| # | 题目 | 答案 |
|:--:|---|---|
| 1 | 高亮指令的机器码是多少？ | 看下方指令表，按你自己页面高亮的那行取 |
| 2 | 地址 `0x34` 处指令的机器码？ | `0x00000073`（ecall） |
| 3 | `sp` 寄存器的值？ | `0x7ffffff0` |
| 4 | 继续单步到 `t1` 的值发生变化，新的 `t1` 是多少？ | `0x00000002` |
| 5 | 此时**当前指令**的机器码？ | `0x00038313`（`mv t1, t2`） |
| 6 | 单步到地址 `0x10`（此时 `t3` 已更新），`t3` 的值？ | `0x10000000`（`n` 的地址） |
| 7 | `t3` 指向的那个**字节**是多少？ | `0x0c`（即 12） |
| 8 | 在 `0x28` 设断点并 Run 到断点，`t0` 的值？ | `0x00000001` |
| 9 | 再继续 6 次后，`t0` 的新值？ | `0x0000000d` |
| 10 | 把寄存器显示切成 Decimal 后，`t0` 的十进制值？ | `13` |
| 11 | 程序最终输出是什么？ | `144` |

**`fib.s` 的机器码对照表**（`java -jar venus.jar -d fib.s` 实测，按 4 字节递增）：

| 地址 | 机器码 | 指令 |
|:--:|:--:|---|
| 0x00 | `0x000002B3` | `add t0, x0, x0` |
| 0x04 | `0x00100313` | `addi t1, x0, 1` |
| 0x08 | `0x10000E17` | `auipc t3, 0x10000`（`la` 上半） |
| 0x0c | `0xFF8E0E13` | `addi t3, t3, -8`（`la` 下半） |
| 0x10 | `0x000E2E03` | `lw t3, 0(t3)` |
| 0x14 | `0x000E0C63` | `beq t3, x0, finish` |
| 0x18 | `0x005303B3` | `add t2, t1, t0` |
| 0x1c | `0x00030293` | `mv t0, t1` |
| 0x20 | `0x00038313` | `mv t1, t2` |
| 0x24 | `0xFFFE0E13` | `addi t3, t3, -1` |
| 0x28 | `0xFEDFF06F` | `j fib` |
| 0x2c | `0x00100513` | `addi a0, x0, 1` |
| 0x30 | `0x00028593` | `addi a1, t0, 0` |
| 0x34 | `0x00000073` | `ecall` |
| 0x38 | `0x00A00513` | `addi a0, x0, 10` |
| 0x3c | `0x00000073` | `ecall` |

> `la` 展开成 `auipc + addi` 两条，所以表里比源码多一行 —— 这就是为什么 `0x28` 落在 `j fib` 而不是源码第 10 行。
> `t0` 逐次循环取值：`1, 1, 2, 3, 5, 8, 13, 21, 34, 55, 89, 144`（第 7 次 = 13，第 12 次 = 144）。

**操作对应**（官方原文要点）：

```text
Q1–2  打开 fib.s → Simulator 标签 → 看高亮行 / 切到 0x34
Q3    Registers 标签 → 看 sp（注意别看成浮点寄存器）
Q4    反复 step，直到 t1 变化
Q6–7  单步到 0x10，读 t3；再用 Memory 标签输入 t3 的地址，Go
Q8–9  点 0x28 行设断点 → Run；再 Continue 6 次
Q10   寄存器面板底部下拉框切到 Decimal
Q11   取消断点后 Run 到结束，看控制台
```

> 💡 官方特别提醒：`prev`（回退）对 `ecall` 的效果不保证 —— 打印和退出的副作用可能撤不回来。
> 💡 Q7 问的是「一个字节」，因为后面用 `lb`（load byte）读，不是 `lw`。

---

## Exercise 3 · Using Memcheck

**目标**：用 Venus 的 Memcheck 工具检测「内存泄漏 / 越界 / 使用未初始化内存」。

**运行方式**：

```bash
java -jar tools/venus.jar -mc lab03/ex3_memcheck.s
#                      ↑ -mc = memcheck（-m 不是这个意思，传了等于没开）
```

> ⚠️ 实测确认：`venus.jar` 的 memcheck 开关是 **`-mc`**（`--memcheck`），详细模式是 **`-mcv`**。
> 写成 `-m` 不会报错，但**根本不是 memcheck**，你会看到原始的 `simulator error` 而不是 `[memcheck]` 报告。

**`ex3_memcheck.s` 的逻辑（已完整）**：

```asm
li a0 40        # 申请 40 字节 = 10 个 int
jal malloc      # utils.s：a0 = 大小，返回指针在 a0
mv t0 a0        # t0 = 数组指针
li t1 0         # t1 = i（索引）
li t2 10        # t2 = size
loop:
    sw x0 0(t0)     # 存 0
    addi t1 t1 1    # i++
    addi t0 t0 4    # 指针 +4 字节
    bge t2 t1 loop  # 若 size >= i 继续
li a0 0
jal exit
```

> ⚠️ **关键：这个程序申请了 40 字节但「从不 free」→ 这正是 memcheck 要报的「内存泄漏（memory leak）」。**

**Memcheck 常见报错与含义**：

| 报错 | 含义 | 修法 |
|---|---|---|
| **Memory leak** | **malloc 了没 free** | **末尾调 `jal free`** |
| **Out of bounds** | **访问超出 malloc 范围** | **检查偏移** |
| **Use of uninitialized** | **读到未写过的内存** | **先初始化** |
| **Double free** | **同一块 free 两次** | **只 free 一次** |
| **Free non-heap** | **free 了栈/静态地址** | **只 free malloc 的** |

**答题方法**：

```text
① 跑 memcheck，读报错行号和类型
② 对照上面表格判断「属于哪类问题」
③ 回答题目问的「错误类型 / 出错地址 / 修复方法」
```

### 官方 8 道题（原文照录）

答案写进 `ex3_answers.txt`。**题号未必等于步骤号**。

| # | 题目 | 答案（本机实测） |
|:--:|---|---|
| 1 | 程序试图访问、但导致报错的**地址**？ | `0x10008058` |
| 2 | 程序想访问**多少字节**？ | `4` |
| 3 | 开 memcheck 后，报错地址是多少？ | `0x10008080` |
| 4 | 与该错误相关的内存块**分配了多少字节**？ | `40` |
| 5 | 是源码**第几行**导致这个错误？ | `18` |
| 6 | 根据 memcheck 报告，`t1` 的值是多少？ | `10` |
| 7 | 修好越界后，程序退出时还有**多少字节没被 free**？ | `40` |
| 8 | 未被释放的内存块的**地址**？ | `0x10008058` |

> ⚠️ **Q1 与 Q3 地址不同**（`…8058` vs `…8080`）不是笔误：memcheck 会改变 malloc 的返回布局，
> 所以两次运行的块地址本来就不同。这正是题目让你对比 Q2（4 字节）与 Q4（40 字节块）的原因。

**解题脉络**（官方原文要点）：

```text
① 直接跑 → 报 simulator error → 答 Q1（地址）、Q2（字节数）
② 开 memcheck（-mc）重跑 → 报 [memcheck] 详情 → 答 Q3、Q4、Q5
③ 对比 Q2 与 Q4：memcheck 可能改变 malloc 返回的地址
④ 读报错里的循环计数 → 答 Q6（t1）
⑤ 改源码修越界（循环条件差一）→ 重跑，不再越界
⑥ 但报 unfreed memory → 答 Q7
⑦ 用 -mcv（verbose）重跑 → 答 Q8（块地址）
⑧ 末尾加 free：把数组首地址放进 a0，jal free
```

**实测参考**（本机用 `-mc` 跑原始骨架的真实输出）：

```text
[memcheck] Invalid memory access of size 4. Address 0x10008080 is 0 bytes after a block of size 40 alloc'd.
	Program Counter: 0x00000014
	File: ex3_memcheck.s:18
	Instruction: sw x0 0(t0)
```

> 💡 这里 `0 bytes after a block of size 40` 说明：`malloc(40)` 拿到 10 个 int，但循环多走了一轮，写到第 11 个位置（偏移 40）→ 正好越界。
> 💡 修法：`bge t2 t1 loop` 的边界条件差一，应改成 `bge t1 t2 loop` 或调整计数起点。
> 💡 修完越界后仍会报 **40 字节未释放**，因为全程没有 `free`。

---

## ⭐ Exercise 4 · Discrete Function（核心实现）

**测试器给出的期望值**：

| 输入 | 期望输出 |
|---|---|
| **f(-3)** | **6** |
| **f(-2)** | **61** |
| **f(-1)** | **17** |
| **f(0)** | **-38** |
| **f(1)** | **19** |
| **f(2)** | **42** |
| **f(3)** | **5** |

**函数签名**：

```text
a0 = 输入值 x
a1 = 输出数组地址（output）
返回：a0 = f(x)
```

**关键约束（官方原文，务必看清）**：

> Implement the function in `ex4_discrete_fn.s` in RISC-V, **with the condition that your code may NOT use any branch and/or jump instructions!**

两条硬性要求：

1. **不得使用任何分支 / 跳转指令** —— 不能出现 `beq` / `bne` / `blt` / `bge` / `bltu` / `bgeu` / `bgt` / `ble` / `j` / `jal` / `jalr`。骨架末尾自带的那行 `jr ra` 是官方给的返回出口，保留。
2. **输出值必须从 `output` 数组 `lw` 取**，不能把常数直接退回。

> 真正考点：**用纯算术表达「按输入取值」**，而不是 if/else。因为 f 的定义域恰好是 7 个连续整数，`下标 = x + 3` **无条件成立** —— 根本不需要越界判断。

**实现思路**：

```text
下标     = x + 3        （addi）
字节偏移 = 下标 × 4      （slli）
地址     = output + 偏移 （add）
返回值   = lw
```

**数组布局**：

```text
output 数组（.word 6, 61, 17, -38, 19, 42, 5）

索引:  0    1    2    3    4    5    6
值:    6   61   17  -38   19   42    5
对应 x: -3  -2   -1    0    1    2    3
```

> ⭐ **下标 = x + 3**（把 -3 映射到 0）。

**汇编实现**：

```asm
.globl f
f:
    addi t0, a0, 3             # t0 = x + 3（下标，无条件成立）
    slli t0, t0, 2             # t0 = 下标 × 4（字节偏移）
    add  t0, a1, t0            # t0 = &output[x+3]
    lw   a0, 0(t0)             # a0 = output[x+3]  ← 返回值
    jr ra
```

四条指令，零分支。

**运行验证**：

```bash
java -jar tools/venus.jar lab03/ex4_discrete_fn_tester.s
```

**期望输出**：

```text
f(-3) should be 6, and it is: 6
f(-2) should be 61, and it is: 61
f(-1) should be 17, and it is: 17
f(0) should be -38, and it is: -38
f(1) should be 19, and it is: 19
f(2) should be 42, and it is: 42
f(3) should be 5, and it is: 5
```

> ⚠️ **三个易错点**：
> 1. **别写分支** —— 本题最大的坑。加 `blt`/`bgt` 做「越界检查」输出仍能对上（测试器只比对打印结果，不检查用了什么指令），但**直接违反题目硬性要求**，判不合规。
> 2. **`slli t0, t0, 2` 不能省** —— word 是 4 字节，下标要乘 4。
> 3. **`addi t0, a0, 3` 是算术不是跳转** —— 常数 `3` 来自定义域起点 -3，题目数据变了要同步改。

---

## ⭐ Exercise 5 · Factorial（可选，核心实现）

**函数签名**：

```text
a0 = n
返回：a0 = n!
```

### 方案 A：迭代（推荐，无栈操作）

```asm
factorial:
    li t0, 1           # t0 = 结果 accumulator = 1
    li t1, 1           # t1 = i = 1
loop:
    bgt t1, a0, done   # if i > n: 结束
    mul t0, t0, t1     # 结果 *= i
    addi t1, t1, 1     # i++
    j loop
done:
    mv a0, t0          # 返回值放 a0
    jr ra
```

> ⚠️ **Venus 支持 `mul`**（RV32M 扩展）。

### 方案 B：递归（需要栈，符合课程进度）

```asm
factorial:
    li t0, 1
    ble a0, t0, base       # 基例：n <= 1

    addi sp, sp, -8        # 保存现场
    sw ra, 4(sp)
    sw a0, 0(sp)

    addi a0, a0, -1
    jal ra, factorial      # 递归 factorial(n-1)

    lw t1, 0(sp)           # 恢复 n
    mul a0, a0, t1         # n * (n-1)!

    lw ra, 4(sp)
    addi sp, sp, 8
    jr ra

base:
    li a0, 1
    jr ra
```

> ⭐ **方案 B 是「函数调用 + 栈帧」的完整练习**，直接对应 L12 的 calling convention。

**运行验证**：

```bash
java -jar tools/venus.jar lab03/ex5_factorial.s
# 期望输出：40320（8! = 40320）
```

> ⚠️ **`n = 8` → `40320`，在 32 位有符号范围内。**
> **若改成 13 以上，`13! = 6227020800` 会溢出。**

---

## fib.s（教程示例，已完成）

**这是 Exercise 1 里逐行讲解的代码**，无需修改。

```asm
add t0, x0, x0     # curr_fib = 0
addi t1, x0, 1     # next_fib = 1
fib:
    beq t3, x0, finish
    add t2, t1, t0     # new = curr + next
    mv t0, t1          # curr = next
    mv t1, t2          # next = new
    addi t3, t3, -1    # i--
    j fib
```

**对应 C 代码**：

```c
int curr_fib = 0, next_fib = 1;
for (int i = n; i != 0; i--) {
    int new_fib = curr_fib + next_fib;
    curr_fib = next_fib;
    next_fib = new_fib;
}
printf("%d\n", curr_fib);
```

**为什么 `addi t3, t3, -1` 放在末尾**：

> **课件原话：「`i` 在循环体里要用。如果紧跟 `beq` 之后就更新，循环体里拿到的值就不对了。」**

**运行结果**：`n = 12` → 输出 `144`

---

## 待补内容

| 项 | 状态 |
|---|---|
| **ex2 的 11 道题** | ✅ 已从官方页面抓全，见上文「官方 11 道题」 |
| **ex3 的 8 道题** | ✅ 已从官方页面抓全，见上文「官方 8 道题」 |
| **ex4 的越界返回值** | ✅ 不存在越界分支 —— 官方禁止分支/跳转，`x+3` 无条件成立 |
| **ex5 是否必做** | ✅ 官方标题即 `[Optional] Exercise 5: Factorial`，**选做** |

**剩余开放项**：无。

---

## 相关

- Labs 总览：[[knowledge/course/cs61c-labs/index\|🧪 Labs]]
- 对应讲次：[[knowledge/course/cs61c/11-RISC-V-Control-Flow\|L11 控制流]] · [[knowledge/course/cs61c/12-RISC-V-Procedures\|L12 函数调用]] · [[knowledge/course/cs61c/13-RISC-V-Instruction-Formats-I\|L13 指令格式]]
