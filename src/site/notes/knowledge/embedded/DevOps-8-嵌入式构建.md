---
{"dg-publish":true,"permalink":"/knowledge/embedded/DevOps-8-嵌入式构建/","tags":["DevOps","嵌入式","C/C++","交叉编译","CMake","Bootloader"],"dg-note-properties":{"date":"2026-07-19","tags":["DevOps","嵌入式","C/C++","交叉编译","CMake","Bootloader"]}}
---


# 八、嵌入式构建

> DevOps 场景下的嵌入式构建：从交叉编译到固件签名，完整工具链。

## 8.1 交叉编译

### 工具链

```
x86 开发机（写代码 + 编译）
    ↓ arm-none-eabi-gcc / arm-linux-gnueabihf-gcc
ARM/嵌入式目标板（运行代码）
```

| 工具链 | 适用 |
|:----|:------|
| **arm-none-eabi-gcc** | Cortex-M 裸机 / RTOS |
| **arm-linux-gnueabihf-gcc** | Cortex-A Linux 应用 |
| **IAR / ARMCC** | 商用认证场景（汽车） |

### 常用编译选项

```makefile
CFLAGS += -mcpu=cortex-m4 -mthumb -mfloat-abi=hard -mfpu=fpv4-sp-d16
CFLAGS += -Os -ffunction-sections -fdata-sections -flto
LDFLAGS += -Wl,--gc-sections -Wl,-Map=output.map
```

## 8.2 构建系统

| 工具 | 特点 | 适用 |
|:----|:------|:------|
| **Make** | 最基础，灵活但容易冗长 | 小型项目 |
| **CMake** | 跨平台，现代目标模型 | 中大型项目（推荐） |
| **Bazel** | Google 出品，快，缓存强 | 大型项目、多仓库 |

### CMake 嵌入式示例

```cmake
cmake_minimum_required(VERSION 3.20)
project(EmbeddedApp C ASM)

set(CMAKE_C_COMPILER arm-none-eabi-gcc)
add_executable(app main.c startup.s)

target_compile_options(app PRIVATE
    -mcpu=cortex-m4 -mthumb -Os
)

target_link_options(app PRIVATE
    -T ${CMAKE_SOURCE_DIR}/linker.ld
    -Wl,-Map=app.map
)

# 构建后生成 .bin / .hex
add_custom_command(TARGET app POST_BUILD
    COMMAND arm-none-eabi-objcopy -O binary app app.bin
    COMMAND arm-none-eabi-objcopy -O ihex app app.hex
)
```

## 8.3 链接脚本

```ld
MEMORY
{
    FLASH (rx) : ORIGIN = 0x08000000, LENGTH = 64K
    RAM   (rwx): ORIGIN = 0x20000000, LENGTH = 128K
}

SECTIONS
{
    .isr_vector : { KEEP(*(.isr_vector)) } > FLASH
    .text       : { *(.text) } > FLASH
    .data : AT(ADDR(.text) + SIZEOF(.text)) { *(.data) } > RAM
    .bss  : { *(.bss) } > RAM
}
```

| 关键概念 | 说明 |
|:----|:------|
| **MEMORY** | 芯片物理内存布局（Flash / RAM 起止地址） |
| **KEEP** | 防止链接器优化掉中断向量表等必需段 |
| **AT()** | .data 段的加载地址（Flash）≠ 运行地址（RAM） |

## 8.4 Bootloader 分区

```
Flash Layout:
┌───────────────────────────────┐
│  0x08000000  Bootloader       │ 64KB
├───────────────────────────────┤
│  0x08010000  Application      │ 448KB
├───────────────────────────────┤
│  0x0807F000  配置/参数区       │ 4KB
└───────────────────────────────┘
```

- BL 和 APP **各自独立链接**，不同链接脚本
- APP 启动后需重映射中断向量：`SCB->VTOR = 0x08010000`
- BL 跳转前校验 APP 的 CRC / 版本号

## 8.5 构建产物

| 格式 | 生成命令 | 用途 |
|:----|:------|:------|
| `.elf` | 链接器直接输出 | 调试（含符号表） |
| `.bin` | `objcopy -O binary` | 烧录（裸二进制） |
| `.hex` | `objcopy -O ihex` | 烧录（Intel HEX 格式） |
| `.map` | `-Wl,-Map=` | 分析内存布局和占用 |

### 固件签名

```bash
# 构建后自动签名
openssl dgst -sha256 -sign private.pem -out app.bin.sig app.bin
# 可选：将签名附加到 .bin 末尾
cat app.bin app.bin.sig > app_signed.bin
```

## 8.6 行业工具链

| 工具 | 场景 | 说明 |
|:----|:------|:------|
| **HIL** | 硬件在环测试 | 真实 ECU + 仿真的物理环境 |
| **SIL** | 软件在环测试 | 纯软件仿真，无需硬件 |
| **CANoe** | 车载网络测试 | CAN / LIN / FlexRay 仿真与测试 |
| **Lauterbach** | 调试器 | 高端嵌入式调试（汽车常用） |
