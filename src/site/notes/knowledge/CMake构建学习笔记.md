---
{"dg-publish":true,"permalink":"/knowledge/CMake构建学习笔记/","dg-note-properties":{}}
---

# CMake 构建系统学习笔记

---

## 📌 一、CMake 是什么

CMake（Cross-platform Make）是一个**跨平台构建系统生成器**。它不是编译器，也不是包管理器——它的工作是**生成**供不同平台原生构建工具（Makefile、Ninja、Visual Studio 工程等）使用的构建文件。

```
┌──────────────┐     CMake     ┌──────────────────┐
│              │     跑一次      │                  │
│ CMakeLists   │──────────────▶│   Makefile       │
│     .txt     │               │   or Ninja       │
│              │               │   or VS .sln     │
└──────────────┘               │   or Xcode       │
                               │   ...            │
                               └──────────────────┘
                                    │
                                     make / ninja / msbuild
                                    ▼
                               ┌──────────────────┐
                               │   可执行文件       │
                               │   / 静态库        │
                               │   / 动态库        │
                               └──────────────────┘
```

**核心价值：**
1. **跨平台** —— 一套 `CMakeLists.txt` 在 Linux/macOS/Windows/嵌入式上都能用
2. **模块化** —— `add_subdirectory()` 天然支持大型项目分模块管理
3. **现代 CMake（3.x+）** —— 基于**目标（Target）**而非变量的设计，更易维护
4. **嵌入式友好** —— 工具链文件（Toolchain File）机制完美适配交叉编译

---

## 📌 二、CMake 构建的两阶段

```
阶段 1：Configure（配置）             阶段 2：Build（构建）
┌────────────────────┐            ┌────────────────────┐
│ cmake -S . -B build │           │ cmake --build build │
│                    │            │                    │
│ 读取 CMakeLists.txt  │            │ 调用底层工具       │
│ 检测编译器          │            │ (make/ninja)      │
│ 解析依赖            │──────▶    │ 编译 → 链接       │
│ 生成 Makefile       │            │ 生成产物           │
│ 缓存到 CMakeCache   │            │                    │
└────────────────────┘            └────────────────────┘
```

**常用命令速查：**

```bash
# 配置（第一次，或修改了 CMakeLists.txt 后）
cmake -S . -B build

# 指定构建类型（Debug / Release / RelWithDebInfo / MinSizeRel）
cmake -S . -B build -DCMAKE_BUILD_TYPE=Debug

# 指定安装目录
cmake -S . -B build -DCMAKE_INSTALL_PREFIX=/usr/local

# 构建
cmake --build build --parallel $(nproc)

# 安装
cmake --install build --prefix /output/dir

# 清理
cmake --build build --target clean
```

---

## 📌 三、CMakeLists.txt 基础语法

### 3.1 最小项目

```cmake
# CMakeLists.txt —— 最小示范
cmake_minimum_required(VERSION 3.16)

project(
    HelloEmbedded          # 项目名称
    VERSION 1.0.0          # 版本号
    LANGUAGES C ASM        # 语言（C/CXX/ASM）
)

# 定义可执行文件目标
add_executable(main main.c startup.s)

# 可选的：指定 C 标准
set(CMAKE_C_STANDARD 11)
set(CMAKE_C_STANDARD_REQUIRED ON)

# 可选的：添加头文件搜索路径
target_include_directories(main PRIVATE
    ${CMAKE_CURRENT_SOURCE_DIR}/inc
)
```

### 3.2 定义目标（Target）

现代 CMake 的核心概念是 **目标（Target）**，即 `add_executable` 或 `add_library` 定义的东西。

```cmake
# --- 可执行文件 ---
add_executable(app main.c)

# --- 静态库 ---
add_library(drivers STATIC
    uart.c
    spi.c
    gpio.c
)

# --- 动态库/共享库 ---
add_library(protocol SHARED
    can.c
    lin.c
)

# --- 对象库（不归档，直接用于链接，编译快） ---
add_library(mcal_objs OBJECT
    adc.c
    timer.c
)
# 使用对象库的其他目标自动获取其编译属性
target_link_libraries(app PRIVATE mcal_objs)
```

### 3.3 目标属性传递 —— PRIVATE / PUBLIC / INTERFACE

这是现代 CMake 最精髓的设计：

