---
{"dg-publish":true,"permalink":"/knowledge/course/CS61C/12-RISC-V-Procedures/","title":"CS61C L12 · RISC-V Procedures","tags":["CS61C","RISC-V","函数调用","栈帧","jal","calling convention","课程笔记"],"dg-note-properties":{"title":"CS61C L12 · RISC-V Procedures","tags":["CS61C","RISC-V","函数调用","栈帧","jal","calling convention","课程笔记"],"date":"2026-11-13","source":"https://cs61c.org/fa26/"}}
---


# CS61C FA26 · L12 RISC-V Procedures

> UC Berkeley · Great Ideas in Computer Architecture
> 授课：Dan Garcia
> 对应课件 12-RISC-V Procedures（44 页）
> 本讲把 L10 的栈、L11 的跳转，组合成**函数调用**

---

## 一、热身：Lab 03 的 factorial

```asm
0x00  main:
0x00    li   a0, 4
0x04    jal  ra, factorial
0x08    addi a1, a0, 0
        ...
0x20    li   a0, 3
0x24    jal  ra, factorial
0x28    addi a1, a0, 0
        ...
0x48  factorial:
0x48    addi t0, x0, 1
0x4c    addi t1, x0, 1
0x50  loop:
0x50    bgt  t0, a0, done
0x54    mul  t1, t1, t0
0x58    addi t0, t0, 1
0x5c    j    loop
0x60  done:
0x60    add  a0, t1, x0
0x64    jr   ra
```

### 四个问题

| # | 问题 | 答案 |
|---|---|---|
| 1 | 为什么指令地址每次 +4？ | **RV32I 指令 32 位宽（4 字节），存在 text 段；pc 默认 +4** |
| 2 | 什么是「调用」factorial？ | **`jal ra factorial`** |
| 3 | 什么是「返回」？ | **`jr ra`** |
| 4 | 参数和返回值在哪个寄存器？ | **参数 `a0`，返回值 `a0`** |

---

## 二、函数调用 = 特殊的跳转

### 2.1 难点在哪里

> **函数调用是：**
> - **无条件跳转**
> - **pc 更新到「不是当前 +4」的地址**
> - **但「返回到哪一行」取决于「在哪里调用」**

**核心问题：怎么记住「从哪来的」？**

### 2.2 两个关键指令

**`jal`：Jump And Link**

```
语法：jal rd Label
语义：R[rd] = PC + 4
      PC = <Label 的地址>
```

**含义：**

1. 把 `PC + 4`（**返回地址**）存进 `rd`
2. **无条件跳转**到 Label

> **课程吐槽**：其实应该叫 **"Link and Jump"** —— 先链接（存返回地址）再跳。

**`jr`：Jump Register（伪指令）**

```
语法：jr rs1
语义：PC = R[rs1]
```

> **就是无条件跳转到「寄存器里的地址」。**

### 2.3 标准配对

```asm
jal ra, FunctionName     # 调用：ra = 返回地址
...
FunctionName:
    ...
jr  ra                   # 返回：跳回 ra 指的地址
```

> **`ra`（Register for Return Address，x1）** —— 专门存返回地址。

### 2.4 执行过程追踪

```
调用前：             0x04  jal ra, factorial
                      ↓
jal 执行时：
  ra  ← 0x08          （PC+4，即下一条指令）
  pc  ← 0x48          （factorial 的地址）

函数体内：
  ...
  0x64  jr ra
         ↓
jr 执行时：
  pc  ← 0x08          （回到调用点的下一条）
```

### 2.5 四种跳转指令

| 指令 | 全称 | 语义 | 说明 |
|---|---|---|---|
| **`jal`** | Jump And Link | `R[rd] = PC+4; PC = Label` | **函数调用的主力** |
| **`jr`** | Jump Register | `PC = R[rs1]` | **伪指令**，函数返回 |
| **`jalr`** | Jump And Link Register | `R[rd] = PC+4; PC = R[rs1]+imm` | **更通用**，支持函数指针 |
| **`j`** | Jump | `PC = Label` | **伪指令**，=`jalr x0, rs1, 0` |

**`jalr` 的用途**：

- **函数指针**（L06 讲过的概念在硬件层的体现）
- **某些函数调用场景**
- 比 `jal` 更灵活（可以指定任意寄存器 + 偏移）

---

