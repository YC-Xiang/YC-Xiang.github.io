---
title: CSAPP - Bomb Lab
date: 2024-06-13
tags:
- CSAPP
categories:
- Project
---

# Setup

新建`solution.txt`保存答案, 运行`./bomb solution.txt`进行测试。

`gdb bomb`

```shell
(gdb) r solution.txt
```

注意如果`solution.txt` 每行的结尾需要是LF结尾，否则无法正确解析。

`objdump -t bomb > bomb_symbol_table.txt`: 生成符号表。  
`objdump -d bomb > bomb.txt`: 生成反汇编文件。

# Answer

## Bomb1

首先根据`bomb.txt`找到`bomb`的执行入口`main`函数。

```txt
  400e32:	e8 67 06 00 00       	call   40149e <read_line>	# 读取输入的一行
  400e37:	48 89 c7             	mov    %rax,%rdi	# 把读取的字符串地址传入%rdi
  400e3a:	e8 a1 00 00 00       	call   400ee0 <phase_1>	# 调用phase_1，传入的第一个参数为%rdi
  400e3f:	e8 80 07 00 00       	call   4015c4 <phase_defused>
  400e44:	bf a8 23 40 00       	mov    $0x4023a8,%edi
  400e49:	e8 c2 fc ff ff       	call   400b10 <puts@plt>

  400e4e:	e8 4b 06 00 00       	call   40149e <read_line>
  400e53:	48 89 c7             	mov    %rax,%rdi
  400e56:	e8 a1 00 00 00       	call   400efc <phase_2>
  400e5b:	e8 64 07 00 00       	call   4015c4 <phase_defused>
  400e60:	bf ed 22 40 00       	mov    $0x4022ed,%edi
  400e65:	e8 a6 fc ff ff       	call   400b10 <puts@plt>
  //...
```

找到这段，可以看出phase_1就是我们要解决的第一个bomb。

```txt
0000000000400ee0 <phase_1>:
  400ee0:	48 83 ec 08          	sub    $0x8,%rsp	# 栈指针-8
  400ee4:	be 00 24 40 00       	mov    $0x402400,%esi	# 传入第二个字符串地址
  400ee9:	e8 4a 04 00 00       	call   401338 <strings_not_equal>	# 调用strings_not_equal，第一个参数为main中传入的字符串地址，保存在%rdi中，第二个参数为第二个字符串地址0x402400。
  400eee:	85 c0                	test   %eax,%eax	# 根据strings_not_equal的返回值设置条件码
  400ef0:	74 05                	je     400ef7 <phase_1+0x17>	# 返回值为0则跳过引爆炸弹
  400ef2:	e8 43 05 00 00       	call   40143a <explode_bomb>
  400ef7:	48 83 c4 08          	add    $0x8,%rsp
  400efb:	c3                   	ret
```

启动gdb `gdb bomb` 打印0x402400地址的字符串，就得到了我们需要的第一个答案：

```shell
(gdb )p (char* )0x402400
$1 = 0x402400 "Border relations with Canada have never been better."
# 或
(gdb) x/s 0x402400
0x402400:       "Border relations with Canada have never been better."
```

"Border relations with Canada have never been better."

## Bomb2

