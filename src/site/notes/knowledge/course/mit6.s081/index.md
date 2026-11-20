---
{"dg-publish":true,"permalink":"/knowledge/course/mit6.s081/index/","title":"6.S081 目录导航","tags":["导航","索引","MIT6.1810","xv6","操作系统"],"dg-note-properties":{"title":"6.S081 目录导航","tags":["导航","索引","MIT6.1810","xv6","操作系统"]}}
---


# 📚 6.S081 · 目录导航

> MIT **6.S081 / 6.1810** · Operating System Engineering
> 教材：xv6: a simple, Unix-like teaching operating system（RISC-V 版）

课程概况、资源与学习路径见 [[knowledge/course/mit6.s081/00-介绍\|00-介绍]]。

返回 [[knowledge/course/index\|📚 课程资源]] · [[首页\|🏠 首页]]

---

## 课程笔记

| 讲 | 主题 | 笔记 |
|:--:|:--|:--|
| **00** | 课程介绍与资源 | [[knowledge/course/mit6.s081/00-介绍\|00-介绍]] |
| 2 | OS Organization and System Calls | [[knowledge/course/mit6.s081/02-OS-Organization-System-Calls\|02-OS-Organization-System-Calls]] |
| 3 | Page Tables | [[knowledge/course/mit6.s081/03-Page-Tables\|03-Page-Tables]] |
| 4 | Traps | [[knowledge/course/mit6.s081/04-Traps\|04-Traps]] |
| 5 | Page Faults | [[knowledge/course/mit6.s081/05-Page-Faults\|05-Page-Faults]] |
| 6 | Interrupts | [[knowledge/course/mit6.s081/06-Interrupts\|06-Interrupts]] |
| 7 | Locks | [[knowledge/course/mit6.s081/07-Locks\|07-Locks]] |
| 8 | Threads | [[knowledge/course/mit6.s081/08-Threads\|08-Threads]] |
| 9 | Sleep and Wakeup | [[knowledge/course/mit6.s081/09-Sleep-Wakeup\|09-Sleep-Wakeup]] |
| 10 | File System | [[knowledge/course/mit6.s081/10-File-System\|10-File-System]] |
| 11 | Crash Recovery | [[knowledge/course/mit6.s081/11-Crash-Recovery\|11-Crash-Recovery]] |

**覆盖：L2 – L11 主干**（L1 导论与 L12 之后的专题未收录）

> L2 是 6.S081 的第一节正课；L1 为课程导论，无对应代码讲解。

---

## 工具与手册

| 内容 | 文档 |
|:--|:--|
| 环境配置（Ubuntu + 工具链 + QEMU） | [[knowledge/course/mit6.s081/18-环境配置\|18-环境配置]] |
| GDB 调试 | [[knowledge/course/mit6.s081/19-GDB调试\|19-GDB调试]] |
| xv6 手册精读（一/二/三/四/八章合订） | [[knowledge/course/mit6.s081/20-xv6-Manual\|20-xv6-Manual]] |

---

## 实验

8 个 Lab 的笔记已移到同级目录：[[knowledge/course/xv6-labs/index\|xv6-labs 目录]]

| Lab | 主题 |
|:--:|:--|
| 1 | Xv6 and Unix utilities |
| 2 | System Calls |
| 3 | Page Tables |
| 4 | Traps |
| 5 | Xv6 lazy page allocation |
| 6 | Copy-on-write |
| 7 | Multithreading |
| 8 | Locks |

---

## 参考

- 课程主页（2020 版，与本笔记对应）：https://pdos.csail.mit.edu/6.828/2020/schedule.html
- 课程主页（最新 6.1810）：https://pdos.csail.mit.edu/6.1810/
- xv6 内核源码：https://github.com/mit-pdos/xv6-riscv
- 教材正文中文版：[[knowledge/course/xv6-book/index\|xv6 book 目录]]