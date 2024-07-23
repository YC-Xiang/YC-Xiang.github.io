---
title: Operating Systems Three Easy Pieces(OSTEP) - Memory Virtualization
date: 2024-03-27 9:30:28
tags:
- OSTEP
categories:
- Book
---

# Chapter14 Memory API

## 14.2 The malloc() call

分配字符串长度的时候要注意：`malloc(strlen(s) + 1)`，多分配一个byte给`\0`。strlen函数计算长度不会统计`\0`。

## 14.4 Common Errors

### Not Allocating Enough Memory

```c
char *src = "hello";
char *dst = (char *) malloc(strlen(src)); // wrong!!!
char *dst = (char *) malloc(strlen(src) + 1);
strcpy(dst, src); // work properly
```

strcpy拷贝的时候会把`\0`也拷贝过去，所以分配长度的时候要strlen()+1。  
否则多拷贝的一个byte，会写到heap的其他变量所属的地址，可能会导致fault。

### Forgetting to Initialize Allocated Memory

malloc分配出来的内存，需要初始化，否则可能是随机内存值。

### Forgetting To Free Memory

malloc分配出来的内存需要free掉。不然长时间运行的进程会导致内存泄漏。

短时间运行就结束的进程可能不会有这个问题，在进程结束后系统会自动回收内存，但free掉不使用的内存仍然是一个好习惯。

### Freeing Memory Before You Are Done With It

### Freeing Memory Repeatedly

# Chapter15 Address Translation 地址转换

# Chapter16 Segmentation 分段

## 16.2 ~ 16.3

在base and bound的基础上改进，将每个段(code, stack, heap)分配一个base和bound寄存器，而不是之前每个进程分配一个base和bound寄存器的方式。

**Explict way:**

进程虚拟地址的高几位用来区分处于哪个Segment，低位用来确定Segment中的offset。

![](https://xyc-1316422823.cos.ap-shanghai.myqcloud.com/20240329170651.png)

伪代码：

```c
// get top 2 bits of 14-bit VA
Segment = (VirtualAddress & SEG_MASK) >> SEG_SHIFT
// now get offset
Offset = VirtualAddress & OFFSET_MASK
if (Offset >= Bounds[Segment])
RaiseException(PROTECTION_FAULT)
else
PhysAddr = Base[Segment] + Offset
Register = AccessMemory(PhysAddr)
```

stack因为是逆生长的，计算方法不一样，为stack addr - (2^12 - offset)。硬件需要用一个grow bit来区分不同segment地址生长方向。

**Implicit way:**

Hardware自动识别地址处于哪个Segment，比如地址来自于PC指针，一定是code segment; 如果地址基于stack pointer，那就是stack segment。

## 16.4 Support for sharing

为了节省内存，增加一个protection的bit，如果是read-only的segment，所有进程都可以访问，也不用担心被破坏。

![](https://xyc-1316422823.cos.ap-shanghai.myqcloud.com/20240329172922.png)

# Chapter17 Free-space Management 空闲空间管理

调用free()函数，只是传入了指针，怎么知道要释放空间的大小呢？

在调用malloc的时候，返回ptr指针指向的地址，在ptr指针前面还会分配一个header包括size和magic number，这样在free释放的时候就知道了释放的大小，将header和分配的内存一起释放。

![](https://xyc-1316422823.cos.ap-shanghai.myqcloud.com/20240424202157.png)
