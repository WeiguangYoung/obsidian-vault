---
{"dg-publish":true,"permalink":"/knowledge/course/csapp-labs/06-Shell-Lab/","title":"CSAPP Lab6 · Shell Lab","tags":["CSAPP","LAB","实验","进程","信号","作业控制"],"dg-note-properties":{"title":"CSAPP Lab6 · Shell Lab","tags":["CSAPP","LAB","实验","进程","信号","作业控制"]}}
---


# Shell Lab

实现一个带作业控制的 Unix shell（`tsh`），需要填 7 个函数。官方提供 16 个 trace 做回归测试。

| 函数 | 作用 |
| --- | --- |
| `eval` | 解析命令行，fork+execve 执行 |
| `builtin_cmd` | 识别 `quit`/`jobs`/`bg`/`fg`/`&` |
| `do_bgfg` | 实现 `bg` 和 `fg` |
| `waitfg` | 等待前台作业结束 |
| `sigchld_handler` | 回收子进程、记录停止/终止 |
| `sigint_handler` | Ctrl-C → 发给前台进程组 |
| `sigtstp_handler` | Ctrl-Z → 挂起前台进程组 |

## 测试方式

```bash
make                     # 编译 tsh、myspin、mysplit、mystop、myint
./sdriver.pl -t trace01.txt -s ./tsh -a "-p"
./sdriver.pl -t trace01.txt -s ./tshref -a "-p"    # 参考实现
```

`-a "-p"` 表示不打印提示符，便于自动比对。

**关键**：直接 `diff` 会失败，因为输出里带 PID，每次运行都不同。归一化后再比：

```bash
norm() { sed -E 's/\(([0-9]+)\)/(PID)/g' "$1"; }
diff <(norm out.txt) <(norm ref.txt)
```

**批量跑全部 16 个**

```bash
norm() { sed -E 's/\(([0-9]+)\)/(PID)/g' "$1"; }
for i in 01 02 03 04 05 06 07 08 09 10 11 12 13 14 15 16; do
  ./sdriver.pl -t trace$i.txt -s ./tsh -a "-p"   > /tmp/out$i.txt 2>&1
  ./sdriver.pl -t trace$i.txt -s ./tshref -a "-p" > /tmp/ref$i.txt 2>&1
  if diff -q <(norm /tmp/out$i.txt) <(norm /tmp/ref$i.txt) >/dev/null; then
    echo "trace$i PASS"
  else
    echo "trace$i FAIL"
  fi
done
```

```text
trace01 PASS
trace02 PASS
...
trace16 PASS
```

---

## 一、eval —— 执行命令的核心

```c
void eval(char *cmdline) 
{
    char *argv[MAXARGS];
    char buf[MAXLINE];
    int bg;
    pid_t pid;
    sigset_t mask_all, mask_one, prev_one;

    strcpy(buf, cmdline);
    bg = parseline(buf, argv);
    if (argv[0] == NULL)
        return;                       /* 空行 */

    if (!builtin_cmd(argv)) {
        sigfillset(&mask_all);
        sigemptyset(&mask_one);
        sigaddset(&mask_one, SIGCHLD);
        sigprocmask(SIG_BLOCK, &mask_one, &prev_one);   /* 先屏蔽 SIGCHLD */

        if ((pid = fork()) == 0) {    /* 子进程 */
            sigprocmask(SIG_SETMASK, &prev_one, NULL);
            setpgid(0, 0);            /* 放进新进程组 */
            if (execve(argv[0], argv, environ) < 0) {
                printf("%s: Command not found\n", argv[0]);
                exit(0);
            }
        }

        sigprocmask(SIG_BLOCK, &mask_all, NULL);
        addjob(jobs, pid, bg ? BG : FG, cmdline);
        sigprocmask(SIG_SETMASK, &prev_one, NULL);

        if (!bg)
            waitfg(pid);              /* 前台：等它结束 */
        else
            printf("[%d] (%d) %s", pid2jid(pid), pid, cmdline);
    }
    return;
}
```

**三个关键点**

**1. fork 前必须屏蔽 SIGCHLD**

这是一个经典竞态：

```text
父进程 fork → 子进程立刻结束 → SIGCHLD 送到 → handler 里 deletejob 找不到这个 pid（还没 addjob）
→ 父进程 addjob → 这个 job 永远不会被回收，waitfg 死等
```

