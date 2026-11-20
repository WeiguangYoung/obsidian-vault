---
{"dg-publish":true,"permalink":"/knowledge/course/cs61c-labs/Lab02/","title":"CS61C FA26 · Lab 02 · C 进阶与调试","tags":["CS61C","lab","lab02","GDB","Valgrind","段错误","C语言","课程笔记"],"dg-note-properties":{"title":"CS61C FA26 · Lab 02 · C 进阶与调试","tags":["CS61C","lab","lab02","GDB","Valgrind","段错误","C语言","课程笔记"],"date":"2026-11-20","source":"https://cs61c.org/fa26/labs/lab02/"}}
---


# Lab 02 · C 进阶与调试

> UC Berkeley · Great Ideas in Computer Architecture · Fall 2026
> 代码仓库：`github.com/61c-teach/fa26-lab-starter`
> 本地路径：`/home/yang/.openclaw/workspace/fa26-lab-starter/lab02/`

返回 [[knowledge/course/cs61c-labs/index\|🧪 Labs 总览]] · [[knowledge/course/cs61c/index\|📚 CS61C 目录]]

---

## 环境准备

```bash
git pull starter main
# 若报 fatal: 'starter' does not appear to be a git repository
git remote add starter https://github.com/61c-teach/fa26-lab-starter.git
git pull starter main
```

本 lab 必须在 hive 机器上完成（`cgdb` 已预装）。

| 命令 | 用途 |
|---|---|
| `gcc -o prog prog.c` | 普通编译 |
| `gcc -g -o prog prog.c` | **带调试信息**，GDB / Valgrind 必需 |
| `cgdb prog` | CGDB 打开**可执行文件**（不是 `.c`） |
| `valgrind ./prog` | memcheck 查内存错误 |
| `valgrind --leak-check=full ./prog` | 列出每处泄漏的分配点 |

> ⚠️ 最常见的低级错误：`cgdb pwd_checker.c` —— GDB 要吃的是编译产物，给源文件会报 "not in executable format"。

---

## 文件清单

| 文件 | 状态 | 说明 |
|---|---|---|
| `ex1_compiler_warnings.c` | ⬜ 待修 | 3 个编译警告 |
| `ex2_commands.txt` | ⬜ 待填 | Intro to GDB 命令序列 |
| `ex3_commands.txt` | ⬜ 待填 | 更多 GDB 命令序列 |
| `pwd_checker.c` | ⬜ 待修 | 密码检查器，含 3 个 bug |
| `pwd_checker.h` | 勿改 | `check_password` 原型 |
| `test_pwd_checker.c` | 勿改 | 7 个 assert 测试 |
| `ex5_segfault.c` | ✅ 已完整 | 段错误演示，只读不改 |
| `ex5_answers.txt` | ⬜ 待填 | 2 题：哪个函数 / 哪一行 |
| `bork.c` | ⬜ 待修 | Bork 翻译器，Valgrind 实战 |
| `ex7_vector.c` | ⬜ 待实现 | 动态数组 `vector_new` / `vector_delete` |
| `ex7_vector.h` | 勿改 | `vector_t` 与函数声明 |
| `ex7_test_vector.c` | 勿改 | 测试代码 |
| `ex8_double_pointers.c` | ✅ 已完整 | 双重指针示例，阅读用 |

---

## Exercise 1 · 编译器警告与错误

**目标**：把「编译期能发现的 bug」在运行前清掉。**永远先让警告清零**，找一行警告比找一次运行时错误便宜几个数量级。

`ex1_compiler_warnings.c` 原始代码：

```c
struct Course *make_course(int id, char *name) {
    struct Course *new_course = malloc(sizeof(struct Course));
    new_course->id = id;
    new_course->name = *name;   // ← ① char 赋给 char *
    return &new_course;         // ← ② 返回了 char **
}
```

```bash
gcc -o ex1_compiler_warnings ex1_compiler_warnings.c
```

**怎么读一条警告**（三段信息：`文件:行:列` → 警告类型 → 具体说明，照 `13:22` 跳过去即可）：

```text
ex1_compiler_warnings.c:13:22: warning: assignment to 'char *' from 'char' makes pointer from integer without a cast
```

