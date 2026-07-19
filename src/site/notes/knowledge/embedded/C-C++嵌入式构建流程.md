---
{"dg-publish":true,"permalink":"/knowledge/embedded/C-C++嵌入式构建流程/","tags":["C","C++","嵌入式","Make","CMake","编译器","链接脚本","交叉编译","静态库","动态库"],"dg-note-properties":{"date":"2026-07-06","tags":["C","C++","嵌入式","Make","CMake","编译器","链接脚本","交叉编译","静态库","动态库"]}}
---


# ⚙️ C/C++ 嵌入式软件构建流程

> 面向嵌入式 CICD 岗位，理解从源码到可执行文件的完整构建链路

---

## 总览：从源码到 ECU 固件

```
源码 (.c/.cpp)
    │
    ▼
  预处理 (Preprocessing)     ── gcc -E
    │
    ▼
  编译 (Compilation)         ── gcc -S     → 汇编文件 (.s)
    │
    ▼
  汇编 (Assembly)            ── as         → 目标文件 (.o)
    │
    ▼
  链接 (Linking)             ── ld         → 可执行文件 (.elf)
    │
    ▼
  格式转换                   ── objcopy    → 二进制 (.bin/.hex/.s19)
    │
    ▼
  ECU Flash 写入             ── UDS/刷写器  → 烧录到 Flash
```

---

## 1️⃣ 编译器 (Compiler)

### GCC 工具链全家桶

以 ARM GCC 为例，一套完整的交叉编译工具链包含：

| 工具 | 文件名示例 | 作用 |
|:----|:----------|:-----|
| **gcc** | `arm-none-eabi-gcc` | C 编译器 |
| **g++** | `arm-none-eabi-g++` | C++ 编译器 |
| **as** | `arm-none-eabi-as` | 汇编器 |
| **ld** | `arm-none-eabi-ld` | 链接器 |
| **objcopy** | `arm-none-eabi-objcopy` | 格式转换（.elf → .bin/.hex） |
| **objdump** | `arm-none-eabi-objdump` | 反汇编，看机器码 |
| **size** | `arm-none-eabi-size` | 查看各段大小 |
| **nm** | `arm-none-eabi-nm` | 列出符号表 |
| **ar** | `arm-none-eabi-ar` | 打包静态库 |
| **readelf** | `arm-none-eabi-readelf` | 查看 ELF 文件结构 |
| **strip** | `arm-none-eabi-strip` | 剥离符号表（减小体积） |
| **gdb** | `arm-none-eabi-gdb` | 调试器 |

### 工具链命名解读

```
arm-none-eabi-gcc
├── arch  = arm     — 目标架构（ARM）
├── vendor = none   — 无特定厂商（通用）
└── abi  = eabi    — 嵌入式 ABI（无操作系统）
```

常见变体：

| 工具链 | 目标 | 场景 |
|:-------|:-----|:-----|
| `arm-none-eabi-` | ARM Cortex-M/R，无 OS | 裸机、FreeRTOS |
| `arm-linux-gnueabihf-` | ARM Cortex-A，Linux | 带 Linux 的嵌入式 |
| `aarch64-none-elf-` | ARM64 裸机 | 现代 MCU |
| `riscv64-unknown-elf-` | RISC-V 裸机 | RISC-V MCU |

### 常用编译选项

```bash
# 最简编译
arm-none-eabi-gcc -c main.c -o main.o

# 常用选项组合
arm-none-eabi-gcc \
  -mcpu=cortex-m4 \          # 指定 CPU 型号
  -mthumb \                   # Thumb-2 指令集
  -mfpu=fpv4-sp-d16 \        # 浮点单元配置
  -mfloat-abi=hard \          # 硬件浮点 ABI
  -O2 \                       # 优化级别 (O0/O1/O2/Os/Og)
  -g \                        # 生成调试信息
  -Wall -Wextra -Werror \     # 警告全部开启
  -ffunction-sections \       # 每个函数独立 section（链接脚本裁剪用）
  -fdata-sections \           # 每个数据独立 section
  -std=c11 \                  # C 标准
  -I./inc \                   # 头文件路径
  -DUSE_FEATURE_X \           # 预定义宏
  -c main.c -o main.o
```

