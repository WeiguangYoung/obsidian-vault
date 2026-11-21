---
{"dg-publish":true,"permalink":"/knowledge/course/csapp-labs/01-Data-Lab/","title":"CSAPP Lab1 · Data Lab","tags":["CSAPP","LAB","实验","位运算","补码","IEEE754"],"dg-note-properties":{"title":"CSAPP Lab1 · Data Lab","tags":["CSAPP","LAB","实验","位运算","补码","IEEE754"]}}
---


# Data Lab

13 道位运算谜题。前 10 题只能用 `! ~ & ^ | + << >>`，禁止 `if`／循环／宏／函数调用，常量不超过 `0xFF`；后 3 题允许控制结构，但禁止浮点类型与浮点运算。

| # | 函数 | 难度 | 运算符上限 | 实际用量 |
| ---: | --- | ---: | ---: | ---: |
| 1 | `bitXor` | 1 | 14 | 7 |
| 2 | `tmin` | 1 | 4 | 1 |
| 3 | `isTmax` | 1 | 10 | 7 |
| 4 | `allOddBits` | 2 | 12 | 7 |
| 5 | `negate` | 2 | 5 | 2 |
| 6 | `isAsciiDigit` | 3 | 15 | 11 |
| 7 | `conditional` | 3 | 16 | 8 |
| 8 | `isLessOrEqual` | 3 | 24 | 15 |
| 9 | `logicalNeg` | 4 | 12 | 5 |
| 10 | `howManyBits` | 4 | 90 | 33 |
| 11 | `floatScale2` | 4 | 30 | 14 |
| 12 | `floatFloat2Int` | 4 | 30 | 14 |
| 13 | `floatPower2` | 4 | 30 | 9 |

## 1. bitXor

仅用 `~` 和 `&` 实现 `x ^ y`

可用运算符：`~ &`　上限：14　难度：1

**思路**

异或的含义是"两位不同才为 1"，即**不是都为 0，也不是都为 1**。`~x & ~y` 筛出都为 0 的位，`x & y` 筛出都为 1 的位，两者都排除即可。因为 `|` 不允许使用，用德摩根律 `A | B = ~(~A & ~B)` 把"或"换成"与"。

```c
int bitXor(int x, int y) {
  int both = x & y;
  int neither = ~x & ~y;
  return ~neither & ~both;
}
```

## 2. tmin

返回最小的二进制补码整数

可用运算符：`! ~ & ^ | + << >>`　上限：4　难度：1

**思路**

补码最小值是 `0x80000000`，只有最高位为 1。`1 << 31` 直接构造，移位本身不涉及有符号溢出的问题。

```c
int tmin(void) {
  return 1 << 31;
}
```

## 3. isTmax

当且仅当 `x` 是最大的补码整数时返回 1

可用运算符：`! ~ & ^ | +`　上限：10　难度：1

**思路**

最大补码整数是 `0x7FFFFFFF`。它加 1 得 `0x80000000`（TMin），而 TMin 取反又回到 `0x7FFFFFFF`。所以令 `y = x+1`，判定 `~y == x`。

**但要排除 -1**：`-1 + 1 = 0`，`~0 = -1`，恰好也满足。所以额外要求 `y != 0`（即 `!!y`）。

```c
int isTmax(int x) {
  int y = x + 1;
  int z = ~y ^ x;
  return !z & !!y;
}
```

## 4. allOddBits

当且仅当 `x` 中所有奇数编号的位都为 1 时返回 1（位编号从 0 起）

可用运算符：`! ~ & ^ | + << >>`　上限：12　难度：2

**思路**

需要掩码 `0xAAAAAAAA`（所有奇数位为 1）。但常量不得超过 `0xFF`，所以从 `0xAA` 起，用移位和"或"拼出完整的 32 位掩码。

然后用 `~x & m` 取出"本该为 1 却为 0"的位，结果为 0 即全部合格。

