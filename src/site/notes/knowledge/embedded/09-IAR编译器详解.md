---
{"dg-publish":true,"permalink":"/knowledge/embedded/09-IAR编译器详解/","title":"IAR 编译器详解：从 GCC 视角看","tags":["IAR","编译器","工具链","嵌入式","ICCARM","ILINK","ICF","C-SPY","交叉编译"],"dg-note-properties":{"title":"IAR 编译器详解：从 GCC 视角看","date":"2026-11-27","tags":["IAR","编译器","工具链","嵌入式","ICCARM","ILINK","ICF","C-SPY","交叉编译"]}}
---


# 🔨 IAR 编译器详解：从 GCC 视角看

> IAR Embedded Workbench 是嵌入式领域**商业工具链的事实标准之一**，和 Keil MDK、GCC 三分天下。
> 这篇不写「怎么点菜单」，而是回答四个问题：
> **它和 GCC 到底差在哪？为什么贵还有人用？从 GCC 工程迁过去要改什么？那些 `__no_init`、`.icf` 是什么鬼？**
>
> 全文用**对照表**组织 —— 每个 IAR 概念都配一个「GCC 里对应什么」，因为对 GCC 用户来说，映射比定义更好记。

返回 [[首页\|🏠 首页]] · 相关：[[knowledge/embedded/02-编译链接装载与库\|编译、链接、装载与库]] · [[knowledge/embedded/03-嵌入式C语言的自我修养\|嵌入式的四层硬功夫]] · [[knowledge/embedded/01-裸机编程指南\|裸机编程指南]]

---

## 一、它是什么

### 1.1 一句话定位

**IAR Embedded Workbench = 编译器 + 链接器 + 调试器 + IDE 的一体化商业工具链。**

不是「一个编译器」，是一整套。名字里的 Workbench 就是工作台的意思。

| 对比项   | IAR EW          | Keil MDK            | GCC（arm-none-eabi）   |
| :---- | :-------------- | :------------------ | :------------------- |
| 性质    | 商业，按 license 收费 | 商业（被 ARM 收购）        | 开源免费                 |
| 编译器   | ICCARM（自有）      | ARMCC / AC6（ARM 自有） | GCC                  |
| 链接脚本  | `.icf`          | `.sct`（scatter）     | `.ld`                |
| IDE   | 自带              | 自带（µVision）         | 无（配 VS Code/Eclipse） |
| 代码密度  | **通常最优**        | 优                   | 良                    |
| 生态/资料 | 少但精             | 中等                  | 海量                   |
| 授权成本  | 高               | 中                   | 0                    |

### 1.2 为什么商业工具链还活着

三个真实原因，和「性能」关系没那么大：

**① 功能安全认证**

IAR 的编译器带 **TÜV SÜD 认证**，可以直接用于 ISO 26262（汽车）、IEC 61508（工业）、EN 50128（铁路）等功能安全项目。

这一点在车载领域是**硬门槛**：认证机构要求「工具链本身经过鉴定」（tool qualification）。用 GCC 不是不行，但你需要自己做工具鉴定（写大量测试用例证明编译器没 bug），成本远超买 license。

> 你在做的车载 Bootloader，如果将来要过 ISO 26262，这条就是绕不过去的。

**② 代码密度**

IAR 的 ICCARM 在 Cortex-M 上的代码密度长期优于 GCC，尤其是开了 size 优化后。小 Flash 的芯片（比如 32 KB 的 STM8、64 KB 的 F0）差这几个 KB 就是能不能装下的问题。

**③ 调试器 C-SPY**

实时 trace、代码覆盖率、函数耗时分析、RTOS 感知，这些是集成好的，不用自己拼 OpenOCD + GDB + 第三方插件。

### 1.3 支持哪些架构

官方口径是 20 多种，常见的：

**Arm**（Cortex-M / Cortex-R，最主流）、**RISC-V**、**8051**、**MSP430**、**AVR / AVR32**、**STM8**、**Renesas** 全家桶（RX / RL78 / RH850 / V850 / 78K / M16C-R8C）

