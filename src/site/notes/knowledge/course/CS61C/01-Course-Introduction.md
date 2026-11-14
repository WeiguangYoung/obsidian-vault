---
{"dg-publish":true,"permalink":"/knowledge/course/CS61C/01-Course-Introduction/","title":"CS61C L01 · Course Introduction","tags":["CS61C","计算机体系结构","课程笔记","RISC-V","C语言"],"dg-note-properties":{"title":"CS61C L01 · Course Introduction","tags":["CS61C","计算机体系结构","课程笔记","RISC-V","C语言"],"date":"2026-11-13","source":"https://cs61c.org/fa26/"}}
---


# CS61C FA26 · L01 Course Introduction

> UC Berkeley · **Great Ideas in Computer Architecture**（a.k.a. Machine Structures）
> 授课：Teaching Professor **Dan Garcia**（称呼 "Dan"，不是 "Prof. Garcia"）
> 课程主页：<https://cs61c.org/fa26/>
> 本讲对应课件 01-Course-Introduction（54 页）

---

## 一、这门课到底讲什么

### 1.1 核心定位

> **CS61C is NOT about C Programming…!! It is about the hardware-software interface.**

- **目标**：让程序员知道如何获得**尽可能高的性能**
- **C 语言的角色**：它是**手段**，不是**目的**
  - C 比 Snap!、Python、Java **更接近底层硬件**
  - 能用 C 显式地**驾驭硬件并行性**
  - 从而可以用高层语言去讨论硬件特性

### 1.2 Old-School vs New-School

**传统「机器结构」的抽象栈**（Old School）：

```
Application (例如浏览器)
        ↓
Compiler / Assembler
        ↓
Operating System (Mac OS X)
        ↓
Hardware: Processor / Memory / I/O system
          Datapath & Control
          Instruction Set Architecture   ← CS61C 关注点
          Digital Design
          Circuit Design
          Transistors
          Fabrication
```

**新式机器结构复杂得多**（New School）：

| 层级 | 并行形式 | 例子 |
|---|---|---|
| **Warehouse-Scale Computer** | Parallel Requests | 把搜索请求「Cats」分配给某台机器 |
| **Core** | Parallel Threads | 一个核心跑 Lookup、另一个跑 Ads |
| **Functional Block / Exec Unit** | Parallel Instructions | 5 条流水线指令同时推进 |
| **Logic Gates** | 全部门同时工作 | 硬件描述层面 |

> **核心结论：Harness Parallelism & Achieve High Performance**
> 现代机器结构的关键词是**并行**——从数据中心到逻辑门，每一层都在并行。

---

## 二、六大 Great Ideas

这是整门课的骨架。**六条线索贯穿所有内容**，也是本讲反复强调的结构。

| # | Great Idea | 一句话解释 |
|:--:|---|---|
| **1** | **Abstraction**（Layers of Representation / Interpretation） | 每一层都是对下一层的抽象，同一段信息在不同层有不同解释 |
| **2** | **Moore's Law** | 晶体管密度指数增长（1965 提出），正在放缓 |
| **3** | **Principle of Locality / Memory Hierarchy** | 程序访问有局部性，所以可以做多级存储 |
| **4** | **Parallelism** | 指令级、线程级并行 |
| **5** | **Performance Measurement & Improvement** | 量化性能，找到瓶颈 |
| **6** | **Dependability via Redundancy** | 用冗余换取可靠性 |

### 2.1 Great Idea #1：Abstraction（抽象）

**核心思想：同一份信息在不同层次有不同的「表示」和「解释」。**

课程用一条完整链路演示「编译 → 汇编 → 机器码」：

```
C 代码                    temp = v[k];
                          v[k] = v[k+1];
                          v[k+1] = temp;

     ↓ Compiler

汇编（RISC-V）            lw  x3, 0(x10)
                          lw  x4, 4(x10)
                          sw  x4, 0(x10)
                          sw  x3, 4(x10)

     ↓ Assembler

机器码（二进制）          1101 1110 0010 1110 0001 0000 1110 0001 ...
```

