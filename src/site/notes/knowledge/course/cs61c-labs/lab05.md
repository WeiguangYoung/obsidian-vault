---
{"dg-publish":true,"permalink":"/knowledge/course/cs61c-labs/lab05/","title":"CS61C FA26 · Lab 05 · Logisim 组合逻辑","tags":["CS61C","lab","lab05","Logisim","数字电路","组合逻辑","课程笔记"],"dg-note-properties":{"title":"CS61C FA26 · Lab 05 · Logisim 组合逻辑","tags":["CS61C","lab","lab05","Logisim","数字电路","组合逻辑","课程笔记"],"date":"2026-11-20","source":"https://cs61c.org/fa26/labs/lab05/"}}
---


# Lab 05 · Logisim 组合逻辑

> UC Berkeley · Great Ideas in Computer Architecture · Fall 2026
> 代码仓库：`github.com/61c-teach/fa26-lab-starter`
> 本地路径：`/home/yang/.openclaw/workspace/fa26-lab-starter/lab05/`

返回 [[knowledge/course/cs61c-labs/index\|🧪 Labs 总览]] · [[knowledge/course/cs61c/index\|📚 CS61C 目录]]

---

## 这一课在讲什么

课程里**第一次真正"动手搭硬件"**：用 **Logisim Evolution** 画电路图（schematic），从 `AND` 门一路搭到 MUX、Adder 和 16 位旋转器。
它与 Project 3（CPU 设计）强绑定 —— 这里练的每个动作（放元件、连线、Tunnel 标签、建子电路、接 Clock）在 P3 里都要重复上百次，
因此**重点不是答案，是把手法练成肌肉记忆**。

> 官方定位："It is critical prep for Project 3." 对应讲义：L17 讲门电路 / MUX / 加法器，L18 讲寄存器与时钟。

---

## 环境准备

### 拉文件 + 下载 Logisim（官方定制版，必须用）

在本地 `labs` 目录下（不是 lab05 目录）：

```bash
git pull origin main     # 先同步自己前几次 lab 的改动
git pull starter main    # 再拉 starter 仓库的最新分支

bash tools/download_tools.sh          # 下载 logisim + venus
bash tools/download_tools.sh logisim  # 只下 Logisim
```

脚本实际做的事（读 `tools/download_tools.sh` 源码确认）：

```bash
curl -L "https://inst.eecs.berkeley.edu/~cs61c/$SEMESTER/tools/logisim-evolution-latest.jar" \
     -o logisim-evolution.jar
```

脚本里 `SEMESTER=sp26`，从 Berkeley 的 `inst.eecs` 站点拉最新构建（本地 jar 更新时 `curl -z` 会跳过下载）。

> ⚠️ **不要用网上下载的其他 Logisim 版本**（包括官方 Logisim Evolution 的正式 release）。
> 课程版带 bugfix + 课程定制内容，测试脚本依赖它的 `-tty` 表格输出格式。

### 启动 + 常见报错

```bash
cd lab05
java -jar ../tools/logisim-evolution.jar
```

会弹出一个"略显古老"的 Swing 窗口；没弹出来就回终端看 Java 报错。

| 现象 | 原因 | 处理 |
|---|---|---|
| `java.lang.UnsupportedClassVersionError: class file version 60.0 … up to 52.0` | Java 版本太旧（班级编译目标是 Java 16，你本地是 Java 8） | 按 Lab 0 的说明升级 JDK |
| Windows 下双击 jar 无反应/报错 | 系统把 `javaw` 指向了 Microsoft Store 的占位程序 | 搜索 "app execution alias" → "Manage app execution aliases" → 关掉 Java 对应项 |
| Git Bash 下路径问题 | Git Bash 的路径转换 | 用相对路径 `../tools/...` |

---

## Logisim 界面速览

