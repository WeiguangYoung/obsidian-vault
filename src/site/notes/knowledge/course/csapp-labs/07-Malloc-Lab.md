---
{"dg-publish":true,"permalink":"/knowledge/course/csapp-labs/07-Malloc-Lab/","title":"CSAPP Lab7 · Malloc Lab","tags":["CSAPP","LAB","实验","内存分配","堆","空闲链表"],"dg-note-properties":{"title":"CSAPP Lab7 · Malloc Lab","tags":["CSAPP","LAB","实验","内存分配","堆","空闲链表"]}}
---


# Malloc Lab

实现自己的 `mm_malloc` / `mm_free` / `mm_realloc`，目标是**正确**、**省空间**、**跑得快**。

评分公式：

```c
Perf index = 60 * util + 40 * thru
```

`util` 是内存利用率，`thru` 是吞吐量。两者加权，想拿高分必须兼顾。

**本实验包的情况**：`traces/` 目录缺失，只有 `short1-bal.rep`、`short2-bal.rep` 两个 trace。下面所有实测数据都基于这两个文件。

---

## 环境准备

**问题 1：`Makefile` 用 `-m32`，本机无 32 位库**

```text
/usr/include/gnu/stubs.h:7:11: fatal error: gnu/stubs-32.h: No such file or directory
```

改动：

```makefile
# 原来
CFLAGS = -Wall -O2 -m32
# 改成
CFLAGS = -Wall -O2 -g
```

**问题 2：64 位下指针是 8 字节**

如果按 32 位教材的写法（`WSIZE=4`、`pred/succ` 各 4 字节），在 64 位机器上会把块结构写崩，直接 core dump。

64 位适配要点：
- 头部/脚部宽度 `WSIZE = 8`
- 空块里的 `pred` / `succ` 各 8 字节
- 最小块 = 8(头) + 16(pred+succ) + 8(脚) = **32 字节**

**问题 3：对齐**

`ALIGNMENT` 在 `config.h` 里是 8，但 64 位下块大小要按 16 对齐才安全（头脚成对出现，天然 16 的倍数）。

---

## 一、设计思路演进

教材讲了三种方案，性能依次提升：

| 方案 | 分配复杂度 | 特点 |
| --- | --- | --- |
| 隐式空闲链表 | O(n) | 靠头部遍历，简单但慢 |
| 显式空闲链表 | O(空闲块数) | 空块里存前驱后继指针，跳过已分配块 |
| 分离空闲链表 | 接近 O(1) | 按大小分级，先在小尺寸里找 |

本实现用**显式空闲链表 + 首次适配**，是性价比最高的方案。

### 块格式

**已分配块**

```text
┌──────────┬─────────────────────┐
│ header 8B│      payload        │
└──────────┴─────────────────────┘
```

**空闲块**（payload 头部塞进了两个指针）

```text
┌──────────┬────────┬────────┬──────────┬──────────┐
│ header 8B│ pred 8B│ succ 8B│  剩余空间 │ footer 8B│
└──────────┴────────┴────────┴──────────┴──────────┘
```

`footer` 的作用是支持**向后合并**：给定当前块，读自己前面的 8 字节就知道前一块的大小。

### header 的位布局

```text
 63                    4   3   2   1   0
┌───────────────────────┬───┬───┬───┬───┐
│        块大小          │ 0 │ 0 │ A │ P │
└───────────────────────┴───┴───┴───┴───┘
                                │   └── bit0: 本块已分配
                                └────── bit1: 前一块已分配
```

块大小按 16 对齐，所以低 4 位空出来放标志。

---

## 二、完整实现

