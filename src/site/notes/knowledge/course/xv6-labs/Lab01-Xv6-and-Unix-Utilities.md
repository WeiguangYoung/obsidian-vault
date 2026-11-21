---
{"dg-publish":true,"permalink":"/knowledge/course/xv6-labs/Lab01-Xv6-and-Unix-Utilities/","title":"6.S081 Lab1 · xv6 与 Unix 工具","tags":["MIT6.1810","xv6","操作系统","LAB","实验"],"dg-note-properties":{"title":"6.S081 Lab1 · xv6 与 Unix 工具","tags":["MIT6.1810","xv6","操作系统","LAB","实验"],"date":"2026-11-18"}}
---


# 6.S081 · Lab1 xv6 与 Unix 工具

> MIT 6.S081 / 6.1810 · Lab 01 — Xv6 and Unix Utilities
> 三部分：实验题目 / 我的实现 / 参考答案与讲解。

返回 [[knowledge/course/xv6-labs/index\|🧪 xv6-labs 目录]] · [[knowledge/course/mit6.s081/index\|📚 6.S081 课程]] · [[首页\|🏠 首页]] · [[knowledge/course/xv6-labs/Lab02-System-Calls\|下一实验 →]]

---

## 一、实验题目

### 实验任务

#### 启动xv6(难度：Easy)

获取实验室的xv6源代码并切换到util分支

```bash
$ git clone git://g.csail.mit.edu/xv6-labs-2020
Cloning into 'xv6-labs-2020'...
...
$ cd xv6-labs-2020
$ git checkout util
Branch 'util' set up to track remote branch 'util' from 'origin'.
Switched to a new branch 'util'
```

Xv6-labs-2020存储库与本书的xv6-riscv稍有不同;它主要添加一些文件。如果你好奇的话，可以执行`git log`:

```bash
$ git log
```

您将需要使用Git版本控制系统管理和提交文件以及后续的实验室作业。接下来，切换到一个分支(执行`git checkout util`)，其中包含针对该实验室定制的xv6版本。要了解关于Git的更多信息，请查看Git用户手册。Git允许您跟踪对代码所做的更改。例如，如果你完成了其中一个练习，并且想检查你的进度，你可以通过运行以下命令来提交你的变化:

```bash
$ git commit -am 'my solution for util lab exercise 1'
Created commit 60d2135: my solution for util lab exercise 1
 1 files changed, 1 insertions(+), 0 deletions(-)
$
```

您可以使用`git diff`命令跟踪您的更改。运行`git diff`将显示自上次提交以来对代码的更改，`git diff origin/util`将显示相对于初始xv6-labs-2020代码的更改。这里，***origin/xv6-labs-2020***是git分支的名称，它是包含您下载的初始代码分支。

- **构建并运行xv6**

```bash
$ make qemu
riscv64-unknown-elf-gcc    -c -o kernel/entry.o kernel/entry.S
riscv64-unknown-elf-gcc -Wall -Werror -O -fno-omit-frame-pointer -ggdb -DSOL_UTIL -MD -mcmodel=medany -ffreestanding -fno-common -nostdlib -mno-relax -I. -fno-stack-protector -fno-pie -no-pie   -c -o kernel/start.o kernel/start.c
...
init: starting sh
$ 
```

如果你在提示符下输入 `ls`，你会看到类似如下的输出:

```bash
$ ls
.              1 1 1024
..             1 1 1024
...
zombie         2 19 22408
console        3 20 0
```

这些是`mkfs`在初始文件系统中包含的文件；大多数是可以运行的程序。你刚刚跑了其中一个：`ls`。

xv6没有`ps`命令，但是如果您键入Ctrl-p，内核将打印每个进程的信息。如果现在尝试，您将看到两行：一行用于init，另一行用于sh。

退出 qemu : `Ctrl-a x`。

#### sleep(难度：Easy)

> [!TIP|label:YOUR JOB]
> **实现xv6的UNIX程序**`sleep`**：您的**`sleep`**应该暂停到用户指定的计时数。一个滴答(tick)是由xv6内核定义的时间概念，即来自定时器芯片的两个中断之间的时间。您的解决方案应该在文件*user/sleep.c*中**

**提示：**

- 在你开始编码之前，请阅读《book-riscv-rev1》的第一章

- 看看其他的一些程序（如***/user/echo.c, /user/grep.c, /user/rm.c***）查看如何获取传递给程序的命令行参数

- 如果用户忘记传递参数，`sleep`应该打印一条错误信息

