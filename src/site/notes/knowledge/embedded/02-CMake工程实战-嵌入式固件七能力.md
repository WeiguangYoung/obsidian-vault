---
{"dg-publish":true,"permalink":"/knowledge/embedded/02-CMake工程实战-嵌入式固件七能力/","title":"CMake 工程实战：嵌入式固件七能力","tags":["CMake","构建系统","交叉编译","链接脚本","嵌入式","toolchain","闭源库","实战"],"dg-note-properties":{"title":"CMake 工程实战：嵌入式固件七能力","date":"2026-11-03","updated":"2026-11-15","tags":["CMake","构建系统","交叉编译","链接脚本","嵌入式","toolchain","闭源库","实战"]}}
---


# CMake 工程实战：嵌入式固件七能力

> 以一个**可跑通的完整工程**为主线，把 CMake 在嵌入式里的 7 个核心能力逐个落到实处：
> **basics → static-lib → cross-compile → linker-script → post-build → imported-lib（闭源 .a + .h）→ advanced**。
> 前八节是工程实战（带完整代码），第九节是语法参考（目标模型、作用域、生成器表达式、陷阱速查）。

返回 [[首页\|🏠 首页]] · 相关：[[knowledge/embedded/01-嵌入式软件构建流程\|嵌入式软件构建流程]] · [[knowledge/embedded/cmake-examples/index\|📘 cmake-examples 逐例精读]] · [[knowledge/devops/practices/06-构建系统实战-Bazel\|Bazel 构建改造实战]]

---


### 📌 零、实训工程总览

### 0.1 目标形态

一个典型的**嵌入式固件工程**，但完全能在 PC 上先跑通（宿主机验证逻辑），再切交叉编译器上板：

```
┌──────────────────────────────────────────────────────┐
│  app           可执行文件（业务层）                     │
│   └── link: libcore.a + libthirdparty.a（闭源）        │
├──────────────────────────────────────────────────────┤
│  core          自研静态库（算法/协议）                  │
├──────────────────────────────────────────────────────┤
│  vendor        FATFS 模拟闭源库（只有 .a + .h）         │
├──────────────────────────────────────────────────────┤
│  构建配套      工具链文件 / 链接脚本 / 版本头生成 / 后处理  │
└──────────────────────────────────────────────────────┘
```

### 0.2 目录结构

```
cmake-workshop/
├── CMakeLists.txt                  # 顶层
├── cmake/
│   ├── toolchain-arm-none-eabi.cmake
│   ├── stm32f407.ld                # 链接脚本（正式）
│   └── host.ld                     # 链接脚本（宿主机演练用，可选）
├── app/
│   ├── CMakeLists.txt
│   ├── main.c
│   └── version.h.in                # 版本头模板
├── core/
│   ├── CMakeLists.txt
│   ├── libinc/crc16.h
│   └── libsrc/crc16.c
└── vendor/
    ├── CMakeLists.txt              # 封装闭源库为 imported target
    ├── inc/fatfs_shim.h            # 第三方头文件
    └── lib/libfatfs_shim.a         # 第三方预编译静态库
```

### 0.3 一次性验证命令

```bash
# 宿主机演练（不需要交叉编译器）
cmake -S . -B build -DCMAKE_BUILD_TYPE=Release
cmake --build build -j

# 交叉编译上板
cmake -S . -B build-arm \
      -DCMAKE_TOOLCHAIN_FILE=cmake/toolchain-arm-none-eabi.cmake \
      -DCMAKE_BUILD_TYPE=Release
cmake --build build-arm -j
```

---

### 📌 一、basics：最小可构建骨架

### 1.1 顶层 `CMakeLists.txt`

```cmake
cmake_minimum_required(VERSION 3.20)

# project() 必须在设置编译器相关变量之前调用
project(CmakeWorkshop
        VERSION 1.2.0
        DESCRIPTION "CMake 实战演练工程"
        LANGUAGES C)

# ---- 全局策略：现代 CMake 行为 ----
set(CMAKE_C_STANDARD 11)
set(CMAKE_C_STANDARD_REQUIRED ON)
set(CMAKE_C_EXTENSIONS OFF)          # 禁用 gnu 扩展，纯 c11

# ---- 统一输出目录（可选，方便打包取产物）----
set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/bin)
set(CMAKE_ARCHIVE_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/lib)

# ---- 默认构建类型（交叉编译时 CMAKE_BUILD_TYPE 默认是空的！）----
if(NOT CMAKE_BUILD_TYPE AND NOT CMAKE_CONFIGURATION_TYPES)
    set(CMAKE_BUILD_TYPE Release CACHE STRING "Build type" FORCE)
endif()

# ---- 公共编译告警 ----
add_compile_options(-Wall -Wextra -Wshadow -Wconversion)

# ---- 子目录 ----
add_subdirectory(core)
add_subdirectory(vendor)
add_subdirectory(app)
```

### 1.2 `app/CMakeLists.txt` + `app/main.c`

```cmake
# 生成版本头（见第六节详细讲）
configure_file(version.h.in ${CMAKE_CURRENT_BINARY_DIR}/version.h @ONLY)

add_executable(app
    main.c
)

target_include_directories(app PRIVATE
    ${CMAKE_CURRENT_BINARY_DIR}      # version.h 所在
)

target_link_libraries(app PRIVATE
    core
    fatfs_shim
)
```

```c
/* app/main.c */
#include <stdio.h>
#include "version.h"        /* 构建时生成 */
#include "crc16.h"          /* core 导出 */
#include "fatfs_shim.h"     /* vendor 导出 */

int main(void)
{
    const char payload[] = "cmake workshop";
    uint16_t crc = crc16_ccitt((const uint8_t *)payload, sizeof(payload) - 1);

    printf("fw=%s ver=%d.%d.%d\n",
           FW_NAME, FW_VER_MAJOR, FW_VER_MINOR, FW_VER_PATCH);
    printf("crc16 = 0x%04X\n", crc);
    printf("fatfs api version = %d\n", fatfs_api_version());

    return 0;
}
```

**要点：**
- `project()` 一定要在最前面（决定编译器检测时机）
- 编译类型必须显式兜底，否则交叉编译下 `-O` 全空
- 不要在根目录用 `include_directories()` / `link_directories()`，全部走 **target_* 系列**

---

### 📌 二、static-lib：自研静态库的正确姿势

### 2.1 `core/CMakeLists.txt`

```cmake
add_library(core STATIC
    libsrc/crc16.c
)

# PUBLIC：链接 core 的人自动拿到 core/libinc 头文件路径
target_include_directories(core PUBLIC
    ${CMAKE_CURRENT_SOURCE_DIR}/libinc
)

# PUBLIC：核心编译约束传导下去（例如要求 c11、特定宏）
target_compile_definitions(core PUBLIC
    CORE_BUILD_VERSION=2
)

# PRIVATE：只在编 core 时生效，外部看不见
target_compile_options(core PRIVATE
    -ffunction-sections -fdata-sections
)

# 位置无关：若将来 core 要被做成静态库塞进共享库，需要 PIC
set_target_properties(core PROPERTIES
    POSITION_INDEPENDENT_CODE ON
    OUTPUT_NAME "core"
)
```

