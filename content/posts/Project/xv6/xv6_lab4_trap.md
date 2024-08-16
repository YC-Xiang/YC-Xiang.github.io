---
title: xv6_lab3 trap
date: 2024-06-04
tags:
  - xv6 OS
categories:
  - Project
---

# Q1 RISC-V assembly

C 代码：

```c
int g(int x) {
  return x+3;
}

int f(int x) {
  return g(x);
}

void main(void) {
  printf("%d %d\n", f(8)+1, 13);
  exit(0);
}
```

生成的汇编代码：

```c
000000000000001c <main>:

void main(void) {
  1c:	1141                	addi	sp,sp,-16 // 分配栈空间
  1e:	e406                	sd	ra,8(sp) // 保存main的返回地址，因为接下来要调用printf
  20:	e022                	sd	s0,0(sp) // 保存前一个函数的frame pointer
  22:	0800                	addi	s0,sp,16 // 现在frame pointer要增加16Bytes
  printf("%d %d\n", f(8)+1, 13);
  24:	4635                	li	a2,13
  26:	45b1                	li	a1,12
  28:	00000517          	auipc	a0,0x0
  2c:	7a050513          	addi	a0,a0,1952 # 7c8 <malloc+0xe8> // "%d %d\n"字符串地址
  30:	00000097          	auipc	ra,0x0 // ra=pc=0x30
  34:	5f8080e7          	jalr	1528(ra) # 628 <printf> // 0x30 + 0x5f8 = 0x628
  exit(0);
  38:	4501                	li	a0,0 // exit的参数，传入0
  3a:	00000097          	auipc	ra,0x0
  3e:	274080e7          	jalr	628(ra) # 2ae <exit>
```

**1.Which registers contain arguments to functions? For example, which register holds 13 in main's call to printf?**

由生成的汇编可知，13 保存在寄存器**a2**中，f(8)+1 即 12 保存在寄存器**a1**中，字符串"%d %d\n"地址被保存到**a0**。

</br>

**2.Where is the call to function f in the assembly code for main? Where is the call to g? (Hint: the compiler may inline functions.)**

f(8)+1 在汇编中直接被展开成了 12。

</br>

**3.At what address is the function printf located?**

```c
30:	00000097  auipc	ra,0x0
34:	5f8080e7  jalr	1528(ra) # 628 <printf>
```

ra=0x30, 0x30+1528=0x628。

</br>

**4.What value is in the register ra just after the jalr to printf in main?**

执行 jalr 指令后，ra 寄存器被赋值为下一条指令，即 0x34+0x4=0x38 地址的指令：li a0,0。

</br>

**5.Run the following code.**

```c
unsigned int i = 0x00646c72;
printf("H%x Wo%s", 57616, &i);
```

What is the output?

HE110, World.

第一个%x, 57616 对应十六进制 0xE110，第二个&i, 因为是小端序，解析出来的字符串排列顺序为`0x72, 0x6c, 0x64, 0x00`，查看 ascii 码表，对应字符串`orld`。

</br>

**6.In the following code, what is going to be printed after 'y='? (note: the answer is not a specific value.) Why does this happen?**

`printf("x=%d y=%d", 3);`

y 打印出来是个随机数。

# Q2 Backtrace

栈分布参考如图：

![](https://xyc-1316422823.cos.ap-shanghai.myqcloud.com/20240225223002.png)

<p class="note note-info">根据CSAPP 3.7.1栈结构看，return address属于前一栈帧保存的，所以fp register应该指向return address的地址，即当前栈帧的顶部。一个栈帧中保存在最高位地址的是previous fp。所以这里最上面一个stack frame也应该把return address去掉。</p>

读出当前的 frame pointer。xv6 给栈分配的空间是一个 page，当 fp 不指向该页的最高地址时，说明不是调用的第一个函数。

读出当前函数的返回地址，即上一层函数调用该函数的下一条指令，在 fp-8 的位置。因为我们要的是调用函数的地址，所以需要在返回地址再减去 8。  
读出上一层函数的 fp 地址，在 fp-16 的位置。

```c
void backtrace(void)
{
	uint64 fp = r_fp();
	uint64 ra;

	printf("backtrace:\n");
	while (fp < PGROUNDUP(fp))
	{
		ra = *(uint64 *)(fp - 8) - 8;
		printf("%p\n", ra);
		fp = *(uint64 *)(fp - 16);
	}
}
```

make qemu 后执行 bttest, 得到函数调用过程：

```c
$ bttest
backtrace:
0x0000000080002132
0x0000000080001fa4
0x0000000080001c8e
```

利用`addr2line -e kernel/kernel`

# Q3 Alarm
