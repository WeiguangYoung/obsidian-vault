---
{"dg-publish":true,"permalink":"/knowledge/devops/practices/06-构建系统实战-Bazel/","title":"Bazel 构建改造实战","tags":["Bazel","构建系统","远程缓存","toolchain","Make","C++","实战"],"dg-note-properties":{"title":"Bazel 构建改造实战","date":"2026-11-03","updated":"2026-11-14","tags":["Bazel","构建系统","远程缓存","toolchain","Make","C++","实战"]}}
---


# Bazel 构建改造实战

> 把 hello-jenkins 工程的纯 C 部分（`UDS_ProtocolStack` + CUnit 单元测试）从 Makefile 迁到 Bazel 的完整记录。
> 含 MODULE.bazel/BUILD.bazel 改造、4 个踩坑、与 Makefile 的实测耗时对照、远程缓存部署、交叉编译 toolchain 卡点复盘。

返回 [[首页\|🏠 首页]] · 相关：[[knowledge/devops/practices/05-构建系统实战-CMake\|CMake 工程实战]] · [[knowledge/embedded/01-编译链接装载与库\|编译、链接、装载与库]]

---



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

- [[knowledge/embedded/cmake-examples/index\|📘 cmake-examples 逐例精读]] — CMake 语法地基：目标模型、作用域、find_package、CTest、CPack
- `05-CI工程化实战.md` 第二部分 — 同一工程的 CI 流水线改造
- `02-Jenkins流水线实战.md` — Make/CMake 路线的构建与排障