## 三、RISC-V 调用约定（Calling Convention）

### 3.1 为什么需要约定

> **寄存器比内存快，所以尽量用寄存器！**
> **但不同函数共享同一组寄存器** —— 必须有「谁负责保存」的规则。

### 3.2 核心寄存器

| 寄存器 | 编号 | 用途 |
|---|---|---|
| **`a0` - `a7`** | x10 - x17 | **8 个参数寄存器**<br>**2 个返回值**（`a0`-`a1`，通常只用 `a0`）|
| **`ra`** | x1 | **返回地址**（1 个）|
| **`sp`** | x2 | **栈指针**，指向栈底 |
| **`s0`-`s11`** | x8-x9, x18-x27 | **保存寄存器**（saved）|
| **`t0`-`t6`** | x5-x7, x28-x31 | **临时寄存器**（temporary）|

### 3.3 调用函数的六个基本步骤

```
1. 设置参数
   → 把参数放进 a0, ..., a7
        ↓
2. 转移控制到被调函数
   → jal ra, fnLabel
        ↓
3. Prologue（序言）
   → 获取函数需要的本地存储：栈、寄存器
        ↓
4. 执行函数任务
   → 把返回值放进 a0（或 a1）
   → 恢复任何用到的寄存器
   → 释放栈上的本地存储
        ↓
5. Epilogue（尾声）
        ↓
6. 返回到调用点
   → jr ra
```

> **步骤 3-4-5 由「被调函数（callee）」负责。**

---

## 四、栈帧（Stack Frame）

### 4.1 复习 C 的栈

```c
int main() { a(0); }
void a(int m) { b(1); }
void b(int n) { c(2); }
void c(int o) { d(3); }
void d(int p) { }
```

```
0xFFFFFFFF  ┌──────────┐
            │   main   │  ← 高地址
            ├──────────┤
            │    a     │
            ├──────────┤
            │    b     │
            ├──────────┤
            │    c     │
            ├──────────┤
            │    d     │  ← 低地址
0x00000000  └──────────┘
```

> **栈向下增长** —— `a()` 的局部变量地址**低于** `main()`。
> **`sp` 决定「分配」与「释放」。**

### 4.2 RISC-V 的栈

> **RISC-V 栈帧（大体）和 C 一样：**
> - **栈指针寄存器：`sp`（x2）**
> - 函数通过操作 `sp` 来使用栈帧：
>   - **调用后：创建栈帧 / `sp` 递减**
>   - **函数结束时：删除栈帧 / `sp` 递增**

**栈帧可以包含：**

| 内容 | 说明 |
|---|---|
| **局部变量空间** | 例如栈上的数组 |
| **返回地址** | 为什么？**如果函数内部还要再调用别的函数** |
| **超出的参数** | 超过 8 个参数的额外部分 |

### 4.3 典型栈帧布局

```asm
fooA:
    addi sp, sp, -8      # 分配 8 字节
    sw   ra, 0(sp)       # 保存返回地址
    sw   s0, 4(sp)       # 保存 callee-saved 寄存器
    ...
    lw   s0, 4(sp)       # 恢复
    lw   ra, 0(sp)
    addi sp, sp, 8       # 释放
    jr   ra
```

**关键规则：**

> ⚠️ **栈帧大小必须是 8 的倍数**（RISC-V ABI 要求，保持 `sp` 对齐）。

---

## 五、Caller-saved vs Callee-saved（最易错的概念）

### 5.1 问题场景：递归

```c
int factorial(int n) {
    if (n == 1) return 1;
    return n * factorial(n - 1);
}
```

**调用 `factorial(2)` 时的问题：**

```
Caller factorial(2)：
  1. 准备为 callee factorial(1) 传新参数 1   ← ⚠ 覆盖了 a0 里的 2！
  2. 把控制交给 factorial(1)
  3. 把 callee 的返回值乘上「原参数 2」      ← 但 2 已经没了！
  4. 返回结果
```

> **`a0` 被递归调用覆盖了** —— 这就是「寄存器被覆盖（clobbered）」问题。

### 5.2 两类寄存器的划分

| 类别 | 寄存器 | 谁负责保存 |
|---|---|---|
| **Caller-saved**（调用者保存） | `ra`、**`t0`-`t6`**、**`a0`-`a7`** | **调用者（caller）** |
| **Callee-saved**（被调者保存） | **`sp`**、**`s0`-`s11`** | **被调者（callee）** |

