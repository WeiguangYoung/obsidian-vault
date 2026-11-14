---
{"dg-publish":true,"permalink":"/knowledge/course/CS61C/11-RISC-V-Control-Flow/","title":"CS61C L11 · RISC-V Control Flow, Bitwise Ops","tags":["CS61C","RISC-V","分支","循环","位运算","PC","课程笔记"],"dg-note-properties":{"title":"CS61C L11 · RISC-V Control Flow, Bitwise Ops","tags":["CS61C","RISC-V","分支","循环","位运算","PC","课程笔记"],"date":"2026-11-13","source":"https://cs61c.org/fa26/"}}
---


# CS61C FA26 · L11 RISC-V Control Flow, Bitwise Ops

> UC Berkeley · Great Ideas in Computer Architecture
> 授课：Dan Garcia
> 对应课件 11-RISC-V Control Flow, Bitwise Ops（59 页）
> 本讲对应 **Great Idea #4：Parallelism** 的前置（分支让程序能「选择」）

---

## 一、位运算指令

### 一.1 基本指令

| 操作 | 寄存器版 | 立即数版 |
|---|---|---|
| **AND** | `and rd rs1 rs2` | `andi rd rs1 imm` |
| **OR** | `or rd rs1 rs2` | `ori rd rs1 imm` |
| **XOR** | `xor rd rs1 rs2` | `xori rd rs1 imm` |
| **NOT** | `not rd rs1`（**伪指令**）| — |
| **左移** | `sll rd rs1 rs2` | `slli rd rs1 imm` |
| **逻辑右移** | `srl rd rs1 rs2` | `srli rd rs1 imm` |
| **算术右移** | `sra rd rs1 rs2` | `srai rd rs1 imm` |

**例子：**

```asm
and  rd rs1 rs2      # 0b1001 & 0b0111 = 0b0001
or   rd rs1 rs2      # 0b1001 | 0b0111 = 0b1111
xor  rd rs1 rs2      # 0b1001 ^ 0b0111 = 0b1110
not  rd rs1          # ~0b1001 = 0b0110
```

### 一.2 ⚠️ `not` 是伪指令

```asm
not rd, rs1    →    xori rd, rs1, -1
```

**原理**：`x XOR 0xFFFFFFFF` = 按位取反。

```
0b 1111 … 1111 0111    rs1
0b 1111 … 1111 1111    -1  (0xFFFFFFFF)
────────────────────
0b 0000 … 0000 1000    rd  ✅
```

### 一.3 立即数版的实用价值：位掩码

> **位运算立即数指令对位掩码非常有用：**

```asm
andi rd, rs1, 0x000000FF    # 只保留最低字节（LSB）
andi rd, rs1, 0xFF000000    # 只保留最高字节（MSB）
```

**这就是 L07 讲过的「AND = 1 保留、0 清零」的硬件实现。**

### 一.4 ⚠️ RISC-V 基础版没有的运算

| 缺失 | 原因 |
|---|---|
| **乘法** | 电路比位运算**复杂得多** |
| **除法 / 取模** | 同上 |
| **浮点加减乘除** | 电路**更复杂** |

> **这些属于 RISC-V 的扩展（M 扩展、F/D 扩展），不在基础 ISA 里。**

### 一.5 例题：位掩码组合

```asm
li   x10, 0x34FF
slli x12, x10, 0x10     # x12 = 0x34FF0000
srli x12, x12, 0x08     # x12 = 0x0034FF00
and  x12, x12, x10      # x12 = ?
```

**逐步演算：**

```
① x10 = 0x0000 34FF

② slli 0x10（左移 16 位）:
   x12 = 0x34FF 0000

③ srli 0x08（右移 8 位）:
   x12 = 0x0034 FF00

④ and 与 x10:
     0x0000 34FF
   & 0x0034 FF00
   ─────────────
     0x0000 3400      ✅
```

**答案：`0x3400`**

**按字节拆解：**

```
   0x34 & 0xFF = 0x34
   0x00 & 0xFF = 0x00
```

> **这就是 L07 讲的「左移再右移 = 提取中间字节」，再与原文做 AND。**

### 一.6 右移的两种（复习）

```
左移 n 位 = 乘 2ⁿ
右移 n 位 = 除 2ⁿ（取下限 floor）
```

