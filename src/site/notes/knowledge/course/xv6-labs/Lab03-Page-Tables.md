---
{"dg-publish":true,"permalink":"/knowledge/course/xv6-labs/Lab03-Page-Tables/","title":"6.S081 Lab3 · 页表","tags":["MIT6.1810","xv6","操作系统","LAB","实验"],"dg-note-properties":{"title":"6.S081 Lab3 · 页表","tags":["MIT6.1810","xv6","操作系统","LAB","实验"],"date":"2026-11-18"}}
---


# 6.S081 · Lab3 页表

> MIT 6.S081 / 6.1810 · Lab 03 — Page Tables
> 三部分：实验题目 / 我的实现 / 参考答案与讲解。

返回 [[knowledge/course/xv6-labs/index\|🧪 xv6-labs 目录]] · [[knowledge/course/mit6.s081/index\|📚 6.S081 课程]] · [[首页\|🏠 首页]] · [[knowledge/course/xv6-labs/Lab02-System-Calls\|← 上一实验]] · [[knowledge/course/xv6-labs/Lab04-Traps\|下一实验 →]]

---

## 一、实验题目

在本实验室中，您将探索页表并对其进行修改，以简化将数据从用户空间复制到内核空间的函数。

> [!WARNING|label:Attention]
> 开始编码之前，请阅读xv6手册的第3章和相关文件：
>
> - ***kernel/memlayout.h***，它捕获了内存的布局。
> - ***kernel/vm.c***，其中包含大多数虚拟内存（VM）代码。
> - ***kernel/kalloc.c***，它包含分配和释放物理内存的代码。

要启动实验，请切换到pgtbl分支：

```bash
$ git fetch
$ git checkout pgtbl
$ make clean
```

### Print a page table (easy)

为了帮助您了解RISC-V页表，也许为了帮助将来的调试，您的第一个任务是编写一个打印页表内容的函数。

> [!TIP|label:YOUR JOB]
> 定义一个名为`vmprint()`的函数。它应当接收一个`pagetable_t`作为参数，并以下面描述的格式打印该页表。在`exec.c`中的`return argc`之前插入`if(p->pid==1) vmprint(p->pagetable)`，以打印第一个进程的页表。如果你通过了`pte printout`测试的`make grade`，你将获得此作业的满分。

现在，当您启动xv6时，它应该像这样打印输出来描述第一个进程刚刚完成`exec()`ing`init`时的页表：

```
page table 0x0000000087f6e000
..0: pte 0x0000000021fda801 pa 0x0000000087f6a000
.. ..0: pte 0x0000000021fda401 pa 0x0000000087f69000
.. .. ..0: pte 0x0000000021fdac1f pa 0x0000000087f6b000
.. .. ..1: pte 0x0000000021fda00f pa 0x0000000087f68000
.. .. ..2: pte 0x0000000021fd9c1f pa 0x0000000087f67000
..255: pte 0x0000000021fdb401 pa 0x0000000087f6d000
.. ..511: pte 0x0000000021fdb001 pa 0x0000000087f6c000
.. .. ..510: pte 0x0000000021fdd807 pa 0x0000000087f76000
.. .. ..511: pte 0x0000000020001c0b pa 0x0000000080007000
```

第一行显示`vmprint`的参数。之后的每行对应一个PTE，包含树中指向页表页的PTE。每个PTE行都有一些“`..`”的缩进表明它在树中的深度。每个PTE行显示其在页表页中的PTE索引、PTE比特位以及从PTE提取的物理地址。不要打印无效的PTE。在上面的示例中，顶级页表页具有条目0和255的映射。条目0的下一级只映射了索引0，该索引0的下一级映射了条目0、1和2。

您的代码可能会发出与上面显示的不同的物理地址。条目数和虚拟地址应相同。

**一些提示：**

- 你可以将`vmprint()`放在***kernel/vm.c***中
- 使用定义在***kernel/riscv.h***末尾处的宏
- 函数`freewalk`可能会对你有所启发
- 将`vmprint`的原型定义在***kernel/defs.h***中，这样你就可以在`exec.c`中调用它了
- 在你的`printf`调用中使用`%p`来打印像上面示例中的完成的64比特的十六进制PTE和地址

