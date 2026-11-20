---
{"dg-publish":true,"permalink":"/knowledge/course/cs61c-labs/Lab04/","title":"CS61C FA26 · Lab 04 · RISC-V 调用约定","tags":["CS61C","lab","lab04","RISC-V","调用约定","栈帧","汇编","课程笔记"],"dg-note-properties":{"title":"CS61C FA26 · Lab 04 · RISC-V 调用约定","tags":["CS61C","lab","lab04","RISC-V","调用约定","栈帧","汇编","课程笔记"],"date":"2026-11-20","source":"https://cs61c.org/fa26/labs/lab04/"}}
---


# Lab 04 · RISC-V 调用约定

> UC Berkeley · Great Ideas in Computer Architecture · Fall 2026
> 代码仓库：`github.com/61c-teach/fa26-lab-starter`
> 本地路径：`/home/yang/.openclaw/workspace/fa26-lab-starter/lab04/`

返回 [[knowledge/course/cs61c-labs/index\|🧪 Labs 总览]] · [[knowledge/course/cs61c/index\|📚 CS61C 目录]]

---

## 文件清单

> **本 Lab 的核心**：Lab 03 只写「一个函数内部」的代码，寄存器随便用；Lab 04 开始**多个函数互相调用**，同一组 32 个寄存器被 caller 和 callee 共用，于是需要一条规则 —— **调用约定 = 「谁改谁负责」**。三个 Exercise 全部围绕它展开。

| 文件 | 状态 | 说明 |
|---|---|---|
| `ex1.s` | ⬜ 待修 | 三个函数 `pow` / `inc_arr` / `helper_fn` 有多处 CC 违规 |
| `ex2.s` | ⬜ 待修 | 递归 `pow`，错在 **`s` 寄存器**与 **`sp`** 的用法（只能加行） |
| `ex3.s` | ⬜ 待修 | 递归 `pow`，错在 **`t` 寄存器**与 **`ra`** 的用法（只能加行） |
| `example_c_to_riscv.s` | ✅ 已完整 | 讲义示例：C 数组求和 → RISC-V，含完整 prologue/epilogue |
| `.gitignore` | 勿改 | — |

> ⚠️ `ex2.s` / `ex3.s` 的题面明确写着 **「You may only add lines」** —— 不能删原行、不能改原行，只能插入。

---

## 运行环境

```bash
java -jar tools/venus.jar -cc lab04/ex1.s     # -cc = calling convention check
java -jar tools/venus.jar lab04/ex2.s         # ex2/ex3 直接跑，看输出对不对
```

> ⚠️ **CC checker 只检查带 `.globl` 导出的函数**。这是本 Lab 一个教学点（Exercise 1.1 第 2 问）。
> ⚠️ CC checker 不完备 —— 它只能查「`s` 未存就改」「`ra` 未还原」「`sp` 不平衡」这类结构错误。

---

## 一、调用约定总表（必背）

### 1.1 寄存器角色对照

| 类别 | 寄存器 | ABI 名 | 谁负责保存 | 典型用途 |
|---|---|---|---|---|
| **参数 / 返回值** | x10–x17 | **`a0`–`a7`** | **Caller-saved** | 前 8 个参数；`a0`/`a1` 放返回值 |
| **返回地址** | x1 | **`ra`** | **Caller-saved** | `jal` 自动写入 |
| **临时寄存器** | x5–x7, x28–x31 | **`t0`–`t6`** | **Caller-saved** | 中间计算结果 |
| **保存寄存器** | x8–x9, x18–x27 | **`s0`–`s11`** | **Callee-saved** | 跨调用要保留的变量 |
| **栈指针** | x2 | **`sp`** | 特殊（必须平衡） | 指向栈顶 |
| **零寄存器** | x0 | `zero` | — | 恒为 0 |
| **全局指针** | x3 | `gp` | 勿动 | 静态数据基址 |

### 1.2 两条铁律

```text
Caller-saved  (a0-a7, t0-t6, ra)
    callee 可以随便改，改坏不管
    → caller 若在调用后还要用，必须【自己】在 jal 前存栈、jal 后取回

Callee-saved  (s0-s11, sp)
    callee 若要改，必须【自己】在 prologue 存、epilogue 还原
    → caller 可以直接信任：调用前后值不变
```