```txt
0000000000400efc <phase_2>:
  400efc:	55                   	push   %rbp
  400efd:	53                   	push   %rbx
  400efe:	48 83 ec 28          	sub    $0x28,%rsp
  400f02:	48 89 e6             	mov    %rsp,%rsi
  400f05:	e8 52 05 00 00       	call   40145c <read_six_numbers> # 第一个参数%rdi为我们的输入字符串，第二个参数%rsi为栈指针寄存器
  400f0a:	83 3c 24 01          	cmpl   $0x1,(%rsp)	# 判断栈顶是否为1
  400f0e:	74 20                	je     400f30 <phase_2+0x34>
  400f10:	e8 25 05 00 00       	call   40143a <explode_bomb>
  400f15:	eb 19                	jmp    400f30 <phase_2+0x34>
  400f17:	8b 43 fc             	mov    -0x4(%rbx),%eax	# 拿出栈顶的数
  400f1a:	01 c0                	add    %eax,%eax	# 将该数*2
  400f1c:	39 03                	cmp    %eax,(%rbx)	# 判断*2后是否和栈顶-0x4相等
  400f1e:	74 05                	je     400f25 <phase_2+0x29>
  400f20:	e8 15 05 00 00       	call   40143a <explode_bomb>
  400f25:	48 83 c3 04          	add    $0x4,%rbx	# %rbx-0x4，继续判断下一个数
  400f29:	48 39 eb             	cmp    %rbp,%rbx	# 判断是否达到要判断的栈底位置0x18(%rsp)
  400f2c:	75 e9                	jne    400f17 <phase_2+0x1b>
  400f2e:	eb 0c                	jmp    400f3c <phase_2+0x40>
  400f30:	48 8d 5c 24 04       	lea    0x4(%rsp),%rbx
  400f35:	48 8d 6c 24 18       	lea    0x18(%rsp),%rbp
  400f3a:	eb db                	jmp    400f17 <phase_2+0x1b>
  400f3c:	48 83 c4 28          	add    $0x28,%rsp
  400f40:	5b                   	pop    %rbx
  400f41:	5d                   	pop    %rbp
  400f42:	c3                   	ret
```

read_six_numbers函数会把我们传入的6个数字的字符串，转换成int型依次存放在栈中。比如传入的是`1 2 4 8 16 32`，那么栈上的数据为(%rsp)=1, 4(%rsp)=2, ...

可以一开始判断栈顶(字符串的第一个数字)是否为1，后面将1*2与第二个数相比...，因此6个数为`1 2 4 8 16 32`

## Bomb3

```txt
0000000000400f43 <phase_3>:
  400f43:	48 83 ec 18          	sub    $0x18,%rsp
  400f47:	48 8d 4c 24 0c       	lea    0xc(%rsp),%rcx	# %rsp+0xc变量地址传入%rcx，下面scanf函数第四个参数
  400f4c:	48 8d 54 24 08       	lea    0x8(%rsp),%rdx	# %rsp+0x8变量地址传入%rdx，下面scanf函数第三个参数
  400f51:	be cf 25 40 00       	mov    $0x4025cf,%esi	# "%d %d"
  400f56:	b8 00 00 00 00       	mov    $0x0,%eax
  400f5b:	e8 90 fc ff ff       	call   400bf0 <__isoc99_sscanf@plt>
  400f60:	83 f8 01             	cmp    $0x1,%eax	# scanf的return value
  400f63:	7f 05                	jg     400f6a <phase_3+0x27>	# scanf读到的input数量大于1，跳转
  400f65:	e8 d0 04 00 00       	call   40143a <explode_bomb>
  400f6a:	83 7c 24 08 07       	cmpl   $0x7,0x8(%rsp)	# 即我们传入字符串的第一个数字要小于7
  400f6f:	77 3c                	ja     400fad <phase_3+0x6a>	# 大于7引爆
  400f71:	8b 44 24 08          	mov    0x8(%rsp),%eax	# %eax=传入的第一个数字
  400f75:	ff 24 c5 70 24 40 00 	jmp    *0x402470(,%rax,8)	# jmp 0x402470+8*%rax
  400f7c:	b8 cf 00 00 00       	mov    $0xcf,%eax	# 0 207
  400f81:	eb 3b                	jmp    400fbe <phase_3+0x7b>
  400f83:	b8 c3 02 00 00       	mov    $0x2c3,%eax	# 2 707
  400f88:	eb 34                	jmp    400fbe <phase_3+0x7b>
  400f8a:	b8 00 01 00 00       	mov    $0x100,%eax	# 3 256
  400f8f:	eb 2d                	jmp    400fbe <phase_3+0x7b>
  400f91:	b8 85 01 00 00       	mov    $0x185,%eax	# 4 389
  400f96:	eb 26                	jmp    400fbe <phase_3+0x7b>
  400f98:	b8 ce 00 00 00       	mov    $0xce,%eax	# 5 206
  400f9d:	eb 1f                	jmp    400fbe <phase_3+0x7b>
  400f9f:	b8 aa 02 00 00       	mov    $0x2aa,%eax	# 6 682
  400fa4:	eb 18                	jmp    400fbe <phase_3+0x7b>
  400fa6:	b8 47 01 00 00       	mov    $0x147,%eax	# 7 327
  400fab:	eb 11                	jmp    400fbe <phase_3+0x7b>
  400fad:	e8 88 04 00 00       	call   40143a <explode_bomb>
  400fb2:	b8 00 00 00 00       	mov    $0x0,%eax
  400fb7:	eb 05                	jmp    400fbe <phase_3+0x7b>
  400fb9:	b8 37 01 00 00       	mov    $0x137,%eax	# 1 311
  400fbe:	3b 44 24 0c          	cmp    0xc(%rsp),%eax
  400fc2:	74 05                	je     400fc9 <phase_3+0x86>
  400fc4:	e8 71 04 00 00       	call   40143a <explode_bomb>
  400fc9:	48 83 c4 18          	add    $0x18,%rsp
  400fcd:	c3                   	ret
```

