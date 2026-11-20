---
{"dg-publish":true,"permalink":"/knowledge/course/cs61c-labs/Projects/","title":"CS61C FA26 · Projects 总览","tags":["CS61C","project","导航","snek","CS61Classify","CS61CPU","RISC-V","机器学习","Logisim","CPU"],"dg-note-properties":{"title":"CS61C FA26 · Projects 总览","tags":["CS61C","project","导航","snek","CS61Classify","CS61CPU","RISC-V","机器学习","Logisim","CPU"],"date":"2026-11-20","source":"https://cs61c.org/fa26/projects/"}}
---


# 🛠️ CS61C FA26 · Projects

> UC Berkeley · Great Ideas in Computer Architecture · Fall 2026
> 本文只做**介绍与规划**，不含具体实现。做题时对照官方页面。

返回 [[knowledge/course/cs61c-labs/index\|🧪 实验与项目总览]] · [[knowledge/course/cs61c/index\|📚 CS61C 目录]] · [[首页\|🏠 首页]]

---

## 三个 Project 一览

| # | 项目 | 主题 | 工具 | 截止 | 拆分 |
|:--:|:--|:--|:--|:--|:--|
| **1** | snek | 用 C 写一个可玩的贪吃蛇 | C · Make | 9/20 | 单阶段 |
| **2** | CS61Classify | RISC-V 汇编实现手写数字分类 | RISC-V · Venus | 9/25 / 10/8 | Part A + B |
| **3** | CS61CPU | 用 Logisim 搭一个真能跑 RISC-V 的 CPU | Logisim | 10/21 / **11/20** | Part A + B |

三个 project 都允许**独立完成或两人组队**。组队需在 Gradar 上注册，且**一旦建组就不能再换人**。

> ⚠️ **Project 3 Part B 的截止就是 11/20（今天）**。若已错过，Project 3 是唯一能靠 Part A 拿部分分的——Part A 在 10/21 已截止。

---

## Project 1 · snek

**一句话**：写一个能在终端里玩的贪吃蛇。

| 项 | 内容 |
|:--|:--|
| 语言 | C |
| 运行环境 | **必须在 hive 机器上**（本地跑不了） |
| 依赖 | Lab 1–2 必修；Lecture 2–4、Lecture 5 的内存布局部分推荐 |
| 提交 | Gradescope，可无限次提交，最后一次为准 |

### 只准改三个文件

```
snake.c          蛇的移动、碰撞判定
game.c           棋盘状态、游戏循环
custom_tests.c   你自己写的测试
```

其余文件（`game.h`、`snake_utils.c`、`asserts.c`、`interactive_snake.c` 等）**不能动**。

### 骨架结构

```
fa26-proj1-starter/
├── src/
│   ├── snake.c              ← 待实现
│   ├── game.c               ← 待实现
│   ├── custom_tests.c       ← 待实现
│   ├── game.h               结构体定义（只读）
│   ├── snake_utils.c        工具函数（只读）
│   ├── asserts.c            断言（只读）
│   ├── unit_tests.c         单元测试（只读）
│   └── interactive_snake.c  交互入口（只读）
├── tests/                   官方测试用例
├── tools/
└── Makefile
```

### 八个 Task 的递进关系

从数据结构的生命周期串起来看，而不是八个孤立小题：

```
Task 1  create_default_game   建局（分配内存、初始化结构体）
   ↓
Task 2  free_game             收尾（释放内存，防泄漏）
   ↓
Task 3  print_board           渲染（把状态画成字符）
   ↓
Task 4  update_game           推进一帧（核心：算下一格→移头→挪尾）
   ↓
Task 5  load_board            读文件建局（read_line 是难点）
   ↓
Task 6  initialize_snakes     从棋盘反推蛇（find_head 找头）
   ↓
Task 7  main                  串起来
   ↓
Task 8  组队/反馈表
```

Task 4 内部还拆了 5 个小步，是整个 project 的重心。

### 特有的坑：Provenance Recorder