| 模式 | 填充 | 例子 |
|---|---|---|
| **`sll`**（逻辑左移） | 补 0 | `1 << 3 = 8` |
| **`srl`**（逻辑右移） | 补 0 | `9 >> 2 = 2` |
| **`sra`**（算术右移） | **补符号位** | `-9 >> 2 = -3` |

**验证：**

```
 9 = 0b 0000 1001
>>2 = 0b 0000 0010 =  2     （逻辑）

-9 = 0b 1111 0111
>>2 = 0b 1111 1101 = -3     （算术，补 1）
```

> **对无符号数，`srl` 和 `sra` 结果相同。**

---

## 二、程序计数器（pc）

### 2.1 机器程序的本质

> **每条 RISC-V 汇编指令：**
> - 被存为**机器码**（比特）
> - 是**字长（32 位）**
> - **存在内存的 text 段**

```
slli x12, x10, 0x10     ← 汇编指令
        ↓
0x01051613              ← 机器码（32 位）
```

### 2.2 pc 寄存器

> **RV32I 非特权 ISA, 2.1：**
> **还有一个额外的非特权寄存器：程序计数器 `pc`，保存当前指令的地址。**

**程序执行的三个步骤：**

```
1. 用 pc 从内存取指令（fetch）
2. 执行指令（execute）
3. 更新 pc 到下一条指令（update）
```

**默认行为：**

> **`pc` 更新为 `pc + 4` 字节（「下一条顺序指令」），除非指令本身另有规定。**

**为什么是 +4？**

> **RV32I 指令是 32 位宽（4 字节）。**

---

## 三、条件分支

### 3.1 跳转的定义

> **在 RISC-V 里，跳转 = 控制流的改变。**
> **即 pc 被更新为「不是下一条指令」的地址。**

**两种跳转：**

| 类型 | 指令 | 含义 |
|---|---|---|
| **无条件跳转** | `j Label`（伪指令）| 直接跳到 Label |
| **条件跳转（分支）** | `bxx rs1 rs2 Label` | **满足条件**才跳 |

### 3.2 `beq` / `bne`

```asm
beq rs1 rs2 Label     # Branch if EQual
        if (R[rs1] == R[rs2]) PC = <Label>
```

**含义：**

1. 检查 `R[rs1] == R[rs2]`
2. **成立** → 跳到 Label
3. **不成立** → 下一条指令（`PC = PC + 4`）

```asm
bne rs1 rs2 Label     # Branch if Not Equal
        if (R[rs1] != R[rs2]) PC = <Label>
```

### 3.3 完整分支指令表

| 指令 | 名称 | 条件 |
|---|---|---|
| **`beq`** | Branch if EQual | `rs1 == rs2` |
| **`bne`** | Branch if Not Equal | `rs1 != rs2` |
| **`blt`** | Branch if Less Than（**有符号**）| `rs1 < rs2` |
| **`bge`** | Branch if Greater or Equal（**有符号**）| `rs1 >= rs2` |
| **`bltu`** | Branch if Less Than（**无符号**）| `rs1 < rs2` |
| **`bgeu`** | Branch if Greater or Equal（**无符号**）| `rs1 >= rs2` |
| **`j`** | 无条件跳转（伪指令）| 总是跳 |

> ⚠️ **`bgt`、`bgtu`、`ble`、`bleu` 不在基础 RISC-V 里！**
> **要「大于」，就交换操作数：`bgt a, b` ≡ `blt b, a`。**

### 3.4 例题 1：if-else

```c
if (i == j) {
    x = y + z;
} else {
    x = y - z;
}
```

**寄存器映射**：`x→x10, y→x11, z→x12, i→x13, j→x14`

**解法 A：用 beq**

```asm
    beq x13, x14, If       # i == j ?
    sub x10, x11, x12      # else: x = y - z
    j   End
If:
    add x10, x11, x12      # x = y + z
End:
    ...
```

**解法 B：用 bne（更贴近 C 结构）**

```asm
    bne x13, x14, Else     # i != j ?
    add x10, x11, x12      # x = y + z
    j   End
Else:
    sub x10, x11, x12      # x = y - z
End:
    ...
```

> **两种都正确，但 B 更贴近原始 C 的 if-else 结构** ——
> **尽管用了「取反的条件」。**

### 3.5 例题 2：没有 else 的 if

```c
if (i == j) {
    x = y + z;
}
// ...
```

**解法 A（需要额外跳转）**