```c
#include <stdio.h>
#include <stdlib.h>
#include <assert.h>
#include <unistd.h>
#include <string.h>

#include "mm.h"
#include "memlib.h"

team_t team = {
    "bovik",
    "Harry Bovik",
    "bovik@cs.cmu.edu",
    "",
    ""
};

#define WSIZE       8               /* 头部/脚部宽度 */
#define DSIZE       16              /* 头 + 脚 */
#define CHUNKSIZE   (1 << 12)

#define MAX(x, y) ((x) > (y) ? (x) : (y))
#define ALIGN(size) (((size) + (DSIZE - 1)) & ~(DSIZE - 1))

#define PACK(size, alloc) ((size) | (alloc))

#define GET(p)      (*(size_t *)(p))
#define PUT(p, val) (*(size_t *)(p) = (val))

#define GET_SIZE(p)  (GET(p) & ~(size_t)0xF)
#define GET_ALLOC(p) (GET(p) & 0x1)

#define HDRP(bp) ((char *)(bp) - WSIZE)
#define FTRP(bp) ((char *)(bp) + GET_SIZE(HDRP(bp)) - DSIZE)

#define NEXT_BLKP(bp) ((char *)(bp) + GET_SIZE(((char *)(bp) - WSIZE)))
#define PREV_BLKP(bp) ((char *)(bp) - GET_SIZE(((char *)(bp) - DSIZE)))

#define PRED_P(bp) ((char *)(bp))
#define SUCC_P(bp) ((char *)(bp) + WSIZE)

#define GET_PRED(bp)    (*(char **)PRED_P(bp))
#define GET_SUCC(bp)    (*(char **)SUCC_P(bp))
#define SET_PRED(bp, p) (*(char **)PRED_P(bp) = (char *)(p))
#define SET_SUCC(bp, p) (*(char **)SUCC_P(bp) = (char *)(p))

#define MIN_BLOCK 32                /* 8(头) + 16(pred/succ) + 8(脚) */

static char *heap_listp;
static void *free_listp;

static void *extend_heap(size_t words);
static void *coalesce(void *bp);
static void *find_fit(size_t asize);
static void place(void *bp, size_t asize);
static void insert_free(void *bp);
static void remove_free(void *bp);
```

### mm_init

```c
int mm_init(void)
{
    if ((heap_listp = mem_sbrk(4 * WSIZE)) == (void *)-1)
        return -1;
    PUT(heap_listp, 0);                              /* 对齐填充 */
    PUT(heap_listp + (1 * WSIZE), PACK(DSIZE, 1));   /* 序言头 */
    PUT(heap_listp + (2 * WSIZE), PACK(DSIZE, 1));   /* 序言脚 */
    PUT(heap_listp + (3 * WSIZE), PACK(0, 1));       /* 结尾头 */
    heap_listp += (2 * WSIZE);
    free_listp = NULL;

    if (extend_heap(CHUNKSIZE / WSIZE) == NULL)
        return -1;
    return 0;
}
```

**序言块和结尾块的作用**

```text
堆布局：
┌────────┬──────────┬──────────┬────────┬─────────────────┬────────┐
│ 填充   │ 序言头   │ 序言脚   │ 块1    │      ...        │ 结尾头 │
│   0    │ 16/1     │ 16/1     │        │                 │  0/1   │
└────────┴──────────┴──────────┴────────┴─────────────────┴────────┘
         ↑ heap_listp                            
```

- **序言块**：永远已分配、大小固定，作用是让第一个真实块的 `PREV_BLKP` 不会越界（它读到序言脚）
- **结尾块**：大小为 0、已分配，作用是让最后一个块的 `NEXT_BLKP` 不会越界

没有这两个哨兵，`coalesce` 里读前后块的头部就会踩到非法内存。

### extend_heap —— 扩展堆

```c
static void *extend_heap(size_t words)
{
    char *bp;
    size_t size;

    size = ALIGN(words * WSIZE);
    if (size < MIN_BLOCK) size = MIN_BLOCK;

    if ((long)(bp = mem_sbrk(size)) == -1)
        return NULL;

    PUT(HDRP(bp), PACK(size, 0));            /* 空闲块头 */
    PUT(FTRP(bp), PACK(size, 0));            /* 空闲块脚 */
    SET_PRED(bp, NULL);
    SET_SUCC(bp, NULL);
    PUT(HDRP(NEXT_BLKP(bp)), PACK(0, 1));    /* 新的结尾头 */

    return coalesce(bp);                     /* 立刻尝试与前面的空闲块合并 */
}
```

**注意 `PUT(HDRP(NEXT_BLKP(bp)), PACK(0, 1))`** —— 原来的结尾头被新块占用了，必须在新的堆末尾补一个。漏了这行，`coalesce` 往后读就会读到垃圾。

