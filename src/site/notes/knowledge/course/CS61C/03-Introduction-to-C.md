---
{"dg-publish":true,"permalink":"/knowledge/course/CS61C/03-Introduction-to-C/","title":"CS61C L03 · Introduction to C","tags":["CS61C","C语言","编译器","数据类型","课程笔记"],"dg-note-properties":{"title":"CS61C L03 · Introduction to C","tags":["CS61C","C语言","编译器","数据类型","课程笔记"],"date":"2026-11-13","source":"https://cs61c.org/fa26/"}}
---


# CS61C FA26 · L03 Introduction to C

> UC Berkeley · Great Ideas in Computer Architecture
> 授课：Dan Garcia
> 对应课件 03-Introduction to C（53 页）
> 本讲对应 **Great Idea #1：Abstraction**

---

## 一、为什么是 C

### 1.1 历史起点：ENIAC → EDSAC

| 机器 | 年份 | 关键特征 |
|---|---|---|
| **ENIAC** | 1946 | 第一台**电子通用计算机**；换程序要用**插线板和开关**，需 **2–3 天**；当时「computer」主要指**做计算的人**（且多为女性，见《Hidden Figures》2016） |
| **EDSAC** | 1949 | 第一台**通用存储程序计算机**；**程序作为数字存在内存里**——「**Program is also data!**」是革命性的突破；35-bit 二进制补码字 |

> 从「插线板」到「程序即数据」，这就是 **Great Idea #1 Abstraction** 的历史具象。

### 1.2 K&R 的定位（原话）

> *C is not a "very high-level" language, nor a "big" one, and is not specialized to any particular area of application. But its absence of restrictions and its generality make it more convenient and effective for many tasks than supposedly more powerful languages.*

**翻译**：C 不是「很高级」的语言，也不「大」，更不专门服务于某个领域。但**它没有限制**、**足够通用**，因此在很多任务上比那些号称更强的语言更方便、更有效。

### 1.3 C 的实际价值

- **能利用架构的底层特性**：内存管理、特殊指令、并行
- **让第一个操作系统不再用汇编写**：**UNIX 由此可移植（portable OS）**
- 40 多年后，C 及其衍生语言（C++ / Obj-C / C#）**仍是最流行的语言之一**

### 1.4 C 的版本演进

| 年份 | 标准 |
|---|---|
| 1989 前 | **K&R C**（书：1st ed 1978，2nd ed 1988） |
| 1989 / 1990 | **ANSI C**（C89/C90） |
| 1999 | **C99** |
| 2011 | **C11** |
| 2017 | **C17** |
| 2024 | **C23** |

> **本课程使用 C17**（部分内容提到 C23 变化）。
> 类比：Python 2 vs Python 3 —— 同一门语言，语法/特性略有差异。

### 1.5 ⚠️ 重要免责声明

> **You will not learn how to fully code in C in these lectures!**

- 仍然需要 **C 参考手册**：K&R 是必读书
- Brian Harvey 的过渡笔记：`inst.eecs.berkeley.edu/~cs61c/resources/HarveyNotesC1-3.pdf`
- **CS61C 只教关键的 C 概念**：指针、数组、对内存管理的影响
- **安全核心概念**：**C 的内存管理是不安全的**。如果程序有错，可能不会报错，而是悄悄做错事

---

## 二、Hello World 与编译

### 2.1 第一个 C 程序

```c
#include <stdio.h>                        // 导入标准 I/O 库

int main(int argc, char *argv[]) {        // 主函数：返回 int，不是 void
    printf("Hello World!\n");
    return 0;                             // 0 表示成功
}
```

**与 Java 的关键差异：**

| 项 | C | Java |
|---|---|---|
| `main` | **普通函数**，不是对象方法 | `public static void main`，类的静态方法 |
| 返回类型 | `int`（0 = 成功） | `void` |
| 范式 | **函数导向**（function-oriented） | 对象导向 |

### 2.2 编译与运行

```bash
$ gcc hello_world.c      # 编译，默认产出 a.out
$ ./a.out
Hello World!
```

> **gcc 是 C 编译器**：一个把 C 程序转换成「可执行机器码」的程序。

**其他常用命令：**

```bash
gcc -o hello_world hello_world.c   # 指定输出文件名
gcc -g hello_world.c               # 生成调试符号（给 gdb 用）
gcc -std=c99 hello_world.c         # 指定标准
gdb a.out                          # 调试
```

### 2.3 编译器 vs 解释器