⚠️ 注意：**每个架构是独立产品、独立授权**。买了 Arm 版不能编 RISC-V。这点和 GCC 不同（GCC 换个 prefix 就行）。

---

## 二、工具链组成

一套 IAR for Arm 装完，`bin/` 下会有一堆 exe。认识这几个就够用了：

| 工具 | 作用 | GCC 对应 |
|:--|:--|:--|
| `iccarm.exe` | C/C++ 编译器 | `arm-none-eabi-gcc -c` |
| `iasmarm.exe` | 汇编器（ARM 汇编） | `arm-none-eabi-as` |
| `ilinkarm.exe` | 链接器 | `arm-none-eabi-ld` |
| `iarchive.exe` | 静态库打包 | `arm-none-eabi-ar` |
| `ielftool.exe` | ELF 转 hex/bin、加校验和 | `objcopy` + `srec_cat` |
| `ielfdumparm.exe` | 反汇编、看段信息 | `objdump` / `readelf` |
| `CSpyBat.exe` | 命令行调试/下载 | `openocd` + `gdb` |

**关键差异**：GCC 的这套工具是**松散的 Unix 工具**，靠 Makefile 串起来；IAR 的这套是**紧耦合的**，由 IDE 统一调度，中间产物格式也是私有的（`.o` 虽然也是 ELF，但有私有扩展段）。

### 2.1 编译流程对照

```
GCC:                                IAR:
foo.c                               foo.c
 ├─ cpp (预处理)                     ├─ iccarm --preprocess
 ├─ cc1 (编译)                       ├─ iccarm (编译+汇编一体)
 └─ as  (汇编)                       │
    ↓                               ↓
 foo.o  (ELF)                       foo.o  (ELF + 私有段)
    ↓ ld                               ↓ ilinkarm
 foo.elf                            foo.out
    ↓ objcopy                          ↓ ielftool
 foo.hex / foo.bin                  foo.hex / foo.bin
```

注意 IAR 的 `iccarm` **把编译和汇编合并成一步**，没有独立的 `.s` 中间文件（除非你显式要）。

---

## 三、工程文件结构

IAR 的工程是一组 XML 文件，不是 Makefile。

| 文件 | 作用 | GCC 对应 |
|:--|:--|:--|
| `xxx.eww` | 工作区（workspace），含多个工程 | 无（一个 Makefile 一个目标） |
| `xxx.ewp` | 工程文件，**XML 格式**，含所有编译选项 | `Makefile` + `CMakeLists.txt` |
| `xxx.ewd` | 调试配置（断点、寄存器视图） | `.gdbinit` |
| `xxx.ewt` | 测试配置 | — |
| `xxx.icf` | **链接配置文件**（链接脚本） | `.ld` |
| `xxx.dep` | 依赖关系（自动生成） | `.d` 文件 |
| `settings/` | 目录，存 IDE 的个人设置 | — |

### 3.1 `.ewp` 是 XML，可以手改

这点很重要 —— 意味着**可以做版本控制、可以脚本生成、可以 diff**：

```xml
<configuration>
  <name>Debug</name>
  <state>S32K144_Bootloader</state>
  <settings>
    <name>General</name>
    <state>ICCARM</state>
    <settings>
      <name>CCDefines</name>
      <state>CPU_S32K144</state>
      <state>START_FROM_FLASH</state>
    </settings>
    <settings>
      <name>CCOptimization</name>
      <state>HIGH</state>
    </settings>
  </settings>
</configuration>
```

**实践**：`.ewp` 一定要进 git。但里面的**绝对路径**（工具链安装位置、源文件路径）会随机器变化 —— 这是 IAR 工程协作的最大痛点。

> 解决方案：用 `$PROJ_DIR$` 这类变量代替绝对路径。
> ```
> $PROJ_DIR$\..\..\common\include
> $TOOLKIT_DIR$\inc\c
> ```
> 但 IDE 自动生成的路径往往是绝对的，**每次新加文件都要手动检查**。

---

## 四、编译器扩展：IAR 的私有语法

这是最容易踩坑的部分。IAR 的 C 扩展和 GCC **语法完全不同**，不能照抄。

### 4.1 核心关键字对照表

