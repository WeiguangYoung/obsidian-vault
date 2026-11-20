---
{"dg-publish":true,"permalink":"/knowledge/course/cs61c-labs/lab01/","title":"CS61C FA26 · Lab 01 · C 基础","tags":["CS61C","lab","lab01","C语言","指针","数组","字符串","struct"],"dg-note-properties":{"title":"CS61C FA26 · Lab 01 · C 基础","tags":["CS61C","lab","lab01","C语言","指针","数组","字符串","struct"],"date":"2026-11-20","source":"https://cs61c.org/fa26/labs/lab01/"}}
---


# Lab 01 · C

> UC Berkeley · Great Ideas in Computer Architecture · Fall 2026
> 代码仓库：`github.com/61c-teach/fa26-lab-starter`
> 本地路径：`/home/yang/.openclaw/workspace/fa26-lab-starter/lab01/`

返回 [[knowledge/course/cs61c-labs/index\|🧪 Labs 总览]] · [[knowledge/course/cs61c/index\|📚 CS61C 目录]]

---

## 这一课在做什么

Lab 1 是**为 Project 1 铺路**的 C 语言热身：7 个小程序，每个都只填空，但覆盖了 C 里最容易翻车的几块——

```text
编译流程 → 指针 → 数组 → 指针算术 → 字符串 → struct
```

课程特别强调**"spirit of the question"（题目的本意）**：如果你的写法是硬编码输出、绕过了本节要考的概念，会被人工判分。

> ⚠️ 本 lab **必须在 hive 机器上完成**（Lab 0 讲过怎么登）。

---

## 环境准备

```bash
# ① 在 labs 目录，先拉自己之前的改动
git pull origin main

# ② 拉本 lab 的骨架文件
git pull starter main
```

如果报 `fatal: 'starter' does not appear to be a git repository`：

```bash
git remote add starter https://github.com/61c-teach/fa26-lab-starter.git
git pull starter main     # 再跑一次原命令
```

---

## 文件清单

| 文件 | 状态 | 说明 |
|---|---|---|
| `ex1_hello.c` | ⬜ 改字符串 | 编译/重编译观念 |
| `ex2_pointers_and_functions.c` | ⬜ 待实现 | 单指针 + 双指针传参 |
| `ex3_arrays.c` | ⬜ 待实现 | `int8_t` 数组地址计算 |
| `ex4_pointer_arithmetic.c` | ⬜ 待实现 | `int16_t` 数组，对比步长 |
| `ex5_strings.c` | ⬜ 待实现 | 手动构造字符串 + `strcpy` |
| `ex6_structs.c` | ⬜ 待实现 | struct 声明、`sizeof`、`.field` |
| `.gitignore` | 勿改 | 忽略编译产物 |

---

## Exercise 1 · Hello!

**目标**：搞清 `gcc` → 可执行文件 → `./` 执行这条链，并亲眼看到"改了源码但没重编译 = 跑的是旧程序"。

### 三个动作，三种结果

```bash
gcc ex1_hello.c          # 默认产物叫 a.out
./a.out                  # 执行当前目录下的 a.out
```

**关于 `./`**：执行可执行文件要给出**路径**。`.` 是当前目录，`..` 是上一级。所以 `./a.out` = "当前目录下的 a.out"。

**`-o` 选项**：指定产物名，避免所有程序都叫 `a.out`。

```bash
gcc -o ex1_hello ex1_hello.c
./ex1_hello
```

### 这个练习的关键

```text
① 改源码（改 printf 里的字符串为 "Hello 61C"）
② 直接跑 ./ex1_hello  → 仍输出 Hello World   ← 旧的可执行文件
③ 重新 gcc 编译
④ 再跑              → 输出 Hello 61C          ← 新程序
```

**核心认知**：`.c` 和可执行文件是**两份独立的文件**。`gcc` 只在你明确调用时才重新翻译。Java 的 `javac` 也是同理（虽然 Java 需要 JVM）。

### 易错点

| 错误 | 说明 |
|---|---|
| 忘了重新编译 | 改了半天代码没生效，90% 是这个 |
| `a.out` 权限丢失 | 正常 `gcc` 会给 `+x`；手工拷贝后可能没有 |
| 用 IDE 的"运行"按钮 | 本 lab 要求理解命令行链路，别跳过 |