| 优化级别 | 说明 | 调试体验 |
|:--------:|:-----|:--------:|
| `-O0` | 不优化，编译最快 | ✅ 完美 |
| `-O1` | 基础优化，平衡 | ⚠️ 基本可用 |
| `-O2` | 标准优化，生产常用 | ❌ 可能跳转乱 |
| `-Os` | 优化体积，ROM 有限时 | ❌ 同上 |
| `-Og` | 优化但保留调试体验 | ✅ 推荐开发阶段 |

---

## 2️⃣ Make / Makefile

### 为什么嵌入式用 Make？

嵌入式项目的特点是：**文件多、依赖复杂、需要精确控制编译参数**。Make 通过**增量编译**（只编译修改过的文件）和**规则匹配**来提高效率。

### Makefile 核心三要素

```makefile
# 目标(target): 依赖(prerequisites)
#     命令(recipe)

main.o: main.c
    arm-none-eabi-gcc -c main.c -o main.o
```

**执行流程**：如果 `main.c` 比 `main.o` 新（或 `main.o` 不存在）→ 执行命令。

### 实战 Makefile

```makefile
# ===== 变量定义 =====
CC      = arm-none-eabi-gcc
CFLAGS  = -mcpu=cortex-m4 -mthumb -O2 -Wall -ffunction-sections -fdata-sections
LDFLAGS = -T link.ld -Wl,--gc-sections -Wl,-Map=output.map
INCDIR  = -I./inc -I./drivers

SRCS    = main.c uart.c gpio.c timer.c
OBJS    = $(SRCS:.c=.o)
TARGET  = firmware.elf

# ===== 默认目标 =====
all: $(TARGET)

# ===== 模式规则：.c → .o =====
%.o: %.c
    $(CC) $(CFLAGS) $(INCDIR) -c $< -o $@

# ===== 链接 =====
$(TARGET): $(OBJS)
    $(CC) $(LDFLAGS) $^ -o $@
    arm-none-eabi-objcopy -O binary $(TARGET) $(TARGET:.elf=.bin)
    arm-none-eabi-size $(TARGET)

# ===== 伪目标 =====
.PHONY: clean
clean:
    rm -f $(OBJS) $(TARGET) $(TARGET:.elf=.bin) *.map
```

Makefile 的关键变量：

| 自动变量 | 含义 | 示例值 |
|:--------:|:-----|:-------|
| `$@` | 目标文件名 | `main.o` |
| `$<` | 第一个依赖 | `main.c` |
| `$^` | 所有依赖 | `main.c uart.c ...` |
| `$*` | 去掉后缀的文件名 | `main` |

### Make 的进阶能力

```makefile
# 条件编译
ifeq ($(BOARD), revB)
    CFLAGS += -DREV_B
endif

# include 其他 makefile
include ./config/target.mk

# 递归调用子目录 make
SUBDIRS = drivers lib app
.PHONY: subdirs
subdirs:
    @for dir in $(SUBDIRS); do $(MAKE) -C $dir; done
```

### Make 的局限性

- 跨平台差（Linux/macOS/Windows 的 shell 命令不同）
- 语法晦涩（Tab 缩进、空格报错）
- 依赖管理靠手写
- 大型项目维护困难

> **所以有了 CMake** 👇

---

## 3️⃣ CMake — 更高级的构建系统

CMake 不直接编译，而是**生成 Makefile 或其他构建系统文件**（Ninja、Visual Studio 等）。

```
CMakeLists.txt
    │
    ▼
cmake -B build -G "Unix Makefiles"
    │
    ▼
build/Makefile (或 build/build.ninja)
    │
    ▼
make -C build (或 ninja -C build)
    │
    ▼
可执行文件
```

### CMakeLists.txt 基础