**核心区别在于「何时」转成低级指令：**

| 语言 | 方式 |
|---|---|
| **C** | **直接编译**成**架构相关的机器码**（一串 0 和 1） |
| **Java** | 编成**架构无关的 bytecode**，再由 **JIT** 在运行时编译 |
| **Python** | 运行时转成 bytecode（「解释执行」） |

**编译的优点：**

- **编译时间合理**：`Makefile` 只重编改动的文件
- **运行快**：针对特定架构优化
  - （不过现在很多性能在库里）

**编译的缺点：**

- **可执行文件与架构绑定**：依赖 CPU 类型（MIPS / x86 / RISC-V）和 OS（Windows / Linux / macOS）
- **换系统必须重新编译**：所谓「porting」= 复制 `.c` 文件 + 重新 `gcc`
- **「改 → 编 → 跑」循环慢**

### 2.4 为什么还要学 C（2025 视角）

课程很诚实地说了：

> C 比其他语言**低级得多**，**天生不安全**，关键字约定和 scope 也很糟……但**综合考虑，它适合教计算机体系结构**。

**如果你更看重实践，有更好的选择：**

| 需求 | 推荐 |
|---|---|
| **性能 + 安全** | **Rust**（「C-but-safe」：理论上，当你的 C 加满所有检查后，速度不会比 Rust 快） |
| **并发** | **Go**（实用的并发编程，易上手） |

### 2.5 完整编译流程（简化版）

```
foo.c ──┐                    ┌──> foo.o ──┐
        ├─> Compiler ────────┤            │
bar.c ──┘                    └──> bar.o ──┼─> Linker ──> a.out
                                           │
                          lib.o（预编译库）─┘
```

- **CPP（C Pre-Processor）**：编译前先跑一遍宏处理器
  - 把注释替换成一个空格
  - 以 `#` 开头的都是 CPP 命令：`#include`、`#define`、`#if/#endif`
  - 用 `gcc -save-temps` 可以看到中间产物 `foo.i`

---

## 三、C vs Java：语法

### 3.1 总体对比

| 维度 | C | Java |
|---|---|---|
| **范式** | 函数导向 | 对象导向 |
| **编译** | `gcc hello.c` → 机器码 | `javac Hello.java` → bytecode |
| **执行** | `./a.out` 直接执行 | `java Hello` 解释 bytecode |
| **内存管理** | **手动**（`malloc` / `free`） | 自动 GC（`new` 分配） |
| **命名风格** | **snake_case** | camelCase |

### 3.2 语法对照

| 项 | C | Java |
|---|---|---|
| 变量声明 | 带类型，**先声明后使用** | 同 |
| 函数声明 | 用大括号 `{}`；`void` 表示无返回值 | 同 |
| 引入库 | `#include <stdio.h>` | `import java.io.File;` |
| 多行注释 | `/* … */` | 同 |
| 行注释 | `// …` | 同 |

**运算符几乎完全一致：**

```
算术：+ - * / %
赋值：=  += -= *= /= %= &= |= ^= <<= >>=
位运算：~ & | ^      移位：<< >>
布尔：! && ||
比较：== != < <= > >=
自增自减：++ --
成员选择：.  ->      （C 独有 ->，因为结构体和结构体指针并存）
```

### 3.3 命令行参数

```c
#include <stdio.h>

int main(int argc, char *argv[]) {
    printf("Received %d args\n", argc);
    for (int i = 0; i < argc; i++) {
        printf("arg %d: %s\n", i, argv[i]);
    }
    return 0;
}
```

| 变量 | 含义 |
|---|---|
| `argc` | 命令行上的**字符串个数**（**可执行文件自身算一个**） |
| `argv` | 指向**字符串数组**的指针 |

### 3.4 ⚠️ 大括号：能省也别省

- 控制流（`if-else`、`for`）允许省略 `{}`，但**函数定义必须有大括号**
- 单行语句可以省略
- **但后续行会被认为在 body 之外** → 大量调试错误来源
- 原话：**"Just because you can, doesn't mean you should."**

---

## 四、变量与基本类型

### 4.1 C 的基本类型

| 类型 | 描述 | 示例 |
|---|---|---|
| `int` | 整数（含负数） | `0, 78, -217, 0x7337` |
| `unsigned int` | 无符号整数（非负） | `0, 6, 35102` |
| `float` | 浮点 | `0.0, 3.14159, 6.02e23` |
| `double` | 精度 ≥ float | `0.0, 3.14159` |
| `char` | 单个字符 | `'a', 'D', '\n'` |
| `short` | 较短整数 | `-7` |
| `long` | 较长整数 | `0, 78, -217` |
| `long long` | 更长整数 | `3170519272109251` |