> **Anything can be a number — data, instructions, etc.**
> 任何东西都可以表示成数字：数据、指令，都是如此。

链路末端是**硬件架构描述**（数据通路 + 控制），图中出现的关键信号：

```
pc       → iAddrA → IMEM → inst
inst     → [rs1] [rs2] [31:7] [24:20] [19:15] [11:7] 字段拆分
pc + 4   → 下一条指令地址
Reg      ← wb 回写
ALU      ← 两个寄存器操作数 + imm
DMEM     ← DataA/AddrD/DataD
```

**CS61A 的对照**：在 CS61A 里「任何东西都可以是 Python 名字」；在 CS61C 里「任何东西都可以是数字」。**两层抽象的共同本质：用统一表示容纳异质内容。**

### 2.2 Great Idea #2：Moore's Law（摩尔定律）

- **1965 年**由 Gordon Moore（Intel 联合创始人，1950 年伯克利本科）提出
- 内容：集成电路上晶体管数量约每两年翻一番
- **2005 年**：仍然有效，但已经有「放缓」的迹象
- **今天（FA26 的讲法）**：**Seems to be tapering** —— 明显减速

### 2.3 Great Idea #3：Principle of Locality / Memory Hierarchy

**存储层次 = 用「距离」换「速度」。**

课程引用 Jim Gray（图灵奖得主，1966 年伯克利本科、1969 年博士）的经典类比：**How Far Away is the Data?** —— 把各级存储的延迟映射成「数据离你多远」。

典型层次（自上而下，容量递增、速度递减）：

```
寄存器  →  On-chip Cache (L1/L2)  →  主存 DRAM  →  磁盘 / SSD  →  网络 / 归档
   最快                                                  最慢
   最小                                                  最大
```

### 2.4 Great Idea #4：Parallelism（并行）

课程区分两种主要形式：

**① Instruction-Level Parallelism（ILP，指令级并行）**

流水线的直观写照：

> 在第 3 个时间片里：
> - **Instruction 1 正在执行（executing）**
> - **Instruction 2 正在译码（decoding）**
> - **Instruction 3 正在取指（fetching）**

三条指令同时在流水线的不同阶段推进，这就是 ILP。

**② Thread-Level Parallelism（TLP，线程级并行）**

多个线程在多个核心上同时运行。

**③ 补充：Amdahl's Law（阿姆达尔定律）**

- 提出者 Gene Amdahl（计算机先驱）
- 含义：**系统加速比受限于不可并行化的那部分**
- 课程标注为「Caveat!（更多内容后面讲）」

### 2.5 Great Idea #5：Performance Measurement & Improvement

**两条主线：**

**① 让应用匹配底层硬件，充分利用：**

- **Locality**（局部性）
- **Parallelism**（并行）
- **Special hardware features**（专用硬件特性，例如矩阵运算指令）

**② 两个必须区分的性能指标：**

| 指标 | 关注什么 |
|---|---|
| **Latency（延迟）** | 多久能启动并把问题做完 |
| **Throughput（吞吐）** | 给定时间内能完成多少任务；以及「跑起来之后能快多少」 |

### 2.6 Great Idea #6：Dependability via Redundancy（用冗余换可靠）

**问题的来源**：

> 晶体管的意外行为可能来自**宇宙射线（cosmic rays）**导致的意外电子流动，以及其他原因。

**解决思路：设计冗余，让单个部件失效不会导致整个系统失效。**

**投票机制示意**（2-of-3 agree）：

```
   分量 A: 1  ┐
   分量 B: 1  ├─→ 2 of 3 agree ✅  → 输出 2
   分量 C: 2  ┘
```

**冗余在各层级的应用：**

| 层级 | 冗余方案 | 能容忍的故障 |
|---|---|---|
| **内存** | **ECC**（Error Correcting Code）冗余位 | 丢 1 bit，数据不丢 |
| **磁盘** | **RAID**（Redundant Arrays of Independent Disks） | 坏 1 块盘，数据不丢 |
| **数据中心** | 冗余数据中心 | 挂 1 个机房，Internet 服务仍在线 |

