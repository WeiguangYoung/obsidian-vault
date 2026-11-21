---
{"dg-publish":true,"permalink":"/knowledge/course/csapp-labs/02-Bomb-Lab/","title":"CSAPP Lab2 · Bomb Lab","tags":["CSAPP","LAB","实验","汇编","GDB","反汇编"],"dg-note-properties":{"title":"CSAPP Lab2 · Bomb Lab","tags":["CSAPP","LAB","实验","汇编","GDB","反汇编"]}}
---


# Bomb Lab

二进制炸弹共 6 个 phase，每关要求输入正确内容，输错立即打印 `BOOM!!!` 并退出。全部输入正确才会输出 `Congratulations! You've defused the bomb!`。

| # | 考点 | 答案 |
| ---: | --- | --- |
| 1 | 字符串比较 | `Border relations with Canada have never been better.` |
| 2 | 循环 + 等比数列 | `1 2 4 8 16 32` |
| 3 | `switch` 跳转表 | `0 207`（共 8 组可行） |
| 4 | 递归二分查找 | `7 0` |
| 5 | 查表 + 低位索引 | `ionefg` |
| 6 | 链表排序 | `4 3 2 1 6 5` |

完整输入，每行对应一关：

```text
Border relations with Canada have never been better.
1 2 4 8 16 32
0 207
7 0
ionefg
4 3 2 1 6 5
```

一键验证：

```bash
./bomb <<'EOF'
Border relations with Canada have never been better.
1 2 4 8 16 32
0 207
7 0
ionefg
4 3 2 1 6 5
EOF
```

输出：

```text
Welcome to my fiendish little bomb. You have 6 phases with
which to blow yourself up. Have a nice day!
Phase 1 defused. How about the next one?
That's number 2.  Keep going!
Halfway there!
So you got that one.  Try this one.
Good work!  On to the next...
Congratulations! You've defused the bomb!
```

输入错误时：

```text
BOOM!!!
The bomb has blown up.
```

---

## 1. 字符串比较

**反汇编**

```bash
objdump -d bomb --disassemble=phase_1
```

```asm
0000000000400ee0 <phase_1>:
  400ee0:	48 83 ec 08          	sub    $0x8,%rsp
  400ee4:	be 00 24 40 00       	mov    $0x402400,%esi     # 第二参数 = 0x402400
  400ee9:	e8 4a 04 00 00       	call   401338 <strings_not_equal>
  400eee:	85 c0                	test   %eax,%eax         # 返回 0 表示相等
  400ef0:	74 05                	je     400ef7 <phase_1+0x17>
  400ef2:	e8 43 05 00 00       	call   40143a <explode_bomb>  # 不等则爆炸
  400ef7:	48 83 c4 08          	add    $0x8,%rsp
  400efb:	c3                   	ret
```

**分析**

整个函数只有三步：把常量地址 `0x402400` 放进 `%esi`（第二参数），调用 `strings_not_equal`，返回非 0 就引爆。

`strings_not_equal` 的签名等价于 `int strings_not_equal(char *s1, char *s2)`，其中 `%rdi` 是用户输入（`phase_1` 的入参原样保留），`%esi` 是目标串地址。所以只要拿到 `0x402400` 处的字符串即可。

**取字符串**

```bash
gdb -batch -q ./bomb -ex 'x/s 0x402400'
```

```text
0x402400:	"Border relations with Canada have never been better."
```

也可以用 `objdump -s -j .rodata bomb` 翻只读数据段，从 `0x402400` 开始读。

**答案**

```text
Border relations with Canada have never been better.
```

**验证**

```bash
printf 'Border relations with Canada have never been better.\n' | ./bomb
```

```text
Phase 1 defused. How about the next one?
```

---

## 2. 循环 + 等比数列

**反汇编**

```bash
objdump -d bomb --disassemble=phase_2
```

```asm
0000000000400efc <phase_2>:
  400efc:	55                   	push   %rbp
  400efd:	53                   	push   %rbx
  400efe:	48 83 ec 28          	sub    $0x28,%rsp
  400f02:	48 89 e6             	mov    %rsp,%rsi              # 第二参数 = 栈顶，存 6 个数
  400f05:	e8 52 05 00 00       	call   40145c <read_six_numbers>
  400f0a:	83 3c 24 01          	cmpl   $0x1,(%rsp)            # a[0] 必须等于 1
  400f0e:	74 20                	je     400f30 <phase_2+0x34>
  400f10:	e8 25 05 00 00       	call   40143a <explode_bomb>
  400f15:	eb 19                	jmp    400f30 <phase_2+0x34>
  400f17:	8b 43 fc             	mov    -0x4(%rbx),%eax        # eax = a[i-1]
  400f1a:	01 c0                	add    %eax,%eax             # eax = 2 * a[i-1]
  400f1c:	39 03                	cmp    %eax,(%rbx)            # a[i] == 2*a[i-1] ?
  400f1e:	74 05                	je     400f25 <phase_2+0x29>
  400f20:	e8 15 05 00 00       	call   40143a <explode_bomb>
  400f25:	48 83 c3 04          	add    $0x4,%rbx             # 下一个元素
  400f29:	48 39 eb             	cmp    %rbp,%rbx              # 走到 a[6] 就结束
  400f2c:	75 e9                	jne    400f17 <phase_2+0x1b>
  400f2e:	eb 0c                	jmp    400f3c <phase_2+0x40>
  400f30:	48 8d 5c 24 04       	lea    0x4(%rsp),%rbx         # rbx = &a[1]
  400f35:	48 8d 6c 24 18       	lea    0x18(%rsp),%rbp        # rbp = &a[6]
  400f3a:	eb db                	jmp    400f17 <phase_2+0x1b>
  400f3c:	48 83 c4 28          	add    $0x28,%rsp
  400f40:	5b                   	pop    %rbx
  400f41:	5d                   	pop    %rbp
  400f42:	c3                   	ret
```