- 命令行参数作为字符串传递; 您可以使用`atoi`将其转换为数字（详见***/user/ulib.c***）

- 使用系统调用`sleep`

- 请参阅***kernel/sysproc.c***以获取实现`sleep`系统调用的xv6内核代码（查找`sys_sleep`），***user/user.h***提供了`sleep`的声明以便其他程序调用，用汇编程序编写的***user/usys.S***可以帮助`sleep`从用户区跳转到内核区。

- 确保`main`函数调用`exit()`以退出程序。

- 将你的`sleep`程序添加到***Makefile***中的`UPROGS`中；完成之后，`make qemu`将编译您的程序，并且您可以从xv6的shell运行它。

- 看看Kernighan和Ritchie编著的《C程序设计语言》（第二版）来了解C语言。

从xv6 shell运行程序：

```bash
$ make qemu
...
init: starting sh
$ sleep 10
(nothing happens for a little while)
$
```

如果程序在如上所示运行时暂停，则解决方案是正确的。运行`make grade`看看你是否真的通过了睡眠测试。

请注意，`make grade`运行所有测试，包括下面作业的测试。如果要对一项作业运行成绩测试，请键入（不要启动XV6，在外部终端下使用）：

```bash
$ ./grade-lab-util sleep
```

这将运行与`sleep`匹配的成绩测试。或者，您可以键入：

```bash
$ make GRADEFLAGS=sleep grade
```

效果是一样的。

#### pingpong（难度：Easy）

> [!TIP|label:YOUR JOB]
> **编写一个使用UNIX系统调用的程序来在两个进程之间“ping-pong”一个字节，请使用两个管道，每个方向一个。父进程应该向子进程发送一个字节;子进程应该打印“`<pid>: received ping`”，其中`<pid>`是进程ID，并在管道中写入字节发送给父进程，然后退出;父级应该从读取从子进程而来的字节，打印“`<pid>: received pong`”，然后退出。您的解决方案应该在文件*user/pingpong.c*中。**

**提示：**

- 使用`pipe`来创造管道

- 使用`fork`创建子进程

- 使用`read`从管道中读取数据，并且使用`write`向管道中写入数据

- 使用`getpid`获取调用进程的pid

- 将程序加入到***Makefile***的`UPROGS`

- xv6上的用户程序有一组有限的可用库函数。您可以在***user/user.h***中看到可调用的程序列表；源代码（系统调用除外）位于***user/ulib.c***、***user/printf.c***和***user/umalloc.c***中。

运行程序应得到下面的输出

```bash
$ make qemu
...
init: starting sh
$ pingpong
4: received ping
3: received pong
$
```

如果您的程序在两个进程之间交换一个字节并产生如上所示的输出，那么您的解决方案是正确的。

#### Primes(素数，难度：Moderate/Hard)