```cmake
# PRIVATE  ：仅对自己有效，不传导给依赖者
# PUBLIC   ：对自己有效，也会传导给依赖者
# INTERFACE：仅传导给依赖者，自己不需要（常用于纯头文件库）

add_library(core STATIC core.c)

# core 自己编译时需要这些头文件
target_include_directories(core PUBLIC
    ${CMAKE_CURRENT_SOURCE_DIR}/core/inc
)

# core 链接时需要这个库
target_link_libraries(core PUBLIC
    mcal_objs
)

# app 链接 core 时，自动获得：
#   - core/inc 头文件路径（因为 PUBLIC）
#   - mcal_objs 的编译属性和头文件（因为 PUBLIC 传导）
target_link_libraries(app PRIVATE
    core
)
```

### 3.4 编译选项管理

```cmake
# 全局设置（影响所有目标，不推荐滥用）
add_compile_options(-Wall -Werror -Wextra)
add_definitions(-DUSE_HAL_DRIVER)

# 推荐：目标级别的编译选项
target_compile_options(app PRIVATE
    $<$<CONFIG:Debug>:-Og -g3>
    $<$<CONFIG:Release>:-Os -flto>
)

target_compile_definitions(app PRIVATE
    STM32F407xx
    USE_HAL_DRIVER
    $<$<CONFIG:Debug>:DEBUG_ENABLE>
)
```

### 3.5 生成器表达式（Generator Expressions）

在构建系统生成时才求值，适用于条件编译、跨平台条件：

```cmake
# 语法：$<condition:value>

# 1. 不同构建类型不同标志
target_compile_options(app PRIVATE
    $<$<CONFIG:Debug>:-Og -g3>
    $<$<CONFIG:Release>:-Os -DNDEBUG>
)

# 2. 不同平台不同链接库
target_link_libraries(app PRIVATE
    $<$<PLATFORM_ID:Linux>:rt pthread>
    $<$<PLATFORM_ID:Windows>:ws2_32>
)

# 3. 编译器版本检测
target_compile_definitions(app PRIVATE
    $<$<C_COMPILER_ID:GNU>:COMPILER_GCC>
    $<$<C_COMPILER_ID:ARMClang>:COMPILER_ARMCLANG>
)

# 4. 目标属性获取
target_include_directories(app PRIVATE
    $<TARGET_PROPERTY:core,INTERFACE_INCLUDE_DIRECTORIES>
)
```

---

## 📌 四、嵌入式交叉编译 —— Toolchain File

嵌入式开发的 CMake 最常用功能：通过 **工具链文件（Toolchain File）** 告诉 CMake 用什么编译器、链接器、以及目标架构。

### 4.1 ARM GCC 工具链文件

```cmake
# arm-gcc-toolchain.cmake
# 用法：cmake -DCMAKE_TOOLCHAIN_FILE=arm-gcc-toolchain.cmake -S . -B build

# 目标系统（告诉 CMake 这是交叉编译）
set(CMAKE_SYSTEM_NAME Generic)
set(CMAKE_SYSTEM_PROCESSOR arm)

# 指定编译器
set(CMAKE_C_COMPILER arm-none-eabi-gcc)
set(CMAKE_CXX_COMPILER arm-none-eabi-g++)
set(CMAKE_ASM_COMPILER arm-none-eabi-gcc)
set(CMAKE_AR arm-none-eabi-ar)
set(CMAKE_OBJCOPY arm-none-eabi-objcopy)
set(CMAKE_OBJDUMP arm-none-eabi-objdump)
set(CMAKE_SIZE arm-none-eabi-size)

# 禁用编译器测试（交叉编译不允许运行可执行文件）
set(CMAKE_TRY_COMPILE_TARGET_TYPE STATIC_LIBRARY)

# 架构相关编译标志
set(CMAKE_C_FLAGS_INIT "-mcpu=cortex-m4 -mthumb -mfloat-abi=hard -mfpu=fpv4-sp-d16")
set(CMAKE_CXX_FLAGS_INIT "${CMAKE_C_FLAGS_INIT}")
set(CMAKE_ASM_FLAGS_INIT "${CMAKE_C_FLAGS_INIT} -x assembler-with-cpp")

# 链接器标志（含 --specs=nano.specs 缩小体积）
set(CMAKE_EXE_LINKER_FLAGS_INIT
    "-mcpu=cortex-m4 -mthumb -mfloat-abi=hard -mfpu=fpv4-sp-d16 \
     --specs=nano.specs --specs=nosys.specs -Wl,--gc-sections"
)
```

### 4.2 在项目中启用工具链

```cmake
# 方法一：命令行指定（推荐）
cmake -S . -B build \
    -DCMAKE_TOOLCHAIN_FILE=cmake/arm-gcc-toolchain.cmake \
    -DCMAKE_BUILD_TYPE=Release

# 方法二：在 CMakeLists.txt 中预设（但必须在使用 project() 之前）
# 优先级低于命令行，不推荐硬编码
set(CMAKE_TOOLCHAIN_FILE cmake/arm-gcc-toolchain.cmake CACHE STRING "")
project(MyProject)
```