```asm
    beq x13, x14, If
    j   End
If:
    add x10, x11, x12
End:
    ...
```

**解法 B（取反，更优）**

```asm
    bne x13, x14, End
    add x10, x11, x12
End:
    ...
```

> **「为了写出可读、高效的 RISC-V 程序，你可能需要取反分支条件！」**（课程原话）

**这是本讲最实用的一个技巧。**

---

## 四、循环

### 4.1 C 的三种循环最终都一样

| 类型 | 语法 |
|---|---|
| `while` | `while(cond) { ... }` |
| `for` | `for(init; cond; incr) { ... }` |
| `do-while` | `do { ... } while(cond);` |

> **（CS61A 讲过）任意一种都能改写成另外两种。**

**课程路径**：`for` → `while` → **`goto` 形式** → RISC-V

### 4.2 ⚠️ 学习 goto，但不要写 goto

> **我们建议永远不要用 goto 写 C 代码。**
> **但理解如何翻译成 goto，有助于翻译成汇编/RISC-V。**

**xkcd 漫画梗**：「goto 是有害的」——但汇编里**只有 goto**。

### 4.3 完整例题：数组求和

```c
int arr[20];
... // fill arr with data
int sum = 0;
for (int i = 0; i < 20; i++) {
    sum += arr[i];
}
```

**第 1 步：改写为 while**

```c
int sum = 0;
int i = 0;
while (i < 20) {
    sum += arr[i];
    i++;
}
```

**第 2 步：改写为 goto**

```c
int sum = 0;
int i = 0;
Loop:
    if (i >= 20) goto End;
    sum += arr[i];
    i++;
    goto Loop;
End:
```

> **注意条件被取反了**：`i < 20` 变成 `i >= 20` 才跳出。

**第 3 步：翻译成 RISC-V**

```asm
# 假设 x8 = &arr[0]
        add  x9,  x8, x0      # x9 = &arr[0]      （指针）
        add  x10, x0, x0      # x10 = sum = 0
        add  x11, x0, x0      # x11 = i = 0
        addi x13, x0, 20      # x13 = 20          （常量进寄存器）
Loop:
        bge  x11, x13, End    # if (i >= 20) goto End
        lw   x12, 0(x9)       # x12 = arr[i]
        add  x10, x10, x12    # sum += arr[i]
        addi x9,  x9, 4       # &arr[i+1]  ← 指针前进 4 字节
        addi x11, x11, 1      # i++
        j    Loop
End:
        ...
```

### 4.4 寄存器映射解析

| 寄存器 | 对应 | 说明 |
|---|---|---|
| **x8** | `&arr[0]` | **数组基址（输入）** |
| **x9** | `&arr[i]` | **当前元素指针** |
| **x10** | `sum` | 累加器 |
| **x11** | `i` | 循环变量 |
| **x12** | `arr[i]` | 当前元素值 |
| **x13** | `20` | 循环上限常量 |

### 4.5 三个关键设计

**① 为什么要把 20 放进寄存器？**

> **`bge` 比较的是两个寄存器；立即数 20 因此必须进寄存器 x13。**

**没有 `bgei` 指令！**

**② 为什么用 `x9` 而不是每次算 `arr[i]`？**

```asm
# ❌ 每次重新计算（慢）
slli x12, x11, 2
add  x12, x12, x8
lw   x12, 0(x12)

# ✅ 指针递增（快）
lw   x12, 0(x9)
addi x9,  x9, 4
```

> **指针递增是汇编优化的基本套路。**

**③ `addi x9, x9, 4` 里的 4 从哪来？**

> **`sizeof(int) = 4`。**
> **这就是 L09 判断题第 3 题的答案：汇编里要手动乘 sizeof。**

### 4.6 条件取反的模式

| C 写法 | RISC-V |
|---|---|
| `if (i < 20) { body }` | `bge x11, x13, End` + body |
| `if (i >= 20) exit` | `blt x11, x13, Body` |

> **规则：「想继续就取反，跳出条件写正着」。**

---

## 五、goto 的其他用法（参考）

### 5.1 goto 的语义

```c
Target:
    line;
    line;
    goto Target;    // 可以向前，也可以向后
    line;
```

> **Label 只是「指向某一行的标识符」，不算一行代码。**
> **每个 label 必须唯一命名。**

### 5.2 用 goto 简化错误处理（**这是唯一推荐用 goto 的场景**）