| 区域 | 位置 | 作用 |
|---|---|---|
| **工具栏（toolbar）** | 窗口顶部 | `Poke` / `Edit` / `Wiring` / `Text` 工具，加上 `AND` `OR` `NOT` `Pin` 等快捷元件图标 |
| **元件库（explorer）** | 左侧树形列表 | `Wiring`、`Gates`、`Plexers`、`Arithmetic`、`Memory` 等库 |
| **电路选择器（circuit selector）** | 左侧树的最上层 | 列出当前 `.circ` 文件里的所有子电路，**双击进入** |
| **属性面板（properties）** | 左下（或右下） | 设置选中元件的位宽 `Data Bits`、朝向 `Facing`、标签 `Label` 等 |
| **缩放（zoom）** | **左下角** | 官方点名推荐：这个 lab 之后几周你都会用到 |

**四个工具（toolbar 按钮）**：`Poke` 点引脚翻转 0/1、点导线看当前值；`Edit` 选中/拖动元件，**连线也用它**；
`Wiring` 直接画线；`Text` 加注释。官方建议画线用 `Edit`：从引脚端点按住拖到门电路输入端。

---

## ⭐ Exercise 1 · 认识界面，画第一张电路

**目标**：用 `AND` 门画最简电路，练 `Poke` 和连线规则。

```text
① 工具栏 AND → 画布点一下，放下 AND 门
② Input Pin → 左边放两个输入；Output Pin → 右边放一个输出
③ 切 Edit 工具，拖拽连线：两个输入引脚 → 门的两输入；门输出 → 输出引脚
④ 切 Poke 工具，点输入引脚翻转 0/1，观察输出
```

**线只能画水平/竖直**。画 L 形要两段：先横拖一段松手，再从那段线的**末端端点**起竖直拖。

### 连线的"交叉 vs 结点"

| 操作 | 结果 |
|---|---|
| 拖线时**直接经过**另一根线、不停 | **不连接**（只是跨过去） |
| 拖线时**停在**另一根线上松手 | **产生 junction（大圆点）→ 电气连接** |

> ⚠️ 最容易出的错：**以为没连其实连了，或以为连了其实没连**。连完用 `Poke` 点导线验证。

### 导线颜色表（调试主武器）

| 颜色 | 含义 |
|---|---|
| **深绿** | 1 位导线，值为 **0** |
| **亮绿** | 1 位导线，值为 **1** |
| **黑** | **多位（multi-bit）导线**，位宽由元件属性 `Data Bits` 决定 |
| **红**（值显示 `EEEE`） | 导线有**冲突值**（例如一个 0 和一个 1 被连在一起） |
| **蓝**（值显示 `UUUU`） | 导线**悬空（floating）**，没有已知值 |
| **橙** | **位宽不匹配**：两端元件位宽不同（如 1 位引脚接到 2 位输出） |

> 💡 `U` = undefined：输出引脚显示 `U` 通常意味着"没连上"或"上游有悬空输入"。
> 有色觉障碍可在 `File → Preferences → Simulation Tab` 换配色，但要告知 course staff。

---

## ⭐ Exercise 2 · 子电路（Sub-Circuits）

**目标**：`ex2.circ` 里用门电路搭出 `NAND2`、`NOR2`、`XOR2`、`MUX2`、`MUX4` 五个子电路。

### 什么是 sub-circuit

像 C 的 helper function：`.circ` 里可以有多个 **sub-circuit**，各占 circuit selector 一行、双击进入编辑，在其他电路里当一个元件用。

> ⚠️ **命名限制**：不允许空格/符号、不允许数字开头、不允许与内置元件重名 ——
> 所以示范叫 `AND2`，因为 `AND` 是内置元件名。

`ex2.circ` 双击 `AND2` 就是示范：两个 1 位输入 `A`、`B`，一个 `RESULT` 输出，中间一个 `AND` 门。

> 要求：`NAND` / `NOR` / `XOR` / `MUX` 内置元件虽然存在但**本练习禁用**，只能用 `AND`、`OR`、`NOT`。
> 目的是理解子电路怎么用，不是"再搭一遍门"。

### 五个子电路的做法