### 2.2 头/源文件

```c
/* core/libinc/crc16.h */
#ifndef CRC16_H
#define CRC16_H

#include <stddef.h>
#include <stdint.h>

#ifdef __cplusplus
extern "C" {
#endif

/**
 * CRC-16/CCITT-FALSE
 * @param data 缓冲区
 * @param len  长度
 * @return 16 位校验值
 */
uint16_t crc16_ccitt(const uint8_t *data, size_t len);

#ifdef __cplusplus
}
#endif
#endif /* CRC16_H */
```

```c
/* core/libsrc/crc16.c */
#include "crc16.h"

uint16_t crc16_ccitt(const uint8_t *data, size_t len)
{
    uint16_t crc = 0xFFFFu;
    for (size_t i = 0; i < len; ++i) {
        crc ^= (uint16_t)data[i] << 8;
        for (int b = 0; b < 8; ++b) {
            crc = (crc & 0x8000u) ? (uint16_t)((crc << 1) ^ 0x1021u)
                                  : (uint16_t)(crc << 1);
        }
    }
    return crc;
}
```

**静态库三板斧：**

| 写法 | 效果 |
|:---|:---|
| `add_library(core STATIC ...)` | 产出 `libcore.a`，链接期才解符号 |
| `target_include_directories(core PUBLIC ...)` | 使用者免写 `-I` |
| `POSITION_INDEPENDENT_CODE ON` | 允许被嵌入共享库/某些 bootloader 场景 |

**常踩坑：** 静态库**不检查未定义符号**。链接顺序错了（`app` 在 `core` 之前）只会在最终链接时炸 `undefined reference`，所以务必用 `target_link_libraries(app PRIVATE core)` 让 CMake 自动排序。

---

### 📌 三、cross-compile：工具链文件 + 交叉编译树

### 3.1 `cmake/toolchain-arm-none-eabi.cmake`

```cmake
# ============ 必须在 project() 之前被加载 ============
set(CMAKE_SYSTEM_NAME      Generic)
set(CMAKE_SYSTEM_PROCESSOR arm)

# ---- 工具链前缀（改这里即换编译器）----
set(TOOLCHAIN_PREFIX arm-none-eabi-)

find_program(CMAKE_C_COMPILER   NAMES ${TOOLCHAIN_PREFIX}gcc REQUIRED)
find_program(CMAKE_ASM_COMPILER NAMES ${TOOLCHAIN_PREFIX}gcc REQUIRED)
find_program(CMAKE_OBJCOPY      NAMES ${TOOLCHAIN_PREFIX}objcopy)
find_program(CMAKE_SIZE         NAMES ${TOOLCHAIN_PREFIX}size)
find_program(CMAKE_AR           NAMES ${TOOLCHAIN_PREFIX}ar)
find_program(CMAKE_RANLIB       NAMES ${TOOLCHAIN_PREFIX}ranlib)

# ---- 交叉编译时禁止编译器自检试运行 ----
set(CMAKE_C_COMPILER_WORKS 1)
set(CMAKE_ASM_COMPILER_WORKS 1)

# ---- 目标架构 ----
set(MCU_FLAGS "-mcpu=cortex-m4 -mthumb -mfpu=fpv4-sp-d16 -mfloat-abi=hard")

set(CMAKE_C_FLAGS_INIT   "${MCU_FLAGS} -fno-common -fno-builtin")
set(CMAKE_ASM_FLAGS_INIT "${MCU_FLAGS}")
set(CMAKE_EXE_LINKER_FLAGS_INIT "${MCU_FLAGS} -Wl,--gc-sections --specs=nano.specs -u _printf_float")

# ---- 只在本工具链 sysroot 里找库，不去污染宿主机 ----
set(CMAKE_FIND_ROOT_PATH_MODE_PROGRAM NEVER)
set(CMAKE_FIND_ROOT_PATH_MODE_LIBRARY ONLY)
set(CMAKE_FIND_ROOT_PATH_MODE_INCLUDE ONLY)
set(CMAKE_FIND_ROOT_PATH_MODE_PACKAGE ONLY)
```

### 3.2 顶层文件里的条件分支

```cmake
# 顶层 CMakeLists.txt 追加
if(CMAKE_CROSSCOMPILING)
    message(STATUS "交叉编译模式: ${CMAKE_SYSTEM_PROCESSOR}, CC=${CMAKE_C_COMPILER}")

    # 嵌入式默认开 bin/hex 后处理
    set(ENABLE_POSTBUILD ON CACHE BOOL "Generate bin/hex after build")

    # 必须关闭宿主机可执行文件探测
    set(CMAKE_CROSSCOMPILING_EMULATOR "" CACHE STRING "")
else()
    message(STATUS "宿主机模式: ${CMAKE_SYSTEM_NAME}")
    set(ENABLE_POSTBUILD OFF CACHE BOOL "Generate bin/hex after build")
endif()
```

**双构建树并存：**

```
build/          ← 宿主机 Release（跑单元测试 / 逻辑验证）
build-arm/      ← 交叉 Release（出固件）
```

两棵树共享同一份源码，**绝不共用 build 目录**（`CMakeCache.txt` 里的编译器路径会锁死）。

---

### 📌 四、linker-script：链接脚本模板化 + 内存布局可配置

### 4.1 链接脚本模板 `cmake/stm32f407.ld.in`

```ld
/* 由 CMake configure_file() 生成，请勿直接修改 build 目录产物 */
ENTRY(Reset_Handler)

MEMORY
{
    FLASH (rx)  : ORIGIN = @FLASH_ORIGIN@, LENGTH = @FLASH_LENGTH@
    RAM   (rwx) : ORIGIN = @RAM_ORIGIN@,   LENGTH = @RAM_LENGTH@
}

_estack = ORIGIN(RAM) + LENGTH(RAM);

SECTIONS
{
    .isr_vector : {
        KEEP(*(.isr_vector))
    } > FLASH

    .text : {
        *(.text*)
        *(.rodata*)
        . = ALIGN(4);
        _etext = .;
    } > FLASH

    .data : AT(_etext) {
        _sdata = .;
        *(.data*)
        . = ALIGN(4);
        _edata = .;
    } > RAM

    .bss (NOLOAD) : {
        _sbss = .;
        *(.bss*)
        *(COMMON)
        . = ALIGN(4);
        _ebss = .;
    } > RAM

    /DISCARD/ : { *(.ARM.exidx*) *(.comment) }
}
```

### 4.2 顶层配置：把内存参数变成 cache 变量