| 需求 | IAR（ICCARM） | GCC |
|:--|:--|:--|
| 变量不初始化（放 noinit 段） | `__no_init uint32_t buf[100];` | `__attribute__((section(".noinit")))` |
| 函数放 RAM 执行 | `__ramfunc void flash_write(void)` | `__attribute__((section(".ramfunc")))` |
| 强制保留（不被优化删除） | `__root const char ver[] = "1.0";` | `__attribute__((used))` |
| 弱符号 | `__weak void foo(void)` | `__attribute__((weak))` |
| 绝对地址定位 | `volatile uint32_t reg @ 0x40021000;` | `#define REG (*(volatile uint32_t*)0x40021000)` |
| 紧凑结构体 | `__packed struct {...};` | `struct __attribute__((packed)) {...};` |
| 对齐 | `#pragma data_alignment=32` | `__attribute__((aligned(32)))` |
| 中断函数 | `#pragma vector=X` + `__interrupt` | `__attribute__((interrupt))` |
| 内联 | `#pragma inline` / `__inline` | `static inline` |
| 函数不返回（不生成返回代码） | `__noreturn` 属性 / `#pragma object_attribute=__noreturn` | `__attribute__((noreturn))` |
| 段放置 | `@ "MY_SECTION"` | `__attribute__((section("my_section")))` |

### 4.2 逐个说清楚

#### `__no_init`：不初始化

```c
__no_init uint8_t g_big_buffer[8192];
```

**语义**：这块内存**不进 `.data` 也不进 `.bss`**，启动时不会被清零，内容保持上电时的随机值（或者上次运行留下的值）。

**为什么要它**：

1. **省启动时间**：8 KB 的缓冲区如果放 `.bss`，启动时要清零 8 KB，慢
2. **省 Flash**：有初值的 `.data` 要在 Flash 里存一份副本
3. **掉电保持**：复位后想保留上次的数据（配合「不初始化 RAM」的复位方式）

**GCC 对应**：
```c
uint8_t g_big_buffer[8192] __attribute__((section(".noinit")));
// 并且要在 .ld 里定义 .noinit 段且标 NOLOAD
```

⚠️ **坑**：`__no_init` 的变量**必须在第一次使用前显式初始化**，否则是随机值。这类 bug 极难查（同样的代码，冷启动正常、热复位异常）。

#### `__ramfunc`：函数放 RAM 跑

```c
__ramfunc void flash_erase_sector(uint32_t addr)
{
    /* 擦除 Flash 的代码不能从 Flash 取指 */
}
```

**原理**：擦写 Flash 时，Flash 控制器被占用，**从 Flash 取指会失败**。所以擦写函数本身必须放到 RAM 里执行。

**GCC 对应**：
```c
__attribute__((section(".ramfunc"))) void flash_erase_sector(uint32_t addr);
// 且 .ld 里要让 .ramfunc 的 LMA 在 Flash、VMA 在 RAM，并加进 .data 的拷贝范围
```

**这是 Bootloader 的必备技能** —— 你在 S32K144 Bootloader 的 `fls_app.c` 里做的 Flash 擦写，如果换成 IAR 编译，就得加 `__ramfunc`。

> 你们那份 `S32K312_PORTING_CHECKLIST.md` 里提到 `START_FUNCTION_DECLARATION_RAMSECTION` 对应 `__ramfunc`，就是这个机制。

#### `@` 绝对地址定位

```c
// IAR：直接在变量名后面写 @ 地址
volatile uint32_t g_reg __no_init @ 0x400FF000;

// 或者放到指定段
__no_init uint32_t g_shared_data @ "SHARED_RAM";
```

**GCC 完全做不到这一点**（不能在变量声明处指定绝对地址），只能用指针：
```c
#define g_reg  (*(volatile uint32_t *)0x400FF000)
```

这是 IAR 的一个实际便利：**变量和寄存器共用同一套访问语法**，不用每处都写 `#define`。

⚠️ 但绝对地址定位的变量**不能有初值**（编译器不会帮你搬），必须配合 `__no_init`。

#### `#pragma vector` + `__interrupt`：中断函数