> **记忆口诀**：**`t` 是「临时」→ 用完就没，自己不存怪自己；`s` 是「saved」→ 名字就叫保存，谁改谁存。**

`ra` 虽然在名义上属于 caller-saved，但**实践中几乎总在 prologue 里保存** —— 因为函数只要内部有 `jal`，`ra` 就必被覆盖（见第三节）。

### 1.3 传参与返回

```asm
# 调用者：
li  a0, 2          # 第 1 个参数
li  a1, 7          # 第 2 个参数
jal ra, pow        # ra = 下一条指令地址，跳到 pow

# 被调者返回时：
mv  a0, s0         # 返回值放 a0（第二个返回值放 a1）
jr  ra             # 跳回 ra
```

| 位置 | 含义 |
|---|---|
| `a0`–`a7` | 第 1–8 个参数（更多参数走栈） |
| `a0` | 主返回值 |
| `a1` | 次返回值（例如 64 位结果的高位） |

---

## 二、栈帧布局

### 2.1 方向与结构

栈**向低地址增长**。分配 = `sp` 减小，释放 = `sp` 增大。

```text
        高地址
     ┌──────────────┐
     │  caller 的栈帧 │
     ├──────────────┤ ← caller 的 sp（= 进入本函数时的 sp）
     │  ra           │  ← 0(sp)     （若本函数还会调用别人）
     │  s0           │  ← 4(sp)
     │  s1           │  ← 8(sp)
     │  局部变量 ...   │
     ├──────────────┤ ← 本函数的 sp（addi sp, sp, -N）
     │              │
     │   未使用 /     │
     │   更深的调用    │
     │              │
     └──────────────┘
        低地址
```

> `sp` **增长方向是「向下」**；图中箭头向下 = 分配更多空间。
> ⚠️ **栈帧大小应为 8 的倍数**（课程代码里 4 的倍数也能跑，正规写法取 8/16 对齐）。

### 2.2 标准序言 / 尾声模板

```asm
# ============ PROLOGUE ============
func:
    addi sp, sp, -16     # ① 开栈帧（大小 = 保存寄存器数 × 4，取 8 倍数）
    sw   ra, 12(sp)      # ② 若函数内部有 jal，必须存 ra
    sw   s0, 8(sp)       # ③ 存所有会被改写的 s 寄存器
    sw   s1, 4(sp)
    # 局部变量或额外的 t 寄存器槽位放 0(sp) / 更低位

    # ... 函数体 ...

# ============ EPILOGUE ============
    lw   s1, 4(sp)       # ④ 逆序还原
    lw   s0, 8(sp)
    lw   ra, 12(sp)
    addi sp, sp, 16      # ⑤ 关栈帧（sp 必须回到入口值）
    jr   ra              # ⑥ 返回
```

**要点**：

- ①②③ 与 ④⑤⑥ **严格镜像**，`sp` 进出必须相等；
- 如果函数是**叶子函数**（不再 `jal`），可**不存 `ra`**；
- 如果函数**不改任何 `s` 寄存器**，也可以不存 `s`；
- 多个返回出口（如早退）时，epilogue 要么复制一份，要么统一 `j` 到一个出口。

### 2.3 讲义里的地址计算例题

设 `func1` 入口 `sp = 0x7FFFFFF0`：

```asm
addi sp sp -8   # sp = 0x7FFFFFF8
sw ra 0(sp)     # ra → 0x7FFFFFF8
sw s0 4(sp)     # s0 → 0x7FFFFFFC

addi sp sp -4   # sp = 0x7FFFFFF4
sw t0 0(sp)     # t0 → 0x7FFFFFF4

jal ra func2
lw t0 0(sp)
addi sp sp 4    # sp = 0x7FFFFFF8

lw s0 4(sp)
lw ra 0(sp)
addi sp sp 8    # sp = 0x7FFFFFF0  ✅ 回到入口值
jr ra
```