> [!TIP|label:YOUR JOB]
> **使用管道编写**`prime sieve`**(筛选素数)的并发版本。这个想法是由Unix管道的发明者Doug McIlroy提出的。请查看[这个网站](http://swtch.com/~rsc/thread/)(翻译在下面)，该网页中间的图片和周围的文字解释了如何做到这一点。您的解决方案应该在*user/primes.c*文件中。**

您的目标是使用`pipe`和`fork`来设置管道。第一个进程将数字2到35输入管道。对于每个素数，您将安排创建一个进程，该进程通过一个管道从其左邻居读取数据，并通过另一个管道向其右邻居写入数据。由于xv6的文件描述符和进程数量有限，因此第一个进程可以在35处停止。

**提示：**

- 请仔细关闭进程不需要的文件描述符，否则您的程序将在第一个进程达到35之前就会导致xv6系统资源不足。

- 一旦第一个进程达到35，它应该使用`wait`等待整个管道终止，包括所有子孙进程等等。因此，主`primes`进程应该只在打印完所有输出之后，并且在所有其他`primes`进程退出之后退出。

- 提示：当管道的`write`端关闭时，`read`返回零。

- 最简单的方法是直接将32位（4字节）int写入管道，而不是使用格式化的ASCII I/O。

- 您应该仅在需要时在管线中创建进程。

- 将程序添加到***Makefile***中的`UPROGS`

如果您的解决方案实现了基于管道的筛选并产生以下输出，则是正确的：

```bash
$ make qemu
...
init: starting sh
...
prime 31
$
```



**参考资料翻译：**

> 考虑所有小于1000的素数的生成。Eratosthenes的筛选法可以通过执行以下伪代码的进程管线来模拟：

```c
p = get a number from left neighbor
print p
loop:
    n = get a number from left neighbor
    if (p does not divide n)
        send n to right neighbor
p = 从左邻居中获取一个数
print p
loop:
    n = 从左邻居中获取一个数
    if (n不能被p整除)
        将n发送给右邻居
```


> 生成进程可以将数字2、3、4、…、1000输入管道的左端：行中的第一个进程消除2的倍数，第二个进程消除3的倍数，第三个进程消除5的倍数，依此类推。

#### find（难度：Moderate）

> [!TIP|label:YOUR JOB]
> **写一个简化版本的UNIX的`find`程序：查找目录树中具有特定名称的所有文件，你的解决方案应该放在*user/find.c***

提示：

- 查看***user/ls.c***文件学习如何读取目录
- 使用递归允许`find`下降到子目录中
- 不要在“`.`”和“`..`”目录中递归
- 对文件系统的更改会在qemu的运行过程中一直保持；要获得一个干净的文件系统，请运行`make clean`，然后`make qemu`
- 你将会使用到C语言的字符串，要学习它请看《C程序设计语言》（K&R）,例如第5.5节
- 注意在C语言中不能像python一样使用“`==`”对字符串进行比较，而应当使用`strcmp()`
- 将程序加入到Makefile的`UPROGS`

如果你的程序输出下面的内容，那么它是正确的（当文件系统中包含文件***b***和***a/b***的时候）

```bash
$ make qemu
...
init: starting sh
$ echo > b
$ mkdir a
$ echo > a/b
$ find . b
./b
./a/b
$ 
```

#### xargs（难度：Moderate）

> [!TIP|label:YOUR JOB]
> 编写一个简化版UNIX的`xargs`程序：它从标准输入中按行读取，并且为每一行执行一个命令，将行作为参数提供给命令。你的解决方案应该在***user/xargs.c***

下面的例子解释了`xargs`的行为

```bash
$ echo hello too | xargs echo bye
bye hello too
$
```

注意，这里的命令是`echo bye`，额外的参数是`hello too`，这样就组成了命令`echo bye hello too`，此命令输出`bye hello too`

请注意，UNIX上的`xargs`进行了优化，一次可以向该命令提供更多的参数。 我们不需要您进行此优化。 要使UNIX上的`xargs`表现出本实验所实现的方式，请将`-n`选项设置为1。例如

```bash
$ echo "1\n2" | xargs -n 1 echo line
line 1
line 2
$
```

**提示：**

- 使用`fork`和`exec`对每行输入调用命令，在父进程中使用`wait`等待子进程完成命令。
- 要读取单个输入行，请一次读取一个字符，直到出现换行符（'\n'）。
- ***kernel/param.h***声明`MAXARG`，如果需要声明`argv`数组，这可能很有用。
- 将程序添加到***Makefile***中的`UPROGS`。
- 对文件系统的更改会在qemu的运行过程中保持不变；要获得一个干净的文件系统，请运行`make clean`，然后`make qemu`



`xargs`、`find`和`grep`结合得很好

```bash
$ find . b | xargs grep hello
```

将对“`.`”下面的目录中名为***b***的每个文件运行`grep hello`。

要测试您的`xargs`方案是否正确，请运行shell脚本***xargstest.sh***。如果您的解决方案产生以下输出，则是正确的：

```bash
$ make qemu
...
init: starting sh
$ sh < xargstest.sh
$ $ $ $ $ $ hello
hello
hello
$ $   
```

你可能不得不回去修复你的`find`程序中的bug。输出有许多`$`，因为xv6 shell没有意识到它正在处理来自文件而不是控制台的命令，并为文件中的每个命令打印`$`。

### 提交实验

**这就完成了实验**。确保你通过了所有的成绩测试。如果这个实验有问题，别忘了把你的答案写在***answers-lab-name.txt***中***。***提交你的更改（包括***answers-lab-name.txt***），然后在实验目录中键入`make handin`以提交实验。

**花费的时间**

创建一个命名为***time.txt***的新文件，并在其中输入一个整数，即您在实验室花费的小时数。不要忘记`git add`和`git commit`文件。

**提交**

你将使用**[提交网站](https://6828.scripts.mit.edu/2020/handin.py/)**提交作业。您需要从提交网站请求一次API密钥，然后才能提交任何作业或实验。

将最终更改提交到实验后，键入`make handin`以提交实验。

```bash
$ git commit -am "ready to submit my lab"
[util c2e3c8b] ready to submit my lab
 2 files changed, 18 insertions(+), 2 deletions(-)

$ make handin
tar: Removing leading `/' from member names
Get an API key for yourself by visiting https://6828.scripts.mit.edu/2020/handin.py/
Please enter your API key: XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100 79258  100   239  100 79019    853   275k --:--:-- --:--:-- --:--:--  276k
$
```

`make handin`将把你的API密钥存储在***myapi.key***中。如果需要更改API密钥，只需删除此文件并让`make handin`再次生成它(***myapi.key***不得包含换行符）。

如果你运行了`make handin`，并且你有未提交的更改或未跟踪的文件，则会看到类似于以下内容的输出：

```
 M hello.c
?? bar.c
?? foo.pyc
Untracked files will not be handed in.  Continue? [y/N]
```

检查上述行，确保跟踪了您的实验解决方案所需的所有文件，即以??开头的行中所显示的文件。您可以使用`git add filename`命令使git追踪创建的新文件。

如果`make handin`无法正常工作，请尝试使用curl或Git命令修复该问题。或者你可以运行`make tarball`。这将为您制作一个tar文件，然后您可以通过我们的web界面上传。

- 请运行“`make grade`”以确保您的代码通过所有测试
- 在运行“`make handin`”之前提交任何修改过的源代码`
- 您可以检查提交的状态，并在以下位置下载提交的代码：https://6828.scripts.mit.edu/2020/handin.py/

### 可选的挑战练习

- 编写一个`uptime`程序，使用`uptime`系统调用以滴答为单位打印计算机正常运行时间。（easy）

- 在`find`程序的名称匹配中支持正则表达式。***grep.c***对正则表达式有一些基本的支持。（easy）

- xv6 shell（***user/sh.c***）只是另一个用户程序，您可以对其进行改进。它是一个最小的shell，缺少建立在真实shell中的许多特性。例如，

  - 在处理**文件中的**shell命令时，将shell修改为不打印$（moderate）
  - 将shell修改为支持`wait`（easy）
  - 将shell修改为支持用“`;`”分隔的命令列表（moderate）
  - 通过实现左括号“`(`” 以及右括号“`)`”来修改shell以支持子shell（moderate）
  - 将shell修改为支持`tab`键补全（easy）
  - 修改shell使其支持命令历史记录（moderate）
  - 或者您希望shell执行的任何其他操作。

- 如果您非常雄心勃勃，可能需要修改内核以支持所需的内核特性；xv6支持的并不多。

---

## 二、我的实现

xv6我运行虚拟环境的是*ubuntu64位20.04.4*,最好不要使用更新的版本，很有可能*make qemu*之后无法运行的情况。

### sleep

```c
#include"kernel/types.h"
#include"kernel/stat.h"
#include"user/user.h"


int
main(int argc, char const *argv[])
{
    if (argc<2){
        printf("You should print the sleeping time\n");
        exit(1);
    }    

    int time=atoi(argv[1]);

    printf("(nothing happens for a little while )\n");
    sleep(time);

    exit(0);

}
```

运行没有问题，但是我似乎需要学着将错误信息这样输出。

```c
fprintf(2,"usage : sleep <time>\n");
```

### pingpong

这是我写的最初版本，特别需要注意**那就是父子进程中不使用的pipe的文件描述符尽快close掉，特别是写端**。不然数据为空的管道read会堵塞。

```c
#include"kernel/types.h"
#include"kernel/stat.h"
#include"user/user.h"

int main(){
    int  p1[2];
    int  p2[2];
    pipe(p1);
    pipe(p2);
    int pid;
    char buf1[10];
    char buf2[10];
    if (fork()>0){//主进程
        close(p2[1]);
        close(p1[0]);
        write(p1[1],"p",1);
        close(p1[1]);
        if (read(p2[0],buf1,1)==1){
            close(p2[0]);
            pid=getpid();
            printf("%d: received pong\n",pid);
        }
    }else{
        close(p1[1]);
        close(p2[0]);
        if (read(p1[0],buf2,1)==1){
            close(p1[0]);
            pid=getpid();
            printf("%d: received ping\n",pid);
            write(p2[1],"p",1);
            close(p2[1]);
            exit(0);
        }
    }
    exit(0);
}
```



这是work的，但是我对比了其他人写的代码，缺少对于系统调用出现错误的反馈。

### primes

使用递归的方法，第一次写的时候没有设置递归基，所以陷入了无限递归，一直在创建新的进程，最后直接资源不够卡住了。

```c
#include"kernel/types.h"
#include"kernel/stat.h"
#include"user/user.h"

#define RD 0
#define WR 1

void prime(int p1[2]){
    close(p1[WR]);
    int p;
    if (read(p1[RD],&p,sizeof(int))!=0){
        fprintf(1,"prime %d\n",p);
    }else{
        exit(0);
    }
    int n;
    int p2[2];
    pipe(p2);
    int pid=fork();
    if (pid>0){
        close(p2[RD]);
        while (read(p1[RD],&n,sizeof(int))!=0)
        {
            if (n%p!=0){
                write(p2[WR],&n,sizeof(int));
            }
        }
        close(p1[RD]);
        close(p2[WR]);
        wait((int *)0);
        exit(0);
    }else if (pid==0){
        prime(p2);
    }
    exit(0);
}


int main(){
    int p1[2];
    pipe(p1);
    int pid=fork();
    if (pid<0){
        fprintf(2,"fork error!\n");
        close(p1[RD]);
        close(p1[WR]);
        exit(1);
    }else if (pid==0){
        prime(p1);
    }else{//main proc
        close(p1[RD]);
        for (int i = 2; i <=35 ; i++)
        {
            write(p1[WR],&i,sizeof(int));     
        }
        close(p1[WR]);
        wait((int *)0);
    }

    exit(0);
    
}
```





### find

（该程序需要查询一个目录树下有特定名称的所有文件）

在测试该程序的时候，需要创建b文件和a/b文件，手册上是这么做的。

```shell
$ echo > b
$ mkdir a 
$ echo > a/b
```

 其实我有点奇怪的是`echo`的代码是这样的，只是做了输出而已。它是怎么创建文件的？

<img src="http://cdn.zhengyanchen.cn/img202206191105442.png" alt="截屏2022-06-19 11.05.15" style="zoom: 40%;" />

稍微思考了，这个操作应该是在`sh.c`做。事实上,`sh.c`也确实这么做了并做了重定向，让文件描述符1重定向到的文件b。

<img src="http://cdn.zhengyanchen.cn/img202206191110226.png" alt="截屏2022-06-19 11.10.44" style="zoom:40%;" />

现在我来重新理解文件描述符：

1. 所有的文件描述符核心都是在服务进程
2. 一个文件可以对应多个文件描述符
3. 可以把文件描述符理解为此进程打开此文件的程度（偏移量）。

回到该lab，要找目录的特定文件，首先阅读`ls.c`。(`.`表示当前目录，`..`表示下上一级目), 然后就可以写了。



```c
#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"
#include "kernel/fs.h"

void find(char* path,char * file){
    int fd;
    struct stat st;
    struct dirent de;
    char buf[512];
    char *p;
    if((fd=open(path,0))<0){
        fprintf(2,"can not open%d\n",path);
        exit(1);
    }
 
    if ((fstat(fd,&st))<0){
        fprintf(2,"can not stat%d\n",path);
        close(fd);
        exit(1);        
    }
    switch (st.type)
    {
    case T_FILE:
        fprintf(2,"please find file in dir");
        close(fd);
        break;
    case T_DIR:
        strcpy(buf, path);
        p = buf+strlen(buf);
        *p++ = '/';
        while(read(fd, &de, sizeof(de)) == sizeof(de)){
            if(de.inum == 0)
            continue;
            memmove(p, de.name, DIRSIZ);
            if(stat(buf, &st) < 0){
                fprintf(2,"find: cannot stat %s\n", buf);
                continue;
            }
            if (st.type==T_DIR){
                if (de.name[0]=='.'){
                    continue;
                }else{
                    find(buf,file);
                }
            }else if(st.type==T_FILE){
                if (strcmp(file,de.name)==0 ){
                    fprintf(1,"%s\n",buf);
                }
            }            
        }
    }

    return ;

}

int main(char argc,char *argv[]){
    if (argc>3){
        fprintf(2,"need less argument\n");
        exit(1);
    }else if (argc==2){
        find(".",argv[1]);
    }else{
        find(argv[1],argv[2]);
    }   
    exit(0); 
}
```





### xargs

```c
#include"kernel/types.h"
#include"kernel/stat.h"
#include"user/user.h"
#include"kernel/fcntl.h"
#include"kernel/fs.h"
#include"kernel/param.h"
int main(char argc,char * argv[]){
    char buf;
    char command[DIRSIZ];
    char *args[MAXARG]={0};
    int argnum=0;
    if (argc>1){
        strcpy(command,argv[1]);
        while (argnum<(argc-1))
        {
            args[argnum]=argv[argnum+1];
            argnum++;
        }
        char line[128];
        int linenum=0;
        while (read(0,&buf,1)>0)
        {
            if (buf=='\n'){
                line[linenum]=0;
                args[argnum]=line;
                int pid=fork();
                if(pid<0){
                    fprintf(2,"xargs: fork error");
                }else if (pid==0){
                    exec(command,args);
                    exit(0);
                }else{
                    linenum=0;
                    wait((int *)0);
                }
            }else{
                line[linenum++]=buf;
            }
        }
    }else{
        fprintf(2,"a command shoud be given");
        exit(1);
    }
    
    exit(0);
}


```

---

## 三、参考答案与讲解

### sleep

这个简单，不多说了

```c
#include "kernel/types.h"
#include "user/user.h"

int main(int argc, char const *argv[])
{
  if (argc != 2) { //参数错误
    fprintf(2, "usage: sleep <time>\n");
    exit(1);
  }
  sleep(atoi(argv[1]));
  exit(0);
}
```

### pingpong

使用两个管道进行父子进程通信，需要注意的是如果管道的写端没有`close`，那么管道中数据为空时对管道的读取将会阻塞。因此对于不需要的管道描述符，要尽可能早的关闭。

```c
#include "kernel/types.h"
#include "user/user.h"

#define RD 0 //pipe的read端
#define WR 1 //pipe的write端

int main(int argc, char const *argv[]) {
    char buf = 'P'; //用于传送的字节

    int fd_c2p[2]; //子进程->父进程
    int fd_p2c[2]; //父进程->子进程
    pipe(fd_c2p);
    pipe(fd_p2c);

    int pid = fork();
    int exit_status = 0;

    if (pid < 0) {
        fprintf(2, "fork() error!\n");
        close(fd_c2p[RD]);
        close(fd_c2p[WR]);
        close(fd_p2c[RD]);
        close(fd_p2c[WR]);
        exit(1);
    } else if (pid == 0) { //子进程
        close(fd_p2c[WR]);
        close(fd_c2p[RD]);

        if (read(fd_p2c[RD], &buf, sizeof(char)) != sizeof(char)) {
            fprintf(2, "child read() error!\n");
            exit_status = 1; //标记出错
        } else {
            fprintf(1, "%d: received ping\n", getpid());
        }

        if (write(fd_c2p[WR], &buf, sizeof(char)) != sizeof(char)) {
            fprintf(2, "child write() error!\n");
            exit_status = 1;
        }

        close(fd_p2c[RD]);
        close(fd_c2p[WR]);

        exit(exit_status);
    } else { //父进程
        close(fd_p2c[RD]);
        close(fd_c2p[WR]);

        if (write(fd_p2c[WR], &buf, sizeof(char)) != sizeof(char)) {
            fprintf(2, "parent write() error!\n");
            exit_status = 1;
        }

        if (read(fd_c2p[RD], &buf, sizeof(char)) != sizeof(char)) {
            fprintf(2, "parent read() error!\n");
            exit_status = 1; //标记出错
        } else {
            fprintf(1, "%d: received pong\n", getpid());
        }

        close(fd_p2c[WR]);
        close(fd_c2p[RD]);

        exit(exit_status);
    }
}
```

### primes

这个感觉还是有些难度的，它的思想是多进程版本的递归，不断地将左邻居管道中的数据筛选后传送给右邻居，每次传送的第一个数据都将是一个素数。

具体还是看代码吧，里面注释应该还是比较清楚的

```c
#include "kernel/types.h"
#include "user/user.h"

#define RD 0
#define WR 1

const uint INT_LEN = sizeof(int);

/**
 * @brief 读取左邻居的第一个数据
 * @param lpipe 左邻居的管道符
 * @param pfirst 用于存储第一个数据的地址
 * @return 如果没有数据返回-1,有数据返回0
 */
int lpipe_first_data(int lpipe[2], int *dst)
{
  if (read(lpipe[RD], dst, sizeof(int)) == sizeof(int)) {
    printf("prime %d\n", *dst);
    return 0;
  }
  return -1;
}

/**
 * @brief 读取左邻居的数据，将不能被first整除的写入右邻居
 * @param lpipe 左邻居的管道符
 * @param rpipe 右邻居的管道符
 * @param first 左邻居的第一个数据
 */
void transmit_data(int lpipe[2], int rpipe[2], int first)
{
  int data;
  // 从左管道读取数据
  while (read(lpipe[RD], &data, sizeof(int)) == sizeof(int)) {
    // 将无法整除的数据传递入右管道
    if (data % first)
      write(rpipe[WR], &data, sizeof(int));
  }
  close(lpipe[RD]);
  close(rpipe[WR]);
}

/**
 * @brief 寻找素数
 * @param lpipe 左邻居管道
 */
void primes(int lpipe[2])
{
  close(lpipe[WR]);
  int first;
  if (lpipe_first_data(lpipe, &first) == 0) {
    int p[2];
    pipe(p); // 当前的管道
    transmit_data(lpipe, p, first);

    if (fork() == 0) {
      primes(p);    // 递归的思想，但这将在一个新的进程中调用
    } else {
      close(p[RD]);
      wait(0);
    }
  }
  exit(0);
}

int main(int argc, char const *argv[])
{
  int p[2];
  pipe(p);

  for (int i = 2; i <= 35; ++i) //写入初始数据
    write(p[WR], &i, INT_LEN);

  if (fork() == 0) {
    primes(p);
  } else {
    close(p[WR]);
    close(p[RD]);
    wait(0);
  }

  exit(0);
}
```

### find

感觉没什么好说的0.0，代码基本上都是COPY的*ls.c*中的内容

```c
#include "kernel/types.h"

#include "kernel/fs.h"
#include "kernel/stat.h"
#include "user/user.h"

void find(char *path, const char *filename)
{
  char buf[512], *p;
  int fd;
  struct dirent de;
  struct stat st;

  if ((fd = open(path, 0)) < 0) {
    fprintf(2, "find: cannot open %s\n", path);
    return;
  }

  if (fstat(fd, &st) < 0) {
    fprintf(2, "find: cannot fstat %s\n", path);
    close(fd);
    return;
  }

  //参数错误，find的第一个参数必须是目录
  if (st.type != T_DIR) {
    fprintf(2, "usage: find <DIRECTORY> <filename>\n");
    return;
  }

  if (strlen(path) + 1 + DIRSIZ + 1 > sizeof buf) {
    fprintf(2, "find: path too long\n");
    return;
  }
  strcpy(buf, path);
  p = buf + strlen(buf);
  *p++ = '/'; //p指向最后一个'/'之后
  while (read(fd, &de, sizeof de) == sizeof de) {
    if (de.inum == 0)
      continue;
    memmove(p, de.name, DIRSIZ); //添加路径名称
    p[DIRSIZ] = 0;               //字符串结束标志
    if (stat(buf, &st) < 0) {
      fprintf(2, "find: cannot stat %s\n", buf);
      continue;
    }
    //不要在“.”和“..”目录中递归
    if (st.type == T_DIR && strcmp(p, ".") != 0 && strcmp(p, "..") != 0) {
      find(buf, filename);
    } else if (strcmp(filename, p) == 0)
      printf("%s\n", buf);
  }

  close(fd);
}

int main(int argc, char *argv[])
{
  if (argc != 3) {
    fprintf(2, "usage: find <directory> <filename>\n");
    exit(1);
  }
  find(argv[1], argv[2]);
  exit(0);
}
```

### xargs

之前这个题目我是做错的，但是仍然通过了测试。需要注意的是题目中要求为每一行执行一个命令。之前刷力扣的时候处理字符串好几次用到了有限状态自动机，虽然写的代码比较多，但是只要搞清楚逻辑，这种方法反而比较容易写出来。

xv6中的`echo`命令并不能输出换行符，例如在xv6和linux中执行命令

xv6执行`echo "1\n2"`输出

```text
"1\n2"
```

linux执行`echo -e "1\n2"`输出（`-e`启用转义）

```text
1
2
```

因此没有想到如何在xv6中验证`xargs`的正确性，于是我将类似（类似是因为头文件不同，且将`exec`替换为了`execvp`）的代码*xargs.c*在linux下编译并测试运行，如下图所示：第一条命令是使用linux中`xargs`的输出，第二条命令是使用自己写的`xargs`的输出，二者是一致的。


有限状态自动机主要就是一系列的状态转换，例如对于

```text
 1 \n 23
0123 456
```

来说，第一行是待读取的字符串，第二行是字符下标，起始时状态为`S_WAIT`，状态转换如下

```text
读取到0处的空格  状态由S_WAIT变为S_WAIT，继续等待参数到来（arg_beg前移）
读取到1处的字符1 状态由S_WAIT变为S_ARG，开始读取参数（arg_beg不动）
读取到2处的空格  状态由S_ARG变为S_ARG_END，储存参数地址（将lines[arg_beg]的地址存入x_argv中）
读取到3处的换行  状态由S_ARG_END变为S_LINE_END，fork后执行程序
...以此类推
```

```c
#include "kernel/types.h"
#include "user/user.h"
#include "kernel/param.h"

#define MAXSZ 512
// 有限状态自动机状态定义
enum state {
  S_WAIT,         // 等待参数输入，此状态为初始状态或当前字符为空格
  S_ARG,          // 参数内
  S_ARG_END,      // 参数结束
  S_ARG_LINE_END, // 左侧有参数的换行，例如"arg\n"
  S_LINE_END,     // 左侧为空格的换行，例如"arg  \n""
  S_END           // 结束，EOF
};

// 字符类型定义
enum char_type {
  C_SPACE,
  C_CHAR,
  C_LINE_END
};

/**
 * @brief 获取字符类型
 *
 * @param c 待判定的字符
 * @return enum char_type 字符类型
 */
enum char_type get_char_type(char c)
{
  switch (c) {
  case ' ':
    return C_SPACE;
  case '\n':
    return C_LINE_END;
  default:
    return C_CHAR;
  }
}

/**
 * @brief 状态转换
 *
 * @param cur 当前的状态
 * @param ct 将要读取的字符
 * @return enum state 转换后的状态
 */
enum state transform_state(enum state cur, enum char_type ct)
{
  switch (cur) {
  case S_WAIT:
    if (ct == C_SPACE)    return S_WAIT;
    if (ct == C_LINE_END) return S_LINE_END;
    if (ct == C_CHAR)     return S_ARG;
    break;
  case S_ARG:
    if (ct == C_SPACE)    return S_ARG_END;
    if (ct == C_LINE_END) return S_ARG_LINE_END;
    if (ct == C_CHAR)     return S_ARG;
    break;
  case S_ARG_END:
  case S_ARG_LINE_END:
  case S_LINE_END:
    if (ct == C_SPACE)    return S_WAIT;
    if (ct == C_LINE_END) return S_LINE_END;
    if (ct == C_CHAR)     return S_ARG;
    break;
  default:
    break;
  }
  return S_END;
}


/**
 * @brief 将参数列表后面的元素全部置为空
 *        用于换行时，重新赋予参数
 *
 * @param x_argv 参数指针数组
 * @param beg 要清空的起始下标
 */
void clearArgv(char *x_argv[MAXARG], int beg)
{
  for (int i = beg; i < MAXARG; ++i)
    x_argv[i] = 0;
}

int main(int argc, char *argv[])
{
  if (argc - 1 >= MAXARG) {
    fprintf(2, "xargs: too many arguments.\n");
    exit(1);
  }
  char lines[MAXSZ];
  char *p = lines;
  char *x_argv[MAXARG] = {0}; // 参数指针数组，全部初始化为空指针

  // 存储原有的参数
  for (int i = 1; i < argc; ++i) {
    x_argv[i - 1] = argv[i];
  }
  int arg_beg = 0;          // 参数起始下标
  int arg_end = 0;          // 参数结束下标
  int arg_cnt = argc - 1;   // 当前参数索引
  enum state st = S_WAIT;   // 起始状态置为S_WAIT

  while (st != S_END) {
    // 读取为空则退出
    if (read(0, p, sizeof(char)) != sizeof(char)) {
      st = S_END;
    } else {
      st = transform_state(st, get_char_type(*p));
    }

    if (++arg_end >= MAXSZ) {
      fprintf(2, "xargs: arguments too long.\n");
      exit(1);
    }

    switch (st) {
    case S_WAIT:          // 这种情况下只需要让参数起始指针前移
      ++arg_beg;
      break;
    case S_ARG_END:       // 参数结束，将参数地址存入x_argv数组中
      x_argv[arg_cnt++] = &lines[arg_beg];
      arg_beg = arg_end;
      *p = '\0';          // 替换为字符串结束符
      break;
    case S_ARG_LINE_END:  // 将参数地址存入x_argv数组中同时执行指令
      x_argv[arg_cnt++] = &lines[arg_beg];
      // 不加break，因为后续处理同S_LINE_END
    case S_LINE_END:      // 行结束，则为当前行执行指令
      arg_beg = arg_end;
      *p = '\0';
      if (fork() == 0) {
        exec(argv[1], x_argv);
      }
      arg_cnt = argc - 1;
      clearArgv(x_argv, arg_cnt);
      wait(0);
      break;
    default:
      break;
    }

    ++p;    // 下一个字符的存储位置后移
  }
  exit(0);
}
```