```c
#pragma vector = TIM2_IRQHandler
__interrupt void my_timer_isr(void)
{
    /* 不需要手动清标志以外的事 */
}
```

**两个作用**：
1. `#pragma vector` 把函数**注册进向量表**（链接器自动填）
2. `__interrupt` 让函数用正确的返回指令（`bx lr` 配合 EXC_RETURN）

**GCC 对应**：函数名直接写成向量表里的名字（`TIM2_IRQHandler`），加 `__attribute__((interrupt))`。

**坑**：IAR 里如果你用 `#pragma vector` 注册，函数名**可以随便起**（不必和向量表名一致）；但如果你按 GCC 习惯直接命名，也**能工作**（弱符号覆盖机制）。两种方式混用会导致一个中断被注册两次 —— 链接时报 `Error[Lp025]` 或 `multiple definition`。

#### `__root`：防止被删

```c
__root const char g_version[] = "v1.0.0";
```

**IAR 的链接器默认会删掉「没被引用」的符号**（和 `--gc-sections` 类似）。`__root` 强制保留。

**典型场景**：
- 只被向量表引用的函数
- 只被链接脚本引用的符号
- 想让它出现在最终镜像里的版本号字符串

**GCC 对应**：`__attribute__((used))` 或链接脚本里的 `KEEP()`

### 4.3 内联汇编：格式也不同

```c
// IAR
__asm volatile ("MRS %0, PRIMASK" : "=r"(x));

// GCC
__asm volatile ("mrs %0, primask" : "=r"(x));
```

看着差不多，但**约束符支持范围不同**：IAR 的 ICCARM 对内联汇编的支持比 GCC 保守，某些 GCC 能用的写法 IAR 不行（比如 `"memory"` clobber 在 IAR 里的语义、多输出约束）。

**实践建议**：写跨编译器的代码时，**把内联汇编隔离到单独的 `.h` 里**，用宏区分：

```c
#if defined(__ICCARM__)
  #define CPU_DISABLE_IRQ()  __asm volatile ("CPSID i")
#elif defined(__GNUC__)
  #define CPU_DISABLE_IRQ()  __asm volatile ("cpsid i" ::: "memory")
#else
  #error "unsupported compiler"
#endif
```

---

## 五、链接脚本 `.icf`

IAR 的链接配置叫 **ICF**（ILINK Configuration File），语法和 GNU ld **完全不是一回事**。

### 5.1 一个最小 ICF

```
/* 定义内存区域 */
define symbol __ICFEDIT_region_ROM_start__ = 0x00000000;
define symbol __ICFEDIT_region_ROM_end__   = 0x0003FFFF;
define symbol __ICFEDIT_region_RAM_start__ = 0x20000000;
define symbol __ICFEDIT_region_RAM_end__   = 0x2000FFFF;

define region ROM_region = mem:[from __ICFEDIT_region_ROM_start__
                               to   __ICFEDIT_region_ROM_end__];
define region RAM_region = mem:[from __ICFEDIT_region_RAM_start__
                               to   __ICFEDIT_region_RAM_end__];

/* 放置段 */
place in ROM_region   { readonly };
place in RAM_region   { readwrite,
                        block CSTACK,
                        block HEAP };
```

对应 GCC 的：

```ld
MEMORY {
  ROM (rx)  : ORIGIN = 0x00000000, LENGTH = 256K
  RAM (rwx) : ORIGIN = 0x20000000, LENGTH = 64K
}
SECTIONS {
  .text   : { *(.text*) *(.rodata*) } > ROM
  .data   : { *(.data*) } > RAM AT > ROM
  .bss    : { *(.bss*) *(COMMON) } > RAM
}
```

### 5.2 语法对照表

| 概念 | ICF | GNU ld |
|:--|:--|:--|
| 定义内存区 | `define region X = mem:[from A to B];` | `MEMORY { X : ORIGIN=..., LENGTH=... }` |
| 放置段 | `place in X { readonly };` | `.text : { ... } > X` |
| 定义符号 | `define symbol S = 0x1000;` | `S = 0x1000;` |
| 初始化段（LMA/VMA 分离） | `initialize by copy { readwrite };` | `.data : {} > RAM AT > ROM` |
| 保留段 | `keep { section .mysec };` | `KEEP(*(.mysec))` |
| 栈块 | `block CSTACK, block HEAP` | 自己算 ```_estack = ORIGIN(RAM)+LENGTH(RAM)``` |

