---
title: Keil相关内存设置
date: 2024-06-21 09:51:28
tags:
- Misc
categories:
- Misc
---

# 内存设置

![](https://xyc-1316422823.cos.ap-shanghai.myqcloud.com/20240621095347.png)

off-chip和on-chip本质上没有区别，只是地址会不同。  
假如ic都是on-chip sram，要分成三块的话，把一块写到off-chip就可以。

NoInit不勾选的话，每次进main函数前会对该memory进行清0的操作。

前面default是默认的变量保存地址。

# 修改变量地址

可以直接在keil左侧对文件右击，选择第一个option for file，对该文件中的变量地址进行修改。

![](https://xyc-1316422823.cos.ap-shanghai.myqcloud.com/20240621171205.png)

Code/Const会保存到default的ROM中。

Zero initialized Data是bss段数据(未初始化的和初始化为0的全局变量，函数中的static变量)。

Other data是data段数据(初始化为非0的全局变量，函数中的static变量)。

对启动文件中的zero initialized data修改位置，将会更改栈和堆的地址。

</br>

还可以直接在code中修改某个变量的地址：

```c++
sdcp_keypair g_tFWKey __attribute__((section(".ARM.__at_0x20000000")));
```
