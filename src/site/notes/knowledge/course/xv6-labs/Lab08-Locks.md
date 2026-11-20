---
{"dg-publish":true,"permalink":"/knowledge/course/xv6-labs/Lab08-Locks/","title":"6.S081 Lab8 · 锁","tags":["MIT6.1810","xv6","操作系统","LAB","实验"],"dg-note-properties":{"title":"6.S081 Lab8 · 锁","tags":["MIT6.1810","xv6","操作系统","LAB","实验"],"date":"2026-11-18"}}
---


# 6.S081 · Lab8 锁

> MIT 6.S081 / 6.1810 · Lab 08 — Locks
> 在 xv6-riscv 上完成的实验，含实现思路与关键代码

返回 [[knowledge/course/xv6-labs/index\|🧪 xv6-labs 目录]] · [[knowledge/course/mit6.s081/index\|📚 6.S081 课程]] · [[首页\|🏠 首页]]

---

## pre

预编译指令

* `#define`

* `#ifdef`

  ```c
  #ifdef x  //先测试x是否被宏定义过  
  程序段1   //如果x被宏定义过，那么就编译程序段1  
  #else 
  程序段2 //如果x没有被定义过则编译程序段2的语句，“忽视”程序段1。 
  #endif 
  ```





为了统计每个`spinlock`被获取的次数和`spinlock`的在`acquire`中循环的次数，修改`spinlock`数据结构如下

```c
// Mutual exclusion lock.
struct spinlock {
  uint locked;       // Is the lock held?

  // For debugging:
  char *name;        // Name of lock.
  struct cpu *cpu;   // The cpu holding the lock.
#ifdef LAB_LOCK
  int nts;
  int n;
#endif
};

```



* `n`用来记录锁企图`acquire`的次数
* `nts`用来记录锁在`acquire`中循环的次数











## 安全性

* 使用`cpuid` 函数需要关中断，目前的
