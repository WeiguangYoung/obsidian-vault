---
{"dg-publish":true,"permalink":"/knowledge/course/CS61C/06-Generics-Function-Pointers/","title":"CS61C L06 · C Generics and Function Pointers","tags":["CS61C","函数指针","泛型","void指针","memcpy","课程笔记"],"dg-note-properties":{"title":"CS61C L06 · C Generics and Function Pointers","tags":["CS61C","函数指针","泛型","void指针","memcpy","课程笔记"],"date":"2026-11-13","source":"https://cs61c.org/fa26/"}}
---


# CS61C FA26 · L06 C Generics and Function Pointers

> UC Berkeley · Great Ideas in Computer Architecture
> 授课：Dan Garcia
> 对应课件 06-C Generics and Function Pointers（48 页）
> 部分幻灯片改编自 Stanford CS107

---

## 一、内存管理会出什么问题

### 1.1 三类内存的难度对比

| 区域 | 难度 | 原因 |
|---|---|---|
| **Code / Static** | ✅ 简单 | 永不变大或缩小 |
| **Stack** | ✅ 简单 | 栈帧按 LIFO 创建/销毁；只要**别返回悬垂引用** |
| **Heap** | ⚠️ **棘手** | 任何时刻都能分配/释放；**内存泄漏**；最终耗尽 |

> **Heap 是所有内存 bug 的源头。**

### 1.2 运行时不会帮你检查

> **运行时不会检查程序员是否管理好内存。**

- **为什么**：内存太关键了，**没时间做检查**
- **常见后果**：你**破坏了分配器的内部结构**，然后在**完全无关的代码里**很久以后才发现

### 1.3 内存泄漏的隐蔽性

> **Memory leak：忘记 `free()` 已分配的内存。**

```
Initial symptoms: Nothing... 🙃
```

> 在达到临界点之前，泄漏**根本不是问题**——然后突然全面崩溃。

### 1.4 找出这段代码里的错误

```c
void free_mem_x() {
    int fnh[3];
    ...
    free(fnh);          // ❌ 错误 1
}

void free_mem_y() {
    int *fum = malloc(4*sizeof(int));
    free(fum + 1);      // ❌ 错误 2
    ...
    free(fum);
    ...
    free(fum);          // ❌ 错误 3
}
```

| # | 错误 | 说明 |
|---|---|---|
| **1** | `free()` 栈上分配的内存 | 数组 `fnh` 在栈上，不是 `malloc` 来的 |
| **2** | `free()` 非 `malloc` 返回的地址 | `fum+1` 是内部地址 |
| **3** | **Double free** | 同一个指针释放两次 |

> `man malloc` 明确写了：`ptr` **必须是** `malloc` / `calloc` / `realloc` 返回的原始指针。

### 1.5 Valgrind 救命工具

> **Valgrind 会让程序慢一个数量级，但是……**

它加入了大量检查，能捕获**大部分（不是全部）**内存错误：

- **内存泄漏**
- **误用 `free`**
- **写入数组末尾之外**

> **「Valgrind 这类工具对调试 C 代码是绝对必需的。」**
> Lab 02 会用到。

### 1.6 Use after Free（释放后使用）

> **Dangling reference（悬垂引用）**：指针被释放后**还继续使用**。

```c
int *foo;
foo = malloc(sizeof(int));
...
free(foo);
...
bar(foo);    // 💥 用了已释放的内存
```

| 操作 | 后果 |
|---|---|
| **读**（free 之后） | 可能读到**被污染的数据** |
| **写**（free 之后） | **破坏别人的数据** → 程序**稍后崩溃** |

> 如果这块内存被别的对象接管了，你的程序会读到错误信息。

### 1.7 Double-Free（重复释放）

**`man malloc` 原话：**

> `free()` 释放 `ptr` 指向的内存空间，**`ptr` 必须是之前 `malloc()`、`calloc()` 或 `realloc()` 返回的**。
> 否则，**或者如果 `free(ptr)` 已经被调用过**，就是**未定义行为**。
> 如果 `ptr` 是 `NULL`，不执行任何操作。

```c
int *foo = malloc(10*sizeof(int));
...
free(foo);
...
free(foo);    // 💥 未定义行为（包括 use-after-free）
```