### insert_free / remove_free —— 维护空闲链表

```c
static void insert_free(void *bp)
{
    SET_PRED(bp, NULL);
    SET_SUCC(bp, free_listp);
    if (free_listp) SET_PRED(free_listp, bp);
    free_listp = bp;                       /* 头插，O(1) */
}

static void remove_free(void *bp)
{
    char *pred = GET_PRED(bp);
    char *succ = GET_SUCC(bp);
    if (pred) SET_SUCC(pred, succ);
    else free_listp = succ;                /* bp 是链表头 */
    if (succ) SET_PRED(succ, pred);
}
```

**双向链表的意义**：`remove_free` 要 O(1)。如果只有单向，删除时必须从头遍历找前驱，退化成 O(n)。

### coalesce —— 四情况合并

```c
static void *coalesce(void *bp)
{
    size_t prev_alloc = GET_ALLOC(FTRP(PREV_BLKP(bp)));
    size_t next_alloc = GET_ALLOC(HDRP(NEXT_BLKP(bp)));
    size_t size = GET_SIZE(HDRP(bp));

    if (prev_alloc && next_alloc) {              /* ① 两侧都占用 */
        insert_free(bp);
        return bp;
    } else if (prev_alloc && !next_alloc) {      /* ② 只合并下一块 */
        size += GET_SIZE(HDRP(NEXT_BLKP(bp)));
        remove_free(NEXT_BLKP(bp));
    } else if (!prev_alloc && next_alloc) {      /* ③ 只合并上一块 */
        size += GET_SIZE(HDRP(PREV_BLKP(bp)));
        bp = PREV_BLKP(bp);
        remove_free(bp);
    } else {                                     /* ④ 两侧都空闲 */
        size += GET_SIZE(HDRP(PREV_BLKP(bp))) +
                GET_SIZE(HDRP(NEXT_BLKP(bp)));
        remove_free(PREV_BLKP(bp));
        remove_free(NEXT_BLKP(bp));
        bp = PREV_BLKP(bp);
    }
    PUT(HDRP(bp), PACK(size, 0));
    PUT(FTRP(bp), PACK(size, 0));
    SET_PRED(bp, NULL);
    SET_SUCC(bp, NULL);
    insert_free(bp);
    return bp;
}
```

**为什么判断前块用 `FTRP(PREV_BLKP(bp))`**

前块的头部在 `bp - 8 - prev_size`，但要先知道 `prev_size` 才能算地址——鸡生蛋问题。而 `bp - 8` 正好是**前块的脚部**，那里也存了 size 和 alloc 位，直接读就行。

这就是为什么必须有 footer。

**④ 情况最容易写错**：合并后 `bp` 要更新成 `PREV_BLKP(bp)`，因为新块的起始位置提前了。而且两个邻居都要先从空闲链表摘掉，否则链表里会有指向已合并区域的悬空节点。

### find_fit —— 首次适配

```c
static void *find_fit(size_t asize)
{
    char *bp;
    for (bp = free_listp; bp != NULL; bp = GET_SUCC(bp)) {
        if (GET_SIZE(HDRP(bp)) >= asize)
            return bp;
    }
    return NULL;
}
```

遍历空闲链表，第一个够大的就用。优于"最佳适配"（要扫完整个链表）在于吞吐快，代价是可能浪费空间。

### place —— 分配并可能拆分

```c
static void place(void *bp, size_t asize)
{
    size_t csize = GET_SIZE(HDRP(bp));
    remove_free(bp);

    if ((csize - asize) >= MIN_BLOCK) {      /* 剩得够多，拆 */
        PUT(HDRP(bp), PACK(asize, 1));
        PUT(FTRP(bp), PACK(asize, 1));
        bp = NEXT_BLKP(bp);
        PUT(HDRP(bp), PACK(csize - asize, 0));
        PUT(FTRP(bp), PACK(csize - asize, 0));
        SET_PRED(bp, NULL);
        SET_SUCC(bp, NULL);
        insert_free(bp);
    } else {                                  /* 剩太少，整块给出去 */
        PUT(HDRP(bp), PACK(csize, 1));
        PUT(FTRP(bp), PACK(csize, 1));
    }
}
```