```cmake
# ---- 内存布局（可在命令行覆盖：-DFLASH_LENGTH=1024K）----
set(FLASH_ORIGIN  0x08000000 CACHE STRING "Flash base address")
set(FLASH_LENGTH  512K       CACHE STRING "Flash size")
set(RAM_ORIGIN    0x20000000 CACHE STRING "RAM base address")
set(RAM_LENGTH    128K       CACHE STRING "RAM size")

if(CMAKE_CROSSCOMPILING)
    set(LD_TEMPLATE ${CMAKE_SOURCE_DIR}/cmake/stm32f407.ld.in)
    set(LD_OUTPUT   ${CMAKE_BINARY_DIR}/stm32f407.ld)

    configure_file(${LD_TEMPLATE} ${LD_OUTPUT} @ONLY)
    message(STATUS "链接脚本已生成: ${LD_OUTPUT}")

    # 全局生效（也可只对 app 生效，见下）
    add_link_options(-T ${LD_OUTPUT} -Wl,-Map=${CMAKE_BINARY_DIR}/app.map)
endif()
```

**或者只对某个目标生效（更推荐，多目标芯片场景必用）：**

```cmake
target_link_options(app PRIVATE
    -T ${LD_OUTPUT}
    -Wl,-Map=$<TARGET_FILE_DIR:app>/app.map
    -Wl,--print-memory-usage
)
```

### 4.3 Bootloader + APP 双镜像布局

> 双镜像的完整工程（两套目录、两份 CMakeLists、跳转与验签）见 [[knowledge/embedded/01-嵌入式软件构建流程\|01-嵌入式软件构建流程]] 第九章。本节只留最容易踩的变量命名约定。

一个 `stm32f407.ld.in` 可被 `configure_file` 复用两次，**但每次必须换一套语义化的变量名**，不要用同一个 `FLASH_ORIGIN` 反复覆盖 —— 覆盖顺序一变，两份镜像的地址就会静默互换，现象是 APP 烧进去不跑、BL 却能启动。

```cmake
# BL: 0x08000000~0x0800BFFF（48K）  APP: 0x0800C000 起（464K）
set(BL_FLASH_ORIGIN   0x08000000)
set(BL_FLASH_LENGTH   48K)
set(APP_FLASH_ORIGIN  0x0800C000)
set(APP_FLASH_LENGTH  464K)

set(FLASH_ORIGIN ${BL_FLASH_ORIGIN})
set(FLASH_LENGTH ${BL_FLASH_LENGTH})
configure_file(${CMAKE_SOURCE_DIR}/cmake/stm32f407.ld.in
               ${CMAKE_BINARY_DIR}/bootloader.ld @ONLY)

set(FLASH_ORIGIN ${APP_FLASH_ORIGIN})
set(FLASH_LENGTH ${APP_FLASH_LENGTH})
configure_file(${CMAKE_SOURCE_DIR}/cmake/stm32f407.ld.in
               ${CMAKE_BINARY_DIR}/application.ld @ONLY)
```

> 工程里更稳的写法是：模板用 `@BL_FLASH_ORIGIN@`/`@APP_FLASH_ORIGIN@` 两套独立占位符，一模板两份文件各自对应，不靠 set 覆盖顺序。

---

### 📌 五、post-build：产物后处理（bin / hex / size / 版本戳 / 签名）

### 5.1 `app/CMakeLists.txt` 追加

> 基础的 `objcopy -O binary/-O ihex` + `size` 三连在 [[knowledge/embedded/01-嵌入式软件构建流程\|01]] 的 9.3 已给过。这里只强调两处工程上容易忽略的细节。

```cmake
if(ENABLE_POSTBUILD)
    add_custom_command(TARGET app POST_BUILD
        # 1) elf -> bin
        COMMAND ${CMAKE_OBJCOPY} -O binary
                $<TARGET_FILE:app> $<TARGET_FILE_DIR:app>/app.bin
        # 2) elf -> hex
        COMMAND ${CMAKE_OBJCOPY} -O ihex
                $<TARGET_FILE:app> $<TARGET_FILE_DIR:app>/app.hex
        # 3) 尺寸报告：-A -x 给段明细，-B 给 CI 易解析的一行
        COMMAND ${CMAKE_SIZE} -A -x $<TARGET_FILE:app>
        COMMAND ${CMAKE_SIZE} -B  $<TARGET_FILE:app>
        COMMENT "Post-building firmware artifacts"
        VERBATIM
    )
endif()
```

**两个细节**：

1. `if(ENABLE_POSTBUILD)` 这层判断必须保留 —— 宿主机演练（不用交叉工具链）时 `CMAKE_OBJCOPY` 为空，会直接报错。
2. `VERBATIM` 不要省。它保证参数里的路径、引号、分号原样传给命令，否则 `$<TARGET_FILE_DIR:app>` 展开出带空格路径时命令会被拆断。
3. 用 `$<TARGET_FILE:app>` 而不是硬编码 `./app`，切换构建目录和 generator 都不用改。

### 5.2 构建后自动拷贝到发布目录 + 版本戳

```cmake
# 版本戳文件：每次成功构建记录 git commit + 时间
add_custom_command(TARGET app POST_BUILD
    COMMAND ${CMAKE_COMMAND}
            -DOUT=$<TARGET_FILE_DIR:app>/version.txt
            -DGIT_HASH=${GIT_HASH}
            -DVER=${PROJECT_VERSION}
            -P ${CMAKE_SOURCE_DIR}/cmake/stamp.cmake
    VERBATIM
)
```

```cmake
# cmake/stamp.cmake
string(TIMESTAMP BUILD_TIME "%Y-%m-%d %H:%M:%S" UTC)
file(WRITE ${OUT} "version=${VER}\ngit=${GIT_HASH}\nbuilt=${BUILD_TIME}Z\n")
```

顶层取 git hash：

```cmake
execute_process(
    COMMAND git rev-parse --short HEAD
    WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}
    OUTPUT_VARIABLE GIT_HASH
    OUTPUT_STRIP_TRAILING_WHITESPACE
    ERROR_QUIET
)
if(NOT GIT_HASH)
    set(GIT_HASH "unknown")
endif()
```

**要点：**
- **POST_BUILD 不会改变产物依赖** → 不会导致每次全量重编；`add_custom_command(OUTPUT ...)` 才会
- 用 `add_custom_target(firmware ALL DEPENDS ...)` 可以把 bin/hex 纳入 `all` 目标
- 想包装成 CMake 内建目标（如 `make flash`）：

```cmake
add_custom_target(flash
    COMMAND openocd -f interface/stlink.cfg -f target/stm32f4x.cfg
                    -c "program $<TARGET_FILE:app> verify reset exit"
    DEPENDS app
    COMMENT "Flashing app via OpenOCD"
)
```

### 5.3 签名与验签（Secure Boot）

> 前面几步只做「格式化 + 记账」，**产物本身没有任何可信性保证**。
> 要防篡改，必须加签名。签名分三段，CMake 只能覆盖第一段：

| 阶段 | 由谁完成 | CMake 的角色 |
|:---|:---|:---|
| 构建后签名 | CMake POST_BUILD | 调签名工具，产出 `.sig` / 已签名镜像 |
| 启动时验签 | Bootloader 运行时 | **完全不参与**（只负责生成正确的镜像布局） |
| 发布侧校验 | CI / 制品库 | 触发 `sha256sum` + GPG，验签通过才允许发版 |