| # | 警告含义 | 修法 |
|---|---|---|
| ① | 把 `char`（一个字符）赋给 `char *`（指针） | `new_course->name = name;` |
| ② | `&new_course` 类型是 `struct Course **` | `return new_course;` |
| ③ | 缺 `#include` / 丢弃 `const` | 补头文件；签名改 `const char *name` |

**修好后**：`new_course->name = (char *) name;` + `return new_course;`，`main` 里补 `free(cs161);`。输出应为 `Welcome to CS161: Computer Security!`。更严格：`gcc -Wall -Wextra -Werror -o ex1 ex1_compiler_warnings.c`。

---

## 什么是 GDB？

GDB 能让你看程序「内部」在干什么：启动程序、让它在指定条件下停下、检查停下来的状态、修改程序里的东西再试。本课用 **CGDB**（上半屏源码、下半屏命令），命令与 GDB 完全一致，前提是 `-g` 编译。

常用命令总表（贯穿 ex2–ex5）：

| 命令 | 缩写 | 作用 |
|---|---|---|
| `start` | `start` | 运行到 `main` 第一行停下 |
| `run` | `r` | 运行到结束或断点 |
| `step` | `s` | 执行一行，**会进入**函数 |
| `next` | `n` | 执行一行，**不进入**函数 |
| `finish` | `fin` | 执行完当前函数返回调用者 |
| `continue` | `c` | 从断点继续 |
| `break <loc>` | `b` | 在函数名 / `文件:行号` 下断点 |
| `break <loc> if <cond>` | — | **条件断点** |
| `print <expr>` | `p` | 打印变量 / 表达式（`p/x` 十六进制，`p/c` 字符） |
| `info registers` | `i r` | 查看所有寄存器 |
| `x/<n><fmt> <addr>` | — | 查看内存：`x/4wx 0x1000` = 4 个字 |
| `backtrace` | `bt` | 打印调用栈，每帧一行 |
| `frame <n>` | `f n` | 切换到第 n 帧 |
| `quit` | `q` | 退出 |

> ⭐ `s` 关心「函数里面怎么错了」，`n` 关心「这行的效果」。默认按 `n`，需要深究才 `s`。
> ⭐ CGDB 箭头指向的行是「**即将执行、尚未执行**」的那行 —— 看到箭头在第 N 行时，第 N 行的效果还没发生。

---

## ⭐ Exercise 2 · Intro to GDB

```bash
gcc -g -o pwd_checker test_pwd_checker.c pwd_checker.c
cgdb pwd_checker
```

| 步 | 要求 | 命令 | 结论 |
|---|---|---|---|
| 1 | 停在 `main` 第一行 | `start` | — |
| 2 | 跳过 `printf` | `next` | 不进入函数 |
| 3 | 走到 `check_password` 调用 | `next` ×N | 直到箭头在该行 |
| 4 | 进入 `check_password` | `step` | — |
| 5 | 进入 `check_lower` | `step` | — |
| 6 | 打印密码 | `print password` | 看到完整字符串 |
| 7 | 立即跳出 `check_lower` | `finish` | 不逐句走完 |
| 8 | 进入 `check_length` | `step` | — |
| 9 | 走到函数最后一行 | `next` ×N | 箭头停在 `return` 行 |
| 10 | 打印返回值 | `print length` | — |
| 11 | 打印 `length` | `print length` | **`23`**，长度正确 |
| 12 | 退出 | `quit` | 输 `y` 确认（`y` 不写进文件） |

**Bug #1**：`length` 是 23 没问题，但返回 `false` ⇒ 问题就在第 24 行本身。

```c
bool meets_len_req = (length <= 10);   // ← 要求「至少 10」，方向反了
bool meets_len_req = (length >= 10);   // ← 修
```

**`ex2_commands.txt` 参考**：

```text
start
next
next
step
step
print password
finish
step
next
print length
quit
```

> ⚠️ 步 3 的 `next` 条数取决于具体行，逐条写进文件；第 10、11 步均为 `print length`，重点看第 24 行的比较方向。

---

## ⭐ Exercise 3 · More GDB

新增：`break` / 条件 `break` / `run` / `continue` / `backtrace`。

