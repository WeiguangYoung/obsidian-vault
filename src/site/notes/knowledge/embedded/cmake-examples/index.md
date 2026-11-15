---
{"dg-publish":true,"permalink":"/knowledge/embedded/cmake-examples/index/","title":"cmake-examples · 总览","tags":["CMake","构建系统","教程","索引"],"dg-note-properties":{"title":"cmake-examples · 总览","date":"2026-11-14","tags":["CMake","构建系统","教程","索引"]}}
---


# cmake-examples · 总览

> 开源仓库 [`ttroy50/cmake-examples`](https://github.com/ttroy50/cmake-examples) 的完整学习笔记（13k+ ⭐，MIT 许可）。
> 7 个章节、**27 个可运行代码示例**（另有 3 个纯文档说明目录），从最小骨架一路讲到包管理。

返回 [[首页\|🏠 首页]] · 相关：[[knowledge/embedded/01-嵌入式软件构建流程\|嵌入式软件构建流程]] · [[knowledge/embedded/02-CMake工程实战-嵌入式固件七能力\|CMake 工程实战]]

---

## 📑 章节索引

| 章 | 主题 | 示例数 | 核心能力 |
|:--|:--|:--:|:--|
| 01 | [[knowledge/embedded/cmake-examples/01-基础篇\|基础篇]] | 12 | 目标、库、作用域、安装、构建类型、第三方库、生成器 |
| 02 | [[knowledge/embedded/cmake-examples/02-子工程篇\|子工程篇]] | 1 | `add_subdirectory` 多级目录组织、目标名工程化 |
| 03 | [[knowledge/embedded/cmake-examples/03-代码生成篇\|代码生成篇]] | 2 | `configure_file` 模板注入、protobuf 代码生成 |
| 04 | [[knowledge/embedded/cmake-examples/04-静态分析篇\|静态分析篇]] | 4 | clang-analyzer、clang-format、cppcheck、compile_commands |
| 05 | [[knowledge/embedded/cmake-examples/05-单元测试篇\|单元测试篇]] | 3 | CTest、Boost.Test、Catch2、GoogleTest |
| 06 | [[knowledge/embedded/cmake-examples/06-打包安装篇\|打包安装篇]] | 1 | CPack 打 deb 包 |
| 07 | [[knowledge/embedded/cmake-examples/07-包管理篇\|包管理篇]] | 4（仅 D 有代码） | 系统包 / vendoring / ExternalProject / **FetchContent** / **Conan 2.x** |

---

## 一、仓库基本信息

| 项 | 值 |
|:--|:--|
| 地址 | `https://github.com/ttroy50/cmake-examples` |
| 星标 | 13,063 |
| Fork | 2,513 |
| 许可 | MIT |
| 创建 | 2015-11-21 |
| **master 最后提交** | **2021-09-14**（`2b27fc7`） |
| 最低 CMake | 3.5 |
| 体积 | 1.9 MB（工作树）/ 2.2 MB（含 `.git`，`depth=1` 浅克隆） |

### 分支说明

| 分支 | 用途 |
|:--|:--|
| `master` | CMake 3.5+ 写法（本文对应） |
| `v3.0-minimum` | CMake 3.0 兼容写法 |
| `v2-style-includes` | CMake 2.x 老语法 |

---

## 二、完整目录树

```
cmake-examples/
├── 01-basic/                      12 个
│   ├── A-hello-cmake              最小骨架
│   ├── B-hello-headers            头文件路径
│   ├── C-static-library           静态库
│   ├── D-shared-library           动态库 + ALIAS
│   ├── E-installing               安装规则
│   ├── F-build-type               构建类型
│   ├── G-compile-flags            编译选项
│   ├── H-third-party-library      find_package(Boost)
│   ├── I-compiling-with-clang     换编译器
│   ├── J-building-with-ninja      换生成器
│   ├── K-imported-targets         IMPORTED 目标
│   └── L-cpp-standard             C++ 标准（3 种写法）
├── 02-sub-projects/
│   └── A-basic                    静态库 + 接口库 + 可执行 三级组织
├── 03-code-generation/
│   ├── configure-files            configure_file 生成头文件
│   └── protobuf                   PROTOBUF_GENERATE_CPP
├── 04-static-analysis/
│   ├── clang-analyzer             编译数据库 + 扫描构建
│   ├── clang-format               格式化检查目标
│   ├── cppcheck                   自定义 analysis 目标
│   └── cppcheck-compile-commands  基于 compile_commands.json
├── 05-unit-testing/
│   ├── boost                      Boost.Test + CTest
│   ├── catch2-vendored            源码内嵌 Catch2
│   └── google-test-download       configure 期下载 GTest
├── 06-installer/
│   └── deb                        CPack 生成 .deb
├── 07-package-management/
│   ├── A-using-system-provide-packages   系统包管理器（无代码，纯说明）
│   ├── B-vendoring-code                  源码 vendor（无代码，纯说明）
│   ├── C-external-project-add            ExternalProject_Add（无代码，纯说明）
│   └── D-conan                           Conan 1.x 集成
├── dockerfiles/                   Docker 测试环境
└── test.sh                        全量构建验证脚本
```

> ⚠️ `07-package-management` 的 A/B/C 三个目录**只有 README 没有代码** —— 它们是概念说明，实际示例分别指向 `01-basic/H`、`05-unit-testing/catch2-vendored`、`05-unit-testing/google-test-download`。

---

## 三、统一构建方式

```bash
git clone https://github.com/ttroy50/cmake-examples.git
cd cmake-examples/01-basic/A-hello-cmake

# 标准三步（out-of-source 构建）
mkdir build && cd build
cmake ..
make
./hello_cmake
```

一键验证全部示例：

```bash
cd cmake-examples && ./test.sh
```

> `test.sh` 会逐个进入示例目录构建并运行，用于验证环境完整性。依赖重的示例（Boost、protobuf、conan）需要先装好对应工具。

### 依赖清单

| 示例 | 额外依赖 |
|:--|:--|
| 01/H、01/K、05/boost | `libboost-all-dev` |
| 01/I | `clang` |
| 01/J | `ninja-build` |
| 03/protobuf | `libprotobuf-dev protobuf-compiler` |
| 04/cppcheck* | `cppcheck` |
| 04/clang-format | `clang-format` |
| 07/D-conan | `python3-pip` + `conan`（1.x） |

---

## 四、七章之间的能力递进

```
01 基础篇 ──────────▶ 能写单个可执行文件 / 库
   │
   ├─ 02 子工程篇 ───▶ 能组织多目录工程
   │
   ├─ 03 代码生成篇 ─▶ 能在构建期生成源码与头文件
   │
   ├─ 04 静态分析篇 ─▶ 能把检查接进构建系统
   │
   ├─ 05 单元测试篇 ─▶ 能把测试接进构建系统（CTest）
   │
   ├─ 06 打包安装篇 ─▶ 能产出可分发的安装包
   │
   └─ 07 包管理篇 ───▶ 能管理第三方依赖
```

一句话概括每章解决什么问题：

| 章 | 解决的问题 |
|:--|:--|
| 01 | 「**怎么写** CMakeLists」——语法与目标模型 |
| 02 | 「**怎么拆**」——目录与目标的责任划分 |
| 03 | 「**怎么生成**」——配置期/构建期的代码生成 |
| 04 | 「**怎么查**」——质量门禁接进构建 |
| 05 | 「**怎么测**」——测试接入 |
| 06 | 「**怎么发**」——产物打包 |
| 07 | 「**怎么用别人的东西**」——依赖管理五条路线（默认 FetchContent） |

---

## 五、这个仓库的时代局限

`master` 停在 2021-09，以下现代 CMake 特性**完全没有覆盖**，学完这套仍需补：

| 特性 | 引入版本 | 替代了什么 |
|:--|:--|:--|
| `FetchContent` | 3.11 | `ExternalProject_Add`（配置期拉取，可直接 `add_subdirectory`） |
| `CMakePresets.json` | 3.19 | 手敲 `-D` 参数、`-G` 生成器 |
| `target_sources` | 3.1 | 部分场景替代 `add_executable` 时列源文件 |
| `FILE_SET HEADERS` | 3.23 | `install(DIRECTORY include/)` |
| `install(TARGETS ... EXPORT)` | 3.x | 让自己的库能被 `find_package` 消费 |
| `cmake --install` / `--build` | 3.15 | `make install` / `make`（生成器无关） |
| Conan 2.x | — | 仓库用的 `conan_basic_setup()` 语法已废弃 |

另外还有几处**当年就不推荐的老写法**在仓库里当正面示例：

- `04/clang-analyzer` 用 `set(CMAKE_BUILD_TYPE Debug)` 硬编码，应该在命令行传
- `07/D-conan` 用 `include(${CMAKE_BINARY_DIR}/conanbuildinfo.cmake)`，现代应改用 `find_package` + Conan 提供的 toolchain
- 多个示例用全局 `CMAKE_CXX_FLAGS` 拼串

**结论：当语法字典和示例模板用，不必照抄全部工程组织方式。**

---

## 六、与其他文档的分工

| 文档 | 定位 |
|:--|:--|
| **本目录（cmake-examples）** | 逐章逐例精读外部教程，打语法地基 |
| [[knowledge/embedded/01-嵌入式软件构建流程\|嵌入式软件构建流程]] | 嵌入式视角：交叉编译、AUTOSAR、STM32 双目标、单元测试接入 |
| [[knowledge/embedded/02-CMake工程实战-嵌入式固件七能力\|CMake 工程实战]] | 完整可跑工程：交叉编译、链接脚本、后处理、闭源库集成 |

学习路径建议：**先读本目录 01 章打基础 → 再读 02 看嵌入式落地 → 最后读 06 看完整工程组织**。