这门课用**溯源记录器**防代写：需要在 hive 机器上装 VS Code 插件，全程录屏记录你的操作，提交时把 `.provenance` 和 `.provenance-manifest` 一起交。

- 必须连 hive 机器（VS Code Remote-SSH）
- 记录日志要一起提交
- 组队时两个人都要录

> 这也是「必须在 hive 上做」的真正原因——不是性能，是溯源。

### 调试策略

课程组明确说：**提交 ticket 前必须证明你调试过**。没调试痕迹、或明显不会调试的，会被要求先做完 Lab 2 再来问。

这跟 Lab 2 的 GDB/Valgrind 训练是直接挂钩的。

### 做完能干嘛

```bash
make interactive-snake
./interactive-snake
```

用 `wasd` 控制，`]` 加速、`[` 减速，`-d 0.5` 调时间步长。

---

## Project 2 · CS61Classify

**一句话**：用 RISC-V 汇编写一个手写数字分类器（简单机器学习）。

| 项 | 内容 |
|:--|:--|
| 语言 | RISC-V 汇编 |
| 运行环境 | **本地机器即可**，不需要 hive |
| 拆分 | Part A（9/25）→ Part B（10/8） |
| 调试 | Venus Web 界面（与 Lab 3 同一套） |
| 依赖 | 调用约定、函数调用、堆、文件 I/O |

### 学的是「怎么用汇编组织一个程序」

这不是写几个函数就完事，而是完整走一遍：

```
读文件 → 矩阵运算 → 分类 → 写文件
```

### 骨架结构（10 个 .s 文件）

```
fa26-proj2-starter/
├── src/
│   ├── main.s            主流程（串起各步）
│   ├── read_matrix.s     从文件读矩阵
│   ├── write_matrix.s    结果写回文件
│   ├── matmul.s          矩阵乘法
│   ├── relu.s            激活函数
│   ├── argmax.s          取最大值下标（分类结果）
│   ├── dot.s             点积
│   ├── abs.s             绝对值
│   ├── classify.s        分类主逻辑
│   └── utils.s           工具（打印等）
├── tests/                官方测试
├── test-src/             测试源码
├── framework.py          测试框架
├── unittests.py
├── studenttests.py
├── convert.py
└── test.sh               下载 Venus、跑测试
```

> 文件名就是**调用链的顺序**：main → read_matrix → matmul → relu → argmax → write_matrix。

### 官方 OH 前置要求（很硬）

去 office hour 前必须：

1. **给每个寄存器加用途注释**——"this holds i" 这种模糊描述会被打回，要求写清具体含义
2. 读过调用约定指南
3. 跑过 **calling convention checker**，且所有报错已解决
4. 跑过 **memcheck**，且所有内存错误已解决
5. 看过调试视频
6. 按模板填 OH Queue，低质量描述（如"它不工作"）会被直接删

> 第 3、4 条和 Lab 3 的 Exercise 2/3 完全对应。Lab 3 没认真做，这里会卡住。

### 环境准备

```bash
cd 61c-proj2
bash test.sh download_tools     # 只跑一次
java -jar tools/venus.jar . -dm # 挂载到 Venus Web
```

然后在 `https://venus.cs61c.org` 里 `mount local vmfs`。

### 调试视频

课程提供了一个专门讲**内存错误 + 调用约定错误**的播放列表，官方要求看。这是 Project 2 最容易翻车的地方——汇编里的栈失衡、寄存器没保存，编译器不会帮你兜。

---

## Project 3 · CS61CPU

**一句话**：用 Logisim 搭一个能真正执行 RISC-V 指令的 CPU。

| 项 | 内容 |
|:--|:--|
| 工具 | Logisim Evolution |
| 运行环境 | **本地机器**，不需要 hive |
| 拆分 | Part A（10/21）→ Part B（**11/20**） |
| 依赖 | Lab 5（组合逻辑）、Lab 6（立即数、分支比较、流水线） |

### 这是 Lab 5 + Lab 6 的合体