| 问题 | 答案 |
|---|---|
| `s0` 存在哪 | `0x7FFFFFF0 - 8 + 4 = 0x7FFFFFEC` |
| `t0` 存在哪 | `0x7FFFFFF0 - 8 - 4 = 0x7FFFFFE4` |
| `jr ra` 时 `sp` | `0x7FFFFFF0`（与入口相同） |

---

## 三、递归为什么必须保存 `ra`

递归是「同一个函数调用自己」，所以 `ra` 会被**同一个函数覆盖多次**。

### 3.1 没有存 `ra` 会发生什么

```asm
pow:                      # 假设 a0=2, a1=2
    beq a1, zero, base
    addi a1, a1, -1
    jal  ra, pow          # ← ra 被改成「这条指令的下一条」
    mul  a0, a0, s0       # 这一行就是「下一条」
    jr   ra               # ← 跳回哪？
```

追踪调用链：

```text
pow(2,2)  ra = main 里的返回点
   jal → ra = pow 内「mul a0,a0,s0」的地址
pow(2,1)  ra = mul 地址（覆盖了 main 的返回点！）
   jal → ra = 还是「mul a0,a0,s0」的地址
pow(2,0)  ra = mul 地址
   base: li a0,1; jr ra  → 跳回「mul a0,a0,s0」，结果乘一次后继续往下…
```

最后一层 `jr ra` 直接跳回中间那条 `mul`，**整个上层调用栈全被丢掉** —— 表现是输出错误结果、或陷入死循环、或「immutable memory 被写」这类诡异报错。

### 3.2 正确做法

```asm
pow:
    addi sp, sp, -8
    sw   ra, 4(sp)        # ★ 每层递归自己的 ra 存自己的一层栈
    sw   s0, 0(sp)        # 同时保存要跨调用保留的 s0
    beq  a1, zero, base
    mv   s0, a0           # 保存本层的 a0（caller-saved，必须自己保管）
    addi a1, a1, -1
    jal  ra, pow          # 这层 ra 被改，但已经存好了
    mul  a0, a0, s0
    j    done
base:
    li   a0, 1
done:
    lw   s0, 0(sp)
    lw   ra, 4(sp)        # ★ 取回本层的 ra
    addi sp, sp, 8
    jr   ra
```

```text
调用栈（向下增长）：
pow(2,2) 的帧:  [s0=2][ra=main 返回点]   ← sp
pow(2,1) 的帧:  [s0=2][ra=pow 内 mul]    ← sp
pow(2,0) 的帧:  [s0=?][ra=pow 内 mul]    ← sp
```

每层一个独立槽位，逐层弹出，**`ra` 自然不冲突**。
> **这就是「递归必须存 `ra`」的全部理由** —— 不是约定在管它，是物理上只有一个 `ra` 寄存器。

---

## ⭐ Exercise 1 · Calling Convention Checker

**目标**：借助 CC checker 的报错，把 `ex1.s` 里三个函数的调用约定全部修对。

### 1.1 读代码：先看清结构

| 符号 | 是函数吗 | 依据 |
|---|---|---|
| `pow` | ✅ 是 | 在 `.globl` 中导出，被 `jal ra pow` 调用 |
| `inc_arr` | ✅ 是 | 在 `.globl` 中导出，被 `jal ra inc_arr` 调用 |
| `check_arr` | ✅ 是 | 被 `jal ra check_arr` 调用（`jr ra` 返回） |
| `helper_fn` | ✅ 是 | 被 `jal ra helper_fn` 调用 |
| `pow_loop` / `pow_end` | ❌ **不是** | 只是跳转目标，用 `j` 跳，不需要「返回」 |
| `inc_arr_loop` / `inc_arr_end` | ❌ 不是 | 同上 |
| `next_test` / `failure` / `continue` | ❌ 不是 | `beq` / `j` 的目标；`failure` 结尾是 `ecall` 退出，**永不返回** |

> **判断标准**：**用 `jal` 跳进去、用 `jr ra` 跳出来 = 函数；用 `j`/`beq` 跳来跳去 = 标签。** 标签不参与调用约定。