| 子电路 | 引脚 | 搭法 |
|---|---|---|
| `NAND2` | `A` `B` → `RESULT` | `AND` 输出接 `NOT` |
| `NOR2` | `A` `B` → `RESULT` | `OR` 输出接 `NOT` |
| `XOR2` | `A` `B` → `RESULT` | `A·B̄ + Ā·B`（两个 `AND` + 一个 `OR` + 两个 `NOT`） |
| `MUX2` | `A` `B` + `SEL` → `RESULT` | `SEL=0` 选 `A`，`SEL=1` 选 `B` |
| `MUX4` | `A` `B` `C` `D` + `SEL0` `SEL1` → `RESULT` | 三级：`SEL0` 选 `(A,B)` 与 `(C,D)`，再 `SEL1` 二选一 |

```text
MUX2: RESULT = SEL ? B : A  =  (SEL & B) | (~SEL & A)

MUX4（用两个 MUX2 嵌套）：
low_half  = MUX2(A, B, SEL0)      # SEL0=0 → A, 1 → B
high_half = MUX2(C, D, SEL0)      # SEL0=0 → C, 1 → D
RESULT    = MUX2(low_half, high_half, SEL1)
```

> ⭐ MUX4 里直接拖两个 **MUX2 实例**进来 —— 这就是「复用已有子电路」的典型场景，整个 Project 3 都这么干。

### 真值表（动手前先写，能省半天时间）

| A | B | NAND | NOR | XOR |
|---|---|---|---|---|
| 0 | 0 | 1 | 1 | 0 |
| 0 | 1 | 1 | 0 | 1 |
| 1 | 0 | 1 | 0 | 1 |
| 1 | 1 | 0 | 0 | 0 |

MUX2 即 `SEL=0` 选 `A`、`SEL=1` 选 `B`；MUX4 的 `SEL1 SEL0 = 00/01/10/11` 依次选 `A/B/C/D`。

> ⚠️ `SEL0` 是**低位**、`SEL1` 是**高位**（与 `ex2-test.ref` 表头 `…,SEL0,SEL1,…` 一致，列顺序别记反）。

> ❗ **别改子电路名字、别新建子电路**（harness 按名字找），**别动 starter 给的引脚**（见下文"最大的坑"）。

---

## ⭐ Exercise 3 · Add Machine（加法器 + 寄存器）

**目标**：`ex3.circ` 的 `AddMachine` 骨架只有三个引脚 —— `clk`（输入 1 位）、`ADD_OUT`（输出 8 位）、`REG_OUT`（输出 8 位）。

### 期望行为（从 `tests/out/ex3-test.ref` 反推）

```text
CYCLE,ADD_OUT,REG_OUT
00000000,00000001,00000000     # REG=0 → ADD = REG+1 = 1
00000001,00000010,00000001     # 时钟上升沿后 REG=1 → ADD=2
00000010,00000011,00000010     # REG=2 → ADD=3
...
```

即一个**累加器（accumulator）**：组合部分算「寄存器值 + 1」，寄存器在 `clk` 上升沿吃进新值 ——
**典型的"组合 + 状态"结构**。

### 怎么搭

| 元件 | 位置 | 属性设置 |
|---|---|---|
| **Adder** | `Arithmetic` 库 | `Data Bits = 8` |
| **Register** | `Memory` 库 | `Data Bits = 8`，默认**上升沿（rising edge）**触发 |
| **Clock** | `Wiring` 库 | 频率 `1 Hz` 即可（测试用 `-tty` 模式，时钟不重要） |
| **Tunnel** | `Wiring` 库 | `Label` 相同的 Tunnel 视为同一根导线（"无线"连接），用来减少跨线 |

> starter 的 `ex3.circ` 里已预置一个 `Splitter` 和一个 `Tunnel`，暗示了出题人期望的画法。

### 搭法（数据流成一圈）

```text
① 放 Adder（Data Bits=8），B 输入接 Constant（Data Bits=8, Value=0x01）
② 放 Register（Data Bits=8），输出同时接 REG_OUT 引脚与 Adder 的 A 输入
③ Adder 输出接 ADD_OUT 引脚，并接回 Register 的数据输入 —— 构成 "REG → Adder → REG" 的累加闭环
④ Clock 接 Register 的时钟输入
⑤ 检查：8 位线应为黑色；出现橙色说明位宽不匹配
```

> ⚠️ `Constant` 元件在 `Wiring` 库里，默认位宽可能是 1，**必须手动改成 8**，否则橙线。