| 步 | 要求 | 命令 | 结论 |
|---|---|---|---|
| 1 | 在 `check_number` 下断点 | `break check_number` | 用函数名，不是 `check_password` |
| 2 | 运行到断点 | `run` | — |
| 3 | 进入 `check_range` | `step` | — |
| 4 | 看调用栈 | `backtrace` | `check_range ← check_number ← check_password ← main` |
| 5 | 第 31 行且 `letter == '0'` 时断 | `break 31 if letter == '0'` | 单引号，`'0'` 是字符 |
| 6 | 继续到条件断点 | `continue` | 跳过前面所有非数字字符 |
| 7 | 验证 | `print letter` | `48 '0'` |
| 8 | 看返回值 | `print is_in_range` | `false` —— 不对劲 |
| 9 | 看下界 | `print lower` | `\000`（即 0） |
| 10 | 看上界 | `print upper` | `\t`（即 9） |
| 11 | 定位 | — | 传的是**整数** `0, 9`，不是字符 |
| 12 | 退出 | `quit` | 输 `y` |

第 7 步输出 `$1 = 48 '0'`：GDB 同时给十进制值与字符。ASCII 表：`'0'`=48，`'9'`=57 —— 所以 `lower=0`、`upper=9` 说明实参根本没按字符传。

**Bug #2**：`check_range` 端点被排除。

```c
bool is_in_range = (letter > lower && letter < upper);    // 错
bool is_in_range = (letter >= lower && letter <= upper);  // 修
```

**Bug #3**：`check_number` 实参传了整数：`check_range(*password, 0, 9)` 应改为 `check_range(*password, '0', '9')`。

> ⭐ `0` 与 `'0'` 都能隐式转成 `char`，**编译器不报警**，只有 GDB 打印出来才看得见。**类型相同 ≠ 语义相同**。

**`ex3_commands.txt` 参考**：

```text
break check_number
run
step
backtrace
break 31 if letter == '0'
continue
print letter
print is_in_range
print lower
print upper
quit
```

---

## Exercise 4 · 自主 Debug


`check_upper` 失败：测试 3、5 含大写字母却返回 `false`。**提示：bug 可能不在 `check_upper` 里**。调用链 `check_upper → check_range`，而 `check_upper` 传的 `'A', 'Z'` 是对的 ⇒ 若 Bug #2 **只修了一半**，端点 `'A'`/`'Z'` 仍判 false。**修全才是修对**。

调试套路：`break check_upper` → `run` → `step` 进循环 → 走到 `check_range` 再 `step` → `print letter / lower / upper / is_in_range` → 对照 `(letter >= lower && letter <= upper)` 手算。

```bash
gcc -g -o pwd_checker test_pwd_checker.c pwd_checker.c
./pwd_checker   # Congrats! You have passed all of the test cases!
```

> ⭐ `check_password` 里 5 个布尔值最后 `&&` 在一起，任一函数出错症状都是「密码被拒」—— 必须先靠断点定位是哪一项为 false，**逐项验证假设，而不是猜**。

---

## ⭐ Exercise 5 · 调试段错误

**Segmentation fault（SIGSEGV）**：进程访问了无权访问的虚拟地址。常见成因：

| 成因 | 例子 |
|---|---|
| **空指针解引用** | `int *p = NULL; *p = 10;` |
| **数组越界** | `arr[10]` 但只有 10 个元素 |
| **野指针 / 未初始化** | `int *p; *p = 1;` |
| **释放后使用（UAF）** | `free(p); *p = 5;` |
| **栈溢出** | 递归无终止条件 |
| **写只读内存** | `"abc"[0] = 'x'` |
| **未检查 `malloc` 返回** | 内存耗尽时解引用 NULL |

```bash
gcc -g -o ex5_segfault ex5_segfault.c
./ex5_segfault        # Segmentation fault (core dumped)

cgdb ex5_segfault
(gdb) run
(gdb) backtrace
```

```text
#0  ben (arr=0x7ffd..., size=10) at ex5_segfault.c:15
#1  0x00000000004011xx in main () at ex5_segfault.c:33
```

读法：`#0` 是最内层（崩溃处），每帧给出函数名、参数、**文件:行号**；`#1` 起是调用者。

`ben` 里：

```c
int *ptr = NULL;
*ptr = 10;      // ← 第 15 行：解引用空指针
```

