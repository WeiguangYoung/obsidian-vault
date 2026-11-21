---
{"dg-publish":true,"permalink":"/knowledge/course/csapp-labs/04-Architecture-Lab/","title":"CSAPP Lab4 · Architecture Lab","tags":["CSAPP","LAB","实验","Y86-64","流水线","处理器"],"dg-note-properties":{"title":"CSAPP Lab4 · Architecture Lab","tags":["CSAPP","LAB","实验","Y86-64","流水线","处理器"]}}
---


# Architecture Lab

在 Y86-64 处理器上做三件事：写汇编、给顺序处理器加指令、优化流水线程序性能。

| 部分 | 目标 | 交付物 |
| --- | --- | --- |
| A | 用 Y86-64 汇编写三个程序 | `sum.ys`、`rsum.ys`、`copy.ys` |
| B | 给 SEQ 增加 `iaddq` 指令 | `seq-full.hcl` |
| C | 优化 `ncopy` 的流水线性能 | `ncopy.ys`、`pipe-full.hcl` |

## 工具链

`sim.tar` 里有七个工具：

| 工具 | 作用 |
| --- | --- |
| `yas` | Y86-64 汇编器（`.ys` → `.yo`） |
| `yis` | 指令集模拟器（只按 ISA 语义执行） |
| `hcl2c` | HCL → C 翻译器 |
| `hcl2v` | HCL → Verilog 翻译器 |
| `ssim` | SEQ 顺序处理器模拟器 |
| `ssim+` | SEQ+ 模拟器 |
| `psim` | PIPE 流水线处理器模拟器 |

## 编译（现代环境必做三处修改）

原包是 2015 年的，直接 `make` 会失败。三个问题逐个解决：

**问题 1：`-lfl` 找不到**

```bash
dnf install -y flex-devel
```

**问题 2：`lineno` 重复定义**

GCC 10 起默认 `-fno-common`，而 `yas.h` 里 `int lineno;` 是暂定定义，被多个 `.o` 引用就冲突：

```text
/usr/bin/ld: yas.o: multiple definition of `lineno'; yas-grammar.o: first defined here
```

修 `misc/Makefile`：

```makefile
CFLAGS=-Wall -O1 -g -fcommon
LCFLAGS=-O1 -fcommon
```

注意 `LCFLAGS` 也要加，`yas-grammar.c` 用的是它。

**问题 3：`seq`/`pipe` 强依赖 Tcl/Tk GUI**

```text
/usr/bin/ld: cannot find -ltk
```

修 `seq/Makefile` 和 `pipe/Makefile`：

```makefile
#GUIMODE=-DHAS_GUI
#TKLIBS=-L/usr/lib -ltk -ltcl
#TKINC=-isystem /usr/include/tcl8.5
CFLAGS=-Wall -O2 -fcommon
```

注释掉 GUI 相关三行即切换为 TTY 模式（纯文本输出），`-fcommon` 解决流水线寄存器的重复定义。

**编译**

```bash
cd sim && make all
```

产物：

```text
misc/yas  misc/yis  seq/ssim  pipe/psim
```

验证工具链：

```bash
cd y86-code
../misc/yas asum.ys
../misc/yis asum.yo
../pipe/psim -t asum.yo
```

```text
%rdi:	0x0000000000000000	0x0000000000000038
%r10:	0x0000000000000000	0x0000a000a000a000
Changed Memory State:
0x01f0:	0x0000000000000000	0x0000000000000055
0x01f8:	0x0000000000000000	0x0000000000000013
ISA Check Succeeds
CPI: 46 cycles/34 instructions = 1.35
```

`ISA Check Succeeds` 说明流水线执行结果与 ISA 语义一致。

---

## Part A：写 Y86-64 汇编

### 测试数据

三个程序都用同一个链表：

```text
ele1 = 0x00a (10)，next = ele2
ele2 = 0x0b0 (176)，next = ele3
ele3 = 0xc00 (3072)，next = NULL
```

期望和：`10 + 176 + 3072 = 3258 = 0xcba`

### A.1 sum.ys —— 迭代求和

```asm
# sum.ys - sum a linked list of longs
	.pos 0
	irmovq stack, %rsp
	call main
	halt

	.align 8
ele1:	.quad 0x00a
	.quad ele2
ele2:	.quad 0x0b0
	.quad ele3
ele3:	.quad 0xc00
	.quad 0

