---
{"dg-publish":true,"permalink":"/knowledge/course/cs61c-labs/lab06/","title":"CS61C FA26 · Lab 06 · CPU 与流水线","tags":["CS61C","lab","lab06","Logisim","CPU","流水线","数据通路","课程笔记"],"dg-note-properties":{"title":"CS61C FA26 · Lab 06 · CPU 与流水线","tags":["CS61C","lab","lab06","Logisim","CPU","流水线","数据通路","课程笔记"],"date":"2026-11-20","source":"https://cs61c.org/fa26/labs/lab06/"}}
---


# Lab 06 · CPU 与流水线

> UC Berkeley · Great Ideas in Computer Architecture · Fall 2026
> 代码仓库：`github.com/61c-teach/fa26-lab-starter`
> 本地路径：`/home/yang/.openclaw/workspace/fa26-lab-starter/lab06/`

返回 [[knowledge/course/cs61c-labs/index\|🧪 Labs 总览]] · [[knowledge/course/cs61c/index\|📚 CS61C 目录]]

---

## 本 Lab 在讲什么

Lab 05 玩的是「组合逻辑 + 状态」的小电路，Lab 06 直接跳到 **CPU 三件套**：

1. **立即数生成器（Immediate Generator）** —— 从 32 位指令里把散落的立即数字段拼出来并符号扩展；
2. **BrUn 控制信号** —— 决定分支比较器用有符号还是无符号；
3. **单周期 vs 流水线** —— 用「乘法 + 加法」这个极简例子，把 **时钟周期、关键路径、流水线寄存器** 这三个概念算一遍；
4. **调试技能** —— 用时序波形图（Timing Diagram）在 Logisim 里真的定位 bug。

对应讲义是 [[knowledge/course/cs61c/20-Datapath-I\|L20 Datapath]] 与 [[knowledge/course/cs61c/22-Control\|L22 Control]]，流水线部分对应 [[knowledge/course/cs61c/23-Pipeline-I\|L23 Pipeline-I]]、[[knowledge/course/cs61c/24-Pipeline-II-Hazards\|L24 Pipeline-II-Hazards]]。

> 💡 Ex1 / Ex2 的电路**直接用进 Project 3**（搭建完整 RISC-V CPU），所以别糊弄。

---

## 环境准备

**必须在本地机器做**（不要在 hive 上做），Logisim Evolution 需要图形界面。

```bash
cd ~/labs
git pull origin main      # 先同步之前 lab 的改动
git pull starter main     # 拉取 lab06 的骨架文件
```

**必须用课程分发的 Logisim 版本**（`../tools/logisim-evolution.jar`），网上其他版本可能有差异 —— 特别是 project 用的自定义库和 bugfix。

### ⚠️ 两个致命坑

| 坑 | 说明 | 后果 |
|---|---|---|
| **Logisim 不自动保存** | 改完电路必须手动 `Ctrl+S`（或 File → Save） | 崩溃/关窗 = 全部白干 |
| **不能动 staff 提供的输入输出引脚** | `ex1.circ` 里的 `instruction`、`immediate` 引脚已经布好线，删了/改名/移位都会让测试挂掉 | `test.py` 输出全 `x` 或列对不上 |

第二点最隐蔽：**引脚名字就是 CSV 的表头**。`test.py` 用 `-tty table,binary,csv` 跑电路，表头顺序由引脚的位置和名字决定，一旦你重新排布引脚，`ex1-test.out` 的列就和 `ex1-test.ref` 对不上 —— 电路逻辑可能完全正确，测试却 FAILED。

第三点：`check_hashes.py` 会校验 `tests/` 下的测试文件和参考输出，**不要改 `tests/` 里的任何东西**（`test.py` 开跑前第一件事就是 `check_hashes()`，改动直接 `sys.exit(1)`）。

---

## 文件清单

| 文件 | 状态 | 说明 |
|---|---|---|
| `ex1.circ` | ⬜ 待实现 | S 型立即数生成器（子电路 `generate_immediate`）|
| `ex2.circ` | ⬜ 待实现 | `BrUn` 生成（子电路 `compute_BrUn`）|
| `ex3.circ` | 只读参考 | 未流水线电路 `non_pipelined`（乘法 → 加法），用于算周期 |
| `ex3_answers.txt` | ⬜ 待填 | 1 题（非流水线周期 / 频率）|
| `ex4.circ` | ⬜ 待实现 | 流水线版 `pipelined`（乘法级 → 加法级）|
| `ex4_answers.txt` | ⬜ 待填 | 2 题（周期 → 频率 + 为什么不需要 bubble）|
| `ex5.circ` | ⬜ 待修 | `num_checker` 电路，有 bug 要调 |
| `ex5_answers.txt` | ⬜ 待填 | 11 题（Tracing / 调试题）|
| `test.py` | 勿改 | 跑 4 个测试（ex1 / ex2 / ex4 / ex5）|
| `check_hashes.py` | 勿改 | 校验 `tests/` 未被篡改 |
| `tests/ex*-test.circ` | 勿改 | 测试驱动电路（ROM 输入、时钟驱动）|
| `tests/out/ex*-test.ref` | 勿改 | 期望输出（CSV）|

**跑测试**：