所以要在 fork **之前**屏蔽 `SIGCHLD`，`addjob` 完成后再恢复。

**2. 子进程要 `setpgid(0, 0)`**

把子进程放进自己的进程组。这样 shell 收到 Ctrl-C 时不会连带自己被杀，而是 `kill(-pid, SIGINT)` 只发给那个进程组。

**3. `execve` 用 `environ`**

不是 `envp`，是全局变量 `environ`（声明在 `unistd.h`）。传 `NULL` 会导致子进程没有环境变量，某些程序会异常。

---

## 二、builtin_cmd —— 内置命令

```c
int builtin_cmd(char **argv) 
{
    if (!strcmp(argv[0], "quit"))
        exit(0);
    if (!strcmp(argv[0], "&"))
        return 1;                 /* 单个 & 忽略 */
    if (!strcmp(argv[0], "jobs")) {
        listjobs(jobs);
        return 1;
    }
    if (!strcmp(argv[0], "bg") || !strcmp(argv[0], "fg")) {
        do_bgfg(argv);
        return 1;
    }
    return 0;     /* 不是内置命令，交给 eval 去 fork */
}
```

返回值语义：**1 = 已处理（不要再 fork），0 = 需要 fork 执行**。

---

## 三、do_bgfg —— fg / bg 命令

```c
void do_bgfg(char **argv) 
{
    struct job_t *job;
    int jid;
    pid_t pid;
    sigset_t mask_all, prev;

    if (argv[1] == NULL) {
        printf("%s command requires PID or %%jobid argument\n", argv[0]);
        return;
    }
    /* 校验参数格式：%<数字> 或 <数字> */
    {
        char *p1 = argv[1];
        int ok;
        if (*p1 == '%') { p1++; ok = (*p1 != '\0'); }
        else { ok = (*p1 != '\0'); }
        while (ok && *p1) {
            if (*p1 < '0' || *p1 > '9') ok = 0;
            p1++;
        }
        if (!ok) {
            printf("%s: argument must be a PID or %%jobid\n", argv[0]);
            return;
        }
    }

    sigfillset(&mask_all);
    sigprocmask(SIG_BLOCK, &mask_all, &prev);

    if (argv[1][0] == '%') {                       /* %jid 形式 */
        jid = atoi(&argv[1][1]);
        job = getjobjid(jobs, jid);
        if (job == NULL) {
            printf("%s: No such job\n", argv[1]);
            sigprocmask(SIG_SETMASK, &prev, NULL);
            return;
        }
    } else {                                       /* pid 形式 */
        pid = atoi(argv[1]);
        job = getjobpid(jobs, pid);
        if (job == NULL) {
            printf("(%d): No such process\n", pid);
            sigprocmask(SIG_SETMASK, &prev, NULL);
            return;
        }
    }

    if (!strcmp(argv[0], "fg")) {
        job->state = FG;
        kill(-job->pid, SIGCONT);        /* 唤醒整个进程组 */
        sigprocmask(SIG_SETMASK, &prev, NULL);
        waitfg(job->pid);
    } else {                             /* bg */
        job->state = BG;
        kill(-job->pid, SIGCONT);
        printf("[%d] (%d) %s", job->jid, job->pid, job->cmdline);
        sigprocmask(SIG_SETMASK, &prev, NULL);
    }
    return;
}
```

**踩坑记录：参数校验**

第一版直接 `atoi(argv[1])`，结果 `fg a` 被转成 `pid = 0`，走到"无此进程"分支，输出：

```text
(0): No such process
```

但参考实现输出的是：

```text
fg: argument must be a PID or %jobid
```

**`atoi("a")` 返回 0 且不报错**，所以必须自己检查字符。trace14 专门测这个。

---

## 四、waitfg —— 等前台作业结束

```c
void waitfg(pid_t pid)
{
    while (pid == fgpid(jobs))
        sleep(0);
    return;
}
```

轮询写法最简单可靠。`fgpid()` 返回当前前台作业的 pid，只要它还等于目标 pid 就一直等。

> 也可以用 `sigsuspend` 配合信号量做无忙等版本，但要注意 `SIGCHLD` 可能在检查之后、`sigsuspend` 之前到达，导致永久挂起。轮询 + `sleep(0)` 能避免这类竞态。

---

## 五、sigchld_handler —— 回收子进程