> **关键洞察**：**Increasing transistor density reduces the cost of redundancy**
> 晶体管密度提升让「冗余的成本」越来越低——这正是摩尔定律带来的红利之一。

---

## 三、为什么今天学计算机体系结构很激动

### 3.1 Reason 1：Constraints 变了

| 约束 | 变化 |
|---|---|
| **Moore's Law** | 正在终结（ending） |
| **Power limitations** | 功耗墙 |
| **Amdahl's Law** | 并行加速有天花板 |

### 3.2 Reason 2：领域专用计算时代（Domain-Specific Computing）

- 部署的设备数量持续增长，但**没有单一的 killer application**
- 需求多样化 → **架构多样化**
- **Machine Learning 是大多数领域的共同需求**

### 3.3 Old vs New Conventional Wisdom

**旧共识（Old Conventional Wisdom）**：

```
Moore's Law + Dennard Scaling
        ↓
每年更快、更便宜、功耗更低 的「通用计算机」
```

- 黄金时期：**每周 1% 的性能提升**
- 当时认为做并行或专用计算机是**愚蠢的**——因为等你设计完，下一代通用芯片已经超过你了

**新共识（New Conventional Wisdom）**：

> **每个领域都需要异构系统（heterogeneous systems）。**

组成要素：

- 多个处理器核心（multiple processor cores）
- **GPU**
- **NPU**
- **加速器（accelerators）**
- 各种接口（interfaces）
- 复杂内存层次

**例子：**

| 例子 | 说明 |
|---|---|
| **Apple M14** | 异构 SoC |
| **Google TPU3** | 专为神经网络训练设计的专用引擎，部署在云上；**1024 颗芯片，> 100 PFLOPS** |

### 3.4 一个彩蛋

> **Patterson 和 Hennessy 获得 2017 年图灵奖**（《计算机体系结构：量化研究方法》作者）。
> **Prof. Patterson 已确认将讲授 CS61C 的倒数第二堂课。**

> **Innovations in computer architecture often defy conventional wisdom.**
> 体系结构的创新，往往来自对常识的挑战。

---

## 四、课程须知（Practical Info）

### 4.1 基本资源

| 项 | 位置 |
|---|---|
| 课程网站 | <https://cs61c.org/> |
| 授课人 | Dan Garcia（**叫 "Dan"**，不要叫 "Prof. Garcia"） |
| 课件 / COURSE NOTES / 可选教材阅读 | 课程网站 |
| Discussion worksheets | 课程网站 |
| Assignments / Past exams | 课程网站 |
| 公告与讨论 | **EdStem**（所有公告、讨论、澄清都在这里） |

> **今日必读：Course Policies 页面。**

### 4.2 课程安排

| 项 | 时间 |
|---|---|
| **lab00（Setup）** | 周五发布 |
| **Welcome Survey** | 随 lab00 一起发布 |
| └ 需提交 section 偏好 | **REGULAR / DEEPER / VIDEO** 三选一，周五截止 |
| **Discussion 01 / Lab 01 / Office Hours** | 下周开始 |

### 4.3 迟交与弹性政策（Lateness and Accommodations）

**三级弹性**：

| 情况 | 处理方式 |
|---|---|
| 只差一点时间 | **预批的自动延期**（pre-approved automated extension） |
| 需要更多时间 | 可申请**一次性的自定义延期**（ONE-TIME custom extension） |
| 生活出状况 | **直接找老师谈**（talk to us!） |

### 4.4 评分制度

> **Absolute Grading（绝对评分），不 curve（不按比例给分）。**

伯克利官方指引（lower division 课程）：

- 典型 GPA 区间：**2.8 – 3.3**
- 大致分布：**35% A、45% B、13% C、7% D/F**

**DSP 学生**：请尽快提交 Letter of Accommodation。

**关于实习面试（课程原话）**：

> 面试官会用技术问题「烤」你，靠的是**你会什么、能做什么**，而不是你的 GPA。
> **CS61C 给你提供了很好的谈资。**