**`MIN_BLOCK` 的判断是关键**

如果剩余空间小于 32 字节（放不下头+pred+succ+脚），拆出来的"块"根本没法当空闲块用。这时宁可整块给出去，**牺牲一点内部碎片换取正确性**。

### mm_malloc

```c
void *mm_malloc(size_t size)
{
    size_t asize;
    size_t extendsize;
    char *bp;

    if (size == 0)
        return NULL;

    if (size <= 2 * WSIZE)              /* 16 字节以内 */
        asize = MIN_BLOCK;
    else
        asize = ALIGN(size + DSIZE);    /* payload + 头脚，再对齐 */

    if ((bp = find_fit(asize)) != NULL) {
        place(bp, asize);
        return bp;
    }

    extendsize = MAX(asize, CHUNKSIZE); /* 至少扩一个 CHUNKSIZE */
    if ((bp = extend_heap(extendsize / WSIZE)) == NULL)
        return NULL;
    place(bp, asize);
    return bp;
}
```

**`size <= 2*WSIZE` 的特判**

请求 8 字节时，`ALIGN(8 + 16) = ALIGN(24) = 32 = MIN_BLOCK`，其实结果一样。但显式写明可以避免 `ALIGN` 边界上的意外。

### mm_free

```c
void mm_free(void *ptr)
{
    size_t size;

    if (ptr == NULL)
        return;

    size = GET_SIZE(HDRP(ptr));
    PUT(HDRP(ptr), PACK(size, 0));    /* 清除分配位 */
    PUT(FTRP(ptr), PACK(size, 0));    /* footer 也要改 */
    SET_PRED(ptr, NULL);
    SET_SUCC(ptr, NULL);
    coalesce(ptr);
}
```

**头脚都要改**。只改头部的话，后面某次 `PREV_BLKP` 读到旧 footer 会认为这块还是"已分配"，导致漏合并。

### mm_realloc

```c
void *mm_realloc(void *ptr, size_t size)
{
    void *newptr;
    size_t copySize;

    if (ptr == NULL)
        return mm_malloc(size);
    if (size == 0) {
        mm_free(ptr);
        return NULL;
    }

    newptr = mm_malloc(size);
    if (newptr == NULL)
        return NULL;

    copySize = GET_SIZE(HDRP(ptr)) - DSIZE;   /* 旧块可用载荷 */
    if (size < copySize)
        copySize = size;                       /* 取小的 */
    memcpy(newptr, ptr, copySize);
    mm_free(ptr);
    return newptr;
}
```

这是"**分配-拷贝-释放**"的朴素实现。更好的做法是就地扩展：如果后一块空闲且合起来够大，直接合并，省掉拷贝。

---

## 三、实测结果

```bash
make clean && make
./mdriver -V -f short1-bal.rep
```

```text
Team Name:bovik
Member 1 :Harry Bovik:bovik@cs.cmu.edu
Measuring performance with gettimeofday().

Testing mm malloc
Reading tracefile: short1-bal.rep
Checking mm_malloc for correctness, efficiency, and performance.

Results for mm malloc:
trace  valid  util     ops      secs  Kops
 0       yes   66%      12  0.000000 60000
Total          66%      12  0.000000 60000

Perf index = 40 (util) + 40 (thru) = 80/100
```

```bash
./mdriver -V -f short2-bal.rep
```

```text
Results for mm malloc:
trace  valid  util     ops      secs  Kops
 0       yes   89%      12  0.000000 60000
Total          89%      12  0.000000 60000

Perf index = 54 (util) + 40 (thru) = 94/100
```

| trace | valid | util | Perf index |
| --- | --- | ---: | ---: |
| short1-bal.rep | yes | 66% | **80**/100 |
| short2-bal.rep | yes | 89% | **94**/100 |

`valid = yes` 表示通过了正确性检查（数据没被破坏、没有重叠分配）。

---

## 四、想拿更高分怎么改

### 提升 util（省空间）

**换成分离空闲链表**

按大小分成多个链表（如 2^4、2^5、2^6…… 各一个），分配时先在小尺寸链表里找：