```c
void sigchld_handler(int sig) 
{
    int olderrno = errno;
    sigset_t mask_all, prev;
    pid_t pid;
    int status;

    sigfillset(&mask_all);
    while ((pid = waitpid(-1, &status, WNOHANG | WUNTRACED)) > 0) {
        sigprocmask(SIG_BLOCK, &mask_all, &prev);
        if (WIFSTOPPED(status)) {
            struct job_t *job = getjobpid(jobs, pid);
            if (job) job->state = ST;
            printf("Job [%d] (%d) stopped by signal %d\n",
                   pid2jid(pid), pid, WSTOPSIG(status));
        } else if (WIFSIGNALED(status)) {
            printf("Job [%d] (%d) terminated by signal %d\n",
                   pid2jid(pid), pid, WTERMSIG(status));
            deletejob(jobs, pid);
        } else {
            deletejob(jobs, pid);      /* 正常退出 */
        }
        sigprocmask(SIG_SETMASK, &prev, NULL);
    }
    errno = olderrno;
    return;
}
```

**三个易错点**

**1. 必须循环 `waitpid`**

一次 `SIGCHLD` 可能对应多个子进程退出，只回收一次会留下僵尸。

**2. `WNOHANG | WUNTRACED` 两个 flag 都要**

- `WNOHANG`：没有已退出的子进程就立即返回，不阻塞
- `WUNTRACED`：连"被信号停止"的子进程也要报告

少 `WUNTRACED` 就检测不到 Ctrl-Z。

**3. 保存并恢复 `errno`**

handler 可能在任何指令之间打断主流程，改动 `errno` 会让主流程的判断出错。

---

## 六、sigint_handler / sigtstp_handler

```c
void sigint_handler(int sig) 
{
    int olderrno = errno;
    sigset_t mask_all, prev;
    pid_t pid;

    sigfillset(&mask_all);
    sigprocmask(SIG_BLOCK, &mask_all, &prev);
    if ((pid = fgpid(jobs)) > 0)
        kill(-pid, SIGINT);            /* 负 pid = 整个进程组 */
    sigprocmask(SIG_SETMASK, &prev, NULL);
    errno = olderrno;
    return;
}

void sigtstp_handler(int sig) 
{
    int olderrno = errno;
    sigset_t mask_all, prev;
    pid_t pid;

    sigfillset(&mask_all);
    sigprocmask(SIG_BLOCK, &mask_all, &prev);
    if ((pid = fgpid(jobs)) > 0)
        kill(-pid, SIGTSTP);
    sigprocmask(SIG_SETMASK, &prev, NULL);
    errno = olderrno;
    return;
}
```

**关键：`kill(-pid, ...)` 用负的 pid**

负值表示"发给整个进程组"。因为 `eval` 里做了 `setpgid(0, 0)`，子进程自成一个组，这样才能一次性把 `./mysplit` 这类会 fork 孙进程的程序全部挂起。

如果写成 `kill(pid, SIGINT)`，只杀进程本身，孙进程会变成孤儿继续跑。

---

## 七、16 个 trace 覆盖什么

| trace | 测试内容 |
| --- | --- |
| 01 | 空行、EOF |
| 02 | 内置命令 `quit` |
| 03 | 单个 `&` |
| 04 | 后台作业 |
| 05 | 前台作业 |
| 06 | 后台 + `jobs` |
| 07 | 前台 + `jobs` |
| 08 | Ctrl-C 前台作业 |
| 09 | Ctrl-C 后台作业（应无效） |
| 10 | Ctrl-Z 前台作业 |
| 11 | Ctrl-Z 后台（应无效） |
| 12 | `jobs` 显示停止状态 |
| 13 | `fg` 唤醒停止的作业 |
| 14 | **错误处理**（非法参数） |
| 15 | `bg` 让停止的作业继续 |
| 16 | 综合场景 |

全部通过：

```text
trace01 PASS ~ trace16 PASS
16/16
```

---

## 常见问题

**`./tsh: Command not found` 但命令是对的**

`execve` 不会搜索 `PATH`。要用 `execvp`，或者自己拼绝对路径。官方 trace 都用 `./xxx` 相对路径，所以 `execve` 够用。

**Ctrl-C 后 shell 自己也退出了**

子进程没做 `setpgid(0, 0)`，和 shell 同组。Ctrl-C 发给整个前台进程组时把 shell 也带走了。

**`waitfg` 死循环**

`SIGCHLD` 在 `fork` 后、`addjob` 前到达，handler 删不掉 job。必须在 fork 前屏蔽 `SIGCHLD`。