---

## 复习 · 指针（Review: Pointers）

> 本节是官方正文的压缩版，后面 Exercise 2~5 全靠它。

**指针 = 存放地址的变量**。每个变量都在内存里，每个内存单元都有地址。

```c
int main() {
    int my_var = 20;
    int* my_var_p;        // int *p 与 int* p 等价
    my_var_p = &my_var;   // & 取地址
}
```

- `&x` → 取 `x` 的地址
- `*x` → 取"地址为 x 的那个位置的内容"（解引用）

改 `my_var` 有两条路：

```c
my_var += 2;        // 直接改
*my_var_p += 2;     // 通过指针间接改
```

### 完整示例与输出

```c
int main() {
    int my_var = 20;
    int* my_var_p;
    my_var_p = &my_var;

    printf("Address of my_var: %p\n", my_var_p);
    printf("Address of my_var: %p\n", &my_var);
    printf("Address of my_var_p: %p\n", &my_var_p);

    *my_var_p += 2;

    printf("my_var: %d\n", my_var);
    printf("my_var: %d\n", *my_var_p);
}
```

```text
Address of my_var: 0xebafb32c
Address of my_var: 0xebafb32c
Address pf my_var_p: 0xebafb330      ← 注意：比 my_var 大 4，int 占 4 字节
my_var: 22
my_var: 22
```

**关键观察**：`my_var_p` **本身也是变量**，所以它也有地址（第三行）；它和 `my_var` 恰好相邻，相差一个 `sizeof(int)`。

### 官方三个追问（数组/指针算术的先声）

| 问题 | 答案 | 理由 |
|---|---|---|
| `my_var_p += 2` 后 `my_var_p` 是多少？ | `0xebafb32c + 8 = 0xebafb334` | 指针加 2 = 加 `2 * sizeof(int)` = 8 |
| 此时 `&my_var_p` 是多少？ | **不变**，仍是 `0xebafb330` | 变量自己的位置没动，改的是内容 |
| 此时 `*my_var_p` 是什么？ | **未定义行为** | 已指向别处：可能打印垃圾值，也可能 **segfault** |

> ⚠️ 第三个是重点：**指针越界不一定崩溃**，但一定有 bug。`0x334` 越过了后面的部分内存，访问到了不该读的地方。

---

## Exercise 2 · Pointers and Functions

**目标**：用三个版本对比 **值传递 vs 指针传递 vs 双指针传递**。

### 为什么 `add_one(x)` 打印还是 5？

```c
void add_one(int input) {   // input 是 x 的【拷贝】
  input += 1;               // 改的是拷贝，函数返回就消失
}
```

C 的函数参数**永远按值传递**。`int input` 得到一份副本，`input += 1` 只动副本；`x` 在调用者的栈帧里毫发无损。

### 怎么改到调用者的变量？

传**地址**，函数里再解引用回去。

### 参考实现

```c
#include <stdio.h>

void add_one(int input) {
  input += 1;
}

// 参数类型是 int*，接收地址
void add_one_pointer(int* input) {
  *input += 1;              // 解引用后自增，改的是 x 本体
}

// 参数是 int**，接收"指向指针的指针"
void add_one_double_ptr(int** input) {
  **input += 1;             // 解两次引用才回到 int
}

int main() {
  int x = 5;

  add_one(x);
  printf("add_one: %d\n", x);            // 5 —— 值传递，未修改

  add_one_pointer(&x);                   // 传地址
  printf("add_one_pointer: %d\n", x);    // 6

  int* y = &x;                           // y 是 int*，保存 x 的地址
  add_one_double_ptr(&y);                // 传"y 的地址"，类型 int**
  printf("add_one_double_ptr: %d\n", x); // 7

  return 0;
}
```

### 类型推理表（填空时的思维链）

| 要填的位置 | 需要的类型 | 填什么 |
|---|---|---|
| `add_one_pointer(______ input)` | 指向 int 的指针 | `int*` |
| `______ += 1;`（单指针） | 解引用得到 int | `*input` |
| `add_one_double_ptr(______ input)` | 指向 int* 的指针 | `int**` |
| `______ += 1;`（双指针） | 双重解引用 | `**input` |
| `add_one_pointer(______);` | 参数是 `int*` | `&x` |
| `______ y = ______;`（类型） | 存 `&x` 的类型 | `int*` |
| `______ y = ______;`（值） | `x` 的地址 | `&x` |
| `add_one_double_ptr(______);` | 参数是 `int**` | `&y` |