```cmake
cmake_minimum_required(VERSION 3.20)
project(Firmware VERSION 1.0.0 LANGUAGES C CXX)

# 设置工具链（交叉编译）
set(CMAKE_SYSTEM_NAME Generic)
set(CMAKE_C_COMPILER arm-none-eabi-gcc)
set(CMAKE_CXX_COMPILER arm-none-eabi-g++)

# 编译选项
set(CMAKE_C_FLAGS "-mcpu=cortex-m4 -mthumb -O2 -Wall -ffunction-sections -fdata-sections")

# 添加可执行目标
add_executable(firmware.elf
    main.c
    uart.c
    gpio.c
    timer.c
)

# 头文件路径
target_include_directories(firmware.elf PRIVATE
    inc
    drivers
)

# 链接脚本
target_link_options(firmware.elf PRIVATE
    -T link.ld
    -Wl,--gc-sections
    -Wl,-Map=output.map
)

# 自定义命令：生成 .bin
add_custom_command(TARGET firmware.elf POST_BUILD
    COMMAND arm-none-eabi-objcopy -O binary firmware.elf firmware.bin
    COMMENT "Generating binary file..."
)

# 打印固件大小
add_custom_command(TARGET firmware.elf POST_BUILD
    COMMAND arm-none-eabi-size firmware.elf
)
```

### CMake 核心概念

| 概念 | 说明 | 示例 |
|:-----|:-----|:-----|
| **target** | 构建目标（可执行/库） | `add_executable`, `add_library` |
| **property** | target 的属性 | `target_include_directories` |
| **generator** | 生成器（Make/Ninja/VS） | `-G Ninja` |
| **variable** | 变量 | `set(SOURCES main.c)` |
| **cache** | 缓存变量，可被用户修改 | `set(VAR "val" CACHE STRING "desc")` |

### 构建库的 CMake

```cmake
# 静态库
add_library(drivers STATIC
    uart.c
    gpio.c
    spi.c
)

# 动态库（嵌入式很少用）
add_library(algorithm SHARED
    fft.c
    filter.c
)

# INTERFACE 库（纯头文件库，不编译）
add_library(hal INTERFACE)
target_include_directories(hal INTERFACE hal/inc)

# 链接库
target_link_libraries(firmware.elf PRIVATE drivers hal)
```

### CMake 变量与条件

```cmake
# 编译宏
target_compile_definitions(firmware.elf PRIVATE
    MCU_STM32F407
    $<$<CONFIG:Debug>:DEBUG_ENABLE>
)

# 条件分支
if(CMAKE_BUILD_TYPE STREQUAL "Debug")
    set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -Og -g")
else()
    set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -O2")
endif()

# 处理源码文件
file(GLOB_RECURSE SOURCES src/*.c)
add_executable(app ${SOURCES})
```

---

## 4️⃣ 链接脚本 (Linker Script)

### 为什么需要链接脚本？

链接脚本告诉链接器：**目标芯片的 Flash/RAM 在哪里、有多大、代码和数据分别放哪里**。

链接脚本的文件格式通常是 `.ld` 或 `.lds`（GNU LD），也有 `.icf`（IAR）或 `.scf`（ARMCC）。

### 嵌入式芯片的存储布局

```text
STM32F407VGT6:
┌────────────────── 0x00000000 ──────────────────┐
│                  Flash (1MB)                     │
│  0x08000000  ┌──────────────────────────────┐    │
│              │  .text（代码段）               │    │
│              │  .rodata（常量、字符串）        │    │
│              │  .init_array（初始化函数表）    │    │
│              └──────────────────────────────┘    │
├────────────────── 0x10000000 ──────────────────┤
│                  RAM (128KB)                     │
│  0x20000000  ┌──────────────────────────────┐    │
│              │  .data（初始化的全局变量）       │    │
│              │  .bss（未初始化的全局变量）       │    │
│              │  .heap（堆，malloc）             │    │
│              │  .stack（栈）                    │    │
│              └──────────────────────────────┘    │
└──────────────────────────────────────────────────┘
```

### 完整链接脚本示例