**`jobs` 里残留已结束的作业**

`sigchld_handler` 里 `waitpid` 没循环，或者用了 `WNOHANG` 却只调用一次。

**`fg %1` 报 `No such job` 但 `jobs` 明明有**

`atoi(&argv[1][1])` 的偏移写错了。`%1` 取 `argv[1][1]` 才是 `"1"`。

**测试全部 FAIL 但输出肉眼看着一样**

PID 不同。用 `sed` 归一化 `(12345)` → `(PID)` 再 diff。

---

## 附录：环境与踩坑

### 直接 `diff` 必然失败 —— PID 每次都不同

```bash
./sdriver.pl -t trace04.txt -s ./tsh   -a "-p" > out.txt 2>&1
./sdriver.pl -t trace04.txt -s ./tshref -a "-p" > ref.txt 2>&1
diff out.txt ref.txt
```

```text
6c6
< [1] (580523) ./myspin 1 &
---
> [1] (580528) ./myspin 1 &
```

内容完全一样，只有 PID 不同。**必须归一化后再比**：

```bash
norm() { sed -E 's/\(([0-9]+)\)/(PID)/g' "$1"; }
diff <(norm out.txt) <(norm ref.txt)
```

如果不做这一步，第一批测试会全报 FAIL，白折腾。

### `sdriver.pl` 的 `-a "-p"` 别漏

```bash
./sdriver.pl -t trace01.txt -s ./tsh -a "-p"
```

`-a` 把参数传给 tsh，`-p` 让它不打印 `tsh> ` 提示符。不加的话输出里混着提示符，比对时更麻烦。

### 必须先 `make` 出所有测试程序

```bash
make
```

产出四个辅助程序：

```text
tsh        shell 本体
myspin     睡眠 N 秒（测试前后台）
mysplit    fork 出孙进程（测试进程组信号）
mystop     自己给自己发 SIGTSTP
myint      自己给自己发 SIGINT
```

缺任何一个，对应 trace 会报"command not found"，但错误信息藏在输出里不容易注意到。

### `fg a` 报错信息对不上

第一版 `do_bgfg` 直接 `atoi`：

```c
pid = atoi(argv[1]);          /* "a" → 0 */
job = getjobpid(jobs, pid);
if (job == NULL) {
    printf("(%d): No such process\n", pid);
```

`atoi("a")` 返回 0 且**不报错**，于是走到"无此进程"分支：

```text
(0): No such process
```

参考实现输出的是：

```text
fg: argument must be a PID or %jobid
```

**必须自己校验字符**：

```c
{
    char *p1 = argv[1];
    int ok;
    if (*p1 == '%') { p1++; ok = (*p1 != '\0'); }
    else { ok = (*p1 != '\0'); }
    while (ok && *p1) {
        if (*p1 < '0' || *p1 > '9') ok = 0;
        p1++;
    }
    if (!ok) {
        printf("%s: argument must be a PID or %%jobid\n", argv[0]);
        return;
    }
}
```

trace14 专门测这个，其他 15 个 trace 都测不出来。

### `%` 在 `printf` 里要写成 `%%`

```c
printf("%s command requires PID or %%jobid argument\n", argv[0]);
```

漏写成单个 `%`，`printf` 会把 `%j` 当格式符，输出乱码或崩溃。

### 子进程必须 `setpgid(0, 0)`

不加的话子进程和 shell 在同一个进程组：

```text
用户按 Ctrl-C
  → 终端把 SIGINT 发给前台进程组（含 shell 自己）
  → shell 也被杀了
```

加了之后子进程自成一个组，shell 里的 handler 用 `kill(-pid, SIGINT)` 只发给那个组：

```c
if ((pid = fork()) == 0) {
    sigprocmask(SIG_SETMASK, &prev_one, NULL);
    setpgid(0, 0);            /* ← 关键 */
    execve(argv[0], argv, environ);
```

### `kill` 必须用负 pid

```c
kill(-pid, SIGINT);      /* 正确：整个进程组 */
kill(pid, SIGINT);       /* 错误：只杀进程本身 */
```

区别在 `mysplit` 这类会 `fork` 孙进程的程序上特别明显 —— 用正 pid 只杀了父进程，孙进程变孤儿继续跑，`jobs` 里状态永远不更新。

### `SIGCHLD` 竞态：fork 前就要屏蔽