**分析**

`read_six_numbers` 会按 `%d %d %d %d %d %d` 读 6 个整数，依次放进 `%rsp` 开头的 6 个 `int`（每个 4 字节）里。

关键只有两处：

1. `cmpl $0x1,(%rsp)` —— `a[0]` 必须是 `1`
2. 循环体 `add %eax,%eax` + `cmp %eax,(%rbx)` —— `a[i]` 必须等于 `2 * a[i-1]`

循环从 `a[1]` 走到 `a[5]`（`%rbx` 从 `&a[1]` 递增到 `&a[6]` 结束）。

所以这是一个首项为 1、公比为 2 的等比数列。

**动态验证**

在第 2 次比较处下断点，观察每轮的 `%eax` 与 `a[i]`：

```bash
gdb -batch -q ./bomb \
  -ex 'b *0x400f1c' \
  -ex 'run < answers.txt' \
  -ex 'p $eax' -ex 'p *(int*)$rbx' -ex 'c' \
  -ex 'p $eax' -ex 'p *(int*)$rbx'
```

```text
Breakpoint 1, 0x0000000000400f1c in phase_2 ()
$1 = 2                          # 2 * a[0] = 2*1
$2 = 2                          # a[1] = 2，相等
Breakpoint 1, 0x0000000000400f1c in phase_2 ()
$3 = 4                          # 2 * a[1] = 2*2
$4 = 4                          # a[2] = 4，相等
```

每轮 `%eax` 是"前一元素的两倍"，`%rbx` 指向的当前元素必须与它相等。

**答案**

```text
1 2 4 8 16 32
```

**验证**

```bash
printf 'Border relations with Canada have never been better.\n1 2 4 8 16 32\n' | ./bomb
```

```text
Phase 1 defused. How about the next one?
That's number 2.  Keep going!
```

---

## 3. `switch` 跳转表

**反汇编**

```bash
objdump -d bomb --disassemble=phase_3
```

```asm
0000000000400f43 <phase_3>:
  400f43:	48 83 ec 18          	sub    $0x18,%rsp
  400f47:	48 8d 4c 24 0c       	lea    0xc(%rsp),%rcx         # 第三参数 = &num2
  400f4c:	48 8d 54 24 08       	lea    0x8(%rsp),%rdx         # 第二参数 = &num1
  400f51:	be cf 25 40 00       	mov    $0x4025cf,%esi         # 格式串 "%d %d"
  400f56:	b8 00 00 00 00       	mov    $0x0,%eax
  400f5b:	e8 90 fc ff ff       	call   400bf0 <__isoc99_sscanf@plt>
  400f60:	83 f8 01             	cmp    $0x1,%eax              # 必须读到至少 2 个（>1）
  400f63:	7f 05                	jg     400f6a <phase_3+0x27>
  400f65:	e8 d0 04 00 00       	call   40143a <explode_bomb>
  400f6a:	83 7c 24 08 07       	cmpl   $0x7,0x8(%rsp)         # num1 <= 7，否则跳去爆炸
  400f6f:	77 3c                	ja     400fad <phase_3+0x6a>
  400f71:	8b 44 24 08          	mov    0x8(%rsp),%eax         # eax = num1
  400f75:	ff 24 c5 70 24 40 00 	jmp    *0x402470(,%rax,8)    # 跳转表：0x402470 + 8*num1
  400f7c:	b8 cf 00 00 00       	mov    $0xcf,%eax             # case 0 -> 207
  400f81:	eb 3b                	jmp    400fbe <phase_3+0x7b>
  400f83:	b8 c3 02 00 00       	mov    $0x2c3,%eax            # -> 707
  400f88:	eb 34                	jmp    400fbe <phase_3+0x7b>
  400f8a:	b8 00 01 00 00       	mov    $0x100,%eax            # -> 256
  400f8f:	eb 2d                	jmp    400fbe <phase_3+0x7b>
  400f91:	b8 85 01 00 00       	mov    $0x185,%eax            # -> 389
  400f96:	eb 26                	jmp    400fbe <phase_3+0x7b>
  400f98:	b8 ce 00 00 00       	mov    $0xce,%eax             # -> 206
  400f9d:	eb 1f                	jmp    400fbe <phase_3+0x7b>
  400f9f:	b8 aa 02 00 00       	mov    $0x2aa,%eax            # -> 682
  400fa4:	eb 18                	jmp    400fbe <phase_3+0x7b>
  400fa6:	b8 47 01 00 00       	mov    $0x147,%eax            # -> 327
  400fab:	eb 11                	jmp    400fbe <phase_3+0x7b>
  400fad:	e8 88 04 00 00       	call   40143a <explode_bomb>
  400fb2:	b8 00 00 00 00       	mov    $0x0,%eax
  400fb7:	eb 05                	jmp    400fbe <phase_3+0x7b>
  400fb9:	b8 37 01 00 00       	mov    $0x137,%eax            # case 1 -> 311
  400fbe:	3b 44 24 0c          	cmp    0xc(%rsp),%eax         # num2 == eax ?
  400fc2:	74 05                	je     400fc9 <phase_3+0x86>
  400fc4:	e8 71 04 00 00       	call   40143a <explode_bomb>
  400fc9:	48 83 c4 18          	add    $0x18,%rsp
  400fcd:	c3                   	ret
```