```c
int allOddBits(int x) {
  int m = 0xAA;
  m = m | (m << 8);
  m = m | (m << 16);
  return !(~x & m);
}
```

## 5. negate

返回 `-x`

可用运算符：`! ~ & ^ | + << >>`　上限：5　难度：2

**思路**

补码取负的位级公式：全部取反再加 1，即 `~x + 1`。

```c
int negate(int x) {
  return ~x + 1;
}
```

## 6. isAsciiDigit

当且仅当 `0x30 <= x <= 0x39`（字符 0~9 的 ASCII 码）时返回 1

可用运算符：`! ~ & ^ | + << >>`　上限：15　难度：3

**思路**

拆成上下界两个判断：`x - 0x30 >= 0` 且 `0x39 - x >= 0`。

减法不可用，用 `~y + 1` 构造。右移 31 位取符号位（负为 -1、非负为 0），再用 `!` 翻转成 1/0，最后 `&` 取交集。

```c
int isAsciiDigit(int x) {
  int lower = x + (~0x30 + 1);
  int upper = 0x39 + (~x + 1);
  return !(lower >> 31) & !(upper >> 31);
}
```

## 7. conditional

与 `x ? y : z` 相同

可用运算符：`! ~ & ^ | + << >>`　上限：16　难度：3

**思路**

需要一个**全 0 或全 1 的掩码**来选值。

`!!x` 把任意非零压成 1、零保持 0；再取反加 1：`x != 0` 时得 `0xFFFFFFFF`，`x == 0` 时得 `0`。

最后 `(y & mask) | (z & ~mask)`，掩码全 1 时取 y，全 0 时取 z。

```c
int conditional(int x, int y, int z) {
  int mask = ~(!!x) + 1;
  return (y & mask) | (z & ~mask);
}
```

## 8. isLessOrEqual

当且仅当 `x <= y` 时返回 1，否则返回 0

可用运算符：`! ~ & ^ | + << >>`　上限：24　难度：3

**思路**

**直接算 `y - x` 会溢出**（如 `x = INT_MIN`），所以必须先分情况：

- **符号不同**：x 为负则必然 `x <= y`，直接返回 x 的符号位
- **符号相同**：减法不会溢出，看 `y - x` 的符号位，非负即成立

用 `sx ^ sy` 判断符号是否不同，再分别取两条分支。

```c
int isLessOrEqual(int x, int y) {
  int sx = (x >> 31) & 1;
  int sy = (y >> 31) & 1;
  int d = sx ^ sy;
  int sub = y + (~x + 1);
  int sn = (sub >> 31) & 1;
  return (d & sx) | ((d ^ 1) & (sn ^ 1));
}
```

## 9. logicalNeg

不使用 `!` 运算符计算 `!x`

可用运算符：`~ & ^ | + << >>`　上限：12　难度：4

**思路**

关键观察：**`x | (-x)` 的最高位一定为 1，当且仅当 `x != 0`**。

（x 为正，`-x` 最高位为 1；x 为负，x 自身最高位为 1。）

右移 31 位得 -1 或 0，加 1 即得 0 或 1。

```c
int logicalNeg(int x) {
  return ((x | (~x + 1)) >> 31) + 1;
}
```

## 10. howManyBits

用二进制补码表示 `x` 所需的最少位数

可用运算符：`! ~ & ^ | + << >>`　上限：90　难度：4

**思路**

正数需要"最高位 1 的位置 + 1 个符号位"；负数取反后同理，因为 `~x` 把负数的有效位模式变成正数。

先 `x = x ^ (x >> 31)`：正数不变，负数全部取反。

再用**二分法**找最高位 1 的位置：依次试探高 16 / 8 / 4 / 2 / 1 位是否有 1，有就累加位移量并把已处理部分右移掉。最后加上符号位本身。