### 1.8 忘记 `realloc` 可能会移动数据

```c
int *nums;
nums = malloc(10*sizeof(int));
...
realloc(nums, 20*sizeof(int));    // ❌ 忘了更新 nums！
// nums 现在可能指向无效内存
// 而且新块的指针丢了 → 内存泄漏
```

> **`realloc` 可能把数据复制到堆的其他位置。**
> **读会读到损坏数据，写会破坏其他内存。**

**正确写法：**

```c
int *tmp = realloc(nums, 20*sizeof(int));
if (tmp == NULL) { /* 处理失败 */ }
nums = tmp;
```

### 1.9 阅读手册页

> **Pro Tip：运行 `man malloc`！**
> Linux 手册页描述了堆 API（以及未定义行为）。

---

## 二、函数指针

### 2.1 指针能指向什么

```c
int *xptr;                    // 指向 int
char *str;                    // 指向 char
struct llist *foo_ptr;        // 指向结构体
void *p;                      // 通用指针（慎用）
int (*fp)(void *, void *) = &foo;   // ← 指向函数！
```

### 2.2 完整示例：把函数「映射」到数组

```c
/* map a function onto int array */
void mutate_map(int arr[], int n, int (*fp)(int)) {
    for (int i = 0; i < n; i++)
        arr[i] = (*fp)(arr[i]);
}

int multiply2 (int x) { return 2 * x; }
int multiply10(int x) { return 10 * x; }

int main() {
    int arr[] = {3, 1, 4};
    int n = sizeof(arr)/sizeof(arr[0]);
    print_array(arr, n);                    // 3 1 4

    mutate_map(arr, n, &multiply2);
    print_array(arr, n);                    // 6 2 8

    mutate_map(arr, n, &multiply10);
    ...
}
```

### 2.3 三个关键语法点

**① 声明**

```c
int (*fp)(int)
```

`fp` 是**函数指针**，类型是「**接受 int 参数、返回 int**」。

> ⚠️ 括号不能省：`int *fp(int)` 意思是「返回 `int *` 的函数」。

**② 初始化**

```c
mutate_map(arr, n, &multiply2);    // 传函数地址
```

> **函数名本身就是地址**，`&multiply2` 和 `multiply2` 通常等价。

**③ 调用**

```c
arr[i] = (*fp)(arr[i]);    // 解引用后调用
```

> 现代 C 也允许 `fp(arr[i])`，但 `(*fp)(...)` 更明确。

### 2.4 函数指针的意义

> **函数指针让 C 支持高阶函数（higher-order functions）。**
> 例如 `map`、`filter`、排序等。

这也是后面「泛型」的基础——**把「怎么处理」作为参数传进去**。

---

## 三、泛型函数（Generics）

### 3.1 为什么要泛型

> **Generic code 减少代码重复**，改进和修 bug 只需改**一处**。

C 标准库里到处都是泛型：**排序任意数组、搜索任意数组、释放任意内存**。

**泛型的两个目标：**

- **不管参数类型都能工作**
- **不管块里存什么类型，都能更新那块内存**

> 这就是 `void *` 的用武之地。

### 3.2 问题：三种 swap 长得一样

```c
void swap_int(int *ptr1, int *ptr2) {
    int temp = *ptr1;
    *ptr1 = *ptr2;
    *ptr2 = temp;
}

void swap_short(short *ptr1, short *ptr2) {
    short temp = *ptr1;
    *ptr1 = *ptr2;
    *ptr2 = temp;
}

void swap_string(char **ptr1, char **ptr2) {
    char *temp = *ptr1;
    *ptr1 = *ptr2;
    *ptr2 = temp;
}
```

> **逻辑完全一样，只是类型不同。** 这正是泛型要消灭的重复。

### 3.3 想写通用 swap

```c
void swap(void *ptr1, void *ptr2) {
    // 1. 把 data1 的副本存到临时存储
    // 2. 把 data2 拷贝到 data1 的位置
    // 3. 把临时副本拷贝到 data2 的位置
}
```

**问题来了**：`void *` 不能解引用！

### 3.4 ⚠️ 为什么不能解引用 `void *`

