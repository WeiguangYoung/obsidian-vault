---
{"dg-publish":true,"permalink":"/knowledge/course/CS61C/07-Bitwise-Ops-ADTs/","title":"CS61C L07 · IEC Prefixes, Bitwise Ops, ADTs","tags":["CS61C","位运算","IEC前缀","ADT","链表","课程笔记"],"dg-note-properties":{"title":"CS61C L07 · IEC Prefixes, Bitwise Ops, ADTs","tags":["CS61C","位运算","IEC前缀","ADT","链表","课程笔记"],"date":"2026-11-13","source":"https://cs61c.org/fa26/"}}
---


# CS61C FA26 · L07 IEC Prefixes, Bitwise Ops, ADTs

> UC Berkeley · Great Ideas in Computer Architecture
> 授课：Dan Garcia
> 对应课件 07-C IEC, Bitwise Ops, ADTs（32 页）
> **本讲是 C 单元的收尾**

---

## 一、IEC 前缀：KiB vs KB

### 1.1 一个真实的困惑

> 「如果字长是 32 位，我们能访问 **4 GiB「gibibytes」** 内存。
> 但如果字长是 64 位，我们就能访问 **16 EiB「exbibytes」** 内存！——哈？」

**困惑点**：为什么既有 GB 又有 GiB？

### 1.2 SI 前缀 vs IEC 前缀

| SI 前缀 | 缩写 | 大小 | IEC 前缀 | 缩写 | 大小 |
|---|---|---|---|---|---|
| **Kilo** | K | 10³ = 1,000 | **Kibi** | Ki | 2¹⁰ = 1,024 |
| **Mega** | M | 10⁶ = 1,000,000 | **Mebi** | Mi | 2²⁰ = 1,048,576 |
| **Giga** | G | 10⁹ = 1,000,000,000 | **Gibi** | Gi | 2³⁰ = 1,073,741,824 |
| **Tera** | T | 10¹² | **Tebi** | Ti | 2⁴⁰ = 1,099,511,627,776 |
| **Peta** | P | 10¹⁵ | **Pebi** | Pi | 2⁵⁰ = 1,125,899,906,842,624 |
| **Exa** | E | 10¹⁸ | **Exbi** | Ei | 2⁶⁰ = 1,152,921,504,606,846,976 |
| **Zetta** | Z | 10²¹ | **Zebi** | Zi | 2⁷⁰ |
| **Yotta** | Y | 10²⁴ | **Yobi** | Yi | 2⁸⁰ |

> **SI 用 10 的幂，IEC 用 2 的幂。** 两者名字相近，容易混淆。

**记忆口诀（课程给的）**：
> "Kilo, Mega, Giga, Tera, Peta, Exa, Zetta, Yotta"
> 例："Kissing Mel Gibson, Teddy Pendergrass exclaimed: 'Zesty, yo!'" — Dan G

### 1.3 必背表：2^X 与 2^Y 的组合

**核心规则**：`2^(10X + Y)` 可以拆成「X 档前缀 + Y 位的倍数」：

| Y \ X | 0 | 3 | 6 | 9 | 12 | 15 | 18 | 21 | 24 |
|---|---|---|---|---|---|---|---|---|---|
| **0** | 1 | ~10³ kibi | ~10⁶ mebi | ~10⁹ gibi | ~10¹² tebi | ~10¹⁵ pebi | ~10¹⁸ exbi | ~10²¹ zebi | ~10²⁴ yobi |
| **1** | 2 | | | | | | | | |
| **2** | 4 | | | | | | | | |
| **3** | 8 | | | | | | | | |
| **4** | 16 | | | | | | | | |
| **5** | 32 | | | | | | | | |
| **6** | 64 | | | | | | | | |
| **7** | 128 | | | | | | | | |
| **8** | 256 | | | | | | | | |
| **9** | 512 | | | | | | | | |

> **MEMORIZE!** 课程明确要求背下这个表。

**例题**：`2³⁴ = ?`