**为什么 `helper_fn` 的 bug 一开始可能不报**：它没有出现在 `.globl` 里 → CC checker 默认不检查它。等 `inc_arr` 修好、`helper_fn` 变成「被正确定义的函数调用」之后，`helper_fn` 自身的问题才可能被暴露（或彻底查不出来）—— 所以题面要求你**主动修**它。

### 1.2 报错解读

```text
[CC Violation] ... Setting of a saved register (s0) which has not been saved! ex1.s:54 li s0, 1
[CC Violation] ... Save register s0 not correctly restored before return!  Expected 0x00000000, Actual 0x00000080. ex1.s:65 jr ra
[CC Violation] ... Setting of a saved register (s0) ... NOT saved! ex1.s:79 mv s0, a0
[CC Violation] ... Setting of a saved register (s1) ... NOT saved! ex1.s:80 mv s1, a1
[CC Violation] ... Setting of a saved register (s0) ... NOT saved! ex1.s:117 addi s0, t1, 1
```

| 报错类型 | 中文含义 | 修法 |
|---|---|---|
| `Setting of a saved register ... not been saved` | **写了 `s` 寄存器但没先存栈** | 加 prologue `sw` |
| `Save register ... not correctly restored before return` | **返回时 `s` 寄存器值不等于入口值** | 加 epilogue `lw` |
| `StoreError: immutable at address ...` | 不只 CC 错，还写了 `.text` 常量区 → 逻辑跑飞 | 修好 CC 后自然消失 |

### 1.3 `pow`：缺 prologue / epilogue

**待填位置**：`# BEGIN PROLOGUE` 与 `# END PROLOGUE` 之间；`# BEGIN EPILOGUE` 与 `# END EPILOGUE` 之间。

`pow` 是**叶子函数**（内部没有 `jal`）→ **不用存 `ra`**，只需保存它改写的 `s0`。

```asm
pow:
    # BEGIN PROLOGUE
    addi sp, sp, -4      # 只需要 1 个槽位
    sw   s0, 0(sp)       # ★ s0 是 callee-saved，改了就要存
    # END PROLOGUE
    li s0, 1
pow_loop:
    beq a1, zero, pow_end
    mul s0, s0, a0
    addi a1, a1, -1
    j pow_loop
pow_end:
    mv a0, s0            # 返回值放 a0
    # BEGIN EPILOGUE
    lw   s0, 0(sp)       # ★ 还原
    addi sp, sp, 4
    # END EPILOGUE
    jr ra
```

> ⚠️ 严格 ABI 下 `sp` 应 8 或 16 字节对齐；Venus 的 CC checker 对 4 字节也能过，但**写成 `-8` / `+8` 更规范**：
> ```asm
> addi sp, sp, -8
> sw   s0, 0(sp)
> ...
> lw   s0, 0(sp)
> addi sp, sp, 8
> ```

### 1.4 `inc_arr`：caller + callee 双重身份

`inc_arr` 同时是**被调者**（`main` 调它）和**调用者**（它调 `helper_fn`）：

| 职责 | 要保存什么 | 位置 |
|---|---|---|
| callee 职责 | 改写的 `s0`、`s1` | prologue / epilogue |
| callee 职责 | `ra`（因为内部有 `jal`） | prologue / epilogue（骨架已给） |
| caller 职责 | **`t0`**（循环计数器，`jal` 后还要用） | `jal` 前后 |

**为什么 `t1` 不用存**：`t1` 在 `jal` 之前算完、`jal` 之后**不再使用**（每轮循环开头重新算）。`t0` 则跨过 `jal` 还要读，所以必须保。

```asm
inc_arr:
    # BEGIN PROLOGUE
    addi sp, sp, -12     # 骨架原本是 -4，扩成 3 个槽：ra / s0 / s1
    sw   ra, 0(sp)
    sw   s0, 4(sp)       # ★ 补
    sw   s1, 8(sp)       # ★ 补
    # END PROLOGUE
    mv s0, a0
    mv s1, a1
    li t0, 0
inc_arr_loop:
    beq t0, s1, inc_arr_end
    slli t1, t0, 2
    add a0, s0, t1
    # Prepare to call helper_fn
    addi sp, sp, -4      # ★ caller 职责：存 t0
    sw   t0, 0(sp)
    jal ra helper_fn
    lw   t0, 0(sp)       # ★ 取回
    addi sp, sp, 4
    addi t0, t0, 1
    j inc_arr_loop
inc_arr_end:
    # BEGIN EPILOGUE
    lw   s1, 8(sp)       # ★ 补
    lw   s0, 4(sp)       # ★ 补
    lw   ra, 0(sp)
    addi sp, sp, 12
    # END EPILOGUE
    jr ra
```

