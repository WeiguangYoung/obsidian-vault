---
{"dg-publish":true,"permalink":"/knowledge/devops/practices/07-构建系统实战-CMake与Bazel/","title":"构建系统实战：CMake 与 Bazel","tags":["CMake","Bazel","构建系统","交叉编译","链接脚本","嵌入式","Make","远程缓存","toolchain","实战"],"dg-note-properties":{"title":"构建系统实战：CMake 与 Bazel","date":"2026-11-03","updated":"2026-11-06","tags":["CMake","Bazel","构建系统","交叉编译","链接脚本","嵌入式","Make","远程缓存","toolchain","实战"]}}
---


# 构建系统实战：CMake 与 Bazel

> 两条路线的完整实操：**CMake 是嵌入式工程的事实标准**（交叉编译、链接脚本、闭源库集成都能拿住），**Bazel 是规模化构建的未来**（可复现、远程缓存、多语言统一）。
> 第一部分给 CMake 的 7 个核心能力逐个落地，第二部分给一次从 Make/CMake 迁到 Bazel 的真实改造。

返回 [[knowledge/devops/practices/README\|🛠️ 实战演练]]

---

## 📑 本文结构

| 部分 | 内容 | 原文档 | 适用 |
|:--|:--|:--|:--|
| 第一部分 | CMake：basics → static-lib → cross-compile → linker-script → post-build → imported-lib → advanced | embedded/04 | 嵌入式固件工程 |
| 第二部分 | Bazel：MODULE.bazel/BUILD.bazel 改造、4 个坑、与 Makefile 对照、远程缓存、交叉编译 toolchain | practices/07 | 多仓库/大工程 |

> **先读哪个**：日常嵌入式项目先看第一部分（够用且成熟）；想解决「构建慢、缓存不共享、多语言混编」再上看第二部分。

---

## 第一部分 · CMake 工程实战演练（原 embedded/04）

> 本文以**一个可跑通的完整工程**为主线，把 CMake 的 7 个核心能力逐个落到实处：
> **basics → static-lib → cross-compile → linker-script → post-build → imported-lib（闭源 .a + .h）→ advanced**。
> 每一节都有可直接复制的 **核心代码实现**，最后一节给出进阶组织手法（模块化、安装导出、测试、包管理）。

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

```cmake
# Bootloader 占 0x08000000~0x0800BFFF（48K）
# APP 从 0x0800C000 起
set(BL_FLASH_LENGTH 48K)
set(APP_FLASH_ORIGIN 0x0800C000)
set(APP_FLASH_LENGTH "464K")   # 512K - 48K

# BL 用独立脚本
configure_file(${CMAKE_SOURCE_DIR}/cmake/stm32f407.ld.in
               ${CMAKE_BINARY_DIR}/bootloader.ld @ONLY)

set(FLASH_ORIGIN  ${APP_FLASH_ORIGIN})
set(FLASH_LENGTH  ${APP_FLASH_LENGTH})
configure_file(${CMAKE_SOURCE_DIR}/cmake/stm32f407.ld.in
               ${CMAKE_BINARY_DIR}/application.ld @ONLY)
```

> ⚠️ 双镜像工程记得 `-DAPPLICATION_FLASH_ORIGIN` 这类语义化命名，别让 `FLASH_ORIGIN` 一个变量被反复覆盖 —— 上面的简写仅演示 configure_file 能力，工程里建议给每个镜像一套独立变量名。

---

### 📌 五、post-build：产物后处理（bin / hex / size / 版本戳 / 签名）

### 5.1 `app/CMakeLists.txt` 追加

