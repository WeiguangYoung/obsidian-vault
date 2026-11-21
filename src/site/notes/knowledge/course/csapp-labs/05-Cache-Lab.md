---
{"dg-publish":true,"permalink":"/knowledge/course/csapp-labs/05-Cache-Lab/","title":"CSAPP Lab5 · Cache Lab","tags":["CSAPP","LAB","实验","缓存","矩阵转置","性能优化"],"dg-note-properties":{"title":"CSAPP Lab5 · Cache Lab","tags":["CSAPP","LAB","实验","缓存","矩阵转置","性能优化"]}}
---


# Cache Lab

两部分：写一个 cache 模拟器（Part A），再优化矩阵转置把 miss 降到下限（Part B）。

| 部分 | 目标 | 交付物 | 通过线 |
| --- | --- | --- | --- |
| A | 模拟 cache 的 hit/miss/eviction | `csim.c` | 与 `csim-ref` 结果完全一致 |
| B | 减少矩阵转置的 cache miss | `trans.c` | 32×32 < 300、64×64 < 1300、61×67 < 2000 |

缓存规格：**1 KB、直接映射、块大小 32 字节**。

## 环境准备

`test-trans` 依赖 valgrind 生成内存 trace，没装会报奇怪的错：

```text
Validation error at function 126! Run ./tracegen -M 32 -N 32 -F 0 for details.
```

`126` 是 shell 的"命令无法执行"（不是你的代码越界，别被误导）。装上即可：

```bash
dnf install -y valgrind      # RHEL/CentOS 系
apt install -y valgrind      # Debian 系
valgrind --version
```

```text
valgrind-3.21.0
```

另外 `driver.py` 是 Python 2 脚本（`print "..."` 语法），Python 3 下跑不了。想用总评分脚本需要装 python2，或者直接看 `test-csim` 和 `test-trans` 的单项输出。

---

## Part A：写 cache 模拟器

**目标**：实现 `csim.c`，读 valgrind 的 trace 文件，模拟 cache 行为。

**命令行参数**

```bash
./csim -s <组索引位数> -E <每组的行数> -b <块内偏移位数> -t <trace 文件> [-v]
```

| 参数 | 含义 |
| --- | --- |
| `-s` | 组索引位数，组数 `S = 2^s` |
| `-E` | 相联度（每组几行） |
| `-b` | 块偏移位数，块大小 `B = 2^b` |
| `-t` | trace 文件路径 |
| `-v` | 打印每条访问 |

**地址划分**

```text
 63                               s+b    s    b    0
┌──────────────────────────────────┬──────┬─────┐
│              tag                 │ set  │ off │
└──────────────────────────────────┴──────┴─────┘
```

```c
set = (addr >> b) & ((1 << s) - 1);
tag = addr >> (s + b);
/* 块内偏移 b 位不参与匹配 */
```

**trace 格式**

```text
 I 0400d7d4,8          ← 指令读取，忽略
 M 0421c7f0,4          ← modify：读+写，算两次访问
 L 04f6b868,8          ← load
 S 04f6b868,8          ← store
```

只处理以空格开头的行（指令取指行以 `I` 开头且不缩进），遇到 `I` 直接跳过，`M` 要访问两次。

**完整实现**

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <getopt.h>
#include "cachelab.h"

typedef struct {
    int valid;
    unsigned long tag;
    int lru;              /* 越大越久未用 */
} line_t;

static int s, E, b;
static int S;
static line_t **cache;
static int ticks;
static int hits, misses, evictions;

static void access(unsigned long addr)
{
    unsigned long set = (addr >> b) & ((1UL << s) - 1);
    unsigned long tag = addr >> (s + b);
    line_t *setp = cache[set];
    int i, victim = 0;

    ticks++;
    /* 命中 */
    for (i = 0; i < E; i++) {
        if (setp[i].valid && setp[i].tag == tag) {
            hits++;
            setp[i].lru = ticks;
            return;
        }
    }
    /* 未命中 */
    misses++;
    for (i = 0; i < E; i++) {
        if (!setp[i].valid) victim = i;
        else if (setp[i].lru < setp[victim].lru) victim = i;
    }
    if (setp[victim].valid) evictions++;
    setp[victim].valid = 1;
    setp[victim].tag = tag;
    setp[victim].lru = ticks;
}