**❌ 朴素写法（内存泄漏）**

```c
int *a = malloc(sizeof(int)*1000);
if (a == NULL) allocation_failed();
int *b = malloc(sizeof(int)*1000000);
if (b == NULL) allocation_failed();      // ⚠ 此时 a 已分配，泄漏！
int *c = malloc(sizeof(int)*1000000000);
if (c == NULL) allocation_failed();      // ⚠ a、b 都泄漏
FILE *d = fopen(filename);
if (d == NULL) allocation_failed();
```

**❌ 嵌套 cleanup（代码爆炸）**

```c
if (b == NULL) {
    free(a);
    allocation_failed();
}
if (c == NULL) {
    free(b);
    free(a);
    allocation_failed();
}
```

**✅ 用 goto 分层清理（Linux 内核风格）**

```c
int *a = malloc(sizeof(int)*1000);
if (a == NULL) goto ErrorA;
int *b = malloc(sizeof(int)*1000000);
if (b == NULL) goto ErrorB;
int *c = malloc(sizeof(int)*1000000000);
if (c == NULL) goto ErrorC;
FILE *d = fopen(filename);
if (d == NULL) {
    free(c);
ErrorC:
    free(b);
ErrorB:
    free(a);
ErrorA:
    allocation_failed();
}
```

> **这是 goto 唯一被广泛认可的合法用途。**

### 5.3 C 结构 → goto 的完整对照

**`break`**

```c
while(true) { line; break; }
line;

// 等价于
while(true) { line; goto AfterWhile; }
AfterWhile: line;
```

**`if`（无 else）**

```c
if(cond) { line1; line2; }
line3;

// 等价于
if(!cond) goto AfterIf;
line1;
line2;
AfterIf: line3;
```

**`do-while`**

```c
do { line1; line2; } while(cond)
line3;

// 等价于
Loop:
line1;
line2;
if(cond) goto Loop;
line3;
```

**`while`**

```c
while(cond) { line1; line2; }
line3;

// 等价于
Loop:
if(!cond) goto AfterLoop;
line1;
line2;
goto Loop;
AfterLoop: line3;
```

**`for`**

```c
for(start; cond; incr) { line1; line2; }
line3;

// 等价于
start;
Loop:
if(!cond) goto AfterLoop;
line1;
line2;
incr;
goto Loop;
AfterLoop: line3;
```

> **记住这个对照表，所有 C 循环都能机械地翻成 RISC-V。**

---

## 六、完整长例题：break

**C 代码：**

```c
int a = 0;
for (int i = 0; i < 10; i++) {
    if (i == 7) break;
    a = a + i;
}
a = a + 50;
```

**第 1 步：break → goto**

```c
int a = 0;
for (int i = 0; i < 10; i++) {
    if (i == 7) goto End;
    a = a + i;
}
End: a = a + 50;
```

**第 2 步：for → while → goto**

```c
int a = 0;
int i = 0;
Loop:
    if (i >= 10) goto End;
    if (i == 7) goto End;
    a = a + i;
    i = i + 1;
    goto Loop;
End:
    a = a + 50;
```

**第 3 步：RISC-V 代码**

```asm
        li   x10, 0        # int a = 0;
        li   x5,  0        # int i = 0;
Loop:
        li   x6, 10        # j = 10        ← 小技巧：常量进寄存器
        bge  x5, x6, End   # if (i >= j) goto End;
        li   x6, 7         # j = 7         ← 复用 x6
        beq  x5, x6, End   # if (i == j) goto End;
        add  x10, x10, x5  # a = a + i;
        addi x5,  x5, 1    # i = i + 1;
        j    Loop
End:
        addi x10, x10, 50  # a = a + 50;
```

**「小技巧」的解释：**

> 课程在 C 版本里加了 `int j = 10; j = 7;` 这样的写法——
> **目的是复用同一个寄存器 `x6` 存两个不同的常量。**
> **因为 `li` 其实就是 `addi x6, x0, imm`，复用寄存器能省寄存器。**

---

## 七、RISC-V 指令总览（到这里为止）

| 类别 | 指令 |
|---|---|
| **算术** | `add` `sub` `and` `or` `xor` `sll` `srl` `sra` |
| **立即数** | `addi` `andi` `ori` `xori` `slli` `srli` `srai` `li`(伪) |
| **Load/Store** | `lw` `lb` `lbu` `sw` `sb` |
| **分支/跳转** | `beq` `bne` `bge` `blt` `bgeu` `bltu` `j`(伪) |