```cmake
# ============ 后处理：仅在交叉编译时启用 ============
if(ENABLE_POSTBUILD)
    add_custom_command(TARGET app POST_BUILD
        # 1) elf -> bin
        COMMAND ${CMAKE_OBJCOPY} -O binary
                $<TARGET_FILE:app> $<TARGET_FILE_DIR:app>/app.bin
        # 2) elf -> hex
        COMMAND ${CMAKE_OBJCOPY} -O ihex
                $<TARGET_FILE:app> $<TARGET_FILE_DIR:app>/app.hex
        # 3) 尺寸报告（输出到构建日志，方便 CI 抓取）
        COMMAND ${CMAKE_SIZE} -A -x $<TARGET_FILE:app>
        COMMAND ${CMAKE_SIZE} -B  $<TARGET_FILE:app>
        # 4) 打印产物列表
        COMMAND ${CMAKE_COMMAND} -E echo
                "== built: $<TARGET_FILE_DIR:app>/app.bin =="
        COMMENT "Post-building firmware artifacts"
        VERBATIM
    )
endif()
```

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

```cmake
target_compile_options(app PRIVATE
    $<$<CONFIG:Release>:-Os -DNDEBUG -flto>
    $<$<CONFIG:Debug>:-Og -g3 -DDEBUG>
    $<$<STREQUAL:${CMAKE_C_COMPILER_ID},GCC>:-Wno-psabi>
    $<$<BOOL:${CMAKE_CROSSCOMPILING}>:-fno-exceptions>
)
```

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

### 📌 九、七节能力对照总表

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

### 📌 十、开工检查清单

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

## 第二部分 · Bazel 构建改造实战（原 practices/07）

> 对象：hello-jenkins 工程的纯 C 部分（`UDS_ProtocolStack` + CUnit 单元测试），用宿主机 gcc 编译，不涉及交叉编译。
> 对照对象：工程现有 `Makefile` 的 `ut` 目标。所有命令与耗时均为真实环境实测。

### 一、为什么值得学 Bazel

| 痛点 | Make / CMake 的现状 | Bazel 的解法 |
|:---|:---|:---|
| 增量构建不准 | 靠 `.d` 依赖文件，漏一个头文件依赖就可能不重编 | 内容哈希 + 完整依赖图，改了就重编、没改秒过 |
| 缓存不可共享 | 无 | 本地 action cache，可扩展远程缓存（团队/CI 共享） |
| 构建结果不一致 | 依赖本地工具链与环境污染 | 沙箱隔离，工具链显式声明 |
| 多语言/多目标 | 各自为政（Make/CMake/Gradle） | 统一用 BUILD 文件描述 |
| 大规模仓库 | 每次都要扫描全量 | 只分析受影响的 target |

适用场景：**自动驾驶、机器人、大型 C++ 单仓**（地平线、它石这类岗位的典型技术栈）。

### 二、环境准备

```bash
# 官方脚本走 GCS 解析版本，国内常超时 → 直接下固定版本二进制
curl -sL -o bazel-9.3.1 \
  https://github.com/bazelbuild/bazel/releases/download/9.3.1/bazel-9.3.1-linux-x86_64
chmod +x bazel-9.3.1 && ./bazel-9.3.1 --version   # bazel 9.3.1

# 单元测试依赖 CUnit
sudo apt-get install -y libcunit1-dev
```

> 这一步就踩了一个坑：`bazelisk` 会去访问 `www.googleapis.com` 列版本，国内 30 秒超时后报 `could not resolve the version 'latest'`。**直接下带版本号的二进制最稳。**

### 三、工程改造（MODULE.bazel + BUILD.bazel）

### 3.1 MODULE.bazel

```python
module(
    name = "uds_bootloader_lab",
    version = "0.1.0",
)

# Bazel 9 起原生 cc 规则需显式引入 rules_cc
bazel_dep(name = "rules_cc", version = "0.2.8")
```

### 3.2 BUILD.bazel