---

## 📌 五、嵌入式实战：STM32 + Bootloader + APP 双目标构建

### 5.1 目录结构

```
project/
├── CMakeLists.txt              # 顶层 CMake 入口
├── cmake/
│   ├── arm-gcc-toolchain.cmake # 工具链文件
│   ├── bootloader.ld.in        # BL 链接脚本模板（@变量替换@）
│   └── application.ld.in       # APP 链接脚本模板
├── bootloader/
│   ├── CMakeLists.txt
│   ├── src/
│   │   ├── startup.s
│   │   ├── main.c
│   │   └── flash_driver.c
│   └── inc/
├── application/
│   ├── CMakeLists.txt
│   ├── src/
│   │   ├── startup.s
│   │   ├── main.c
│   │   └── app_task.c
│   └── inc/
├── shared/
│   ├── CMakeLists.txt
│   ├── src/
│   │   └── crc.c
│   └── inc/
│       └── crc.h
└── hal/
    ├── CMakeLists.txt
    ├── src/
    │   ├── uart.c
    │   └── gpio.c
    └── inc/
```

### 5.2 顶层 CMakeLists.txt

```cmake
cmake_minimum_required(VERSION 3.20)

# 项目定义（必须在工具链文件之后生效）
project(
    Stm32Fw
    VERSION 1.0.0
    LANGUAGES C ASM
)

# ===== 全局编译选项 =====
set(CMAKE_C_STANDARD 11)
set(CMAKE_C_STANDARD_REQUIRED ON)

# 公共编译标志（依赖工具链文件已设置 -mcpu 等）
add_compile_options(
    -ffunction-sections -fdata-sections
    -Wall -Wextra -Werror
)

# 公共宏定义
add_compile_definitions(
    STM32F407xx
    USE_HAL_DRIVER
)

# ===== 链接脚本模板处理 =====
# 使用 configure_file() 替换 @变量@ 为 CMake 变量值
# 这样可以在链接脚本中使用 CMake 变量

set(BL_FLASH_ORIGIN  "0x08000000")
set(BL_FLASH_LENGTH  "64K")
set(APP_FLASH_ORIGIN "0x08010000")
set(APP_FLASH_LENGTH "448K")
set(RAM_ORIGIN       "0x20000000")
set(RAM_LENGTH       "128K")

# 生成为链接脚本（交给各子目录使用）
configure_file(
    cmake/bootloader.ld.in
    ${CMAKE_BINARY_DIR}/linker/bootloader.ld
    @ONLY
)
configure_file(
    cmake/application.ld.in
    ${CMAKE_BINARY_DIR}/linker/application.ld
    @ONLY
)

# ===== 子模块 =====
add_subdirectory(hal)
add_subdirectory(shared)
add_subdirectory(bootloader)
add_subdirectory(application)

# ===== 自定义目标：产物汇总与烧录 =====

# 固件签名
add_custom_command(OUTPUT ${CMAKE_BINARY_DIR}/app_signed.bin
    COMMAND python3 ${CMAKE_SOURCE_DIR}/scripts/sign.py
        -i ${CMAKE_BINARY_DIR}/application/application.bin
        -o ${CMAKE_BINARY_DIR}/app_signed.bin
    DEPENDS application_bin
    COMMENT "Signing application firmware..."
)

add_custom_target(firmware ALL
    DEPENDS
        ${CMAKE_BINARY_DIR}/bootloader/bootloader.hex
        ${CMAKE_BINARY_DIR}/app_signed.bin
)

# 烧录 Bootloader
add_custom_target(flash_bl
    COMMAND st-flash write
        ${CMAKE_BINARY_DIR}/bootloader/bootloader.bin 0x08000000
    DEPENDS bootloader_bin
)

# 烧录 Application
add_custom_target(flash_app
    COMMAND st-flash write
        ${CMAKE_BINARY_DIR}/app_signed.bin 0x08010000
    DEPENDS app_signed.bin
)

# 合并烧录
add_custom_target(flash_all
    COMMAND st-flash write
        ${CMAKE_BINARY_DIR}/bootloader/bootloader.bin 0x08000000
    COMMAND st-flash write
        ${CMAKE_BINARY_DIR}/app_signed.bin 0x08010000
    DEPENDS firmware
)
```

### 5.3 bootloader/CMakeLists.txt

