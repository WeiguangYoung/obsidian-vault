---
{"dg-publish":true,"permalink":"/knowledge/devops/practices/05-构建系统实战-CMake/","title":"CMake 工程实战","tags":["CMake","构建系统","交叉编译","链接脚本","嵌入式","toolchain","闭源库","实战","事故复盘"],"dg-note-properties":{"title":"CMake 工程实战","date":"2026-11-03","updated":"2026-11-25","tags":["CMake","构建系统","交叉编译","链接脚本","嵌入式","toolchain","闭源库","实战","事故复盘"]}}
---


# 🔨 CMake 工程实战

> 本文不是语法教程，而是**从实际工程的代码、提交记录和踩过的坑里总结出来的**。
> 每个结论背后都有具体证据：某次提交、某段注释、某个被误提交进仓库的构建产物。
> 覆盖：裸机交叉编译、ROS 2 sysroot 交叉、ESP-IDF 组件化、可复用库的导出消费。
>
> 配套语法地基见 [[knowledge/embedded/01-编译链接装载与库\|编译、链接、装载与库]]；
> 逐例语法讲解见 [[knowledge/embedded/cmake-examples/index\|cmake-examples 逐例精读]]。

返回 [[首页\|🏠 首页]] · 相关：[[knowledge/embedded/01-编译链接装载与库\|编译、链接、装载与库]] · [[knowledge/devops/practices/06-构建系统实战-Bazel\|Bazel 构建改造实战]]

---

## 📋 cmake-examples 知识点 × 本篇覆盖情况

以 [[knowledge/embedded/cmake-examples/index\|cmake-examples]]（2021 年外部教程，7 章 / 27 个示例）为知识地图，逐个知识点标出本篇是否覆盖：

**图例**：✅ 已覆盖（且比 examples 更贴近真实工程） · ⚠️ 部分覆盖 · ➖ 未覆盖（去 examples 看）

### 01 基础篇

| 知识点 | 状态 | 本篇位置 |
|:--|:--:|:--|
| A · 最小骨架 `project()` / `add_executable()` | ✅ | 第二章 |
| B · 头文件引入 `target_include_directories` | ✅ | 第三章 3.1 |
| C · 静态库 + `PUBLIC`/`PRIVATE` 作用域 | ✅ | 第三章 3.1（含判断口诀） |
| D · 动态库 `.so` + 运行时找不到的坑 | ➖ | 本篇只讲了 `ALIAS`（3.3），没讲动态库 |
| E · `install()` 安装规则 / `CMAKE_INSTALL_PREFIX` | ⚠️ | 第九章讲了 `install(TARGETS)`，未讲安装前缀 |
| F · `CMAKE_BUILD_TYPE` 构建类型 | ➖ | — |
| G · 编译选项分组 `target_compile_options` | ✅ | 第二章 2.3 |
| H · 引入第三方库 | ✅ | 第九章 9.3（专注闭源 `.a`+`.h`） |
| I · 换编译器（clang） | ➖ | — |
| J · 换生成器（ninja） | ➖ | — |
| K · `IMPORTED` 导入目标 | ✅ | 第九章 9.3 |
| L · C++ 标准设置 | ➖ | 本篇针对 C 工程 |

### 02 子工程篇

| 知识点 | 状态 | 本篇位置 |
|:--|:--:|:--|
| 目录结构与顶层 `CMakeLists.txt` | ⚠️ | 第一章对比了四种工程的入口写法 |
| **`add_subdirectory` 多目录组织** | ➖ | 仅在一处例子里顺带提到，未系统讲 |
| `INTERFACE` 库（header-only） | ⚠️ | 第三章讲了可见性，未讲 header-only 组织 |
| 子可执行文件 / 子库 | ➖ | — |
| `target_sources` / `OBJECT` 库 | ➖ | — |

### 03 代码生成篇

| 知识点 | 状态 | 本篇位置 |
|:--|:--:|:--|
| `configure_file` 模板→头文件 | ⚠️ | 第八章用于生成版本头；未讲 `@ONLY` 深入 |
| `#cmakedefine` | ➖ | — |
| protobuf 代码生成 | ➖ | — |
| 生成的时机（配置期 vs 构建期） | ✅ | 第八章 8.1（时间戳求值时机那个坑） |

### 04 静态分析篇

| 知识点 | 状态 | 本篇位置 |
|:--|:--:|:--|
| `compile_commands.json` | ➖ | — |
| clang-analyzer / clang-format | ➖ | — |
| cppcheck 自定义 analysis 目标 | ➖ | — |

### 05 单元测试篇

| 知识点 | 状态 | 本篇位置 |
|:--|:--:|:--|
| CTest 三步 / `add_test` | ➖ | — |
| boost / catch2 / gtest 三种接入 | ➖ | — |
| `FetchContent` 拉测试框架 | ➖ | — |

### 06 打包安装篇

| 知识点 | 状态 | 本篇位置 |
|:--|:--:|:--|
| **CPack 打包 / DEB 变量 / 组件化** | ➖ | — |
| `install(EXPORT)` + `find_package` 消费 | ✅ | 第九章（**examples 把这列为“未覆盖”，本篇补齐**） |
| `configure_package_config_file` / `ConfigVersion` | ✅ | 第九章 9.1、9.2 |

### 07 包管理篇

| 知识点 | 状态 | 本篇位置 |
|:--|:--:|:--|
| 系统包管理器 / vendoring | ➖ | — |
| `ExternalProject_Add` | ➖ | — |
| `FetchContent` | ➖ | — |
| Conan | ➖ | — |

### 🆕 本篇独有（cmake-examples 完全没有）

| 知识点 | 为什么 examples 没有 | 本篇位置 |
|:--|:--|:--|
| **交叉编译三场景**（裸机 `TRY_COMPILE_TARGET_TYPE` / sysroot `PACKAGE BOTH` / 框架注入） | 教程只跑宿主编译 | 第四章 |
| **ESP-IDF 的 CMake 形态** | 教程不涉及嵌入式框架 | 第十章 |
| **`LINK_DEPENDS`：改了链接脚本不重链接** | 教程无链接脚本场景 | 第七章 7.1 |
| **`--gc-sections` + `KEEP(*(.isr_vector))`** | 同上 | 第七章 7.3 |
| **`$<BUILD_INTERFACE>` 导出时的绝对路径报错** | 教程只讲 `PUBLIC` 语义 | 第三章 3.2 |
| **`SameMajorVersion` 在 0.x 的特殊语义** | 教程不涉及包版本 | 第九章 9.2 |
| **三个事故复盘**（缓存不清理 / 产物入库 / 工具链静默失效） | 教程无工程场景 | 第五、六、十一章 |

### 📊 覆盖率小结

| | 数量 | 占比 |
|:--|--:|--:|
| ✅ 已覆盖 | 9 | 26% |
| ⚠️ 部分 | 4 | 12% |
| ➖ 未覆盖（留给 examples） | 21 | 62% |
| **小计（examples 知识点）** | **34** | 100% |
| 🆕 本篇独有 | 7 | — |

