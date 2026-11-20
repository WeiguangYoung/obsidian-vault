---
{"dg-publish":true,"permalink":"/knowledge/course/cs61c-labs/index/","title":"CS61C FA26 · Labs 总览","tags":["CS61C","lab","实验","RISC-V","Venus","Logisim","导航"],"dg-note-properties":{"title":"CS61C FA26 · Labs 总览","tags":["CS61C","lab","实验","RISC-V","Venus","Logisim","导航"],"date":"2026-11-14","source":"https://cs61c.org/fa26/labs/"}}
---


# 🧪 CS61C FA26 · Labs

> UC Berkeley · Great Ideas in Computer Architecture · Fall 2026
> 代码仓库：`github.com/61c-teach/fa26-lab-starter`
> 本地路径：`/home/yang/.openclaw/workspace/fa26-lab-starter/`

返回 [[knowledge/course/cs61c/index\|📚 CS61C 目录]] · [[首页\|🏠 首页]]

---

## 实验清单

| Lab | 主题 | 工具 | 笔记 |
|:--:|:--|:--|:--|
| **00** | 环境配置与服务注册 | — | [[knowledge/course/cs61c-labs/Lab00\|lab00]] |
| **01** | C 基础（指针/数组/字符串/结构体） | C | [[knowledge/course/cs61c-labs/Lab01\|lab01]] |
| **02** | C 进阶与调试（GDB/Valgrind/vector） | C · GDB | [[knowledge/course/cs61c-labs/Lab02\|lab02]] |
| **03** | **RISC-V & Venus** | RISC-V 汇编 · Venus | [[knowledge/course/cs61c-labs/Lab03\|lab03]] |
| **04** | RISC-V 函数与调用约定 | RISC-V 汇编 | [[knowledge/course/cs61c-labs/Lab04\|lab04]] |
| **05** | Logisim 组合逻辑 | Logisim | [[knowledge/course/cs61c-labs/Lab05\|lab05]] |
| **06** | CPU 搭建与流水线 | Logisim | [[knowledge/course/cs61c-labs/Lab06\|lab06]] |
| **07** | 并行编程：SIMD 与 OpenMP | C · AVX · OpenMP | [[knowledge/course/cs61c-labs/Lab07\|lab07]] |
| **08** | *OPTIONAL* 61kaChow（Project 4） | — | — |

> Lab08 为可选实验，仅一段说明，无独立笔记。

## 项目（Projects）

三个 Project 的介绍与规划见 **[[knowledge/course/cs61c-labs/Projects\|🛠️ Projects 总览]]**（snek · CS61Classify · CS61CPU）。

---

---

## 环境准备

### 下载工具

```bash
cd /home/yang/.openclaw/workspace/fa26-lab-starter

# 全部（Venus + Logisim）
bash tools/download_tools.sh

# 只装 Venus
bash tools/download_tools.sh venus

# 只装 Logisim
bash tools/download_tools.sh logisim
```

> ⚠️ 脚本从 `inst.eecs.berkeley.edu` 下载，源里写死 `SEMESTER=sp26`。
> **网络不通时可改成本地镜像或手动下载。**

### Venus 常用命令

```bash
# 运行单个汇编文件
java -jar tools/venus.jar lab03/ex1_hello.s

# 开启 memcheck（检测内存错误）
java -jar tools/venus.jar lab03/ex3_memcheck.s -m

# 挂载目录到 Web 前端（双屏编辑）
java -jar tools/venus.jar . -dm

# 然后在 https://venus.cs61c.org 终端里：
mount local labs
```

### ecall 速查

| a0 | 功能 | 参数 |
|---|---|---|
| **1** | 打印整数 | a1 = 值 |
| **4** | 打印字符串 | a1 = 地址 |
| **10** | 退出程序 | — |
| **11** | 打印字符 | a1 = 字符 |
| **17** | 退出（带返回值） | a1 = 返回值 |

> Venus 扩展：`a0 = 0x3CC`（972）+ `a6` 区分 `1`=malloc / `4`=free。

---

## 仓库结构

```
fa26-lab-starter/
├── lab00/            环境配置
├── lab01/            C 基础
├── lab02/            C 进阶
├── lab03/            RISC-V & Venus      ← 已有笔记
├── lab04/            RISC-V 函数
├── lab05/            Logisim
├── lab06/            CPU 搭建
└── tools/
    └── download_tools.sh
```

---

## 相关课程笔记

| Lab | 对应讲次 |
|:--|:--|
| **Lab01–02** | [[knowledge/course/cs61c/03-Introduction-to-C\|L03 引言]] · [[knowledge/course/cs61c/04-Pointers-Arrays-Strings\|L04 指针]] · [[knowledge/course/cs61c/05-Memory-Management\|L05 内存]] |
| **Lab03** | [[knowledge/course/cs61c/11-RISC-V-Control-Flow\|L11 控制流]] · [[knowledge/course/cs61c/12-RISC-V-Procedures\|L12 函数调用]] |
| **Lab04** | [[knowledge/course/cs61c/12-RISC-V-Procedures\|L12 函数调用]] · [[knowledge/course/cs61c/15-CALL\|L15 CALL]] |
| **Lab05** | [[knowledge/course/cs61c/17-SDS-II-Combinational-Logic\|L17 组合逻辑]] · [[knowledge/course/cs61c/18-SDS-III-State\|L18 状态]] |
| **Lab06** | [[knowledge/course/cs61c/20-Datapath-I\|L20 Datapath]] · [[knowledge/course/cs61c/22-Control\|L22 控制]] |
| **Lab07** | [[knowledge/course/cs61c/31-Parallelism-II-DLP-SIMD\|L31 SIMD]] · [[knowledge/course/cs61c/32-Parallelism-III-TLP-OpenMP\|L32 OpenMP]] |