```cmake
# Bootloader 子项目

# 编译定义
add_compile_definitions(BUILD_BOOTLOADER)

# 源文件
file(GLOB_RECURSE BL_SOURCES
    ${CMAKE_CURRENT_SOURCE_DIR}/src/*.c
    ${CMAKE_CURRENT_SOURCE_DIR}/src/*.s
)

# 目标
add_executable(bootloader ${BL_SOURCES})

# 链接其他库
target_link_libraries(bootloader PRIVATE
    hal
    shared
)

# 头文件路径
target_include_directories(bootloader PRIVATE
    ${CMAKE_CURRENT_SOURCE_DIR}/inc
)

# 使用 Bootloader 专用链接脚本
target_link_options(bootloader PRIVATE
    -T ${CMAKE_BINARY_DIR}/linker/bootloader.ld
    -Wl,-Map=${CMAKE_CURRENT_BINARY_DIR}/bootloader.map
)

# 自定义命令：构建后生成 .bin 和 .hex
add_custom_command(TARGET bootloader POST_BUILD
    COMMAND ${CMAKE_OBJCOPY} -O binary
        ${CMAKE_CURRENT_BINARY_DIR}/bootloader
        ${CMAKE_CURRENT_BINARY_DIR}/bootloader.bin
    COMMAND ${CMAKE_OBJCOPY} -O ihex
        ${CMAKE_CURRENT_BINARY_DIR}/bootloader
        ${CMAKE_CURRENT_BINARY_DIR}/bootloader.hex
    COMMAND ${CMAKE_SIZE}
        ${CMAKE_CURRENT_BINARY_DIR}/bootloader
        | tee ${CMAKE_CURRENT_BINARY_DIR}/bootloader_size.txt
    COMMENT "Post-build: generating bin/hex/size for bootloader"
)

# 供顶层引用的别名目标
add_custom_target(bootloader_bin DEPENDS
    ${CMAKE_CURRENT_BINARY_DIR}/bootloader.bin
)
add_dependencies(bootloader_bin bootloader)
```

### 5.4 application/CMakeLists.txt

```cmake
# Application 子项目

add_compile_definitions(BUILD_APPLICATION)

file(GLOB_RECURSE APP_SOURCES
    ${CMAKE_CURRENT_SOURCE_DIR}/src/*.c
    ${CMAKE_CURRENT_SOURCE_DIR}/src/*.s
)

add_executable(application ${APP_SOURCES})

target_link_libraries(application PRIVATE
    hal
    shared
)

target_include_directories(application PRIVATE
    ${CMAKE_CURRENT_SOURCE_DIR}/inc
)

target_link_options(application PRIVATE
    -T ${CMAKE_BINARY_DIR}/linker/application.ld
    -Wl,-Map=${CMAKE_CURRENT_BINARY_DIR}/application.map
)

add_custom_command(TARGET application POST_BUILD
    COMMAND ${CMAKE_OBJCOPY} -O binary
        ${CMAKE_CURRENT_BINARY_DIR}/application
        ${CMAKE_CURRENT_BINARY_DIR}/application.bin
    COMMAND ${CMAKE_OBJCOPY} -O ihex
        ${CMAKE_CURRENT_BINARY_DIR}/application
        ${CMAKE_CURRENT_BINARY_DIR}/application.hex
    COMMAND ${CMAKE_SIZE}
        ${CMAKE_CURRENT_BINARY_DIR}/application
    COMMENT "Post-build: generating bin/hex/size for application"
)

add_custom_target(application_bin
    DEPENDS ${CMAKE_CURRENT_BINARY_DIR}/application.bin
)
add_dependencies(application_bin application)
```

### 5.5 链接脚本模板（.ld.in）

利用 CMake 的 `configure_file()` 在配置阶段注入变量：

```ld
/* bootloader.ld.in — 链接脚本模板 */
MEMORY
{
    FLASH (rx) : ORIGIN = @BL_FLASH_ORIGIN@, LENGTH = @BL_FLASH_LENGTH@
    RAM   (rwx): ORIGIN = @RAM_ORIGIN@,      LENGTH = @RAM_LENGTH@
}
/* ... 其余段定义同上文 ... */
```

**优点：** 内存布局参数在顶层 `CMakeLists.txt` 中集中定义，一处修改，处处生效。

---

## 📌 六、AUTOSAR 项目中的 CMake 实践

### 6.1 AUTOSAR 模块化的 CMake 组织