> ⚠️ **具体日程可能微调**：「Always in motion, the future is…」——讲座、作业、lab 的时间安排可能小幅变动。

---

## 五、学术诚信（Collaboration Policy）

> **We expect that what you hand in for this course is yours.**
> 交给这门课的东西，必须是**你自己的**。

### 5.1 不可接受的行为

- ❌ 抄袭其他同学的答案
- ❌ 从网上抄（或以网上的答案起手）
- ❌ 制造让别人抄袭的机会——**公开 GitHub**、登录后不锁屏就走、随手留下笔记

### 5.2 可以做什么

- ✅ **帮助别人调试（teach others to debug）**
- ✅ 在 Ed 上回答**帮助同学 debug** 的问题
- ❌ 但**不要分享代码或代码片段**
- ❌ 不要和其他同学分享思路、想法、代码，也不要一起画白板
  - 原话：**「The pseudocode/algorithm is sometimes the entire point!」**
  - HKN /  tutoring 帮忙过伪代码，也**不允许**

### 5.3 作业与项目的协作边界

| 类型 | 规则 |
|---|---|
| **Homework / Labs** | 必须**独自完成**（ALONE） |
| **Projects** | 可以**单人**，也可以**两人组队** |
| └ 组队要求 | 两人承诺**同步协作**，不能和其他组交叉合作 |

### 5.4 违规的后果

> **We have tools, developed over many years, for detecting this. You WILL be caught.**

| 项 | 说明 |
|---|---|
| **最低处罚** | **降一个字母等级** + **上报学生行为中心** |
| **责任划分** | **给的人和收的人同罪**（equally culpable） |
| **查处量** | 每学期 **50–100 名学生**被抓 |
| **检查范围** | 也查**中间产物**（intermediate student work），占近期案例的 **60%** |

> 任何阶段的抄袭都不可接受。

### 5.5 AI 使用边界（重要）

| ✅ 可以 | 不可以 |
|---|---|
| 「教我这个 CS61C 概念」 | 「这是项目要求，帮我做」（整个项目） |
| └ 技巧：说「把我当成小学生教我」，再逐步到初中 → 高中 → 大学 | 「帮我写这个函数」 |
| 「有没有助记法帮我记住 X？」 | 「这是我的坏代码，帮我 debug」 |
| 「给我生成一些练习题」 | 「帮我的作业/项目/lab 做」 |
| 「教我 C 或 RISC-V 的调试技巧」 | |

> **判断标准**：
> 用来**备考/学习** → 基本 OK
> 用来**做作业 / 项目 / lab** → 不 OK

---

## 六、教师团队的期望

### 6.1 目标一：让体验尽可能愉快且有收获

- 幽默、热情、嘉宾讲座、科技新闻
- 有趣且有挑战的项目与作业
- **亲学生的 accommodations 政策**
- 在 section 里的同伴讨论机会，**学习如何学习（learn how to learn）**

### 6.2 目标二：维持伯克利的卓越标准

- 项目与考试的**严格程度和往年一致**
- 「**你还是会有充足的机会去展示成功**」

### 6.3 最终目标

> **Our ultimate goal is for you to learn the material, not to stress you out.**
> If you are feeling lost, please reach out.
> **It is much better to do so than to violate our trust. We are in this together!**

---

## 七、本讲总结（课程自己的收尾）

> **CS61C: Learn 6 great ideas in computer architecture to enable high performance programming via parallelism, not just learn C.**

1. **Abstraction**（Layers of Representation / Interpretation）
2. **Moore's Law**
3. **Principle of Locality / Memory Hierarchy**
4. **Parallelism**
5. **Performance Measurement and Improvement**
6. **Dependability via Redundancy**

---

## 八、我的笔记与理解

### 8.1 这门课真正的逻辑主线

L1 表面是「课程介绍」，实际已经把**整门课的骨架**铺完了：

```
六大 Great Ideas
    ↓ 支撑
并行（Parallelism）
    ↓ 实现
高性能编程（High Performance Programming）
    ↓ 载体
硬件-软件接口（Hardware-Software Interface）
    ↓ 工具
C 语言 + RISC-V 汇编
```