```python
load("@rules_cc//cc:cc_library.bzl", "cc_library")
load("@rules_cc//cc:cc_test.bzl", "cc_test")

package(default_visibility = ["//visibility:public"])

# 头文件库：让项目头以裸文件名出现在 include 路径上
cc_library(
    name = "porting_headers",
    hdrs = [
        "UDS_PortingFiles/common_types.h",
        "UDS_PortingFiles/toolchain.h",
    ],
    strip_include_prefix = "UDS_PortingFiles",
)

# 静态库：自实现的 libc 子集
cc_library(
    name = "autolibc",
    srcs = ["UDS_ProtocolStack/autolibc.c"],
    hdrs = ["UDS_ProtocolStack/autolibc.h"],
    includes = ["UDS_ProtocolStack"],
    deps = [":porting_headers"],
    copts = ["-std=c99", "-Wno-pointer-to-int-cast"],
)

# 单元测试
cc_test(
    name = "ut_run",
    srcs = ["test/test_main.c", "test/test_autolibc.c"],
    deps = [":autolibc"],
    includes = ["UDS_ProtocolStack"],
    copts = ["-std=c99", "-Wno-pointer-to-int-cast"],
    linkopts = ["-lcunit"],
)
```

### 四、踩坑清单（4 条）

### 坑 1 · Bazel 9 把 cc_* 规则移出 builtins

```
Error: This rule has been removed from Bazel. Please add a `load()` statement for it.
```

Bazel 8 起逐步把原生规则迁移到 `rules_cc` 外部仓库，9.x 已强制。解决：
- `MODULE.bazel` 加 `bazel_dep(name = "rules_cc", ...)`
- BUILD 文件顶部 `load("@rules_cc//cc:cc_library.bzl", "cc_library")`

> 认知：这是 Bazel 的一个重要方向——**原生规则外置化**，升级大版本时这类报错会反复出现，看到 `removed from Bazel` 先想 load。

### 坑 2 · `includes` 不等于 `-I`（最容易卡住的一条）

报错发生在编译阶段，但文件明明就在那儿：

```
UDS_ProtocolStack/autolibc.c:92:10: fatal error: common_types.h: No such file or directory
```

**根因**：`includes = [...]` 只是把路径拼进编译命令行（生成 `-I`），**并不会把头文件登记进 target 的依赖图**。Bazel 的沙箱只把「声明过的输入」映射进构建目录，没声明的头文件在沙箱里根本不存在。

**三种正确姿势**：

| 方式 | 写法 | 适用 |
|:---|:---|:---|
| 头文件库 + `strip_include_prefix` | 独立 `cc_library`，头文件放 `hdrs` | **推荐**，头文件归属清晰 |
| `include_prefix` | 保留子目录前缀 | 需要按 `foo/bar.h` 引用时 |
| `hdrs` 直接挂在同一个 target | `hdrs = [...]` | 头文件少、只被本 target 用 |

本项目用的是第一种。

### 坑 3 · 沙箱报错信息不完整

`--verbose_failures` 才能看到完整的 gcc 命令行，是排查 include 类问题的关键手段。用 `--spawn_strategy=local` 绕开沙箱会让问题「变形」（报成另一个头文件找不到），反而不利于定位。

> 认知：**先用 `--verbose_failures` 看真实命令行，别急着关沙箱。** 沙箱正是 Bazel 保证可复现的核心，不是障碍。

### 坑 4 · 系统库要显式声明

CUnit 这类系统库需要 `linkopts = ["-lcunit"]`。Bazel 不会自动带上系统搜索路径，好处是**链接依赖一目了然**，坏处是从传统构建迁移时要补一轮。

### 五、与 Makefile 的对照

### 5.1 写法对照

| 维度 | Makefile | Bazel |
|:---|:---|:---|
| 源文件收集 | `$(wildcard *.c)` 自动扫 | **逐个写死** |
| 头文件依赖 | 隐式，靠 `-I` + `.d` 文件 | **显式 `hdrs`，进入依赖图** |
| 目标声明 | 规则 + 变量 | `cc_library` / `cc_test` 等规则 |
| 链接系统库 | `-lcunit` 写在 `UT_LIBS` | `linkopts` |
| 产物 | `build/ut/ut_run` | `bazel-bin/ut_run` |
| 执行测试 | `./ut_run` | `bazel test //:ut_run`（自带结果汇总） |

