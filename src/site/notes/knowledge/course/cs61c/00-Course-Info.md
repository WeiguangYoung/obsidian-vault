---
{"dg-publish":true,"permalink":"/knowledge/course/cs61c/00-Course-Info/","title":"CS61C · Course Info","tags":["CS61C","课程介绍","资源","计算机体系结构"],"dg-note-properties":{"title":"CS61C · Course Info","tags":["CS61C","课程介绍","资源","计算机体系结构"],"date":"2026-11-17","source":"https://cs61c.org/fa26/"}}
---


# CS61C FA26 · Course Info

> UC Berkeley · **Great Ideas in Computer Architecture**（a.k.a. Machine Structures）
> Fall 2026 · 授课：Dan Garcia

返回 [[knowledge/course/cs61c/index\|📚 CS61C 目录]] · [[knowledge/course/index\|📚 课程资源]] · [[首页\|🏠 首页]]

---

## 一、课程概况

| 项 | 内容 |
|:--|:--|
| 开课院校 | UC Berkeley |
| 学期 | Fall 2026 |
| 授课 | Dan Garcia |
| 讲次 | 37 讲（L35 为客座讲，无课件） |
| 难度 | 🌟🌟🌟🌟 |
| 预计投入 | 100 小时 |
| 先修 | CS61A + CS61B |

**课程主线**：从 C 语言与数据表示出发 → RISC-V 指令集 → 数字电路 → 处理器数据通路与流水线 → 存储层次与缓存 → 并行计算与虚拟内存。核心是建立「软件如何落到硬件上执行」的完整心智模型。

---

## 二、官方资源

| 资源 | 链接 | 说明 |
|:--|:--|:--|
| 课程主页 | https://cs61c.org/fa26/ | Fall 2026 |
| 日历 / 日程 | https://cs61c.org/fa26/calendar/ | 讲次、作业、考试时间 |
| 讲义课件 | https://cs61c.org/fa26/lectures/lecNN/ | 把 `NN` 换成 01~37 |
| 参考速查卡 | https://cs61c.org/fa26/pdfs/resources/reference-card.pdf | RISC-V 指令速查，考前必看 |
| 历年考试 | https://cs61c.org/fa26/resources/exams/ | 期中期末真题 |
| 常见错误 | https://cs61c.org/fa26/resources/common-errors/ | 踩坑合集 |

---

## 三、作业实现仓库

| 资源 | 链接 | 说明 |
|:--|:--|:--|
| PKUFlyingPig/CS61C-summer20 | https://github.com/PKUFlyingPig/CS61C-summer20 | csdiy 作者本人的完整实现 |
| InsideEmpire | https://github.com/InsideEmpire | Fall20 作业（仓库名已变，进主页找） |
| RisingUppercut | https://github.com/RisingUppercut | Fall24 作业（仓库名已变，进主页找） |

---

## 四、课程 Project（4 个）

| # | 内容 |
|:--:|:--|
| 1 | 用 C 写 Game of Life |
| 2 | 用 **RISC-V 汇编**写神经网络识别 MNIST 手写数字 |
| 3 | 用 **Logisim** 搭二级流水线 CPU，并在上面跑汇编 |
| 4 | 用 **OpenMP / SIMD** 优化矩阵运算，实现简易 Numpy |

---

## 五、六大 Great Ideas

| # | 思想 | 对应讲次 |
|:--:|:--|:--|
| 1 | **Abstraction** | L15 CALL · L26 内存幻觉 |
| 2 | **Moore's Law** | L16 晶体管数增长 |
| 3 | **Locality / Memory Hierarchy** | L26–L29 缓存 |
| 4 | **Parallelism** | L20 五层并行 · L23 流水线 · **L30–L34 并行五讲** |
| 5 | **Performance Measurement** | L23 latency vs throughput · L29 AMAT |
| 6 | **Dependability via Redundancy** | L17 Majority Circuit |

---

## 六、学习路径

```
L01–L02  数据表示         (2 讲)
L03–L08  C 语言           (6 讲)
L09–L15  RISC-V 指令集    (7 讲)
L16–L19  数字电路         (4 讲)
L20–L22  Datapath + 控制  (3 讲)
L23–L25  流水线           (3 讲)
L26–L29  缓存             (4 讲)
L30–L34  并行             (5 讲)
L35      客座讲（无课件）
L36–L37  虚拟内存         (2 讲)
```

---

## 七、说明

- 笔记正文按讲课内容整理，术语保留英文原文以便与课件、作业对照。
- **L35 无课件**：`[Guest Lecture] - Parallelism VI`，课程页面未挂 Slides / Drive，无 PDF 可整理。