**分析**

格式串取自 `0x4025cf`：

```bash
gdb -batch -q ./bomb -ex 'x/s 0x4025cf'
```

```text
0x4025cf:	"%d %d"
```

所以输入是两个十进制整数。

流程：

1. `sscanf` 读两个数，返回值必须 `> 1`（即两个都读到了）
2. `num1` 必须 `<= 7`（`ja` 是无符号比较，负数会被当成巨大值而失败，实际上等价于 `0 <= num1 <= 7`）
3. `jmp *0x402470(,%rax,8)` —— 以 `num1` 为下标，从跳转表取地址

**取跳转表**

```bash
gdb -batch -q ./bomb -ex 'x/8a 0x402470'
```

```text
0x402470:	0x400f7c <phase_3+57>	0x400fb9 <phase_3+118>
0x402480:	0x400f83 <phase_3+64>	0x400f8a <phase_3+71>
0x402490:	0x400f91 <phase_3+78>	0x400f98 <phase_3+85>
0x4024a0:	0x400f9f <phase_3+92>	0x400fa6 <phase_3+99>
```

对应关系与每格设定的值：

| num1 | 目标地址 | 设定 `%eax` | `num2` 应为 |
| ---: | --- | --- | ---: |
| 0 | `0x400f7c` | `0xcf` | **207** |
| 1 | `0x400fb9` | `0x137` | **311** |
| 2 | `0x400f83` | `0x2c3` | **707** |
| 3 | `0x400f8a` | `0x100` | **256** |
| 4 | `0x400f91` | `0x185` | **389** |
| 5 | `0x400f98` | `0xce` | **206** |
| 6 | `0x400f9f` | `0x2aa` | **682** |
| 7 | `0x400fa6` | `0x147` | **327** |

注意表里第 1 项指向 `0x400fb9`（在函数末尾附近），不是顺序排列的——编译器把 case 1 的代码块放到了最后。用 `x/8a` 按顺序读表就能看出真实映射，不要按地址顺序猜。

**八组全部实测通过**

```bash
for i in 0 1 2 3 4 5 6 7; do ... done
```

```text
n=0 v=207 -> 通过
n=1 v=311 -> 通过
n=2 v=707 -> 通过
n=3 v=256 -> 通过
n=4 v=389 -> 通过
n=5 v=206 -> 通过
n=6 v=682 -> 通过
n=7 v=327 -> 通过
```

**答案**（八组任选）

```text
0 207
```

**验证**

```bash
printf 'Border relations with Canada have never been better.\n1 2 4 8 16 32\n0 207\n' | ./bomb
```

```text
Halfway there!
```

---

## 4. 递归二分查找

**反汇编**

```bash
objdump -d bomb --disassemble=phase_4
objdump -d bomb --disassemble=func4
```

```asm
000000000040100c <phase_4>:
  40100c:	48 83 ec 18          	sub    $0x18,%rsp
  401010:	48 8d 4c 24 0c       	lea    0xc(%rsp),%rcx         # &num2
  401015:	48 8d 54 24 08       	lea    0x8(%rsp),%rdx         # &num1
  40101a:	be cf 25 40 00       	mov    $0x4025cf,%esi         # "%d %d"
  40101f:	b8 00 00 00 00       	mov    $0x0,%eax
  401024:	e8 c7 fb ff ff       	call   400bf0 <__isoc99_sscanf@plt>
  401029:	83 f8 02             	cmp    $0x2,%eax              # 必须正好读到 2 个
  40102c:	75 07                	jne    401035 <phase_4+0x29>
  40102e:	83 7c 24 08 0e       	cmpl   $0xe,0x8(%rsp)         # num1 <= 14
  401033:	76 05                	jbe    40103a <phase_4+0x2e>
  401035:	e8 00 04 00 00       	call   40143a <explode_bomb>
  40103a:	ba 0e 00 00 00       	mov    $0xe,%edx              # 第三参数 hi = 14
  40103f:	be 00 00 00 00       	mov    $0x0,%esi              # 第二参数 lo = 0
  401044:	8b 7c 24 08          	mov    0x8(%rsp),%edi         # 第一参数 x = num1
  401048:	e8 81 ff ff ff       	call   400fce <func4>
  40104d:	85 c0                	test   %eax,%eax             # 返回值必须为 0
  40104f:	75 07                	jne    401058 <phase_4+0x4c>
  401051:	83 7c 24 0c 00       	cmpl   $0x0,0xc(%rsp)         # num2 必须为 0
  401056:	74 05                	je     40105d <phase_4+0x51>
  401058:	e8 dd 03 00 00       	call   40143a <explode_bomb>
  40105d:	48 83 c4 18          	add    $0x18,%rsp
  401061:	c3                   	ret
```