> [!NOTE|label:QUESTION]
> 根据文本中的图3-4解释`vmprint`的输出。page 0包含什么？page 2中是什么？在用户模式下运行时，进程是否可以读取/写入page 1映射的内存？

### A kernel page table per process (hard)

Xv6有一个单独的用于在内核中执行程序时的内核页表。内核页表直接映射（恒等映射）到物理地址，也就是说内核虚拟地址`x`映射到物理地址仍然是`x`。Xv6还为每个进程的用户地址空间提供了一个单独的页表，只包含该进程用户内存的映射，从虚拟地址0开始。因为内核页表不包含这些映射，所以用户地址在内核中无效。因此，当内核需要使用在系统调用中传递的用户指针（例如，传递给`write()`的缓冲区指针）时，内核必须首先将指针转换为物理地址。本节和下一节的目标是允许内核直接解引用用户指针。

> [!TIP|label:YOUR JOB]
> 你的第一项工作是修改内核来让每一个进程在内核中执行时使用它自己的内核页表的副本。修改`struct proc`来为每一个进程维护一个内核页表，修改调度程序使得切换进程时也切换内核页表。对于这个步骤，每个进程的内核页表都应当与现有的的全局内核页表完全一致。如果你的`usertests`程序正确运行了，那么你就通过了这个实验。

阅读本作业开头提到的章节和代码；了解虚拟内存代码的工作原理后，正确修改虚拟内存代码将更容易。页表设置中的错误可能会由于缺少映射而导致陷阱，可能会导致加载和存储影响到意料之外的物理页存页面，并且可能会导致执行来自错误内存页的指令。

**提示：**

- 在`struct proc`中为进程的内核页表增加一个字段
- 为一个新进程生成一个内核页表的合理方案是实现一个修改版的`kvminit`，这个版本中应当创造一个新的页表而不是修改`kernel_pagetable`。你将会考虑在`allocproc`中调用这个函数
- 确保每一个进程的内核页表都关于该进程的内核栈有一个映射。在未修改的XV6中，所有的内核栈都在`procinit`中设置。你将要把这个功能部分或全部的迁移到`allocproc`中
- 修改`scheduler()`来加载进程的内核页表到核心的`satp`寄存器(参阅`kvminithart`来获取启发)。不要忘记在调用完`w_satp()`后调用`sfence_vma()`
- 没有进程运行时`scheduler()`应当使用`kernel_pagetable`
- 在`freeproc`中释放一个进程的内核页表
- 你需要一种方法来释放页表，而不必释放叶子物理内存页面。
- 调式页表时，也许`vmprint`能派上用场
- 修改XV6本来的函数或新增函数都是允许的；你或许至少需要在***kernel/vm.c***和***kernel/proc.c***中这样做（但不要修改***kernel/vmcopyin.c***, ***kernel/stats.c***, ***user/usertests.c***, 和***user/stats.c***）
- 页表映射丢失很可能导致内核遭遇页面错误。这将导致打印一段包含`sepc=0x00000000XXXXXXXX`的错误提示。你可以在***kernel/kernel.asm***通过查询`XXXXXXXX`来定位错误。

### Simplify `copyin`/`copyinstr`（hard）

内核的`copyin`函数读取用户指针指向的内存。它通过将用户指针转换为内核可以直接解引用的物理地址来实现这一点。这个转换是通过在软件中遍历进程页表来执行的。在本部分的实验中，您的工作是将用户空间的映射添加到每个进程的内核页表（上一节中创建），以允许`copyin`（和相关的字符串函数`copyinstr`）直接解引用用户指针。

> [!TIP|label:YOUR JOB]
> 将定义在***kernel/vm.c***中的`copyin`的主题内容替换为对`copyin_new`的调用（在***kernel/vmcopyin.c***中定义）；对`copyinstr`和`copyinstr_new`执行相同的操作。为每个进程的内核页表添加用户地址映射，以便`copyin_new`和`copyinstr_new`工作。如果`usertests`正确运行并且所有`make grade`测试都通过，那么你就完成了此项作业。