```c
// Case 1
void *ptr = ...;
printf("%p\n", *ptr);        // ❌ 编译警告 + 运行时错误

// Case 2
void **doubleptr = ...;
printf("%p\n", *doubleptr);  // ✅ 可以（有一层具体类型）
```

**编译器报错：**

```
warning: dereferencing 'void *' pointer
error:   invalid use of void expression
```

**根本原因：**

> **要解引用指针，必须在编译期知道「要访问内存中的多少字节」。**
> **泛型指针（`void *`）因此无法使用解引用运算符！**

### 3.5 解决方案：`memcpy` / `memmove`

> **要用 `void *` 访问若干字节，请用 `<string.h>` 里的 `memcpy` / `memmove`。**

```c
void *memcpy (void *dest, const void *src, size_t n);
// man memcpy: "The memory areas must not overlap."

void *memmove(void *dest, const void *src, size_t n);
// man memmove: "copying takes place as if the source bytes
//              were first copied into a temporary array"
```

| 函数 | 适用场景 |
|---|---|
| **`memcpy`** | 两块内存**不重叠**（更快） |
| **`memmove`** | 可能**重叠**（安全但稍慢） |

> 两者都**逐字节复制 n 字节**，并返回 `dest`。

### 3.6 关于 `<string.h>`

> **`<string.h>` 不只处理字符串，也处理内存。**

**常用内存函数**：`memcpy`、`memmove`、`memset`……

> 头文件名字有点误导，但**内存操作是逐字节的**，而**字符串本质上就是（以 null 结尾的）字节数组**——所以放在一起很合理。

### 3.7 用 memcpy 重写 swap

**原始版本：**

```c
void swap_int(int *ptr1, int *ptr2) {
    int temp = *ptr1;    // ①
    *ptr1 = *ptr2;       // ②
    *ptr2 = temp;        // ③
}
```

**泛型版本：**

```c
void swap(void *ptr1, void *ptr2, size_t nbytes) {
    char temp[nbytes];            // ① 按字节分配临时空间
    memcpy(temp, ptr1, nbytes);   // ② 存 data1
    memcpy(ptr1, ptr2, nbytes);   // ③ data2 → data1
    memcpy(ptr2, temp, nbytes);   // ④ temp → data2
}
```

**使用：**

```c
int data1 = 22;
int data2 = 61;
swap(&data1, &data2, sizeof(data1));
```

**执行过程（假设 `sizeof(int) == 4`）：**

```
初始：  data1=22  data2=61   temp=??

第 1 次 memcpy(temp, ptr1, nbytes):
        data1=22  data2=61   temp=22

第 2 次 memcpy(ptr1, ptr2, nbytes):
        data1=61  data2=61   temp=22

第 3 次 memcpy(ptr2, temp, nbytes):
        data1=61  data2=22   temp=22   ✅
```

### 3.8 关键：`nbytes` 参数不能省

> **泛型必须知道「指向多少字节」。**
> 所以 `swap` 需要一个 `nbytes` 参数。

---

## 四、进阶：泛型数组操作（swap_ends）

### 4.1 任务

> 用 `swap` 交换数组的**第一个和最后一个元素**。

```c
void swap(void *ptr1, void *ptr2, size_t nbytes) { ... }   // 已有

int main() {
    int32_t arr[] = {1, 2, 3, 4, 5};
    int32_t n = sizeof(arr)/sizeof(arr[0]);
    swap_ends(arr, n, sizeof(arr[0]));    // 待实现
    ...
}
```

### 4.2 参数设计

```c
void swap_ends(void *arr, size_t nelems, size_t nbytes) {
    ...
}
```

| 参数 | 为什么需要 |
|---|---|
| `void *arr` | 泛型数组——**但丢了元素类型信息** |
| `size_t nelems` | 「数组」参数**需要元素个数** |
| `size_t nbytes` | 泛型**需要每个元素的字节大小** |

### 4.3 ⚠️ 关键陷阱：指针算术必须按字节

**错误想法：**

```c
arr + nelems - 1              // ❌ arr 是 void*，无法算术
arr + (nelems - 1)*nbytes     // ❌ 仍然不行
```

**正确答案：**

```c
(char *)arr + (nelems - 1) * nbytes    // ✅
```