int main(int argc, char **argv)
{
    int opt, verbose = 0;
    char *path = NULL, op;
    unsigned long addr;
    int size, i;
    FILE *fp;
    char line[256];

    while ((opt = getopt(argc, argv, "hvs:E:b:t:")) != -1) {
        switch (opt) {
        case 'v': verbose = 1; break;
        case 's': s = atoi(optarg); break;
        case 'E': E = atoi(optarg); break;
        case 'b': b = atoi(optarg); break;
        case 't': path = optarg; break;
        default: return 1;
        }
    }
    if (!path) return 1;

    S = 1 << s;
    cache = malloc(sizeof(line_t *) * S);
    for (i = 0; i < S; i++) cache[i] = calloc(E, sizeof(line_t));

    fp = fopen(path, "r");
    if (!fp) return 1;
    while (fgets(line, sizeof line, fp)) {
        if (line[0] != ' ') continue;                 /* 跳过指令取指 */
        if (sscanf(line, " %c %lx,%d", &op, &addr, &size) != 3) continue;
        if (op == 'I') continue;
        access(addr);
        if (op == 'M') access(addr);                  /* modify = load + store */
        if (verbose) printf("%c %lx,%d\n", op, addr, size);
    }
    fclose(fp);
    printSummary(hits, misses, evictions);
    return 0;
}
```

**关键点**

1. **`M` 访问算两次** —— 先 load 再 store，命中的话 `hits += 2`，未命中的话 `misses++` 然后 `hits++`
2. **`I` 指令行必须跳过** —— 指令取指不占数据缓存
3. **只处理首字符是空格的行** —— valgrind 的指令行格式是 `I 0400d7d4,8`（无前导空格）
4. **LRU 用时间戳** —— `ticks` 单调递增，替换时选 `lru` 最小的行

**验证**

```bash
gcc -g -Wall -Werror -std=c99 -o csim csim.c cachelab.c -lm
```

逐个 trace 与 `csim-ref`（官方参考答案）比对：

```bash
for t in traces/*.trace; do
  echo "=== $t ==="
  ./csim -s 2 -E 1 -b 4 -t $t | tail -1
  ./csim-ref -s 2 -E 1 -b 4 -t $t | tail -1
done
```

```text
=== traces/dave.trace ===
hits:2 misses:3 evictions:1
hits:2 misses:3 evictions:1
=== traces/long.trace ===
hits:234988 misses:51976 evictions:51972
hits:234988 misses:51976 evictions:51972
=== traces/trans.trace ===
hits:193 misses:45 evictions:41
hits:193 misses:45 evictions:41
=== traces/yi2.trace ===
hits:16 misses:1 evictions:0
hits:16 misses:1 evictions:0
=== traces/yi.trace ===
hits:4 misses:5 evictions:3
hits:4 misses:5 evictions:3
```

**5 个 trace 全部逐位一致**，Part A 完成。

> `long.trace` 有 23 万次命中，是最有说服力的一个。如果 LRU 或 tag 划分写错，这个数字不可能对上。

---

## Part B：优化矩阵转置

**目标**：实现 `B = A^T`，在 1 KB 直接映射缓存（块 32 字节 = 8 个 int）下把 miss 降到最低。

**评分线**

| 矩阵 | 满分线 | 及格线 | 简单实现 |
| --- | ---: | ---: | ---: |
| 32×32 | < 300 | — | 1183 |
| 64×64 | < 1300 | — | 4723 |
| 61×67 | < 2000 | — | 4423 |

朴素的双重循环（`for i: for j: B[j][i] = A[i][j]`）对 B 是**列访问**，每次跳 32×4=128 字节，几乎每次都 miss。

**为什么缓存参数重要**

- 块 32 字节 = 8 个 int → **一次读入 8 个连续元素**
- 总容量 1 KB = 32 行 × 32 字节 → **共 32 组**（直接映射，`S=32, E=1, B=32`）
- 一行 8 个 int，共 32 行 → 能放下 **32×32 矩阵的一半**

**关键矛盾**：A 按行访问（连续，命中好），B 按列写入（跳跃，每次都 miss）。而且 A 和 B 的同一列索引会**映射到同一组**，造成冲突驱逐。

### 32×32：8×8 分块

把矩阵切成 8×8 的小块，每块正好覆盖缓存的 8 个块。块内先把 A 的一行读到局部变量，再写到 B 的对应列。

```c
char transpose_submit_desc[] = "Transpose submission";
void transpose_submit(int M, int N, int A[N][M], int B[M][N])
{
    int i, j, ii, t0, t1, t2, t3, t4, t5, t6, t7;

    for (i = 0; i < N; i += 8) {
        for (j = 0; j < M; j += 8) {
            for (ii = i; ii < i + 8; ii++) {
                t0 = A[ii][j+0]; t1 = A[ii][j+1];
                t2 = A[ii][j+2]; t3 = A[ii][j+3];
                t4 = A[ii][j+4]; t5 = A[ii][j+5];
                t6 = A[ii][j+6]; t7 = A[ii][j+7];
                B[j+0][ii] = t0; B[j+1][ii] = t1;
                B[j+2][ii] = t2; B[j+3][ii] = t3;
                B[j+4][ii] = t4; B[j+5][ii] = t5;
                B[j+6][ii] = t6; B[j+7][ii] = t7;
            }
        }
    }
}
```

**为什么用 8 个局部变量**

如果直接写 `B[j][ii] = A[ii][j]`，编译器会生成"读 A → 写 B → 读 A → 写 B"的交替序列，每次切到 B 都可能把 A 的缓存行挤掉。

先把整行 8 个元素读进寄存器（`t0`~`t7`），再一次性写 B，**把 16 次内存访问压缩成 2 次缓存行搬运**。

**效果**

```text
Summary for official submission (func 0): correctness=1 misses=288
```

**288** < 300，达标。

### 64×64：8×8 分块 + 四块细分

64×64 更难，因为**一行 64 个 int = 256 字节 = 8 个缓存块**，而缓存只有 32 组。A 的第 i 行和 B 的第 i 行会映射到同一组，8×8 直接分块会因为组冲突反复驱逐。

标准解法是"**先搬 4 行到 B 的上半，中途借 B 当临时区**"：

```c
} else if (M == 64 && N == 64) {
    for (j = 0; j < M; j += 8) {          /* 注意：j 在外，i 在内（换序） */
        for (i = 0; i < N; i += 8) {
            /* 步骤 1：A 的左上 4×8 → B 的右上 4×8，暂存到 B 左上 */
            for (ii = i; ii < i + 4; ii++) {
                t0 = A[ii][j+0]; t1 = A[ii][j+1];
                t2 = A[ii][j+2]; t3 = A[ii][j+3];
                t4 = A[ii][j+4]; t5 = A[ii][j+5];
                t6 = A[ii][j+6]; t7 = A[ii][j+7];
                B[j+0][ii] = t0; B[j+1][ii] = t1;
                B[j+2][ii] = t2; B[j+3][ii] = t3;
                B[j+0][ii+4] = t4; B[j+1][ii+4] = t5;   /* 暂存 */
                B[j+2][ii+4] = t6; B[j+3][ii+4] = t7;
            }
            /* 步骤 2：交换 B 左上暂存区 与 A 的左下块 */
            for (jj = j; jj < j + 4; jj++) {
                t0 = A[i+4][jj]; t1 = A[i+5][jj];
                t2 = A[i+6][jj]; t3 = A[i+7][jj];
                t4 = B[jj][i+4]; t5 = B[jj][i+5];
                t6 = B[jj][i+6]; t7 = B[jj][i+7];
                B[jj][i+4] = t0; B[jj][i+5] = t1;
                B[jj][i+6] = t2; B[jj][i+7] = t3;
                B[jj+4][i+0] = t4; B[jj+4][i+1] = t5;
                B[jj+4][i+2] = t6; B[jj+4][i+3] = t7;
            }
            /* 步骤 3：A 的右下 4×4 → B 的右下 */
            for (ii = i + 4; ii < i + 8; ii++) {
                t4 = A[ii][j+4]; t5 = A[ii][j+5];
                t6 = A[ii][j+6]; t7 = A[ii][j+7];
                B[j+4][ii] = t4; B[j+5][ii] = t5;
                B[j+6][ii] = t6; B[j+7][ii] = t7;
            }
        }
    }
}
```

**三步的用意**

```text
A 块             B 块
┌────┬────┐    ┌────┬────┐
│ ①  │ ③  │    │    │ ①' │   ①: A 左上 → B 右上（经 B 左上暂存）
├────┼────┤ →  ├────┼────┤   ②: A 左下 → B 右上，暂存换出到 B 左下
│ ②  │    │    │ ②' │ ③' │   ③: A 右下 → B 右下
└────┴────┘    └────┴────┘
```

如果只做 8×8 一次搬运，A 的右半列会撞上 B 的同组行，反复踢出。**用 B 的左上角当临时缓冲区**，把"读 A 右下 + 写 B 左下"这两个冲突的访问错开。

**换序的细节**：`j` 放在外层、`i` 放内层，是因为按列块遍历时 B 的写是连续的。

**效果**

```text
Summary for official submission (func 0): correctness=1 misses=1180
```

**1180** < 1300，达标。

### 61×67：16×16 分块

非 8 的倍数，用 16×16 分块 + 边界检查：

```c
} else {
    for (i = 0; i < N; i += 16) {
        for (j = 0; j < M; j += 16) {
            for (ii = i; ii < i + 16 && ii < N; ii++) {
                for (jj = j; jj < j + 16 && jj < M; jj++) {
                    B[jj][ii] = A[ii][jj];
                }
            }
        }
    }
}
```

61×67 尺寸不规整，缓存冲突没 64×64 那么致命，简单分块就够。

**效果**

```text
Summary for official submission (func 0): correctness=1 misses=1993
```

**1993** < 2000，达标。

### 三组结果汇总

| 矩阵 | 目标 | 实测 | 结果 |
| --- | ---: | ---: | --- |
| 32×32 | < 300 | **288** | ✅ |
| 64×64 | < 1300 | **1180** | ✅ |
| 61×67 | < 2000 | **1993** | ✅ |

三组全部 `correctness=1`（转置结果正确），全部达标。

### 测试命令

```bash
gcc -g -Wall -Werror -std=c99 -O0 -c trans.c
gcc -g -Wall -Werror -std=c99 -o test-trans test-trans.c cachelab.c trans.o
gcc -g -Wall -Werror -std=c99 -O0 -o tracegen tracegen.c trans.o cachelab.c