### 5.2 实测耗时

| 场景 | 耗时 | 说明 |
|:---|:---|:---|
| 首次构建 `//:autolibc` | ~18s | 含 rules_cc 下载与工具链配置 |
| clean 后 `bazel test //:ut_run` | **1.03s** | 14 个 action |
| 重复构建 | **0.33s** | 缓存全命中 |
| 触碰头文件后构建 | **0.16s** | 无 action 执行（内容未变 → 哈希不变） |
| 单元测试执行 | PASSED | CUnit 全用例通过 |

> 注意「触碰头文件」那一条：`touch` 只改时间戳不改内容，Bazel 按内容哈希判定，所以判定为「无需重编」。Make 按时间戳判定，同样场景会重编——**这就是增量构建精度差异的直接体现**。

### 5.3 得与失

**得到**：
- 依赖关系精确到文件级，不会漏编
- 缓存可跨机器共享（接远程缓存后，同事/CI 直接命中）
- 构建结果可复现（沙箱 + 显式工具链）
- 测试与构建统一入口（`bazel test` 自带失败汇总）

**付出**：
- 新文件必须写进 BUILD，`wildcard` 的便利没有了
- 学习曲线陡：Bazel 有自己的概念体系（target、label、visibility、沙箱、action graph）
- 存量工程改造量大（本项目只改了纯 C 的一小块）

**结论**：**小工程/快速迭代用 Make；大规模、多人、多语言、需要共享缓存的工程才值 Bazel 的迁移成本。**

### 六、进阶能力（自定义 rule · select · 远程缓存）

前面三节是最小可用改造。这一节记录向「可上生产的 Bazel 工程」推进的三步。

### 6.1 自定义 rule：构建期生成版本头

原流水线里版本信息靠 `git describe` 脚本在 CI 阶段生成（对应 Jenkins/GitLab 的 `versioning` stage）。Bazel 里可以把它变成构建图中的一个正规 target。

**`gen_version.bzl`**（Starlark 自定义 rule）：

```python
def _gen_version_impl(ctx):
    out = ctx.actions.declare_file(ctx.attr.out)
    ctx.actions.write(
        output = out,
        content = """#ifndef CI_VERSION_H
#define CI_VERSION_H
#define CI_VERSION      "%s"
#define CI_BUILD_NUMBER "%s"
#endif
""" % (ctx.attr.version, ctx.attr.build_number),
    )
    return [DefaultInfo(files = depset([out]))]

gen_version = rule(
    implementation = _gen_version_impl,
    attrs = {
        "version": attr.string(mandatory = True),
        "build_number": attr.string(default = "local"),
        "out": attr.string(default = "ci_version.h"),
    },
)
```

**BUILD 里使用**：

```python
load("//:gen_version.bzl", "gen_version")

gen_version(
    name = "ci_version",
    version = "0.1.0-bazel-lab",
    build_number = "local",
)

cc_library(
    name = "version_header",
    hdrs = [":ci_version"],   # 生成物可直接当头文件用
)
```

产物实测：

```c
#ifndef CI_VERSION_H
#define CI_VERSION_H
#define CI_VERSION      "0.1.0-bazel-lab"
#define CI_BUILD_NUMBER "local"
#endif
```

**价值**：版本号从「CI 脚本的副产物」变成「构建图里有明确依赖的 target」。谁依赖它，依赖关系就写在 BUILD 里；哪天忘了生成，构建直接失败，不会像 shell 脚本那样静默产出空值。

### 6.2 `select()`：编译选项按配置分支

```python
config_setting(
    name = "release",
    values = {"compilation_mode": "opt"},
)

cc_library(
    name = "autolibc",
    # ...
    copts = ["-std=c99"] + select({
        ":release": ["-O2"],
        "//conditions:default": ["-O0", "-g"],
    }),
)
```

`aquery` 验证：

| 命令 | 实际编译选项 |
|:---|:---|
| `bazel aquery //:autolibc` | `-O0 -g` |
| `bazel aquery //:autolibc -c opt` | `-O2` |