```bash
cd /home/yang/.openclaw/workspace/fa26-lab-starter/lab06
python3 test.py
```

输出三个关键文件（在 `tests/out/`）：

```text
ex1-test.out   # 你电路的输出（CSV）
ex1-test.ref   # 期望输出（CSV）
ex1-test.out 与 .ref 逐行 diff → 哪一行不匹配就是哪个输入出的问题
```

`test.py` 的实际流程（读源码可得）：

```text
① check_hashes()  → tests/ 被改动直接退出
② fix_circ()      → 自动修 tests/ex#-test.circ 里对 ../ex#.circ 的相对路径
③ java -jar ../tools/logisim-evolution.jar -tty table,binary,csv tests/ex#-test.circ
④ 逐行对比 stdout 与 .ref，首行不同即 FAILED
```

> 💡 第一行就是表头（如 `CYCLE,INSTRUCTION,IMMEDIATE`），列顺序 = 引脚在电路里出现的顺序。

---

## 背景知识 1 · 立即数生成（Immediate Generation）

### 为什么需要单独一个「立即数生成器」

RISC-V 是定长 32 位指令，但立即数在五种格式里被**切碎、错位、甚至倒序**地塞在各处。数据通路里的 ALU 只有一个 32 位输入口，所以必须有一个组合逻辑模块：**输入 32 位指令 → 输出符号扩展后的 32 位立即数**，即讲义里的 `ImmGen`（[[knowledge/course/cs61c/20-Datapath-I\|L20]]）。

### 五种格式的位域图

```text
R-type  | funct7 |  rs2  | rs1 |funct3|  rd  |opcode|
        | 31..25 | 24..20|19..15|14..12|11..7| 6..0 |
        |<-------------------- 无立即数 -------------------->|

I-type  |      imm[11:0]      | rs1 |funct3|  rd  |opcode|
        |       31..20        |19..15|14..12|11..7| 6..0 |
                  ↑ 12 位，符号扩展

S-type  |  imm[11:5]  |  rs2  | rs1 |funct3| imm[4:0] |opcode|
        |   31..25    |24..20 |19..15|14..12|  11..7   | 6..0 |
        └─ 高 7 位           imm[4:0] ─┘   ← 立即数被劈成两半

B-type  |imm[12]|imm[10:5]|  rs2  | rs1 |funct3|imm[4:1]|imm[11]|opcode|
        |  31   | 30..25  |24..20 |19..15|14..12| 11..8 |  7    | 6..0 |
        └ imm[12] 和 imm[11] 分居两端，imm[0] 恒为 0（跳转总是 2 字节对齐）

U-type  |              imm[31:12]              |  rd  |opcode|
        |               31..12                  |11..7 | 6..0 |
        |← 20 位高位，低 12 位补 0

J-type  |imm[20]|imm[10:1]|imm[11]|imm[19:12]|  rd  |opcode|
        |  31   | 30..21  |  20   |  19..12  |11..7 | 6..0 |
        └ 同样：imm[0]=0，位序被完全打乱（这是硬件复用的代价）
```

### 拼接规则表

| 类型 | 用于 | 立即数位组成（→ 32 位结果）| 备注 |
|---|---|---|---|
| **I** | `addi` `lw` `jalr` | `{{20{inst[31]}}, inst[31:20]}` | 唯一「连续」的，最简单 |
| **S** | `sw` `sb` `sh` | `{{20{inst[31]}}, inst[31:25], inst[11:7]}` | **本 lab Ex1** |
| **B** | `beq` `blt` `bge` 等 | `{{19{inst[31]}}, inst[31], inst[7], inst[30:25], inst[11:8], 1'b0}` | 位序错乱 + 末尾补 0 |
| **U** | `lui` `auipc` | `{inst[31:12], 12'b0}` | 只取高 20 位，低 12 位清零；**不符号扩展** |
| **J** | `jal` | `{{11{inst[31]}}, inst[31], inst[19:12], inst[20], inst[30:21], 1'b0}` | 位序最乱，末尾补 0 |

> ⚠️ **符号扩展一律看 `inst[31]`**。I/S/B 都复制 bit31 共 20 位；U 型不是符号扩展而是「低位补 0」，因为它构造的是 `imm << 12`。

### Ex1 具体做法（S 型）

**输入**：`instruction`（32 位，staff 已接好）
**输出**：`immediate`（32 位）

```text
instruction
 ├─ bit 31..25  (7 位) ──┐
 ├─ bit 11..7   (5 位) ──┤
 └─ bit 31      (1 位) ──┴──► 复制 20 次 ──► {20{bit31}, 7 位, 5 位} ──► immediate
```

**Logisim 操作步骤**：

```text
① Splitter（Wiring 库）劈 instruction：
   - Fan Out = 2，Bit Width In = 32
   - 分支0：Bit 31（1 位）     ← 符号位
   - 分支1：Bit 30..25（6 位） ← 本 lab 不需要，忽略
   - 分支2：Bit 24..7（18 位） ← 忽略
   - 分支3：Bit 6..0（7 位）   ← 取其中的 bit 11..7
   （更省事的做法：直接做两个 splitter，一个取 31，一个取 [11:7]）

② 符号扩展：bit31 接一个 1→20 位的分配器（Splitter 反向使用 / 或
   用 20 个 Tunnel 复制），得到 20 个相同的 bit31。

③ 三路拼接（Wiring 库的 Splitter 反用「合并」模式，或直接用
   多个 bit 组合成 32 位线）：
   [ 20×bit31 ][ 7 位 inst[31:25] ][ 5 位 inst[11:7] ] = 32 位
```