```cmake
# 顶层 CMakeLists.txt
cmake_minimum_required(VERSION 3.20)
project(AUTOSAR_ECU C ASM)

# ===== AUTOSAR 基础软件层（BSW） =====

# --- MCAL （微控制器抽象层） ---
add_library(mcal STATIC)
target_sources(mcal PRIVATE
    mcal/Adc/Adc.c
    mcal/Dio/Dio.c
    mcal/Spi/Spi.c
    mcal/Can/Can.c
    mcal/Fls/Fls.c  # Flash 驱动
)
target_include_directories(mcal PUBLIC
    ${CMAKE_CURRENT_SOURCE_DIR}/mcal/Adc/inc
    ${CMAKE_CURRENT_SOURCE_DIR}/mcal/Dio/inc
    ${CMAKE_CURRENT_SOURCE_DIR}/mcal/Spi/inc
    ${CMAKE_CURRENT_SOURCE_DIR}/mcal/Can/inc
    ${CMAKE_CURRENT_SOURCE_DIR}/mcal/Fls/inc
)

# --- 操作系统（OS） ---
add_library(os STATIC)
target_sources(os PRIVATE
    os/SchM.c       # 调度管理器
    os/Task.c       # 任务管理
    os/Counter.c    # 定时计数器
    os/Alarm.c      # 闹钟
)
target_include_directories(os PUBLIC
    ${CMAKE_CURRENT_SOURCE_DIR}/os/inc
)

# --- 通信栈（COM） ---
add_library(com STATIC)
target_sources(com PRIVATE
    com/CanIf.c     # CAN 接口层
    com/CanTp.c     # CAN TP（传输协议）
    com/Com.c       # 通信管理器
    com/PduR.c      # PDU 路由器
)
target_include_directories(com PUBLIC
    ${CMAKE_CURRENT_SOURCE_DIR}/com/inc
)
target_link_libraries(com PUBLIC mcal)

# --- 诊断（Diag） ---
add_library(diag STATIC)
target_sources(diag PRIVATE
    diag/Dcm.c      # 诊断通信管理器
    diag/Dem.c      # 诊断事件管理器
)
target_include_directories(diag PUBLIC
    ${CMAKE_CURRENT_SOURCE_DIR}/diag/inc
)

# --- NVM（非易失性存储管理） ---
add_library(nvm STATIC)
target_sources(nvm PRIVATE
    nvm/NvM.c
    nvm/NvM_Cfg.c
)
target_include_directories(nvm PUBLIC
    ${CMAKE_CURRENT_SOURCE_DIR}/nvm/inc
)

# ===== RTE（运行时环境） =====
# （通常由工具生成，纯 C 代码）
add_library(rte STATIC)
target_sources(rte PRIVATE
    rte/Rte.c
    rte/Rte_Type.c
)
target_include_directories(rte PUBLIC
    ${CMAKE_CURRENT_SOURCE_DIR}/rte/inc
)
target_link_libraries(rte PUBLIC
    com
    os
    diag
    nvm
)

# ===== SWC（应用软件组件） =====
add_library(swc_engine STATIC
    swc/EngineControl/EngineControl.c
    swc/EngineControl/EngineControl_Cfg.c
)
target_include_directories(swc_engine PUBLIC
    ${CMAKE_CURRENT_SOURCE_DIR}/swc/EngineControl
)
target_link_libraries(swc_engine PUBLIC rte)

add_library(swc_brake STATIC
    swc/BrakeControl/BrakeControl.c
)
target_include_directories(swc_brake PUBLIC
    ${CMAKE_CURRENT_SOURCE_DIR}/swc/BrakeControl
)
target_link_libraries(swc_brake PUBLIC rte)

# ===== 最终可执行文件 =====
add_executable(ecu_fw)

target_link_libraries(ecu_fw PRIVATE
    mcal
    os
    com
    diag
    nvm
    rte
    swc_engine
    swc_brake
)

# 静态库链接时注意顺序（从底层到上层）
target_link_options(ecu_fw PRIVATE
    -T ${CMAKE_BINARY_DIR}/linker/autosar.ld
    -Wl,--start-group
    $<TARGET_FILE:swc_engine>
    $<TARGET_FILE:swc_brake>
    $<TARGET_FILE:rte>
    $<TARGET_FILE:com>
    $<TARGET_FILE:os>
    $<TARGET_FILE:diag>
    $<TARGET_FILE:nvm>
    $<TARGET_FILE:mcal>
    -Wl,--end-group
    -Wl,-Map=${CMAKE_CURRENT_BINARY_DIR}/ecu_fw.map
)
```

### 6.2 AUTOSAR 条件编译（模块开关）