./test-trans -M 32 -N 32
./test-trans -M 64 -N 64
./test-trans -M 61 -N 67
```

`-O0` 是必须的：优化会改变内存访问模式，测出来的 miss 数就不是你代码的真实表现。Makefile 里已经写死 `-O0`，自己手动编译时别忘。

---

## 常用命令

**Part A**

```bash
gcc -g -Wall -Werror -std=c99 -o csim csim.c cachelab.c -lm
./csim -s 2 -E 1 -b 4 -t traces/yi.trace
./csim-ref -s 2 -E 1 -b 4 -t traces/yi.trace   # 参考答案，用来比对
./test-csim                                     # 官方评测脚本
```

`test-csim` 会跑多种 `(s, E, b)` 组合，每项 1 分，共 27 分。

**Part B**

```bash
make clean && make
./test-trans -M 32 -N 32
./tracegen -M 32 -N 32 -F 0      # 单独生成某个函数的 trace
```

**看 trace 内容**

```bash
head -20 traces/yi.trace
```

```text
 L 10,1
 M 20,1
 L 22,1
 S 18,1
 L 110,1
 L 210,1
 M 12,1
```

不写结尾逗号后面那个大小（单位字节），模拟器只需要地址。

---

## 常见问题

**`Validation error at function 126`**

不是代码错误，是 **valgrind 没装**，`system()` 返回 `126`（命令无法执行）。`dnf install -y valgrind` 解决。

**`/usr//bin/python: not found`**