#### 5.3.1 签名工具封装（`cmake/sign.cmake`）

```cmake
# 用法：
#   cmake -DIN=<镜像> -DOUT=<镜像+尾签> -DKEY=<私钥> -DSIG=<.sig 输出> -P sign.cmake
# 私钥通过 -DSIGN_KEY=... 或环境变量注入，绝不进仓库

if(NOT EXISTS "${IN}")
    message(FATAL_ERROR "待签名镜像不存在: ${IN}")
endif()
if(NOT SIGN_KEY)
    message(FATAL_ERROR "未提供签名私钥（-DSIGN_KEY=/path/to/priv.pem）")
endif()

# 1) 计算 sha256 摘要
file(SHA256 "${IN}" IMG_HASH)
message(STATUS "image sha256 = ${IMG_HASH}")

# 2) 私钥签名（openssl dgst）
execute_process(
    COMMAND openssl dgst -sha256 -sign "${SIGN_KEY}" -out "${SIG}" "${IN}"
    RESULT_VARIABLE sign_rc
    OUTPUT_VARIABLE sign_out
    ERROR_VARIABLE  sign_err
)
if(NOT sign_rc EQUAL 0)
    message(FATAL_ERROR "签名失败: ${sign_err}")
endif()

# 3) 镜像 || 签名 拼接成可烧录的最终镜像（可选，视 BL 格式而定）
file(READ    "${IN}"  img_hex HEX)
file(READ    "${SIG}" sig_hex HEX)
# 前置一个 4 字节签名长度头，方便 BL 解析
string(LENGTH "${sig_hex}" sig_hex_len)
math(EXPR sig_bytes "${sig_hex_len} / 2")
math(EXPR sig_bytes_be "((${sig_bytes} & 0xFF) << 24) | ((${sig_bytes} & 0xFF00) << 8) | ((${sig_bytes} >> 8) & 0xFF00) | ((${sig_bytes} >> 24) & 0xFF)")

# 注意：纯 CMake 拼大文件效率低，工程里建议调用外部脚本
# 这里保持自包含，便于理解流程；实际用 srec_cat / python 脚本更稳
file(WRITE "${OUT}" "${img_hex}${sig_hex}")

# 4) 输出校验文件
file(WRITE "${OUT}.sha256" "${IMG_HASH}  ${OUT}\n")
message(STATUS "已生成: ${OUT} / ${SIG} / ${OUT}.sha256")
```

> 更稳妥的二进制拼接用外部工具（CMake 里照样一行调用）：
>
> ```bash
> # 镜像 + 签名 → 最终镜像（STM32 常用）
> srec_cat app.bin -binary app.sig -binary -offset <app_size> -o app_signed.bin -binary
> # 或 python 脚本
> python tools/append_sig.py app.bin app.sig app_signed.bin
> ```

#### 5.3.2 挂到 POST_BUILD

```cmake
# ---- 签名配置（不进仓库：用 -DSIGN_KEY=... 或 $ENV{SIGN_KEY}）----
set(SIGN_KEY "$ENV{SIGN_KEY}" CACHE FILEPATH "镜像签名私钥路径（勿入版本库）")
option(ENABLE_SIGN "Enable image signing" OFF)

if(ENABLE_SIGN AND ENABLE_POSTBUILD)
    if(NOT SIGN_KEY OR NOT EXISTS "${SIGN_KEY}")
        message(FATAL_ERROR "ENABLE_SIGN=ON 但 SIGN_KEY 无效: '${SIGN_KEY}'")
    endif()

    set(FW_DIR $<TARGET_FILE_DIR:app>)

    add_custom_command(TARGET app POST_BUILD
        # 1) 先出裸 bin
        COMMAND ${CMAKE_OBJCOPY} -O binary $<TARGET_FILE:app> ${FW_DIR}/app.bin
        # 2) 签名 + 拼接 + 生成 sha256
        COMMAND ${CMAKE_COMMAND}
                -DIN=${FW_DIR}/app.bin
                -DOUT=${FW_DIR}/app_signed.bin
                -DSIG=${FW_DIR}/app.sig
                -DSIGN_KEY=${SIGN_KEY}
                -P ${CMAKE_SOURCE_DIR}/cmake/sign.cmake
        # 3) 回环自检：用公钥验证刚签出的镜像，防止密钥/流程配错
        COMMAND openssl dgst -sha256 -verify ${CMAKE_SOURCE_DIR}/keys/pub.pem
                -signature ${FW_DIR}/app.sig ${FW_DIR}/app.bin
        COMMENT "Signing firmware + verify round-trip"
        VERBATIM
    )

    message(STATUS "镜像签名已启用，密钥: <hidden>")
endif()
```

**关键设计：**

1. **私钥只经 `SIGN_KEY` 传入**，`keys/priv.pem` 加入 `.gitignore`，CI 走 Secret
2. **签完立刻验一遍**（`openssl -verify`）→ 密钥配错、公钥不匹配当场暴露，而不是等第一次上板变砖
3. **签名只对 `.bin` 做**，不要对 `.elf` 签（含调试信息，会随构建路径变化）
4. **`.sig` 与镜像同目录产出**，BL 侧解析时按约定的尾部/头部格式读

#### 5.3.3 公私钥与 CI 流程

```bash
# 一次性生成（本地，私钥永不外传）
openssl genpkey -algorithm RSA -pkeyopt rsa_keygen_bits:2048 -out keys/priv.pem
openssl pkey -in keys/priv.pem -pubout -out keys/pub.pem

# .gitignore
keys/priv.pem
build*/
*.sig
```

```yaml
# CI 片段（GitHub Actions）
- name: Inject signing key
  run: |
    echo "${{ secrets.FW_SIGN_KEY }}" > /tmp/priv.pem
    chmod 600 /tmp/priv.pem
    export SIGN_KEY=/tmp/priv.pem
- name: Build & sign
  run: |
    cmake --preset arm-release -DENABLE_SIGN=ON -DSIGN_KEY=/tmp/priv.pem
    cmake --build --preset arm-release
- name: Gate on verification
  run: |
    openssl dgst -sha256 -verify keys/pub.pem \
      -signature build-arm/firmware/app.sig build-arm/firmware/app.bin
    sha256sum -c build-arm/firmware/app_signed.bin.sha256
- name: Cleanup
  if: always()
  run: shred -u /tmp/priv.pem
```

#### 5.3.4 Bootloader 侧验签（示意，非 CMake 范畴）

```c
/* 仅示意流程：BL 收到 APP 镜像后先验签，通过才跳转 */
bool app_verify(const uint8_t *img, size_t img_len,
                const uint8_t *sig, size_t sig_len)
{
    /* 1) 按约定切出镜像体与签名段 */
    /* 2) 用内置公钥做 RSA-2048/PSS 或 ECDSA 验签 */
    /* 3) 失败则停在 BL，不跳 APP，并置失败标志 */
    return crypto_verify(img, img_len, sig, sig_len, g_app_pubkey, sizeof(g_app_pubkey));
}
```