```shell
(gdb) b phase_3
(gdb) r
(gdb) p/x $rcx
$1 = 0x7fffffffdc9c
(gdb) p/x $rdx
$2 = 0x7fffffffdc98
```

%rcx和%rdx寄存器保存了栈上两个数据的地址，用来传给下面的scanf函数。


```shell
(gdb) x/s 0x4025cf
0x4025cf:       "%d %d"
```

打印`mov    $0x4025cf,%esi`0x4025cf地址的字符串出来，发现是"%d %d"。是scanf的第二个参数。第一个参数保存在`%rdi`是我们传入的字符串。

根据上面两条可以推断出scanf的调用为：`scanf("<传入的字符串>" "%d %d", &a, &b)`

`cmpl   $0x7,0x8(%rsp)`: 第一个传入的数字不可以大于7，否则会引爆炸弹。

`jmp    *0x402470(,%rax,8)`: 这句是关键，根据传入的第一个参数不同，跳转到0x402470+8*%rax地址内存中保存的执行地址。

```shell
(gdb) p $rax
$1 = 5
(gdb) x/x 0x402470+8*$rax
0x402498:	0x00400f98
```

即执行`jmp 0x400f98`。

接下来的事情就是判断传入的第二个参数是否和某个数相等。注意答案中要转换成十进制，因为scanf接受的format是%d。

答案有8组，放在了汇编代码注释中。

## Bomb4

```txt
000000000040100c <phase_4>:
  40100c:	48 83 ec 18          	sub    $0x18,%rsp
  401010:	48 8d 4c 24 0c       	lea    0xc(%rsp),%rcx	# 传入的第二个数字
  401015:	48 8d 54 24 08       	lea    0x8(%rsp),%rdx	# 传入的第一个数字
  40101a:	be cf 25 40 00       	mov    $0x4025cf,%esi	# "%d %d"
  40101f:	b8 00 00 00 00       	mov    $0x0,%eax
  401024:	e8 c7 fb ff ff       	call   400bf0 <__isoc99_sscanf@plt>
  401029:	83 f8 02             	cmp    $0x2,%eax
  40102c:	75 07                	jne    401035 <phase_4+0x29>
  40102e:	83 7c 24 08 0e       	cmpl   $0xe,0x8(%rsp)	# 第一个数字要小于等于0xe
  401033:	76 05                	jbe    40103a <phase_4+0x2e>
  401035:	e8 00 04 00 00       	call   40143a <explode_bomb>
  40103a:	ba 0e 00 00 00       	mov    $0xe,%edx	# func4 第三个参数，0xe
  40103f:	be 00 00 00 00       	mov    $0x0,%esi	# func4 第二个参数，0x0
  401044:	8b 7c 24 08          	mov    0x8(%rsp),%edi	# func4 第一个参数，我们传入的第一个参数
  401048:	e8 81 ff ff ff       	call   400fce <func4>
  40104d:	85 c0                	test   %eax,%eax	# 返回值不等于0引爆
  40104f:	75 07                	jne    401058 <phase_4+0x4c>
  401051:	83 7c 24 0c 00       	cmpl   $0x0,0xc(%rsp)	# 传入的第二个数字不等于0引爆
  401056:	74 05                	je     40105d <phase_4+0x51>
  401058:	e8 dd 03 00 00       	call   40143a <explode_bomb>
  40105d:	48 83 c4 18          	add    $0x18,%rsp
  401061:	c3                   	ret
```