`driver.py` 是 Python 2 脚本（`print "..."`）。要么装 python2，要么直接看 `test-csim` / `test-trans` 的单项输出。

**`-Werror` 编译失败**

`CFLAGS` 里有 `-Wall -Werror`，任何警告都当错误。常见是未使用变量、隐式类型转换。

**miss 数明显偏高**

检查 `trans.c` 是否被 `-O0` 编译。用 `-O2` 时编译器可能把循环重排，测出的数据没有意义。

**32×32 冲不到 300 以下**

检查是否真的用了 8 个局部变量。直接 `B[j][ii] = A[ii][j]` 会因为在 A/B 之间反复切换而超线。

---

## 附录：环境与踩坑

### `Validation error at function 126` —— 不是代码问题

这是我在这份 lab 上遇到的最迷惑的报错：

```bash
./test-trans -M 32 -N 32
```

```text
Validation error at function 126! Run ./tracegen -M 32 -N 32 -F 0 for details.
Skipping performance evaluation for this function.
Summary for official submission (func 0): correctness=0 misses=2147483647
```

`126` 不是"第 126 个函数"（`MAX_TRANS_FUNCS` 只有 100，看 `cachelab.c` 就知道）。它是 `system()` 的返回值：

```c
flag = WEXITSTATUS(system(cmd));
if (0 != flag) {
    printf("Validation error at function %d!\n", flag-1);   /* flag-1 = 126 */
```