> ⚠️ 为什么 `ra` 在 `inc_arr` 里要存、而 `pow` 里不用？
> **`inc_arr` 内部有 `jal ra helper_fn`** → `ra` 被覆盖 → 必须存。
> `pow` 内部没有任何 `jal` → `ra` 从头到尾没变 → 不用存。

### 1.5 `helper_fn`：藏起来的 `s0` 写入

```asm
helper_fn:
    lw   t1, 0(a0)
    addi s0, t1, 1       # ← 偷偷改写 s0，且没存
    sw   s0, 0(a0)
    jr   ra
```

**修法**：不应改 `s0`，改用临时寄存器 `t1` 即可 —— 这是最干净的写法：

```asm
helper_fn:
    # BEGIN PROLOGUE
    # 无需保存：改用 t 寄存器后不再触碰任何 s 寄存器
    # END PROLOGUE
    lw   t1, 0(a0)
    addi t1, t1, 1       # ★ 只需要 +1，不需要额外寄存器
    sw   t1, 0(a0)
    # BEGIN EPILOGUE
    # END EPILOGUE
    jr ra
```

> ⚠️ 但题面要求「**只在 FIXME 处加行**」，不改原有的 `addi s0, t1, 1` / `sw s0, 0(a0)`。
> 那就保留原行，在 prologue/epilogue 里把 `s0` 存好：
> ```asm
> helper_fn:
>     # BEGIN PROLOGUE
>     addi sp, sp, -8
>     sw   s0, 0(sp)
>     # END PROLOGUE
>     lw t1, 0(a0)
>     addi s0, t1, 1
>     sw s0, 0(a0)
>     # BEGIN EPILOGUE
>     lw   s0, 0(sp)
>     addi sp, sp, 8
>     # END EPILOGUE
>     jr ra
> ```

### 1.6 验收

```bash
java -jar tools/venus.jar -cc lab04/ex1.s
```

期望：

```text
Tests passed.
Found 0 warnings!
```

**每题必答的四个问题**：

| 问题 | 答案 |
|---|---|
| `next_test` 是函数吗 | **不是**，只是 `beq` 的跳转目标，没有 `jal`/`jr ra` 配对 |
| `helper_fn` 的错为何不报 | **没有 `.globl`**，CC checker 只检查导出函数 |
| `pow` 的报错根因 | **缺 prologue 和 epilogue**（`s0` 未存未还原） |
| 跳到 `pow_loop` / `pow_end` 要不要守约定 | **不要** —— 它们不是函数，不需要返回 |
| `inc_arr` 报错根因 | 未存 `s0`/`s1`；调用 `helper_fn` 前未存 `t0` |
| 为什么只有 `inc_arr` 存 `ra` | 只有它内部有 `jal`，`ra` 会被覆盖 |

---

## ⭐ Exercise 2 · Fixing CC Errors: `s` Edition

**规则**：只能加行，不能改/删原行。

### 2.1 先读错在哪

```asm
ex2:
    sw s0 0(sp)          # ← 问题 1：往 sp 写，但 sp 根本没动过！
    beq a1 x0 ex2_zero_case
    mv s0 a0
    addi a1 a1 -1
    jal ra ex2           # ← 问题 2：ra 没存，递归会覆盖
    mul a0 a0 s0
    j ex2_end
ex2_zero_case:
    li a0 1
ex2_end:
    lw s0 0(sp)          # ← 问题 3：sp 也没加回去
    jr ra
```

### 2.2 三个 bug

| # | bug | 后果 |
|---|---|---|
| ① | **`sp` 没有先减小** | `sw s0 0(sp)` 写到了 **caller 的栈帧顶部**，直接覆盖别人数据 |
| ② | **`ra` 未保存** | 递归第二层起 `ra` 指向 `mul a0 a0 s0`，`jr ra` 跳回中间 → 结果错 / 死循环 |
| ③ | **`sp` 没有恢复** | 即使 ① 修好，`sp` 也回不到入口值，栈永久下移 |

