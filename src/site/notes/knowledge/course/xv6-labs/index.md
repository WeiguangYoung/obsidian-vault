---
{"dg-publish":true,"permalink":"/knowledge/course/xv6-labs/index/","title":"xv6-labs 目录导航","tags":["导航","索引","MIT6.1810","xv6","LAB","实验"],"dg-note-properties":{"title":"xv6-labs 目录导航","tags":["导航","索引","MIT6.1810","xv6","LAB","实验"]}}
---


# 🧪 xv6-labs · 实验目录

返回 [[knowledge/course/mit6.s081/index\|📚 6.S081 课程]] · [[knowledge/course/index\|📚 课程资源]] · [[首页\|🏠 首页]]

---

## 实验列表

| Lab | 主题 | 笔记 |
|:--:|:--|:--|
| 1 | Xv6 and Unix Utilities | [[knowledge/course/xv6-labs/Lab01-Xv6-and-Unix-Utilities\|Lab01-Xv6-and-Unix-Utilities]] |
| 2 | System Calls | [[knowledge/course/xv6-labs/Lab02-System-Calls\|Lab02-System-Calls]] |
| 3 | Page Tables | [[knowledge/course/xv6-labs/Lab03-Page-Tables\|Lab03-Page-Tables]] |
| 4 | Traps | [[knowledge/course/xv6-labs/Lab04-Traps\|Lab04-Traps]] |
| 5 | Lazy Page Allocation | [[knowledge/course/xv6-labs/Lab05-Lazy-Page-Allocation\|Lab05-Lazy-Page-Allocation]] |
| 6 | Copy-on-Write | [[knowledge/course/xv6-labs/Lab06-Copy-on-Write\|Lab06-Copy-on-Write]] |
| 7 | Multithreading | [[knowledge/course/xv6-labs/Lab07-Multithreading\|Lab07-Multithreading]] |
| 8 | Locks | [[knowledge/course/xv6-labs/Lab08-Locks\|Lab08-Locks]] |

---

## 实验与课程的对应

| Lab | 前置课程笔记 |
|:--:|:--|
| 1 | [[knowledge/course/mit6.s081/02-OS-Syscalls\|02-OS-Syscalls]] |
| 2 | [[knowledge/course/mit6.s081/04-Traps\|04-Traps]] |
| 3 | [[knowledge/course/mit6.s081/03-Page-Tables\|03-Page-Tables]] |
| 4 | [[knowledge/course/mit6.s081/04-Traps\|04-Traps]] |
| 5 | [[knowledge/course/mit6.s081/05-Page-Faults\|05-Page-Faults]] |
| 6 | [[knowledge/course/mit6.s081/05-Page-Faults\|05-Page-Faults]] |
| 7 | [[knowledge/course/mit6.s081/08-Threads\|08-Threads]] |
| 8 | [[knowledge/course/mit6.s081/07-Locks\|07-Locks]] |

---

## 说明

每篇 Lab 文档统一分三部分：

| 部分 | 内容 |
| --- | --- |
| 一、实验题目 | 官方题目中文翻译，含任务要求、提示、评分命令 |
| 二、我的实现 | 实际动手的代码与踩坑记录 |
| 三、参考答案与讲解 | 另一份实现的思路说明与完整代码 |

- 题目与参考答案译自 `duguosheng/6.S081-All-in-one`，该仓库的配图未随文收录。
- 本目录不存放任何图片文件；原笔记中的截图保留为原始外链（共 10 处）。
- 每个 Lab 通常在课程对应讲次后布置，建议先读完同名课程笔记再做。
- 实验环境搭建见 [[knowledge/course/mit6.s081/18-Environment-Setup\|环境配置]]，调试方法见 [[knowledge/course/mit6.s081/19-GDB-Debug\|GDB 调试]]。

---

## 官方资源

- Lab 说明：https://pdos.csail.mit.edu/6.828/2020/labs/
- Lab 骨架（官方，`git://` 协议，无 https 镜像）：
  ```bash
  git clone git://g.csail.mit.edu/xv6-labs-2020
  cd xv6-labs-2020 && git checkout util
  ```
- xv6 源码：https://github.com/mit-pdos/xv6-riscv