**验签链路总览：**

```
CMake POST_BUILD
   ├─ objcopy → app.bin
   ├─ openssl sign → app.sig
   ├─ 拼接 → app_signed.bin
   ├─ openssl verify（回环自检）
   └─ sha256sum → app_signed.bin.sha256
              │
              ▼
      烧录 / OTA 下载
              │
              ▼
     Bootloader → crypto_verify() → 通过才跳转 APP
```

**避坑：**

| 坑 | 后果 |
|:---|:---|
| 私钥提交进 git | 签名彻底失去意义，必须吊销重签 |
| 只签不验 | 签了等于没签，BL 必须实现 verify 分支 |
| 签 elf 而非 bin | 调试信息/构建路径变化导致签名不可复现 |
| 镜像头和签名长度不对齐 | 烧录后 BL 解析越界，随机重启 |
| 用开发密钥上产 | 所有设备共用私钥，泄露即全线失守 |

---

### 📌 六、imported-lib：闭源库（只有 .a + .h）集成

> 现实里最棘手的场景：供应商只给你 `libxxx.a` + `xxx.h`，没有源码、没有 CMake。
> 正确做法是 **IMPORTED target** —— 把“一个二进制 + 一组头文件”包装成一个和自研库平权的目标。

### 6.1 `vendor/inc/fatfs_shim.h`

```c
#ifndef FATFS_SHIM_H
#define FATFS_SHIM_H

#ifdef __cplusplus
extern "C" {
#endif

/** 返回第三方库 API 版本号 */
int fatfs_api_version(void);

/**
 * 模拟第三方接口：把 buffer 内容"写"到文件
 * @return 0 成功，负值失败
 */
int fatfs_write_file(const char *path, const void *buf, unsigned len);

#ifdef __cplusplus
}
#endif
#endif /* FATFS_SHIM_H */
```

### 6.2 `vendor/CMakeLists.txt`（方案 A：**推荐**）

```cmake
# ---------- 1. 定义 IMPORTED 目标 ----------
add_library(fatfs_shim STATIC IMPORTED GLOBAL)

# ---------- 2. 指定库文件位置（按架构/构建类型分开更好） ----------
set_target_properties(fatfs_shim PROPERTIES
    IMPORTED_LOCATION
        ${CMAKE_CURRENT_SOURCE_DIR}/lib/libfatfs_shim.a
    # 多配置生成器才需要：
    # IMPORTED_CONFIGURATIONS "RELEASE;DEBUG"
    # IMPORTED_LOCATION_RELEASE ${CMAKE_CURRENT_SOURCE_DIR}/lib/libfatfs_shim.a
    # IMPORTED_LOCATION_DEBUG   ${CMAKE_CURRENT_SOURCE_DIR}/lib/libfatfs_shim_d.a
    INTERFACE_INCLUDE_DIRECTORIES
        ${CMAKE_CURRENT_SOURCE_DIR}/inc
)

# ---------- 3. 传导第三方库要求的编译约束（关键！）----------
target_compile_definitions(fatfs_shim INTERFACE
    FATFS_SHIM_API=1
)
target_compile_options(fatfs_shim INTERFACE
    -Wno-unused-parameter        # 头文件里可能有你不喜欢的告警
)

# ---------- 4. 可选：链接它时必须带上的系统库 ----------
# target_link_libraries(fatfs_shim INTERFACE m)

# ---------- 5. 存在性校验：早失败好过链接期炸 ----------
if(NOT EXISTS $<TARGET_FILE:fatfs_shim>)
    message(FATAL_ERROR
        "闭源库缺失: ${CMAKE_CURRENT_SOURCE_DIR}/lib/libfatfs_shim.a")
endif()
```

### 6.3 方案 B：`find_library` 动态查找（跨平台 / 库位置不固定）

```cmake
find_library(FATFS_LIB
    NAMES fatfs_shim
    PATHS ${CMAKE_CURRENT_SOURCE_DIR}/lib
    NO_DEFAULT_PATH
    REQUIRED)

if(NOT FATFS_LIB)
    message(FATAL_ERROR "找不到 libfatfs_shim.a，请检查 vendor/lib/")
endif()

add_library(fatfs_shim STATIC IMPORTED GLOBAL)
set_target_properties(fatfs_shim PROPERTIES
    IMPORTED_LOCATION "${FATFS_LIB}"
    INTERFACE_INCLUDE_DIRECTORIES ${CMAKE_CURRENT_SOURCE_DIR}/inc
)
```

### 6.4 方案 C：预编译静态库以 `INTERFACE` 库形式导出（**要对外分发时用**）

```cmake
# 把"头文件 + 二进制"组合成一个可被 find_package 消费的包
add_library(fatfs::shim STATIC IMPORTED GLOBAL)
set_target_properties(fatfs::shim PROPERTIES
    IMPORTED_LOCATION "${FATFS_LIB}"
    INTERFACE_INCLUDE_DIRECTORIES ${CMAKE_CURRENT_SOURCE_DIR}/inc
)
```

### 6.5 避坑清单

| 坑 | 说明 |
|:---|:---|
| ❌ 用 `link_directories()` + `-lfatfs_shim` | 路径作用域全工程污染、符号顺序不可控 |
| ❌ 忘了 `INTERFACE_INCLUDE_DIRECTORIES` | 每个使用者都要手写 `-I`，等于没做封装 |
| ❌ 忽略 ABI 匹配 | 交叉编译器版本、`-mfloat-abi`、`newlib vs newlib-nano` 必须与闭源库一致 |
| ❌ 架构不匹配 | PC 上编译出的 `.a` 塞进 ARM 工程 → `file` 一看就露馅 |
| ✅ 打印校验 | `message(STATUS "vendor: ${FATFS_LIB}")` 让构建日志可追溯 |
| ✅ 版本记录 | 在 `vendor/inc/` 放 `LIB_VERSION.txt`，便于审计 |

**ABI 自检命令（CI 里加一条）：**

```bash
file vendor/lib/libfatfs_shim.a
# 期望: ELF 32-bit LSB relocatable, ARM, EABI5 version 1 (SYSV)
```

---

### 📌 七、advanced：进阶工程化

### 7.1 模块化：`cmake/` 目录抽公共函数

`cmake/firmware_helpers.cmake`：

```cmake
# 统一封装"出固件"的动作，任何可执行目标一行调用
function(firmware_targets target)
    cmake_parse_arguments(ARG "" "OUTPUT_DIR" "" ${ARGN})
    if(NOT ARG_OUTPUT_DIR)
        set(ARG_OUTPUT_DIR ${CMAKE_BINARY_DIR}/firmware)
    endif()

    add_custom_command(TARGET ${target} POST_BUILD
        COMMAND ${CMAKE_COMMAND} -E make_directory ${ARG_OUTPUT_DIR}
        COMMAND ${CMAKE_OBJCOPY} -O binary $<TARGET_FILE:${target}>
                ${ARG_OUTPUT_DIR}/${target}.bin
        COMMAND ${CMAKE_OBJCOPY} -O ihex   $<TARGET_FILE:${target}>
                ${ARG_OUTPUT_DIR}/${target}.hex
        COMMAND ${CMAKE_SIZE} -B $<TARGET_FILE:${target}>
        COMMENT "Generating firmware artifacts for ${target}"
        VERBATIM
    )
endfunction()
```