**关键洞察**：ICF 的 `readwrite` + `initialize by copy` 一行，等价于 ld 里「`.data` 的 LMA 在 Flash、VMA 在 RAM，启动时拷贝」这一整套约定。**IAR 把这件事变成了一步声明。**

### 5.3 最经典的 IAR 链接错误

**`Error[Lp011]: section placement failed`**

含义：某个段放不下了。原因和修法：

| 原因 | 现象 | 修法 |
|:--|:--|:--|
| RAM 真不够 | 加了大数据结构 | 加大 region、优化、用 `__no_init` |
| 栈/堆设太大 | `CSTACK` 配了 8 KB | 改 ICF 里的 block 大小 |
| `.icf` 地址和芯片不符 | 换芯片没改 ICF | 核对 datasheet 的 Flash/RAM 起止 |
| 段没地方放 | 自定义段没 `place` | 加 `place in ROM_region { section .mysec };` |

这个错误信息**不给具体是哪个段**（旧版本），得靠 `--log` 生成 map 文件看。

---

## 六、调试：C-SPY

IAR 的调试器叫 **C-SPY**，集成在 IDE 里。特色功能：

| 功能 | 说明 | GCC 侧等价物 |
|:--|:--|:--|
| Live Watch | 变量实时刷新（不停机） | 需 SWO/RTT + 手写 |
| 数据断点 | 某地址被读写时中断 | GDB `watch` + 硬件断点 |
| 代码覆盖率 | 哪些行被执行过 | `gcov`（嵌入式难搞） |
| 函数耗时分析 | 每个函数花了多少周期 | 需 DWT + 手写 |
| Trace | 指令级执行轨迹 | 需 ETM/MTB 硬件 + 昂贵工具 |
| RTOS 感知 | 直接看任务列表、栈水位 | 需 GDB 插件 |

**Live Watch 是最实用的**：串口打印要改代码、可能影响时序；Live Watch 直接读内存，不停机。

### 6.1 支持的调试探针

官方列出：**I-jet / I-jet Trace**（IAR 自家）、**J-Link**（SEGGER）、**PE Micro**、**ST-LINK**

⚠️ **DAP-Link（CMSIS-DAP）支持有限**。你手上那块 F407 用 DAP-Link，如果换 IAR 可能要用 J-Link 或 ST-LINK。

---

## 七、常见编译错误速查

| 错误 | 含义 | 常见原因 |
|:--|:--|:--|
| `Error[Pe020]: identifier "xxx" is undefined` | 标识符未定义 | 忘 `#include`、拼写错、条件编译没进 |
| `Error[Pe1696]: cannot open source file` | 找不到头文件 | include 路径没加、路径是绝对路径换机器失效 |
| `Warning[Pe177]: variable was declared but never referenced` | 声明未使用 | 正常警告，`__root` 可消除 |
| **`Error[Li005]: no definition for "xxx"`** | **链接时找不到符号** | **源文件没加进工程**（最常见！） |
| `Error[Lp011]: section placement failed` | 段放不下 | 见上一节 |
| `Error[Lp025]: absolute placement conflict` | 绝对定位冲突 | 两个变量 `@` 到同一地址 |
| `Fatal Error[Lc002]: could not open file` | 打不开文件 | 路径含空格/中文 |

**`Li005` 是 IAR 新手第一大坑**：GCC 的 Makefile 里源文件列表是明文的，漏了立刻能看出来；IAR 的 `.ewp` 里源文件是 XML 节点，**在 IDE 里右键添加很容易漏**。

> 排查法：`.ewp` 里搜 `<name>$PROJ_DIR$\...\foo.c</name>`，确认文件真的在列表里。

---

## 八、版本与授权

### 8.1 版本

IAR EW for Arm 目前是 **9.x** 系列（具体小版本以官网为准）。每个大版本会更新：
- 支持新的 Cortex 核（比如 Cortex-M85、Cortex-R52）
- 更新 C/C++ 标准支持（C11 / C17 / C++17 逐步跟进）
- 优化器改进