```asm
0000000000400fce <func4>:
  400fce:	48 83 ec 08          	sub    $0x8,%rsp
  400fd2:	89 d0                	mov    %edx,%eax              # eax = hi
  400fd4:	29 f0                	sub    %esi,%eax              # eax = hi - lo
  400fd6:	89 c1                	mov    %eax,%ecx
  400fd8:	c1 e9 1f             	shr    $0x1f,%ecx             # ecx = 符号位（负数才为 1）
  400fdb:	01 c8                	add    %ecx,%eax              # 处理负数除法偏置
  400fdd:	d1 f8                	sar    %eax                   # eax = (hi-lo)/2  向下取整
  400fdf:	8d 0c 30             	lea    (%rax,%rsi,1),%ecx     # ecx = mid = lo + (hi-lo)/2
  400fe2:	39 f9                	cmp    %edi,%ecx              # mid 与 x 比较
  400fe4:	7e 0c                	jle    400ff2 <func4+0x24>    # mid <= x 跳走
  400fe6:	8d 51 ff             	lea    -0x1(%rcx),%edx        # hi = mid - 1
  400fe9:	e8 e0 ff ff ff       	call   400fce <func4>         # 递归左半
  400fee:	01 c0                	add    %eax,%eax              # 返回值 ×2
  400ff0:	eb 15                	jmp    401007 <func4+0x39>
  400ff2:	b8 00 00 00 00       	mov    $0x0,%eax              # eax = 0
  400ff7:	39 f9                	cmp    %edi,%ecx              # mid 与 x 再比较
  400ff9:	7d 0c                	jge    401007 <func4+0x39>    # mid >= x 即相等，直接返回 0
  400ffb:	8d 71 01             	lea    0x1(%rcx),%esi         # lo = mid + 1
  400ffe:	e8 cb ff ff ff       	call   400fce <func4>         # 递归右半
  401003:	8d 44 00 01          	lea    0x1(%rax,%rax,1),%eax  # 返回值 ×2 + 1
  401007:	48 83 c4 08          	add    $0x8,%rsp
  40100b:	c3                   	ret
```

**分析：`func4` 的逻辑**

把汇编翻译成 C：

```c
int func4(int x, int lo, int hi)
{
    int mid = lo + (hi - lo) / 2;      /* sar 对非负数即向下取整 */
    if (mid > x)
        return 2 * func4(x, lo, mid - 1);
    if (mid < x)
        return 2 * func4(x, mid + 1, hi) + 1;
    return 0;                           /* mid == x */
}
```

第三个分支的 `return 0` 很关键：`mov $0x0,%eax` 之后 `cmp %edi,%ecx` 满足 `jge` 就直接跳到 `ret`，不再改 `%eax`。

所以这是一个**二分查找**，找到时返回 0；往左递归返回值翻倍，往右递归返回值翻倍再加 1。

**分析：整个关卡的约束**

1. 必须正好输入 2 个整数
2. `num1 <= 14`（`jbe` 是无符号比较，负数也过不了）
3. `func4(num1, 0, 14)` 必须返回 `0`
4. `num2` 必须为 `0`

要让 `func4` 返回 0，只有一条路：**第一次就命中**，不进入任何递归分支。此时 `lo=0, hi=14`，`mid = 0 + (14-0)/2 = 7`。所以 `num1 = 7`。

**动态验证首次调用的参数**

```bash
gdb -batch -q ./bomb \
  -ex 'b func4' \
  -ex 'run < answers.txt' \
  -ex 'info registers rdi rsi rdx'
```

```text
Breakpoint 1, 0x0000000000400fce in func4 ()
rdi            0x7                 7
rsi            0x0                 0
rdx            0xe                 14
```

三个参数确认是 `(7, 0, 14)`，`mid = 7` 恰好等于 `x`，直接返回 0。

**若 `num1` 不等于 7 会怎样**

比如 `num1 = 6`：`mid=7 > 6`，走左分支 `func4(6, 0, 6)`，返回 `2 * (...) `，不可能是 0。所以只有 `x = 7` 能让返回值直接为 0。

**答案**

```text
7 0
```

**验证**

```bash
printf 'Border relations with Canada have never been better.\n1 2 4 8 16 32\n0 207\n7 0\n' | ./bomb
```

```text
Halfway there!
So you got that one.  Try this one.
```

---

## 5. 查表 + 低位索引

**反汇编**

```bash
objdump -d bomb --disassemble=phase_5
```

```asm
0000000000401062 <phase_5>:
  401062:	53                   	push   %rbx
  401063:	48 83 ec 20          	sub    $0x20,%rsp
  401067:	48 89 fb             	mov    %rdi,%rbx              # rbx = 输入字符串
  40106a:	64 48 8b 04 25 28 00 	mov    %fs:0x28,%rax          # 栈保护金丝雀
  401071:	00 00
  401073:	48 89 44 24 18       	mov    %rax,0x18(%rsp)
  401078:	31 c0                	xor    %eax,%eax
  40107a:	e8 9c 02 00 00       	call   40131b <string_length>
  40107f:	83 f8 06             	cmp    $0x6,%eax              # 长度必须为 6
  401082:	74 4e                	je     4010d2 <phase_5+0x70>
  401084:	e8 b1 03 00 00       	call   40143a <explode_bomb>
  401089:	eb 47                	jmp    4010d2 <phase_5+0x70>
  40108b:	0f b6 0c 03          	movzbl (%rbx,%rax,1),%ecx      # ecx = s[i]
  40108f:	88 0c 24             	mov    %cl,(%rsp)
  401092:	48 8b 14 24          	mov    (%rsp),%rdx
  401096:	83 e2 0f             	and    $0xf,%edx              # 取低 4 位
  401099:	0f b6 92 b0 24 40 00 	movzbl 0x4024b0(%rdx),%edx # 查表
  4010a0:	88 54 04 10          	mov    %dl,0x10(%rsp,%rax,1)  # 存到结果缓冲
  4010a4:	48 83 c0 01          	add    $0x1,%rax
  4010a8:	48 83 f8 06          	cmp    $0x6,%rax              # 循环 6 次
  4010ac:	75 dd                	jne    40108b <phase_5+0x29>
  4010ae:	c6 44 24 16 00       	movb   $0x0,0x16(%rsp)        # 结果串结尾补 '\0'
  4010b3:	be 5e 24 40 00       	mov    $0x40245e,%esi         # 目标串
  4010b8:	48 8d 7c 24 10       	lea    0x10(%rsp),%rdi        # 结果串
  4010bd:	e8 76 02 00 00       	call   401338 <strings_not_equal>
  4010c2:	85 c0                	test   %eax,%eax
  4010c4:	74 13                	je     4010d9 <phase_5+0x77>
  4010c6:	e8 6f 03 00 00       	call   40143a <explode_bomb>
  4010d2:	b8 00 00 00 00       	mov    $0x0,%eax
  4010d7:	eb b2                	jmp    40108b <phase_5+0x29>
  4010d9:	48 8b 44 24 18       	mov    0x18(%rsp),%rax
  4010de:	64 48 33 04 25 28 00 	xor    %fs:0x28,%rax          # 校验金丝雀
  4010e5:	00 00
  4010e7:	74 05                	je     4010ee <phase_5+0x8c>
  4010e9:	e8 42 fa ff ff       	call   400b30 <__stack_chk_fail@plt>
```