**原因：**

> **泛型里的指针算术必须按字节进行：**
> 1. **把 `void *` 转成 `char *`**
> 2. **这样指针算术就变成了字节级**

因为 `char` 的大小是 **1 字节**，所以 `char *` 上的 `+n` 就是「前进 n 字节」。

**完整实现：**

```c
void swap_ends(void *arr, size_t nelems, size_t nbytes) {
    swap(arr,                                // 首元素
         (char *)arr + (nelems - 1) * nbytes, // 末元素
         nbytes);
}
```

---

## 五、本讲总结（课程原话）

- **函数指针让 C 支持高阶函数**
  - `map`、`filter`、排序等
- **泛型函数用 `void *` 操作内存**
  - 泛型在 C 标准库里广泛存在（`malloc` 就是）
  - **泛型要求对内存有扎实理解**：操作任意字节，你可能越过数据边界——比如把两个 int 的**一半拼在一起（"Frankenstein"-ing）**
- **写泛型时要注意**（详见课件末页）

---

## 六、我的笔记与理解

### 6.1 本讲的两条主线

```
内存会坏  →  Valgrind + 手工检查
写泛型    →  void* + memcpy
```

**两者其实是同一个主题的两面**：`void *` 给了你最强能力，也给了你最大的犯错空间。

### 6.2 三种 free 错误的一句话记法

| 错误 | 记法 |
|---|---|
| **free 栈内存** | 只能 free「堆」的 |
| **free 中间地址** | 只能 free「原始」的 |
| **double free** | 只能 free「一次」 |

**口诀：「堆、原、一」。**

### 6.3 为什么 `void *` 不能解引用

这是本讲最关键的一个「为什么」：

```
解引用 *p  →  编译器要生成「读多少字节」的指令
              ↓
        必须知道 p 的类型
              ↓
      void 没有类型 → 不知道字节数 → 无法生成指令
```

> **泛型的代价**：你放弃了类型信息，也就放弃了自动的「按类型读写」。

### 6.4 `memcpy` vs `memmove` 的选择

| 场景 | 选哪个 |
|---|---|
| 确定不重叠 | **`memcpy`**（更快） |
| 不确定 / 可能重叠 | **`memmove`**（安全） |
| 自己给自己赋值 | 两者都行 |

> `swap` 里用 `memcpy` 是安全的，因为 `ptr1` 和 `ptr2` 指向不同变量。

### 6.5 「按字节」这个思想

泛型编程的核心是**「把类型擦除，回到字节」**：

```
具体类型 swap_int  →  知道 4 字节
        ↓
泛型 swap(void*, void*, nbytes)  →  「nbytes 字节」而已
```

**这和解引用失败的原因是一致的**：只有回到「字节」层级，才能处理任意类型。

### 6.6 与后续的衔接

| 讲次 | 关系 |
|---|---|
| **L07** | 位运算 + 用 struct 构建 ADT（链表示例） |
| **L08** | 浮点数（`void *` 的另一个经典用途：在 float 和 bits 之间转换） |
| **L09+** | RISC-V：函数指针在汇编里就是「地址」 |

---

## 七、术语速查

| 术语 | 中文 | 说明 |
|---|---|---|
| **function pointer** | 函数指针 | `int (*fp)(int)` |
| **higher-order function** | 高阶函数 | 接受/返回函数 |
| **generic** | 泛型 | 与类型无关的代码 |
| **void \*** | 通用指针 | 可以指向任何类型 |
| **memcpy** | 内存复制 | 非重叠 |
| **memmove** | 内存移动 | 支持重叠 |
| **Valgrind** | 内存检查工具 | 慢但能找到大部分错误 |
| **dangling reference** | 悬垂引用 | 指向已释放内存 |
| **double free** | 重复释放 | 未定义行为 |
| **use after free** | 释放后使用 | 未定义行为 |

---

## 相关

- 课程导航：[[knowledge/course/index\|课程 目录导航]]
- 上一讲：[[knowledge/course/CS61C/05-Memory-Management\|L05 C Memory Management]]
- 下一讲：[[knowledge/course/CS61C/07-Bitwise-Ops-ADTs\|L07 IEC, Bitwise Ops, ADTs]]