```cmake
# 通过 CMake 变量控制 AUTOSAR 模块启用
option(BUILD_CAN   "Enable CAN stack"   ON)
option(BUILD_LIN   "Enable LIN stack"   OFF)
option(BUILD_FLEX  "Enable FlexRay"     OFF)
option(BUILD_DIAG  "Enable Diagnostics" ON)

# 根据选项添加源文件和宏定义
if(BUILD_CAN)
    target_sources(com PRIVATE
        com/CanIf.c
        com/CanTp.c
    )
    target_compile_definitions(com PRIVATE CAN_STACK_ENABLED)
endif()

if(BUILD_LIN)
    target_sources(com PRIVATE
        com/LinIf.c
        com/LinTp.c
    )
    target_compile_definitions(com PRIVATE LIN_STACK_ENABLED)
endif()

if(BUILD_DIAG)
    target_link_libraries(ecu_fw PRIVATE diag)
    target_compile_definitions(ecu_fw PRIVATE DIAG_ENABLED)
endif()
```

### 6.3 自动代码生成集成

```cmake
# 调用 AUTOSAR 代码生成器
set(ARXML_DIR ${CMAKE_SOURCE_DIR}/arxml)
set(GEN_DIR    ${CMAKE_BINARY_DIR}/generated)

# Vector DaVinci 示例
add_custom_command(
    OUTPUT
        ${GEN_DIR}/rte/Rte.c
        ${GEN_DIR}/rte/Rte.h
    COMMAND davinci_configurator
        --input ${ARXML_DIR}/System.arxml
        --output ${GEN_DIR}
    DEPENDS
        ${ARXML_DIR}/System.arxml
        ${ARXML_DIR}/EcuCfg.arxml
    COMMENT "Generating RTE code from ARXML..."
)

# 将生成的代码添加到构建目标
target_sources(rte PRIVATE
    ${GEN_DIR}/rte/Rte.c
    ${GEN_DIR}/rte/Rte.h
)
target_include_directories(rte PUBLIC
    ${GEN_DIR}/rte
)
```

---

## 📌 七、CMake 常用核心命令速查表

### 7.1 项目与目标

| 命令 | 作用 |
|:---|:---|
| `add_executable(name src...)` | 定义可执行文件目标 |
| `add_library(name [STATIC/SHARED/OBJECT] src...)` | 定义库目标 |
| `add_subdirectory(dir)` | 添加子目录（含子 CMakeLists.txt） |
| `target_sources(target PRIVATE/PUBLIC/INTERFACE src...)` | 为目标追加源文件 |

### 7.2 属性传递

| 命令 | 作用 |
|:---|:---|
| `target_include_directories(target [PUBLIC/PRIVATE/INTERFACE] dirs...)` | 头文件搜索路径 |
| `target_link_libraries(target [PUBLIC/PRIVATE/INTERFACE] libs...)` | 链接依赖 |
| `target_compile_definitions(target [PUBLIC/PRIVATE/INTERFACE] defs...)` | 编译宏定义 |
| `target_compile_options(target [PUBLIC/PRIVATE/INTERFACE] opts...)` | 编译选项 |
| `target_link_options(target [PUBLIC/PRIVATE/INTERFACE] opts...)` | 链接选项 |

### 7.3 文件操作

| 命令 | 作用 |
|:---|:---|
| `file(GLOB var pattern)` | 收集匹配的文件列表（慎用，不会自动感知新增文件） |
| `file(GLOB_RECURSE var pattern)` | 递归收集 |
| `configure_file(input output @ONLY)` | 模板文件生成，替换 `@VAR@` |
| `file(COPY src DESTINATION dest)` | 复制文件 |
| `file(WRITE path content)` | 写入文件 |

### 7.4 构建行为

| 命令 | 作用 |
|:---|:---|
| `add_custom_command(TARGET target POST_BUILD COMMAND ...)` | 构建后自动执行 |
| `add_custom_command(OUTPUT out COMMAND ... DEPENDS deps)` | 自定义输出文件 |
| `add_custom_target(name ALL DEPENDS deps)` | 自定义目标（构建时触发） |
| `add_dependencies(target deps...)` | 显式声明目标间依赖 |

### 7.5 条件与循环

| 命令 | 作用 |
|:---|:---|
| `if() / elseif() / else() / endif()` | 条件分支 |
| `foreach(var RANGE n)` / `foreach(var IN LISTS list)` | 循环 |
| `option(var "desc" default)` | 可配置的布尔开关 |
| `set(var value CACHE BOOL "desc")` | 缓存变量（在 cmake-gui 中可见） |

---

## 📌 八、嵌入式 CMake 常见陷阱与最佳实践

### 8.1 构建类型（HAS NO EFFECT）

**陷阱：** 交叉编译时 `CMAKE_BUILD_TYPE` 默认**空**，很多新手以为设置成 `Release` 就会自动加 `-O3`，其实 CMake 只有**单构建类型生成器**（Makefile/Ninja）才使用 `CMAKE_BUILD_TYPE`。Visual Studio 等多配置生成器用 `--config` 参数。

