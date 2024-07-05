---
title: CSAPP - Attack Lab
date: 2024-07-05
tags:
- CSAPP
categories:
- Project
---

# Setup

`ctarget`: An executable program vulnerable to code-injection(CI) attacks.
`rtarget`: An executable program vulnerable to return-oriented-programming(ROP) attacks

输入`ctarget -h`查看帮助信息：

```shell
Usage: [-hq] ./ctarget -i <infile>
  -h          Print help information
  -q          Don't submit result to server
  -i <infile> Input file
```

我们在本地跑的话必须加上`-q`选项，否则会提示`Running on an illegal host`错误。

`hex2raw` 可以将文本中保存的ascii码转化成字符串。注意每个ascii需要以空格分开，可以加注释(注释/* */前后需要空格)。

比如在`exploit.txt`中保存`30 31 32 33 34 35 /* 012345 */`，通过`cat exploit.txt | ./hex2raw/`，可以转换成字符串`012345`