**和 Make/CMake 的区别**：Makefile 靠 `ifeq` 拼字符串、CMake 靠 `CMAKE_BUILD_TYPE` 分支，本质都是**构建脚本里的条件判断**；Bazel 的 `select` 是**构建图级别的分支**，配置一变，受影响的 target 自动重新解析，不存在「变量没传进去」这类隐蔽错误。

### 6.3 远程缓存（最能体现 Bazel 价值的一项）

#### 架构

```
Bazel 客户端 ──HTTP──> bazel-remote（CAS + action cache）
                          │
                          └── 本地磁盘（zstd 压缩，按 LRU 淘汰）
```

#### 部署 bazel-remote

```bash
# 下载二进制（GitHub releases，版本需与 Bazel 兼容）
curl -sL -o bazel-remote \
  https://github.com/buchgr/bazel-remote/releases/download/v2.4.4/bazel-remote-2.4.4-linux-x86_64
chmod +x bazel-remote

# 启动：5GiB 磁盘缓存
./bazel-remote --dir=/home/user/bazel-remote/data --max_size=5 \
  --http_address=0.0.0.0:8091 --grpc_address=0.0.0.0:9092

# 验证
curl -s http://127.0.0.1:8091/status
```

#### 客户端启用

```bash
bazel build //:autolibc --remote_cache=http://<cache-host>:8091
```

#### 实测数据

| 场景 | 耗时 | 关键指标 |
|:---|:---|:---|
| 首次构建（action 全执行 + 上传缓存） | **1.07s** | 6 个 action 执行 |
| **全新工作区**（不同目录、本地缓存为空）+ 远程缓存 | **18.5s** | **6 个 remote cache hit，0 个 action 执行** |

> 第二次反而更慢，是因为**新工作区首次运行需要建立仓库映射、下载 rules_cc、解析工具链**——这 18 秒花在初始化上，编译动作一个没跑。真实团队场景里这部分开销会被反复利用的缓存摊薄。

#### 为什么大团队必须上远程缓存

| 场景 | 没有远程缓存 | 有远程缓存 |
|:---|:---|:---|
| CI 每个 job 都是新 Pod | 每次从零编 | 命中上次产物 |
| 新人 clone 仓库 | 全量编译（可能几十分钟） | 直接拉产物 |
| 多分支并行 | 每个分支各编一遍 | 公共依赖只编一次 |
| rebase 后重新构建 | 视改动范围重编 | 未变的 action 全命中 |

**核心认知**：Bazel 的真正价值不在「单机快」，而在**编译成本可以在团队内摊销**。一个人的构建结果，其他人直接复用——这是 Make/CMake 体系无法做到的（可以接 ccache，但只解决编译缓存，不解决测试结果和 action 图级复用）。

### 七、交叉编译 toolchain（做到哪、卡在哪）

前面全部是**宿主机编译**。这一节记录把整个 S32K144 固件交给 Bazel 交叉编译的尝试，**没有跑到最终成功**，但踩清了 Bazel 9 在 toolchain 这条链上的版本变化。

### 7.1 目标与路线

目标：`bazel build //:firmware_elf --platforms=//:s32k144` 产出 `firmware.elf`，等价于 Makefile 里 `CROSS_COMPILE=arm-none-eabi-` 那条路。

改造分成四层：

```
platform(//:s32k144)          目标平台：cortex-m4 + 裸机 OS
        │
toolchain(//toolchain)        注册：exec=x86_64-linux，target=cortex-m4
        │
cc_toolchain → config         参数：-mcpu/-mthumb/-mfpu、--specs=nano.specs
        │
cc_library × 4 → cc_binary    依赖图：sdk_headers / uds_stack / app / sdk_drivers
```

### 7.2 踩坑清单（Bazel 9 的 6 个版本变化）

