---
{"dg-publish":true,"permalink":"/knowledge/course/mit6.s081/11-Crash-Recovery/","title":"6.S081 11 · 崩溃恢复","tags":["MIT6.1810","xv6","操作系统","课程笔记"],"dg-note-properties":{"title":"6.S081 11 · 崩溃恢复","tags":["MIT6.1810","xv6","操作系统","课程笔记"],"date":"2026-11-18","source":"https://pdos.csail.mit.edu/6.828/2020/schedule.html"}}
---


# 6.S081 · 11 崩溃恢复

> MIT 6.S081 / 6.1810 · Operating System Engineering
> 教材：xv6: a simple, Unix-like teaching operating system（RISC-V 版）
> 本节基于 xv6-riscv 源码

返回 [[knowledge/course/mit6.s081/index\|📚 6.S081 目录]] · [[knowledge/course/index\|📚 课程资源]] · [[首页\|🏠 首页]]

---

## File system crash 概述

我们关心的 crash 包括了

* 在文件系统操作过程中的电力故障

* 在文件系统操作过程中的内核 panic。包括 Xv6 在内的大部分内核都会 panic，panic 可能是由内核 bug 引起，它会突然导致你的系统故障，但是你肯定期望能够在重启之后还能使用文件系统。

---

> 注：本节原始笔记仅记录了开头部分，尚未展开。