```c
#define NUM_CLASSES 16
static void *free_lists[NUM_CLASSES];

/* 大小 -> 分级索引 */
static int get_class(size_t size) {
    int c = 0;
    while ((size >>= 1) > 0) c++;
    return c < NUM_CLASSES ? c : NUM_CLASSES - 1;
}
```

好处是可以做 **best-fit** 而不必扫全表——只在最合适的尺寸级别里找，浪费的块更少。

**优化 realloc**

当前实现每次都重新分配+拷贝。改成：

```c
void *mm_realloc(void *ptr, size_t size)
{
    size_t oldsize = GET_SIZE(HDRP(ptr));
    size_t newsize = ALIGN(size + DSIZE);

    if (newsize <= oldsize) {
        return ptr;                    /* 缩小，原块够用，直接返回 */
    }
    /* 尝试与后块合并 */
    char *next = NEXT_BLKP(ptr);
    if (!GET_ALLOC(HDRP(next)) &&
        oldsize + GET_SIZE(HDRP(next)) >= newsize) {
        remove_free(next);
        PUT(HDRP(ptr), PACK(oldsize + GET_SIZE(HDRP(next)), 1));
        return ptr;
    }
    /* 实在不行才重新分配 */
    ...
}
```

对 `realloc-bal.rep` 这类以 realloc 为主的 trace，能大幅减少拷贝和碎片。

### 提升 thru（快）

**减少 `place` 里的拆分判断开销** —— 可以预先算好 `csize - asize` 的阈值表。

**first-fit 换成 next-fit** —— 记录上次找到的位置，下次从那开始扫，避免每次都撞到同一批不合适的小块。

**只存头不存脚** —— 用 `bit1`（前块已分配标志）省掉 footer，省 8 字节/块，但向后合并时要靠遍历找前块头部，速度换空间。

---

## 五、常见问题

**`timeout: the monitored command dumped core`**

块结构被写坏。64 位下最常见的原因是 `WSIZE` 还写 4、`pred/succ` 按 4 字节存，指针被截断。

**`util` 很低（< 30%）**

- 检查 `place` 里的 `MIN_BLOCK` 判断，阈值太小会拆出无数碎片
- 检查 `coalesce` 是否真的合并了，读错 footer 会漏合并
- 检查 `mm_free` 是否同时改了头尾

**`valid = no`**

数据被覆盖，通常是：
- `mm_malloc` 返回的 payload 大小算小了（少算头脚）
- `place` 拆分时新块的 footer 没写
- `extend_heap` 忘了补新的结尾头

**`mdriver` 编译报 `stubs-32.h`**

`Makefile` 里 `-m32` 去掉。注意等号两边可能有空格，`sed` 写严一点。

**只有一个 trace 测不出问题**

本实验包缺 `traces/` 目录，默认的 11 个 trace 都不在。要完整评测需要从官方 handout 补回 traces 目录——那些 trace 覆盖了重分配、极端碎片、随机大小等场景。

---

## 六、调试建议

`mdriver.c` 里有 `#define VERBOSE` 和 `-V` 选项，打开后能看到每个操作的细节：

```bash
./mdriver -V -f short1-bal.rep
```

自己再加临时打印更直接——在 `mm_malloc` / `mm_free` 里打印 `bp`、`size`、`asize`，对照 trace 里的操作序列手工核对。

`mem_heap_lo()` 和 `mem_heap_hi()` 能拿到堆的边界，写个断言检查 `bp` 是否越界：

```c
assert((char *)ptr >= (char *)mem_heap_lo() &&
       (char *)ptr <= (char *)mem_heap_hi());
```

---

## 附录：环境与踩坑

### 问题 1：`Makefile` 写死 `-m32`

```text
/usr/include/gnu/stubs.h:7:11: fatal error: gnu/stubs-32.h: No such file or directory
make: *** [<builtin>: mdriver.o] Error 1
```

`Makefile` 第 9 行：

```makefile
CFLAGS = -Wall -O2 -m32
```

**注意等号两边有空格**，`sed 's/^CFLAGS=...'` 的写法匹配不上。正确写法：