`126` 是 shell 的"**命令无法执行**"，即 `cmd` 里的 `valgrind` 不存在。

确认：

```bash
which valgrind
# 空
```

装：

```bash
dnf install -y valgrind      # RHEL/CentOS 系
apt install -y valgrind      # Debian 系
valgrind --version
```
```text
valgrind-3.21.0
```

装完三组矩阵立刻正常输出 miss 数。

**教训**：看到奇怪的大数字（126、127、2147483647）先别怀疑自己代码，`126`/`127` 是 shell 的固定错误码（命令无法执行 / 命令未找到），`2147483647` 是 `INT_MAX`，表示"没测成"。

### `driver.py` 是 Python 2 脚本

```python
#!/usr//bin/python
...
print "Part A: Testing cache simulator"
```

两个问题：

1. `#!/usr//bin/python`（多一个斜杠），而且系统通常只有 `python3`
2. `print "..."` 是 Python 2 语法，Python 3 直接 `SyntaxError`

`driver.py` 是总评分脚本（Part A + Part B 一起算总分）。不想折腾的话，直接看单项输出：

```bash
./test-csim          # Part A 得分
./test-trans -M 32 -N 32   # Part B 逐组 miss 数
```

想用总评分：

```bash
dnf install -y python2     # 或 apt install python2
sed -i '1s|.*|#!/usr/bin/env python2|' driver.py
```

### `-O0` 编译 `trans.c` 是必须的

`Makefile` 里写死了：

```makefile
trans.o: trans.c
	$(CC) $(CFLAGS) -O0 -c trans.c
```

手动编译时如果忘了加 `-O0`，编译器会做循环展开、指令重排，**内存访问模式完全变了**，测出的 miss 数没有意义（可能比真实值低很多，看起来"优化得很好"，其实是编译器帮你优化的）。

**自己编译的完整命令**：

```bash
gcc -g -Wall -Werror -std=c99 -O0 -c trans.c
gcc -g -Wall -Werror -std=c99 -o test-trans test-trans.c cachelab.c trans.o
gcc -g -Wall -Werror -std=c99 -O0 -o tracegen tracegen.c trans.o cachelab.c
```

### `-Werror` 让警告变错误

```makefile
CFLAGS = -g -Wall -Werror -std=c99 -m64
```

任何警告都编译失败。常见触发点：

- 未使用的变量（`int tmp;` 声明了没用）
- 有符号/无符号比较（`int i` 和 `size_t` 比）
- 隐式类型转换

**不要靠删 `-Werror` 绕过**，按提示改代码。评分时会用原始 Makefile，改坏了要还原。

### `-m64` 是硬性要求

```makefile
CFLAGS = -g -Wall -Werror -std=c99 -m64
```