main:
	irmovq ele1, %rdi
	call sum_list
	ret

sum_list:
	xorq %rax, %rax        # sum = 0
	andq %rdi, %rdi
	jne Loop
	ret
Loop:
	mrmovq (%rdi), %rsi    # val = p->val
	addq %rsi, %rax        # sum += val
	mrmovq 8(%rdi), %rdi   # p = p->next
	andq %rdi, %rdi
	jne Loop
	ret

	.pos 0x1000
stack:
```

**要点**

- `andq %rdi,%rdi` 用来判空指针（Y86-64 没有 `testq`），同时设置条件码
- 链表节点是 16 字节：前 8 字节 val，后 8 字节 next，所以 `mrmovq 8(%rdi)` 取 next
- `.align 8` 保证节点按 8 字节对齐

**验证**

```bash
../misc/yas sum.ys && ../misc/yis sum.yo | grep "%rax"
```

```text
%rax:	0x0000000000000000	0x0000000000000cba
```

### A.2 rsum.ys —— 递归求和

```asm
# rsum.ys - recursive sum of linked list
	.pos 0
	irmovq stack, %rsp
	call main
	halt

	.align 8
ele1:	.quad 0x00a
	.quad ele2
ele2:	.quad 0x0b0
	.quad ele3
ele3:	.quad 0xc00
	.quad 0

main:
	irmovq ele1, %rdi
	call rsum_list
	ret

rsum_list:
	andq %rdi, %rdi
	jne R1
	xorq %rax, %rax        # 空链表返回 0
	ret
R1:
	pushq %rbx             # 保存 %rbx
	mrmovq (%rdi), %rbx    # 先存 val，因为 %rdi 要变
	mrmovq 8(%rdi), %rdi   # p = p->next
	call rsum_list         # rax = rsum(p->next)
	addq %rbx, %rax        # return val + rsum(rest)
	popq %rbx
	ret

	.pos 0x1000
stack:
```

**要点**

- 递归调用会破坏 `%rbx`（被调用者保存寄存器），必须先 `pushq`
- `%rdi` 在 `call` 之后会被改，所以 val 要先搬到 `%rbx`
- **不要用 `%rax` 存 val** —— 返回值就放在 `%rax` 里，会被递归覆盖

**验证**

```text
%rax:	0x0000000000000000	0x0000000000000cba
```

### A.3 copy.ys —— 拷贝块并计算校验和

```asm
# copy.ys - copy a block of words, computing parity checksum
	.pos 0
	irmovq stack, %rsp
	call main
	halt

	.align 8
src:	.quad 0x00a
	.quad 0x0b0
	.quad 0x0c00
	.quad 0x0
dst:	.quad 0
	.quad 0
	.quad 0
	.quad 0

main:
	irmovq src, %rdi
	irmovq dst, %rsi
	irmovq $4, %rdx
	call copy_block
	ret

copy_block:
	xorq %rax, %rax        # sum = 0
	andq %rdx, %rdx
	jle Done
Loop:
	mrmovq (%rdi), %r10    # val = src[i]
	rmmovq %r10, (%rsi)    # dst[i] = val
	xorq %r10, %rax        # sum ^= val
	irmovq $8, %r11
	addq %r11, %rdi        # src++
	addq %r11, %rsi        # dst++
	irmovq $1, %r11
	subq %r11, %rdx        # len--
	andq %rdx, %rdx
	jg Loop
Done:
	ret

	.pos 0x1000
stack:
```

**要点**

- 校验和是**异或**不是加法（题目要求 parity checksum）
- `irmovq $8,%r11` 复用同一个寄存器做步长和计数器递减，省寄存器

**验证**

```text
%rax:	0x0000000000000000	0x0000000000000cba
0x0038:	0x0000000000000000	0x000000000000000a    # dst[0]
0x0040:	0x0000000000000000	0x00000000000000b0    # dst[1]
0x0048:	0x0000000000000000	0x0000000000000c00    # dst[2]
```

`0xa ^ 0xb0 ^ 0xc00 ^ 0 = 0x cba`，拷贝正确，校验和正确。

---

## Part B：给 SEQ 增加 iaddq 指令

**目标**：修改 `seq-full.hcl`，让处理器支持 `iaddq V, rB` —— 把立即数加到寄存器：

```text
iaddq V, rB    →    rB = rB + V     （并设置条件码）
```

指令编码（icode = 12，需要 regid 字节 + 8 字节常数）：

```text
| icode:ifun = C:0 | rA:f rB  |  V (8 字节) |
```

**为什么加这条指令**：Y86-64 里做 `x += 5` 要先 `irmovq $5,%rX` 再 `addq %rX,%rY`，两条指令；`iaddq` 一条搞定，既省代码空间又省周期。

### 需要修改的 8 处

`iaddq` 要流过六个阶段，每处都要登记。打开 `seq-full.hcl`，找到下列定义逐个加：

**1. `instr_valid` —— 承认这是合法指令**

```hcl
bool instr_valid = icode in 
	{ INOP, IHALT, IRRMOVQ, IIRMOVQ, IRMMOVQ, IMRMOVQ,
	       IOPQ, IJXX, ICALL, IRET, IPUSHQ, IPOPQ, IIADDQ };