```bash
sed -i 's/^CFLAGS *= *-Wall -O2 -m32$/CFLAGS = -Wall -O2 -g/' Makefile
grep -n "^CFLAGS" Makefile
```
```text
9:CFLAGS = -Wall -O2 -g
```

### 问题 2：64 位下 WSIZE 用 4 会 core dump

这是本次最致命的坑。

教材第 9 章的示例代码是按 **32 位**写的：

```c
#define WSIZE 4          /* 头部 4 字节 */
#define DSIZE 8          /* 头+脚 8 字节 */
/* 空闲块的 pred/succ 各 4 字节 */
```

在 64 位机器上，指针是 8 字节。如果 `pred`/`succ` 只留 4 字节，写进去的 8 字节指针会把相邻数据覆盖掉，块结构立刻错乱：

```bash
./mdriver -V -f short1-bal.rep
```
```text
timeout: the monitored command dumped core
```

**64 位适配的四个参数**：

```c
#define WSIZE       8       /* 头部/脚部宽度 = 指针宽度 */
#define DSIZE       16      /* 头 + 脚 */
#define PSIZE       8       /* 指针宽度 */
#define MIN_BLOCK   32      /* 8(头) + 16(pred+succ) + 8(脚) */
```

以及 `GET`/`PUT` 要用 `size_t`：

```c
#define GET(p)      (*(size_t *)(p))
#define PUT(p, val) (*(size_t *)(p) = (val))
#define GET_SIZE(p)  (GET(p) & ~(size_t)0xF)      /* 低 4 位做标志 */
```

### 问题 3：`traces/` 目录缺失

`config.h` 里的默认 trace 列表有 11 个：

```c
#define DEFAULT_TRACEFILES \
  "amptjp-bal.rep", "cccp-bal.rep", "cp-decl-bal.rep", \
  "expr-bal.rep", "coalescing-bal.rep", "random-bal.rep", \
  "random2-bal.rep", "binary-bal.rep", "binary2-bal.rep", \
  "realloc-bal.rep", "realloc2-bal.rep"
```

但实验包内**只有两个**：

```bash
ls *.rep
```
```text
short1-bal.rep
short2-bal.rep
```

没有 `traces/` 目录。直接跑 `./mdriver` 会去找那 11 个文件然后失败。**必须用 `-f` 指定具体文件**：

```bash
./mdriver -V -f short1-bal.rep
./mdriver -V -f short2-bal.rep
```

要完整评测需要从官方 handout 补回 traces 目录 —— 那些 trace 覆盖了重分配、极端碎片、随机大小等场景，只跑 short1/short2 测不出很多问题。

### `-f` 参数一次只能给一个文件

```bash
./mdriver -V -f short1-bal.rep -f short2-bal.rep
```

命令行能解析，但实际只跑了最后一个。要测多个就分开跑：

```bash
for t in short1-bal.rep short2-bal.rep; do
  echo "=== $t ==="
  ./mdriver -V -f $t 2>&1 | grep -E "^ 0|Perf index"
done
```

### 序言块和结尾块不能省

```c
int mm_init(void)
{
    if ((heap_listp = mem_sbrk(4 * WSIZE)) == (void *)-1)
        return -1;
    PUT(heap_listp, 0);                              /* 对齐填充 */
    PUT(heap_listp + (1 * WSIZE), PACK(DSIZE, 1));   /* 序言头 */
    PUT(heap_listp + (2 * WSIZE), PACK(DSIZE, 1));   /* 序言脚 */
    PUT(heap_listp + (3 * WSIZE), PACK(0, 1));       /* 结尾头 */
    heap_listp += (2 * WSIZE);
```

**省掉会怎样**：

`coalesce` 里要读前一块的状态：

```c
size_t prev_alloc = GET_ALLOC(FTRP(PREV_BLKP(bp)));
```

对第一个块，`PREV_BLKP(bp)` 会算到堆外面去。序言脚就是给这个读操作兜底。

同理 `NEXT_BLKP` 在最后一个块上会跑到堆外，结尾头兜底。

**这两个哨兵是边界安全检查的基石，不是可有可无的填充。**

### `extend_heap` 忘了补新的结尾头