**一句话**：本篇只覆盖了 examples 的三分之一多，但把这一块写到了「真实工程踩坑」的深度，
并补了它完全没有的交叉编译与嵌入式场景。**语法查 examples，工程决策查本篇。**

---

## 📑 本页导航

| 章 | 主题 | 核心问题 |
|:--|:--|:--|
| 一 | [工程形态对比](#一工程形态对比) | 同是 CMake，结构差别有多大 |
| 二 | [最小可构建骨架](#二最小可构建骨架basics) | 第一行写什么、为什么 |
| 三 | [静态库与可见性传播](#三静态库与可见性传播static-lib) | PUBLIC/PRIVATE/INTERFACE 到底传什么 |
| 四 | [交叉编译三种真实场景](#四交叉编译三种真实场景) | 裸机 / sysroot / 容器内 |
| 五 | [🩸 事故一：交叉编译不清理 build](#五-事故一交叉编译不清理-build-目录) | 为什么缓存会骗你 |
| 六 | [🩸 事故二：构建产物进了仓库](#六-事故二构建产物进了仓库) | 38000 行垃圾的来历 |
| 七 | [链接脚本与链接期控制](#七链接脚本与链接期控制) | 改了脚本为什么不重链接 |
| 八 | [构建后处理](#八构建后处理post-build) | POST_BUILD 的三个陷阱 |
| 九 | [包导出与消费](#九包导出与消费imported-lib) | 写库给别人用要做什么 |
| 十 | [ESP-IDF 的 CMake 形态](#十esp-idf-的-cmake-形态) | 框架自己定了规则 |
| 十一 | [🩸 事故三：工具链三处静默失效](#十一-事故三工具链三处静默失效) | 不报错但产物不对 |
| 十二 | [检查清单](#十二检查清单) | 照着勾 |

---

## 一、工程形态对比

先把家底摆出来。同样叫「用 CMake」，不同工程的的组织方式完全不同：

| 仓库 | CMakeLists 数 | 形态 | 特点 |
|:--|--:|:--|:--|
| `cmake-learn` | 8 | **学习型多模块** | 每个子目录一个独立工程，各有自己的 `project()` |
| `ros2-ci-demo` | 3 | **ament_cmake** | ROS 2 生态，`colcon build` 驱动，包之间靠 `ament_target_dependencies` |
| `esp32c3-demo` | 2 | **ESP-IDF 组件制** | 顶层 3 行，靠 `idf_component_register()` 注册 |
| `sherpa-onnx-cpp` | 5 | **上游大型 C++ 工程** | 分层目录 + 大量 `.cmake` 模块 |

### 1.1 三种「入口」写法，差别在哪

**A. 标准 CMake（cmake-learn / sherpa-onnx-cpp）**

```cmake
cmake_minimum_required(VERSION 3.16)
project(myproj VERSION 0.1.0 LANGUAGES C)
add_executable(app src/main.c)
```

自己完全掌控。`cmake -S . -B build` 就能跑。

**B. ESP-IDF（esp32c3-demo）**

顶层只有三行：

```cmake
cmake_minimum_required(VERSION 3.16)
include($ENV{IDF_PATH}/tools/cmake/project.cmake)   # 拉入整套框架
project(esp32c3-probe)
```

组件目录里：

```cmake
idf_component_register(SRCS "main.c" INCLUDE_DIRS ".")
```

> **`include(...project.cmake)` 必须在 `project()` 之前**。这一行做了海量工作：扫描 `components/`、注入默认组件、定义 `idf_component_register` 宏、设置交叉编译工具链（riscv32-esp-elf）。**框架接管了 `project()` 之前的一切**，所以你在后面没法自定义编译器。

**C. ROS 2 ament（ros2-ci-demo）**

```cmake
find_package(ament_cmake REQUIRED)
ament_package()      # 必须是最后一行的宏
```

依赖不写 `target_link_libraries`，而是：

```cmake
ament_target_dependencies(my_node rclcpp std_msgs)
```

> ament 会把这些转成「target + include + link」三件套。**但它的传播语义和原生 CMake 不完全一致** —— 混用 `ament_target_dependencies` 和 `target_link_libraries` 时容易出现「编译能过，链接找不到」。

### 1.2 选哪种

| 场景 | 选 | 理由 |
|:--|:--|:--|
| 裸机 / RTOS 固件 | A（手写 CMake）+ toolchain file | 最大控制权 |
| 用 ESP32 做产品 | B（ESP-IDF） | 框架给了 WiFi/BLE 栈，重写没意义 |
| ROS 2 节点 | C（ament） | 生态强制，`colcon` 只认这个 |
| 给外部提供 C/C++ 库 | A + `install(EXPORT)` | 见第九章 |

---

## 二、最小可构建骨架（basics）

`cmake-learn/basics/CMakeLists.txt` 全文 29 行，但每一行都有理由。挑关键的讲：

### 2.1 前两行不能省

```cmake
cmake_minimum_required(VERSION 3.16)
project(basics VERSION 0.1.0 LANGUAGES C)
```

**必须有 `cmake_minimum_required`**：它不只检查版本，还会**启用对应版本的策略（policy）**。老工程不写这行，在新 CMake 上会走兼容模式，行为可能和预期相反（典型是 `if()` 的比较语义）。

**`LANGUAGES C` 别省略**：不写的话 CMake 会探测 C 和 CXX 两个编译器。纯 C 工程多探测一个 g++ 毫无意义，在交叉编译环境里还可能因为找不到 `arm-none-eabi-g++` 而直接配置失败。

### 2.2 C 标准的三件套

```cmake
set(CMAKE_C_STANDARD 11)
set(CMAKE_C_STANDARD_REQUIRED ON)   # 编译器降级就报错
set(CMAKE_C_EXTENSIONS OFF)         # 用 -std=c11 而非 -std=gnu11
```

`CMAKE_C_EXTENSIONS OFF` 这条**在嵌入式上要慎重**：

| 设置 | 生成参数 | 后果 |
|:--|:--|:--|
| `OFF` | `-std=c11` | 严格标准，`asm` 关键字、`__attribute__` 之外的 GNU 扩展全不可用 |
| `ON`（默认） | `-std=gnu11` | 允许 GNU 扩展，`typeof`、语句表达式、`__auto_type` 都能用 |

裸机代码里大量用 `asm volatile(...)` 和 `__attribute__((section(...)))` —— 前者在严格 C11 下是关键字冲突，后者是 GNU 扩展但编译器都认。**我的教训是：纯业务逻辑库设 `OFF`，带内联汇编/链接脚本的固件工程设 `ON`。**

### 2.3 用 `target_*` 而不是全局变量

```cmake
# ✅ 现代写法：作用域限定在 hello 这个目标
target_compile_options(hello PRIVATE -Wall -Wextra)

# ❌ 老写法：污染所有目标
# set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -Wall -Wextra")
```

全局 `CMAKE_C_FLAGS` 的问题：第三方依赖（`add_subdirectory` 进来的）也会吃到你的告警选项，可能直接因为别人的代码告警而 `-Werror` 失败。

### 2.4 字符串宏的转义坑

```cmake
target_compile_definitions(hello PRIVATE
  APP_NAME=\"${PROJECT_NAME}\"
  APP_VERSION=\"${PROJECT_VERSION}\")
```

**必须写 `\"...\"`**。这个坑很隐蔽：

| 写法 | 生成结果 | 源码里 `printf("%s", APP_NAME)` 输出 |
|:--|:--|:--|
| `APP_NAME=\"abc\"` | `-DAPP_NAME="abc"` | `abc` ✅ |
| `APP_NAME="abc"` | `-DAPP_NAME="abc"` 被 shell 吃掉引号 → `-DAPP_NAME=abc` | 编译错误（`abc` 未定义） |
| `APP_NAME=\\\"abc\\\"` | `-DAPP_NAME=\"abc\"` | `"abc"`（含引号）❌ |

---

## 三、静态库与可见性传播（static-lib）

这一章是 CMake 最核心也最容易搞错的地方。`cmake-learn/static-lib` 用 61 行把它演示清楚了。

### 3.1 三种可见性到底传什么

```cmake
add_library(calc STATIC src/add.c src/mul.c)

target_include_directories(calc
    PUBLIC
        $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>
        $<INSTALL_INTERFACE:${CMAKE_INSTALL_INCLUDEDIR}>)
```

当别的目标 `target_link_libraries(app PRIVATE calc)` 时：

| 修饰符 | 自己编译用 | 依赖我的人也用 | 典型用途 |
|:--|:--:|:--:|:--|
| `PRIVATE` | ✅ | ❌ | 内部实现细节（`-DCALC_INTERNAL=1`） |
| `PUBLIC` | ✅ | ✅ | 公开头文件目录、公开依赖 |
| `INTERFACE` | ❌ | ✅ | 纯传播（header-only 库、编译选项包） |

**判断口诀**：问「如果我这个库的头文件里 `#include` 了它，使用者会不会也需要？」

- 会 → `PUBLIC`
- 不会 → `PRIVATE`

### 3.2 `$<BUILD_INTERFACE>` / `$<INSTALL_INTERFACE>` 为什么必须加

```cmake
target_include_directories(calc PUBLIC
    $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>   # 构建树：源码绝对路径
    $<INSTALL_INTERFACE:${CMAKE_INSTALL_INCLUDEDIR}>)        # 安装后：相对安装前缀
```

**不加会怎样**：

- 只写绝对路径 → 安装导出后，`calcTargets.cmake` 里写死了 `/home/you/project/include`。别人 `find_package(calc)` 拿到的是**你的机器上的路径**，在他们机器上不存在 → 编译失败。
- CMake 甚至会在 `install(EXPORT)` 时直接报错：

```
Target "calc" INTERFACE_INCLUDE_DIRECTORIES property contains
path "/home/user/proj/include" which is prefixed in the source directory.
```

> **原则**：任何要 `install(EXPORT)` 出去的目标，`target_include_directories` 都必须用生成器表达式包住源码路径。

### 3.3 `add_library(... ALIAS ...)` 的作用

```cmake
add_library(libmaker::libmaker ALIAS libmaker)
```

**为什么要别名**：让「构建树内引用」和「安装后 find_package 引用」**写法一致**。

```cmake
target_link_libraries(consumer PRIVATE libmaker::libmaker)   # 两种场景同一行
```

没有别名的话，构建树里得写 `libmaker`，安装后得写 `libmaker::libmaker`，切换时到处改。

---

## 四、交叉编译三种真实场景

三个项目，三种完全不同的交叉编译需求。

### 4.1 裸机（Cortex-M）：`CMAKE_TRY_COMPILE_TARGET_TYPE`

`cmake-learn/cross-compile/cmake/toolchains/arm-none-eabi.cmake`：

```cmake
set(CMAKE_SYSTEM_NAME      Generic)     # 裸机：无操作系统
set(CMAKE_SYSTEM_PROCESSOR arm)

set(CMAKE_C_COMPILER   arm-none-eabi-gcc)
set(CMAKE_ASM_COMPILER arm-none-eabi-gcc)

# ⭐ 关键：裸机没有可执行运行环境，try_compile 只能编到静态库
set(CMAKE_TRY_COMPILE_TARGET_TYPE STATIC_LIBRARY)

set(CPU_FLAGS "-mcpu=cortex-m4 -mthumb -mfpu=fpv4-sp-d16 -mfloat-abi=hard")
set(CMAKE_C_FLAGS_INIT   "${CPU_FLAGS}")
```

**`CMAKE_TRY_COMPILE_TARGET_TYPE STATIC_LIBRARY` 为什么是必需的**：

CMake 配置阶段会「试编」一小段程序来探测编译器能力（`try_compile`）。默认行为是编译**并链接成可执行文件**。裸机环境：

1. 没有 `_start`、没有 `main` 的标准启动
2. 没有链接脚本 → 链接器不知道把代码放哪
3. 链接必然失败 → CMake 判定「编译器坏了」→ 配置直接失败

改成 `STATIC_LIBRARY` 后，只编译不链接，探测就能通过。

> **`set(CMAKE_SYSTEM_NAME Generic)` 的另一个作用**：它让 `CMAKE_CROSSCOMPILING` 自动变 `TRUE`，`CMAKE_CROSSCOMPILING_EMULATOR` 之类逻辑才会走对分支。

### 4.2 sysroot 交叉（ROS 2 / aarch64）：`PACKAGE BOTH`

`ros2-ci-demo/docker/aarch64-toolchain-sysroot.cmake` —— 这个文件里有一句注释是血泪：

```cmake
set(CMAKE_SYSROOT "${SYSROOT}")
set(CMAKE_FIND_ROOT_PATH "${SYSROOT}" "/usr/aarch64-linux-gnu")

# 程序在宿主找，头文件和库只在目标找
set(CMAKE_FIND_ROOT_PATH_MODE_PROGRAM NEVER)
set(CMAKE_FIND_ROOT_PATH_MODE_LIBRARY ONLY)
set(CMAKE_FIND_ROOT_PATH_MODE_INCLUDE ONLY)

# ⭐ 关键：PACKAGE 设为 BOTH，否则找不到工作区内刚编译的包
set(CMAKE_FIND_ROOT_PATH_MODE_PACKAGE BOTH)
```

**三个 MODE 的语义**：

| MODE | `ONLY` | `NEVER` | `BOTH` |
|:--|:--|:--|:--|
| `PROGRAM` | 只在 sysroot 找可执行程序 | 只在宿主找 | 都找 |
| `LIBRARY` | 只在 sysroot 找 `.so`/`.a` | 只在宿主找 | 都找 |
| `INCLUDE` | 只在 sysroot 找头文件 | 只在宿主找 | 都找 |
| `PACKAGE` | 只在 sysroot 找 `*Config.cmake` | 只在宿主找 | **都找** |

**为什么 `PACKAGE` 不能设 `ONLY`**：

ROS 2 工作区编译是「先编 `msg` 包，再编依赖它的节点包」。节点包 `find_package(robot_interfaces)` 时，`robot_interfaces` 的 `Config.cmake` 在**当前工作区的 `install/` 目录**里，不在 sysroot 里。

设 `ONLY` → CMake 只去 sysroot 找 → 找不到工作区刚编的包 → 报：

```
Could not find a package configuration file provided by "robot_interfaces"
```

**为什么 `PROGRAM` 必须是 `NEVER`**：编译过程要跑宿主上的工具（`python3`、`protoc`、`objcopy`）。设成 `ONLY` 会去 sysroot 找 aarch64 版本的可执行文件，在 x86 上根本跑不起来。

### 4.3 容器内交叉（ESP-IDF）：框架全都配好了

`esp32c3-demo` 完全不用写 toolchain file —— `include($ENV{IDF_PATH}/tools/cmake/project.cmake)` 已经设好了 riscv32-esp-elf 全套。你只需要 `idf.py build`。

**但代价是**：想覆盖工具链参数只能通过 `sdkconfig`（`CONFIG_ESPTOOLPY_FLASHMODE_DIO` 之类），不能直接改 `CMAKE_C_FLAGS`。**框架接管的地方，你只能按它的规矩来。**

### 4.4 三种场景对比

| 场景 | `CMAKE_SYSTEM_NAME` | sysroot | `TRY_COMPILE_TARGET_TYPE` | `PACKAGE` MODE |
|:--|:--|:--|:--|:--|
| 裸机 Cortex-M | `Generic` | 无 | **`STATIC_LIBRARY`** | 不涉及 |
| ROS aarch64 | `Linux` | 有（`/opt/ros-aarch64-sysroot`） | 默认（`EXECUTABLE`） | **`BOTH`** |
| ESP-IDF | 框架设 | 框架设 | 框架设 | 框架设 |

---

## 五、🩸 事故一：交叉编译不清理 build 目录

这是本文最有价值的一节，因为它是**真实发生过的 bug**，有提交记录。

### 5.1 事故经过

`ros2-ci-demo` 的提交历史：

```
2026-11-08 12:58  feat: 交叉编译脚本 + toolchain 修正            +38064 -11
2026-11-08 12:59  fix(cross-build): 交叉编译前必须清理 build 目录   +3 -0
```

**相隔一分钟**。前一个提交加上交叉编译，后一个提交立刻修复 —— 因为第一次跑就炸了。

修复只加了 3 行，就在 `docker/cross-build.sh` 里：

```bash
echo "[cross-build] 清理旧构建（toolchain 变更必须重新配置）"
rm -rf build install log
```

### 5.2 为什么必须清理

**CMake 把工具链信息缓存在 `CMakeCache.txt` 里**，其中包括：

```
CMAKE_C_COMPILER:FILEPATH=/usr/bin/aarch64-linux-gnu-gcc
CMAKE_TOOLCHAIN_FILE:UNINITIALIZED=/path/to/aarch64-toolchain-sysroot.cmake
CMAKE_SYSTEM_NAME:UNINITIALIZED=Linux
```

**关键行为**：`CMAKE_TOOLCHAIN_FILE` 只在**首次配置**（`CMakeCache.txt` 不存在）时读取。一旦缓存建立，再传 `-DCMAKE_TOOLCHAIN_FILE=...` **会被静默忽略**。

所以这个流程：

```bash
# 第一次：本机编译
cmake -S . -B build           # 缓存里记下 gcc / x86
cmake --build build

# 第二次：想交叉编译到 aarch64
cmake -S . -B build -DCMAKE_TOOLCHAIN_FILE=aarch64.cmake   # ← 被忽略！
cmake --build build
```

结果是：**命令没报错，但产物还是 x86 的**。

### 5.3 故障表现：编译全过，运行时才炸

典型症状链：

| 阶段 | 现象 |
|:--|:--|
| 编译 | ✅ 一路绿灯 |
| 链接 | ✅ 成功 |
| 打包镜像 | ✅ 成功 |
| 部署到 aarch64 板子 | ❌ `Exec format error` 或直接段错误 |
| 排查 | `file binary` 显示 `ELF 64-bit LSB executable, x86-64` |

**最恶心的地方**：CMake 编译和链接用的是**同一套缓存里的编译器**，所以 x86 编译 + x86 链接是完全自洽的，不会报任何错。

### 5.4 实战脚本怎么防

`ros2-ci-demo` 最终的做法（`docker/cross-build.sh`）：

```bash
#!/bin/bash
set -e

WS="${1:-$PWD}"
TC_REL="${3:-docker/aarch64-toolchain-sysroot.cmake}"

cd "$WS"
echo "[cross-build] 清理旧构建（toolchain 变更必须重新配置）"
rm -rf build install log          # ← 核心防御

colcon build --merge-install \
  --cmake-args -DCMAKE_TOOLCHAIN_FILE="$WS/$TC_REL"

# ⭐ 产物架构校验：不信编译日志，直接验产物
ARM_COUNT=$(find install -type f \( -executable -o -name "*.so" \) -print0 \
    | xargs -0 file | grep -c "ARM aarch64" || true)
X86_COUNT=$(find install -type f \( -executable -o -name "*.so" \) -print0 \
    | xargs -0 file | grep -c "x86-64" || true)

echo "[cross-build] ARM64 产物: $ARM_COUNT 个 / x86 产物: $X86_COUNT 个"

if [ "$ARM_COUNT" -eq 0 ]; then
  echo "[cross-build] FAILED: 没有 ARM 产物"; exit 1
fi
if [ "$X86_COUNT" -gt 0 ]; then
  echo "[cross-build] FAILED: 混入 x86 产物"; exit 1
fi
```

**两个防御措施值得抄走**：

1. **构建前无条件 `rm -rf build install log`** —— 交叉编译的 CI 里，宁可多花 30 秒全量重编，也不要赌缓存
2. **构建后用 `file` 验证产物架构** —— 这是**独立于编译系统的第二重证据**。编译日志说成功不算数，产物的 ELF 头才是事实

> 📌 **这条可以推广成通则**：任何「切换了工具链/编译器/关键参数」的场景，都要 **清缓存 + 验产物**。不信日志，信产物。

### 5.5 什么情况可以不清

不是所有情况都要全删。判断依据：

| 变更内容 | 需要清理？ | 说明 |
|:--|:--:|:--|
| **换了 toolchain file** | ✅ 必须 | 缓存会忽略新的 toolchain |
| `CMAKE_C_COMPILER` 改了 | ✅ 必须 | 同上 |
| `CMAKE_SYSTEM_NAME` 改了 | ✅ 必须 | 同上 |
| 只改了源码 | ❌ 不用 | 增量编译正常 |
| 只改了 `-D` 选项 | ❌ 不用 | cache 变量会自动更新 |
| 改了链接脚本 | ❌ 不用 | 但要有 `LINK_DEPENDS`（见第七章） |
| **改了 `CMakeLists.txt` 里的 `project()`** | ✅ 建议清 | `project()` 会重置大量变量 |

---

## 六、🩸 事故二：构建产物进了仓库

同一个提交 `+38064 -11`，这个数字本身就是证据。

### 6.1 38064 行是什么

我把 `ros2-ci-demo` 克隆下来数了一下：

```bash
$ find ros2-ci-demo -name "*.cmake" -not -path "*/.git/*" | cut -d/ -f1-2 | sort | uniq -c
     78 build_arm/robot_interfaces
     27 install_arm/share
      7 build_arm/robot_core
      1 docker/aarch64-toolchain-sysroot.cmake
```

**112 个 `.cmake` 文件里，只有 1 个是手写的**（toolchain），其余 111 个全是 CMake 生成的构建产物。

真实的源码 CMakeLists 只有 3 个：
```
src/robot_core/CMakeLists.txt
src/robot_interfaces/CMakeLists.txt
build_arm/robot_interfaces/*/CMakeLists.txt    ← 这个是生成物
```

### 6.2 为什么这些文件不该进仓库

| 文件类型 | 为什么不该提交 |
|:--|:--|
| `build_*/CMakeCache.txt` | **含本机绝对路径**（`/home/yang/...`），别人拉下来直接失效 |
| `build_*/CMakeFiles/**` | 编译器探测结果、临时 `.o`、依赖文件，与机器强绑定 |
| `install_*/share/**/*.cmake` | 导出文件，含绝对路径，每次构建重新生成 |
| `build_*/Makefile` | 由 CMake 生成，不是手写的 |

**最直接的危害**：`CMakeCache.txt` 里记录的编译器路径，会让别人 clone 后第一次 `cmake` 就报错或行为诡异。

### 6.3 对照：esp32c3-demo 的 `.gitignore` 是对的

```gitignore
# PlatformIO 构建产物
.pio/
.pioenvs/
.piolibdeps/

# 构建输出
*.bin
*.elf
*.map

# PlatformIO 生成的 IDF 配置（由 sdkconfig.defaults 派生）
sdkconfig
sdkconfig.c3
sdkconfig.old
```

注意最后一条 —— **`sdkconfig` 被 ignore，但 `sdkconfig.defaults` 被保留**。这是正确的分工：

| 文件 | 是否入库 | 理由 |
|:--|:--:|:--|
| `sdkconfig.defaults` | ✅ 入库 | **手写的**配置基线，团队共享 |
| `sdkconfig` | ❌ 忽略 | `menuconfig` 生成的完整配置，含环境相关项 |

### 6.4 CMake 工程的标准 `.gitignore`

```gitignore
# CMake 构建树（任意位置）
build*/
cmake-build-*/
install*/
out/
CMakeCache.txt
CMakeFiles/
CMakeScripts/
cmake_install.cmake
CTestTestfile.cmake

# 产物
*.o
*.a
*.so
*.elf
*.bin
*.hex
*.s19
*.map

# IDE
.vscode/
.idea/
*.swp

# 但配置文件要留
!sdkconfig.defaults
!*.cmake.in
!CMakeLists.txt
```

**注意 `build*/` 这个模式** —— 它同时覆盖 `build/`、`build_arm/`、`build-host/`、`build-cov/`。多配置构建（主机侧测试 + 目标侧固件）时特别有用。

> 📌 **一个自查方法**：`git ls-files | grep -E "CMakeCache|CMakeFiles|\.o$"` —— 有输出就说明产物进仓库了。

---

## 七、链接脚本与链接期控制

### 7.1 改了链接脚本为什么不重新链接

`cmake-learn/linker-script/CMakeLists.txt` 里有一句注释，是绕不开的坑：

```cmake
target_link_options(plugin PRIVATE
    "-Wl,--version-script=${CMAKE_CURRENT_SOURCE_DIR}/linker/plugin.map")

# ⭐ 关键：让「改了脚本」也能触发重新链接。
# 光用 target_link_options 不会建立依赖，必须显式声明 LINK_DEPENDS。
set_property(TARGET plugin APPEND PROPERTY LINK_DEPENDS
    ${CMAKE_CURRENT_SOURCE_DIR}/linker/plugin.map)
```

**问题机理**：`target_link_options` 只是往链接命令行里加参数。CMake 生成的依赖图里**不包含这个文件** —— 因为它只是「命令行里的一个字符串」，CMake 不知道那是个文件。

**后果**：改了 `plugin.map`（比如新增导出符号），`cmake --build` 发现源文件没变 → 跳过链接 → 用旧的 `.so`。**改了没生效，而且没有任何提示**。

**`LINK_DEPENDS` 就是补上这条依赖边**：

```cmake
set_property(TARGET firmware APPEND PROPERTY LINK_DEPENDS ${LDSCRIPT})
target_link_options(firmware PRIVATE -T${LDSCRIPT})
```

裸机工程改链接脚本（内存布局调整）后不重链接，是同一类问题。

### 7.2 符号版本脚本：控制导出哪些符号

```cmake
add_library(plugin SHARED src/plugin.c)
target_link_options(plugin PRIVATE
    "-Wl,--version-script=${CMAKE_CURRENT_SOURCE_DIR}/linker/plugin.map")
```

`plugin.map` 内容大致是：

```
{
  global:
    plugin_*;      # 只导出这些
  local:
    *;             # 其余全部隐藏
};
```

**为什么需要**：动态库默认导出**所有非 static 符号**。库内部的辅助函数（`internal_log`、`helper_parse`）全都暴露出去，会导致：

1. 别人可以调你的内部函数 → 以后想改就是 ABI 破坏
2. 符号冲突 → 两个库都有 `helper_parse` 时，运行时加载哪个不确定

**验证导出结果**（`cmake-learn` 的做法）：

```cmake
if(CMAKE_C_COMPILER_ID STREQUAL "GNU")
    add_custom_command(TARGET plugin POST_BUILD
        COMMAND ${CMAKE_NM} -D --defined-only $<TARGET_FILE:plugin>
        COMMENT "plugin 导出的动态符号:" VERBATIM)
endif()
```

> **`if(CMAKE_C_COMPILER_ID STREQUAL "GNU")` 这个判断不能省** —— `--version-script` 是 GNU ld 的选项，换成 MSVC 的 link.exe 或 Apple 的 ld64 会直接报错。

### 7.3 `--gc-sections` 与 `KEEP`

```cmake
target_compile_options(sc_app PRIVATE -ffunction-sections -fdata-sections)
target_link_options(sc_app PRIVATE "-Wl,--gc-sections")
```

两步配合才能生效：

| 步骤 | 作用 |
|:--|:--|
| `-ffunction-sections -fdata-sections` | 让每个函数/变量**独立成段** |
| `-Wl,--gc-sections` | 从入口点做可达性分析，删掉没被引用的段 |

**只加 `--gc-sections` 不加 `-f*`-sections 是无效的** —— 所有代码都在一个巨大的 `.text` 段里，只要有一条被引用，整个段都保留。

**裸机上的致命坑**：中断向量表虽然没被任何代码「调用」，但必须保留。链接脚本里要 `KEEP`：

```ld
.text : {
    KEEP(*(.isr_vector))    /* 没有这行，--gc-sections 会把它删掉 */
    *(.text*)
} > FLASH
```

症状：**烧进去完全不跑**，或者**一进中断就 HardFault**。

---

## 八、构建后处理（post-build）

`cmake-learn/post-build` 模块演示了产物体积、校验和、格式转换。这里有三个真实陷阱。

### 8.1 陷阱一：`string(TIMESTAMP)` 必须在 `add_custom_command` 之前

```cmake
# ✅ 正确位置：先取时间戳
string(TIMESTAMP BUILD_TIMESTAMP "%Y-%m-%dT%H:%M:%SZ" UTC)

add_executable(firmware src/main.c)

add_custom_command(TARGET firmware POST_BUILD
    COMMAND ${CMAKE_COMMAND}
            -DTIMESTAMP=${BUILD_TIMESTAMP}          # 这里用到
            -P ${CMAKE_CURRENT_SOURCE_DIR}/cmake/gen_artifacts.cmake
    ...)
```

**写在后面会怎样**：`add_custom_command` 的 `COMMAND` 在**配置期**就把 `${BUILD_TIMESTAMP}` 展开成字符串了。如果那时变量还是空的，展开结果就是 `-DTIMESTAMP=` → 生成的文件里时间戳为空。

> 这条规则的通用形式：**`add_custom_command` 里所有 `${}` 都是配置期展开**，不是构建期。想要构建期求值，得用 `$<...>` 生成器表达式。

### 8.2 陷阱二：`COMMAND` 不走 shell，不能用 `>` 重定向

```cmake
# ❌ 错误：不会写入文件，会尝试执行名为 ">" 的程序
add_custom_command(TARGET firmware POST_BUILD
    COMMAND ${CMAKE_SIZE} $<TARGET_FILE:firmware> > size.txt)

# ✅ 正确：用 -P 脚本内部用 file() 写
add_custom_command(TARGET firmware POST_BUILD
    COMMAND ${CMAKE_COMMAND} -DARTIFACT=$<TARGET_FILE:firmware>
            -P ${CMAKE_CURRENT_SOURCE_DIR}/cmake/gen_artifacts.cmake)
```

`cmake-learn` 的注释写得很清楚：

> 注意：custom command 的 `COMMAND` 不走 shell，不能用 `>` 重定向；字符串里的 `\n` 也会被 cmake 解析成真换行 —— 一律交给脚本用 `file()` 干。

`gen_artifacts.cmake` 里的写法：

```cmake
file(WRITE "${OUTDIR}/build-info.txt"
"artifact=${APPNAME}\nversion=${APPVER}\nbuilt=${TIMESTAMP}\n")

file(SHA256 "${ARTIFACT}" HASH)
file(WRITE "${OUTDIR}/firmware.sha256" "${HASH}  ${ALGO_NAME}\n")
```

**为什么用 `-P` 脚本而不是直接写命令**：

| 直接写 COMMAND | 用 `-P` 脚本 |
|:--|:--|
| 不能用重定向 | `file(WRITE)` 随便写 |
| 不能做条件判断 | `if()/else()` 都能用 |
| 不能解析输出 | `execute_process()` + `OUTPUT_VARIABLE` |
| 参数转义噩梦 | 通过 `-D` 传，干净 |

### 8.3 陷阱三：POST_BUILD 只在「真的重新链接」时跑

`cmake-learn` 注释：

> `add_custom_command(TARGET <t> POST_BUILD ...)` 会在目标链接完成后执行。
> 注意：POST_BUILD 命令**只在目标真的被重新链接时**才跑（不是每次都跑）。

**这意味着**：

- 只改了注释、改了一个 `.md` 文件 → 不重链接 → 不生成新的 `build-info.txt`
- 想强制每次都跑 → 必须让目标「看起来变了」，或者改用 `add_custom_target(... ALL)` + `add_dependencies`

**验证产物是否最新**的方法：

```cmake
add_custom_target(verify_artifacts ALL
    COMMAND ${CMAKE_COMMAND} -E echo "==== 产物校验 ===="
    COMMAND ${CMAKE_SIZE} $<TARGET_FILE:firmware>
    DEPENDS firmware
    VERBATIM)
```

`add_custom_target(... ALL)` 每次构建都会跑，`DEPENDS firmware` 保证顺序。

### 8.4 完整的产物加工链

从 `gen_artifacts.cmake` 提炼出的标准流程：

```cmake
# 1. 生成构建信息头（给源码 include）
configure_file(
    ${CMAKE_CURRENT_SOURCE_DIR}/cmake/build_info.h.in
    ${CMAKE_CURRENT_BINARY_DIR}/generated/build_info.h
    @ONLY)

# 2. 链接后：size / sha256 / objcopy
add_custom_command(TARGET firmware POST_BUILD
    COMMAND ${CMAKE_SIZE} $<TARGET_FILE:firmware>
    COMMAND ${CMAKE_COMMAND}
            -DARTIFACT=$<TARGET_FILE:firmware>
            -DOUTDIR=$<TARGET_FILE_DIR:firmware>
            -DAPPNAME=${PROJECT_NAME}
            -DAPPVER=${PROJECT_VERSION}
            -DTIMESTAMP=${BUILD_TIMESTAMP}
            -P ${CMAKE_CURRENT_SOURCE_DIR}/cmake/gen_artifacts.cmake
    COMMENT "post-build: 体积/校验和/固件打包")
```

| 产物 | 用途 |
|:--|:--|
| `firmware.elf` | 调试、反汇编、`addr2line` 定位崩溃 |
| `firmware.bin` | 烧录（裸二进制） |
| `firmware.hex` | 烧录（带地址，适合多段） |
| `firmware.sha256` | 校验烧进去的和编出来的是不是同一份 |
| `build-info.txt` | 版本 + 时间戳，现场问题排查 |

---

## 九、包导出与消费（imported-lib）

`cmake-learn/imported-lib` 是两个独立工程：`libmaker`（生产方）和 `consumer`（消费方），演示完整的「我写库给别人用」流程。

### 9.1 生产方要做的四件事

```cmake
# ① 库目标 + 别名
add_library(libmaker STATIC src/libmaker.c)
add_library(libmaker::libmaker ALIAS libmaker)

# ② 头文件用生成器表达式包住
target_include_directories(libmaker PUBLIC
    $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>
    $<INSTALL_INTERFACE:${CMAKE_INSTALL_INCLUDEDIR}>)

# ③ 安装并导出
install(TARGETS libmaker EXPORT libmakerTargets
    ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR}
    INCLUDES  DESTINATION ${CMAKE_INSTALL_INCLUDEDIR})
install(DIRECTORY include/ DESTINATION ${CMAKE_INSTALL_INCLUDEDIR})
install(EXPORT libmakerTargets
    FILE libmakerTargets.cmake
    NAMESPACE libmaker::
    DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/libmaker)

# ④ 生成 Config + ConfigVersion
configure_package_config_file(
    ${CMAKE_CURRENT_SOURCE_DIR}/cmake/libmakerConfig.cmake.in
    ${CMAKE_CURRENT_BINARY_DIR}/libmakerConfig.cmake
    INSTALL_DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/libmaker)
write_basic_package_version_file(
    ${CMAKE_CURRENT_BINARY_DIR}/libmakerConfigVersion.cmake
    VERSION ${PROJECT_VERSION} COMPATIBILITY SameMajorVersion)
```

安装后的目录树：

```
install/
├── lib/
│   ├── liblibmaker.a
│   └── cmake/libmaker/
│       ├── libmakerTargets.cmake          ← install(EXPORT) 生成
│       ├── libmakerConfig.cmake           ← configure_package_config_file 生成
│       └── libmakerConfigVersion.cmake    ← write_basic_package_version_file 生成
└── include/libmaker.h
```

**三个文件各司其职**：

| 文件 | 作用 | 谁找 |
|:--|:--|:--|
| `libmakerConfig.cmake` | 入口，`find_package(libmaker)` 找到它 | `find_package` |
| `libmakerConfigVersion.cmake` | 版本兼容性判断 | `find_package(libmaker 0.2)` |
| `libmakerTargets.cmake` | 定义 `libmaker::libmaker` 这个 IMPORTED 目标 | Config.cmake 内 include |

### 9.2 `SameMajorVersion` 对 0.x 的特殊行为

```cmake
write_basic_package_version_file(... COMPATIBILITY SameMajorVersion)
```

消费方：

```cmake
find_package(libmaker 0.2 CONFIG REQUIRED)
```

**0.x 版本的特殊规则**：SemVer 规定 `0.x` 是「不稳定的主开发阶段」，**次版本号变化 = 不兼容变更**。所以：

| 库版本 | 请求 0.2 | 结果 |
|:--|:--|:--|
| 0.2.0 | ✅ | 匹配 |
| 0.2.5 | ✅ | 匹配 |
| **0.3.0** | ❌ | **不匹配**（0.x 里次版本变化算 breaking） |
| 1.2.0 | ❌ | 不匹配 |

> 这条规则很多人不知道，导致「明明 SameMajorVersion，为什么 0.3 找不到 0.2」的困惑。**1.0 之后才变成常规的「主版本一致即可」。**

### 9.3 闭源库（只有 `.a` + `.h`）怎么接

`cmake-learn/imported-lib` 之外，实际工程的三种做法：

**方案 A：`IMPORTED` 目标（推荐）**

```cmake
add_library(vendor STATIC IMPORTED)
set_target_properties(vendor PROPERTIES
    IMPORTED_LOCATION ${CMAKE_CURRENT_SOURCE_DIR}/vendor/lib/libvendor.a
    INTERFACE_INCLUDE_DIRECTORIES ${CMAKE_CURRENT_SOURCE_DIR}/vendor/inc)

target_link_libraries(app PRIVATE vendor)   # 用法和普通库一样
```

优点：**头文件路径、链接关系全部封装在目标里**，别人 `target_link_libraries` 就够。

**方案 B：`find_library` 动态查找**

```cmake
find_library(VENDOR_LIB NAMES vendor PATHS ${VENDOR_ROOT}/lib)
if(NOT VENDOR_LIB)
    message(FATAL_ERROR "找不到 libvendor.a")
endif()
target_link_libraries(app PRIVATE ${VENDOR_LIB})
target_include_directories(app PRIVATE ${VENDOR_ROOT}/inc)
```

优点：库位置不固定时能用。缺点：**不封装**，每个消费方都要写两行。

**方案 C：`INTERFACE` 库包装（要对外分发时用）**

```cmake
add_library(vendor INTERFACE)
target_link_libraries(vendor INTERFACE ${VENDOR_LIB})
target_include_directories(vendor INTERFACE ${VENDOR_ROOT}/inc)
```

然后 `vendor` 可以作为 `install(EXPORT)` 的一部分导出去，别人 `find_package` 就能用你的封装。

### 9.4 `install(EXPORT)` 的隐形要求

`cmake-learn/advanced` 的注释点出了这个坑：

> 导出 `advanced_lib` 时，它(INTERFACE)依赖的 `advanced_warnings` / `advanced_features` 必须一并 `install(EXPORT)`，否则报 "requires target ... not in any export set"。

```cmake
install(TARGETS advanced_lib advanced_warnings advanced_features
    EXPORT advancedTargets      # ← 三个都要在这里
    ...)
```

**为什么**：`advanced_lib` 通过 `target_link_libraries(... PUBLIC advanced_features)` 依赖了 INTERFACE 库。导出时 CMake 生成的 `Targets.cmake` 里会引用 `advanced::advanced_features`，如果那个目标没被导出，消费方 `find_package` 后 include 这个文件就会失败。

**排查方法**：报错信息会直接点名缺哪个 target：

```
Target "advanced::advanced_lib" requires target "advanced::advanced_features"
that is not in any export set.
```

---

## 十、ESP-IDF 的 CMake 形态

`esp32c3-demo` 是最好的「框架接管 CMake」样本。

### 10.1 只有两个文件

```
CMakeLists.txt          3 行
main/CMakeLists.txt     2 行
```

顶层：

```cmake
cmake_minimum_required(VERSION 3.16)
include($ENV{IDF_PATH}/tools/cmake/project.cmake)
project(esp32c3-probe)
```

组件：

```cmake
idf_component_register(SRCS "main.c" INCLUDE_DIRS ".")
```

### 10.2 `include(...project.cmake)` 做了什么

这一行在 `project()` **之前**执行，它做了：

| 动作 | 后果 |
|:--|:--|
| 注入工具链（riscv32-esp-elf-gcc） | 你不能在外面设 `CMAKE_C_COMPILER` |
| 定义 `idf_component_register()` 宏 | 组件只能这么注册 |
| 建立 `components/` 扫描机制 | 目录结构被约定死 |
| 注入默认编译选项 | 想加要挂到 `IDF_COMPONENT_*` 或 `target_compile_options` |
| 接管 `project()` 的部分行为 | 版本、构建类型受 `sdkconfig` 控制 |

### 10.3 想加自己的编译选项

框架虽然接管了，但仍可挂上去：

```cmake
# 方式一：组件内部，针对自己的源文件
idf_component_register(SRCS "main.c" INCLUDE_DIRS ".")
target_compile_options(${COMPONENT_LIB} PRIVATE -Wno-unused-parameter)

# 方式二：全局（慎用，会影响所有组件）
idf_build_set_property(COMPILE_OPTIONS "-fno-common" APPEND)
```

> **`${COMPONENT_LIB}` 是 `idf_component_register()` 之后才有的变量**，指向本组件自动创建的目标。在它之前用会得到空值。

### 10.4 和纯 CMake 工程的心智差异

| 维度 | 纯 CMake | ESP-IDF |
|:--|:--|:--|
| 工具链 | 你写 toolchain file | 框架注入 |
| 编译选项 | `target_compile_options` | 同上，但受 `sdkconfig` 约束 |
| 分区表 | 不管 | 必须配 `partitions.csv` |
| Flash 参数 | 你自己写在烧录命令 | `sdkconfig` → 自动生成烧录命令 |
| 多目标 | `-DCMAKE_TOOLCHAIN_FILE` 切换 | `idf.py set-target` |

**最重要的一条**：ESP-IDF 里**改配置的正确姿势是改 `sdkconfig.defaults`**，不是改 CMakeLists。比如：

```ini
CONFIG_ESPTOOLPY_FLASHMODE_DIO=y
CONFIG_ESPTOOLPY_FLASHFREQ_40M=y
CONFIG_ESPTOOLPY_FLASHSIZE_4MB=y
```

这比在 CMakeLists 里拼 `-Wl,...` 可靠得多 —— 因为框架的烧录命令、分区表布局都与这些配置联动。

---

## 十一、🩸 事故三：工具链三处静默失效

这个事故来自 `esp32c3-demo` 的裸机部分，但根因全在**构建配置**上。它的特征是「编译成功、烧录成功、运行没输出」，三处错误都不报错。

### 11.1 三处失效点

| # | 失效点 | 现象 | 根因 |
|:--:|:--|:--|:--|
| 1 | `-march` 缺 `_zicsr` | `csrr`/`csrw` 报未识别指令 | GCC ≥12 不再把 Zicsr 隐含在 `rv32i` |
| 2 | ROM `printf` 地址错 | 编译链接全过，串口无输出 | `link.ld` 抄了 xtensa 版的地址 |
| 3 | 中断未使能 | tick 恒为 0 | 启动代码没置 `mstatus.MIE` |

**第 2 条是纯粹的 CMake/构建配置问题**：

```ld
/* ❌ 错：0x40000040 实际是 ets_printf */
PROVIDE(printf = 0x40000040);

/* ✅ 对：C3 的 newlib printf 在 0x400004a4 */
PROVIDE(printf = 0x400004a4);
```

`PROVIDE` 的作用是「如果这个符号没被定义，就用这个地址」。地址错了，链接器照样成功 —— 它只是在 `.text` 里填了一个错误地址。**运行时跳到那个地址，行为完全不可预期**（这里恰好是另一个函数 `ets_printf`，签名不兼容但勉强能跑，所以表现为「无输出」而非崩溃）。

### 11.2 `-march` 的正确写法

```cmake
set(CPU_FLAGS "-march=rv32imc_zicsr -mabi=ilp32")
```

**为什么 `_zicsr` 不能省**：

| GCC 版本 | `rv32i` 是否含 Zicsr | 后果 |
|:--|:--:|:--|
| < 12 | ✅ 隐含 | 能编过 |
| ≥ 12 | ❌ 不再隐含 | `unrecognized opcode 'csrr'` |

这条规则的教训：**工具链升级会改变「默认隐含的扩展集」**。CMake 工程里显式写全 `-march` 是防御性做法。

### 11.3 用 CMake 表达这套配置

```cmake
set(CMAKE_SYSTEM_NAME Generic)
set(CMAKE_SYSTEM_PROCESSOR riscv)
set(CMAKE_TRY_COMPILE_TARGET_TYPE STATIC_LIBRARY)

set(CMAKE_C_COMPILER riscv32-esp-elf-gcc)

set(RISCV_FLAGS "-march=rv32imc_zicsr -mabi=ilp32")
set(CMAKE_C_FLAGS_INIT "${RISCV_FLAGS} -ffunction-sections -fdata-sections")

# 链接期：链接脚本 + gc-sections
set(CMAKE_EXE_LINKER_FLAGS_INIT
    "-nostartfiles -T${CMAKE_CURRENT_LIST_DIR}/../link.ld -Wl,--gc-sections")
```

**注意 `CMAKE_*_FLAGS_INIT` 而不是 `CMAKE_*_FLAGS`**：

| 变量 | 何时用 |
|:--|:--|
| `CMAKE_C_FLAGS_INIT` | **只在 toolchain file 里用**，作为初始值 |
| `CMAKE_C_FLAGS` | 在 CMakeLists 里用（会覆盖命令行 `-D`） |

`_INIT` 后缀的变量只在第一次配置时作为初始值，之后可以被 `-D` 覆盖 —— 这才是 toolchain file 该用的。

---

## 十二、检查清单

### 12.1 新建工程

- [ ] `cmake_minimum_required(VERSION 3.16+)` 在第一行
- [ ] `project()` 显式写 `LANGUAGES C`（或 `C CXX`）
- [ ] 用 `target_*` 系列，不用全局 `CMAKE_C_FLAGS`
- [ ] `.gitignore` 已包含 `build*/`、`install*/`、`CMakeCache.txt`
- [ ] 需要导出的目标，`target_include_directories` 用 `$<BUILD_INTERFACE:>` 包住

### 12.2 交叉编译

- [ ] 裸机：`CMAKE_TRY_COMPILE_TARGET_TYPE STATIC_LIBRARY`
- [ ] 裸机：`CMAKE_SYSTEM_NAME Generic`
- [ ] sysroot：`PACKAGE` MODE 设 `BOTH`（工作区包要能找到）
- [ ] sysroot：`PROGRAM` MODE 设 `NEVER`（宿主工具要能跑）
- [ ] **切换工具链时清理 `build/`**
- [ ] **构建后用 `file` 验证产物架构**

### 12.3 链接

- [ ] 链接脚本 / 版本脚本 加了 `LINK_DEPENDS`
- [ ] `--gc-sections` 配合 `-ffunction-sections -fdata-sections`
- [ ] 向量表用 `KEEP(*(.isr_vector))` 保护
- [ ] 链接期 `ASSERT` 检查 Flash/RAM 溢出

### 12.4 产物后处理

- [ ] `string(TIMESTAMP)` 在 `add_custom_command` **之前**
- [ ] 不用 `>` 重定向，改用 `-P` 脚本 + `file(WRITE)`
- [ ] 需要每次都跑的处理，用 `add_custom_target(... ALL)` 而非 `POST_BUILD`
- [ ] `size` 报告纳入构建检查（超限报错）

### 12.5 库导出

- [ ] `add_library(ns::name ALIAS name)` 有别名
- [ ] `install(TARGETS ... EXPORT ...)` 覆盖所有被依赖的目标（含 INTERFACE 库）
- [ ] `configure_package_config_file` + `write_basic_package_version_file` 都生成
- [ ] 0.x 版本注意 `SameMajorVersion` 的实际语义

---

## 📊 附：工程形态速查

| | cmake-learn | ros2-ci-demo | esp32c3-demo | sherpa-onnx-cpp |
|:--|:--|:--|:--|:--|
| 定位 | 学习模板 | ament 工程 | IDF 工程 | 上游 C++ 大工程 |
| CMakeLists | 8 | 3（真） | 2 | 5 |
| 交叉编译 | 手写 toolchain | sysroot + colcon | 框架注入 | 可选 aarch64 |
| 关键坑 | 可见性传播 | 缓存不清理 | `_zicsr` / ROM 地址 | 无（成熟项目） |
| 值得抄 | 8 个模块递进 | 产物架构校验 | sdkconfig 收敛 | 目录分层 |

---

## 📂 关联文件

- [[knowledge/embedded/01-编译链接装载与库\|编译、链接、装载与库]] — 编译器、链接器、装载、库的底层原理
- [[knowledge/devops/practices/06-构建系统实战-Bazel\|Bazel 构建改造实战]] — 同一工程迁到 Bazel 的对照实验
- [[knowledge/embedded/cmake-examples/index\|📘 cmake-examples 逐例精读]] — 逐例语法讲解
- [[knowledge/embedded/02-裸机编程指南\|裸机编程指南]] — 手写 Makefile + 链接脚本的端到端实践

---

> 🦐 虾管家 · 2026-11-25 重写（基于实际工程的代码与提交历史）