```c
int howManyBits(int x) {
  int sign = x >> 31;
  int b16, b8, b4, b2, b1, b0;
  x = x ^ sign;
  b16 = (!!(x >> 16)) << 4;
  x = x >> b16;
  b8 = (!!(x >> 8)) << 3;
  x = x >> b8;
  b4 = (!!(x >> 4)) << 2;
  x = x >> b4;
  b2 = (!!(x >> 2)) << 1;
  x = x >> b2;
  b1 = (!!(x >> 1)) << 0;
  x = x >> b1;
  b0 = x;
  return b16 + b8 + b4 + b2 + b1 + b0 + 1;
}
```

## 11. floatScale2

对单精度浮点参数 `f`，返回 `2*f` 的等价位级表示

可用运算符：`任意整数运算 + if/while`　上限：30　难度：4

**思路**

按 IEEE 754 三种情况分别处理：

1. **特殊值**（exp 全 1，即 NaN/±INF）：×2 无意义，原样返回
2. **非规格化**（exp 全 0）：尾数左移一位即 ×2；若左移后进到 exp 位，自然过渡为最小的规格化数
3. **规格化**：exp 加 1 即 ×2；若 exp 加到 0xFF 说明溢出成无穷

```c
unsigned floatScale2(unsigned uf) {
  unsigned sign = uf & 0x80000000;
  unsigned exp = (uf >> 23) & 0xFF;
  unsigned frac = uf & 0x7FFFFF;
  if (exp == 0xFF) return uf;
  if (exp == 0) return sign | (frac << 1);
  exp = exp + 1;
  if (exp == 0xFF) return sign | 0x7F800000;
  return sign | (exp << 23) | frac;
}
```

## 12. floatFloat2Int

对单精度浮点参数 `f`，返回 `(int) f` 的等价位级表示；超出范围（含 NaN、无穷）返回 `0x80000000u`

可用运算符：`任意整数运算 + if/while`　上限：30　难度：4

**思路**

拆出 sign / exp / frac，补上隐含的 1（`frac | 0x800000`）。

- `exp < 0`：|f| < 1，向零截断得 0
- `exp > 30`：超出 int 范围（含 NaN、无穷），返回 `0x80000000`
- 否则按 `exp` 与 23 的大小关系左移或右移尾数，再按符号取负

**注意**：`-2147483648` 恰好可以表示，但 `exp > 30` 已把它排除在外——这是标准答案的边界取舍。

```c
int floatFloat2Int(unsigned uf) {
  unsigned sign = uf >> 31;
  int e = (uf >> 23) & 0xFF;
  int exp = e - 127;
  int frac = (uf & 0x7FFFFF) | 0x800000;
  if (exp < 0) return 0;
  if (exp > 30) return 0x80000000;
  if (exp > 23) frac = frac << (exp - 23);
  else frac = frac >> (23 - exp);
  if (sign) return -frac;
  return frac;
}
```

## 13. floatPower2

返回 `2.0^x` 的等价位级表示；过小返回 0，过大返回 +INF

可用运算符：`任意整数运算 + if/while`　上限：30　难度：4

**思路**

`2^x` 的规格化表示就是阶码为 `x+127`、尾数为 0。三种边界：

- `x > 127`：阶码溢出，返回 +INF（`0x7F800000`）
- `-126 <= x <= 127`：正常规格化，`(x+127) << 23`
- `-149 <= x < -126`：非规格化区，结果是最低位为 1 的 `1 << (x+149)`
- `x < -149`：太小，返回 0

```c
unsigned floatPower2(int x) {
  if (x > 127) return 0x7F800000;
  if (x >= -126) return (x + 127) << 23;
  if (x >= -149) return 1u << (x + 149);
  return 0;
}
```

> 以上 13 题均通过 `dlc` 规则检查与 `btest` 全部测试（36/36）。

---

## 附录：环境与踩坑

记录本次实际编译测试时遇到的问题，供复现参考。