### 输出

```text
add_one: 5
add_one_pointer: 6
add_one_double_ptr: 7
```

### 易错点

- 写成 `add_one_pointer(y)` 而 `y` 已是 `int*` → 对单指针版是对的，但骨架要求的是 `&x`。
- 双指针里写成 `*input += 1` → 加的是**指针的值**（地址），不是整数。
- 以为 `int *p` 和 `int* p` 有语义差别 → **没有**，空格随意。

---

## 复习 · 数组（Review: Arrays）

- 数组**定长**，元素**同类型**、**内存连续**。
- **数组名 = 首元素地址**。
- **传入函数时退化为指针（decay）**，指针本身**不携带长度**，要长度就得另传变量。
- 对指针做加减即移动到相邻元素。

---

## Exercise 3 · Arrays

**目标**：用指针算术拿到下标 2 的元素的地址，再通过它写/读。

```c
#include <stdint.h>
#include <stdio.h>
#include <stdlib.h>

int main() {
  int8_t x = 5;                  // int8_t 占 1 字节

  int8_t some_array[4];          // 类型 int8_t，长度 4
  printf("address of the start of the array: %p\n", some_array);

  // some_array 是 int8_t*，+2 → 前进 2 * 1 = 2 字节
  int8_t* ptr_to_idx_2 = some_array + 2;
  printf("address of index 2: %p\n", ptr_to_idx_2);

  *ptr_to_idx_2 = 10;            // 解引用后写入
  printf("value at index 2: %d\n", *ptr_to_idx_2);

  return 0;
}
```

### 输出该长什么样

```text
address of the start of the array: 0x7ffd5e2a1b30
address of index 2: 0x7ffd5e2a1b32
value at index 2: 10
```

**关键结论**：两个地址**相差 2 字节** —— 因为 `sizeof(int8_t) == 1`，下标 2 就是偏移 `2 × 1`。

### 易错点

| 错误 | 后果 |
|---|---|
| `*ptr_to_idx_2` 当成值打印 | 只要解引用，符号要对齐 |
| 数组未初始化就 print | 打印未定义值（此处只用到 index 2，写入后才读，安全） |
| 写成 `some_array + 2` 却用 `*` 打印地址 | `%p` 要给指针，不是值 |

---

## 复习 · 指针算术（Review: Pointer Arithmetic）

`ptr + n` **不是**地址加 n 字节，而是：

```text
ptr + n  ⟹  地址 + n × sizeof(*ptr)
```

`int* + 5` 且 `sizeof(int) == 4` → 加 **20** 字节。

---

## Exercise 4 · Pointer Arithmetic

**目标**：和第 3 题完全同构，只把 `int8_t` 换成 `int16_t`，**对比偏移量变化**。

```c
#include <stdint.h>
#include <stdio.h>
#include <stdlib.h>

int main() {
  int16_t x = 5;                 // int16_t 占 2 字节

  int16_t some_array[4];
  printf("address of the start of the array: %p\n", some_array);

  // 同样是 +2，但 sizeof(int16_t) == 2 → 前进 4 字节
  int16_t* ptr_to_idx_2 = some_array + 2;
  printf("address of index 2: %p\n", ptr_to_idx_2);

  *ptr_to_idx_2 = 10;
  printf("value at index 2: %d\n", *ptr_to_idx_2);

  return 0;
}
```

### 两题对比

| | Ex3（`int8_t`） | Ex4（`int16_t`） |
|---|---|---|
| 元素大小 | 1 字节 | **2 字节** |
| `some_array + 2` 的偏移 | +2 字节 | **+4 字节** |
| 首地址到 index 2 的距离 | 2 | **4** |

```text
地址差值  =  2 × sizeof(element)
          =  2 × 1 = 2   (Ex3)
          =  2 × 2 = 4   (Ex4)
```