```
34 = 30 + 4
   = (3 × 10) + 4
2³⁰ = 1 GiB
2⁴  = 16
∴ 2³⁴ = 16 GiB
```

---

## 二、位运算

### 2.1 一输入 / 两输入布尔运算（真值表）

**AND（"both"，两者都为 1 才是 1）**

| a | b | y = a && b |
|---|---|---|
| 0 | 0 | 0 |
| 0 | 1 | 0 |
| 1 | 0 | 0 |
| 1 | 1 | **1** |

**OR（"either"，任意一个为 1 就是 1）**

| a | b | y = a \|\| b |
|---|---|---|
| 0 | 0 | 0 |
| 0 | 1 | **1** |
| 1 | 0 | **1** |
| 1 | 1 | **1** |

**NOT（取反）**

| a | y = !a |
|---|---|
| 0 | **1** |
| 1 | **0** |

**XOR（"different"，不同为 1）**

| a | b | y = (a != b) |
|---|---|---|
| 0 | 0 | 0 |
| 0 | 1 | **1** |
| 1 | 0 | **1** |
| 1 | 1 | 0 |

**NAND / NOR**：`~(a&b)` 和 `~(a|b)`

### 2.2 三个「透视」理解

课程给出了极其好用的三个视角：

| 运算 | 视角 | 含义 |
|---|---|---|
| **AND** | **"Set-to-0 / Keep"** | `a` 为 0 → 结果 0；`a` 为 1 → 保留 `b` |
| **OR** | **"Keep / Set-to-1"** | `a` 为 0 → 保留 `b`；`a` 为 1 → 结果 1 |
| **XOR** | **"Conditional Inverter"** | `a` 为 0 → 保留 `b`；`a` 为 1 → **反转 `b`** |

| AND 的透视 | OR 的透视 | XOR 的透视 |
|---|---|---|
| `0 & b = 0` | `0 \| b = b` | `0 ^ b = b` |
| `1 & b = b` | `1 \| b = 1` | `1 ^ b = !b` |

> **这三个透视是理解位掩码（bitmask）的关键。**

### 2.3 按位运算（多 bit）

> **Bitwise AND、OR、XOR、NOT 都是「一次作用于每一位」。**

**例子：**

```
AND   0b1001 & 0b0011 ⇒ 0b0001
OR    0b1001 | 0b0011 ⇒ 0b1011
XOR   0b1001 ^ 0b0011 ⇒ 0b1010
NOT   ~0b1001        ⇒ 0b0110（取反）
```

### 2.4 掩码（Bitmask）

> **AND 用于位掩码。**

```c
N & 0xFF    // 只保留最低字节（LSB）
```

**原理**：`0xFF = 0b1111 1111`，按 AND 的透视「1 保留、0 清零」，所以高位的都被清零，只留低 8 位。

| 用途 | 写法 |
|---|---|
| **提取**低字节 | `N & 0xFF` |
| **提取**某个 bit | `(N >> k) & 1` |
| **设置**某个 bit | `N \| (1 << k)` |
| **清除**某个 bit | `N & ~(1 << k)` |
| **翻转**某个 bit | `N ^ (1 << k)` |

### 2.5 左移：乘 2ⁿ

> **左移 n 位 = 乘以 2ⁿ**

```c
1 << 3 = 8
```

```
0b ...0000 0001   (1)
     ↓ 左移 3 位
0b ...0000 1000   (8)
         ↑
    3 位被移出，新补 3 个 0
```

> **逻辑左移（Logical Left Shift）：补 0。**

### 2.6 右移：除 2ⁿ，但有两种

> **右移 n 位 ≈ 除以 2ⁿ（取下限 floor）**

**关键差异：填 0 还是填符号位？**

```c
 9 >> 2 = floor(2.25) =  2
-9 >> 2 = floor(-2.25) = -3
```

| 类型 | 说明 | 填充 |
|---|---|---|
| **Logical right shift**（逻辑右移） | 补 0 | `0b 1001 → 0b 0010` |
| **Arithmetic right shift**（算术右移） | 补符号位 | `0b 1111...0111 → 0b 1111...1101` |