> `0x7337` 是十六进制字面量（详见 L02）。

### 4.2 `sizeof()`

> **`sizeof(arg)` 是编译期运算符**，返回**字节数**（参数可以是类型或变量）。

**关于 `int` 的常见选择题**（多选）：

| 选项 | 对错 |
|---|---|
| A. 是**有符号**整数类型 | ✅ |
| B. 至少能容纳 `[-32767, 32767]` | ✅ |
| C. `sizeof(int) = 2` | ❌（现代机器是 4） |
| D. `sizeof(int) = 16` | ❌ |
| E. `sizeof(int) = 4` | ✅（常见） |
| F. `sizeof(int) = 64` | ❌ |

### 4.3 ⚠️ C 不规定整数类型的绝对大小

> **除 `char` 以外，C 标准不定义整数类型的绝对大小。**

只保证**相对大小**：

```
sizeof(long long) ≥ sizeof(long) ≥ sizeof(int) ≥ sizeof(short)
```

- `int` 的大小**取决于机器**，取「对处理器最高效」的宽度
- **要写可移植代码 → 显式指定位数**（`<stdint.h>`）：
  - 用 `intN_t` / `uintN_t`（N = 8, 16, 32, 64）
  - 例：`int32_t x;` 取代 `int x;`

### 4.4 ⚠️ Undefined Behavior（未定义行为）

> C 里有很多「未定义行为」——**结果不可预测**：
> - 在一台机器上这样跑
> - 在另一台上那样跑
> - **甚至同一程序每次运行都不一样**

**两类 bug：**

| 类型 | 特征 |
|---|---|
| **Heisenbug**（海森堡 bug） | 看似随机、难以复现，**调试时消失或改变** |
| **Bohrbug**（玻尔 bug） | 可稳定复现 |

### 4.5 ⚠️ 声明变量 ≠ 初始化变量

**这是本讲最重要的坑**：

```c
int x;          // 声明：x 里是「垃圾值」
...
x = 42;         // 初始化

int y = 38;     // 声明 + 初始化
```

> **变量不会自动初始化为默认值！**
> 如果声明时未初始化，它存的是**垃圾（garbage）**，内容**未定义**……但 **C 仍然允许你使用它**。

**危险示例：**

```c
int x = 0;
int y;                      // 未初始化！
printf("before: x=%d, y=%d, ", x, y);
x++;
y += x;
printf(" after: x=%d, y=%d\n", x, y);
```

**实际输出（某次运行）：**
```
before: x=0, y=22621
after: x=1, y=22622
```

> `%d` 是占位符，`d` 表示按十进制格式化。

**⚠️ 更危险的地方**：这类 bug 可能在你写了程序其他部分之后**才显现**。

### 4.6 为什么 C 的变量要有类型

- **类型不可变**（同一 block 里 `int` 变量不能改存 `float`），但**值可以类型转换（typecast）**
- **类型帮编译器决定如何翻译成机器码**：
  - 变量占**多少字节**
  - 支持**哪些运算符**

```c
uint16_t y = 38;   // 无符号 16 位整数（见 stdint.h），占 16 bit = 2 字节
```

### 4.7 C 的 bool：真与假

> **最初 C 里没有内建的 boolean 类型！**

| 值 | 判定 |
|---|---|
| **FALSE** | `0`（整数，所有位为 0）、`NULL`（指针） |
| **TRUE** | **其他一切** |

```c
if (42) {
    printf("meaning of life\n");   // 会打印！
}
```

> 注意：**Python 里也一样**（非空/非零即真）。

**现在：**

- `true` / `false` 由 `<stdbool.h>` 提供
- **C23 起成为内建类型**（但本课用 C17）

---

## 五、更多 C 特性

### 5.1 const / enum / #define

**`const`**：声明时赋值一次，之后**整个程序运行期间不可修改**

```c
const float golden_ratio = 1.618;
const int   days_in_week = 7;
const double the_law     = 2.99792458e8;
```

**`#define`**：**CPP（C Pre-Processor）宏**

```c
#define PI (3.14159)
```

> **编译前**，预处理器会基于所有 `#define` 做**字符串替换**。

### 5.2 ⚠️ CPP 宏的陷阱

大家都爱用 `#define` 写「小函数」：