---

## ⭐ Exercise 4 · Rotate Right（16 位循环右移）

**目标**：`ex4.circ` 里 5 个电路 —— **主电路 `RotRight`**（要你实现），
外加辅助子电路 `rot1` / `rot2` / `rot4` / `rot8`（分别循环右移 1 / 2 / 4 / 8 位）。

### 主电路引脚

`AMOUNT`（输入，4 位，取值 0–15 = 移多少位）、`INPUT0`（输入，16 位）、`OUTPUT0`（输出，16 位）。

### 期望行为（来自 `tests/out/ex4-test.ref`）

harness 把 `INPUT0` 固定为 `1100011000000000`（16 位二进制，即 `0xC600`），
`AMOUNT` 每行递增 1，输出就是**逐位右移、移出的最低位从最高位绕回来**：

```text
AMOUNT=0000 → 1100011000000000
AMOUNT=0001 → 0110001100000000
AMOUNT=0010 → 0011000110000000
...
AMOUNT=1111 → 1000110000000001
```

### 为什么拆成 rot1/rot2/rot4/rot8（log shifter）

`AMOUNT` 正好 4 位，可以**按二进制位分解**成 4 级 MUX2 级联 —— 这就是 **log shifter（对数移位器）**：

```text
AMOUNT = b3 b2 b1 b0
x      先过 b0 控制的一级：b0 ? rot1(x) : x
       再过 b1 一级：      b1 ? rot2(·) : ·
       再过 b2 一级：      b2 ? rot4(·) : ·
       再过 b3 一级：      b3 ? rot8(·) : ·
```

> 顺序无关（各级交换律成立），但**位与级要对齐**：`AMOUNT[0]`↔`rot1`、`[1]`↔`rot2`、`[2]`↔`rot4`、`[3]`↔`rot8`。

### 每个 rotN 怎么写

**`rot1`（1 位循环右移）—— 一个 Splitter 就够**：用 Splitter（`Bit Width In`=16、`Fan Out`=2），
一栏 `1` 取输入 bit 15、另一栏 `15` 取 bit 14..0，两路交叉接到输出的 `[0]` 与 `[15..1]`。

关系式：`OUTPUT[i] = x[(i+1) mod 16]`。`rotN`（N = 2, 4, 8）同理，只是拆成 `N + (16-N)` 两段。

> 💡 **更省事的做法**：`rot2 = rot1 串联两次`，`rot4 = rot2 两次`，`rot8 = rot4 两次` ——
> 又是 sub-circuit 复用。行为对就行，题目没限制实现方式（但**别改 rot1/rot2/rot4/rot8 的名字**）。

**主电路接线**：`INPUT0` 依次穿过 4 个 MUX2（每级另一个输入接对应的 `rotN` 输出，选择端接 `AMOUNT` 的一位），
最后到 `OUTPUT0`。用 `Splitter` 把 4 位 `AMOUNT` 拆成 4 根 1 位线。

> ⚠️ 注意 `ex4.circ` 的 toolbar 里有 `Plexers` 库（lib 2）—— 里面有现成的 `Multiplexer`。
> 本 lab 没有禁止用它（禁止清单只在 Exercise 2 里），但**自己用 MUX2 子电路更符合练习目的**。

---

## 如何运行测试

### 正确命令

```bash
cd lab05
python3 test.py
```

> ⚠️ **`test.py` 不接受 `python3 test.py ex2` 这类参数** —— 源码里 `__main__` 直接 `run_tests(tests)`，
> 传参不会筛选；它会**顺次跑 ex2 / ex3 / ex4**。

输出形如：

```text
Testing files...
PASSED test: Exercise 2: Sub-Circuits
FAILED test: Exercise 3: Add Machine (Did not match expected output)
FAILED test: Exercise 4: Rotate
Passed 1/3 tests
```

### 测试原理 + 对不上时怎么查

```text
① 先跑 check_hashes.py：校验 tests/ 下 harness 与 .ref 的 MD5，改过就直接退出
② java -jar ../tools/logisim-evolution.jar -tty table,binary,csv tests/exN-test.circ
   把你的电路嵌进 harness，离线跑出真值表
③ stdout 与 tests/out/exN-test.ref 逐行对比，实际输出另存到 exN-test.out
```

