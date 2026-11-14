---
{"dg-publish":true,"permalink":"/knowledge/course/xv6-book/index/","title":"xv6 book 目录","tags":["导航","索引","MIT6.1810","xv6","操作系统"],"dg-note-properties":{"title":"xv6 book 目录","tags":["导航","索引","MIT6.1810","xv6","操作系统"]}}
---


# 📖 xv6 book

> *xv6: a simple, Unix-like teaching operating system*
> Russ Cox · Frans Kaashoek · Robert Morris
> 在线：[英文原版（RISC-V）](https://mit-pdos.github.io/xv6-riscv-book/) · 源码：`github.com/mit-pdos/xv6-riscv-book`
> 中文参考：[课程视频中文稿（RISC-V 版配套）](https://mit-public-courses-cn-translatio.gitbook.io/mit6-s081/) · [旧版中文文档（x86，仅参考）](https://th0ar.gitbooks.io/xv6-chinese/)

返回 [[knowledge/course/mit6.1810/index\|📚 MIT 6.1810]] · [[首页\|🏠 首页]]

---

## 全书章节

| 章 | 标题 | 文档 | 主题 |
|:--:|:--|:--|:--|
| 1 | Operating system interfaces | [[knowledge/course/xv6-book/01-操作系统接口\|01-操作系统接口]] | 进程与内存、文件描述符、管道、文件系统 |
| 2 | Operating system organization | [[knowledge/course/xv6-book/02-操作系统组织\|02-操作系统组织]] | 用户/内核态、内核组织、进程概览、启动 |
| 3 | Page tables | [[knowledge/course/xv6-book/03-页表\|03-页表]] | 分页硬件、内核地址空间、物理内存分配、exec |
| 4 | Traps and system calls | [[knowledge/course/xv6-book/04-Trap与系统调用\|04-Trap与系统调用]] | RISC-V trap 机制、用户态 trap、系统调用参数 |
| 5 | Page faults | [[knowledge/course/xv6-book/05-缺页\|05-缺页]] | 惰性分配、COW fork、按需分页、mmap |
| 6 | Interrupts and device drivers | [[knowledge/course/xv6-book/06-中断与设备驱动\|06-中断与设备驱动]] | 控制台输入输出、驱动并发、定时器中断 |
| 7 | Locking | [[knowledge/course/xv6-book/07-锁\|07-锁]] | 竞态、自旋锁、死锁与锁序、内存序、睡眠锁 |
| 8 | Scheduling | [[knowledge/course/xv6-book/08-调度\|08-调度]] | 多路复用、上下文切换、调度器、mycpu/myproc |
| 9 | Sleep and Wakeup | [[knowledge/course/xv6-book/09-睡眠与唤醒\|09-睡眠与唤醒]] | sleep/wakeup 机制、管道、wait/exit/kill |
| 10 | File system | [[knowledge/course/xv6-book/10-文件系统\|10-文件系统]] | 缓冲缓存、块分配、inode、目录层、路径名 |
| 11 | Logging | [[knowledge/course/xv6-book/11-日志\|11-日志]] | 日志设计、崩溃恢复、write-ahead log |
| 12 | Concurrency revisited | [[knowledge/course/xv6-book/12-并发再探\|12-并发再探]] | 加锁模式、类锁模式、无锁、并行性 |
| 13 | Summary | [[knowledge/course/xv6-book/13-总结\|13-总结]] | 全书总结 |

---

## 阅读顺序建议

按书本原顺序即可，但有三条更实用的路径：

| 路径 | 顺序 | 适合 |
|:--|:--|:--|
| **完整精读** | 1 → 2 → 3 → 4 → 5 → 6 → 7 → 8 → 9 → 10 → 11 → 12 → 13 | 系统学 |
| **先懂机制** | 1 → 2 → 3 → 4 → 8 → 9 → 7 | 先建立进程/内存/调度的主干 |
| **主题速查** | 页表 3·5 ｜ Trap 4 ｜ 并发 7·12 ｜ 调度 8·9 ｜ 文件 10·11 | 按需查 |

---

## 与课程讲次的对应

| 书本章 | 课程讲次 |
|:--|:--|
| 1~2 | 第 1~3 讲（导论、C in xv6、OS design）|
| 3 | 第 4、8~10 讲（页表、缺页、大页、用户态 VM）|
| 4 | 第 6~7 讲（系统调用入口/出口、拦截）|
| 5 | Lab cow（写时复制）|
| 6 | 第 11 讲（设备驱动）|
| 7 | 第 5 讲（Locking）|
| 8~9 | 第 13~14 讲（线程切换、协调）|
| 10~11 | 第 17~18 讲（文件系统、崩溃恢复）|
| 12 | 第 20 讲（多核可扩展性与 RCU 前置）|
| 13 | — |

---

## 说明

正文由 `github.com/mit-pdos/xv6-riscv-book` 的 LaTeX 源（`*.tex`）转换而来，章节结构和代码示例保留。