```c
#define min(X,Y) ((X)<(Y)?(X):(Y))
```

**但它不是函数，只是文本替换**：

```c
next = min(w, foo(z));
// 展开成：
next = ((w)<(foo(z))?(w):(foo(z)));
//       ⚠ foo(z) 被调用了两次！
```

> 如果 `foo(z)` 有副作用（side effect），结果就错了。

### 5.3 typedef 与 struct

**`typedef`：定义新类型**

```c
typedef uint8_t BYTE;
BYTE b1, b2;
```

**`struct`：结构化变量组**

```c
typedef struct {
    int length_in_seconds;
    int year_recorded;
} SONG;

SONG song1;
song1.length_in_seconds = 213;
song1.year_recorded     = 1994;

SONG song2;
song2.length_in_seconds = 248;
song2.year_recorded     = 1988;
```

> **点号记法**：`x.y = val`

---

## 六、本讲总结（课程原话）

- **选 C 是为了利用硬件底层特性**
  - 指针、数组及对内存管理的影响
  - **下次会讲得更多**
- **C 编译+链接成可执行文件**
  - 优点：速度
  - 缺点：慢速的「编辑-编译」循环
- **C 看起来很像 Java，除了：**
  - 没有 OOP，ADT 用 `struct` 定义
  - `0`（和 `NULL`）为假，**其他一切为真**（C99 起有 `bool` 类型）
  - **可移植代码用 `intN_t` / `uintN_t`**
- **未初始化变量里装的是垃圾！**

---

## 七、我的笔记与理解

### 7.1 本讲的真正定位

L03 是「**C 的入场券**」，不是「C 的教科书」。课程明说了：

> 你**不会**在这些课里学会完整地写 C。

所以**真正的教学目标**是：
1. 知道 C 为什么存在（贴近硬件）
2. 知道它和 Java 的差异在哪（能读懂就够了）
3. **踩过那几个致命的坑**（未初始化、UB、宏）

### 7.2 三个坑的严重性排序

| 坑 | 危险程度 | 为什么 |
|---|---|---|
| **未初始化变量** | 🔴 最高 | 不报错、值随机、**可能几天后才炸** |
| **Undefined Behavior** | 🔴 最高 | 换机器/换编译器/换运行就变 |
| **CPP 宏副作用** | 🟡 中 | 至少是确定性的错误 |
| **省略大括号** | 🟡 中 | 看代码能发现 |

### 7.3 「声明不等于初始化」的深层原因

C 的设计哲学是**「不为你不用的东西付费」**：
- 自动初始化会带来运行时开销
- C 假设**你知道自己在做什么**，愿意手动初始化

**这与 Java/Python 的「安全优先」哲学完全相反。** 这也是为什么 Rust 出现——它想同时要「零成本」和「安全」。

### 7.4 与 L02 的衔接

L02 讲「同一串比特，不同解释」，L03 讲「**C 让你直接操作那串比特**」。

```c
int x = 0x7337;    // 字面量直接对应内存里的比特
```

**C 的「低级」= 没有中间层遮挡**，这就是它能「利用硬件特性」的原因。

### 7.5 与后续的衔接

| 讲次 | 关系 |
|---|---|
| **L04** | 指针、数组、字符串——**C 的核心难点** |
| **L05** | 内存管理：栈、堆、`malloc`/`free` |
| **L06** | 泛型与函数指针 |
| **L07** | 位运算与 ADT |
| **L08** | 浮点数（回答 L02 的 −12.75） |

---

## 八、术语速查

| 术语 | 中文 | 说明 |
|---|---|---|
| **gcc** | GNU 编译器集合 | C 编译器 |
| **CPP** | C 预处理器 | 编译前做宏替换 |
| **K&R** | Kernighan & Ritchie | C 的经典教材 |
| **UB** | 未定义行为 | Undefined Behavior |
| **Heisenbug** | 海森堡 bug | 调试时消失的 bug |
| **Bohrbug** | 玻尔 bug | 稳定复现的 bug |
| **snake_case** | 蛇形命名 | C 的命名约定 |
| **typecast** | 类型转换 | |
| **side effect** | 副作用 | 函数除了返回值还改了状态 |

---

## 相关

- 课程导航：[[knowledge/course/index\|课程 目录导航]]
- 上一讲：[[knowledge/course/CS61C/02-Number-Representation\|L02 Number Representation]]
- 下一讲：[[knowledge/course/CS61C/04-Pointers-Arrays-Strings\|L04 C Pointers, Arrays, Strings]]
