---
{"dg-publish":true,"permalink":"/knowledge/course/CS61C/labs/lab03/","title":"CS61C FA26 · Lab 03 · RISC-V & Venus","tags":["CS61C","lab","lab03","RISC-V","Venus","实现方案","课程笔记"],"dg-note-properties":{"title":"CS61C FA26 · Lab 03 · RISC-V & Venus","tags":["CS61C","lab","lab03","RISC-V","Venus","实现方案","课程笔记"],"date":"2026-11-14","source":"https://cs61c.org/fa26/labs/lab03/"}}
---


# Lab 03 · RISC-V, Venus

> UC Berkeley · Great Ideas in Computer Architecture · Fall 2026
> 代码仓库：`github.com/61c-teach/fa26-lab-starter`
> 本地路径：`/home/yang/.openclaw/workspace/fa26-lab-starter/lab03/`

返回 [[knowledge/course/CS61C/labs/index\|🧪 Labs 总览]] · [[knowledge/course/CS61C/index\|📚 CS61C 目录]]

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

```
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

```
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

> ⚠️ **本题答案依赖具体题目内容**，需对照 lab spec 的 11 个问题逐条回答 —— 页面正文未完整抓取，**待补题目后再填**。

---

## Exercise 3 · Using Memcheck

**目标**：用 Venus 的 Memcheck 工具检测「内存泄漏 / 越界 / 使用未初始化内存」。

**运行方式**：

```bash
java -jar tools/venus.jar lab03/ex3_memcheck.s -m
#                                         ↑ 开启 memcheck
```

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

```
① 跑 memcheck，读报错行号和类型
② 对照上面表格判断「属于哪类问题」
③ 回答题目问的「错误类型 / 出错地址 / 修复方法」
```

> ⚠️ **同样待补具体 8 道题。**

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

```
a0 = 输入值 x
a1 = 输出数组地址（output）
返回：a0 = f(x)
```

**关键约束**：**输出值必须「从 output 数组里读」**，不能直接把常数返回。

> 这是题目刻意设计的 —— 训练「用 `lw` 从数组取数据」而不是「硬编码返回值」。

**实现思路（核心三步）**：

```
① 判断 x 是否在 [-3, 3] 范围内
② 若在 → 计算下标 = x + 3，lw 取出对应值
③ 若不在 → 返回默认值
```

**数组布局**：

```
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
    # a0 = x, a1 = &output
    li t0, -3
    blt a0, t0, out_of_range   # 下界检查
    li t0, 3
    bgt a0, t0, out_of_range   # 上界检查

    addi t0, a0, 3             # t0 = x + 3（下标）
    slli t0, t0, 2             # t0 = 下标 × 4（字节偏移）
    add  t0, a1, t0            # t0 = &output[x+3]
    lw   a0, 0(t0)             # a0 = output[x+3]  ← 返回值
    jr ra

out_of_range:
    li a0, 0                   # 越界返回 0（按题目要求调整）
    jr ra
```

**运行验证**：

```bash
java -jar tools/venus.jar lab03/ex4_discrete_fn_tester.s
```

**期望输出**：

```
f(-3) should be 6, and it is: 6
f(-2) should be 61, and it is: 61
f(-1) should be 17, and it is: 17
f(0) should be -38, and it is: -38
f(1) should be 19, and it is: 19
f(2) should be 42, and it is: 42
f(3) should be 5, and it is: 5
```

> ⚠️ **两个易错点**：
> 1. **`slli t0, t0, 2` 不能省** —— word 是 4 字节，下标要乘 4
> 2. **越界判断用 `blt`/`bgt`，不是 `bltu`/`bgtu`** —— x 是**有符号**的

---

## ⭐ Exercise 5 · Factorial（可选，核心实现）

**函数签名**：

```
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

| 项 | 原因 |
|---|---|
| **ex2 的 11 道题** | lab spec 正文未完整抓取 |
| **ex3 的 8 道题** | 同上 |
| **ex4 的越界返回值** | 需题目确认（0？-1？） |
| **ex5 是否必做** | 标为 optional |

**补齐方式**：把 lab spec 的题目文字/截图发我，再逐题补上答案。

---

## 相关

- Labs 总览：[[knowledge/course/CS61C/labs/index\|🧪 Labs]]
- 对应讲次：[[knowledge/course/CS61C/11-RISC-V-Control-Flow\|L11 控制流]] · [[knowledge/course/CS61C/12-RISC-V-Procedures\|L12 函数调用]] · [[knowledge/course/CS61C/13-RISC-V-Instruction-Formats-I\|L13 指令格式]]