```bash
diff tests/out/ex2-test.ref tests/out/ex2-test.out   # 查差异行
```

输出文件**每列一个引脚、每行一组输入输出**：

```text
CYCLE,A,B,C,D,SEL0,SEL1,NAND_OUT,NOR_OUT,XOR_OUT,MUX2_OUT,MUX4_OUT
000,0,0,1,0,0,1,1,1,0,0,1
```

> ⚠️ **绝对不要改 `tests/` 下的 `.circ` / `.ref`**：`check_hashes.py` 写死了 MD5，改了就报
> `was changed from starter` 并中止整个测试。唯一允许的改动是 `test.py` 的 `fix_circ()` 自动修正导入路径
> （`desc="file#..."`），由脚本自己完成。`.gitignore` 只有 `!tests/out`，你的 `.out` 不会污染 git。

---

## ⚠️ 最大的坑：不要移动 staff 提供的引脚

harness 通过**名字**匹配你电路里的引脚（`ADD_OUT`、`REG_OUT`、`INPUT0`……），
而且 `ex2.circ`/`ex3.circ`/`ex4.circ` 里这些引脚在 XML 中带 `locked="true"`。

| 你的操作 | 后果 |
|---|---|
| 删除引脚 | 测试找不到 → 输出列缺失或全 `U` |
| 改引脚名（含大小写） | 列名对不上 → 比对失败 |
| 改引脚位宽 / 方向 | 列宽或接法不对 → 全部 mismatch、全 `E` |
| 挪位置但保持连线 | 通常能过，但线容易断，**不推荐** |

> 官方原话："do not change the names of the subcircuits or create new ones, or your circuit may not work properly."
> 引脚名就是**接口约定（interface contract）**。

> ❗ **不要"删了重放一个同名引脚"**：新引脚不继承 `locked`，默认位宽 1、默认方向 input。
> 名字看着一样，实际是 1 位——测试表那列变成 `0`/`1`，与参考的 8/16 位对不上。
> 要点位置就整体框选移动，或先接好新线再删旧线。

---

## ⚠️ 第二大的坑：Logisim 不自动保存

> 官方原文："Logisim doesn't auto-save your work. Remember to save (and commit) frequently as you work!"

- `Ctrl+S`（macOS `Cmd+S`）**每做好一点就按一下**；隔一阵 `git add ex2.circ ex3.circ ex4.circ && git commit -m "lab05 progress"`
- 崩溃/断电后只会回到最后一次保存的状态 —— 没有自动恢复
- undo 栈虽深但不无限：连错一大片线时，**删掉重画往往比 undo 快**

---

## 常见错误速查

| 现象 | 原因 | 处理 |
|---|---|---|
| 输出引脚显示 `U` | 没连上 / 上游悬空 | `Poke` 逐段点导线看值 |
| 导线变红 `E` | 两个不同值接到同一根线 | 找结点（大圆点），拆掉冲突 |
| 导线变橙 | 位宽不匹配 | 检查 `Data Bits`，必要时加 Splitter |
| `test.py` 报 `was changed from starter` | 改了 `tests/` 下的文件 | `git checkout starter main -- tests/` 恢复 |
| `test.py` 报 Java 错误 | Java 版本太旧 | 升级 JDK |
| 电路"看似在位、但测试全错" | 引脚被删重放 | 重新确认引脚位宽/方向 |
| 只有某几行错 | 大概是某组输入边界（如全 0、全 1） | `diff` 出具体行再看对应的输入组合 |

---

## 文件清单