前面和bomb3类似，通过scanf读入两个整型，可以看出我们要输入的也是两个数字。

根据`cmpl   $0xe,0x8(%rsp)`可以推断出第一个数字不能超过0xe。

根据`cmpl   $0x0,0xc(%rsp)`可以看出**第二个数字一定是0**。

根据`test   %eax,%eax`可以推断出func4的返回值为0才不会引爆炸弹。

剩下的就看在跳转到\<func4\>中对第一个数还有什么限制，传递给func4的参数分别为输入的第一个数，0x0, 0xe。

```txt
0000000000400fce <func4>:
  400fce:	48 83 ec 08          	sub    $0x8,%rsp
  400fd2:	89 d0                	mov    %edx,%eax	# %eax = 0xe
  400fd4:	29 f0                	sub    %esi,%eax	# %eax = 0xe - 0x0
  400fd6:	89 c1                	mov    %eax,%ecx	# %ecx = 0xe
  400fd8:	c1 e9 1f             	shr    $0x1f,%ecx	# 逻辑右移31位 %ecx = 0xe >> 0x1f
  400fdb:	01 c8                	add    %ecx,%eax	# %eax = 0xe
  400fdd:	d1 f8                	sar    %eax	# 右移一位。%eax = 0x7
  400fdf:	8d 0c 30             	lea    (%rax,%rsi,1),%ecx	# %ecx = %rax+%rsi = 0x7
  400fe2:	39 f9                	cmp    %edi,%ecx	# %ecx <= %edi 跳转到400ff2
  400fe4:	7e 0c                	jle    400ff2 <func4+0x24>
  400fe6:	8d 51 ff             	lea    -0x1(%rcx),%edx	# %edx = 0x6
  400fe9:	e8 e0 ff ff ff       	call   400fce <func4>
  400fee:	01 c0                	add    %eax,%eax
  400ff0:	eb 15                	jmp    401007 <func4+0x39>
  400ff2:	b8 00 00 00 00       	mov    $0x0,%eax
  400ff7:	39 f9                	cmp    %edi,%ecx	# %ecx >= %edi
  400ff9:	7d 0c                	jge    401007 <func4+0x39>
  400ffb:	8d 71 01             	lea    0x1(%rcx),%esi
  400ffe:	e8 cb ff ff ff       	call   400fce <func4>
  401003:	8d 44 00 01          	lea    0x1(%rax,%rax,1),%eax	# 到这里是错误了
  401007:	48 83 c4 08          	add    $0x8,%rsp
  40100b:	c3                   	ret
```

可以推断出这里的c code是一个递归函数。`%ecx`不断在减小，当小于等于我们传入的`%edi`时跳转到0x400ff2。

跳转到0x400ff2，执行到0x400ff7又要比较`cmp    %edi,%ecx`, `%ecx`又需要大于等于`%edi`才不会引爆炸弹。

结合上面两点，只有在迭代过程中`%ecx`和传入的第一个参数`%edi`相等才可以成功返回。

如果`%edi`为0，%ecx的值在迭代过程中变化为：

| %edx | %rax | %ecx | %rsi |
|------|------|------|------|
| 0xe  | 0x7  | 0x7  | 0x0  |
| 0x6  | 0x3  | 0x3  | 0x0  |
| 0x2  | 0x1  | 0x1  | 0x0  |
| 0x1  | 0x7  | 0x0  | 0x0  |

0x7->0x3->0x1->0x0. 如果`%edi`为0x1，则为0x7->0x3->0x1就返回。

因此最后的答案为`7 0`/`3 0`/`1 0`/`0 0`。