```cmake
# ✅ 正确做法：对于嵌入式，显式设置编译标志
set(CMAKE_C_FLAGS_RELEASE "-Os -DNDEBUG -flto")
set(CMAKE_C_FLAGS_DEBUG   "-Og -g3 -DDEBUG")

# 或者在项目级别的编译选项中使用生成器表达式
target_compile_options(app PRIVATE
    $<$<CONFIG:Release>:-Os -DNDEBUG>
    $<$<CONFIG:Debug>:-Og -g3 -DDEBUG>
)
```

### 8.2 GLOB 不自动检测新文件

```cmake
# ❌ 不推荐：GLOB 在 cmake 配置时求值，新增文件不会自动触发 reconfigure
file(GLOB APP_SOURCES src/*.c)

# ✅ 推荐：显式列出源文件
set(APP_SOURCES
    src/main.c
    src/task.c
    src/uart.c
)

# 或者：用 GLOB 但告知开发者需手动 re-configure
file(GLOB APP_SOURCES src/*.c)
# 在 CMakeLists.txt 开头注释：
# !! 新增文件后请执行 cmake -S . -B build 重新配置 !!
```

### 8.3 工具链文件与 project() 的顺序

```cmake
# ❌ 错误：project() 在 toolchain 之前，CMake 会用宿主编译器
project(MyProject)
set(CMAKE_TOOLCHAIN_FILE arm-gcc.cmake)

# ✅ 正确：toolchain 在命令行或缓存中先于 project()
# cmake -DCMAKE_TOOLCHAIN_FILE=arm-gcc.cmake -S . -B build
cmake_minimum_required(VERSION 3.20)

# 或使用 CMAKE_TOOLCHAIN_FILE 变量必须在 project() 前
set(CMAKE_TOOLCHAIN_FILE ${CMAKE_CURRENT_SOURCE_DIR}/cmake/arm-gcc.cmake
    CACHE STRING "")
project(MyProject C ASM)
```

### 8.4 不要用 file(GLOB) 递归 AUTOSAR 生成代码

```cmake
# ❌ 危险：生成代码目录可能很大，递归 GLOB 可能包含临时文件
file(GLOB_RECURSE RTE_SOURCES ${GENERATED_DIR}/rte/*.c)

# ✅ 正确：明确列出或使用子目录 CMakeLists.txt
# rte/CMakeLists.txt 由生成器一起输出
add_subdirectory(${GENERATED_DIR}/rte)
```

### 8.5 静态库链接顺序

```cmake
# ❌ 陷阱：GCC 链接器是单次扫描，顺序错误会导致 "undefined reference"
target_link_libraries(ecu_fw PRIVATE
    swc_engine  # 引用了 rte 的符号
    rte         # 引用了 com 的符号
    com         # 引用了 mcal 的符号
    mcal
)

# ✅ 使用 --start-group / --end-group 处理循环依赖
target_link_options(ecu_fw PRIVATE
    -Wl,--start-group
    $<TARGET_FILE:swc_engine>
    $<TARGET_FILE:rte>
    $<TARGET_FILE:com>
    $<TARGET_FILE:mcal>
    -Wl,--end-group
)
```

### 8.6 build 目录与 out-of-source build

```cmake
# ✅ CMake 强制推荐 out-of-source build
# 生成的中间文件和产物与源码彻底分离

# 正确做法（不是 cd build && cmake ..）
cmake -S . -B build

# 切换构建类型无需清空 build
cmake -S . -B build -DCMAKE_BUILD_TYPE=Debug

# 要完全重建 -> 删 build 目录重来
rm -rf build && cmake -S . -B build
```

---

## 📌 九、常用进阶技巧

### 9.1 依赖版本自动注入

```cmake
# 自动从 Git 获取版本信息
execute_process(
    COMMAND git describe --tags --always --dirty
    WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}
    OUTPUT_VARIABLE GIT_VERSION
    OUTPUT_STRIP_TRAILING_WHITESPACE
    ERROR_QUIET
)

execute_process(
    COMMAND git rev-parse --short HEAD
    WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}
    OUTPUT_VARIABLE GIT_HASH
    OUTPUT_STRIP_TRAILING_WHITESPACE
    ERROR_QUIET
)

# 生成版本头文件
configure_file(
    ${CMAKE_SOURCE_DIR}/cmake/version.h.in
    ${CMAKE_BINARY_DIR}/generated/version.h
    @ONLY
)

target_include_directories(app PRIVATE
    ${CMAKE_BINARY_DIR}/generated
)
```