| Lab 学到的 | Project 3 里用在哪 |
|:--|:--|
| Lab 5 组合逻辑、子电路 | 所有 `.circ` 的搭法 |
| Lab 6 Ex1 立即数生成 | `imm-gen.circ` |
| Lab 6 Ex2 分支比较 | `branch-comp.circ` |
| Lab 6 Ex3–5 CPU | `cpu.circ` 主体 |

Lab 6 没吃透，这个 project 会非常痛苦。

### 骨架结构

```
fa26-proj3-starter/
├── cpu/
│   ├── cpu.circ             CPU 主体（核心）
│   ├── alu.circ             ALU
│   ├── regfile.circ         寄存器堆
│   ├── imm-gen.circ         立即数生成
│   ├── branch-comp.circ     分支比较
│   ├── control-logic.circ   控制逻辑
│   ├── mem.circ             存储器
│   ├── partial-load.circ    部分加载（lb/lh 等）
│   └── partial-store.circ   部分存储（sb/sh 等）
├── harnesses/               测试外壳
├── tests/                   测试用例
├── test.sh                  下载工具、跑测试
└── tools/
```

### 环境准备

```bash
cd 61c-proj3
bash test.sh download_tools
java -jar tools/logisim-evolution.jar
```

### 三个必须提前知道的坑

1. **不能用网上下载的 Logisim**，必须用课程定制版（有 bug 修复和课程专用组件）
2. **不能移动 staff 提供的引脚**——引脚名是测试 CSV 的表头，动了测试必挂
3. **Logisim 不自动保存**，勤存勤提交

> 这三条和 Lab 5、Lab 6 里踩的是同一批坑。

---

## 三个 Project 的能力递进

```
Proj1  snek            用 C 组织一个完整程序
        ↓              内存分配/释放、结构体、文件读写、测试
Proj2  CS61Classify    用汇编组织一个完整程序
        ↓              调用约定、栈管理、堆、文件 I/O
Proj3  CS61CPU         用硬件搭一个能跑程序的机器
                       → 回到 Proj2：你写的汇编，跑在你搭的 CPU 上
```

第三条是这门课的设计意图：**Project 2 写的汇编，理论上能跑在 Project 3 搭的 CPU 上**。

---

## 时间规划建议

按截止日期倒推，三个 project 是**串行**的，没有并行空间：

| 时间 | 任务 | 前置 |
|:--|:--|:--|
| 9 月上旬 | Lab 1–2 → **Proj1** | C 基础、GDB |
| 9 月中 | **Proj2 Part A** | Lab 3（Venus）、调用约定 |
| 10 月初 | **Proj2 Part B** | 同上 + 文件 I/O |
| 10 月中 | **Proj3 Part A** | Lab 5、Lab 6 Ex1–2 |
| 11 月中 | **Proj3 Part B** | Lab 6 全部 |

> 每个 project 都卡在前一个的知识点上，跳着做会两头空。

---

## 与 Labs 的对应关系

| Project | 必须先做完的 Lab | 对应讲义 |
|:--|:--|:--|
| Proj1 snek | Lab 1（C 基础）、Lab 2（调试） | L03–L05 |
| Proj2 CS61Classify | Lab 3（RISC-V & Venus） | L09–L15 |
| Proj3 CS61CPU | Lab 5（组合逻辑）、Lab 6（CPU） | L17–L24 |

---

## 参考

- 官方 Projects 页：`https://cs61c.org/fa26/projects/`
- 骨架仓库（公开，可直接 clone 看结构）：
  - `github.com/61c-teach/fa26-proj1-starter`
  - `github.com/61c-teach/fa26-proj2-starter`
  - `github.com/61c-teach/fa26-proj3-starter`
- 常见错误汇总：`https://cs61c.org/fa26/resources/common-errors/`

> 骨架仓库是公开的，但**学生仓库**（`61c-student/fa26-projN-USERNAME`）需要 Gradar 注册后自动创建，clone 时要换成自己的用户名。

---

返回 [[knowledge/course/cs61c-labs/index\|🧪 实验与项目总览]] · [[knowledge/course/cs61c/index\|📚 CS61C 目录]]