**「不是学 C」这句话很关键**：C 只是因为你必须有一个能触碰硬件的手柄。后面 L03~L08 讲 C，L09~L14 讲 RISC-V，L16~L29 讲数字设计/流水线/缓存——全部都在服务「怎么榨干硬件」。

### 8.2 六条 Idea 的相互关系

它们不是并列的六条，而是有依赖关系：

| 关系 | 说明 |
|---|---|
| **Abstraction** 是**方法论** | 所有其他五条都建立在「层次抽象」之上 |
| **Moore's Law** 是**历史动力** | 它带来了红利（冗余成本下降），也带来了结束后的危机（必须转并行） |
| **Locality** 是**性能的来源** | 存储层次能work，全靠程序访问有局部性 |
| **Parallelism** 是**唯一出路** | 摩尔定律结束后，性能提升只能靠并行（ILP / TLP / 数据并行） |
| **Performance Measurement** 是**裁判** | 没有量化就没有优化；Latency vs Throughput 是最基本的两个视角 |
| **Redundancy** 是**保费** | 规模越大越容易出错，冗余是必须付的成本 |

### 8.3 值得记住的几个细节

**① 「Old vs New」对比的深意**

旧时代认为**做专用硬件是愚蠢的**（通用芯片迭代太快会碾过去）。现在反过来——**通用芯片停滞，专用硬件才成为唯一出路**。这个认知反转，正是 CS61C 存在的最强理由。

**② Amdahl's Law 的位置**

L01 只提了名字就说「more later」。但它是**并行时代的定罪书**：再怎么堆核心，串行部分永远卡着你。后面讲流水线和多核时必然会回来。

**③ AI 使用政策的设计很聪明**

它按「**是否用于交付物**」划线，而不是按「是否用了 AI」。这是可执行的标准——学生知道边界在哪。

**④ 「中间产物也查」这句话的分量**

60% 的案例来自中间产物检查。这意味检测不是只看最终提交，而是**比对过程**（比如 commit 历史）。

### 8.4 与后续讲次的衔接

| 后续 | 内容 | 对应本篇 |
|---|---|---|
| **L02** | Number Representation | Idea #1 Abstraction（数字表示是第一层抽象） |
| **L03–L08** | C 语言基础 → 浮点 | 「C 是触碰硬件的手柄」 |
| **L09–L14** | RISC-V 指令集 | Idea #1 的汇编层 |
| **L16–L22** | 数字逻辑 → Datapath → Control | Idea #6 的硬件实现 |
| **L23–L25** | 5 级流水线 → Hazards | Idea #4 ILP |
| **L26–L29** | Caches I–IV | Idea #3 Locality / Memory Hierarchy |

---

## 九、术语速查

| 术语 | 中文 | 说明 |
|---|---|---|
| **ISA** | 指令集架构 | Instruction Set Architecture，硬件与软件的契约 |
| **ILP** | 指令级并行 | 流水线让多条指令同时处于不同阶段 |
| **TLP** | 线程级并行 | 多核心多线程 |
| **Amdahl's Law** | 阿姆达尔定律 | 加速比受串行部分限制 |
| **Dennard Scaling** | 登纳德缩放 | 晶体管变小功耗同步降低（已失效） |
| **Locality** | 局部性 | 时间局部性 + 空间局部性 |
| **ECC** | 纠错码 | Error Correcting Code，内存冗余 |
| **RAID** | 磁盘阵列 | Redundant Arrays of Independent Disks |
| **DSP** | 残障学生服务 | Disabled Students' Program |
| **HKN** | 荣誉学会 | Eta Kappa Nu，提供辅导 |
| **Ed / EdStem** | 课程讨论平台 | 所有公告与讨论所在 |

---

## 相关

- 课程导航：[[knowledge/course/index\|课程 目录导航]]
- 下一讲：[[knowledge/course/CS61C/02-Number-Representation\|L02 Number Representation]]