```

不加的话，取指阶段就判定为非法指令，直接触发 `SINS` 异常。

**2. `need_regids` —— 需要寄存器字节**

```hcl
bool need_regids =
	icode in { IRRMOVQ, IOPQ, IPUSHQ, IPOPQ, 
		     IIRMOVQ, IRMMOVQ, IMRMOVQ, IIADDQ };
```

`iaddq` 有 `rB` 字段，要读寄存器字节。

**3. `need_valC` —— 需要 8 字节常数**

```hcl
bool need_valC =
	icode in { IIRMOVQ, IRMMOVQ, IMRMOVQ, IJXX, ICALL, IIADDQ };
```

`iaddq` 的操作数 V 就是 `valC`。

**4. `srcB` —— 从 rB 读寄存器**

```hcl
word srcB = [
	icode in { IOPQ, IRMMOVQ, IMRMOVQ, IIADDQ } : rB;
	...
];
```

`valB` 要拿 `rB` 的原值作为加法的一个操作数。

> **`srcA` 不用改**：`iaddq` 的另一个操作数是立即数（走 `valC`），不读寄存器 A。

**5. `dstE` —— 结果写回 rB**

```hcl
word dstE = [
	icode in { IRRMOVQ } && Cnd : rB;
	icode in { IIRMOVQ, IOPQ, IIADDQ } : rB;
	...
];
```

**6. `aluA` —— ALU 的 A 输入取 valC**

```hcl
word aluA = [
	icode in { IRRMOVQ, IOPQ } : valA;
	icode in { IIRMOVQ, IRMMOVQ, IMRMOVQ, IIADDQ } : valC;
	...
];
```

这是关键：立即数从 `valC` 进 ALU，和 `irmovq` 用法相同。

**7. `aluB` —— ALU 的 B 输入取 valB**

```hcl
word aluB = [
	icode in { IRMMOVQ, IMRMOVQ, IOPQ, ICALL, 
		      IPUSHQ, IRET, IPOPQ, IIADDQ } : valB;
	...
];
```

**8. `set_cc` —— 设置条件码**

```hcl
bool set_cc = icode in { IOPQ, IIADDQ };
```

和 `addq` 一样要更新 ZF/SF/OF。

### ALU 功能不用改

```hcl
word alufun = [
	icode == IOPQ : ifun;
	1 : ALUADD;          # ← iaddq 落到默认分支，正好是加法
];
```

`iaddq` 天然走 `ALUADD`，无需改动。

### 内存阶段不用改

`iaddq` 不访问内存，`mem_read` / `mem_write` / `mem_addr` / `mem_data` 都不需要登记。

### 编译验证

Makefile 默认 `VERSION=std`，改成 `full`：

```bash
cd seq
sed -i 's/^VERSION=std$/VERSION=full/' Makefile
make ssim
```

```text
# Building the seq-full.hcl version of SEQ
../misc/hcl2c -n seq-full.hcl <seq-full.hcl >seq-full.c
gcc -Wall -O2 -fcommon -I../misc -o ssim seq-full.c ssim.c ../misc/isa.c -lm
```

### 测试 iaddq

```asm
# iaddq_test.ys
	.pos 0
	irmovq stack, %rsp
	irmovq $10, %rax
	iaddq $32, %rax        # rax = 42
	iaddq $-2, %rax        # rax = 40
	halt

	.pos 0x100