⚠️ **不同版本对 C 标准的支持程度不同**。用 `_Static_assert`、`_Generic`、`stdatomic.h` 这些 C11 特性前，先查该版本的 release notes。

### 8.2 授权类型

| 类型 | 说明 |
|:--|:--|
| **永久授权（Perpetual）** | 买断某版本，可一直用；升级要另付费 |
| **订阅（Subscription）** | 按年付费，期间可升级 |
| **网络授权** | License Server 分发，团队共用若干席位 |
| **评估版（Evaluation）** | 有代码大小限制，仅用于评估 |
| **Kickstart** | 免费，代码大小限制（历史上是 32 KB，**以官网当前说明为准**） |

⚠️ **具体价格和 Kickstart 的限制，IAR 调整过多次，务必查官网，别信网上旧帖。**

### 8.3 代码大小怎么算

评估版的限制通常按**最终镜像的代码段大小**算（`.text` + `.rodata` 之类），不是按源文件行数。加了 `__root` 的常量、字符表都会计入。

---

## 九、什么时候该用、什么时候不该用

### 9.1 该用

| 场景 | 理由 |
|:--|:--|
| **功能安全项目**（ISO 26262 / IEC 61508） | TÜV 认证，省掉工具鉴定的巨大成本 |
| Flash 极度紧张（< 64 KB） | 代码密度优势能换来实打实的空间 |
| 团队已有 IAR 工程和 license | 迁移成本高于收益 |
| 需要 C-SPY 的高级调试（Live Watch、覆盖率） | 自建 OpenOCD 方案费时 |
| 瑞萨 / STM8 / 8051 等小众架构 | 这些架构 GCC 支持很差甚至没有 |

### 9.2 不该用

| 场景 | 理由 |
|:--|:--|
| **学习嵌入式原理** | 商业 IDE 会挡住很多细节，`__ramfunc` 背后的机制被隐藏了 |
| **CI/CD 自动化** | 虽然 IAR 支持命令行 + CMake，但 license 服务器和 Docker 化很麻烦 |
| 个人项目 / 无预算 | 免费替代（GCC）足够 |
| 需要长期可复现构建 | 商业工具链的版本管理、license 过期都是隐患 |
| 开源项目 | license 不兼容 |

### 9.3 一个务实建议

**两条路都走一遍**：

1. 用 **GCC + Makefile** 把工程从零搭起来 —— 这个过程逼你理解链接脚本、启动代码、段布局（正是你 STAGE 1/2 在做的事）
2. 用 **IAR** 打开一个现成工程跑一遍 —— 看懂 `.icf` 和 `__ramfunc`，知道商业工具链长什么样

面试时如果被问「你用什么工具链」，能说清「IAR 的 `__ramfunc` 和 GCC 的 `section` 属性做的是同一件事，只是 IAR 把 VMA/LMA 分离的配置藏进了 ICF 的 `initialize by copy`」，比只说「我会用 Keil」强得多。

---

## 十、从 GCC 工程迁到 IAR：检查清单

假设你有个 GCC 工程（`Makefile` + `.ld` + `startup.s`），要迁到 IAR：

### 10.1 工程搭建

- [ ] 新建 IAR 工程，选对芯片型号（决定默认 `.icf` 和器件头文件）
- [ ] 把所有 `.c` 文件加进工程（**逐个确认，别漏**，`Li005` 的头号原因）
- [ ] 头文件搜索路径：**用 `$PROJ_DIR$` 相对路径**，不要用绝对路径
- [ ] 预定义宏：把 `Makefile` 里的 `-D` 全部搬到 `CCDefines`
- [ ] 优化等级：先设 None 调通，再开 High

### 10.2 源码适配

