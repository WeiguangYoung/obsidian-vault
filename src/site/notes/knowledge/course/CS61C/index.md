---
{"dg-publish":true,"permalink":"/knowledge/course/CS61C/index/","title":"CS61C 目录导航","tags":["导航","索引","CS61C","计算机体系结构"],"dg-note-properties":{"title":"CS61C 目录导航","tags":["导航","索引","CS61C","计算机体系结构"]}}
---


# 📚 CS61C · 计算机体系结构

> UC Berkeley · Great Ideas in Computer Architecture（a.k.a. Machine Structures）
> Fall 2026 · 授课：Dan Garcia
> 难度 🌟🌟🌟🌟 · 预计 100 小时 · 先修 CS61A + CS61B

返回 [[knowledge/course/index\|📚 课程资源]] · [[首页\|🏠 首页]]

---

## 讲次索引

| 讲 | 主题 | 笔记 |
|:--:|:--|:--|
| 1 | Course Introduction | [[knowledge/course/CS61C/01-Course-Introduction\|01-Course-Introduction]] |
| 2 | Number Representation | [[knowledge/course/CS61C/02-Number-Representation\|02-Number-Representation]] |
| 3 | Introduction to C | [[knowledge/course/CS61C/03-Introduction-to-C\|03-Introduction-to-C]] |
| 4 | C Pointers, Arrays, and Strings | [[knowledge/course/CS61C/04-Pointers-Arrays-Strings\|04-Pointers-Arrays-Strings]] |
| 5 | C Memory (Mis)Management | [[knowledge/course/CS61C/05-Memory-Management\|05-Memory-Management]] |
| 6 | C Generics and Function Pointers | [[knowledge/course/CS61C/06-Generics-Function-Pointers\|06-Generics-Function-Pointers]] |
| 7 | IEC Prefixes, Bitwise Ops, ADTs | [[knowledge/course/CS61C/07-Bitwise-Ops-ADTs\|07-Bitwise-Ops-ADTs]] |
| 8 | IEEE 754 Floating Point | [[knowledge/course/CS61C/08-Floating-Point\|08-Floating-Point]] |
| 9 | RISC-V Basics | [[knowledge/course/CS61C/09-RISC-V-Basics\|09-RISC-V-Basics]] |
| 10 | RISC-V Data Transfer | [[knowledge/course/CS61C/10-RISC-V-Data-Transfer\|10-RISC-V-Data-Transfer]] |
| 11 | RISC-V Control Flow, Bitwise Ops | [[knowledge/course/CS61C/11-RISC-V-Control-Flow\|11-RISC-V-Control-Flow]] |
| 12 | RISC-V Procedures | [[knowledge/course/CS61C/12-RISC-V-Procedures\|12-RISC-V-Procedures]] |
| 13 | RISC-V Instruction Formats I | [[knowledge/course/CS61C/13-RISC-V-Instruction-Formats-I\|13-RISC-V-Instruction-Formats-I]] |
| 14 | RISC-V Instruction Formats II | [[knowledge/course/CS61C/14-RISC-V-Instruction-Formats-II\|14-RISC-V-Instruction-Formats-II]] |
| 15 | Running a Program (CALL) | [[knowledge/course/CS61C/15-CALL\|15-CALL]] |
| 16 | SDS I: Intro to SDS | [[knowledge/course/CS61C/16-SDS-I-Intro\|16-SDS-I-Intro]] |
| 17 | SDS II: Combinational Logic | [[knowledge/course/CS61C/17-SDS-II-Combinational-Logic\|17-SDS-II-Combinational-Logic]] |
| 18 | SDS III: State | [[knowledge/course/CS61C/18-SDS-III-State\|18-SDS-III-State]] |
| 19 | SDS IV: Blocks and ALU | [[knowledge/course/CS61C/19-SDS-IV-Blocks-ALU\|19-SDS-IV-Blocks-ALU]] |
| 20 | Datapath I | [[knowledge/course/CS61C/20-Datapath-I\|20-Datapath-I]] |
| 21 | Datapath II | [[knowledge/course/CS61C/21-Datapath-II\|21-Datapath-II]] |
| 22 | Control | [[knowledge/course/CS61C/22-Control\|22-Control]] |
| 23 | Pipeline I: 5-Stage Pipeline | [[knowledge/course/CS61C/23-Pipeline-I\|23-Pipeline-I]] |
| 24 | Pipeline II: Hazards | [[knowledge/course/CS61C/24-Pipeline-II-Hazards\|24-Pipeline-II-Hazards]] |
| 25 | Pipeline III: Hazards, continued | [[knowledge/course/CS61C/25-Pipeline-III-Hazards-Continued\|25-Pipeline-III-Hazards-Continued]] |
| 26 | Caches I | [[knowledge/course/CS61C/26-Caches-I\|26-Caches-I]] |
| 27 | Caches II | [[knowledge/course/CS61C/27-Caches-II\|27-Caches-II]] |
| 28 | Caches III | [[knowledge/course/CS61C/28-Caches-III\|28-Caches-III]] |
| 29 | Caches IV | [[knowledge/course/CS61C/29-Caches-IV\|29-Caches-IV]] |

**进度：29 / 29 讲 ✅ 全部完成**

### 六大 Great Ideas 分布

| # | 思想 | 对应讲次 |
|:--:|:--|:--|
| 1 | **Abstraction** | L15 CALL · L26 内存幻觉 |
| 2 | **Moore's Law** | L16 晶体管数增长 |
| 3 | **Locality / Memory Hierarchy** | L26–L29 缓存 |
| 4 | **Parallelism** | L20 五层并行 · L23 流水线 |
| 5 | **Performance Measurement** | L23 latency vs throughput · L29 AMAT |
| 6 | **Dependability via Redundancy** | L17 Majority Circuit |

### 学习路径

```
L01–L02  数据表示        (2 讲)
L03–L08  C 语言          (6 讲)
L09–L15  RISC-V 指令集   (7 讲)
L16–L19  数字电路        (4 讲)
L20–L22  Datapath + 控制 (3 讲)
L23–L25  流水线          (3 讲)
L26–L29  缓存            (4 讲)
```

---

## 课程 Project（4 个）

| # | 内容 |
|:--:|:--|
| 1 | 用 C 写 Game of Life |
| 2 | 用 **RISC-V 汇编**写神经网络识别 MNIST 手写数字 |
| 3 | 用 **Logisim** 搭二级流水线 CPU，并在上面跑汇编 |
| 4 | 用 **OpenMP / SIMD** 优化矩阵运算，实现简易 Numpy |

---

## 官方资源

| 资源 | 链接 |
|:--|:--|
| 课程主页 | https://cs61c.org/fa26/ |
| 日历 / 日程 | https://cs61c.org/fa26/calendar/ |
| 讲义课件 | https://cs61c.org/fa26/lectures/lecNN/ |
| 参考速查卡 | https://cs61c.org/fa26/pdfs/resources/reference-card.pdf |
| 历年考试 | https://cs61c.org/fa26/resources/exams/ |
| 常见错误 | https://cs61c.org/fa26/resources/common-errors/ |

## 作业实现

| 资源 | 链接 |
|:--|:--|
| PKUFlyingPig/CS61C-summer20 | https://github.com/PKUFlyingPig/CS61C-summer20 |
| InsideEmpire | https://github.com/InsideEmpire |
| RisingUppercut | https://github.com/RisingUppercut |

---

## 说明

笔记正文按讲课内容整理，术语保留英文原文以便与课件、作业对照。