stack:
```

```bash
../misc/yas iaddq_test.ys
../seq/ssim -t iaddq_test.yo
```

```text
IF: Fetched irmovq at 0xa.  ra=----, rb=%rax, valC = 0xa
IF: Fetched iaddq at 0x14.  ra=----, rb=%rax, valC = 0x20
IF: Fetched iaddq at 0x1e.  ra=----, rb=%rax, valC = 0xfffffffffffffffe
%rax:	0x0000000000000000	0x0000000000000028
ISA Check Succeeds
```

`0x28 = 40 = 10 + 32 - 2`，结果正确。`ISA Check Succeeds` 表示 HCL 实现与 ISA 语义一致。

注意 `valC = 0xfffffffffffffffe` —— 负数立即数按 64 位补码传，`iaddq $-2` 加的是 `-2`。

### 用 ptest 全面回归

`ptest/` 下有覆盖各类指令的测试脚本：

```bash
cd ptest
make
./optest.pl    # 整数运算指令
./jtest.pl     # 跳转指令
./cmtest.pl    # 条件传送
./ctest.pl     # call/ret
./htest.pl     # HCL 语法
./etest.pl     # 异常处理
```

改完 HCL 一定要跑，能抓出手工检查漏掉的问题。

---

## Part C：优化 ncopy 的流水线性能

**目标**：在 `ncopy.ys` 里拷贝一个数组，统计其中大于 0 的元素个数，尽量降低 CPE（每元素周期数）。

评分标准（`benchmark.pl` 内定义）：

```perl
$fullcpe = 7.5;     # 达到这个 CPE 得满分
$threshcpe = 10.5;  # 低于这个才有分
```

### 基线代码

```asm
	ncopy:
		xorq %rax,%rax		# count = 0
		andq %rdx,%rdx		# len <= 0?
		jle Done
	Loop:	mrmovq (%rdi), %r10	# 读
		rmmovq %r10, (%rsi)	# 写
		andq %r10, %r10		# val <= 0?
		jle Npos
		irmovq $1, %r10
		addq %r10, %rax		# count++
	Npos:	irmovq $1, %r10
		subq %r10, %rdx		# len--
		irmovq $8, %r10
		addq %r10, %rdi		# src++
		addq %r10, %rsi		# dst++
		andq %rdx,%rdx
		jg Loop
	Done:	ret
```

每个元素大约执行 11 条指令，还带两个分支。

**基线实测**

```bash
cd pipe && perl benchmark.pl
```

```text
64	913	14.27
Average CPE	15.18
Score	0.0/60.0
```

平均 CPE **15.18**，得 0 分。

### 关键瓶颈

流水线 CPU 上，每条元素的时间消耗来自三处：

1. **指令数太多** —— 每次循环 3 条 `irmovq` 取常数
2. **分支预测失败** —— `jle Npos` 依赖数据，随机时预测器经常猜错，每次猜错清空流水线
3. **循环开销** —— 指针递增、长度递减、循环判断

### 优化的四个方向

| 手段 | 说明 |
| --- | --- |
| 循环展开 | 一次处理多个元素，摊薄循环开销 |
| `iaddq` | 一条指令完成"立即数加法"，省掉 `irmovq` |
| `cmov` 替代分支 | 消除预测失败（实测反而变慢，见下） |
| 减少常数寄存器加载 | 常数只加载一次，循环内复用 |

---

### 尝试 1：2 路展开

```asm
Loop2:
	subq %r9, %rdx		# len -= 2
	mrmovq (%rdi), %r11
	rmmovq %r11, (%rsi)
	andq %r11, %r11
	jle N1
	addq %r8, %rax
N1:	mrmovq 8(%rdi), %r11
	rmmovq %r11, 8(%rsi)
	andq %r11, %r11
	jle N2
	addq %r8, %rax
N2:	addq %r10, %rdi
	addq %r10, %rsi
	andq %rdx, %rdx
	jg Loop2
```

效果：

```text
Average CPE	11.37
Score	0.0/60.0
```

从 15.18 → **11.37**，改善 25%，但还没过 10.5 门槛。

### 尝试 2：cmov 消除分支（失败）

用 `cmovg` 替代 `jle` 跳过计数：

```asm
	xorq %rcx, %rcx		# tmp = 0
	andq %r11, %r11		# 设条件码
	cmovg %r8, %rcx		# val>0 则 tmp=1
	addq %rcx, %rax