> 💡 最省心的实现：用 **两个 Splitter**。
> - Splitter A（32 位输入）：输出 `bit31`(1) 与 `inst[30:0]`(31)；
> - Splitter B（31 位输入）：输出 `inst[30:25]`(6)、`inst[24:7]`(18)、`inst[11:7]`(5)… 
> 注意各分支 bit 范围必须**首尾相接、覆盖全部 32 位**，否则 Logisim 会报错。

**验证方法（手工对答案）**：

`tests/out/ex1-test.ref` 第 3 行：

```text
INSTRUCTION = 11111110010101000010100000100011
```

拆开：

```text
31..25 = 1111111    (0x7F)
24..20 = 00101      (rs2)
19..15 = 01000      (rs1)
14..12 = 010        (funct3 = sw)
11..7  = 01100      (0x0C)
6..0   = 0100011    (opcode = 0x23, STORE)
```

`{20{1}, 1111111, 01100}` = `11111111_11111111_11110111_11110000`... 手算极易出错。

**不要手算猜，直接跑脚本对答案**（本机已验证可用）：

```bash
cd /home/yang/.openclaw/workspace/fa26-lab-starter/lab06
python3 - <<'PY'
for line in open('tests/out/ex1-test.ref').read().splitlines()[1:]:
    cyc, ins, imm = line.split(',')
    b = lambda hi, lo: ins[31-hi:32-lo]
    sign = '1'*20 if ins[0] == '1' else '0'*20
    print(cyc, sign + b(31,25) + b(11,7), '| ref:', imm, '| OK' if sign+b(31,25)+b(11,7) == imm else '| MISMATCH')
PY
```

全部输出 `OK` 即证明拼接公式正确 —— 这也顺便成了你电路的「标准答案表」。

做题时正确姿势：先写电路 → `python3 test.py` → 看 `.out` 与 `.ref` 差在哪行 → 把该行的 `INSTRUCTION` 丢进上面的脚本，看自己拼出的 32 位错在哪一段。

---

## 背景知识 2 · 分支比较（Branch Comparison）

### 三个比较结果的信号

数据通路的 Branch Comp 模块（[[knowledge/course/cs61c/20-Datapath-I\|L20]]）比较 `rs1` 与 `rs2`，输出三个 1 位信号：

| 信号 | 含义 | 生成的指令 |
|---|---|---|
| `BrEq` | `A == B` | `beq` / `bne` |
| `BrLT` | `A < B`（按 `BrUn` 决定有/无符号）| `blt` / `bltu` / `bge`(取反) / `bgeu`(取反) |
| `BrUn` | **控制信号**：`1` = 无符号比较，`0` = 有符号比较 | 见下表 |

拿到 `BrEq`/`BrLT` 后，Control 用它算 `BrTaken`：

```text
beq   →  BrTaken =  BrEq
bne   →  BrTaken = ~BrEq
blt   →  BrTaken =  BrLT
bge   →  BrTaken = ~BrLT
bltu  →  BrTaken =  BrEq·0 + BrLT(BrUn=1)
bgeu  →  BrTaken = ~BrLT      (BrUn=1 时)
```

对应 `funct3` 编码（LAB 里要用）：

| funct3 | 指令 | 比较类型 | `BrUn` |
|---|---|---|---|
| `000` | `beq` | 相等 | Don't care |
| `001` | `bne` | 相等 | Don't care |
| `100` | `blt` | 有符号 | **0** |
| `101` | `bge` | 有符号 | **0** |
| `110` | `bltu` | 无符号 | **1** |
| `111` | `bgeu` | 无符号 | **1** |

> 规律：`funct3` 的 **bit0 = 1** 就是无符号。所以 `BrUn = funct3[0]`，且仅在 opcode 是 BRANCH 时有效。

### 有符号 vs 无符号：为什么差一个 bit

32 位补码里，最高位是符号位：

```text
0xFFFFFFFF
  有符号解读 → -1
  无符号解读 → 4294967295

所以  -1 <  0   为真（有符号）
     4G  <  0   为假（无符号）
```

硬件层面，比较器就是「减法 + 看符号位/进位」，`BrUn` 决定看哪个：

```text
BrUn = 0 (signed)   →  负数（bit31=1）天然更小，用补码减法后的 sign 位判定
BrUn = 1 (unsigned) →  bit31 只是普通数据位，用减法 borrow/carry 判定
```

**关键限制**（lab 明确写了）：**两个操作数必须同为有符号或同为无符号**。硬件不支持混比 —— 这正是 `BrUn` 存在的意义：它在**比较的那一刻**统一告诉硬件「这一对按哪种类型看」。

### Ex2 具体做法

**输入**：`instruction`（32 位）
**输出**：`BrUn`（1 位）

核心洞察（lab 原文）：**opcode 可以不管**。