`jero` 的 `malloc` 检查过返回值，`jedi` 也没问题 —— **光看代码容易怀疑错地方，`backtrace` 一次就锁死行号**。

**`ex5_answers.txt`**（勿改排版）：

```text
1. ben
2. 15
```

> 第 1 题填**函数名** `ben`，第 2 题只填**行号数字** `15`，不带单位。

---

## Valgrind · 内存错误检测

- **bohrbug**：条件确定、能稳定复现 —— 调试器能抓。
- **heisenbug**：一调试就消失，通常源于手动管理的内存 —— 调试器抓不到。

Valgrind 通过模拟 CPU、**跟踪每一次内存访问**来发现后者。代价是慢很多，所以按需使用。规则是**每个元音后面加 `f`**，`hello` → `hefllof`。

```bash
gcc -g -o bork bork.c
./bork hello
```

坏输出（出现乱码）：

```text
Input string: "hello"
Length of translated string: 21
Translate to Bork: "hefl2?^?Ul2?^?Uof?^?U"
```

```bash
valgrind ./bork hello
```

**读大日志的第一原则：只看第一条错误**，后续多半是同一根因的连锁反应。

```text
==10170== Invalid read of size 1
==10170==    at 0x4C34D04: strlen (...)
==10170==    by 0x10879F: make_Str (bork.c:22)
==10170==    by 0x108978: translate_to_bork (bork.c:56)
==10170==    by 0x1089F2: main (bork.c:68)
==10170==  Address 0x522f041 is 0 bytes after a block of size 1 alloc'd
==10170==    by 0x108781: alloc_str (bork.c:10)
```

**从栈底往上读**：`main:68` 只是传一个字符，`translate_to_bork:56` 只是包装 `make_Str`，`make_Str:22` 调 `strlen`。`strlen` 读到 `'\0'` 才停，而 `0 bytes after a block of size 1` 说明**读到了分配区外一字节**。谁分配的？`alloc_str:10`，只给了 `len` 字节，**没留终止符的位置**。

**Bug #4**：

```c
char *alloc_str(int len) {
    char *data = malloc((len + 1) * sizeof(char));
    if (data == NULL) { fprintf(stderr, "Out of memory.\n"); exit(1); }
    data[len] = '\0';
    return data;
}
```

```bash
./bork hello
# Translate to Bork: "hefllof"
```

### 还有内存泄漏

```text
==29797==     in use at exit: 8 bytes in 1 blocks
==29797==   total heap usage: 11 allocs, 10 frees, 1,061 bytes allocated
==29797==    definitely lost: 8 bytes in 1 blocks
==29797== ERROR SUMMARY: 0 errors from 0 contexts
```

| 字段 | 含义 |
|---|---|
| `in use at exit` | 退出时仍占用的堆字节 |
| `total heap usage` | allocs vs frees —— **11 vs 10，少 free 一次** |
| `definitely lost` | 谁都不指向的泄漏块 |
| `still reachable` | 全局变量还指着，退出才回收 |
| `ERROR SUMMARY` | 越界类错误数（这里 0，说明读越界已修）|

```bash
valgrind --leak-check=full ./bork hello
```

```text
==32334== 8 bytes in 1 blocks are definitely lost in loss record 1 of 1
==32334==    by 0x108784: alloc_str (bork.c:10)
==32334==    by 0x10884E: concat (bork.c:32)
==32334==    by 0x108A30: main (bork.c:69)
```

顺着栈回到 `main:69`：`dest_str = concat(...)` 分配了堆内存但**结束时没释放**。在 `return 0;` 前加 `free_Str(dest_str);`。

**为什么不用释放 `src_str`**：`make_Str(argv[1])` 不分配堆内存，`data` 直接指向 `argv[1]`（启动代码管理）。**只 free 自己 malloc 的东西**。
**为什么 `concat` 已有 `free(a.data); free(b.data);` 还漏**：它释放的是**输入**，它**返回**的 `new_str` 是调用者的责任。修好后应为 `11 allocs, 11 frees`、`in use at exit: 0 bytes`、`All heap blocks were freed`。

> ⭐ 检查清单：`ERROR SUMMARY: 0`、allocs 数 == frees 数、`in use at exit: 0`；出现 `definitely lost` 就 `--leak-check=full` 找分配点。