> **对无符号数，两者一样。对负数，算术右移保持负号。**

**验证：**

```
 9 = 0b 0000 1001  → >> 2 → 0b 0000 0010 =  2   ✅
-9 = 0b 1111 0111  → >> 2 → 0b 1111 1101 = -3   ✅（补了 1）
```

### 2.7 例题：掩码与移位组合

```c
uint32_t N = 0x34FF;
return N & ((N << 0x10) >> 0x8);
```

**答案：`0x3400`**

**逐步演算：**

```
N              = 0x0000 34FF

N << 0x10      = 0x34FF 0000      （左移 16 位）

(N << 16) >> 8 = 0x0034 FF00      （右移 8 位 → 相当于左移 8 位）

N & 上式        = 0x0000 34FF
               & 0x0034 FF00
               ---------------
                 0x0000 3400      ✅
```

**拆解**：

```
   0x34 & 0xFF = 0x34
   0x00 & 0xFF = 0x00
```

---

## 三、链表：简单示例

### 3.1 代码

```c
typedef struct _node node_t;

struct _node {
    char *data;
    node_t *next;
};

void list_add_front(node_t **head_ptr, char *data) {
    node_t *node = (node_t *)malloc(sizeof(node_t));
    node->data = (char *)malloc(strlen(data) + 1);   // extra byte
    strcpy(node->data, data);
    ...
}

int main() {
    node_t *head = NULL;
    list_add_front(&head, "abc");
    ...
    // free nodes, strings here…
}
```

### 3.2 两个「明显的错误」

课件在这里留了作业：**这段代码有 2 个 glaring errors**。参考 L05/L06 的知识，可以推断：

| # | 问题 | 说明 |
|---|---|---|
| **1** | `malloc` 后**没检查 NULL** | 内存耗尽时会崩 |
| **2** | `node->next` **未赋值** | 新节点没有连到链表上（应为 `*head_ptr`）|

> **还有第 3 个**：`head` 本身没有更新（`*head_ptr = node;` 缺失）

**这正是「C 管理内存容易出错」的现场教学。**

---

## 四、构建链表 ADT（抽象数据类型）

### 4.1 目标

> 创建一个**可以从其他函数使用**的链表 ADT。

**设计 API：**

| 函数 | 作用 |
|---|---|
| `list_create` | 创建空链表 |
| `list_add_front` | 头插 |
| `list_print` | 打印 |
| `list_free` | 释放 |

> **为避免污染命名空间，所有函数都加 `list_` 前缀。**

### 4.2 三文件结构

| 文件 | 作用 |
|---|---|
| **`linkedlist.h`** | ADT 接口（头文件） |
| **`linkedlist.c`** | 实现 |
| **`list_demo.c`** | 顶层代码（含 `main`） |

**这是 C 的多文件编程范式** —— 后面 CMake/Makefile 的基础。

### 4.3 头文件 `linkedlist.h`

```c
#ifndef LINKEDLIST_H
#define LINKEDLIST_H           // ← 防止重复包含
                               //   （include guard）

typedef struct _node node_t;                       // forward declaration
typedef struct _linked_list linked_list_t;         // opaque list type

// 创建一个新的空链表
linked_list_t *list_create(void);

// 在链表头部添加节点
void list_add_front(linked_list_t *list, const char *data);

// 打印链表
void list_print(const linked_list_t *list);

// ...
#endif
```

**两个关键设计：**

| 技术 | 作用 |
|---|---|
| **include guard** | `#ifndef / #define / #endif` 防止重复包含 |
| **forward declaration**（前向声明） | `typedef struct _node node_t;` —— 头文件里**只声明类型存在，不暴露内部结构** |

### 4.4 实现 `linkedlist.c`