> 注意题目只说「`s` Edition」，但修 `s` 的前提是把 `sp` 用对 —— 所以 `sp` 的增删是必须补的。

### 2.3 参考修正（只加行）

```asm
ex2:
    # Note: Add code BELOW without altering existing lines.
    addi sp sp -8        # ★① 开栈帧（8 字节 = 2 槽，8 对齐）
    sw   ra 4(sp)        # ★② 递归必须存 ra
    sw s0 0(sp)          # （原行，现在 sp 已经正确了）

    beq a1 x0 ex2_zero_case

    mv s0 a0
    addi a1 a1 -1
    jal ra ex2
    mul a0 a0 s0
    j ex2_end

ex2_zero_case:
    li a0 1

ex2_end:
    lw   ra 4(sp)        # ★②' 还原 ra
    lw s0 0(sp)          # （原行）
    addi sp sp 8         # ★③ 关栈帧，sp 回到入口值
    jr ra
```

> ⚠️ **必须同时加 `addi sp sp -8` 与 `addi sp sp 8`**，且**地址不重叠**：`ra` 用 `4(sp)`，`s0` 用 `0(sp)`。
> 如果只加 `-8` 不给 `ra` 槽，`ra` 与被存的 `s0` 挤在同一位，递归返回时 `ra` 就是垃圾。

### 2.4 为什么用 `s0` 存 `a0` 是对的

`a0` 是 caller-saved —— 递归调用 `jal ra ex2` 后它必然被 callee 用来当参数。所以「把 `a0` 存到 `s0`」是**正确解法**，因为 `s0` 是 callee-saved，本函数的 callee 会替它保留。这也是 **Ex2 教学点**：`s` 寄存器就是为这种「跨调用要保留的值」准备的。

**验证**：

```bash
java -jar tools/venus.jar lab04/ex2.s
# 期望输出：1024   （2^10）
```

---

## ⭐ Exercise 3 · Fixing CC Errors: `t` Edition

**同样的递归 `pow`，但这次错误在 `t` 和 `ra` 上。**

### 3.1 先读错在哪

```asm
ex3:
    beq a1 x0 ex3_zero_case
    mv t0 a0          # ← 把 a0 存进 caller-saved 的 t0
    addi a1 a1 -1
    jal ra ex3        # ← t0 不保证保留；ra 也被覆盖
    mul a0 a0 t0      # ← 这里读到的 t0 可能是垃圾
    j ex3_end
ex3_zero_case:
    li a0 1
ex3_end:
    jr ra
```

| # | bug | 后果 |
|---|---|---|
| ① | **`ra` 未保存** | 与 Ex2 相同，递归返回点被覆盖 |
| ② | **用 `t0` 保存 `a0`** | `t0` 是 caller-saved，下一层递归可以随便改它 → 乘法因子错乱 |
| ③ | **完全没有栈帧** | 无处保存 |

> 这就是 Ex3 的「`t` Edition」：**别用临时寄存器保存跨调用的值**。

### 3.2 修法思路

要救回来的东西有两个：

1. **`ra`** → 直接存栈（每层一个槽）；
2. **跨调用的因子 `a0`** → 不能留在 `t0`，必须搬进 **`s` 寄存器**（本函数内自存自还原）。

但原行是 `mul a0 a0 t0`，读的是 `t0`。所以在 `jal` 之后、`mul` 之前，要**把 `s0` 里的因子搬回 `t0`**，让原行拿到正确值。

### 3.3 参考修正（只加行，能跑出 1024）