| # | 报错 | 根因 | 解法 |
|:-:|:---|:---|:---|
| 1 | `constraint_value is not defined` | `constraint_value` 是 **BUILD 原生规则**，不能在 `.bzl` 里直接调用 | 直接定义在 BUILD 文件里（或写 `native.constraint_value`） |
| 2 | `provides = [CcToolchainConfigInfo]` → `got NoneType` | Bazel 9 不再把这个 Provider 暴露给 Starlark | 去掉 `provides`，靠 rule 返回的 `CcToolchainConfigInfo` 隐式提供 |
| 3 | `cc_toolchain has been removed from Bazel` | 和 `cc_library` 一样，`cc_toolchain` 规则也外置了 | `load("@rules_cc//cc/private/rules_impl:cc_toolchain.bzl", "cc_toolchain")` |
| 4 | `cc_common has no field create_cc_toolchain_config_info` | 该方法从 `cc_common` 移到 rules_cc 私有实现 | `load("@rules_cc//cc/private/toolchain_config:cc_toolchain_config_info.bzl", ...)` |
| 5 | `No matching toolchains found for types: @@bazel_tools//tools/cpp:toolchain_type` | 只写 `toolchain()` 不够，还要 `register_toolchains()` + 平台约束对得上 | `MODULE.bazel` 里 `register_toolchains("//toolchain:s32k144_toolchain")`，且 `exec/target_compatible_with` 填全 |
| 6 | 头文件找不到（反复） | `includes` 只加编译参数，不进依赖图；`hdrs` 没声明的头在沙箱里根本不存在 | `hdrs` 用 `glob` 收全，关键是 **`drivers/src/**/*.h` 不能漏**（`power_manager.h` 里用相对路径 `../src/...` 引它） |

**第 4 条最隐蔽**：官方文档仍在介绍 `cc_common.create_cc_toolchain_config_info()`，但 Bazel 9 已经移走。定位办法只有一个——去 `~/.cache/bazel/*/external/rules_cc+/` 里翻实际文件：

```bash
grep -rn "def create_cc_toolchain_config_info" \
  ~/.cache/bazel/_bazel_*/external/rules_cc+/
```

### 7.3 验证到的进度

**工具链确实生效了**——报错信息里的编译器已经变成：

```
/usr/bin/arm-none-eabi-gcc -MD -MF bazel-out/.../can_tp_cfg.d ...
```

而不是宿主机的 `/usr/bin/gcc`。说明 platform → toolchain → config 这条链是通的，`-mcpu=cortex-m4` 等参数也已经生效。

### 7.4 卡点演进：从系统头到归档（记录最终状态）

这一段记录了卡点逐步推进的过程，**最终没跑通**，但每一步的原因都有价值。

#### 阶段 A：绝对路径输入（已解决）

```
ERROR: absolute path inclusion(s) found in rule '//:uds_stack':
  '/usr/lib/gcc/arm-none-eabi/9.2.1/include/stdint.h'
  '/usr/lib/gcc/arm-none-eabi/9.2.1/include/stdbool.h'
  '/usr/lib/gcc/arm-none-eabi/9.2.1/include/stddef.h'
```

**根因**：Bazel 要求所有编译输入都是**声明的 target**。编译器自带的系统头不在工作区里，不能隐式带入沙箱。

**解法**：把 GNU ARM 的三个头目录 vendored 进仓库，用 `-nostdinc` + `-isystem` 走仓库内副本：

```python
SYSINCLUDE_COPTS = [
    "-nostdinc",
    "-isystem", "toolchain/sysinclude",          # gcc 自带（stdint/stdbool/stddef）
    "-isystem", "toolchain/sysinclude/include",  # newlib 主头（assert.h 等）
    "-isystem", "toolchain/sysinclude/newlib",   # newlib 配置头（newlib.h）
]
```

三个目录缺一不可，缺 `include` 报 `assert.h`，缺 `newlib` 报 `newlib.h`——**编译器内部是分三层找头的**，vendoring 时必须全部搬过来。

