---
{"dg-publish":true,"permalink":"/knowledge/language/c/index/","title":"C 语言 导航","tags":["导航","索引","C语言"],"dg-note-properties":{"title":"C 语言 导航","tags":["导航","索引","C语言"]}}
---


# 🔧 C 语言 · 目录导航

> 从语法到机器：C 语言在真实工程里的落地

返回 [[首页\|🏠 首页]]

---

## 文档索引

| # | 文档 | 说明 | 行数 |
|:--|:-----|:-----|:--|
| 01 | [[knowledge/language/c/01-C语言进阶\|01-C语言进阶]] | **总纲（15 章）**：类型与内存布局 / 内存段与启动代码 / 指针与函数指针 / `const`·`static`·`extern` / `volatile` / 整数提升与 UB / 位运算与寄存器 / 静态分配与环形缓冲 / 状态机 / 协议解析器 / CRC16+CRC32 / 工具链 / 可测性 / 验收标准 | 1,633 |
| 02 | [[knowledge/language/c/02-指针与数组\|02-指针与数组]] | 数组名是什么 / `sizeof` 的三种命运 / 二维数组真实布局 / 数组指针 vs 指针数组 / `const` 三种组合 / 传参为什么必须带长度 | 567 |
| 03 | [[knowledge/language/c/03-结构体位域与内存对齐\|03-结构体位域与内存对齐]] | 对齐两条规则 / 手算 `sizeof` / 成员顺序定大小 / 嵌套与联合体 / 位域四个致命问题 / `#pragma pack` / 柔性数组 / `container_of` | 703 |
| 04 | [[knowledge/language/c/04-位操作\|04-位操作]] | 六个运算符 / `BIT()` 与读写改 / 多位字段 / 寄存器操作（W1C 坑）/ 协议位拼接 / 位图 / 经典技巧 / 有符号数移位 | 645 |
| 05 | [[knowledge/language/c/05-宏与条件编译\|05-宏与条件编译]] | 预处理是文本替换 / 括号与副作用 / `do{}while(0)` / `#` 与 `##` / 条件编译 / 头文件保护 / X-Macro | 910 |

---

## 📖 与《C Primer Plus》的对应

| Primer Plus 章 | 主题 | 对应本文档 |
|:--|:--|:--|
| 第 10 章 | 数组和指针 | [[knowledge/language/c/02-指针与数组\|02-指针与数组]] |
| 第 14 章 | 结构和其他数据形式 | [[knowledge/language/c/03-结构体位域与内存对齐\|03-结构体位域与内存对齐]] |
| 第 15 章 | 位操作 | [[knowledge/language/c/04-位操作\|04-位操作]] |
| 第 16 章 | C 预处理器和 C 库 | [[knowledge/language/c/05-宏与条件编译\|05-宏与条件编译]] |

> 这四章是 Primer Plus 里**语法密度最高、也最容易被「看过就忘」**的部分。
> 本套文档的写法是：每个结论都配上 `sizeof` / `offsetof` / `gcc -E` 的**实测结果**，以及真实仓库里的代码证据 —— 而不是复述书上的语法。

### 与总纲的分工

| 想学 | 读哪 |
|:--|:--|
| 整体路线、验收标准、达标题目 | [[knowledge/language/c/01-C语言进阶\|01-C语言进阶]] |
| 数组/指针的细节与坑 | [[knowledge/language/c/02-指针与数组\|02-指针与数组]] |
| 结构体大小算不准、协议结构体怎么定义 | [[knowledge/language/c/03-结构体位域与内存对齐\|03-结构体位域与内存对齐]] |
| 寄存器怎么操作、协议字段怎么拆 | [[knowledge/language/c/04-位操作\|04-位操作]] |
| 宏写不对、条件编译开关乱、头文件冲突 | [[knowledge/language/c/05-宏与条件编译\|05-宏与条件编译]] |

---

## 与相关文档的关系

| 想深入 | 去看 |
|:--|:--|
| 编译 / 链接 / 装载的完整链路 | [[knowledge/embedded/02-编译链接装载与库\|编译、链接、装载与库]] |
| CMake 怎么组织这些代码 | [[knowledge/devops/practices/05-构建系统实战-CMake\|CMake 工程实战]] |
| 链接脚本与启动代码 | [[knowledge/embedded/01-裸机编程指南\|裸机编程指南]] |
| CAN + UDS 协议实现 | [[knowledge/embedded/04-ZLG-CAN库与UDS刷写详解\|ZLG CAN 库与 UDS 刷写详解]] |

---

## 达标目标

不靠教程写出 **300–500 行**可编译、可测试的系统编程风格 C，并通过 `-Wall -Wextra -Werror`。
题目与自检清单见 [[knowledge/language/c/01-C语言进阶#十五学习计划与验收标准\|第十五章]]。