| 文件 | 状态 | 说明 |
|---|---|---|
| `ex2.circ` | ⬜ 待实现 | 5 个子电路：`NAND2` / `NOR2` / `XOR2` / `MUX2` / `MUX4`（`AND2` 已给示范） |
| `ex3.circ` | ⬜ 待实现 | `AddMachine`：8 位累加器（Adder + Register + Clock） |
| `ex4.circ` | ⬜ 待实现 | `RotRight` 主电路 + `rot1` / `rot2` / `rot4` / `rot8` 四个子电路 |
| `test.py` | 勿改 | 测试驱动，顺次跑三个 exercise |
| `check_hashes.py` | 勿改 | 校验 `tests/` 未被修改（MD5 写死） |
| `.gitignore` | 勿改 | 内容为 `!tests/out` |
| `tests/ex2-test.circ` | 勿改 | Exercise 2 harness，内嵌 `ex2.circ` |
| `tests/ex3-test.circ` | 勿改 | Exercise 3 harness，含 Clock / Register / Adder / Comparator / Constant |
| `tests/ex4-test.circ` | 勿改 | Exercise 4 harness，按 `CYCLE` 递增 `AMOUNT` |
| `tests/out/ex2-test.ref` | 勿改 | 参考输出：`A,B,C,D,SEL0,SEL1` → 5 个输出 |
| `tests/out/ex3-test.ref` | 勿改 | 参考输出：`CYCLE` → `ADD_OUT, REG_OUT` |
| `tests/out/ex4-test.ref` | 勿改 | 参考输出：`CYCLE` → `ROTRIGHT_OUT`（16 行） |

---

## 小抄：本 lab 用到的元件

| 元件 | 库 | 关键属性 | 用途 |
|---|---|---|---|
| `Pin` | Wiring | `Output?` / `Data Bits` / `Label` | 输入输出引脚 |
| `AND` `OR` `NOT` `XOR` | Gates | `Data Bits` | 门电路 |
| `Splitter` | Wiring | `Bit Width In` / `Fan Out` / `Bit x y` | 把多位线拆成多根，或反向合并 |
| `Tunnel` | Wiring | `Label` | 同名无线连接，图更干净 |
| `Constant` | Wiring | `Data Bits` / `Value` | 常数（如 `+1`） |
| `Adder` | Arithmetic | `Data Bits` | 加法器 |
| `Register` | Memory | `Data Bits` / 触发沿 | 时钟驱动的存储 |
| `Clock` | Wiring | `Frequency` / `Phase` | 时钟信号 |
| `Multiplexer` | Plexers | `Select Bits` / `Data Bits` | 多路选择器（Ex4 可用） |
| `Comparator` | Arithmetic | — | 测试 harness 内部用 |

### Splitter 用法（最容易卡住的一个）

`Splitter` 既能**拆**（1 根 N 位 → 多根低位线）也能**合**（多根 → 1 根 N 位），
方向由接线方式决定，**元件本身没有方向**。

拆 16 位为 1+15 的示例：

```text
Bit Width In : 16
Fan Out      : 2
Bit 0        : 1        ← 上面那根扇出线 = 输入的最高 1 位
Bit 1        : 15       ← 下面那根扇出线 = 输入的其余 15 位
```

> `Bit x` 的数字就是"该扇出线取的输入位区间"，属性面板会画出对应扇出线。
> 拿不准就先接好，用 `Poke` 试几组值验证。

---

## 待补充

| 项 | 原因 |
|---|---|
| Lab Slides 的逐步截图 | Google Slides 未抓取；正文已覆盖全部操作步骤 |
| Exercise 3 / 4 的官方文字描述 | 抓取的 md 在 Exercise 2 测试段后截断，Ex3/Ex4 由 starter 电路 + `.ref` 反推 |
| `rotN` 官方推荐实现 | 截断内容中没有；本笔记给了两种等价做法 |

> 已核对 `.circ` 文件：`AddMachine` 引脚位宽 1/8/8，`RotRight` 引脚位宽 4/16/16，`rot1~rot8` 各两个 16 位引脚。

---

## 相关

- 讲义：[[knowledge/course/cs61c/17-SDS-II-Combinational-Logic\|L17 SDS II · Combinational Logic]]
- 讲义：[[knowledge/course/cs61c/18-SDS-III-State\|L18 SDS III · State]]
- Labs 总览：[[knowledge/course/cs61c-labs/index\|🧪 Labs]]
- 上一课：[[knowledge/course/cs61c-labs/lab03\|Lab 03 · RISC-V & Venus]]