```asm
ex3:
    # Note: Add code BELOW without altering existing lines.
    addi sp sp -16       # ★ 开栈帧（4 槽，8 的倍数）
    sw   ra 12(sp)       # ★ ra：递归必须存
    sw   s0 8(sp)        # ★ s0：本函数要用，callee-saved 自己保
    sw   t0 4(sp)        # ★ 顺手保住 t0 原值，保证进出对称

    beq a1 x0 ex3_zero_case

    mv t0 a0
    mv s0 a0             # ★ 关键：因子备份到 callee-saved
    addi a1 a1 -1
    jal ra ex3           # 下层会改坏 t0，但 s0 完好
    mv t0 s0             # ★ 把因子搬回 t0，让原行 mul 拿到正确值

    mul a0 a0 t0
    j ex3_end

    # Note: Add code ABOVE without altering existing lines.

ex3_zero_case:
    li a0 1

ex3_end:
    lw   s0 8(sp)        # ★ 还原 s0
    lw   ra 12(sp)       # ★ 还原 ra
    lw   t0 4(sp)
    addi sp sp 16        # ★ 关栈帧，sp 回到入口值
    jr ra
```

> ⚠️ **为什么不能只存 `t0`**：`sw t0` 在 `jal` **之后**执行时，`t0` 早已被下层递归改掉，存进去的是垃圾。
> 加上 `addi sp sp -16` / `+16` 与三组 `sw`/`lw`，`sp` 与 `ra` 都平衡，`s0` 也归还 —— 这是「只加行」约束下唯一能跑对的组合。

**验证**：

```bash
java -jar tools/venus.jar lab04/ex3.s
# 期望输出：1024   （2^10）
```

### 3.4 Ex2 vs Ex3 对照

| | Ex2（`s` 版） | Ex3（`t` 版） |
|---|---|---|
| 跨调用的因子存在哪 | `s0`（**本来就对**） | `t0`（**错**） |
| 需要修的 | `sp` + `ra` | `sp` + `ra` + 把因子搬进/搬出 `s` |
| 教学点 | `s` 寄存器就是干这个的 | `t` 寄存器**不能**干这个 |
| 栈槽需求 | 8 字节（ra + s0） | 16 字节（ra + s0 + t0 + 对齐） |

---

## 四、讲义示例：C → RISC-V 全流程

`example_c_to_riscv.s` 是完整可跑的范本，值得逐块对照。

```c
int source[] = {3, 1, 4, 1, 5, 9, 0};
int dest[10];
int fun(int x) { return -x * (x + 1); }

int main() {
    int k;
    int sum = 0;
    for (k = 0; source[k] != 0; k++) {
        dest[k] = fun(source[k]);
        sum += dest[k];
    }
    printf("sum: %d\n", sum);
}
```

### 4.1 `fun`：叶子函数，零栈操作

```asm
fun:
    addi t0, a0, 1     # t0 = x + 1
    sub  t1, x0, a0    # t1 = -x
    mul  a0, t0, t1    # a0 = (x+1) * (-x)
    jr   ra
```

> **为什么不用存 `t1`**：`t1` 是 caller-saved。`fun` 作为 callee 有权改它；若 caller 在 `fun` 之后还要用 `t1`，那该 **caller** 自己存。

### 4.2 `main`：五个寄存器要保

`main` 改了 `s0`（sum）、`s1`（&source）、`s2`（&dest）、`s3`（byte offset），且内部有 `jal fun` → 还要存 `ra`。

```asm
main:
    # BEGIN PROLOGUE
    addi sp, sp, -20
    sw   s0, 0(sp)
    sw   s1, 4(sp)
    sw   s2, 8(sp)
    sw   s3, 12(sp)
    sw   ra, 16(sp)
    # END PROLOGUE
    ...
exit:
    ...
    # BEGIN EPILOGUE
    lw   s0, 0(sp)
    lw   s1, 4(sp)
    lw   s2, 8(sp)
    lw   s3, 12(sp)
    lw   ra, 16(sp)
    addi sp, sp, 20
    # END EPILOGUE
```

> ⚠️ `-20` 不是 8 的倍数，课程代码这样写能跑（Venus 不强制对齐），但**规范写法是 `-24` 或其他 8/16 倍数**。

### 4.3 只存 `t0`，不存 `t1`/`t2`/`t3`

```asm
    add a0, x0, t2          # 传参
    addi sp, sp, -4
    sw   t0, 0(sp)          # ★ 只有 t0 跨调用还要用
    jal  fun
    lw   t0, 0(sp)
    addi sp, sp, 4

    add  t3, s2, s3         # t3 在 jal 之后才第一次写 → 旧值无所谓
    sw   a0, 0(t3)
    add  s0, s0, a0
```