> 因为非分支指令 `BrUn` 是 don't care，而 `beq`/`bne` 也是 don't care，所以只有 `bltu`/`bgeu` 需要 `1`。它们的 `funct3` 是 `110`/`111`，共同特征 = **`funct3[0] = 1` 且 `funct3[2:1] = 11`**。

先看清 `funct3` 的六种编码（`funct3[2]funct3[1]funct3[0]`）：

```text
000 beq   → 0 0 0
001 bne   → 0 0 1
100 blt   → 1 0 0
101 bge   → 1 0 1
110 bltu  → 1 1 0   ← funct3[1] = 1
111 bgeu  → 1 1 1   ← funct3[1] = 1
```

⚠️ **不能用 `funct3[0]`（= `inst[12]`）**：`100 blt` 的 bit0 = 0，而 `110 bltu` 的 bit0 也是 0 —— 区分不开。

✅ **正确规律：`funct3[1] = 1` 恰好圈出 `{110, 111}`**（注意 `101 bge` 的 bit1 也是 0，不会误中）。而 `funct3[1]` 就是 `inst[13]`，所以：

```text
方案 A（最简，推荐）：BrUn = instruction[13]
```

一根线搞定。三种实现（任选其一）：

```text
方案 B（查表式，最直观）：
   Splitter 取 inst[14:12] → funct3 (3 位)
   Comparator(funct3, 0b110, unsigned) 的 = 输出  ─┐
   Comparator(funct3, 0b111, unsigned) 的 = 输出  ─┴─ OR ─► BrUn
```

> 方案 B 用的是 Arithmetic 库里的 **Comparator**，把 `Data Bits = 3`、`Comparison = Unsigned` 设好，取 `=` 那根输出线即可。

> ⚠️ Logisim 的 Comparator 属性里要设 **Comparison = Unsigned**（比较 funct3 这种小正整数，有符号无符号结果一样，但设 unsigned 更符合语义）；`Data Bits = 3`。
>
> 💡 方案 A 一条线，但**方案 B 更能体现 spec 的意图**。考试/项目里也更通用（如果以后还要看 opcode）。

**用 `.ref` 验证**：

```text
01,01100101011001000101000001100011,0   → opcode 1100011 (BRANCH)，funct3=010? 
10,01010110010101000110000001100011,1   → funct3 = 110 → bltu → BrUn=1 ✓
```

（`instruction[14:12]` = `110` ⇒ `inst[13] = 1` ⇒ 输出 1，和 `.ref` 一致。）

---

## 背景知识 3 · 单周期 vs 流水线

### 单周期 CPU 的数据通路

[[knowledge/course/cs61c/20-Datapath-I\|L20]] 的经典 RISC-V datapath：

```text
 PC ──► IMEM ──► [Instr] ─┬─► RegFile(rs1,rs2) ──► ALU ──► DMEM ──► RegFile(rd)
                          ├─► ImmGen ─────────────┘
                          └─► Control ──► 所有 mux/使能信号
```

**单周期的致命问题**：**一个周期要装下最慢的那条指令的全部阶段**。

```text
周期长度 = max(每条指令的路径延迟)
        = 最慢指令（通常是 lw）的 IF + ID + EX + MEM + WB
```

`add` 明明只需要 IF+ID+EX+WB，却也被迫等满 MEM 的时间 —— 硬件在空转。

**流水化的思路**：在阶段之间插入**流水线寄存器**，让不同指令的不同阶段在**同一时刻**并行。

```text
单周期（串行执行 3 条指令）：
时间 ──────────────────────────────────────────────►
 I1  [F][D][E][M][W]
 I2                 [F][D][E][M][W]
 I3                                 [F][D][E][M][W]

五级流水（重叠执行 3 条指令）：
时间 ──────────────────────────────────────────────►
 I1  [F][D][E][M][W]
 I2     [F][D][E][M][W]
 I3        [F][D][E][M][W]
      ← 稳态后每个周期完成一条 →
```

### 流水化后周期怎么变

```text
T_clk_nonpipe = T_reg(CtQ) + max_path_delay + T_setup
T_clk_pipelined = T_reg(CtQ) + max(单级延迟) + T_setup
```

理想情况：**n 级流水 → 吞吐率提升约 n 倍 → 时钟频率提升约 n 倍**（前提是负载均衡）。

代价：

| 代价 | 说明 |
|---|---|
| 流水线寄存器开销 | 增加 `CtQ + setup`，占周期比例变大 |
| 最慢一级成为瓶颈 | 其他级要等它（本 lab 的乘法 = 60ns，加法 = 45ns）|
| 冒险 | 需 forwarding / stall / flush（见下一节）|
| 填充/排空 | n 条指令的启动/结束时延 |

### 控制信号怎么沿流水线传递

单周期里 Control 很「自由」，信号直接连到要用的地方；**流水化后必须跟着指令一起走**。

原则：**控制信号在哪一级被产生/被需要，就打包进对应级的流水线寄存器**。

```text
IF/ID 寄存器： PC+4
ID/EX 寄存器： PC, rs1值, rs2值, imm, rs1号, rs2号, rd号,
              ALUSrc, ALUOp, MemRead, MemWrite, RegWrite, Branch, MemToReg
EX/MEM 寄存器： ALU 结果, rs2值, rd号,
              MemRead, MemWrite, RegWrite, MemToReg
MEM/WB 寄存器： 读出的数据 / ALU 结果, rd号, RegWrite, MemToReg
```