### 5.3 规则的含义

**Caller-saved**：

> **这些寄存器可能不会在函数调用后保留原值！**
> **Caller 有责任在调用 callee 之前保存它们**（对于自己还需要用的值）。
> **保存方式**：复制到 `s0`-`s11`，或者**存到自己的栈帧**。

**Callee-saved**：

> **Callee 如果要用这些寄存器，必须先保存（通常压栈），返回前恢复。**
> **对 caller 来说，这些寄存器在调用后「保证不变」。**

### 5.4 为什么这样设计

| 类型 | 设计理由 |
|---|---|
| **Caller-saved** | **叶子函数**（不调用别人）无需保存任何东西 → **快** |
| **Callee-saved** | 需要跨调用保留的值，交给 callee 统一管理 → **省指令** |

> **核心目标：减少昂贵的 load/store。**
> 大多数函数是叶子函数，**不给它们增加保存开销**是关键优化。

### 5.5 速记

```
a  = argument / return   →  Caller-saved（会被覆盖！）
t  = temporary           →  Caller-saved（临时用完不管）
s  = saved               →  Callee-saved（跨调用保留）
ra = return address      →  Caller-saved
sp = stack pointer       →  Callee-saved
```

---

## 六、Caller 与 Callee 各自做什么

### 6.1 Caller 的步骤

```
1. 为 callee 做准备
   a. 如果需要，保存 caller-saved 寄存器
      （复制到 s0-s11 或存到栈）
   b. 把参数放进 a0, ..., a7（或栈）
        ↓
2. 转移控制
   jal ra, fnLabel
        ↓
3-5. （callee 执行）
        ↓
6. 恢复控制后
   如果需要，恢复 caller-saved 寄存器
```

### 6.2 Callee 的步骤

```
1-2. （caller 已完成）
        ↓
3. Prologue
   - 压栈 / sp 减小
   - 保存将要用到的 callee-saved 寄存器（s0-s11）
   - 保存 ra（如果还要调用子函数）
   - 为本地的非寄存器变量留空间（如栈数组）
        ↓
4. 执行任务
   - 返回值和 a0（必要时 a1）
        ↓
5. Epilogue
   - 恢复用到的 callee-saved 寄存器，恢复 ra
   - 弹出栈帧 / sp 增加
        ↓
6. 控制返回
```

---

## 七、完整例题：递归 factorial

### 7.1 RISC-V 代码

```asm
0x00  main:
0x00    li   a0, 3
0x04    jal  ra, factorial
        ...
0x48  factorial:
# === Prologue ===
0x48    addi sp, sp, -8       # 分配 8 字节栈帧
0x4c    sw   ra, 0(sp)        # 保存返回地址
0x50    sw   s0, 4(sp)        # 保存 caller 的 s0
0x54    mv   s0, a0           # 把参数 n 存进 s0
# === 主体 ===
0x58    li   t0, 1            # t0 = 1（base case）
0x5c    bne  s0, t0, recurse  # if (n != 1) goto recurse
0x60    li   a0, 1            # return 1
0x64    j    epilogue
# === 递归 ===
0x68  recurse:
0x68    addi a0, s0, -1       # a0 = n - 1
0x6c    jal  ra, factorial    # 递归调用
0x70    mul  a0, s0, a0       # a0 = n * factorial(n-1)
# === Epilogue ===
0x70  epilogue:
0x74    lw   ra, 0(sp)        # 恢复 ra
0x78    lw   s0, 4(sp)        # 恢复 s0
0x7c    addi sp, sp, 8        # 释放栈帧
0x80    jr   ra               # 返回
```

### 7.2 五个关键问题

**问题 1：栈帧多大？保存了哪些寄存器？**

> **8 字节。保存了 `ra` 和 `s0`。**
>
> - **`ra`**：因为**要递归调用**，`jal` 会覆盖它
> - **`s0`**：因为**我们用 `s0` 存了 `n`**，而 `s0` 是 callee-saved ——
>   我们改动了 caller 的 `s0`，**必须保存并恢复**

**问题 2：递归调用前保存了什么？为什么？**

