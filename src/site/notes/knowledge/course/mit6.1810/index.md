---
{"dg-publish":true,"permalink":"/knowledge/course/mit6.1810/index/","title":"MIT 6.1810 目录导航","tags":["导航","索引","MIT6.1810","操作系统","xv6"],"dg-note-properties":{"title":"MIT 6.1810 目录导航","tags":["导航","索引","MIT6.1810","操作系统","xv6"]}}
---


# 📚 MIT 6.1810 · Operating System Engineering

> MIT PDOS 实验室 · Fall 2026
> 23 讲 + 9 个实验 · 难度 🌟🌟🌟🌟🌟 · 预计 150 小时
> 先修：体系结构 + 扎实 C 功底 + RISC-V 汇编

返回 [[knowledge/course/index\|📚 课程资源]] · [[首页\|🏠 首页]]

---

## 讲次索引

| 讲 | 主题 | 讲义 |
|:--:|:--|:--|
| 1 | Introduction and examples | [[knowledge/course/mit6.1810/01-操作系统概览\|01-操作系统概览]] |
| 2 | C in xv6 and examples | [[knowledge/course/mit6.1810/02-用C编程xv6\|02-用C编程xv6]] |
| 3 | OS design | [[knowledge/course/mit6.1810/03-操作系统设计\|03-操作系统设计]] |
| 4 | Page tables | [[knowledge/course/mit6.1810/04-虚拟内存与页表\|04-虚拟内存与页表]] |
| 5 | Locking | [[knowledge/course/mit6.1810/05-锁\|05-锁]] |
| 6 | System call entry/exit | [[knowledge/course/mit6.1810/06-系统调用入口与出口\|06-系统调用入口与出口]] |
| 7 | System call interposition | [[knowledge/course/mit6.1810/07-系统调用拦截\|07-系统调用拦截]] |
| 8 | Page faults | [[knowledge/course/mit6.1810/08-缺页\|08-缺页]] |
| 9 | Transparent super pages | [[knowledge/course/mit6.1810/09-换页与超级页\|09-换页与超级页]] |
| 10 | Virtual memory for applications | [[knowledge/course/mit6.1810/10-用户态虚拟内存\|10-用户态虚拟内存]] |
| 11 | Device drivers | [[knowledge/course/mit6.1810/11-设备驱动与中断\|11-设备驱动与中断]] |
| 12 | OS Organization | [[knowledge/course/mit6.1810/12-操作系统组织与微内核\|12-操作系统组织与微内核]] |
| 13 | Thread switching | [[knowledge/course/mit6.1810/13-线程切换\|13-线程切换]] |
| 14 | Coordination | [[knowledge/course/mit6.1810/14-协调（sleep与wakeup）\|14-协调（sleep与wakeup）]] |
| 15 | Networking | [[knowledge/course/mit6.1810/15-网络\|15-网络]] |
| 16 | OS network latency | [[knowledge/course/mit6.1810/16-操作系统网络延迟\|16-操作系统网络延迟]] |
| 17 | File systems | [[knowledge/course/mit6.1810/17-文件系统\|17-文件系统]] |
| 18 | Crash recovery | [[knowledge/course/mit6.1810/18-崩溃恢复与日志\|18-崩溃恢复与日志]] |
| 19 | Verification | ⚠️ 官网无 .txt 讲义 |
| 20 | Multi-Core scalability and RCU | [[knowledge/course/mit6.1810/20-多核可扩展性与RCU\|20-多核可扩展性与RCU]] |
| 21 | Containers and Virtual Machines | [[knowledge/course/mit6.1810/21-容器与虚拟机\|21-容器与虚拟机]] |
| 22 | Kernel extensibility | [[knowledge/course/mit6.1810/22-内核可扩展性BPF\|22-内核可扩展性BPF]] |
| 23 | Meltdown | [[knowledge/course/mit6.1810/23-Meltdown\|23-Meltdown]] |

**配套论文 FAQ**

| # | 内容 |
|:--|:--|
| A1 | [[knowledge/course/mit6.1810/A1-微内核与L4常见问题\|A1-微内核与L4常见问题]] |
| A2 | [[knowledge/course/mit6.1810/A2-用户态虚拟内存论文常见问题\|A2-用户态虚拟内存论文常见问题]] |
| A3 | [[knowledge/course/mit6.1810/A3-Meltdown论文常见问题\|A3-Meltdown论文常见问题]] |

---

## 实验清单

| # | Lab | 名称 |
|:--:|:--|:--|
| 1 | util | Unix utilities |
| 2 | syscall | System calls |
| 3 | pgtbl | Page tables |
| 4 | traps | Traps |
| 5 | cow | Copy-on-write fork |
| 6 | lock | Parallelism / locking |
| 7 | net | Network driver |
| 8 | fs | File system |
| 9 | mmap | Mmap |

---

## 考试

| 考试 | 时间 | 范围 |
|:--|:--|:--|
| 期中 | 10/28 | Lec 1~12，lab 到 cow |
| 期末 | 12/16 | Lec 13~23，lab net 到 mmap |

开卷、开放笔记、开放笔记本，**关闭网络**。

---

## 官方资源

| 资源 | 链接 |
|:--|:--|
| 课程主页 | https://pdos.csail.mit.edu/6.1810/2026/ |
| 日程表 | https://pdos.csail.mit.edu/6.1810/2026/schedule.html |
| 课程概览 | https://pdos.csail.mit.edu/6.1810/2026/overview.html |
| 实验总览 | https://pdos.csail.mit.edu/6.1810/2026/labs/ |
| 工具说明 | https://pdos.csail.mit.edu/6.1810/2026/tools.html |
| 讲义目录 | https://pdos.csail.mit.edu/6.1810/2026/lec/ |

## 视频与中文资料

| 资源 | 链接 |
|:--|:--|
| 视频（YouTube） | https://www.youtube.com/watch?v=L6YqHxYHa7A |
| 课程视频中文稿 | https://mit-public-courses-cn-translatio.gitbook.io/mit6-s081/ |

## 作业实现

| 资源 | 链接 |
|:--|:--|
| PKUFlyingPig/MIT6.S081-2020fall | https://github.com/PKUFlyingPig/MIT6.S081-2020fall |

---

## xv6 book · 教材

> *xv6: a simple, Unix-like teaching operating system*
> Russ Cox · Frans Kaashoek · Robert Morris · 13 章

### 链接

| 资源 | 链接 |
|:--|:--|
| **在线阅读（RISC-V）** | https://mit-pdos.github.io/xv6-riscv-book/ |
| 教材源码 | https://github.com/mit-pdos/xv6-riscv-book |
| **xv6 内核源码** | https://github.com/mit-pdos/xv6-riscv |
| 课程视频中文稿 | https://mit-public-courses-cn-translatio.gitbook.io/mit6-s081/ |

### 全书 13 章

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

---

## 说明

讲义正文来自官网 `lec/*.txt`（教授讲课原始提纲），按讲次转换为 Markdown 保存，保留原始缩进层次。
第 19 讲官网未提供 `.txt`，需参考课程视频或其他材料。