- [ ] `__attribute__((packed))` → `__packed`
- [ ] `__attribute__((section("x")))` → `@ "x"`（变量）或 `__ramfunc`（函数）
- [ ] `__attribute__((weak))` → `__weak`
- [ ] `__attribute__((used))` → `__root`
- [ ] `__attribute__((interrupt))` → `#pragma vector` + `__interrupt`
- [ ] 绝对地址寄存器宏 → 可以用 `@ 地址` 改写（更清爽），或保持宏不动
- [ ] 内联汇编的约束符逐个检查（IAR 支持范围更窄）
- [ ] 确认没有依赖 GCC 特有内建函数（`__builtin_*`）—— IAR 有自己的 `__*` 内建

### 10.3 链接配置

- [ ] `.icf` 里的 ROM/RAM 起止地址**对照 datasheet 核对**（别照抄默认模板）
- [ ] `CSTACK` / `HEAP` 大小按实际需求设
- [ ] 自定义段要加 `place in ...`
- [ ] 需要 RAM 执行的代码：`__ramfunc` + ICF 里确保它在 `readwrite` 区（且启动时会被拷贝）
- [ ] 检查 `.noinit` 段是否声明（`__no_init` 变量需要）

### 10.4 启动文件

IAR 的启动文件通常由 IDE 自动生成（`cstartup_M.s`），**不要用 GCC 的 `startup_stm32f10x.s`**。

它做的事情一样（配栈、清零 `.bss`、拷 `.data`、调 `main`），但语法是 IAR 汇编（不同伪指令）。

> 这一层和第 [[knowledge/embedded/08-STM32入门教程\|STM32 入门教程]] 里讲的 `startup_xx.s` + `__main` 是同一个位置，只是换了实现。

### 10.5 验证

- [ ] 编译零错误零警告
- [ ] 对比 GCC 和 IAR 的 map 文件，看 `.text` / `.data` / `.bss` 大小差异
- [ ] 烧录后跑通原有功能
- [ ] **重点验证中断**（`#pragma vector` 和弱符号覆盖容易出问题）
- [ ] **重点验证 `__ramfunc`**（Flash 擦写如果没放对位置，一擦就崩）

---

## 十一、和 Keil MDK 的关系

经常有人问「IAR 和 Keil 学哪个」。简单对比：

| | IAR | Keil MDK |
|:--|:--|:--|
| 归属 | IAR Systems（瑞典） | ARM 官方（被 ARM 收购） |
| 编译器 | ICCARM | ARMCC（旧）/ AC6（基于 Clang） |
| 链接脚本 | `.icf` | `.sct`（scatter file） |
| 免费版限制 | Kickstart，有代码大小限制 | MDK-Lite，32 KB 限制 |
| 生态 | 欧洲、工业、汽车用得多 | 国内教学、学生用得多 |
| 对 Cortex-M 的贴合度 | 好 | **最好**（ARM 自家） |

**国内情况**：Keil 因为教学资料多、免费版够用，学生群体压倒性；IAR 在**有功能安全要求的外企/汽车零部件供应商**里更常见。

**对你（车载方向）的实际意义**：两者都可能遇到，但只要理解了「编译器差异 + 链接脚本 + RAM 执行」这三个核心，换哪个都能快速上手。

---

## 📎 关联文档

| 想深入 | 去看 |
|:--|:--|
| 编译 → 链接 → 装载完整链路 | [[knowledge/embedded/02-编译链接装载与库\|编译、链接、装载与库]] |
| 启动代码与 `__main` 做了什么 | [[knowledge/embedded/08-STM32入门教程\|STM32 入门教程]] |
| `__attribute__` 九种用法 | [[knowledge/embedded/03-嵌入式C语言的自我修养\|嵌入式的四层硬功夫]] |
| 裸机工程从零搭建 | [[knowledge/embedded/01-裸机编程指南\|裸机编程指南]] |
| CI 里怎么用商业工具链 | 你的 `UDS_S32K144_Bootloader` 仓库 `docs/WINDOWS_BUILD_NOTES.md` |

> ⚠️ **本文事实性说明**：产品定位、支持架构、调试器功能、探针支持等，来自 IAR 官网（2026-11 抓取）。
> 具体版本号、License 价格、Kickstart 的代码大小限制，IAR 多次调整过，**使用前请查官网当前说明**。
> IAR 私有语法的细节，以随安装包的 *IAR C/C++ Development Guide* 为准。