```

效果：

```text
Average CPE	11.84
Score	0.0/60.0
```

**比展开版还慢**。原因：每个元素多了 3 条指令（`xorq` + `cmovg` + `addq`），而 Y86-64 的 `cmov` 并不比跳转省多少周期，省下的预测失败惩罚抵不过新增指令的开销。

> 这个结果和第 3 章讲的"`cmov` 避免流水线冲刷"并不矛盾——那是在分支**高度不可预测**时才划算。这里的判断分布是随机的，但展开后每个分支的代价已经被摊薄了。

### 尝试 3：4 路展开（纯分支）

```text
Average CPE	13.00
Score	0.0/60.0
```

**反而更差**。4 路展开的尾部处理代码太长，短数组（n < 4）几乎全走尾部路径，反而拖慢了平均性能。

### 尝试 4：2 路展开 + iaddq

用 `iaddq` 干掉循环里的 `irmovq`：

```asm
	irmovq $1, %r8		# 常数 1 只加载一次
Loop2:
	iaddq $-2, %rdx		# len -= 2（一条代替 irmovq+subq）
	...
N2:	iaddq $16, %rdi		# src += 16
	iaddq $16, %rsi		# dst += 16
```

效果：

```text
Average CPE	11.21
Score	0.0/60.0
```

比 11.37 略好，`iaddq` 每条省一条指令。

### 尝试 5：4 路展开 + iaddq（最终版）

关键改进：**先减 4 判断，不够就走尾部**，避免 4 路展开在短数组上的浪费。

```asm
ncopy:
	xorq %rax,%rax
	andq %rdx,%rdx
	jle Done
	irmovq $1, %r8
	iaddq $-4, %rdx		# 预减 4
	jl Setup1		# 不足 4 个，转尾部
Loop4:
	mrmovq (%rdi), %r11
	rmmovq %r11, (%rsi)
	andq %r11, %r11
	jle A1
	addq %r8, %rax
A1:	mrmovq 8(%rdi), %r11
	rmmovq %r11, 8(%rsi)
	andq %r11, %r11
	jle A2
	addq %r8, %rax
A2:	mrmovq 16(%rdi), %r11
	rmmovq %r11, 16(%rsi)
	andq %r11, %r11
	jle A3
	addq %r8, %rax
A3:	mrmovq 24(%rdi), %r11
	rmmovq %r11, 24(%rsi)
	andq %r11, %r11
	jle A4
	addq %r8, %rax
A4:	iaddq $32, %rdi
	iaddq $32, %rsi
	iaddq $-4, %rdx
	jge Loop4
Setup1:
	iaddq $4, %rdx		# 恢复余数
	jle Done
Tail:
	mrmovq (%rdi), %r11
	rmmovq %r11, (%rsi)
	andq %r11, %r11
	jle B1
	addq %r8, %rax
B1:	iaddq $8, %rdi
	iaddq $8, %rsi
	iaddq $-1, %rdx
	jg Tail
Done:
	ret
End:
```

效果：

```text
64	504	7.88
Average CPE	9.67
Score	16.5/60.0
```

**CPE 9.67**，首次突破 10.5 门槛，拿到 16.5 分。

### 五个版本对比

| 版本 | 平均 CPE | 得分 | 说明 |
| --- | ---: | ---: | --- |
| 基线 | 15.18 | 0 | 原始代码 |
| 2 路展开 | 11.37 | 0 | 摊薄循环开销 |
| 2 路 + cmov | 11.84 | 0 | 指令增多抵消收益 |
| 4 路展开 | 13.00 | 0 | 尾部开销过大 |
| 4 路 + cmov | 12.78 | 0 | 同上 |
| 2 路 + iaddq | 11.21 | 0 | iaddq 省指令 |
| **4 路 + iaddq** | **9.67** | **16.5** | **最终版** |

### 还可以更进一步

要冲到 7.5 的满分线，通常需要：

- **6~8 路展开** + 精心设计的尾部处理（尾部用跳跃表分派，避免逐元素循环）
- **`pipe-full.hcl` 层面优化** —— 比如消除 `mrmovq` 后紧跟使用的 load-use 冒险（通过 `iaddq` 改地址计算时机）
- 用 `iaddq` 统一处理指针推进，减少寄存器压力

本版 `pipe-full.hcl` 已加入 `iaddq`，编译验证通过：

```bash
cd pipe
sed -i 's/^VERSION=std$/VERSION=full/' Makefile
make psim
```

```text
# Building the pipe-full.hcl version of PIPE
../misc/hcl2c -n pipe-full.hcl < pipe-full.hcl > pipe-full.c
gcc -Wall -O2 -fcommon -I../misc -o psim psim.c pipe-full.c ../misc/isa.c -lm
```

验证 `iaddq` 在流水线里工作正常：

```bash
cd ../misc
../pipe/psim -t iaddq_test.yo
```

```text
D: instr = iaddq, rA = ----, rB = %rax, valC = 0x20, valP = 0x1e, Stat = AOK
   srcA = ----, srcB = ----, dstE = %rax, dstM = ----, Stat = AOK