一句话：**「往后还要用的，全塞进寄存器」**。`RegWrite` 要走到 WB 级，所以要一直带到底；`ALUSrc` 只在 EX 用，带进 ID/EX 就够了。

> ⚠️ 这是 Project 3 的核心，本 lab 的 Ex4 只是这个思想的**最小可运行实例**。

---

## 背景知识 4 · 流水线冒险（Hazards）

三条危险来源：

| 类型 | 触发条件 | 典型指令对 |
|---|---|---|
| **结构冒险（Structural）** | 两条指令同一周期抢同一硬件资源 | 单 memory 同时 IF 和 MEM |
| **数据冒险（Data）** | 后一条指令要读前一条还没写回的结果 | `add x1,x2,x3` → `sub x4,x1,x5` |
| **控制冒险（Control）** | 分支结果未定，取指已往下走了 | `beq x1,x2,L` 后面跟着的指令 |

### 数据冒险：forwarding vs stall

```text
add  x1, x2, x3     IF  ID  EX  MEM WB
sub  x4, x1, x5         IF  ID  EX  MEM WB
                               ↑ 这里 sub 要读 x1，但 add 的 WB 在 2 周期后
```

两种解法：

| 方案 | 做法 | 何时可用 |
|---|---|---|
| **Forwarding（旁路）** | 把 EX/MEM 或 MEM/WB 的 ALU 结果**直接引回 ALU 输入**，不等写回 | 前一条是 **ALU 指令**（`add`/`sub`/`and`/`slli`…）|
| **Stall（气泡）** | 冻结 IF/ID、清空 ID/EX，插入 nop | 前一条是 **`lw`**（数据在 MEM 级才出来）|

**典型的 load-use 缺口**：

```text
lw   x1, 0(x2)      IF  ID  EX  MEM WB
add  x3, x1, x4         IF  ID  EX  MEM WB
                               ↑ add 在 EX 要 x1，lw 的数据此刻才从 MEM 出来
                            ⇒ 必须 stall 1 周期（可用 forwarding 从 MEM/WB 补，但仍是 1 拍）
```

**Hazard 判断表**（Project 3 里直接照抄这个逻辑）：

| 前一条（EX/MEM 级）| 后一条（EX 级）读的寄存器 | 需要做什么 |
|---|---|---|
| ALU 指令写 `rd` | `rs1` 或 `rs2` = `rd` | **Forwarding**（EX/MEM → ALU 输入 B/A）|
| `lw` 写 `rd` | `rs1` 或 `rs2` = `rd` | **Stall 1 拍 + Forwarding**（MEM/WB → ALU）|
| 上上一条 ALU 写 `rd` | `rs1`/`rs2` = `rd`，且上一条没写 | **Forwarding**（MEM/WB → ALU）|
| 写 `x0` | 任何 | **无冒险**（`x0` 恒为 0，硬件保证）|
| 两条都写同一 `rd` | — | 近的先算，远的丢弃（按「最近一条」优先） |

> ⚠️ **`x0` 陷阱**：`rd == 0` 时永远不触发 forwarding/stall。忘判这个，`lw x0, ...` 之后的指令会读到垃圾数据。

### 控制冒险

分支要等到 **EX 级算完 `BrTaken`** 才知道跳不跳，但此时 IF 已经取了下一条。

| 方案 | 说明 | 代价 |
|---|---|---|
| **Stall 到分支出结果** | 简单，但每次分支罚 2-3 拍 | 慢 |
| **Static prediction（静态预测）** | 固定「不跳」/「跳」；错则 flush | 简单 |
| **Dynamic prediction** | BHT/2-bit 计数器，学历史 | 复杂但准 |
| **Delayed branch** | 编译器把一条有用的指令填进延迟槽 | RISC-V **不采用** |
| **Early branch resolution** | 在 ID 级就算 `BrTaken`（多加点硬件）| 只罚 1 拍 |

**Flush（冲刷）**：预测错时，把 IF/ID、ID/EX 里已取的错误指令清成 nop。

```text
beq  x1, x2, TARGET     IF  ID  EX  MEM WB          ← EX 算出 BrTaken=1
add  x3, x4, x5             IF  ID  ✗               ← flush
sub  x6, x7, x8                 IF  ✗               ← flush
TARGET: ...
```

⚠️ **Flush 时绝不能写寄存器**：清空 `ID/EX` 的 `RegWrite` 是关键，否则被冲刷的指令仍会写入 `rd`。

### 本 lab 的 Ex4：为什么「不需要 bubble」

Ex4 的流水线里，**一条「指令」= 一次乘法 + 一次加法**，乘法和加法是**同一指令的两个阶段**，不是两条独立指令。

```text
        ┌── 级1 ──┐         ┌── 级2 ──┐
in1 ──► │ 乘法器  │ ──reg──►│ 加法器  │──► out
in2 ──► └─────────┘         └─────────┘
                              (+ 当前累加值)
```