对应的模板文件 `version.h.in`：

```c
#ifndef VERSION_H
#define VERSION_H

#define FW_VERSION "@GIT_VERSION@"
#define FW_GIT_HASH "@GIT_HASH@"
#define FW_BUILD_TIME "@CMAKE_TIMESTAMP@"

#endif
```

### 9.2 编译数据库（compile_commands.json）

```cmake
# 对 IDE（CLion、VS Code、Vim）非常有用
# 在 CMakeLists.txt 中：
set(CMAKE_EXPORT_COMPILE_COMMANDS ON)

# 然后配置后生成：
# build/compile_commands.json

# 可以把它 symlink 到源码根目录
execute_process(
    COMMAND ${CMAKE_COMMAND} -E create_symlink
        ${CMAKE_BINARY_DIR}/compile_commands.json
        ${CMAKE_SOURCE_DIR}/compile_commands.json
    ERROR_QUIET
)
```

### 9.3 固件二进制大小报告

```cmake
# 构建后自动输出 Flash/RAM 占用
add_custom_command(TARGET app POST_BUILD
    COMMAND ${CMAKE_SIZE} --format=berkeley $<TARGET_FILE:app>
    COMMAND ${CMAKE_OBJSIZE} -B --target=elf32-littlearm $<TARGET_FILE:app>
        | awk '{print $1, $2, $3, $4}'
    COMMENT "Firmware size report:"
    USES_TERMINAL
)
```

### 9.4 Ninja（替代 Make）加速嵌入式构建

```cmake
# 使用 Ninja 构建系统（比 Make 快得多）
# 配置时指定：
cmake -S . -B build -G Ninja -DCMAKE_TOOLCHAIN_FILE=arm-gcc.cmake

# 构建时（会自动检测并调用 ninja）：
cmake --build build --parallel
```

### 9.5 预编译头文件（PCH）加速

```cmake
# 适用于 AUTOSAR 项目中大量共用头文件的场景
target_precompile_headers(app PRIVATE
    <Rte.h>
    <Std_Types.h>
    <Compiler.h>
)
```

---

## 📌 十、从 Makefile 迁移到 CMake 对照表

| 概念 | Makefile | CMake |
|:---|:---|:---|
| 变量 | `VAR = value` | `set(VAR value)` |
| 引用变量 | `$(VAR)` 或 `${VAR}` | `${VAR}` |
| 条件 | `ifeq ($(VAR),val)` | `if(VAR STREQUAL "val")` |
| 目标 | `target: deps\n\tcommand` | `add_executable(target src)` |
| 静态库 | `ar rcs libfoo.a foo.o` | `add_library(foo STATIC foo.c)` |
| 头文件路径 | `-Ipath` | `target_include_directories(tgt PRIVATE path)` |
| 宏定义 | `-DNAME=val` | `target_compile_definitions(tgt PRIVATE NAME=val)` |
| 链接库 | `-lfoo` | `target_link_libraries(tgt PRIVATE foo)` |
| 自定义命令 | 写在 target recipe 中 | `add_custom_command(TARGET POST_BUILD COMMAND ...)` |
| 目标依赖 | `target: dep1 dep2` | `add_dependencies(target dep1 dep2)` |
| 模式匹配 | `%.o: %.c` | 不需要，CMake 自动处理 |
| 构建类型 | `make DEBUG=1` | `-DCMAKE_BUILD_TYPE=Debug` |
| 跨平台 | 手动处理 | 工具链文件 + 生成器表达式 |

---

## 📌 总结

CMake 在嵌入式开发（尤其是 Bootloader + AUTOSAR 场景）中的核心优势：

1. **工具链文件机制** —— 一套代码，任意交叉编译器，一条命令切换
2. **现代目标模型** —— PUBLIC/PRIVATE/INTERFACE 精准管理依赖传导
3. **生成器表达式** —— 优雅处理不同平台/编译器的条件编译
4. **链接脚本模板化** —— `configure_file()` 让内存布局参数集中可控
5. **模块化** —— `add_subdirectory()` 天然适配 AUTOSAR 的多层架构
6. **构建后动作** —— `add_custom_command(TARGET POST_BUILD)` 自动完成 bin/hex/size/sign

**记住三个核心理念：**

> 1. **目标（Target）**是一切 —— 不是变量，不是目录
> 2. **生成器表达式**是"可移植的条件编译" —— 不要写死的编译器判断
> 3. **工具链文件**是嵌入式 CMake 的第一步 —— 先配好它，再写逻辑

---

*参考命令：查看当前 CMake 版本* `cmake --version` *，查看 cmake 帮助* `cmake --help` *