```c
#include "linkedlist.h"
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

struct _node {
    char *data;
    node_t *next;
};

struct _linked_list {
    node_t *head;
};

linked_list_t *list_create(void) {
    linked_list_t *list = malloc(sizeof(linked_list_t));
    if (!list) return NULL;        // ✅ 检查 NULL
    list->head = NULL;
    return list;
}

void list_add_front(linked_list_t *list, const char *data) {
    node_t *node = malloc(sizeof(node_t));
    if (NULL == node) {
        printf("Error in list_add_front: Couldn't allocate node_t\n");
        exit(1);
    }
    node->data = (char *)malloc(strlen(data) + 1);   // +1 给 '\0'
    if (NULL == node->data) {
        printf("Error in list_add_front: Couldn't allocate string\n");
        exit(1);
    }
    strcpy(node->data, data);
    ...
}
```

### 4.5 「Opaque Type」的意义

```c
typedef struct _linked_list linked_list_t;
```

> **头文件里只写「有这么一个类型」，但不告诉使用者里面是什么。**

**好处**：

| 优点 | 说明 |
|---|---|
| **封装** | 使用者不能直接访问 `head` |
| **可替换** | 内部实现改了，接口不变 |
| **安全性** | 防止误操作内部状态 |

**这就是 C 里模拟「私有成员」的方式。**

### 4.6 使用 ADT

```c
#include "linkedlist.h"
#include <stdio.h>

int main(void) {
    linked_list_t *list = list_create();
    if (NULL == list) {
        printf("Error in main: Couldn't allocate linked_list_t\n");
        return 1;
    }
    list_add_front(list, "Alice");
    list_add_front(list, "Bob");
    list_add_front(list, "Cara");
    list_print(list);
    // Cara -> Bob -> Alice -> NULL
    list_free(list);
    return 0;
}
```

### 4.7 编译与运行

```bash
$ ls
linkedlist.c  linkedlist.h  list_demo.c  Makefile

$ make
gcc -Wall -Wextra -std=c17 -c list_demo.c -o list_demo.o
gcc -Wall -Wextra -std=c17 -c linkedlist.c -o linkedlist.o
gcc -Wall -Wextra -std=c17 -o list_demo list_demo.o linkedlist.o

$ ./list_demo
Cara -> Bob -> Alice -> NULL
```

**编译选项说明：**

| 选项 | 作用 |
|---|---|
| `-Wall -Wextra` | **打开所有警告**（强烈推荐） |
| `-std=c17` | 指定标准 |
| `-c` | 只编译不链接，产出 `.o` |
| `-o` | 指定输出文件 |

---

## 五、本讲总结（课程原话）

> **这标志着 C 单元的结束！下周是浮点数和 RISC-V，请回来上课！**

| 主题 | 要点 |
|---|---|
| **IEC 前缀** | kibi、mebi、gibi、tebi、pebi、exbi、zebi、yobi |
| **位运算** | `&`、`\|`、`^`、`~` 对应 AND、OR、XOR、NOT |
| └ 特点 | **一次作用于所有位** |
| └ `&` | **掩码**：保留一些、把其余设为 0 |
| └ `\|` | 保留一些、把其余设为 1 |
| └ `^` | 保留一些、反转其余 |
| **移位** | 左移/右移 |

---

## 六、我的笔记与理解

### 6.1 本讲的三个主题其实是分开的

L07 把三个不太相关的东西塞在一起：

```
IEC 前缀      →  单位换算（2¹⁰ vs 10³）
位运算        →  二进制操作（掩码、移位）
ADT           →  软件工程（模块化、封装）
```

**它们共同的背景是「C 单元的收尾」**：把前面散落的知识点补齐。

### 6.2 三个透视是真精华

`AND / OR / XOR` 的「透视」理解，比真值表好记一百倍：

| 运算 | 一句话 | 用途 |
|---|---|---|
| **AND** | 「1 保留，0 清零」 | **提取**某些位 |
| **OR** | 「0 保留，1 置一」 | **设置**某些位 |
| **XOR** | 「0 保留，1 反转」 | **翻转**某些位 |