**没有数据冒险**：`in1`/`in2` 每条周期都由外部提供，后一条指令**不依赖前一条的计算结果**（状态累加是级2内部的寄存器，不是跨指令依赖）。

**没有控制冒险**：没有分支。

所以理论上**一条 bubble 都不需要**。实际会看到输出「晚 1 拍」（领先的 0），那是流水线**填充**（pipeline fill），不是 bubble。

> ⭐ **Fill ≠ Bubble**：fill 是「启动时下游还没货」，bubble 是「有货但不能往下走」。`.ref` 里的前两行 `00000000` 属于 fill。

---

## ⭐ Exercise 1 · Constructing Immediates

**目标**：在 `ex1.circ` 里补全 S 型立即数生成器。

- 输入 `instruction` 32 位，输出 `immediate` 32 位（**引脚位置不要动**）
- 要用到 **Splitter**（拆分/合并位段）
- 结果 = `{{20{inst[31]}}, inst[31:25], inst[11:7]}`

**实现骨架**：

```text
[instruction] ──Splitter(32→1+6+18+7)──┬─ bit31 (1) ──► 扩展成 20 位
                                        ├─ [30:25] (6) → 只用其中 [31:25] 需要 bit31，
                                        │                实际取 7 位：{bit31, inst[30:25]}
                                        └─ [11:7] (5)
                                        
拼：{ 20×bit31 , 7 位(inst[31:25]) , 5 位(inst[11:7]) } = 32 位 → immediate
```

**自检清单**：

```text
□ instruction 与 immediate 引脚名字/位置未改
□ bit31 复制了 20 份（符号扩展），不是 19 份也不是 21 份
□ inst[31:25] 是 7 位（包含 bit31），别和符号扩展部分重复算错
□ 保存（Ctrl+S）后再跑 python3 test.py
□ ex1-test.out 与 ex1-test.ref 逐行一致 → PASSED
```

**常见失败模式**：

| 现象 | 原因 |
|---|---|
| 输出全 `x`（不定态）| Splitter 分支没覆盖全部 32 位，有悬空位 |
| 负数（`1111...`）输出变成 `0000...` | 符号扩展没做，高位补 0 |
| 差一点点（正数对、负数错）| 符号位取了 `inst[25]` 而不是 `inst[31]` |
| 列错位 / 表头不对 | 引脚被移动或改名 |

---

## ⭐ Exercise 2 · Constructing the BrUn Control Signal

**目标**：在 `ex2.circ` 里实现 `BrUn`。

| 指令类型 | `BrUn` |
|---|---|
| 无符号分支（`bltu`/`bgeu`，funct3=`110`/`111`）| `1` |
| 有符号分支（`blt`/`bge`，funct3=`100`/`101`）| `0` |
| `beq`/`bne`、非分支 | Don't care |

**最快解**：`BrUn = instruction[13]`（= `funct3[1]`）。

**查表解（推荐理解）**：

```text
instruction ──Splitter──► funct3 = inst[14:12] (3 位)
                             │
                ┌────────────┼─────────────┐
                ▼            ▼             
          == 0b110      == 0b111
          (Constant)    (Constant)
                │            │
                └─── OR ─────┘ ──► BrUn
```

⚠️ 注意 `funct3[0]`（= `inst[12]`）**区分不了** `blt`(100) 和 `bge`(101)，也区分不了 `bltu`(110) 和 `bgeu`(111) —— 正确的是 `funct3[1]`（= `inst[13]`），把 `{100,101}` 和 `{110,111}` 分开。

> ⚠️ Comparator 的 `Data Bits` 必须设 3，`Comparison` 设 **Unsigned**。

---

## Exercise 3 · Inefficiencies Everywhere（非流水线）

**给定时序参数**：

| 部件 | 延迟 |
|---|---|
| 加法器 | **45 ns** |
| 乘法器 | **60 ns** |
| 寄存器 CLK-to-Q (`T_cq`) | **10 ns** |
| 寄存器 setup time (`T_setup`) | **10 ns** |
| 寄存器 hold time (`T_hold`) | **5 ns** |

**电路结构**（`ex3.circ` 的 `non_pipelined`）：

```text
        ┌──────────┐    ┌──────────┐    ┌─────────┐
in1 ───►│          │    │          │    │         │
        │ 乘法器   ├───►│ 加法器   ├───►│ 寄存器  ├──► out
in2 ───►│ (60 ns)  │    │ (45 ns)  │    │         │
        └──────────┘    └──────────┘    └─────────┘
                            ▲
                     当前状态值（寄存器输出）
```

**关键路径** = 输入寄存器 `CtQ` → 乘法器 → 加法器 → 输出寄存器 `setup`：

```text
T_clk = T_cq + T_mult + T_add + T_setup
      = 10 + 60 + 45 + 10
      = 125 ns
```

**频率**：

```text
f = 1 / 125ns = 8 MHz
```

**答案格式**（`ex3_answers.txt` 只问周期）：

```text
1. 125   # ns（周期）
```

> 💡 题目假设「两个输入都从寄存器来」，所以 `T_cq` 只算一次。`T_hold` 与**最小**周期无关（它约束的是太快的情况），本题不需要。
>
> 💡 本题**不需要**写频率，但顺手知道：`1/125ns = 8 MHz`。

---

