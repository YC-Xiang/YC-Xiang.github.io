---
title: CSAPP - Cache Lab
date: 2024-08-05
tags:
- CSAPP
categories:
- Project
---

# 实验说明

在cache lab实验中，包含两个部分：

- 第一部分要求我们写一个C program来模拟cache memory。
- 第二部分要求我们通过减少cache miss来优化一个矩阵转置函数。

对应的文件分别为`csim.c`, `trans.c`

可通过`make`和`make clean`来编译和删除编译文件。

# PartA

我们利用`valgrind`生成的trace files作为我们cache simulator的input。

格式为`[space]operation address, size`:

```txt
I 0400d7d4,8
 M 0421c7f0,4
 L 04f6b868,8
 S 7ff0005c8,8
```

- I 加载指令
- L 加载数据
- S 存储数据
- M 内存数据修改(先加载数据，再存储数据)

I前面不跟空格，L/S/M前面需要加一个空格。

</br>

`csim-ref`是一个参考的cache simulator实现。使用`LRU`替换算法。

用法: `Usage: ./csim-ref [-hv] -s <s> -E <E> -b <b> -t <tracefile>`

- `-h`: Optional help flag that prints usage info
- `-v`: Optional verbose flag that displays trace info
- `-s <s>`: Number of set index bits (S = 2s is the number of sets)
- `-E <E>`: Associativity (number of lines per set)
- `-b <b>`: Number of block bits (B = 2b is the block size)
- `-t <tracefile>`: Name of the valgrind trace to replay

e.g.

```shell
./csim-ref -s 4 -E 1 -b 4 -t traces/yi.trace
hits:4 misses:5 evictions:3
```

```shell
./csim-ref -v -s 4 -E 1 -b 4 -t traces/yi.trace
L 10,1 miss
M 20,1 miss hit
L 22,1 hit
S 18,1 hit
L 110,1 miss eviction
L 210,1 miss eviction
M 12,1 miss eviction hit
hits:4 misses:5 evictions:3
```