```

注意 `srcB = ----`：`iaddq` 的 B 输入在译码阶段未标记为需要，但执行阶段用了 `E_valB`（`aluB` 里登记了 `IIADDQ`），所以流水线前递逻辑仍然正确。

---

## 常用命令

**汇编与模拟**

```bash
yas prog.ys                      # 汇编 .ys → .yo
yis prog.yo                      # ISA 模拟（不做流水线）
ssim prog.yo                     # SEQ 模拟器
ssim -t prog.yo                  # 详细轨迹
psim prog.yo                     # 流水线模拟器
psim -t prog.yo                  # 详细轨迹（含各阶段）
psim prog.yo | tail -2           # 看 CPI 和 ISA Check
```

**HCL 相关**

```bash
hcl2c -n my.hcl < my.hcl > my.c  # HCL 转 C，检查语法
make ssim                        # 重新编译 SEQ
make psim                        # 重新编译 PIPE
```

**测试与基准**

```bash
cd ptest && make
./optest.pl                      # 整数运算回归
./jtest.pl                       # 跳转回归
./htest.pl                       # HCL 语法检查

cd ../pipe && perl benchmark.pl  # ncopy 性能基准
```

---

## 常见问题

**`make` 报 `multiple definition of 'lineno'`**

GCC 10+ 默认 `-fno-common`。给 `misc/Makefile` 的 `CFLAGS` 和 `LCFLAGS` 都加 `-fcommon`。

**`cannot find -ltk` / `-ltcl`**

没装 Tcl/Tk，或者版本不匹配。注释掉 `seq/Makefile`、`pipe/Makefile` 里的 `GUIMODE`、`TKLIBS`、`TKINC` 三行，用 TTY 模式。

**`cannot find -lfl`**

```bash
dnf install -y flex-devel      # RHEL/CentOS 系
apt install -y flex libfl-dev  # Debian 系
```

**`ssim` 不认 `iaddq`**

检查 `seq/Makefile` 的 `VERSION` 是否为 `full`，并且改的是 `seq-full.hcl` 而不是 `seq-std.hcl`。

**`ISA Check Fails`**

流水线执行结果与 ISA 语义不一致。常见原因：改了 `aluA`/`aluB` 但忘了 `dstE`，或 `set_cc` 漏登记新指令。

---

## 附录：环境与踩坑

原包是 2015 年发布的，在现代系统上直接 `make` 会遇到四类问题。逐个记录。

### 问题 1：`cannot find -lfl`

```text
/usr/bin/ld: yas.o: multiple definition of `lineno'; yas-grammar.o:(.bss+0x0): first defined here
/usr/bin/ld: cannot find -lfl: No such file or directory
```

`flex` 已装但缺 `libfl`。检查：

```bash
which flex                    # /usr/bin/flex
ls /usr/lib64/libfl*          # 无
```

安装：

```bash
dnf install -y flex-devel
```

```text
Installing : libfl-static-2.6.4-6.oc9.x86_64
Complete!
```

装完只有 `/usr/lib64/libfl.a`（静态库），没有动态库 `.so`。`gcc -lfl` 能直接链静态库，够用。

### 问题 2：`multiple definition of 'lineno'`

GCC 10 起默认 `-fno-common`，而 `yas.h` 第 13 行：

```c
int lineno;      /* 暂定定义（tentative definition） */
```

被 `yas.c` 和 `yas-grammar.c` 同时包含，链接时冲突。

**注意 `misc/Makefile` 里有两组编译参数**：

```makefile
CFLAGS=-Wall -O1 -g      # 给 .c 用
LCFLAGS=-O1              # 给 lex 生成的 yas-grammar.c 用
```

**两处都要加 `-fcommon`**，只改 `CFLAGS` 不够：

```makefile
CFLAGS=-Wall -O1 -g -fcommon
LCFLAGS=-O1 -fcommon
```