| 寄存器 | `jal` 前用过？ | `jal` 后用？ | 要存吗 |
|---|---|---|---|
| `t0`（k） | ✅ | ✅ | **要** |
| `t1`（源地址） | ✅ | ❌ 每轮重算 | 不要 |
| `t2`（源值） | ✅ | ❌ | 不要 |
| `t3`（目标地址） | ❌ | ✅ 但先写后读 | 不要 |
| `a0` | ✅（传参） | ✅（返回值） | 不要 —— 返回值本来就该从 `a0` 拿 |
| `s3`（offset） | ✅ | ✅ | 但它只被 **callee 改**，不会 | 它是 callee-saved，`fun` 保证不动它 |

> **判断「要不要存 t 寄存器」的三问**：①`jal` 前写过它吗？②`jal` 后读它吗？③读的是自己写的旧值吗？三者都「是」才需要存。

---

## 五、易错点清单

| 错误 | 症状 | 修法 |
|---|---|---|
| **`sw`/`lw` 前后 `sp` 不平衡** | 返回后整个栈错位，下一次调用写坏数据 | prologue 的 `addi sp, sp, -N` 与 epilogue 的 `+N` 必须等值 |
| **只 `sw s0` 不 `addi sp`** | 写穿 caller 栈帧（Ex2 原始 bug） | 先减 `sp` 再 `sw` |
| **递归没存 `ra`** | `jr ra` 跳回自身中间 / 死循环 | 有 `jal` 就必须存 `ra` |
| **用 `t` 寄存器跨调用保数据** | 乘法因子莫名变化（Ex3） | 搬进 `s` 寄存器 |
| **用 `s` 寄存器不存不还原** | 上层函数变量被污染 | 加 prologue/epilogue |
| **两个值挤同一个栈槽** | 后写的覆盖先写的 | 每个值独立偏移（0/4/8…） |
| **给非函数标签加 prologue** | 重复 `sp` 调整 | 只有 `jal`/`jr ra` 配对的才是函数 |
| **`.globl` 漏写** | CC checker 查不出来 | 调试期加上 `.globl` 让检查器覆盖 |

---

## 六、调试套路

```text
① 先跑 CC checker：java -jar tools/venus.jar -cc lab04/exX.s
② 报错的行号 → 定位是「缺存」还是「缺还原」
③ 缺存 → prologue 加 sw + addi sp, sp, -N
④ 缺还原 → epilogue 加 lw + addi sp, sp, +N
⑤ 看 sp 平衡：入口打断点记 sp，`jr ra` 前再记一次，必须相等
⑥ 递归看 ra：单步进第二层，Registers 面板确认 ra 与上一层不同
```

> ⭐ **Venus 小技巧**：在 `jr ra` 那行打断点，右侧看 `sp`；与函数入口的 `sp` 比对，一眼看出栈帧是否泄漏。

---

## 七、提交

```bash
cd /home/yang/.openclaw/workspace/fa26-lab-starter
git add lab04/
git commit -m "lab04: fix calling convention in ex1/ex2/ex3"
git push origin main
```

然后到 Gradescope 的 **Lab 4** 作业提交。

---

## 待补充

| 项 | 原因 |
|---|---|
| `ex3.s` 在「只加行」约束下的官方参考解 | 题面未给出；本文给出的是等价可用解，若与 staff 解法不同以能跑出 1024 为准 |
| CC checker 对 `sp` 4 字节对齐的容忍度 | Venus 实测能过，但规范要求 8/16 对齐；以课程 staff 说明为准 |

---

## 相关

- Labs 总览：[[knowledge/course/cs61c-labs/index\|🧪 Labs]]
- 讲义：[[knowledge/course/cs61c/12-RISC-V-Procedures\|L12 函数调用]] · [[knowledge/course/cs61c/15-CALL\|L15 CALL]]
- 前一 Lab：[[knowledge/course/cs61c-labs/Lab03\|Lab 03 · RISC-V & Venus]]