这是最隐蔽的一个 bug。错误顺序：

```text
fork 成功
  → 子进程立刻结束
  → 内核送 SIGCHLD
  → handler 执行 waitpid，回收了
  → handler 调 deletejob(pid)，但 jobs 表里还没有这个 pid（addjob 还没执行）
  → 父进程执行 addjob，加入一个已经死掉的 job
  → waitfg 死等，因为 fgpid() 一直返回这个 pid
```

**正确做法**：

```c
sigemptyset(&mask_one);
sigaddset(&mask_one, SIGCHLD);
sigprocmask(SIG_BLOCK, &mask_one, &prev_one);   /* fork 之前就屏蔽 */

if ((pid = fork()) == 0) {
    sigprocmask(SIG_SETMASK, &prev_one, NULL);   /* 子进程恢复 */
    ...
}

sigprocmask(SIG_BLOCK, &mask_all, NULL);
addjob(jobs, pid, bg ? BG : FG, cmdline);        /* 先加进表 */
sigprocmask(SIG_SETMASK, &prev_one, NULL);       /* 再放开，此时 handler 能找到 job */
```

### `waitpid` 必须循环 + 双 flag

```c
while ((pid = waitpid(-1, &status, WNOHANG | WUNTRACED)) > 0) {
```

三个要素缺一不可：

| 要素 | 缺了会怎样 |
| --- | --- |
| 循环 | 一次 SIGCHLD 对应多个子进程退出时，剩下的变僵尸 |
| `WNOHANG` | 没有已退出的子进程时阻塞住，shell 卡死 |
| `WUNTRACED` | 检测不到 Ctrl-Z 停止的子进程，`jobs` 不显示 stopped |

### 保存并恢复 `errno`

```c
void sigchld_handler(int sig) 
{
    int olderrno = errno;
    ...
    errno = olderrno;
}
```

信号 handler 可能在任何指令之间打断主流程。如果主流程正在检查 `errno`（比如 `read` 返回 -1 后判断错误类型），handler 改了 `errno` 就会误判。

### 16 个 trace 一次性验证脚本

```bash
cd /tmp/sl
norm() { sed -E 's/\(([0-9]+)\)/(PID)/g' "$1"; }
for i in 01 02 03 04 05 06 07 08 09 10 11 12 13 14 15 16; do
  timeout 60 ./sdriver.pl -t trace$i.txt -s ./tsh   -a "-p" > /tmp/out$i.txt 2>&1
  timeout 60 ./sdriver.pl -t trace$i.txt -s ./tshref -a "-p" > /tmp/ref$i.txt 2>&1
  if diff -q <(norm /tmp/out$i.txt) <(norm /tmp/ref$i.txt) >/dev/null 2>&1; then
    printf "trace%s PASS\n" $i
  else
    printf "trace%s FAIL\n" $i
  fi
done
```

```text
trace01 PASS
...
trace16 PASS
```

**每个 trace 加 `timeout`** —— 如果是 shell 挂死（而不是快速失败），不加 timeout 会一直卡着。

### 只测失败的那个

跑全量要几分钟。定位到某个 trace 后单独测：

```bash
./sdriver.pl -t trace14.txt -s ./tsh -a "-p" > /tmp/o.txt 2>&1
./sdriver.pl -t trace14.txt -s ./tshref -a "-p" > /tmp/r.txt 2>&1
diff /tmp/o.txt /tmp/r.txt
```

`diff` 不归一化，因为要看具体哪行不同。

### 这次实际改了两次

第一版 16 个里 15 过、trace14 挂（`fg a` 报错信息不同）。加了参数校验后 16/16 全过。

**如果只看前 13 个 trace 就收工，会漏掉错误处理这条线。**

### `tshref` 是官方参考实现

别删。它是逐字节比对的标准答案，也是唯一能告诉你"正确输出长什么样"的东西。

### 交互式手动测试

想自己玩一下：

```bash
./tsh
```
```text
tsh> ./myspin 3 &
[1] (12345) ./myspin 3 &
tsh> jobs
[1] (12345) Running ./myspin 3 &
tsh> ./myspin 10
^Z
Job [2] (12346) stopped by signal 20
tsh> jobs
[2] (12346) Stopped ./myspin 10
tsh> bg %2
[2] (12346) ./myspin 10
tsh> quit
```

对照这个行为检查：后台提示格式、`jobs` 输出格式、停止信号提示，都和参考实现一致才对。