> **`mv s0, a0`** —— 把参数 `n` 从 `a0` 复制到 `s0`。
>
> **Why？** 因为 **`a0` 是 caller-saved**，
> 递归调用 `factorial(n-1)` 会**覆盖 `a0`**。
> 而我们在递归返回后**还需要用到 `n`**（做乘法），
> 所以必须**先存到 `s0`（callee-saved）**。

**问题 3：为什么用临时寄存器 `t0`？**

> **因为 base case 是 1，而 `bne` 只能比较两个寄存器。**
> `li t0, 1` 把常量 1 放进寄存器供比较。

**问题 4：Epilogue 里先恢复寄存器还是先弹栈？**

> **先恢复寄存器，最后弹栈帧。**
>
> ```asm
> lw   ra, 0(sp)      # ① 用 sp 读
> lw   s0, 4(sp)      # ② 用 sp 读
> addi sp, sp, 8      # ③ 最后才改 sp
> jr   ra
> ```
>
> **如果先 `addi sp, sp, 8`，就再也读不到保存的 `ra` 了！**

**问题 5：为什么结尾是 `j epilogue` 而不是 `j ra`？**

> **因为返回前必须做两件事：**
> 1. **恢复寄存器**
> 2. **释放栈帧**
>
> `j ra` 会跳过这些步骤 → **栈永远不释放，寄存器也不恢复 → 灾难**。

### 7.3 栈帧演化

```
main 调用 factorial(3):

  ┌─────────────┐
  │    main     │
  ├─────────────┤
  │ factorial(3)│  ra=?, s0=old_s0, n=3
  ├─────────────┤
  │ factorial(2)│  ra=0x70, s0=?, n=2
  ├─────────────┤
  │ factorial(1)│  ra=0x70, s0=?, n=1  ← base case
  └─────────────┘
        sp
```

**每层递归都有一套独立的 `ra`、`s0`、`n`。**

> **这就是为什么必须用栈** —— 寄存器只有一份，递归需要多份。

---

## 八、完整调用约定总结

| 寄存器 | ABI 名 | 编号 | 类别 | 用途 |
|---|---|---|---|---|
| `x0` | zero | 0 | — | 恒为 0 |
| `x1` | **ra** | 1 | **Caller-saved** | 返回地址 |
| `x2` | **sp** | 2 | **Callee-saved** | 栈指针 |
| `x3` | gp | 3 | — | 全局指针（**本课不用**）|
| `x4` | tp | 4 | — | 线程指针（**本课不用**）|
| `x5-x7` | **t0-t2** | 5-7 | **Caller-saved** | 临时 |
| `x8` | **s0/fp** | 8 | **Callee-saved** | 保存 / 帧指针 |
| `x9` | **s1** | 9 | **Callee-saved** | 保存 |
| `x10-x17` | **a0-a7** | 10-17 | **Caller-saved** | 参数 / 返回值 |
| `x18-x27` | **s2-s11** | 18-27 | **Callee-saved** | 保存 |
| `x28-x31` | **t3-t6** | 28-31 | **Caller-saved** | 临时 |

### 8.1 不要用的两个寄存器

| 寄存器 | 用途 | 为什么不用 |
|---|---|---|
| **`gp`（x3）** | 指向堆的引用（global pointer） | 由运行时管理 |
| **`tp`（x4）** | 各线程独立的栈（thread pointer） | 多线程用（本课后段才讲）|

---

## 九、RISC-V 指令总览（完整版）

| 类别 | 指令 |
|---|---|
| **算术** | `add` `sub` `and` `or` `xor` `sll` `srl` `sra` |
| **立即数** | `addi` `andi` `ori` `xori` `slli` `srli` `srai` `li`(伪) |
| **Load/Store** | `lw` `lb` `lbu` `sw` `sb` |
| **分支** | `beq` `bne` `bge` `blt` `bgeu` `bltu` |
| **跳转** | `j`(伪) `jalr` `jal` `jr`(伪) |

---

## 十、我的笔记与理解

### 10.1 本讲是「拼图完成」的一讲

```
L09  寄存器 + 算术
L10  Load/Store + 栈指针
L11  分支 + pc
L12  ← 全部组合 → 函数调用
```

**函数调用需要：**

| 需要 | 来自 |
|---|---|
| 跳转能力 | L11 的 `j` |
| 记住返回地址 | L12 的 `jal`（新） |
| 保存局部数据 | L10 的 `sp` + `sw`/`lw` |
| 参数传递 | L09 的寄存器 |