```ld
/* ===== link.ld — STM32F407 链接脚本 ===== */

/* 芯片内存布局定义 */
MEMORY
{
    FLASH (rx)  : ORIGIN = 0x08000000, LENGTH = 1024K  /* 1MB Flash，只读+执行 */
    RAM   (rwx) : ORIGIN = 0x20000000, LENGTH = 128K   /* 128KB RAM，读写+执行 */
    CCMRAM (rw) : ORIGIN = 0x10000000, LENGTH = 64K    /* 64KB CCM RAM（内核专用）*/
}

/* 入口点（Reset 中断向量） */
ENTRY(Reset_Handler)

/* 段定义 */
SECTIONS
{
    /* ===== Flash 区域 ===== */
    .text :
    {
        . = ALIGN(4);
        KEEP(*(.isr_vector))         /* 中断向量表 — 必须放在最前面 */
        *(.text*)                     /* 代码段 */
        *(.rodata*)                   /* 只读数据（常量） */
        KEEP(*(.init_array))          /* C++ 构造函数表 */
        . = ALIGN(4);
        _etext = .;                   /* 代码结束地址（用于 Copy .data）*/
    } > FLASH

    /* ===== RAM 区域 ===== */
    /* 初始化的全局变量（启动时从 Flash 复制到 RAM）*/
    .data : AT(_etext)
    {
        . = ALIGN(4);
        _sdata = .;                   /* data 起始地址 */
        *(.data*)
        . = ALIGN(4);
        _edata = .;                   /* data 结束地址 */
    } > RAM

    /* 未初始化的全局变量（启动时清零）*/
    .bss :
    {
        . = ALIGN(4);
        _sbss = .;                    /* bss 起始地址 */
        *(.bss*)
        *(COMMON)
        . = ALIGN(4);
        _ebss = .;                    /* bss 结束地址 */
    } > RAM

    /* ===== Heap & Stack ===== */
    .heap (COPY) :
    {
        _heap_start = .;
        . = . + 0x8000;               /* 32KB heap */
        _heap_end = .;
    } > RAM

    .stack (COPY) :
    {
        _stack_end = .;
        . = . + 0x4000;               /* 16KB stack */
        _stack_top = .;
    } > RAM
}
```

### 启动时数据搬运（C 启动代码）

链接脚本定义的符号（`_sdata`、`_edata`、`_etext` 等）在启动代码里被用到：

```c
// startup_stm32f407xx.c（简化版）
extern uint32_t _sdata, _edata, _etext;
extern uint32_t _sbss, _ebss;

void Reset_Handler(void)
{
    // 1. 复制 .data 从 Flash → RAM
    uint32_t *src = &_etext;
    uint32_t *dst = &_sdata;
    while (dst < &_edata) *dst++ = *src++;

    // 2. 清零 .bss
    for (dst = &_sbss; dst < &_ebss; ) *dst++ = 0;

    // 3. 跳转到 main
    main();
}
```

### 链接脚本进阶技巧

```ld
/* 版本号嵌入固件 */
. = ALIGN(4);
.version_info :
{
    KEEP(*(.version_info))
    LONG(1)        /* 主版本 */
    LONG(2)        /* 次版本 */
    LONG(3456)     /* Build number */
    LONG(0x20260706) /*日期*/
} > FLASH

/* 自定义段——把关键函数放到 RAM 里跑（加快速度）*/
.fast_ram :
{
    *(.ramfunc*)
} > RAM AT > FLASH

/* 检查溢出 */
ASSERT(SIZEOF(.text) < 1024K - 4K, "Flash 溢出！")
ASSERT(ORIGIN(RAM) + LENGTH(RAM) - _stack_top > 0x200, "RAM 溢出！")
```

---

## 5️⃣ 静态库 vs 动态库

### 对比

| 特性 | 静态库 (.a) | 动态库 (.so / .dll) |
|:----|:-----------|:-------------------|
| **链接时机** | 编译时直接打入可执行文件 | 运行时加载 |
| **最终体积** | 大（代码复制进每个可执行文件） | 小（多进程共享一份代码） |
| **部署** | 单文件，省心 | 依赖 .so 路径配置 |
| **更新** | 需要重新链接 | 替换 .so 即可 |
| **嵌入式使用** | ✅ **常用** | ❌ **基本不用**（无MMU/无OS） |
| **性能** | 无额外开销 | 动态链接有少量开销 |
| **工具** | `ar` 打包 | `gcc -shared` 生成 |

### 嵌入式场景：99% 用静态库

原因：
- 裸机/RTOS 通常没有动态加载器
- 代码放到 Flash，RAM 宝贵
- 版本管控要求确定性（不依赖外部 .so）

### 制作和使用静态库