> 这里走的是「copts 硬塞」的下策。正规做法是把这些目录声明进 toolchain 的 builtin include（`cc_toolchain` 的 `compiler_files` + feature）。我试过写 feature，但 `flag_set(actions = ...)` 里的 `-isystem` 没有生效（`aquery` 里看不到），于是退回 copts。**这也算一个记录在案的问题。**

#### 阶段 B：feature 里的 flag 不生效（未解决）

`vendored_sysinclude` 这个 feature 写了 `enabled = True`，`aquery` 却查不到 `-nostdinc` 和 `-isystem`；而同结构、写成 `actions = all_compile_actions + all_link_actions` 的 `arm_cortex_m4` feature 却正常生效（`-mcpu=cortex-m4` 能看到）。

**推测原因**：`ACTION_NAMES.c_compile` 在 Bazel 9 里的实际字符串与 rules_cc 内部使用的 action 名不一致，导致 `flag_set.actions` 匹配不到具体 action，feature 静默失效。**这一条没有验证完毕。**

#### 阶段 C：归档失败（当前卡点）

编译全部通过后，卡在静态库归档：

```
/usr/bin/arm-none-eabi-ar: supported targets: elf32-littlearm elf32-littlearm-fdpic ...
```

`ar` 打印支持目标列表 = **它认为自己没收到正确参数**（`ar` 在参数不合法时的标准行为）。推测是 Bazel 9 传给 `ar` 的参数格式与 GNU ARM `ar` 的预期不一致，或 `cc_toolchain` 的 `ar_files` 未正确配置。

**为什么停在这里**：这属于 toolchain 参数格式的细节调试，与前面 6 个「版本变化」类问题不同——已经不再产出新认知，继续投入边际收益很低。

### 7.5 最终状态总结

| 环节 | 状态 | 说明 |
|:---|:---:|:---|
| 平台约束 + toolchain 注册 | ✅ | `--platforms=//:s32k144` 能找到工具链 |
| 编译器切换 | ✅ | 命令行确认为 `/usr/bin/arm-none-eabi-gcc` |
| CPU 参数（`-mcpu/-mthumb/-mfpu`） | ✅ | `aquery` 可见 |
| 目标平台宏定义 | ✅ | 5 个 `-D` 全部生效 |
| 系统头（三层 vendored） | ✅ | `aquery` 输入含 `sysinclude` 全部文件 |
| **C 源码编译** | ✅ | 编译阶段全部通过 |
| 汇编（`.S`） | ⬜ 未验证 | 卡在归档，未走到链接 |
| **静态库归档（`ar`）** | ❌ **卡点** | 参数格式不匹配 |
| 链接产出 ELF | ⬜ 未触及 | — |

**一句话**：**编译链路已经打通，死在了归档这一步。**

### 7.6 遗留问题与后续路径

| # | 遗留问题 | 原因 | 建议解法 |
|:-:|:---|:---|:---|
| 1 | `ar` 归档参数不匹配 | Bazel 9 传给 `ar` 的参数格式与 GNU ARM `ar` 不一致 | 用 `aquery` 抓实际命令行，与手工 `arm-none-eabi-ar rcs` 对比；必要时用 feature 显式覆盖 `cpp_link_static_library` 的 flag |
| 2 | feature 里 `-isystem` 不生效 | `flag_set.actions` 的 action 名匹配不上 | 改用 `--toolchain_resolution_debug` 或 `aquery --include_commandline` 排查；或直接用新式 rule-based toolchain |
| 3 | 用 copts 硬塞系统头（下策） | feature 走不通的临时绕过 | 解决 #2 后改回 feature |
| 4 | 汇编文件未验证 | 卡在归档 | 归档通了自然验证 |
| 5 | 链接脚本 `-T` 路径 | 未验证 | Bazel 里要用 `$(location ...)` + `data` 声明，已写好但未跑通 |