```c
static void *extend_heap(size_t words)
{
    ...
    PUT(HDRP(bp), PACK(size, 0));
    PUT(FTRP(bp), PACK(size, 0));
    SET_PRED(bp, NULL);
    SET_SUCC(bp, NULL);
    PUT(HDRP(NEXT_BLKP(bp)), PACK(0, 1));    /* ← 这行不能少 */

    return coalesce(bp);
}
```

新块占用了原来的结尾头位置，必须在新的堆末尾重新放一个。漏了这行，`coalesce` 往后读就是垃圾数据，`valid = no`。

### `mm_free` 只改头部不改脚部

```c
void mm_free(void *ptr)
{
    size = GET_SIZE(HDRP(ptr));
    PUT(HDRP(ptr), PACK(size, 0));    /* 改头 */
    PUT(FTRP(ptr), PACK(size, 0));    /* ← 脚也要改 */
```

只改头部的话，后面某次 `coalesce` 用 `GET_ALLOC(FTRP(PREV_BLKP(bp)))` 读到旧 footer，会认为这块还是已分配，**漏掉一次合并**。

表现：`util` 偏低，但没有明显错误。

### `place` 的 MIN_BLOCK 判断

```c
if ((csize - asize) >= MIN_BLOCK) {
    /* 拆分 */
} else {
    /* 整块给出去 */
}
```

如果阈值设成 `DSIZE`（16）而不是 `MIN_BLOCK`（32），拆出来的"空闲块"只有 16 字节，**放不下 pred+succ+头+脚**，后续 `insert_free` 会写越界。

**宁可牺牲内部碎片，也不能破坏块结构。**

### `size <= 2*WSIZE` 的特判

```c
if (size <= 2 * WSIZE)
    asize = MIN_BLOCK;
else
    asize = ALIGN(size + DSIZE);
```

请求 8 字节时，`ALIGN(8 + 16) = ALIGN(24) = 32 = MIN_BLOCK`，结果其实一样。但显式写出来能避免 `ALIGN` 在边界上的意外（比如 `size=0` 的情况已经在前面 return 了）。

### `realloc` 的 copySize 算错会越界

```c
copySize = GET_SIZE(HDRP(ptr)) - DSIZE;   /* 减去头脚 */
if (size < copySize)
    copySize = size;
memcpy(newptr, ptr, copySize);
```

如果忘了 `- DSIZE`，`copySize` 会比实际可用载荷大 16 字节，`memcpy` 把下一个块的头部也拷走，**内存立刻损坏**。

### `util` 和 `thru` 的权衡

评分公式：

```c
Perf index = 60 * util + 40 * thru
```

- `util` 高 = 省空间（少浪费）
- `thru` 高 = 分配快（少遍历）

两者冲突：**best-fit 省空间但慢，first-fit 快但浪费**。

本次用 first-fit，实测：

| trace | util | Perf index |
| --- | ---: | ---: |
| short1 | 66% | 80/100 |
| short2 | 89% | 94/100 |

short1 的 66% 偏低，因为块少（12 次操作），一次浪费就影响很大。short2 有 89% 说明合并逻辑是对的。

**想提高 util**：换成分离空闲链表，按大小分级后可以做 best-fit 而不必扫全表。

### 输出里的 `60000` 是什么

```text
trace  valid  util     ops      secs  Kops
 0       yes   66%      12  0.000000 60000
```

`Kops = 60000` 是**上限封顶值**（`config.h` 里 `AVG_LIBC_THRUPUT 600E3`）。short1 只有 12 次操作，耗时太短测不出来，被截到上限。不是 bug。

### 加断言辅助调试

```c
assert((char *)ptr >= (char *)mem_heap_lo() &&
       (char *)ptr <= (char *)mem_heap_hi());
```

放在 `mm_malloc` 返回值处和 `mm_free` 入口处，能第一时间抓出越界。

更细的可以遍历整条空闲链表检查有无环：

```c
static void check_free_list(void)
{
    int n = 0;
    for (char *b = free_listp; b; b = GET_SUCC(b)) {
        assert(n++ < 10000);          /* 有环会超限 */
    }
}
```

`GET_SUCC` 写错方向或没更新 pred/succ 时，链表会成环，`find_fit` 死循环。这个断言能快速定位。