**分析**

流程：

1. 输入长度必须为 6
2. 逐个字符取 **低 4 位**（`and $0xf`）作为下标
3. 用下标查表 `0x4024b0`
4. 6 个查表结果拼成新字符串，必须等于 `0x40245e` 处的目标串

取表内容和目标串：

```bash
gdb -batch -q ./bomb -ex 'x/16cb 0x4024b0' -ex 'x/s 0x40245e'
```

```text
0x4024b0 <array.3449>:	109 'm'	97 'a'	100 'd'	117 'u'	105 'i'	101 'e'	114 'r'	115 's'
0x4024b8 <array.3449+8>:	110 'n'	102 'f'	111 'o'	116 't'	118 'v'	98 'b'	121 'y'	108 'l'
0x40245e:	"flyers"
```

表：`maduiersnfotvbyl`（16 个字符）
目标：`flyers`

**反查下标**

| 目标字符 | 表中下标 |
| --- | ---: |
| `f` | 9 |
| `l` | 15 |
| `y` | 14 |
| `e` | 5 |
| `r` | 6 |
| `s` | 7 |

所以输入的 6 个字符，其 **ASCII 低 4 位**必须依次是 `9, 15, 14, 5, 6, 7`。

**构造输入**

只要满足低 4 位即可，高位随意。取 ASCII 码 `0x60 + 目标下标` 附近的可打印字符最方便：

```text
f -> 低4位=0x9  -> chr(0x69) = 'i'
l -> 低4位=0xf  -> chr(0x6f) = 'o'
y -> 低4位=0xe  -> chr(0x6e) = 'n'
e -> 低4位=0x5  -> chr(0x65) = 'e'
r -> 低4位=0x6  -> chr(0x66) = 'f'
s -> 低4位=0x7  -> chr(0x67) = 'g'
```

拼起来就是 `ionefg`。

**答案有多种**

任何满足低 4 位为 `9,15,14,5,6,7` 的 6 字符都可，例如：

```text
ionefg
```

也可以用 `0x60` 以外的偏移，比如 `0x70 + idx`（`y`、`DEL` 之类不可打印，不推荐）。

**验证**

```bash
printf 'Border relations with Canada have never been better.\n1 2 4 8 16 32\n0 207\n7 0\nionefg\n' | ./bomb
```

```text
So you got that one.  Try this one.
```

---

## 6. 链表排序

**反汇编**

```bash
objdump -d bomb --disassemble=phase_6
```