**如果要继续，推荐的下一步**：放弃 `cc_toolchain_config_info` 这套老 API，改用 **新式 rule-based toolchain**（`@rules_cc//cc/toolchains`）——它把工具路径、参数集合、include 目录都用 Starlark 声明式描述，能一次性绕开第 1、2、3 条遗留问题。

### 7.7 这一节的真实价值

**没有编出 ELF，但对 Bazel 的理解跨了一大步**：

1. 知道 `cc_library` 之于 Makefile 是「自动 vs 显式」的差别
2. 知道 toolchain 不是一句 `CROSS_COMPILE=` 能替代的——它要声明编译器路径、参数集合、系统头来源、exec/target 平台约束
3. 亲手踩了 **6 个 Bazel 9 的版本变化**，这比看文档有用得多
4. 理解了 Bazel 的「输入完整性」哲学：**沙箱里的一切都必须是声明过的**，这既是它的力量，也是迁移成本的主要来源
5. 学会了**如实记录「做到哪、卡在哪、为什么停」**——这比一个 hack 出来的「成功」更有工程价值

> 面试可以这么讲：「我尝试把嵌入式固件迁到 Bazel 交叉编译。做到 toolchain 注册、编译器切到 arm-none-eabi-gcc、系统头 vendoring、C 源码全部编译通过，最后卡在静态库归档的参数格式上。停下来的判断是：前面 6 个问题都是 Bazel 9 的版本变化，属于新认知；归档那一步是参数调试，不再产出新知识，投入产出比不合适。
>
> 这个过程让我真正理解了 Bazel 的 toolchain 体系为什么比 Makefile 的 `CROSS_COMPILE` 复杂那么多——它要保证可复现，所以每个输入都得声明，连编译器自带的 `stdint.h` 都不例外。」

### 八、后续可扩展方向


- [x] **自定义 rule / genrule 代码生成**：版本头生成已用 Starlark rule 实现
- [x] **`select()` 配置分支**：debug/opt 编译选项切换已验证
- [x] **远程缓存**：bazel-remote 部署 + 跨工作区命中实测
- [~] **交叉编译 toolchain**：编译链路全通（toolchain 生效 / 编译器切换 / 系统头 vendoring / C 源码编译通过），卡在 `ar` 归档参数（见 7.5 状态表、7.6 遗留问题）
- [ ] **新式 rule-based toolchain**（`@rules_cc//cc/toolchains`）：绕开 `cc_toolchain_config_info` 的版本坑，预计能一次性解决 7.6 的 #1/#2/#3
- [ ] **接入 CI**：在 GitLab CI 里加一个 Bazel job，与 Makefile 路线做耗时对比
- [ ] **构建事件分析**：`--build_event_json_file` 输出构建指标，接 DORA 度量
- [ ] **远程执行（RBE）**：把编译分发到多台机器并行（远程缓存的自然演进）

### 九、面试话术

> 「我在嵌入式 C 项目上做过 Make→Bazel 的改造验证。核心收获是理解了两种构建范式：Make 靠时间戳和隐式路径，Bazel 靠内容哈希和显式依赖图。
>
> 最有价值的一条经验是 **`includes` 加了路径不代表头文件进了依赖图** ——Bazel 的沙箱只认声明过的输入，这是从传统构建迁移时最容易踩的坑。
>
> 实测下来，clean 后全量重建 1 秒，重复构建 0.33 秒，增量精度比 Make 高一个量级。
>
> 还落地了远程缓存：部署 bazel-remote 后，一个全新的工作区构建时 6 个 action 全部 remote cache hit、一个都没执行。这让我真正理解了 Bazel 在大团队里的价值——**编译成本是可以在团队内摊销的**，一个人的构建结果其他人直接复用，这是 Make/CMake 做不到的。
>
> 但我也很清楚 Bazel 的成本：新文件要手写进 BUILD，学习曲线陡，小工程不值得。」

### 相关文档

- `05-CI工程化实战.md` 第二部分 — 同一工程的 CI 流水线改造
- `02-Jenkins流水线实战.md` — Make/CMake 路线的构建与排障
