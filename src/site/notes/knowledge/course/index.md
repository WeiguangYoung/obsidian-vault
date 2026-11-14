---
{"dg-publish":true,"permalink":"/knowledge/course/index/","title":"课程 目录导航","tags":["导航","索引","课程","计算机体系结构","操作系统"],"dg-note-properties":{"title":"课程 目录导航","tags":["导航","索引","课程","计算机体系结构","操作系统"]}}
---


# 📚 课程 · 目录导航

> 系统性的公开课程学习资源索引。

返回 [[首页\|🏠 首页]]

---

## CS61C · UC Berkeley

> CS61C: Great Ideas in Computer Architecture · Fall 2026
> 32 讲 · 难度 🌟🌟🌟🌟 · 预计 100 小时 · 先修 CS61A + CS61B

### 官方

| 资源 | 链接 | 说明 |
|:--|:--|:--|
| 课程主页 | https://cs61c.org/fa26/ | Fall 2026 |
| 日历 / 日程 | https://cs61c.org/fa26/calendar/ | 讲次、作业、考试时间 |
| 讲义课件 | https://cs61c.org/fa26/lectures/lecNN/ | 把 `NN` 换成 01~32 |
| 参考速查卡 | https://cs61c.org/fa26/pdfs/resources/reference-card.pdf | RISC-V 指令速查，考前必看 |
| 历年考试 | https://cs61c.org/fa26/resources/exams/ | 期中期末真题 |
| 常见错误 | https://cs61c.org/fa26/resources/common-errors/ | 踩坑合集 |

### 作业实现仓库

| 资源 | 链接 | 说明 |
|:--|:--|:--|
| PKUFlyingPig/CS61C-summer20 | https://github.com/PKUFlyingPig/CS61C-summer20 | csdiy 作者本人的完整实现 |
| InsideEmpire | https://github.com/InsideEmpire | Fall20 作业（仓库名已变，进主页找）|
| RisingUppercut | https://github.com/RisingUppercut | Fall24 作业（仓库名已变，进主页找）|

### 课程 Project（4 个）

| # | 内容 |
|:--:|:--|
| 1 | 用 C 写 Game of Life |
| 2 | 用 **RISC-V 汇编**写神经网络识别 MNIST 手写数字 |
| 3 | 用 **Logisim** 搭二级流水线 CPU，并在上面跑汇编 |
| 4 | 用 **OpenMP / SIMD** 优化矩阵运算，实现简易 Numpy |

---

## MIT 6.1810 · Operating System Engineering

> MIT 6.1810（原 6.S081）· Fall 2026
> 23 讲 + 9 个实验 · 难度 🌟🌟🌟🌟🌟 · 预计 150 小时
> 先修：体系结构 + 扎实 C 功底 + RISC-V 汇编

### 官方

| 资源 | 链接 | 说明 |
|:--|:--|:--|
| 课程主页 | https://pdos.csail.mit.edu/6.1810/2026/ | Fall 2026 |
| 日程表 | https://pdos.csail.mit.edu/6.1810/2026/schedule.html | 23 讲 + 作业 + 考试安排 |
| 课程概览 | https://pdos.csail.mit.edu/6.1810/2026/overview.html | 评分构成、先修要求 |
| 实验总览 | https://pdos.csail.mit.edu/6.1810/2026/labs/ | 9 个 lab |
| 工具说明 | https://pdos.csail.mit.edu/6.1810/2026/tools.html | 环境搭建 |
| xv6 页面 | https://pdos.csail.mit.edu/6.1810/2026/xv6.html | 代码 + 教材入口 |

### 视频与中文资料

| 资源 | 链接 | 说明 |
|:--|:--|:--|
| 视频（YouTube） | https://www.youtube.com/watch?v=L6YqHxYHa7A | 官方录播，每讲链接见课程页 |
| **课程视频中文稿** | https://mit-public-courses-cn-translatio.gitbook.io/mit6-s081/ | 逐讲翻译，最完整的中文资源 |

### 作业实现仓库

| 资源 | 链接 | 说明 |
|:--|:--|:--|
| PKUFlyingPig/MIT6.S081-2020fall | https://github.com/PKUFlyingPig/MIT6.S081-2020fall | 全部 lab 实现 |

### xv6 book · 教材

> *xv6: a simple, Unix-like teaching operating system*
> Russ Cox · Frans Kaashoek · Robert Morris · 13 章

#### 链接

| 资源 | 链接 | 说明 |
|:--|:--|:--|
| **在线阅读（RISC-V）** | https://mit-pdos.github.io/xv6-riscv-book/ | 官方 HTML 版，最新 |
| **PDF 下载（rev4）** | https://pdos.csail.mit.edu/6.1810/2024/xv6/book-riscv-rev4.pdf | 最新可下载版本，2024 版 |
| 教材源码 | https://github.com/mit-pdos/xv6-riscv-book | LaTeX 源 |
| **xv6 内核源码** | https://github.com/mit-pdos/xv6-riscv | RISC-V 版教学内核 |
| 课程视频中文稿 | https://mit-public-courses-cn-translatio.gitbook.io/mit6-s081/ | 与 RISC-V 版配套 |

**本地 Markdown 全文**

| 章 | 标题 |
|:--:|:--|
| 01 | [[knowledge/course/xv6-book/01-操作系统接口\|操作系统接口]] |
| 02 | [[knowledge/course/xv6-book/02-操作系统组织\|操作系统组织]] |
| 03 | [[knowledge/course/xv6-book/03-页表\|页表]] |
| 04 | [[knowledge/course/xv6-book/04-Trap与系统调用\|Trap 与系统调用]] |
| 05 | [[knowledge/course/xv6-book/05-缺页\|缺页]] |
| 06 | [[knowledge/course/xv6-book/06-中断与设备驱动\|中断与设备驱动]] |
| 07 | [[knowledge/course/xv6-book/07-锁\|锁]] |
| 08 | [[knowledge/course/xv6-book/08-调度\|调度]] |
| 09 | [[knowledge/course/xv6-book/09-睡眠与唤醒\|睡眠与唤醒]] |
| 10 | [[knowledge/course/xv6-book/10-文件系统\|文件系统]] |
| 11 | [[knowledge/course/xv6-book/11-日志\|日志]] |
| 12 | [[knowledge/course/xv6-book/12-并发再探\|并发再探]] |
| 13 | [[knowledge/course/xv6-book/13-总结\|总结]] |

#### 全书 13 章

```
 1  Operating system interfaces      操作系统接口
 2  Operating system organization    操作系统组织
 3  Page tables                      页表
 4  Traps and system calls           Trap 与系统调用
 5  Page faults                      缺页
 6  Interrupts and device drivers    中断与设备驱动
 7  Locking                          锁
 8  Scheduling                       调度
 9  Sleep and Wakeup                 睡眠与唤醒
10  File system                      文件系统
11  Logging                          日志
12  Concurrency revisited            并发再探
13  Summary                          总结
```