```asm
00000000004010f4 <phase_6>:
  4010f4:	41 56                	push   %r14
  4010f6:	41 55                	push   %r13
  4010f8:	41 54                	push   %r12
  4010fa:	55                   	push   %rbp
  4010fb:	53                   	push   %rbx
  4010fc:	48 83 ec 50          	sub    $0x50,%rsp
  401100:	49 89 e5             	mov    %rsp,%r13
  401103:	48 89 e6             	mov    %rsp,%rsi
  401106:	e8 51 03 00 00       	call   40145c <read_six_numbers>
  40110b:	49 89 e6             	mov    %rsp,%r14              # r14 = a
  40110e:	41 bc 00 00 00 00    	mov    $0x0,%r12d             # 外层计数器 i = 0
  401114:	4c 89 ed             	mov    %r13,%rbp             # rbp = &a[i]
  401117:	41 8b 45 00          	mov    0x0(%r13),%eax
  40111b:	83 e8 01             	sub    $0x1,%eax
  40111e:	83 f8 05             	cmp    $0x5,%eax               # a[i]-1 <= 5  →  1 <= a[i] <= 6
  401121:	76 05                	jbe    401128 <phase_6+0x34>
  401123:	e8 12 03 00 00       	call   40143a <explode_bomb>
  401128:	41 83 c4 01          	add    $0x1,%r12d             # i++
  40112c:	41 83 fc 06          	cmp    $0x6,%r12d             # 满 6 个就跳出
  401130:	74 21                	je     401153 <phase_6+0x5f>
  401132:	44 89 e3             	mov    %r12d,%ebx             # j = i+1
  401135:	48 63 c3             	movslq %ebx,%rax
  401138:	8b 04 84             	mov    (%rsp,%rax,4),%eax     # eax = a[j]
  40113b:	39 45 00             	cmp    %eax,0x0(%rbp)         # a[i] == a[j] ?
  40113e:	75 05                	jne    401145 <phase_6+0x51>
  401140:	e8 f5 02 00 00       	call   40143a <explode_bomb>  # 相等则爆炸（要求互不相同）
  401145:	83 c3 01             	add    $0x1,%ebx
  401148:	83 fb 05             	cmp    $0x5,%ebx
  40114b:	7e e8                	jle    401135 <phase_6+0x41>
  40114d:	49 83 c5 04          	add    $0x4,%r13
  401151:	eb c1                	jmp    401114 <phase_6+0x20>
  401153:	48 8d 74 24 18       	lea    0x18(%rsp),%rsi        # rsi = &a[6]
  401158:	4c 89 f0             	mov    %r14,%rax              # rax = a
  40115b:	b9 07 00 00 00       	mov    $0x7,%ecx              # ecx = 7
  401160:	89 ca                	mov    %ecx,%edx
  401162:	2b 10                	sub    (%rax),%edx            # edx = 7 - a[i]
  401164:	89 10                	mov    %edx,(%rax)            # a[i] = 7 - a[i]
  401166:	48 83 c0 04          	add    $0x4,%rax
  40116a:	48 39 f0             	cmp    %rsi,%rax
  40116d:	75 f1                	jne    401160 <phase_6+0x6c>
  40116f:	be 00 00 00 00       	mov    $0x0,%esi              # 第二轮循环 i = 0
  401174:	eb 21                	jmp    401197 <phase_6+0xa3>
  401176:	48 8b 52 08          	mov    0x8(%rdx),%rdx         # 顺着 next 指针走
  40117a:	83 c0 01             	add    $0x1,%eax
  40117d:	39 c8                	cmp    %ecx,%eax
  40117f:	75 f5                	jne    401176 <phase_6+0x82>
  401181:	eb 05                	jmp    401188 <phase_6+0x94>
  401183:	ba d0 32 60 00       	mov    $0x6032d0,%edx         # 链表头 node1
  401188:	48 89 54 74 20       	mov    %rdx,0x20(%rsp,%rsi,2) # 存节点指针到栈上
  40118d:	48 83 c6 04          	add    $0x4,%rsi
  401191:	48 83 fe 18          	cmp    $0x18,%rsi
  401195:	74 14                	je     4011ab <phase_6+0xb7>
  401197:	8b 0c 34             	mov    (%rsp,%rsi,1),%ecx     # ecx = a[i]（已换成 7-x）
  40119a:	83 f9 01             	cmp    $0x1,%ecx
  40119d:	7e e4                	jle    401183 <phase_6+0x8f>  # <= 1 直接取头节点
  40119f:	b8 01 00 00 00       	mov    $0x1,%eax
  4011a4:	ba d0 32 60 00       	mov    $0x6032d0,%edx
  4011a9:	eb cb                	jmp    401176 <phase_6+0x82>
  4011ab:	48 8b 5c 24 20       	mov    0x20(%rsp),%rbx        # 重建链表
  4011b0:	48 8d 44 24 28       	lea    0x28(%rsp),%rax
  4011b5:	48 8d 74 24 50       	lea    0x50(%rsp),%rsi
  4011ba:	48 89 d9             	mov    %rbx,%rcx
  4011bd:	48 8b 10             	mov    (%rax),%rdx
  4011c0:	48 89 51 08          	mov    %rdx,0x8(%rcx)         # 前一个->next = 当前
  4011c4:	48 83 c0 08          	add    $0x8,%rax
  4011c8:	48 39 f0             	cmp    %rsi,%rax
  4011cb:	74 05                	je     4011d2 <phase_6+0xde>
  4011cd:	48 89 d1             	mov    %rdx,%rcx
  4011d0:	eb eb                	jmp    4011bd <phase_6+0xc9>
  4011d2:	48 c7 42 08 00 00 00 	movq   $0x0,0x8(%rdx)         # 最后节点 next = NULL
  4011d9:	00
  4011da:	bd 05 00 00 00       	mov    $0x5,%ebp              # 比较 5 次
  4011df:	48 8b 43 08          	mov    0x8(%rbx),%rax         # rax = cur->next
  4011e3:	8b 00                	mov    (%rax),%eax           # eax = next->value
  4011e5:	39 03                	cmp    %eax,(%rbx)            # cur->value >= next->value ?
  4011e7:	7d 05                	jge    4011ee <phase_6+0xfa>
  4011e9:	e8 4c 03 00 00       	call   40143a <explode_bomb>  # 小于则爆炸（要求降序）
  4011ee:	48 8b 5b 08          	mov    0x8(%rbx),%rbx
  4011f2:	83 ed 01             	sub    $0x1,%ebp
  4011f5:	75 e8                	jne    4011df <phase_6+0xeb>
```

**分析**

关卡分三段：

**第一段：校验 6 个数各为 1~6 且互不相同**

- `sub $0x1,%eax` 后 `cmp $0x5` + `jbe` —— 无符号比较，等价于 `1 <= a[i] <= 6`
- 内层双重循环两两比较 `a[i] == a[j]`，相等就引爆 —— 要求 6 个数互不重复

既然 6 个互不相同的数都在 1~6 之间，那必然是 `1,2,3,4,5,6` 的一个排列。

**第二段：`a[i] = 7 - a[i]`**