用法：

```cmake
include(${CMAKE_SOURCE_DIR}/cmake/firmware_helpers.cmake)
firmware_targets(app OUTPUT_DIR ${CMAKE_BINARY_DIR}/release)
```

### 7.2 生成器表达式：一套代码多配置/多芯片

> 语法与四种基本用法见 [[knowledge/embedded/01-嵌入式软件构建流程\|01]] 的 5.4。这里只补两个**嵌入式工程特有**、01 没讲的用法。

```cmake
target_compile_options(app PRIVATE
    # 交叉编译时才加的限制：目标端不需要异常和 RTTI
    # （主机侧跑单元测试时保留异常，gtest 需要）
    $<$<BOOL:${CMAKE_CROSSCOMPILING}>:-fno-exceptions -fno-rtti>

    # 按编译器屏蔽已知无害的告警，避免 -Werror 挡路
    $<$<STREQUAL:${CMAKE_C_COMPILER_ID},GCC>:-Wno-psabi>
)
```

**为什么这两个值得单独记**：

- `CMAKE_CROSSCOMPILING` 判断让「同一份 CMakeLists 既编固件又编测试」成为可能 —— 目标端关异常省 Flash，主机端开异常跑 gtest，不用维护两套工程。
- `-Wno-psabi` 是 ARM 上最常见的误报（GCC 升级后大量 `note: parameter passing for argument ... changed in GCC 7.1`），不加会把真正的告警淹掉。

### 7.3 目标级安装 + 导出包（库复用）

```cmake
# core/CMakeLists.txt 追加
install(TARGETS core
    EXPORT coreTargets
    ARCHIVE DESTINATION lib
    INCLUDES DESTINATION include
)
install(DIRECTORY libinc/ DESTINATION include)
install(EXPORT coreTargets
    FILE coreTargets.cmake
    NAMESPACE workshop::
    DESTINATION lib/cmake/core
)
```

生成 `coreConfig.cmake`：

```cmake
# cmake/coreConfig.cmake.in
@PACKAGE_INIT@
include("${CMAKE_CURRENT_LIST_DIR}/coreTargets.cmake")
check_required_components(core)
```

```cmake
include(CMakePackageConfigHelpers)
configure_package_config_file(
    ${CMAKE_SOURCE_DIR}/cmake/coreConfig.cmake.in
    ${CMAKE_BINARY_DIR}/coreConfig.cmake
    INSTALL_DESTINATION lib/cmake/core
)
install(FILES ${CMAKE_BINARY_DIR}/coreConfig.cmake
        DESTINATION lib/cmake/core)
```

下游使用：

```cmake
find_package(core REQUIRED)
target_link_libraries(app PRIVATE workshop::core)
```

### 7.4 测试：宿主机跑 CTTest + 交叉编译自动跳过

```cmake
# 顶层
option(BUILD_TESTING "Enable host-side tests" ON)
if(BUILD_TESTING AND NOT CMAKE_CROSSCOMPILING)
    enable_testing()
    add_subdirectory(tests)
endif()
```

```cmake
# tests/CMakeLists.txt
add_executable(test_crc16 test_crc16.c)
target_link_libraries(test_crc16 PRIVATE core)
add_test(NAME crc16 COMMAND test_crc16)
```

```bash
cmake --build build && ctest --test-dir build --output-on-failure
```

### 7.5 Presets：把一长串命令行参数固化

`CMakePresets.json`：

```json
{
  "version": 6,
  "configurePresets": [
    {
      "name": "host-release",
      "binaryDir": "${sourceDir}/build",
      "cacheVariables": { "CMAKE_BUILD_TYPE": "Release" }
    },
    {
      "name": "arm-release",
      "binaryDir": "${sourceDir}/build-arm",
      "toolchainFile": "${sourceDir}/cmake/toolchain-arm-none-eabi.cmake",
      "cacheVariables": {
        "CMAKE_BUILD_TYPE": "Release",
        "FLASH_LENGTH": "512K",
        "RAM_LENGTH": "128K"
      }
    }
  ],
  "buildPresets": [
    { "name": "host-release", "configurePreset": "host-release", "jobs": 8 },
    { "name": "arm-release",  "configurePreset": "arm-release",  "jobs": 8 }
  ]
}
```

```bash
cmake --preset arm-release
cmake --build --preset arm-release
```

### 7.6 依赖管理：三种路线的取舍

| 方式 | 适用 | 写法 |
|:---|:---|:---|
| `FetchContent` | 纯源码依赖、版本可控 | `FetchContent_Declare(...); FetchContent_MakeAvailable(...)` |
| `find_package` + vcpkg/Conan | 大型 C 生态、跨团队 | 外部包管理器，CMake 只负责消费 |
| IMPORTED target | **闭源二进制** | 第六节 |

```cmake
include(FetchContent)
FetchContent_Declare(
    unity
    GIT_REPOSITORY https://github.com/ThrowTheSwitch/Unity.git
    GIT_TAG        v2.6.0
)
FetchContent_MakeAvailable(unity)
```

### 7.7 构建性能

```cmake
# 用 Ninja 而非 Make（嵌入式工程默认推荐）
cmake --preset arm-release -G Ninja

# 编译期：ccache（工具链文件里设）
set(CMAKE_C_COMPILER_LAUNCHER ccache)

# 链接期：LTO（Release）
set(CMAKE_INTERPROCEDURAL_OPTIMIZATION_RELEASE ON)
```

---

### 📌 八、CI 集成片段

```yaml
# .github/workflows/firmware.yml（节选）
- name: Configure
  run: cmake --preset arm-release
- name: Build
  run: cmake --build --preset arm-release
- name: ABI check (闭源库架构自检)
  run: |
    file vendor/lib/libfatfs_shim.a | grep -q "ARM" \
      || { echo "vendor 库架构不匹配"; exit 1; }
- name: Artifacts
  uses: actions/upload-artifact@v4
  with:
    name: firmware
    path: |
      build-arm/firmware/*.bin
      build-arm/firmware/*.hex
      build-arm/firmware/version.txt
```

---

### 📌 九、语法参考：从 01 搬过来的补充

> 本节接收 [[knowledge/embedded/01-嵌入式软件构建流程\|01-嵌入式软件构建流程]] 重构时下沉的 CMake 语法内容。01 只保留「CMake 在构建链路里的位置」，语法细节全部集中到这里。

### 9.1 核心概念速览

| 概念 | 说明 | 示例 |
|:--|:--|:--|
| **target** | 构建目标（可执行/库） | `add_executable`、`add_library` |
| **property** | target 的属性 | `target_include_directories` |
| **generator** | 生成器（Make/Ninja/VS） | `-G Ninja` |
| **variable** | 普通变量 | `set(SOURCES main.c)` |
| **cache** | 缓存变量，可被用户覆盖 | `set(VAR "val" CACHE STRING "desc")` |