这个 lab 必须在 64 位下跑（指针 8 字节）。如果环境只有 32 位，Makefile 会失败，而且 `tracegen` 生成的地址格式也会对不上。

### Part A：`M` 指令必须算两次访问

trace 里的 `M` 表示 modify（读后写），一次 `M` 产生两次缓存访问：

```c
access(addr);
if (op == 'M') access(addr);   /* 第二次 */
```

漏了这行，`traces/trans.trace` 的统计会是 `hits:193 misses:45 evictions:41` 里的 hits 少算一半，对不上 `csim-ref` 的输出。

**自检方法**：

```bash
./csim -s 2 -E 1 -b 4 -t traces/trans.trace     | tail -1
./csim-ref -s 2 -E 1 -b 4 -t traces/trans.trace | tail -1
```

必须逐位相同。`long.trace`（23 万次命中）是最好的验证用例 —— LRU 或 tag 划分错一点，这个数字就对不上。

### Part A：`I` 指令要跳过

```text
I 0400d7d4,8          ← 指令取指，不占数据缓存
 M 0421c7f0,4         ← 注意这行有前导空格
 L 04f6b868,8
```

两个特征：

1. `I` 行**没有前导空格**，数据访问行**有前导空格**
2. `I` 是取指令，不该计入数据缓存统计

所以过滤条件：

```c
if (line[0] != ' ') continue;      /* 跳过所有指令取指行 */
```

比判断 `op == 'I'` 更简单可靠。

### Part B 32×32：不用局部变量冲不到 300

朴素写法：

```c
for (i = 0; i < N; i += 8)
    for (j = 0; j < M; j += 8)
        for (ii = i; ii < i+8; ii++)
            for (jj = j; jj < j+8; jj++)
                B[jj][ii] = A[ii][jj];
```

即使分了 8×8 块，仍然是"A 读一个 → B 写一个"交替，每次切到 B 都可能把 A 的缓存行挤掉。实测会超过 300。

**必须用 8 个局部变量把整行读出来再写**：

```c
t0 = A[ii][j+0]; t1 = A[ii][j+1]; ... t7 = A[ii][j+7];
B[j+0][ii] = t0; B[j+1][ii] = t1; ... B[j+7][ii] = t7;
```

这样 16 次内存访问压缩成 2 次缓存行搬运，288 < 300。

### Part B 64×64：换序是关键

64×64 比 32×32 难，因为一行 64 个 int = 256 字节 = **8 个缓存块**，而缓存只有 32 组。A 的第 i 行和 B 的第 i 行会撞同一组。

两个必须做的：

1. **`j` 放外层、`i` 放内层**（和 32×32 相反）
2. **用 B 的左上角当临时缓冲区**，把"读 A 右下 + 写 B 左下"这两个冲突访问错开

第一版按 32×32 的思路直接套，得到 1180+ 甚至更差。改成"三步走"后降到 1180。

其实 1180 仍然不是最优（有人能做到 1179 以下），但已经过线（< 1300）。

### `test-trans` 一次只能测一组尺寸

```bash
./test-trans -M 32 -N 32
./test-trans -M 64 -N 64
./test-trans -M 61 -N 67
```

三个尺寸要分别跑，因为 `transpose_submit` 里按 `M`/`N` 分支处理。

**每次跑之前先 `make`**，否则测的是旧的 `trans.o`。`test-trans` 依赖 `trans.o`，而 `trans.o` 是独立编译的：

```bash
make clean && make
```

### `free-port.sh` / `port-for-user.pl` 不存在

那是 proxylab 的文件。cachelab 不需要端口。

### 实测数据汇总

| 矩阵 | 目标 | 实测 | 说明 |
| --- | ---: | ---: | --- |
| 32×32 | < 300 | **288** | 8 局部变量 |
| 64×64 | < 1300 | **1180** | 三步搬运 + j/i 换序 |
| 61×67 | < 2000 | **1993** | 16×16 分块 + 边界检查 |

61×67 的 1993 离 2000 只有 7 个 miss 的余量，比较惊险。尺寸不规整时缓存冲突没 64×64 那么致命，简单分块够用，但**没有优化空间了** —— 想更稳需要更细的分块策略。