> **XOR 叫「条件反转器」是最妙的命名**——`a` 决定要不要反转 `b`。

### 6.2.5 一个位运算的实用速查

```c
x & 0xFF             // 提取低字节
x | 0xFF             // 低字节全置 1
x ^ 0xFF             // 低字节反转（= ~x & 0xFF）
x << 1               // ×2
x >> 1               // ÷2（无符号）
x & (1 << k)         // 测试第 k 位
x | (1 << k)         // 设置第 k 位
x & ~(1 << k)        // 清除第 k 位
x ^ (1 << k)         // 翻转第 k 位
```

### 6.3 左右移的本质区别

| | 左移 | 右移 |
|---|---|---|
| **数学意义** | × 2ⁿ | ÷ 2ⁿ（floor） |
| **填充** | 恒补 0 | 逻辑：补 0<br>算术：补符号位 |

> **左移只有一种，右移有两种。** 这是很多 bug 的来源。

**实际影响**：对负数右移，`>>` 在 C 里是实现定义（implementation-defined），不同编译器行为可能不同。**要可移植就用无符号数做位运算。**

### 6.4 为什么 32 位字长只能访问 4 GiB

```
32 位地址  →  2³² 个不同地址
          →  4,294,967,296 字节
          →  4 GiB（GiB 是 2³⁰ 字节）
```

**如果是 64 位：**

```
2⁶⁴ = 16 EiB
```

> **这就是为什么 32 位系统最多 4 GB 内存。**

### 6.5 ADT 是 C 的「面向对象」

C 没有 class，但用**「头文件 + opaque type + 前缀命名」**实现了同样的封装：

| OOP 概念 | C 的对应 |
|---|---|
| **class** | `struct` + 一组函数 |
| **private** | **opaque type**（头文件不暴露结构） |
| **方法** | `list_xxx(list, ...)` 形式 |
| **构造函数** | `list_create()` |
| **析构函数** | `list_free()` |
| **this 指针** | 第一个参数 `list` |

> **这是 C 项目组织代码的标准范式**，后面 RISC-V、xv6 都在用。

### 6.6 `-Wall -Wextra` 的必要性

```bash
gcc -Wall -Wextra -std=c17 ...
```

**不开这两个选项，很多 bug 编译器不会告诉你**：

- 未使用的变量
- 隐式类型转换
- **`void *` 解引用**（L06 提到的）

> **写 C 时，永远加 `-Wall -Wextra`。**

### 6.7 与后续的衔接

| 讲次 | 关系 |
|---|---|
| **L08** | 浮点数——**C 单元的最后一块** |
| **L09+** | RISC-V：位运算在硬件层面就是门电路 |
| **L16-19** | 数字电路：AND/OR/NOT 变成实际的逻辑门 |

> **L07 的位运算，在 L16 会变成「用门电路搭出来」。**

---

## 七、术语速查

| 术语 | 中文 | 说明 |
|---|---|---|
| **IEC prefix** | IEC 前缀 | 2¹⁰ 进制（Ki/Mi/Gi…） |
| **SI prefix** | 国际单位前缀 | 10³ 进制（K/M/G…） |
| **bitmask** | 位掩码 | 用于选择/修改特定位 |
| **logical shift** | 逻辑移位 | 补 0 |
| **arithmetic shift** | 算术移位 | 补符号位 |
| **ADT** | 抽象数据类型 | 接口与实现分离 |
| **opaque type** | 不透明类型 | 头文件只声明不暴露 |
| **include guard** | 头文件保护 | `#ifndef/#define/#endif` |
| **forward declaration** | 前向声明 | 先声明后定义 |

---

## 相关

- 课程导航：[[knowledge/course/index\|课程 目录导航]]
- 上一讲：[[knowledge/course/CS61C/06-Generics-Function-Pointers\|L06 Generics and Function Pointers]]
- 下一讲：[[knowledge/course/CS61C/08-Floating-Point\|L08 IEEE 754 Floating Point]]
