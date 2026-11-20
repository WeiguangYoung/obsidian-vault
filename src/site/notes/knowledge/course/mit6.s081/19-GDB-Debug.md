---
{"dg-publish":true,"permalink":"/knowledge/course/mit6.s081/19-GDB-Debug/","title":"6.S081 · GDB 调试","tags":["MIT6.1810","xv6","操作系统","GDB","调试"],"dg-note-properties":{"title":"6.S081 · GDB 调试","tags":["MIT6.1810","xv6","操作系统","GDB","调试"],"date":"2026-11-18"}}
---


# 6.S081 · GDB 调试

> 在 QEMU 上调试 xv6 内核：断点、单步、查寄存器与内存

返回 [[knowledge/course/mit6.s081/index\|📚 6.S081 目录]] · [[knowledge/course/index\|📚 课程资源]] · [[首页\|🏠 首页]]

---

## 一、启动调试

终端 A 启动带调试桩的 QEMU（它会停在等 GDB 连接的状态）：

```shell
make CPUS=1 qemu-gdb
```

终端 B 连接：

```shell
gdb-multiarch kernel/kernel
```

`CPUS=1` 把核数降到 1，避免多核并发打断点时的干扰。

---

## 二、单步

| 命令 | 简写 | 行为 |
|:--|:--:|:--|
| `step` | `s` | 执行一行源码，**会进入**被调用的函数 |
| `next` | `n` | 执行一行源码，**不进入**函数 |
| `stepi` | `si` | 执行一条机器指令，**会进入**函数 |
| `nexti` | `ni` | 执行一条机器指令，**不进入**函数 |

读内核汇编时用 `si` / `ni`；看 C 代码逻辑时用 `s` / `n`。

---

## 三、查看变量

`print`（简写 `p`）可打印变量或表达式：

```sh
(gdb) p $pc            # 打印 pc 寄存器
(gdb) p/x $satp        # 以十六进制打印 satp
(gdb) p num            # 打印局部变量 num
```

格式化前缀（紧跟在 `p` 后）：

| 格式 | 含义 |
|:--:|:--|
| `d` | 有符号十进制 |
| `u` | 无符号十进制 |
| `x` | 十六进制 |
| `o` | 八进制 |
| `t` | 二进制 |
| `f` | 浮点 |
| `c` | 字符 |

`p` 与格式符之间可插入数字，表示连续查看多个元素。

---

## 四、查看内存

命令 `x`（examine）按指定格式读取内存：

```sh
(gdb) x/2c $a1        # 以字符形式看 a1 指向的 2 个字符
```

格式为 `x/<数量><格式><单位>`，例如：

| 写法 | 含义 |
|:--|:--|
| `x/2c addr` | 读 2 个字符 |
| `x/4x addr` | 读 4 个十六进制字 |
| `x/8g addr` | 读 8 个 8 字节（giant）单元 |

---

## 五、查看指令

```sh
(gdb) x/3i address     # 反汇编 address 开始的 3 条指令
```

定位「`sepc` 指向的那条指令到底做了什么」时最常用。

---

## 六、断点

```sh
(gdb) b *0x80000000    # 在地址处下断点
(gdb) b main           # 在函数名下断点
(gdb) info break       # 查看所有断点
(gdb) delete 1         # 删除 1 号断点
(gdb) c                # 继续运行
```

内核启动调试的常规套路：在 `_entry`（`0x80000000`）下第一个断点，然后 `b main`、`b syscall`，逐段推进观察状态变化。

---

## 七、其他常用

| 命令 | 用途 |
|:--|:--|
| `layout split` | 分屏同时显示源码与汇编 |
| `info registers` | 一次打印全部寄存器 |
| `info mem` | QEMU 监视器命令，查看当前页表映射 |
| `help <cmd>` | 查某个命令的用法 |

> `info mem` 要在 QEMU 的 console 里用（`Ctrl-a c` 切换），不是在 GDB 里。

---

## 参考

- [GDB 官方手册](https://sourceware.org/gdb/onlinedocs/gdb/index.html)
- [100 个 GDB 小技巧](https://wizardforcel.gitbooks.io/100-gdb-tips/content/break-on-address.html)