```bash
# 1. 编译目标文件
arm-none-eabi-gcc -c uart.c -o uart.o
arm-none-eabi-gcc -c gpio.c -o gpio.o

# 2. 打包成静态库
arm-none-eabi-ar rcs libdrivers.a uart.o gpio.o

# 3. 使用静态库链接
arm-none-eabi-gcc main.o -L. -ldrivers -o firmware.elf
```

对应的 CMake 写法：

```cmake
add_library(drivers STATIC uart.c gpio.c)
target_link_libraries(firmware.elf PRIVATE drivers)
```

### 动态库（嵌入式 Linux 场景）

在嵌入式 Linux（如 ARM Cortex-A 跑 Linux）中，动态库是常态：

```bash
# 编译动态库
arm-linux-gnueabihf-gcc -fPIC -shared algorithm.c -o libalgorithm.so

# 使用
arm-linux-gnueabihf-gcc main.c -L. -lalgorithm -o app

# 运行时设路径
export LD_LIBRARY_PATH=/usr/lib:$LD_LIBRARY_PATH
```

---

## 6️⃣ 交叉编译 (Cross Compilation)

### 什么是交叉编译？

> **在 PC（x86）上编译出给 MCU（ARM）用的代码**

之所以需要交叉编译，是因为 MCU 性能太弱，不能直接在上面跑编译器。

```
开发机 (x86 Linux)                         目标 ECU (ARM Cortex-M4)
┌────────────────────┐                    ┌──────────────────────┐
│ arm-none-eabi-gcc  │                    │                      │
│ arm-none-eabi-g++  │  ─── .elf ───►     │  Flash → 执行        │
│ arm-none-eabi-ld   │  ─── .bin ───►     │                      │
└────────────────────┘                    └──────────────────────┘
  "我帮你编译"                               "我直接运行"
```

### 完整交叉编译流程

```bash
# ===== 1. 安装工具链 =====
# Ubuntu / CentOS
sudo apt install gcc-arm-none-eabi binutils-arm-none-eabi

# 或从 ARM 官网下载预编译包
wget https://developer.arm.com/-/media/Files/downloads/gnu-rm/...-arm-none-eabi.tar.bz2
tar xjf gcc-arm-none-eabi-*.tar.bz2 -C /opt
export PATH=/opt/gcc-arm-none-eabi/bin:$PATH

# ===== 2. 编译 =====
arm-none-eabi-gcc -mcpu=cortex-m4 -mthumb -O2 -c main.c -o main.o

# ===== 3. 链接 =====
arm-none-eabi-gcc -T link.ld main.o uart.o -o firmware.elf -Wl,-Map=output.map

# ===== 4. 转格式 =====
arm-none-eabi-objcopy -O binary firmware.elf firmware.bin
arm-none-eabi-objcopy -O ihex firmware.elf firmware.hex
arm-none-eabi-objcopy -O srec firmware.elf firmware.s19

# ===== 5. 查看结构 =====
arm-none-eabi-size firmware.elf
arm-none-eabi-objdump -d firmware.elf | head -50
arm-none-eabi-readelf -a firmware.elf
```

### CMake 交叉编译工具链文件

创建一个 `cmake/arm-none-eabi.cmake`：

```cmake
# ===== 工具链文件 — arm-none-eabi.cmake =====
set(CMAKE_SYSTEM_NAME Generic)
set(CMAKE_SYSTEM_PROCESSOR arm)

# 编译器
set(CMAKE_C_COMPILER arm-none-eabi-gcc)
set(CMAKE_CXX_COMPILER arm-none-eabi-g++)

# 编译器标志
set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -mcpu=cortex-m4 -mthumb -Wall -ffunction-sections -fdata-sections")
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -mcpu=cortex-m4 -mthumb -Wall -ffunction-sections -fdata-sections -fno-exceptions -fno-rtti")

# 链接器标志
set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} -Wl,--gc-sections -Wl,-Map=output.map")

# 不要尝试链接系统库（嵌入式没有）
set(CMAKE_FIND_ROOT_PATH_MODE_PROGRAM NEVER)
set(CMAKE_FIND_ROOT_PATH_MODE_LIBRARY ONLY)
set(CMAKE_FIND_ROOT_PATH_MODE_INCLUDE ONLY)
set(CMAKE_FIND_ROOT_PATH_MODE_PACKAGE ONLY)
```