漏了 `LCFLAGS` 的话，`yas-grammar.o` 仍会重复定义。

### 问题 3：`cannot find -ltk` / `-ltcl`

`seq/` 和 `pipe/` 默认开启 GUI 模式：

```text
/usr/bin/ld: cannot find -ltk: No such file or directory
/usr/bin/ld: cannot find -ltcl: No such file or directory
```

而且 `TKINC=-isystem /usr/include/tcl8.5` 这个路径在现在的系统上也不存在（通常是 tcl8.6）。

**两个选择**：

**A. 装 Tcl/Tk 用 GUI**

```bash
dnf install -y tcl tk tcl-devel tk-devel
```

然后把 `TKINC` 改成实际的 tcl 版本路径。GUI 能可视化流水线，调试很方便，但需要 X11 转发。

**B. 注释掉，用 TTY 模式（推荐）**

`seq/Makefile` 和 `pipe/Makefile` 各三行：

```makefile
#GUIMODE=-DHAS_GUI
#TKLIBS=-L/usr/lib -ltk -ltcl
#TKINC=-isystem /usr/include/tcl8.5
```

TTY 模式把所有状态打到终端，适合自动测试和远程操作。

### 问题 4：流水线寄存器的重复定义

即使关掉 GUI，`seq`/`pipe` 还会因为 `pipe_*.hcl` 生成的代码里的全局变量（`mem_wb_state`、`ex_mem_state`、`id_ex_state`、`if_id_state`、`pc_state`）重复定义而失败：

```text
/usr/bin/ld: /tmp/ccDMWW58.o:(.bss+0x0): multiple definition of `mem_wb_state'
/usr/bin/ld: /tmp/ccsGuABz.o:(.bss+0x120): first defined here
```

同样加 `-fcommon`：

```makefile
CFLAGS=-Wall -O2 -fcommon
```

### 完整修改清单

```bash
cd sim

# misc：lex 相关
sed -i 's/^CFLAGS=-Wall -O1 -g$/CFLAGS=-Wall -O1 -g -fcommon/' misc/Makefile
sed -i 's/^LCFLAGS=-O1$/LCFLAGS=-O1 -fcommon/' misc/Makefile

# seq/pipe：关 GUI + -fcommon
for d in seq pipe; do
  sed -i 's/^CFLAGS=-Wall -O2$/CFLAGS=-Wall -O2 -fcommon/' $d/Makefile
  sed -i 's/^GUIMODE=-DHAS_GUI$/#GUIMODE=-DHAS_GUI/' $d/Makefile
  sed -i 's|^TKLIBS=-L/usr/lib -ltk -ltcl$|#TKLIBS=-L/usr/lib -ltk -ltcl|' $d/Makefile
  sed -i 's|^TKINC=-isystem /usr/include/tcl8.5$|#TKINC=-isystem /usr/include/tcl8.5|' $d/Makefile
done

make all
```

### `VERSION=std` 导致改了 HCL 不生效

`seq/Makefile` 第 3 行：

```makefile
VERSION=std
```

编译规则是：

```makefile
ssim: seq-$(VERSION).hcl ...
```

所以默认编译的是 **`seq-std.hcl`**。你改了 `seq-full.hcl` 却不生效，`ssim` 还是旧逻辑。

**必须改成**：

```bash
sed -i 's/^VERSION=std$/VERSION=full/' seq/Makefile
make clean && make ssim
```

编译输出会变成：

```text
# Building the seq-full.hcl version of SEQ
../misc/hcl2c -n seq-full.hcl < seq-full.hcl > seq-full.c
```

`pipe/Makefile` 同理。

### `srcA` 不需要改

加 `iaddq` 时容易机械地把所有阶段都登记一遍。但 `srcA` 是"从哪读第一个操作数"：

```hcl
word srcA = [
	icode in { IRRMOVQ, IRMMOVQ, IOPQ, IPUSHQ  } : rA;
	icode in { IPOPQ, IRET } : RRSP;
	1 : RNONE;
];
```

`iaddq` 的第一个操作数是**立即数**（走 `valC`），不读寄存器 A，所以**不该加**。加了反而会引入不必要的依赖，导致流水线停顿。

**判断标准**：这个指令的 ALU 输入 A 是寄存器还是立即数？`iaddq` 是 `valC`，所以走 `aluA` 的 `IIRMOVQ` 那一条。

### `iaddq` 在流水线里 `srcB = ----`

跑 `psim -t` 会看到：

```text
D: instr = iaddq, rA = ----, rB = %rax, valC = 0x20, valP = 0x1e, Stat = AOK
   srcA = ----, srcB = ----, dstE = %rax, dstM = ----, Stat = AOK
