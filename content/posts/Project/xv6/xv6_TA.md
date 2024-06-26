---
title: xv6 calling conventions and stack frames RISC-V
date: 2024-02-25
tags:
- xv6 OS
categories:
- Project
---

![](https://xyc-1316422823.cos.ap-shanghai.myqcloud.com/20240225201553.png)

caller: not preserved across fn call. 需要调用函数来保存寄存器。参考下面例子中的ra寄存器值。
callee: preserved across fn call. 被调用函数来保存寄存器。

</br>

![](https://xyc-1316422823.cos.ap-shanghai.myqcloud.com/20240225223002.png)

<p class="note note-info">图中frame pointer应该指向顶部，return address+8的地址，应该往上移一点，而不是指向保存return address的地址。</p>

```c
sum_to:
	mv t0, a0
	li a0, 0
    loop:
	add a0, a0, t0
	addi t0, t0, -1
	bnez t0, loop
	ret // 返回到li t0, 2

sum_then_double:
	addi sp, sp, -16; // 分配栈空间
	sd ra, 0(sp) // ra的值存入sp+0地址。caller保存sum_the_double执行完的返回地址
	call sum_to // 这里调用call，ra的值被设置为下一条指令地址，即li t0, 2
	li t0, 2
	mul a0, a0, t0
	ld ra, 0(sp) // 恢复sum_them_double的返回地址
	addi sp, sp, 16
	ret
```