常用命令：

```bash
cmake -S . -B build                          # 配置
cmake -S . -B build -DCMAKE_BUILD_TYPE=Release
cmake -S . -B build -DCMAKE_INSTALL_PREFIX=/usr/local
cmake --build build --parallel $(nproc)      # 构建
cmake --install build --prefix /output/dir   # 安装
cmake --build build --target clean           # 清理
```

> **永远用 out-of-source 构建**（`-B build`）。在源码目录跑 `cmake .` 会生成 `CMakeCache.txt`/`CMakeFiles/` 污染源码树，且无法同时保留 Debug 和 Release。

### 9.2 定义目标的四种形态

`add_library` 在嵌入式里四种都要认识：

```cmake
# --- 可执行文件 ---
add_executable(app main.c startup.s)

# --- 静态库（嵌入式主力） ---
add_library(drivers STATIC
    uart.c
    spi.c
    gpio.c
)

# --- 动态库（嵌入式 Linux 才有意义） ---
add_library(protocol SHARED
    can.c
    lin.c
)

# --- 对象库：只编译不归档，多目标复用时省一次编译 ---
add_library(mcal_objs OBJECT
    adc.c
    timer.c
)
# 其他目标链接它时，自动获得编译属性和目标文件
target_link_libraries(app PRIVATE mcal_objs)

# --- INTERFACE：纯头文件库，不产生任何编译产物 ---
add_library(hal INTERFACE)
target_include_directories(hal INTERFACE hal/inc)
target_link_libraries(app PRIVATE hal)
```

**为什么 `ALIAS` 值得用**：

```cmake
add_library(drivers STATIC drivers.c)
add_library(Fw::drivers ALIAS drivers)

target_link_libraries(app PRIVATE Fw::drivers)   # 命名空间风格
```

读起来和 `find_package` 导出的 target 一致；日后把本地库换成外部包，只改 target 名，链接代码不动。别名的限制是**只读** —— 不能用于 `install()` / `add_custom_command` 的目标参数。

### 9.3 作用域传导：PRIVATE / PUBLIC / INTERFACE

现代 CMake 最精髓的设计。三个关键字决定「属性传给谁」：

```cmake
# PRIVATE  ：仅对自己有效，不传导给依赖者
# PUBLIC   ：对自己有效，也会传导给依赖者
# INTERFACE：仅传导给依赖者，自己不需要（纯头文件库）

add_library(core STATIC core.c)

target_include_directories(core PUBLIC
    ${CMAKE_CURRENT_SOURCE_DIR}/core/inc
)

target_link_libraries(core PUBLIC mcal_objs)

# app 链接 core 时，自动获得：
#   - core/inc 头文件路径（因为 PUBLIC）
#   - mcal_objs 的编译属性和头文件（因为 PUBLIC 传导）
target_link_libraries(app PRIVATE core)
```

| 关键字 | 自己编译需要 | 传给链接我的目标 |
|:--|:--:|:--:|
| `PRIVATE` | ✅ | ❌ |
| `PUBLIC` | ✅ | ✅ |
| `INTERFACE` | ❌ | ✅ |

**为什么不能随手写成 `PRIVATE`** —— 一个嵌入式里天天遇到的真场景：

```
drivers/include/drivers/uart.h    ← drivers 库的头文件
app/src/main.c                    ← #include "drivers/uart.h"
```

如果 `drivers` 的头文件目录写成 `PRIVATE`，`app` 编译时**找不到头文件** —— 因为 `main.c` 需要这个路径，但它自己没声明，只能靠 `drivers` 通过 `PUBLIC` 传递过来。

```
drivers ── PUBLIC include/ ──▶ app 自动获得 -Iinclude/
```

**`INTERFACE` 用在哪**：纯头文件库（如 AUTOSAR 的 `Std_Types.h`、`Compiler.h` 这类基础类型定义）—— 自己不编译任何东西，只负责把头文件路径传出去。

### 9.4 编译选项三兄弟

```cmake
# ❌ 全局设置：影响当前目录及所有子目录的全部 target，无法按目标区分
add_compile_options(-Wall -Werror -Wextra)
add_compile_definitions(-DUSE_HAL_DRIVER)
set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -Os")

# ✅ 推荐：目标级别
target_compile_options(app PRIVATE
    -Wall -Wextra
    -ffunction-sections -fdata-sections
    $<$<CONFIG:Debug>:-Og -g3>
    $<$<CONFIG:Release>:-Os -flto>
)

target_compile_definitions(app PRIVATE
    STM32F407xx
    USE_HAL_DRIVER
    $<$<CONFIG:Debug>:DEBUG_ENABLE>
)
```

| 函数 | 用途 | 生成的标志 |
|:--|:--|:--|
| `target_compile_definitions(t PRIVATE FOO=1)` | 宏定义 | `-DFOO=1` |
| `target_compile_options(t PRIVATE -Wall)` | 任意编译选项 | `-Wall` |
| `target_include_directories(t PRIVATE dir)` | 头文件路径 | `-Idir` |
| `target_link_options(t PRIVATE -T x.ld)` | 链接选项 | 传给链接器 |

### 9.5 生成器表达式四个基础用法

在**生成阶段**才求值，天然适合「不同构建类型/平台/编译器用不同参数」：

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

# 3. 编译器判定（换 IAR / ARMClang 时特别有用）
target_compile_definitions(app PRIVATE
    $<$<C_COMPILER_ID:GNU>:COMPILER_GCC>
    $<$<C_COMPILER_ID:ARMClang>:COMPILER_ARMCLANG>
)

# 4. 取别的 target 的属性
target_include_directories(app PRIVATE
    $<TARGET_PROPERTY:core,INTERFACE_INCLUDE_DIRECTORIES>
)
```

> **关键理解**：生成器表达式写的时候不会求值。`message()` 打印出来是一串 `$<...>`，这是正常的 —— 它在生成 Makefile/Ninja 文件时才展开成真实参数。

### 9.6 条件与循环

```cmake
# 条件分支
if(CMAKE_BUILD_TYPE STREQUAL "Debug")
    target_compile_options(app PRIVATE -Og -g3)
else()
    target_compile_options(app PRIVATE -O2)
endif()

# 可配置开关（cmake-gui 里能勾选）
option(BUILD_CAN "Enable CAN stack" ON)
if(BUILD_CAN)
    target_sources(com PRIVATE com/CanIf.c)
    target_compile_definitions(com PRIVATE CAN_STACK_ENABLED)
endif()