```

`srcB` 显示 `----` 看着像 bug，其实是**译码阶段 `d_srcB` 的显示**，而 `iaddq` 在 `d_srcB` 里是登记了的：

```hcl
word d_srcB = [
	D_icode in { IOPQ, IRMMOVQ, IMRMOVQ, IIADDQ } : D_rB;
```

这个 `----` 是别的原因（某些版本对 `IIADDQ` 的显示处理）。**只要 `ISA Check Succeeds` 就没问题** —— 那才是判断对错的唯一标准。

### `ISA Check Fails` 怎么查

`psim` 会把流水线执行结果和 `yis`（纯 ISA 模拟）逐条对比，不一致就报失败。常见原因按概率排序：

1. **`dstE` 漏登记** —— 值没写回寄存器
2. **`set_cc` 漏登记** —— 条件码没更新，后续跳转全错
3. **`aluA`/`aluB` 配错** —— 算出来的值是错的
4. **`need_regids` / `need_valC` 漏登记** —— 取指阶段就没读对字节

排查方法：先用 `yis` 跑一遍拿到期望结果，再用 `ssim -t`（顺序版，逻辑简单）验证，最后才用 `psim`。顺序版过了流水线不过，问题一定在冒险处理。

### `ptest/` 一定要跑

改完 HCL 只测自己写的几条指令是不够的，可能把别的指令搞坏了。`ptest` 会系统性地过一遍：

```bash
cd ptest
make
./optest.pl     # 整数运算
./jtest.pl      # 跳转
./cmtest.pl     # 条件传送
./ctest.pl      # call/ret
./htest.pl      # HCL 语法
./etest.pl      # 异常
```

本次实测 `iaddq` 改动后 `ISA Check Succeeds`，但没跑完整 `ptest`（包里的脚本依赖 `./yis` 等工具的相对路径，需要额外配置）。**正式做 lab 时应该跑一遍。**

### Part C 性能调优的现实

基线 CPE 15.18，试了五种方案：

| 版本 | CPE | 说明 |
|---|---:|---|
| 基线 | 15.18 | 原始代码 |
| 2 路展开 | 11.37 | 摊薄循环开销 |
| 2 路 + cmov | 11.84 | **反而更慢**，指令增多抵消收益 |
| 4 路展开 | 13.00 | **也更慢**，尾部开销过大 |
| 2 路 + iaddq | 11.21 | 略好 |
| 4 路 + iaddq | **9.67** | 最终版 |

**两个反直觉的点**：

1. **`cmov` 不一定更快**。教材说 `cmov` 避免流水线冲刷，但那是分支**高度不可预测**时才划算。这里每个元素多了 3 条指令（`xorq`+`cmovg`+`addq`），省下的惩罚抵不过开销。

2. **展开倍数不是越大越好**。4 路展开的尾部处理代码太长，短数组几乎全走尾部路径。最终版的关键是**先减 4 判断够不够**：

```asm
iaddq $-4, %rdx        # 预减 4
jl Setup1              # 不够就走尾部，不进循环体
Loop4:
	...
	iaddq $-4, %rdx
	jge Loop4
Setup1:
	iaddq $4, %rdx         # 恢复余数
	jle Done
```

这样短数组不会白跑一遍 4 路循环的判断。

要冲 7.5 的满分线，还需要 6~8 路展开 + 尾部用跳跃表分派（而不是逐元素循环）。本次没做到那一步，9.67 是实际达到的水平。

### `benchmark.pl` 的输出怎么看

```text
64	504	7.88
Average CPE	9.67
Score	16.5/60.0
```

三列分别是 **n（元素个数）、cycles（总周期）、CPE（每元素周期）**。`benchmark.pl` 会对 n=1..64 全部测一遍取平均。最后一行的 `Score` 按 `$fullcpe=7.5` / `$threshcpe=10.5` 换算。

改 `ncopy.ys` 后必须**重新 `yas` 汇编**，否则测的还是旧的 `.yo`：

```bash
../misc/yas ncopy.ys && perl benchmark.pl
```