> **这就是指针算术的全部真相**：编译器按 `*ptr` 的类型自动乘上步长，所以你只需要写 `+ i`，不用自己算字节。

### 易错点

- 以为换成 `int16_t` 后偏移还是 2 —— **指针的类型变了，步长就变了**。
- 把 `int16_t*` 写成 `int8_t*` → 偏移回到 2，得到错误的中间地址，写入会踩到元素 1 的后半字节。

---

## 复习 · 字符串（Review: Strings）

- C 字符串 = `char` 数组，**必须以 `\0` 结尾**。
- 数组不含长度信息，`\0` 就是结束标记。
- 分配空间要装下**所有字符 + 1 个 `\0`**。
- 标准库：`strlen`（长度，不含 `\0`）、`strcpy`（拷贝，**连同 `\0`**）。
- 用双引号创建的字符串字面量**可能在只读内存**，不可修改。

---

## ⭐ Exercise 5 · Strings

**目标**：手动逐字符构造 `"hello"`，再用 `strcpy` + 指针算术拼出 `"hello world"`。这是本 lab 最长、最容易出错的一题。

### 参考实现

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

int main() {
  // "hello" 有 5 个字符 + 1 个 '\0' = 6 字节
  char hello_str[6];

  hello_str[0] = 'h';     // 单引号表示字符，双引号是字符串（含 '\0'）
  hello_str[1] = 'e';
  hello_str[2] = 'l';
  hello_str[3] = 'l';
  hello_str[4] = 'o';

  hello_str[5] = '\0';    // 手动补 null terminator

  printf("prints hello: %s\n", hello_str);

  printf("length of hello: %lu\n", strlen(hello_str));   // 5

  char world_str[6];                                     // "world" 同样 5+1

  // 字符串字面量 → 可以是 char*（静态区，只读）
  char* static_world_str = "world";

  // 目标在前，源在后；会连 '\0' 一起复制
  strcpy(world_str, static_world_str);

  printf("prints world: %s\n", world_str);
  printf("address of world_str: %p\n", world_str);

  // "world" = w o r l d → 'r' 在下标 2
  char* ptr_to_r = world_str + 2;
  printf("address of 'r': %p\n", ptr_to_r);

  char hello_world_str[12];    // "hello world" = 11 + 1 = 12

  strcpy(hello_world_str, hello_str);      // 写入 "hello"（含 '\0'）

  hello_world_str[5] = ' ';                // 用空格覆盖那个 '\0'

  // 目标地址 = 基址 + 6，正好接在 "hello " 后面
  strcpy(hello_world_str + 6, world_str);

  printf("prints hello world: %s\n", hello_world_str);

  return 0;
}
```

### 方案示意

```text
hello_world_str  (12 字节)
┌─────┬─────┬─────┬─────┬─────┬─────┬─────┬─────┬─────┬─────┬─────┬─────┐
│ 'h' │ 'e' │ 'l' │ 'l' │ 'o' │ '\0'│ ?   │ ?   │ ?   │ ?   │ ?   │ ?   │  ① strcpy("hello")
└─────┴─────┴─────┴─────┴─────┴─────┴─────┴─────┴─────┴─────┴─────┴─────┘
                                  ↑[5]
┌─────┬─────┬─────┬─────┬─────┬─────┬─────┬─────┬─────┬─────┬─────┬─────┐
│ 'h' │ 'e' │ 'l' │ 'l' │ 'o' │ ' ' │ ?   │ ?   │ ?   │ ?   │ ?   │ ?   │  ② [5] = ' '
└─────┴─────┴─────┴─────┴─────┴─────┴─────┴─────┴─────┴─────┴─────┴─────┘
                                        ↑ 基址 + 6