```asm
mov $0x7,%ecx
sub (%rax),%edx     # edx = 7 - a[i]
mov %edx,(%rax)     # 写回
```

**第三段：按值取链表节点并检查降序**

- 用 `7 - a[i]` 作为序号，从链表头 `0x6032d0` 出发走 `n-1` 步取到第 n 个节点
- 把 6 个节点指针依次存到栈上，再重新串成链表
- 最后逐对比较 `cur->value >= next->value`，**必须严格满足降序**（`jge` 才不爆，相等也过但值本身都不同）

**取链表节点值**

```bash
gdb -batch -q ./bomb -ex 'x/12gx 0x6032d0'
```

```text
0x6032d0 <node1>:	0x000000010000014c	0x00000000006032e0
0x6032e0 <node2>:	0x00000002000000a8	0x00000000006032f0
0x6032f0 <node3>:	0x000000030000039c	0x0000000000603300
0x603300 <node4>:	0x00000004000002b3	0x0000000000603310
0x603310 <node5>:	0x00000005000001dd	0x0000000000603320
0x603320 <node6>:	0x00000006000001bb	0x0000000000000000
```

每行是 16 字节：前 8 字节打包了 `value`（低 4 字节）和 `index`（高 4 字节），后 8 字节是 `next` 指针。

拆开看：

| 节点 | 地址 | value（十进制） | index | next |
| ---: | --- | ---: | ---: | --- |
| 1 | `0x6032d0` | **332** | 1 | `0x6032e0` |
| 2 | `0x6032e0` | **168** | 2 | `0x6032f0` |
| 3 | `0x6032f0` | **924** | 3 | `0x603300` |
| 4 | `0x603300` | **691** | 4 | `0x603310` |
| 5 | `0x603310` | **477** | 5 | `0x603320` |
| 6 | `0x603320` | **443** | 6 | `NULL` |

**求降序排列的节点序号**

按 `value` 从大到小排：

```text
924 (node3) > 691 (node4) > 477 (node5) > 443 (node6) > 332 (node1) > 168 (node2)
```

节点序号序列：`3, 4, 5, 6, 1, 2`

**反推输入**

程序内部的序号是 `7 - 输入值`，所以：

```text
输入值 = 7 - 序号
      = 7-3, 7-4, 7-5, 7-6, 7-1, 7-2
      = 4,   3,   2,   1,   6,   5
```

**动态验证**

在降序比较处下断点，查看 `7-x` 之后栈上的 6 个序号：

```bash
gdb -batch -q ./bomb \
  -ex 'b *0x4011df' \
  -ex 'run < answers.txt' \
  -ex 'x/6wx $rsp'
```

```text
Breakpoint 1, 0x00000000004011df in phase_6 ()
0x7fffffffe5e0:	0x00000003	0x00000004	0x00000005	0x00000006
0x7fffffffe5f0:	0x00000001	0x00000002
```

栈上正是 `3, 4, 5, 6, 1, 2` —— 与按值降序的节点序号完全一致，说明输入 `4 3 2 1 6 5` 是对的。

**答案**

```text
4 3 2 1 6 5
```

**验证**

```bash
printf 'Border relations with Canada have never been better.\n1 2 4 8 16 32\n0 207\n7 0\nionefg\n4 3 2 1 6 5\n' | ./bomb
```

```text
Good work!  On to the next...
Congratulations! You've defused the bomb!
```

---

## 常用命令

**反汇编**

```bash
objdump -d bomb                              # 全部反汇编
objdump -d bomb --disassemble=phase_1        # 只看某一个函数
objdump -d bomb --disassemble=func4          # 看辅助函数
```

**读数据段**

```bash
objdump -s -j .rodata bomb                   # 只读数据：字符串、跳转表、查找表
objdump -s -j .data bomb                     # 可写数据：链表节点
```

**gdb 交互式**

```bash
gdb bomb
```

```gdb
break phase_1                # 在关卡入口下断点
break explode_bomb           # 抓住爆炸发生的瞬间，看是哪个条件不满足
run < answers.txt            # 带输入运行

x/s 0x402400                 # 按字符串读内存
x/16cb 0x4024b0              # 按字符读 16 字节
x/8a 0x402470                # 按地址读 8 项（跳转表）
x/6gx 0x6032d0               # 按 8 字节十六进制读（链表节点）
x/6wx $rsp                   # 按 4 字节读栈（数组）

info registers               # 全部寄存器
info registers rdi rsi rdx   # 只看指定几个
print $eax                   # 打印寄存器值
print *(int*)$rbx            # 按 int 解引用指针

stepi                        # 单条汇编指令步进
nexti                        # 单条指令步进（不进入 call）
continue                     # 继续到下一个断点
```

**非交互式（批量）**

```bash
gdb -batch -q ./bomb \
  -ex 'b *0x400f1c' \
  -ex 'run < answers.txt' \
  -ex 'p $eax' \
  -ex 'c'
```

`-batch` 执行完自动退出，`-q` 不打印版权信息，`-ex` 依次执行命令。

---

## 解题套路

1. **先 `objdump -d --disassemble=<函数>` 看结构** —— 找出调用 `explode_bomb` 的所有分支，那些就是必须绕开的检查点
2. **在 `explode_bomb` 上下断点跑一遍** —— 确认自己在哪一关、哪个条件失败
3. **倒着推** —— 从最后的 `cmp` / `test` 往前找，看被比较的值从哪来
4. **凡是常量地址，都用 `x/s` 或 `x/Ngx` 读出来** —— 字符串、跳转表、查找表、链表都藏在 `.rodata` 和 `.data` 里
5. **数组/结构体用断点看栈** —— `x/6wx $rsp` 比手工推偏移快得多
6. **辅助函数单独反汇编** —— `func4` 这种递归函数，先翻译成 C 再分析