## ⭐ Exercise 4 · Pipe that Line（流水线）

**目标**：把 Ex3 的电路改成两级流水 —— 乘法级 → 加法级，中间加一个寄存器。

```text
        ┌──────────┐    ┌──────────┐    ┌─────────┐
in1 ───►│ 乘法器   ├───►│ [流水线] ├───►│ 加法器  ├───► [寄存器] ──► out
in2 ───►│ (60 ns)  │    │ 寄存器   │    │ (45 ns) │
        └──────────┘    └──────────┘    └─────────┘
                              ▲
                       上一级的乘法结果
```

### 第一问：周期与频率

**每一级的延迟**：

```text
级 1（乘法）：T_cq + T_mult + T_setup = 10 + 60 + 10 = 80 ns
级 2（加法）：T_cq + T_add  + T_setup = 10 + 45 + 10 = 65 ns
```

**周期 = max(80, 65) = 80 ns**

```text
f = 1 / 80ns = 12.5 MHz
```

**答案**：

```text
1. 80     # ns（周期）
2. 12.5   # MHz（频率）
```

**加速比**：

```text
125 / 80 ≈ 1.5625×   （理想两级 = 2×，实际被 60ns 的乘法拖累）
```

> ⚠️ 若你少了 `T_cq`/`T_setup`，会得到 60 / 45，周期 60ns → 16.67 MHz。**错**：寄存器开销每一级都要算。

### 第二问：为什么不需要 bubble

题目给的标准答案（原文）：

> **For this exercise, ONE instruction consists of BOTH an addition and a multiplication.** 加法和乘法不是两条独立指令，而是**同一条指令的两个阶段** —— 就像 EX 和 MEM 处理同一条指令。
> **We do not have any bubbles in this particular circuit because none of the individual instructions depend on each other.**

用中文写：

```text
3. 不需要 bubble。因为乘法级和加法级是同一条指令的两个阶段（类似 EX 和 MEM），
   不同指令之间没有数据依赖 —— in1/in2 每条周期由外部提供，
   后一条指令不读取前一条指令的结果。
```

**`.ref` 里的两行 0（`CYCLE=0000` 和 `0001`）**：

| 周期 | 值 | 原因 |
|---|---|---|
| `0000` | `00000000` | 电路的**初始状态**（尚未开始任何计算）|
| `0001` | `00000000` | 级 2 **空**（乘法结果还在级 1 → 级 2 的流水线寄存器里）|
| `0002` | `00001100` | 第一条结果出来：`-4 × 0x70 = -448`… 见下 |

**核对 `.ref` 第一组数据**：

```text
CYCLE,OUT1,    IN1,      IN2
0010, 00001100, 11111100, 01110000
```

`IN1 = 0xFC = -4`（8 位补码），`IN2 = 0x70 = 112`。若运算是 `(a+b)×c` 或 `a×b + 累加`，需按 `ex4.circ` 实际结构核对 —— **具体算式以电路为准，此处待补充精确推导**。

⚠️ 注意：`.ref` 的输出在 `CYCLE=0111` 之后输入变成 `00000000`，输出停在 `00111111`，说明加速器停止输入后流水线**排空/保持不变**。测试只比你输出的前 N 行。

**自检**：

```bash
python3 test.py   # 应看到 PASSED test: Exercise 4: Pipe that Line
diff tests/out/ex4-test.out tests/out/ex4-test.ref && echo "完全一致"
```

---

## Exercise 5 · Debugging Circuits

**任务**：`num_checker` 电路有 bug，输入 32 位数 `num`，输出：

| 输出 | 含义 |
|---|---|
| `is_odd` | 奇数 |
| `is_even` | 偶数 |
| `is_zero` | 等于 0 |
| `greater_than_fifteen` | **大于 15** |
| `five_or_ten` | 等于 5 或 10 |

**骨架观察**（从 `ex5.circ` 提取的信号名）：

```text
num (32) ──Splitter──┬─ bit_zero   (1)      ← inst[0]
                     ├─ bits_15    (28)     ← 高 28 位
                     ├─ bits_five  (3)      ← 低 3 位
                     ├─ five_rest_zero (29)
                     └─ bits_ten   (2)
```

判断逻辑很明显：

```text
is_odd   =  bit_zero
is_zero  =  ~OR(bits_15, ...)         → 32 位全 0
gt_15    =  高 28 位中有任何一位为 1  → bits_15 经 OR 归约
five     =  (bits_five == 0b101)
ten      =  (bits_ten  == 0b10)  ← 2 位只能表示 0..3，这是 bug 苗头
```

> 💡 **`bits_ten` 只有 2 位**，而 10 = `0b1010` 需要 **4 位**。这是最可能的 bug 之一 —— splitter 位宽取错了。

**调试流程（lab 要求照做）**：