┌─────┬─────┬─────┬─────┬─────┬─────┬─────┬─────┬─────┬─────┬─────┬─────┐
│ 'h' │ 'e' │ 'l' │ 'l' │ 'o' │ ' ' │ 'w' │ 'o' │ 'r' │ 'l' │ 'd' │ '\0'│  ③ strcpy(+6, "world")
└─────┴─────┴─────┴─────┴─────┴─────┴─────┴─────┴─────┴─────┴─────┴─────┘
```

### 预期输出

```text
prints hello: hello
length of hello: 5
prints world: world
address of world_str: 0x7ffe...
address of 'r': 0x7ffe... (比上一条大 2)
prints hello world: hello world
```

### 易错点（这题雷最多）

| 坑 | 说明 |
|---|---|
| `char hello_str[5]` | **放不下 `\0`**，`printf("%s")` 会读到数组外，打印乱码或崩 |
| 用 `"h"` 而不是 `'h'` | 双引号是字符串（2 字节），赋给 `char` 是类型错 |
| 忘写 `hello_str[5] = '\0'` | 长度变随机，`strlen` 走飞 |
| `strcpy` 参数顺序反了 | 是 **(目标, 源)**，反了直接 segfault |
| 没先覆盖 `[5]` 就是 `' '` | 直接 `strcpy(base+6, ...)` 会得到 `"hello\0world"`，打印只看到 `hello` |
| 用 `char hello_str[] = "hello"` 再改 | 合法，但这题要求**逐字符**，属于 spirit 违规 |
| `hello_world_str` 只开 11 | 少了 `\0` 的位置 |
| 试图改 `static_world_str[0]` | 字面量在只读区 → **segfault** |

> **为什么 `strlen` 是 `%lu` 不是 `%d`**：`strlen` 返回 `size_t`（无符号长整型），用 `%d` 属于类型不匹配。

---

## Exercise 6 · Structs

**目标**：声明 struct、查大小、用点号访问字段。

```c
#include <stdio.h>
#include <stdlib.h>

struct Student {
    int id;
    char* name;
};

int main() {
  struct Student student;                 // 完整类型名要带 struct 关键字
  printf("Size of a struct Student: %lu bytes\n", sizeof(struct Student));

  student.id = 5;                         // . 访问字段
  printf("Student's ID: %d\n", student.id);

  return 0;
}
```

### `sizeof` 的坑

```text
int        4 字节
char*      8 字节（64 位机）
------------------
理论总和   12 字节
实际输出   16 字节  ← 内存对齐（padding）
```

编译器会在 `char*` 前插入 4 字节 padding，让指针按 8 字节对齐。**所以别用字段大小手算，永远用 `sizeof`。**

> ⚠️ `sizeof` 的结果类型也是 `size_t`，`printf` 用 `%lu`；32 位机器上应是 `%zu` 更稳妥。

### 可选：`typedef`

```c
typedef struct {
    int id;
} Student;

Student s;      // 不用再写 struct Student
```

本 lab 不要求，了解即可（见 Lab 正文里的 StackOverflow 链接）。

---

## Exercise 7 / 8 · 问卷

| 练习 | 内容 | 是否计分 |
|---|---|---|
| Exercise 7 | 课程反馈问卷 | 填写即可 |
| Exercise 8 | AutoRemind 研究项目意向调查 | 填写即可 |

两者都是 Google Form，**没有代码**。

---

## 提交

```bash
git add -A
git commit -m "lab01 done"
git push origin main
```

然后到 **Gradescope → Lab 1** 提交。

---

## 易错点总表

| 主题 | 最容易犯的错 |
|---|---|
| 编译 | 改源码后忘 `gcc`，跑的还是旧 `a.out` |
| 指针 | 以为函数参数能直接改调用者变量（C 永远值传递） |
| 双指针 | `*p` 和 `**p` 差一层，加了地址不是加整数 |
| 数组 | 以为函数里能 `sizeof(arr)` 得长度（已退化为指针） |
| 指针算术 | 忘了步长 = `sizeof(*ptr)` |
| 字符串 | 数组少开一个字节放 `\0` |
| 字符串 | `strcpy(dst, src)` 顺序写反 |
| 字符串 | 修改字符串字面量（只读内存 → segfault） |
| struct | 手算大小忽略 padding |

---

## 相关

- Labs 总览：[[knowledge/course/cs61c-labs/index\|🧪 Labs]]
- 前置：[[knowledge/course/cs61c-labs/lab00\|Lab 00 · 环境配置]]
- 对应讲次：[[knowledge/course/cs61c/03-Introduction-to-C\|L03 引言到 C]] · [[knowledge/course/cs61c/04-Pointers-Arrays-Strings\|L04 指针、数组与字符串]] · [[knowledge/course/cs61c/05-Memory-Management\|L05 内存管理]]