> **L12 没有引入全新的概念，只是把前面的全串起来。**

### 10.2 Caller-saved vs Callee-saved 的直觉

**一句话记忆**：

> **「a、t、ra 是别人的东西，用完你负责；s、sp 是你的东西，你要负责还回去。」**

**再换个角度**：

| 视角 | Caller-saved | Callee-saved |
|---|---|---|
| **caller 的心声** | 「我得自己兜着」 | 「callee 会帮我保住」 |
| **callee 的心声** | 「随便改，不关我事」 | 「我改了就得恢复」 |

### 10.3 为什么 `ra` 是 caller-saved 很反直觉

```
jal ra, foo     ← 我调用 foo，ra 被设成返回地址
                ⚠ 但 foo 内部如果再 jal ra, bar
                ⚠ ra 就被 bar 的返回地址覆盖了！
```

> **所以 `ra` 虽然是「我的返回地址」，但它会被我调用的函数破坏。**
> **我必须在调用前把它存起来（压栈）。**
>
> **这是递归函数必须压 `ra` 的根本原因** —— 也是 L12 例题里 `sw ra, 0(sp)` 的理由。

### 10.4 「先恢复后弹栈」的物理必然

```asm
lw   ra, 0(sp)      # ✅ 此时 sp 还指向栈帧
lw   s0, 4(sp)      # ✅
addi sp, sp, 8      # ✅ 最后才动 sp
```

**如果顺序反了：**

```asm
addi sp, sp, 8      # ❌ sp 已经指向「上层」
lw   ra, -8(sp)     # 必须用负偏移才能找回……太容易错
```

> **规则：栈帧在「用完」之后才释放。**
> **这和 L05 讲的「函数返回时栈帧被扔掉」是同一件事。**

### 10.5 Prologue / Epilogue 必须配对

```
Prologue                Epilogue
addi sp, sp, -8   ↔    addi sp, sp, 8
sw   ra, 0(sp)    ↔    lw   ra, 0(sp)
sw   s0, 4(sp)    ↔    lw   s0, 4(sp)
```

> **顺序相反，缺一不可。**
> **这是汇编里最常见的 bug 来源。**

### 10.6 递归的代价

**每层递归 = 一个栈帧（8 字节）+ 一次函数调用开销。**

```
factorial(1000)   →  1000 个栈帧  →  8 KB
factorial(100000) →  栈溢出（stack overflow）
```

> **这就是为什么递归有深度限制。**
> **也是为什么尾递归优化（tail call optimization）很重要** —— 但 **RISC-V 汇编里要手动做**。

### 10.7 与后续的衔接

| 讲次 | 关系 |
|---|---|
| **L13-14** | 指令格式：`jal`/`jalr` 的 32 位编码（J-type / I-type）|
| **L15** | CALL：更复杂的调用场景 |
| **L20-22** | Datapath：`jal` 如何实际改变 pc |
| **L23-25** | 流水线：函数调用/返回是控制冒险的重点 |

---

## 十一、术语速查

| 术语 | 中文 | 说明 |
|---|---|---|
| **jal** | 跳转并链接 | Jump And Link |
| **jalr** | 寄存器跳转并链接 | Jump And Link Register |
| **jr** | 寄存器跳转 | 伪指令 |
| **ra** | 返回地址寄存器 | x1 |
| **sp** | 栈指针 | x2，callee-saved |
| **a0-a7** | 参数/返回值寄存器 | x10-x17，caller-saved |
| **s0-s11** | 保存寄存器 | callee-saved |
| **t0-t6** | 临时寄存器 | caller-saved |
| **prologue** | 序言 | 函数开头，建栈帧 |
| **epilogue** | 尾声 | 函数结尾，拆栈帧 |
| **caller** | 调用者 | |
| **callee** | 被调者 | |
| **clobber** | 覆盖 | 寄存器值被破坏 |
| **ABI** | 应用二进制接口 | 调用约定的规范 |

---

## 相关

- 课程导航：[[knowledge/course/index\|课程 目录导航]]
- 上一讲：[[knowledge/course/CS61C/11-RISC-V-Control-Flow\|L11 RISC-V Control Flow, Bitwise Ops]]
- 下一讲：[[knowledge/course/CS61C/13-RISC-V-Instruction-Formats-I\|L13 RISC-V Instruction Formats I]]