此方案依赖于用户的虚拟地址范围不与内核用于自身指令和数据的虚拟地址范围重叠。Xv6使用从零开始的虚拟地址作为用户地址空间，幸运的是内核的内存从更高的地址开始。然而，这个方案将用户进程的最大大小限制为小于内核的最低虚拟地址。内核启动后，在XV6中该地址是`0xC000000`，即PLIC寄存器的地址；请参见***kernel/vm.c***中的`kvminit()`、***kernel/memlayout.h***和文中的图3-4。您需要修改xv6，以防止用户进程增长到超过PLIC的地址。

**一些提示：**

- 先用对`copyin_new`的调用替换`copyin()`，确保正常工作后再去修改`copyinstr`
- 在内核更改进程的用户映射的每一处，都以相同的方式更改进程的内核页表。包括`fork()`, `exec()`, 和`sbrk()`.
- 不要忘记在`userinit`的内核页表中包含第一个进程的用户页表
- 用户地址的PTE在进程的内核页表中需要什么权限？(在内核模式下，无法访问设置了`PTE_U`的页面）
- 别忘了上面提到的PLIC限制

Linux使用的技术与您已经实现的技术类似。直到几年前，许多内核在用户和内核空间中都为当前进程使用相同的自身进程页表，并为用户和内核地址进行映射以避免在用户和内核空间之间切换时必须切换页表。然而，这种设置允许边信道攻击，如Meltdown和Spectre。

> [!NOTE|label:QUESTION]
> 解释为什么在`copyin_new()`中需要第三个测试`srcva + len < srcva`：给出`srcva`和`len`值的例子，这样的值将使前两个测试为假（即它们不会导致返回-1），但是第三个测试为真 （导致返回-1）。

### 可选的挑战练习

- 使用超级页来减少页表中PTE的数量
- 扩展您的解决方案以支持尽可能大的用户程序；也就是说，消除用户程序小于PLIC的限制
- 取消映射用户进程的第一页，以便使对空指针的解引用将导致错误。用户文本段必须从非0处开始，例如4096

---

## 二、我的实现

首先切换`pgtbl`分支

```shell
git fetch
git checkout pgtbl
make clean
```

这里使用`git fetch`老是出错，不过也不影响写lab。

![截屏2022-09-15 16.52.31](/img/user/knowledge/course/xv6-labs/attachments/img202209151652331.png)

### Print a page table

* 首先在`exec.c`中的`exec`函数添加以下代码。在第一个用户进程调用`exec`时使用`vmprint` 来打印该进程的用户页表。

  ```c
  if (p->pid==1) vmprint(p->pagetable);
  return argc; // this ends up in a0, the first argument to main(argc, argv)
  ```

  * 在`vm.c`里添加`vmprint`函数,使用递归的方法，打印各级的PTE。在写`pgprint`函数的时候，由于我不想其他文件访问该函数，所以添加了`static`.


  ```c
  static void pgprint(pagetable_t pagetable,int level){
    if (level<0) return;
    for (int i=0; i < 512; i++)
      {
        pte_t pte=pagetable[i];
        if ((pte & PTE_V)!=0){
          uint64 pa=PTE2PA(pte);
          for (int k = 2; k>level; k--)
          {
            printf(".. ");
          }
          printf("..%d: pte %p pa %p\n",i,pte,pa);
          pgprint((pagetable_t)pa,level-1);
        }
    }        
  }
  
  void vmprint(pagetable_t pagetable){
    printf("page table %p\n",pagetable);  
    pgprint(pagetable,2);
  }
  ```

* 需要在`def.h`里添加`vmprint`函数的定义，这样才可以在`exec`里调用

  ```c
  void            vmprint(pagetable_t);
  ```

在`userinit`中给第一个用户程序`initcode`创建了第一个用户进程，并为其分配了第一个进程号 `pid=1`,正是这个程序系统调用了`exec`。

### A kernel page table per process

* 首先在`proc.h`里的`proc`结构体中添加

```c
pagetable_t kpagetable;      // Kernel page table per process 
```

* 下面在`allocproc`中在每一个用户进程的`kpagetable`中填入和`kernel_pagetable`一样的映射。这里需要注意两点：

1. 用户的内核页表其他部分需要和全局的`kernel_pagetable`保持一致，但用户的内核栈却不必。事实上在全局内核页表的映射中，每个进程的内核栈自高到低地分布，因此建立映射时虚拟地址是这样的：

   ```c
    uint64 va = KSTACK((uint64)(p-proc));
   ```

   但用户的内核页表很自由，我干脆把每个用户进程的内核栈都放在`TRAMPOLINE`下面。

```c
	// A kernel page table of user process 
	p->kpagetable=newkvminit();
	if (p->kpagetable==0){
  	freeproc(p);
   	release(&p->lock);
   	return 0;
	}

  // Allocate a page for the process's kernel stack.
	char *pa = kalloc();
  if(pa == 0){
    freeproc(p);
    release(&p->lock);
    return 0;
  }
  uint64 va = KSTACK(0）;
  newkvmmap(p->kpagetable,va,(uint64)pa, PGSIZE, PTE_R | PTE_W);
  p->kstack=va;
```

​		还有一个很重要的问题，就是需要==修改`kvmpa` 函数==（这个函数用于内核栈地址的翻译）,尤其是在我修改了内核栈的虚拟地址以后，于是将

```c
  pte = walk(kernel_pagetable, va, 0);
```

修改为

```c
  pte = walk(myproc()->kpagetable, va, 0);
```

但会报错说结构体不完全，加上包含结构体信息的头文件就好了。

```c
#include "spinlock.h"
#include "proc.h"
```

2. 这里调用了我新写的函数`newkvminit`和`newkvmmap`,用来建立映射，这两个函数写在`vm.c`中。

```c

pagetable_t newkvminit(){
  pagetable_t kpagetable=(pagetable_t)kalloc();
  memset(kpagetable,0,PGSIZE);// every byte is 0
  // uart registers
  newkvmmap(kpagetable,UART0, UART0, PGSIZE, PTE_R | PTE_W);

  // virtio mmio disk interface
  newkvmmap(kpagetable,VIRTIO0, VIRTIO0, PGSIZE, PTE_R | PTE_W);

  // CLINT
  newkvmmap(kpagetable,CLINT, CLINT, 0x10000, PTE_R | PTE_W);

  // PLIC
  newkvmmap(kpagetable,PLIC, PLIC, 0x400000, PTE_R | PTE_W);

  // map kernel text executable and read-only.
  newkvmmap(kpagetable,KERNBASE, KERNBASE, (uint64)etext-KERNBASE, PTE_R | PTE_X);

  // map kernel data and the physical RAM we'll make use of.
  newkvmmap(kpagetable,(uint64)etext, (uint64)etext, PHYSTOP-(uint64)etext, PTE_R | PTE_W);

  // map the trampoline for trap entry/exit to
  // the highest virtual address in the kernel.
  newkvmmap(kpagetable,TRAMPOLINE, (uint64)trampoline, PGSIZE, PTE_R | PTE_X);  

  return  kpagetable;
}


void 
newkvmmap(pagetable_t kpagetable,uint64 va,uint64 pa,uint64 sz,int perm)
{
  if(mappages(kpagetable,va,sz,pa,perm)!=0)
    panic("newkvmmap");
}
```

* 还需要在`freeproc`添加一段释放用户的内核页表内存的代码

```c
if(p->kpagetable){
    uvmunmap(p->kpagetable,p->kstack,1,1);   
    p->kstack=0;
    freekpagetable(p->kpagetable);
}
```

直接借用用户页表`unmap`的函数，来看看`uvmunmap`的代码

```c
void
uvmunmap(pagetable_t pagetable, uint64 va, uint64 npages, int do_free)
{
  uint64 a;
  pte_t *pte;

  if((va % PGSIZE) != 0)
    panic("uvmunmap: not aligned");

  for(a = va; a < va + npages*PGSIZE; a += PGSIZE){
    if((pte = walk(pagetable, a, 0)) == 0)
      panic("uvmunmap: walk");
    if((*pte & PTE_V) == 0)
      panic("uvmunmap: not mapped");
    if(PTE_FLAGS(*pte) == PTE_V)
      panic("uvmunmap: not a leaf");
    if(do_free){
      uint64 pa = PTE2PA(*pte);
      kfree((void*)pa);
    }
    *pte = 0;
  }
}
```

简单来说，设`do_free`参数为`1`,所做的就是通过`walk`函数找到最低一级的PTE，释放掉指向的内核栈内存，然后将PTE归0。特别需要注意这里==并没有回收掉各级页表的内存==。这时候就轮到`freekpagetable`登场了,通过递归的方法，将每一级的PTE归0并释放掉当前级的页表。

```c
void freekpagetable(pagetable_t kpagetable){
  for (int i = 0; i < 512; i++)
  {
    if (kpagetable[i] & PTE_V){
      if ((kpagetable[i] & (PTE_R | PTE_W | PTE_X))==0){
        freekpagetable((pagetable_t)PTE2PA(kpagetable[i]));
      }
      kpagetable[i]=0;
    }
  }
  kfree((void *)kpagetable);
}
```



* 修改`scheduler`函数,使切换进程时更换页表。这部分的代码我有疑惑，保留下来。

```c
p->state = RUNNING;
c->proc = p;
w_satp(MAKE_SATP(p->kpagetable));//now satp is p->kpagetable
sfence_vma();
swtch(&c->context, &p->context);
w_satp(MAKE_SATP(kernel_pagetable));//now satp is kernel_kpagetable
sfence_vma();
```





很奇怪的是，到这里已经可以通过所有的`usertests`了，但是会出现一些`usertrap`





![截屏2022-09-17 18.42.04](/img/user/knowledge/course/xv6-labs/attachments/img202209171847256.png)



![截屏2022-09-17 18.44.37](/img/user/knowledge/course/xv6-labs/attachments/img202209171847898.png)

![截屏2022-09-17 18.44.00](/img/user/knowledge/course/xv6-labs/attachments/img202209171848675.png)

​	 

很奇怪的是，到这里已经可以通过所有的`usertests`了，但是









### Simplify `copyin`/`copyinstr`

---

## 三、参考答案与讲解

## 1. Print a page table

本实验主要是实现一个打印页表内容的函数， 首先根据提示在`exec.c`中的`return argc`之前插入`if(p->pid==1) vmprint(p->pagetable)`

然后看一下*kernel/vm.c*里面的`freewalk`方法，主要的代码如下：

```c
// Recursively free page-table pages.
// All leaf mappings must already have been removed.
void
freewalk(pagetable_t pagetable)
{
  // there are 2^9 = 512 PTEs in a page table.
  for(int i = 0; i < 512; i++){
    pte_t pte = pagetable[i];
    if((pte & PTE_V) && (pte & (PTE_R|PTE_W|PTE_X)) == 0){
      // this PTE points to a lower-level page table.
      uint64 child = PTE2PA(pte);
      freewalk((pagetable_t)child);
      pagetable[i] = 0;
    } else if(pte & PTE_V){
      panic("freewalk: leaf");
    }
  }
  kfree((void*)pagetable);
}
```

它首先会遍历整个页表。当遇到有效的页表项并且不在最后一层的时候，它会递归调用。`PTE_V`是用来判断页表项是否有效，而`(pte & (PTE_R|PTE_W|PTE_X)) == 0`则是用来判断是否不在最后一层。因为最后一层页表中页表项中W位，R位，X位起码有一位会被设置为1。注释里面说所有最后一层的页表项已经被释放了，所以遇到不符合的情况就`panic("freewalk: leaf")`。

那么，根据`freewalk`，我们可以写下递归函数。对于每一个有效的页表项都打印其和其子项的内容。如果不是最后一层的页表就继续递归。通过`level`来控制前缀`..`的数量。

```c
/**
 * @param pagetable 所要打印的页表
 * @param level 页表的层级
 */
void
_vmprint(pagetable_t pagetable, int level){
  // there are 2^9 = 512 PTEs in a page table.
  for(int i = 0; i < 512; i++){
    pte_t pte = pagetable[i];
    // PTE_V is a flag for whether the page table is valid
    if(pte & PTE_V){
      for (int j = 0; j < level; j++){
        if (j) printf(" ");
        printf("..");
      }
      uint64 child = PTE2PA(pte);
      printf("%d: pte %p pa %p\n", i, pte, child);
      if((pte & (PTE_R|PTE_W|PTE_X)) == 0){
        // this PTE points to a lower-level page table.
        _vmprint((pagetable_t)child, level + 1);
      }
    }
  }
}

/**
 * @brief vmprint 打印页表
 * @param pagetable 所要打印的页表
 */
void
vmprint(pagetable_t pagetable){
  printf("page table %p\n", pagetable);
  _vmprint(pagetable, 1);
}
```

最后记得加到*kernel/defs.h*里面。

```c
int             copyin(pagetable_t, char *, uint64, uint64);
int             copyinstr(pagetable_t, char *, uint64, uint64);
void            vmprint(pagetable_t);
```

重新跑一下`make qemu`，可以看到下面的输出：




## 2. A kernel page table per process

本实验主要是让每个进程都有自己的内核页表，这样在内核中执行时使用它自己的内核页表的副本。

**(1)**. 首先给*kernel/proc.h*里面的`struct proc`加上内核页表的字段。

```c
uint64 kstack;               // Virtual address of kernel stack
uint64 sz;                   // Size of process memory (bytes)
pagetable_t pagetable;       // User page table
pagetable_t kernelpt;      // 进程的内核页表
struct trapframe *trapframe; // data page for trampoline.S
```

**(2)**. 在`vm.c`中添加新的方法`proc_kpt_init`，该方法用于在`allocproc` 中初始化进程的内核页表。这个函数还需要一个辅助函数`uvmmap`，该函数和`kvmmap`方法几乎一致，不同的是`kvmmap`是对Xv6的内核页表进行映射，而`uvmmap`将用于进程的内核页表进行映射。

```c
// Just follow the kvmmap on vm.c
void
uvmmap(pagetable_t pagetable, uint64 va, uint64 pa, uint64 sz, int perm)
{
  if(mappages(pagetable, va, sz, pa, perm) != 0)
    panic("uvmmap");
}

// Create a kernel page table for the process
pagetable_t
proc_kpt_init(){
  pagetable_t kernelpt = uvmcreate();
  if (kernelpt == 0) return 0;
  uvmmap(kernelpt, UART0, UART0, PGSIZE, PTE_R | PTE_W);
  uvmmap(kernelpt, VIRTIO0, VIRTIO0, PGSIZE, PTE_R | PTE_W);
  uvmmap(kernelpt, CLINT, CLINT, 0x10000, PTE_R | PTE_W);
  uvmmap(kernelpt, PLIC, PLIC, 0x400000, PTE_R | PTE_W);
  uvmmap(kernelpt, KERNBASE, KERNBASE, (uint64)etext-KERNBASE, PTE_R | PTE_X);
  uvmmap(kernelpt, (uint64)etext, (uint64)etext, PHYSTOP-(uint64)etext, PTE_R | PTE_W);
  uvmmap(kernelpt, TRAMPOLINE, (uint64)trampoline, PGSIZE, PTE_R | PTE_X);
  return kernelpt;
}
```

然后在*kernel/proc.c*里面的`allocproc`调用。

```c
...
// An empty user page table.
p->pagetable = proc_pagetable(p);
if(p->pagetable == 0){
  freeproc(p);
  release(&p->lock);
  return 0;
}

// Init the kernal page table
p->kernelpt = proc_kpt_init();
if(p->kernelpt == 0){
  freeproc(p);
  release(&p->lock);
  return 0;
}
...
```

**(3)**. 根据提示，为了确保每一个进程的内核页表都关于该进程的内核栈有一个映射。我们需要将`procinit`方法中相关的代码迁移到`allocproc`方法中。很明显就是下面这段代码，将其剪切到上述内核页表初始化的代码后。

```
// Allocate a page for the process's kernel stack.
// Map it high in memory, followed by an invalid
// guard page.
char *pa = kalloc();
if(pa == 0)
  panic("kalloc");
uint64 va = KSTACK((int) (p - proc));
uvmmap(p->kernelpt, va, (uint64)pa, PGSIZE, PTE_R | PTE_W);
p->kstack = va;
```

**(4)**. 我们需要修改`scheduler()`来加载进程的内核页表到SATP寄存器。提示里面请求阅读`kvminithart()`。

```c
// Switch h/w page table register to the kernel's page table,
// and enable paging.
void
kvminithart()
{
  w_satp(MAKE_SATP(kernel_pagetable));
  sfence_vma();
}
```

`kvminithart`是用于原先的内核页表，我们将进程的内核页表传进去就可以。在*vm.c*里面添加一个新方法`proc_inithart`。

```c
// Store kernel page table to SATP register
void
proc_inithart(pagetable_t kpt){
  w_satp(MAKE_SATP(kpt));
  sfence_vma();
}
```

然后在`scheduler()`内调用即可，但在结束的时候，需要切换回原先的`kernel_pagetable`。直接调用调用上面的`kvminithart()`就能把Xv6的内核页表加载回去。

```c
...
p->state = RUNNING;
c->proc = p;

// Store the kernal page table into the SATP
proc_inithart(p->kernelpt);

swtch(&c->context, &p->context);

// Come back to the global kernel page table
kvminithart();
...
```

**(5)**. 在`freeproc`中释放一个进程的内核页表。首先释放页表内的内核栈，调用`uvmunmap`可以解除映射，最后的一个参数（`do_free`）为一的时候，会释放实际内存。

```c
// free the kernel stack in the RAM
uvmunmap(p->kernelpt, p->kstack, 1, 1);
p->kstack = 0;
```

然后释放进程的内核页表，先在*kernel/proc.c*里面添加一个方法`proc_freekernelpt`。如下，历遍整个内核页表，然后将所有有效的页表项清空为零。如果这个页表项不在最后一层的页表上，需要继续进行递归。

```c
void
proc_freekernelpt(pagetable_t kernelpt)
{
  // similar to the freewalk method
  // there are 2^9 = 512 PTEs in a page table.
  for(int i = 0; i < 512; i++){
    pte_t pte = kernelpt[i];
    if(pte & PTE_V){
      kernelpt[i] = 0;
      if ((pte & (PTE_R|PTE_W|PTE_X)) == 0){
        uint64 child = PTE2PA(pte);
        proc_freekernelpt((pagetable_t)child);
      }
    }
  }
  kfree((void*)kernelpt);
}
```

**(6)**. 将需要的函数定义添加到 `kernel/defs.h` 中

```c
// vm.c
void            kvminit(void);
pagetable_t     proc_kpt_init(void); // 用于内核页表的初始化
void            kvminithart(void); 
void            proc_inithart(pagetable_t); // 将进程的内核页表保存到SATP寄存器
...
```

**(7)**. 修改`vm.c`中的`kvmpa`，将原先的`kernel_pagetable`改成`myproc()->kernelpt`，使用进程的内核页表。

```c
#include "spinlock.h" 
#include "proc.h"

uint64
kvmpa(uint64 va)
{
  uint64 off = va % PGSIZE;
  pte_t *pte;
  uint64 pa;
  
  pte = walk(myproc()->kernelpt, va, 0); // 修改这里
  if(pte == 0)
    panic("kvmpa");
  if((*pte & PTE_V) == 0)
    panic("kvmpa");
  pa = PTE2PA(*pte);
  return pa+off;
}
```

**(8)**. 测试一下我们的代码，先跑起`qemu`，然后跑一下`usertests`。这部分耗时会比较长。

```bash
$ make qemu
> usertests
```

得到以下输出：




## 3. Simplify `copyin/copyinstr`

本实验是实现将用户空间的映射添加到每个进程的内核页表，将进程的页表复制一份到进程的内核页表就好。

首先添加复制函数。需要注意的是，在内核模式下，无法访问设置了`PTE_U`的页面，所以我们要将其移除。

```c
void
u2kvmcopy(pagetable_t pagetable, pagetable_t kernelpt, uint64 oldsz, uint64 newsz){
  pte_t *pte_from, *pte_to;
  oldsz = PGROUNDUP(oldsz);
  for (uint64 i = oldsz; i < newsz; i += PGSIZE){
    if((pte_from = walk(pagetable, i, 0)) == 0)
      panic("u2kvmcopy: src pte does not exist");
    if((pte_to = walk(kernelpt, i, 1)) == 0)
      panic("u2kvmcopy: pte walk failed");
    uint64 pa = PTE2PA(*pte_from);
    uint flags = (PTE_FLAGS(*pte_from)) & (~PTE_U);
    *pte_to = PA2PTE(pa) | flags;
  }
}
```

然后在内核更改进程的用户映射的每一处 （`fork()`, `exec()`, 和`sbrk()`），都复制一份到进程的内核页表。

- `exec()`：

```c
int
exec(char *path, char **argv){
  ...
  sp = sz;
  stackbase = sp - PGSIZE;

  // 添加复制逻辑
  u2kvmcopy(pagetable, p->kernelpt, 0, sz);

  // Push argument strings, prepare rest of stack in ustack.
  for(argc = 0; argv[argc]; argc++) {
  ...
}
```

- `fork()`:

```c
int
fork(void){
  ...
  // Copy user memory from parent to child.
  if(uvmcopy(p->pagetable, np->pagetable, p->sz) < 0){
    freeproc(np);
    release(&np->lock);
    return -1;
  }
  np->sz = p->sz;
  ...
  // 复制到新进程的内核页表
  u2kvmcopy(np->pagetable, np->kernelpt, 0, np->sz);
  ...
}
```

- `sbrk()`， 在*kernel/sysproc.c*里面找到`sys_sbrk(void)`，可以知道只有`growproc`是负责将用户内存增加或缩小 n 个字节。以防止用户进程增长到超过`PLIC`的地址，我们需要给它加个限制。

```c
int
growproc(int n)
{
  uint sz;
  struct proc *p = myproc();

  sz = p->sz;
  if(n > 0){
    // 加上PLIC限制
    if (PGROUNDUP(sz + n) >= PLIC){
      return -1;
    }
    if((sz = uvmalloc(p->pagetable, sz, sz + n)) == 0) {
      return -1;
    }
    // 复制一份到内核页表
    u2kvmcopy(p->pagetable, p->kernelpt, sz - n, sz);
  } else if(n < 0){
    sz = uvmdealloc(p->pagetable, sz, sz + n);
  }
  p->sz = sz;
  return 0;
}
```

然后替换掉原有的`copyin()`和`copyinstr()`

```c
// Copy from user to kernel.
// Copy len bytes to dst from virtual address srcva in a given page table.
// Return 0 on success, -1 on error.
int
copyin(pagetable_t pagetable, char *dst, uint64 srcva, uint64 len)
{
  return copyin_new(pagetable, dst, srcva, len);
}

// Copy a null-terminated string from user to kernel.
// Copy bytes to dst from virtual address srcva in a given page table,
// until a '\0', or max.
// Return 0 on success, -1 on error.
int
copyinstr(pagetable_t pagetable, char *dst, uint64 srcva, uint64 max)
{
  return copyinstr_new(pagetable, dst, srcva, max);
}
```

并且添加到 `kernel/defs.h` 中

```c
// vmcopyin.c
int             copyin_new(pagetable_t, char *, uint64, uint64);
int             copyinstr_new(pagetable_t, char *, uint64, uint64);
```

最后跑一下最终测试：

```bash
$ make grade
```

**注**：这里笔者加了*time.txt*和*answers-pgtbl.txt*来通过全部测试，不影响上述的代码实现。