# 文件收集（⚠️ 慎用，见 9.7）
file(GLOB_RECURSE SOURCES src/*.c)
add_executable(app ${SOURCES})
```

### 9.7 CMake 陷阱速查

01 重构时这一整章下沉到这里，共 8 条：

**① 构建类型不生效**

```cmake
# 单配置生成器（Makefile/Ninja）才用 CMAKE_BUILD_TYPE
# 多配置生成器（VS/Xcode/Ninja Multi-Config）用 --config，上面那个完全无效

# ✅ 嵌入式推荐：显式覆盖各配置的标志
set(CMAKE_C_FLAGS_RELEASE "-Os -DNDEBUG -flto")
set(CMAKE_C_FLAGS_DEBUG   "-Og -g3 -DDEBUG")
```

> 不设构建类型时 CMake 默认**不加任何优化标志**，性能可能差一个数量级。这是「为什么我的程序这么慢」的最常见原因。

**② GLOB 不自动检测新文件**

```cmake
# ❌ file(GLOB) 在配置阶段求值，新增文件不会触发重新配置
file(GLOB APP_SOURCES src/*.c)

# ✅ 显式列出源文件
set(APP_SOURCES src/main.c src/task.c src/uart.c)

# 如果非要用 GLOB，至少在文件头注释提醒
# !! 新增文件后请执行 cmake -S . -B build 重新配置 !!
```

**③ 工具链文件与 project() 的顺序**

```cmake
# ❌ 错误：project() 已执行，编译器已定为主机 gcc
project(MyProject)
set(CMAKE_TOOLCHAIN_FILE arm-gcc.cmake)

# ✅ 正确：命令行传入，在 CMake 处理 project() 前生效
# cmake -DCMAKE_TOOLCHAIN_FILE=arm-gcc.cmake -S . -B build
```

程序出现 `cannot execute binary file: Exec format error` 基本就是这个原因。

**④ 不要 GLOB 递归生成代码目录**

```cmake
# ❌ 危险：生成目录可能很大，可能含临时文件、备份文件
file(GLOB_RECURSE RTE_SOURCES ${GENERATED_DIR}/rte/*.c)

# ✅ 明确列出，或用子目录 CMakeLists.txt
add_subdirectory(${GENERATED_DIR}/rte)
```

生成器常在目录里留 `.tmp`、`~` 备份、旧版本文件 —— 递归 GLOB 会把这些全编进去。

**⑤ 静态库链接顺序**

```cmake
# GCC 链接器单次从左到右扫描，顺序错误就 undefined reference
target_link_options(ecu_fw PRIVATE
    -Wl,--start-group
    $<TARGET_FILE:swc> $<TARGET_FILE:rte> $<TARGET_FILE:com> $<TARGET_FILE:mcal>
    -Wl,--end-group
)
```

`--start-group` 让链接器反复扫描直到符号全部解析，代价是链接稍慢，工程不大时无感知。

**⑥ out-of-source 与增量构建**

```bash
cmake -S . -B build                    # 标准用法
cmake -S . -B build -DCMAKE_BUILD_TYPE=Debug   # 切换类型无需清空
rm -rf build && cmake -S . -B build    # 完全重建才删
```

**⑦ ALIAS target 只读**

```cmake
add_library(Fw::drivers ALIAS drivers)
install(TARGETS Fw::drivers ...)   # ❌ 报错：别名不能用于 install
install(TARGETS drivers ...)       # ✅ 用原名
```

**⑧ 其他高频坑**

| 现象 | 原因 | 解决 |
|:--|:--|:--|
| 改了 `CMakeLists.txt` 但行为没变 | 配置缓存 | 删 `CMakeCache.txt` 或整个 build 目录 |
| 链接器报 `section .text will not fit` | Flash 超限 | 减小 `.ld` 长度或开 `-Os`/`--gc-sections` |
| `CMAKE_C_FLAGS` 加选项不生效 | 被 target 级选项覆盖 | 检查 `target_compile_options` 顺序 |
| 主机库被误链接进固件 | 缺 `FIND_ROOT_PATH_MODE_*` | 见第三节工具链文件 |

---

### 📌 十、七节能力对照总表

| 章节 | 核心 CMake 命令 | 落地价值 |
|:---|:---|:---|
| basics | `project()` / `add_executable()` / `add_subdirectory()` | 工程骨架，一套源码多方构建 |
| static-lib | `add_library(STATIC)` / `target_include_directories(PUBLIC)` | 自研模块化，依赖自动传导 |
| cross-compile | Toolchain File / `CMAKE_SYSTEM_NAME Generic` | 一条命令切宿主机↔目标板 |
| linker-script | `configure_file()` / `target_link_options(-T)` | 内存布局参数化，双镜像易维护 |
| post-build | `add_custom_command(TARGET POST_BUILD)` | bin/hex/size/版本戳/签名全自动 |
| imported-lib | `add_library(IMPORTED)` / `IMPORTED_LOCATION` | 闭源 `.a` 与自研库平权使用 |
| advanced | `install(EXPORT)` / `CMakePresets` / `FetchContent` | 可交付、可复用、可维护 |

---

### 📌 十一、开工检查清单

- [ ] `project()` 在第一条 `cmake_minimum_required` 之后、其他设置之前
- [ ] `CMAKE_BUILD_TYPE` 有兜底（交叉编译尤其）
- [ ] 所有 include / link / define 都走 `target_*`，无全局污染
- [ ] 工具链文件里设了 `CMAKE_FIND_ROOT_PATH_MODE_*`
- [ ] 链接脚本用 `.ld.in` 模板 + `configure_file()`，不硬编码地址
- [ ] POST_BUILD 只做后处理，不当依赖机制
- [ ] 签名私钥走 `-DSIGN_KEY=` / CI Secret，`keys/priv.pem` 已进 `.gitignore`
- [ ] 签名后立即 `openssl -verify` 回环自检，且 Bootloader 侧有 `crypto_verify()`
- [ ] 只对 `.bin` 签名，不对 `.elf` 签
- [ ] 闭源库用 IMPORTED target，且 CI 里有 ABI 校验
- [ ] 宿主机构建树和交叉构建树分目录
- [ ] `CMakePresets.json` 固定参数，README 里只写 `--preset`
- [ ] Release 开 LTO + ccache，CI 构建时间可观测

---

*配套练习：把本工程骨架做完后，尝试①加第二个芯片（FLASH 1M/RAM 192K）用 Preset 切换；②把 core 做成 `find_package` 可消费的包；③给 vendor 库加 `IMPORTED_LOCATION_DEBUG` 支持 Debug 版闭源库。*

---

### 📌 十二、与其他文档的分工

| 文档 | 定位 |
|:--|:--|
| **本文** | 一个完整工程的**逐能力落地**：交叉编译、链接脚本、后处理、闭源库集成、安装导出 |
| [[knowledge/embedded/01-嵌入式软件构建流程\|嵌入式软件构建流程]] | 构建链路原理与语法地基：编译器、Make、CMake 语义、链接脚本原理、AUTOSAR 分层 |
| [[knowledge/embedded/cmake-examples/index\|cmake-examples 逐例精读]] | 外部开源教程逐例阅读，打语法地基 |
| [[knowledge/devops/practices/06-构建系统实战-Bazel\|Bazel 构建改造实战]] | 同一工程迁到 Bazel 的对照实验（远程缓存、toolchain） |

> 学习顺序建议：**构建流程（原理）→ 本文（工程落地）→ Bazel（规模化对照）**。
