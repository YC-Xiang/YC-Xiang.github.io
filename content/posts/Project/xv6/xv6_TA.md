---
title: xv6 calling conventions and stack frames RISC-V
date: 2024-02-25
tags:
- xv6 OS
categories:
- Project
---

![](https://xyc-1316422823.cos.ap-shanghai.myqcloud.com/20240225201553.png)

caller: not preserved across fn call. 需要调用函数来保存寄存器。参考下面例子中的 ra 寄存器值。  
callee: preserved across fn call. 被调用函数来保存寄存器。