> **这是 RV32I 的基础骨架，L12 会加上函数调用指令。**

---

## 八、我的笔记与理解

### 8.1 本讲的两条主线

```
位运算  →  在寄存器内部操作比特
控制流  →  改变 pc，让程序能「选择」
```

**两者共同完成「程序的基本能力」**：

- 位运算处理**数据**
- 分支处理**逻辑**

### 8.2 「取反条件」是本讲最实用的技巧

对比：

```asm
# ❌ 需要两次跳转
beq x13, x14, If
j   End
If:
    add x10, x11, x12
End:

# ✅ 只需一次
bne x13, x14, End
    add x10, x11, x12
End:
```

> **规则：把「不满足就跳过」写在前面。**
> **这既是优化，也是可读性的提升** ——
> **跳转越少，流水线越顺（L23 会讲 pipe 的代价）。**

### 8.3 goto 是汇编的「真相」

C 里的 `if`、`while`、`for`、`break`、`continue` **全都是 goto 的语法糖**。

```
if     → 条件 goto 跳过
while  → 循环 label + 条件 goto 跳出 + 无条件 goto 回跳
for    → start + while + incr
break  → goto 循环结束
```

> **理解这一层，就能机械地翻译任何 C 控制流。**
> **这也是为什么课程说「NEVER write C with goto」但又要你学 goto** ——
> **不是让你写，是让你能在脑子里做转换。**

### 8.4 `bge` 没有立即数版本的设计后果

```asm
addi x13, x0, 20
bge  x11, x13, End
```

**为了一个常量，多了一条指令。**

> **这是 RISC 的取舍**：分支指令的 32 位编码里，
> **已经用了 12 位给立即数偏移（跳多远）**，**再放一个比较立即数就挤不下**。
>
> L13 讲指令格式时会看到这个空间分配的细节。

### 8.5 指针递增 vs 下标计算

```asm
# 下标计算（每次都要算）
slli x12, x11, 2
add  x12, x12, x8
lw   x12, 0(x12)

# 指针递增（只需加 4）
lw   x12, 0(x9)
addi x9,  x9, 4
```

| 方式 | 每条迭代指令数 | 优点 |
|---|---|---|
| 下标计算 | 3 条 | 不改基址 |
| **指针递增** | 2 条 | **更快** |

> **这是编译器优化的经典点。**
> **也是为什么 C 里 `*p++` 常常比 `arr[i]` 快** —— 但差别在现代编译器下已经很小。

### 8.6 「小技巧」的深意

```asm
li x6, 10
bge x5, x6, End
li x6, 7
beq x5, x6, End
```

**为什么复用 x6 存两个常量？**

> **因为 `li` 展开就是 `addi x6, x0, imm`** ——
> **复用寄存器 = 省一个寄存器。**
>
> 在只有 32 个寄存器的世界里，**每个寄存器都很贵**。

### 8.7 与后续的衔接

| 讲次 | 关系 |
|---|---|
| **L12** | 函数调用：`jal`/`jr` + 栈帧（本讲的 `j` 是基础）|
| **L13-14** | 指令格式：`beq` 的 32 位编码长什么样 |
| **L23-25** | 流水线：**分支是最难流水化的指令**（branch hazard）|
| **L20-22** | Datapath：分支如何改变 pc |

---

## 九、术语速查

| 术语 | 中文 | 说明 |
|---|---|---|
| **pc** | 程序计数器 | program counter，存当前指令地址 |
| **branch** | 分支 | 条件跳转 |
| **label** | 标号 | 指向某条指令的标识符 |
| **beq / bne** | 等于/不等于分支 | |
| **blt / bge** | 有符号小于/大于等于 | |
| **bltu / bgeu** | 无符号版本 | |
| **sll / srl / sra** | 左移/逻辑右移/算术右移 | |
| **text segment** | 代码段 | 存放指令的内存区 |
| **fetch** | 取指 | 从内存读指令 |

---

## 相关

- 课程导航：[[knowledge/course/index\|课程 目录导航]]
- 上一讲：[[knowledge/course/CS61C/10-RISC-V-Data-Transfer\|L10 RISC-V Data Transfer]]
- 下一讲：[[knowledge/course/CS61C/12-RISC-V-Procedures\|L12 RISC-V Procedures]]