```text
① 打开 tests/ex5-test.circ（File → Open）
   → 看到 ROM（大块十六进制数字）依次喂数进去
② 右键 num_checker → View num_checker（或 Simulate 选项卡切换）
③ 在 num_checker 里搭一个「周期计数器」小装置：
       Clock ──► 寄存器 ──► (自己回环) ，寄存器名字改成 num_cycles
   → 双击寄存器可改名
④ Simulate → Timing Diagram → Add or Remove Signals
   → 移除 halt、CYCLE
   → 加上 num_cycles（在 num_checker 下拉里找）
   → 把 num_cycles、num 右键改成 Signed Decimal
⑤ 用 Poke 工具点着看 bit_zero、is_odd 等内部信号
⑥ 一次 tick 一个完整周期，对照 .ref 看哪一拍开始错
⑦ python3 test.py → 打开 tests/out/ex5-test.out 与 ex5-test.ref
   → 找出第一个不一致的行 = 出错的输入值
⑧ 回到电路修 bug，重复
```

**`ex5_answers.txt` 题目（11 题，题干未完整抓取）**：

从 lab 原文可确认的问题：

```text
Q?:  "What time step is the circuit at? The answer should be an integer."
     → 第 8 步 tick 后作答。起点 CYCLE=0，每次 tick +1。
       具体到第几拍取决于题目顺序，按你实际 tick 次数记。

Q?:  "Why are such bubbles unnecessary for this particular circuit?"
     （这是 Ex4 的问题，不要写进 ex5）

其余 Q1-Q11：待补充（需要完整 lab 页面正文/截图）
```

**已知的 `.ref` 数据**（供对照）：

```text
CYCLE,num,                              is_odd,is_even,is_zero,gt15,five_or_ten
0000, 00000000000000000000000011001000,   0,    1,      0,     1,     0   → num=200
0001, 00000000000000000000000011001001,   1,    0,      0,     1,     0   → num=201
0010, 00000000000000000000000000000000,   0,    1,      1,     0,     0   → num=0
0011, 00000000000000000000000000001111,   1,    0,      0,     0,     0   → num=15 ★边界
0100, 00000000000000000000000000010001,   1,    0,      0,     1,     0   → num=17
0101, 00000000000000000000000000000101,   1,    0,      0,     0,     1   → num=5
0110, 00000000000000000000000000001010,   0,    1,      0,     0,     1   → num=10
```

> ⭐ **`num = 15` 那一行**是边界测试：`15 > 15` 为假 → `greater_than_fifteen = 0`。如果 bug 电路用了 `>=`，这里就会错。

> ⚠️ **`num = 200` 那行**：`11001000` 最高位是 1。如果 `greater_than_fifteen` 用**有符号**比较（200 → -56），结果会变成 0 而 `.ref` 要求 1。这是第二个高频 bug。

**修复清单**：

```text
□ greater_than_fifteen 用「高 28 位 OR 归约」（等价于无符号 >= 16），
  不要用有符号 Comparator 直接比 15
□ bits_ten 必须是 4 位（取 inst[3:0]），常量 0b1010
□ is_zero 用 32 位全宽 OR 归约后取反
□ num_cycles 计数器只用于调试，不影响功能（放哪都行，但别接到输出）
```

---

## 知识串联：这个 Lab 在 Project 3 里怎么用

```text
Ex1 (ImmGen)  ──┐
Ex2 (BrUn)    ──┼──► Project 3 的 CPU 数据通路
Ex3/Ex4       ──┼──► 理解「为什么流水线更快」→ 讲清加速比的计算
Ex5 (调试)    ──┴──► 学会用 Timing Diagram + ROM 测试台 debug CPU
```

Project 3 的测试台结构和 `ex5-test.circ` **几乎一样**：ROM 存指令 → 时钟驱动 → CPU 输出逐拍比对。所以 Ex5 的调试流程值得反复看。

---

## 待补内容

| 项 | 原因 |
|---|---|
| `ex5_answers.txt` Q1–Q11 完整题干 | lab 页面正文未完整抓取（只有片段）|
| Ex4 `.ref` 的精确算式推导 | 需确认 `ex4.circ` 的加法输入接法（累加？常数？）|
| Ex3 是否还问频率 | `ex3_answers.txt` 只有 1 个 bullet，推测只问周期 |

**补齐方式**：给我 lab 06 页面的完整正文（或题号截图），我逐题补上。

---

## 相关

- Labs 总览：[[knowledge/course/cs61c-labs/index\|🧪 Labs]]
- 数据通路：[[knowledge/course/cs61c/20-Datapath-I\|L20 Datapath-I]] · [[knowledge/course/cs61c/21-Datapath-II\|L21 Datapath-II]]
- 控制逻辑：[[knowledge/course/cs61c/22-Control\|L22 Control]]
- 流水线：[[knowledge/course/cs61c/23-Pipeline-I\|L23 Pipeline-I]] · [[knowledge/course/cs61c/24-Pipeline-II-Hazards\|L24 Pipeline-II-Hazards]] · [[knowledge/course/cs61c/25-Pipeline-III-Hazards-Continued\|L25 Pipeline-III]]
- 状态元件基础：[[knowledge/course/cs61c/18-SDS-III-State\|L18 SDS-III State]] · [[knowledge/course/cs61c/19-SDS-IV-Blocks-ALU\|L19 SDS-IV Blocks & ALU]]
- 指令格式（立即数位域）：[[knowledge/course/cs61c/13-RISC-V-Instruction-Formats-I\|L13 指令格式 I]] · [[knowledge/course/cs61c/14-RISC-V-Instruction-Formats-II\|L14 指令格式 II]]