---

## 附录：环境与踩坑

### `hex2raw` 必须吃十六进制文本，不能喂原始字节

第一版直接写二进制：

```python
payload = b'A'*40 + struct.pack('<Q', 0x4017c0) + b'\n'
open('/tmp/l1.txt','wb').write(payload)
```

```bash
./hex2raw < /tmp/l1.txt
```

```text
Invalid hex value [AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA\x17@].
Please specify only single byte hex values separated by whitespace.
```

**正确做法**：生成空格分隔的两位十六进制。

```bash
python3 -c "print('41 '*40 + 'c0 17 40 00 00 00 00 00')" > l1.hex
./hex2raw < l1.hex > l1.raw
./ctarget -q -i l1.raw
```

每一行都必须是以 `#` 开头的注释，或者偶数个十六进制字节。`hex2raw` 会把它们转成原始字节流。

### 别在该用 `-q` 的时候漏掉

自学版已经关掉评分服务器连接，但不加 `-q` 有些版本仍会尝试联网然后卡住：

```bash
./ctarget -q -i exploit.raw
./rtarget -q -i exploit.raw
```

### `objdump -d` 里 `callq` / `retq` 的 `q` 后缀

反汇编输出用 `callq`、`retq`，`gdb` 的 `disassemble` 也用这个名字，但汇编源码里写 `call`、`ret` 就行。两者等价，`q` 只是强调 64 位版本。

### 断点位置决定你读到什么

这是最容易搞混的一点：

```bash
# 错：函数入口，此时 %rsp 还是调用者的
gdb -batch -q ./ctarget -ex 'b getbuf' -ex 'run -q -i empty.txt' -ex 'p/x $rsp'
```
```text
$1 = 0x5561dca0
```

```bash
# 对：sub 指令之后，%rsp 已下移 0x28
gdb -batch -q ./ctarget -ex 'b *0x4017af' -ex 'run -q -i empty.txt' -ex 'p/x $rsp'
```
```text
$1 = 0x5561dc78
```

两者相差 `0x28 = 40` 字节，正是 `sub $0x28,%rsp` 分配的大小。用错的地址做返回地址，程序跳到缓冲区别处，直接崩。

### `bomb` 崩溃时的输出看不出是哪关

加 `-q` 之外没有别的办法定位。用 `explode_bomb` 断点：

```bash
gdb -batch -q ./bomb -ex 'b explode_bomb' -ex 'run < answers.txt' -ex 'bt'
```

`backtrace` 会显示从哪个 phase 调过来的。

### 读取 `.rodata` 时 `objdump -s` 的输出要手工拼

```bash
objdump -s -j .rodata bomb | grep -A2 "402450"
```

```text
402450 73656372 65742073 74616765 2100666c  secret stage!.fl
402460 79657273 00000000 00000000 00000000  yers............
```

右边那列是可打印字符，但**跨行才构成完整字符串** —— `flyers` 被拆在 `!fl` 和 `yers` 两行里，拼起来才是 `flyers`（`!` 是前一个串 `secret stage!` 的结尾）。

用 `gdb` 的 `x/s` 更直观：

```bash
gdb -batch -q ./bomb -ex 'x/s 0x40245e'
```
```text
0x40245e:	"flyers"
```

### `strings` 有时找不到字符串

```bash
strings -t x bomb | awk '$1>="402400" && $1<="402500"'
```

没有输出，因为 `strings` 默认只打印长度 ≥4 的可打印串，而且偏移是文件偏移不是虚拟地址。用 `gdb` 的 `x/s` 或 `objdump -s` 按虚拟地址读才靠谱。

### 第 3 关的跳转表顺序不是顺序排列

```bash
gdb -batch -q ./bomb -ex 'x/8a 0x402470'
```

```text
0x402470:	0x400f7c <phase_3+57>	0x400fb9 <phase_3+118>
0x402480:	0x400f83 <phase_3+64>	0x400f8a <phase_3+71>
0x402490:	0x400f91 <phase_3+78>	0x400f98 <phase_3+85>
0x4024a0:	0x400f9f <phase_3+92>	0x400fa6 <phase_3+99>
```

注意第 2 项是 `0x400fb9`，而第 3 项是 `0x400f83` —— **不是递增的**。编译器把 `case 1` 的代码块放到了函数末尾。如果按地址顺序猜，`case 1` 对应值会算错（我第一版把 `0x2c3` 的十进制写成 705，正确是 707，靠逐组实测才发现）。

正确做法：**按表项顺序读地址，再去那个地址看 `mov $imm,%eax` 的立即数**。

### 多次 `run` 后 gdb 会累积旧状态

`-batch` 模式每次都是新进程，没问题。交互式 gdb 里如果要重新跑，先 `kill` 再 `run`，否则断点状态可能混乱。

### 第 6 关的 `7 - x` 容易算反

程序先把输入 `x` 换成 `7-x`，再用这个值作为链表节点序号。所以：

```text
需要的降序节点序号：3, 4, 5, 6, 1, 2
实际输入 = 7 - 序号：4, 3, 2, 1, 6, 5
```

直接输入 `3 4 5 6 1 2` 会失败，因为程序内部会变成 `4 3 2 1 6 5`，顺序正好反了。