---

## Exercise 6 · Valgrind 判断练习

Google Form 上 7 段输出对应 7 个 bug（无文件产出），可无限次重做。

| 关键短语 | bug |
|---|---|
| `Invalid read/write ... 0 bytes after a block of size N` | **越界（off-by-one）** |
| `Inside a block of size N` | UAF 或读未初始化区 |
| `Invalid free() / delete` | 重复释放或 free 非堆地址 |
| `definitely lost: X bytes` | **内存泄漏** |
| `Conditional jump depends on uninitialised value` | **使用未初始化变量** |
| `... free'd` | **use-after-free** |
| `Syscall param ... unaddressable` | 把非法指针传给系统调用 |

> 三步定类：先看**错误类型关键字**，再看**栈顶函数**，最后看 `Address ...` 与块大小的关系。
---

## ⭐ Exercise 7 · 动态数组（可选，强烈建议）
```c
struct vector_t {
    size_t size;   // 当前（也是已分配）元素个数
    int *data;     // 堆上的 int 数组
};
```

`.h` 里 `typedef struct vector_t vector_t;` 是不完整类型 —— 布局封装在 `.c` 里（信息隐藏）。

**`bad_vector_new()` 为什么错**：

```c
vector_t *retval, v;   // ← v 在栈上
retval = &v;
...
return retval;         // ← 函数一返回栈帧失效，悬垂指针
```

正确做法：struct 本身也 `malloc` 到堆。

**待填位置与参考实现**：

```c
vector_t *vector_new() {
    vector_t *retval;

    retval = malloc(sizeof(vector_t));          // ① 堆上分配 struct
    if (retval == NULL) {                       // ② 检查
        allocation_failed();
    }

    retval->size = 1;                           // ③ size = 1
    retval->data = malloc(sizeof(int));         //    data 分配 1 个 int

    if (retval->data == NULL) {                 // ④ 检查 data
        free(retval);                           //   必须先释放 struct，否则泄漏
        allocation_failed();
    }

    retval->data[0] = 0;                        // ⑤ malloc 不清零，手动置 0
    return retval;                              // 原文件这里是 return NULL
}

void vector_delete(vector_t *v) {
    if (v == NULL) return;
    free(v->data);   // 先里：内部数组
    free(v);         // 后外：结构体本身
}
```

> ⭐ `free` 顺序**先里后外**：先 `free(v)` 就再也读不到 `v->data`，内部数组永久泄漏。
> ⭐ `free(retval)` 那行必要：struct 成功了、`data` 失败时直接 `exit` 会让 Valgrind 报「退出时仍有块占用」，而 `allocation_failed` 的语义只负责报错退出。

**已给好的 `vector_set` 怎么扩容**：

```c
void vector_set(vector_t *v, size_t loc, int value) {
    if (loc >= v->size) {
        v->data = realloc(v->data, (loc + 1) * sizeof(int));
        if (v->data == NULL) allocation_failed();
        for (int i = v->size; i < loc; i++) v->data[i] = 0;  // 中间空洞补 0
        v->size = loc + 1;
    }
    v->data[loc] = value;
}
```

| 要点 | 说明 |
|---|---|
| `realloc` | 扩容后**旧内容自动保留**，无需 memcpy |
| `(loc + 1)` | 要存到下标 loc 就得有 loc+1 个元素 |
| 补零循环 `i < loc` | 不是 `<=`：loc 位置马上被 `value` 覆盖 |
| `v->size = loc + 1` | 扩容后更新 size |

`vector_get` 在 `loc >= v->size` 时返回 0 —— 这就是「未设置的元素都是 0」的语义保证（`malloc` 不清零，全靠 `vector_new` 与 `vector_set` 维护）。

```bash
gcc -g -o ex7_vector ex7_vector.c ex7_test_vector.c
./ex7_vector
valgrind --leak-check=full ./ex7_vector
```

期望：`These should be equal:` 之后 `98 = 98 / 15 = 15 / 65 = 65 / -123 = -123 / 21 = 21 / 43 = 43`，三个 `0 = 0`，最后 `3 = 3`；Valgrind 应报 `in use at exit: 0 bytes`、`All heap blocks were freed`。

