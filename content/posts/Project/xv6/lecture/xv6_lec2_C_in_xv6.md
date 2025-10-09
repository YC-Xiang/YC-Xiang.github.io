---
date: 2024-12-26T21:09:33+08:00
title: "xv6_lecture2 C in xv6"
tags:
  - CMake
categories:
  - Book
---

经典的一道指针和数组题：

假设 x 的地址为 0x0, 打印出来的值分别是多少？

```c++
#include <stdio.h>
int main() {
 int x[5];
 printf("%p\n", x);
 printf("%p\n", x+1);
 printf("%p\n", &x)
 printf("%p\n", &x+1);
 return 0;
}
```

```shell
0x0 # 打印的数组的地址
0x4 # 打印的是数组第二个元素的地址
0x0 # &x也是数组的地址
0x14 # x + sizeof(x)的地址
```