参考[https://www.viseator.com/2017/06/21/CS_APP_BombLab/](https://www.viseator.com/2017/06/21/CS_APP_BombLab/)，可以逆推出C code：

```c
int phase_4(int a1, int a2, int x){
    int b = (a1 - a2) >> 31;
    int result = ((a1-a2) + b) >> 1;
    b = result + a2;
    if(b == x) return 0;
    if(b < x) {
        result = fun(a1, b + 1, x);
        return result * 2 + 1;
    }else{
        result = fun(b - 1, a2, x);
        return result * 2;
    }
}

int main(void){
    for(int i = 0; i <= 0xe; i++){
        if(fun(0xe,0,i) == 0){
            printf("%d\n",i) ;
            return 0;
        }
    }
    return 0;
}
```

## Bomb5

```txt
0000000000401062 <phase_5>:
  401062:	53                   	push   %rbx
  401063:	48 83 ec 20          	sub    $0x20,%rsp
  401067:	48 89 fb             	mov    %rdi,%rbx	# 传入的字符串地址
  40106a:	64 48 8b 04 25 28 00 	mov    %fs:0x28,%rax
  401071:	00 00
  401073:	48 89 44 24 18       	mov    %rax,0x18(%rsp)
  401078:	31 c0                	xor    %eax,%eax	# 清零
  40107a:	e8 9c 02 00 00       	call   40131b <string_length>
  40107f:	83 f8 06             	cmp    $0x6,%eax	# string length必须是6
  401082:	74 4e                	je     4010d2 <phase_5+0x70>
  401084:	e8 b1 03 00 00       	call   40143a <explode_bomb>
  401089:	eb 47                	jmp    4010d2 <phase_5+0x70>
  40108b:	0f b6 0c 03          	movzbl (%rbx,%rax,1),%ecx	# 把字符串第%rax个字符的ascii码值传入%ecx
  40108f:	88 0c 24             	mov    %cl,(%rsp)
  401092:	48 8b 14 24          	mov    (%rsp),%rdx
  401096:	83 e2 0f             	and    $0xf,%edx	# ascii码值 & 0xf
  401099:	0f b6 92 b0 24 40 00 	movzbl 0x4024b0(%rdx),%edx	# 0x4024b0+%rdx地址的值传入%edx
  4010a0:	88 54 04 10          	mov    %dl,0x10(%rsp,%rax,1)	# 存入栈中
  4010a4:	48 83 c0 01          	add    $0x1,%rax
  4010a8:	48 83 f8 06          	cmp    $0x6,%rax
  4010ac:	75 dd                	jne    40108b <phase_5+0x29>	# %rax+=1, %rax<6
  4010ae:	c6 44 24 16 00       	movb   $0x0,0x16(%rsp)
  4010b3:	be 5e 24 40 00       	mov    $0x40245e,%esi	# "flyers"
  4010b8:	48 8d 7c 24 10       	lea    0x10(%rsp),%rdi	# 栈上的字符串
  4010bd:	e8 76 02 00 00       	call   401338 <strings_not_equal>
  4010c2:	85 c0                	test   %eax,%eax
  4010c4:	74 13                	je     4010d9 <phase_5+0x77>
  4010c6:	e8 6f 03 00 00       	call   40143a <explode_bomb>
  4010cb:	0f 1f 44 00 00       	nopl   0x0(%rax,%rax,1)
  4010d0:	eb 07                	jmp    4010d9 <phase_5+0x77>
  4010d2:	b8 00 00 00 00       	mov    $0x0,%eax
  4010d7:	eb b2                	jmp    40108b <phase_5+0x29>
  4010d9:	48 8b 44 24 18       	mov    0x18(%rsp),%rax
  4010de:	64 48 33 04 25 28 00 	xor    %fs:0x28,%rax
  4010e5:	00 00
  4010e7:	74 05                	je     4010ee <phase_5+0x8c>
  4010e9:	e8 42 fa ff ff       	call   400b30 <__stack_chk_fail@plt>
  4010ee:	48 83 c4 20          	add    $0x20,%rsp
  4010f2:	5b                   	pop    %rbx
  4010f3:	c3                   	ret
```

`cmp    $0x6,%eax`: 推断出传入的是一个6字节长度的字符串。

`and    $0xf,%edx`: 接着把字符串每个字节的ascii码值 & 0xf

`movzbl 0x4024b0(%rdx),%edx`: 根据传入字符串的ascii码&0xf, 找到对应的不同的地址`0x4024b0+%rdx`，这些地址中保存的值会组成最终我们需要对比的字符串。

`mov    %dl,0x10(%rsp,%rax,1)`: 将字符按顺序存入栈中。

`mov    $0x40245e,%esi`: 打印出0x40245e保存的字符串，是"flyers"。

`call   401338 <strings_not_equal>` 将栈上的字符串和"flyers"比较。

"flyers"对应的ascii码为：0x66 0x6c 0x79 0x65 0x72 0x73

根据`0x4024b0+%rdx`，`%rdx`的范围是0~0xf，分别找到存储这些值的对应地址：0x4024b9 0x4024bf 0x4024be 0x4024b5 0x4024b6 0x4024b7。

因此需要传入的asicii低位：0x9 0xf 0xe 0x5 0x6 0x7。

因此我们可以传入的字符串有很多组，挑一组：  
`0x69 0x6f 0x6e 0x65 0x66 0x67`, `ionefg`

## Bomb6

```txt
00000000004010f4 <phase_6>:
  4010f4:	41 56                	push   %r14
  4010f6:	41 55                	push   %r13
  4010f8:	41 54                	push   %r12
  4010fa:	55                   	push   %rbp
  4010fb:	53                   	push   %rbx
  4010fc:	48 83 ec 50          	sub    $0x50,%rsp
  401100:	49 89 e5             	mov    %rsp,%r13
  401103:	48 89 e6             	mov    %rsp,%rsi	# 栈指针地址传入%rsi
  401106:	e8 51 03 00 00       	call   40145c <read_six_numbers>
  40110b:	49 89 e6             	mov    %rsp,%r14
  40110e:	41 bc 00 00 00 00    	mov    $0x0,%r12d
  401114:	4c 89 ed             	mov    %r13,%rbp	# 和401151地址指令组成循环，%r13为栈地址，for i=1;i++;i<6
  401117:	41 8b 45 00          	mov    0x0(%r13),%eax
  40111b:	83 e8 01             	sub    $0x1,%eax
  40111e:	83 f8 05             	cmp    $0x5,%eax	# 减1后要≤5，因此每个数字都要≤6
  401121:	76 05                	jbe    401128 <phase_6+0x34>
  401123:	e8 12 03 00 00       	call   40143a <explode_bomb>
  401128:	41 83 c4 01          	add    $0x1,%r12d	# i=1
  40112c:	41 83 fc 06          	cmp    $0x6,%r12d	# for i=1;i++;i<6
  401130:	74 21                	je     401153 <phase_6+0x5f>
  401132:	44 89 e3             	mov    %r12d,%ebx	# %ebx从第二个数循环到最后一个数
  401135:	48 63 c3             	movslq %ebx,%rax	# 和40114b构成第二层循环
  401138:	8b 04 84             	mov    (%rsp,%rax,4),%eax	# %rax从1开始，%rsp+4，因此这边是我们传入的第二个数
  40113b:	39 45 00             	cmp    %eax,0x0(%rbp)	# 后面的数都不能和第一层循环的第1个数相同，否则引爆
  40113e:	75 05                	jne    401145 <phase_6+0x51>
  401140:	e8 f5 02 00 00       	call   40143a <explode_bomb>
  401145:	83 c3 01             	add    $0x1,%ebx
  401148:	83 fb 05             	cmp    $0x5,%ebx
  40114b:	7e e8                	jle    401135 <phase_6+0x41>
  40114d:	49 83 c5 04          	add    $0x4,%r13
  401151:	eb c1                	jmp    401114 <phase_6+0x20>
  401153:	48 8d 74 24 18       	lea    0x18(%rsp),%rsi	# 最后一个数的地址，循环结束条件
  401158:	4c 89 f0             	mov    %r14,%rax	# 第一个数地址保存进%rax
  40115b:	b9 07 00 00 00       	mov    $0x7,%ecx
  401160:	89 ca                	mov    %ecx,%edx	# 和40116d构成循环，遍历每个数
  401162:	2b 10                	sub    (%rax),%edx	# 7减去每一个数
  401164:	89 10                	mov    %edx,(%rax)	# 将该值替换进第一个数的地址
  401166:	48 83 c0 04          	add    $0x4,%rax
  40116a:	48 39 f0             	cmp    %rsi,%rax
  40116d:	75 f1                	jne    401160 <phase_6+0x6c>
  40116f:	be 00 00 00 00       	mov    $0x0,%esi
  401174:	eb 21                	jmp    401197 <phase_6+0xa3>
  401176:	48 8b 52 08          	mov    0x8(%rdx),%rdx	# 0x6032d8地址的值(0x6032e0)->0x6032e8的值(0x6032f0)->...
  40117a:	83 c0 01             	add    $0x1,%eax
  40117d:	39 c8                	cmp    %ecx,%eax
  40117f:	75 f5                	jne    401176 <phase_6+0x82>
  401181:	eb 05                	jmp    401188 <phase_6+0x94>
  401183:	ba d0 32 60 00       	mov    $0x6032d0,%edx
  401188:	48 89 54 74 20       	mov    %rdx,0x20(%rsp,%rsi,2)
  40118d:	48 83 c6 04          	add    $0x4,%rsi
  401191:	48 83 fe 18          	cmp    $0x18,%rsi
  401195:	74 14                	je     4011ab <phase_6+0xb7>
  401197:	8b 0c 34             	mov    (%rsp,%rsi,1),%ecx	# 经过上面处理(被7减了)的第一个数
  40119a:	83 f9 01             	cmp    $0x1,%ecx
  40119d:	7e e4                	jle    401183 <phase_6+0x8f>	# ≤1的跳转
  40119f:	b8 01 00 00 00       	mov    $0x1,%eax
  4011a4:	ba d0 32 60 00       	mov    $0x6032d0,%edx
  4011a9:	eb cb                	jmp    401176 <phase_6+0x82>
  4011ab:	48 8b 5c 24 20       	mov    0x20(%rsp),%rbx
  4011b0:	48 8d 44 24 28       	lea    0x28(%rsp),%rax
  4011b5:	48 8d 74 24 50       	lea    0x50(%rsp),%rsi
  4011ba:	48 89 d9             	mov    %rbx,%rcx
  4011bd:	48 8b 10             	mov    (%rax),%rdx
  4011c0:	48 89 51 08          	mov    %rdx,0x8(%rcx)
  4011c4:	48 83 c0 08          	add    $0x8,%rax
  4011c8:	48 39 f0             	cmp    %rsi,%rax
  4011cb:	74 05                	je     4011d2 <phase_6+0xde>
  4011cd:	48 89 d1             	mov    %rdx,%rcx
  4011d0:	eb eb                	jmp    4011bd <phase_6+0xc9>
  4011d2:	48 c7 42 08 00 00 00 	movq   $0x0,0x8(%rdx)
  4011d9:	00
  4011da:	bd 05 00 00 00       	mov    $0x5,%ebp
  4011df:	48 8b 43 08          	mov    0x8(%rbx),%rax
  4011e3:	8b 00                	mov    (%rax),%eax
  4011e5:	39 03                	cmp    %eax,(%rbx)
  4011e7:	7d 05                	jge    4011ee <phase_6+0xfa>
  4011e9:	e8 4c 02 00 00       	call   40143a <explode_bomb>
  4011ee:	48 8b 5b 08          	mov    0x8(%rbx),%rbx
  4011f2:	83 ed 01             	sub    $0x1,%ebp
  4011f5:	75 e8                	jne    4011df <phase_6+0xeb>
  4011f7:	48 83 c4 50          	add    $0x50,%rsp
  4011fb:	5b                   	pop    %rbx
  4011fc:	5d                   	pop    %rbp
  4011fd:	41 5c                	pop    %r12
  4011ff:	41 5d                	pop    %r13
  401201:	41 5e                	pop    %r14
  401203:	c3                   	ret
```

read_six_numbers把我们传入的六个数的字符串，转化成int保存在栈中%rsp, %rsp+4, %rsp+8...

## Secret Bomb