使用方式：

```bash
cmake -B build -DCMAKE_TOOLCHAIN_FILE=cmake/arm-none-eabi.cmake
cmake --build build
```

### 常见交叉编译问题排查

| 症状 | 原因 | 解决 |
|:----|:-----|:-----|
| `SIGSEGV` 在 target 上 | 链接脚本内存布局错误 | 检查 Flash/RAM 地址 |
| `undefined reference` | 没链接对应的 .o 或 .a | 检查 Makefile 的 OBJS |
| HardFault 在启动时 | 中断向量表位置不对 | `KEEP(*(.isr_vector))` 必须在 .text 段首 |
| 编译通过但跑飞 | 优化级别太高或 volatile 缺失 | `-O0` 试一下，检查变量加 volatile |
| 浮点运算异常 | FPU 单元设置不匹配 | `-mfpu` + `-mfloat-abi` 要跟芯片匹配 |
| Flash 写不进去 | 固件尺寸超过芯片 Flash | 看 `arm-none-eabi-size` 输出 |

---

## 7️⃣ CI/CD 中的构建流水线（嵌入式特化）

### 典型流水线

```yaml
# Jenkinsfile（嵌入式项目）
pipeline {
    agent { label 'arm-builder' }

    stages {
        stage('Checkout') {
            steps { checkout scm }
        }

        stage('Static Analysis') {
            steps {
                sh 'sonar-scanner -Dsonar.sources=. -Dsonar.language=c'
                sh 'cppcheck --enable=all --std=c11 --suppress=*:test/* src/'
                sh 'misra-checker --config=misra2012.xml src/'
            }
        }

        stage('Build') {
            steps {
                sh 'cmake -B build -DCMAKE_TOOLCHAIN_FILE=cmake/arm-none-eabi.cmake'
                sh 'cmake --build build'
            }
        }

        stage('Unit Test') {
            steps {
                // 主机上编译 x86 版本的单元测试
                sh 'cmake -B build-test -DTESTING=ON'
                sh 'cmake --build build-test'
                sh './build-test/run_tests'
            }
        }

        stage('Archive') {
            steps {
                archiveArtifacts artifacts: 'build/*.bin, build/*.hex, build/*.map, build/*.elf'
            }
        }
    }
}
```

### 嵌入式 CICD 的特殊点

| 环节 | 注意事项 |
|:-----|:---------|
| **构建环境** | 交叉编译工具链需提前安装或使用 Docker 镜像 |
| **软件许可** | IAR/KEIL 等商业编译器需要 License 服务器 |
| **静态分析** | MISRA 检查是嵌入式必修课 |
| **单元测试** | 需在 x86 主机上用 `gcc -DTESTING` 编译，mock 硬件接口 |
| **HIL 测试** | 真正的验证需要把固件刷到 ECU 上跑 |
| **版本号注入** | 编译时注入 Git commit / build number 到代码中 |
| **产物管理** | .bin/.hex + .map 文件需要长期保存（可追溯） |

---

## 📚 推荐学习资源

| 资源 | 说明 |
|:-----|:------|
| **CMake 官方教程** (cmake.org) | Step-by-step 入门到精通 |
| **GCC 官方文档** (gcc.gnu.org) | 编译器选项权威参考 |
| **《ARM Cortex-M 嵌入式系统》** | 嵌入式构建从原理到实践 |
| **《CMake Cookbook》** | CMake 进阶技巧 |
| **GNU ld 手册** (sourceware.org/binutils/docs/ld) | 链接脚本完全指南 |
| **ARM 信息中心** (developer.arm.com) | ARM 工具链官方文档 |

## 📂 关联文件

- 奇瑞 CICD 岗位：`../job/奇瑞汽车-CICD工程师/奇瑞汽车-CICD工程师.md`
- 奇瑞学习路线：`../job/奇瑞汽车-CICD工程师/技术栈学习路线-奇瑞汽车.md`
- UDS 入门：`UDS入门指南.md`
- Docker 底层原理：`Docker底层原理入门.md`

---

> 🦐 虾管家 · 2026-07-06