### OpenCloudOS 9 没有 i686 包，`-m32` 编不了

`Makefile` 里写的是 `CFLAGS = -O -Wall -m32`，但系统装不上 32 位开发库：

```bash
dnf install -y glibc-devel.i686 libgcc.i686
```

```text
No match for argument: glibc-devel.i686
No match for argument: libgcc.i686
Error: Unable to find a match: glibc-devel.i686 libgcc.i686
```

`gcc -m32` 直接失败：

```text
/usr/bin/ld: cannot find crt1.o: No such file or directory
/usr/bin/ld: cannot find crti.o: No such file or directory
/usr/bin/ld: cannot find -lgcc
/usr/bin/ld: cannot find crtn.o: No such file or directory
```

**解决办法：直接不用 `-m32`，64 位编译。**

```bash
gcc -O -Wall -o btest bits.c btest.c decl.c tests.c -lm
```

**为什么可行**：x86-64 上 `int` 本身就是 32 位，Data Lab 的位操作（`1 << 31`、`~0`、`0xAAAAAAAA`）语义完全一致。实测 `btest` 和 `dlc` 都正常工作，13 题拿到 36/36。

**唯一要注意**：不要用 `long` 做 32 位假设（64 位下是 8 字节）。Data Lab 全程用 `int`，无影响。

### `dlc` 不允许显式类型转换

`floatFloat2Int` 和 `floatPower2` 里如果写了 `(int)` 或 `(unsigned)`，`dlc` 直接报错：

```text
dlc:bits.c:314:floatFloat2Int: Illegal cast
dlc:bits.c:320:floatFloat2Int: Illegal cast
dlc:bits.c:338:floatPower2: Illegal cast
```

**解决办法**：去掉所有 cast，靠隐式转换。

```c
/* 错误 */
int frac = (int)((uf & 0x7FFFFF) | 0x800000);
/* 正确 */
int frac = (uf & 0x7FFFFF) | 0x800000;
```

`dlc` 用的是 MIT CILK 小组魔改的 ANSI C 编译器，比 gcc 严格得多。

### `dlc` 对声明位置的要求比 C99 更严

所有声明必须在语句之前：

```c
int foo(int x) {
    int a = x;
    a *= 3;          /* 语句 */
    int b = a;       /* ERROR: Declaration not allowed here */
}
```

先集中声明，再写语句。

### 不要 `#include <stdio.h>`

会让 `dlc` 报一堆看不懂的错。想用 `printf` 调试可以直接调，gcc 会警告但能跑。

### 每次改完 `bits.c` 都要重新编译

`btest` 是独立可执行文件，不重新编译的话测的还是旧代码：

```bash
gcc -O -Wall -o btest bits.c btest.c decl.c tests.c -lm
./btest -f bitXor
```

### 官方 PDF 的题目表有笔误

`01-datalab.md` 早期版本收录的官方表格里：

| 原文 | 应为 |
| --- | --- |
| `bitXor(x,y)` 描述写 `x \|\| y` | `x ^ y` |
| `isTmax(x)` 描述写 `x x` | `x` |
| `isAsciDigit(x)` 拼写错误、不等式缺上界 | `isAsciiDigit`，`0x30 <= x <= 0x39` |
| `negate` 描述 "Return -x with using - operator" | 反向矛盾，`Legal ops` 里根本没有 `-` |

**一律以 `bits.c` 的注释为准。** `tests.c` 的参考实现可以作为交叉验证（比如 `bitXor` 那里写着 `return x^y;`）。

### 运算符计数怎么看

```bash
./dlc -e bits.c
```

```text
dlc:bits.c:149:bitXor: 7 operators
dlc:bits.c:158:tmin: 1 operators
dlc:bits.c:171:isTmax: 7 operators
...
```

只对通过规则检查的函数输出计数。如果某个函数报 Illegal ops，不会出现在这个列表里。