> ⚠️ 测试里 `vector_set(v, 500, 3)` 一次跳到 500 号，`realloc` 到 501 个 int 并补 496 个 0。若 `vector_get(v, 23)` 非 0，说明补零循环写错。`size_t` 无符号，严格写法循环变量也统一成 `size_t`。

---

## Exercise 8 · 双重指针：「改指针本身」vs「改指向的内容」

```c
student_t *create_student_t_1(int id) {          // 写法 1：返回指针
  student_t *p = malloc(sizeof(student_t));
  p->id = id;
  return p;
}

void create_student_t_2(student_t **p, int id) { // 写法 2：双重指针出参
  *p = malloc(sizeof(student_t));
  (*p)->id = id;
}
```

| 写法 | 想改谁 | 需要几层指针 |
|---|---|---|
| 改**指向的内容**（`s->id = 5`） | 堆上的 struct | **1 层** `T *` |
| 改**指针本身**（`s = malloc(...)`） | 调用者的指针变量 | **2 层** `T **` |

为什么必须两层？**C 传参一律按值**。传 `student_t *s` 进去，函数拿到的是指针的**副本**；`s = malloc(...)` 只改副本，调用者的指针纹丝不动。想让副本改原变量，就传「原变量的地址」`&ptr`。

| 表达式 | 含义 |
|---|---|
| `p` | `student_t **`，指向调用者的指针变量 |
| `*p` | `student_t *`，即调用者的指针本身 |
| `(*p)->id` | 解一层拿到 struct 指针，再取字段 |
| `*p->id` ❌ | 优先级错误，等价于 `*(p->id)` |

**调用端提醒**：`main` 里先 `malloc` 一次，`create_student_t_2` 里又 `malloc` 覆盖，第一块再也无人指向 ⇒ **示例代码其实有泄漏**。正确用法是让函数负责分配：`student_t *s = NULL; create_student_t_2(&s, 6);`

> ⭐ 一句话记忆：「**内容**要改」传 `T *`；「**指针**要改」传 `T **`；「双重指针的内容要改」传 `T ***`。链表中「头插要传 `Node **head`」是同一件事。

---

## 常见错误速查

| 现象 | 原因 | 处理 |
|---|---|---|
| `not in executable format` | 传了 `.c` 源文件 | 传可执行文件 |
| `<optimized out>` / 行号错乱 | 未加 `-g` 或开了 `-O2` | 用 `gcc -g` 重编 |
| `printf` 打不出东西 | stdout 缓冲，崩在 flush 前 | 加 `\n` 或 `fflush` |
| `print` 数组只给首地址 | 默认行为 | `p *arr@10` |
| `Invalid write of size 4` | 越界或空指针写 | 回到第一条错误顺栈看 |
| 修一处又冒新错 | 工具一次只暴露一处 | 一条一条修 |

## 本 Lab 的 Bug 一览

| 位置 | Bug | 修法 |
|---|---|---|
| `ex1_compiler_warnings.c:13` | `*name` 赋给 `char *` | 去掉 `*` |
| `ex1_compiler_warnings.c:15` | `return &new_course` | `return new_course` |
| `pwd_checker.c:24` | `length <= 10` 方向反 | `>=` |
| `pwd_checker.c:30` | 严格不等号漏端点 | `>=` / `<=` |
| `pwd_checker.c:61` | `0, 9` 应为 `'0', '9'` | 加单引号 |
| `bork.c:10` + `main` 末尾 | 缺终止符 + 未 free | `(len+1)` 并置 `'\0'`；`free_Str(dest_str)` |

> `ex5_segfault.c`（不改，只答题）的 bug 在 `ben` 第 15 行：解引用 `NULL`。

---

## 相关

- Labs 总览：[[knowledge/course/cs61c-labs/index\|🧪 Labs]]
- 对应讲次：
  - [[knowledge/course/cs61c/03-Introduction-to-C\|L03 C 语言入门]]
  - [[knowledge/course/cs61c/04-Pointers-Arrays-Strings\|L04 指针、数组与字符串]]
  - [[knowledge/course/cs61c/05-Memory-Management\|L05 内存管理]]
- 其他 lab：[[knowledge/course/cs61c-labs/Lab03\|Lab 03 · RISC-V & Venus]]
