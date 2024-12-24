---
title: xv6 Misc
date: 2024-02-01
tags:
  - xv6 OS
categories:
  - Project
---

# 环境安装

https://pdos.csail.mit.edu/6.828/2023/tools.html

```shell
$ sudo apt-get update && sudo apt-get upgrade
sudo apt-get install git build-essential gdb-multiarch qemu-system-misc gcc-riscv64-linux-gnu binutils-riscv64-linux-gnu
```

测试环境:

```shell
$ riscv64-unknown-elf-gcc --version
$ qemu-system-riscv64 --version
```

在 ubuntu22.04 上的输出 log 为:

```shell
~ ❯ qemu-system-riscv64 --version
QEMU emulator version 6.2.0 (Debian 1:6.2+dfsg-2ubuntu6.24)
Copyright (c) 2003-2021 Fabrice Bellard and the QEMU Project developers

~ ❯ riscv64-linux-gnu-gcc --version
riscv64-linux-gnu-gcc (Ubuntu 11.4.0-1ubuntu1~22.04) 11.4.0
Copyright (C) 2021 Free Software Foundation, Inc.
This is free software; see the source for copying conditions.  There is NO
warranty; not even for MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.
```

最后进入 xv6 目录后,执行`make qemu`观察能否正确进入 qemu.

# 下载 xv6 源码

```shell
git clone git://g.csail.mit.edu/xv6-labs-2023
```

修改 git 远程仓库为自己的 github 仓库：

```shell
git remote add origin https://github.com/YC-Xiang/xv6-2023.git
git push -u origin util
```

# xv6 运行

`make qemu`  
`make clean`

`Ctrl-p` 打印进程。  
`Ctrl-a x` 退出 qemu。

`make grade` 检查所有 lab 得分。  
`./grade-lab-util sleep` or `make GRADEFLAGS=sleep grade` 检查某一项作业得分。

## GDB 调试

`sudo aptitude install gdb-multiarch`: 解决 ubuntu22.04 gdb 版本不匹配的问题。

一个 shell 运行`make qemu-gdb`， 另一个 shell 运行`gdb-multiarch`，会自动加载`.gdbinit`。如果出现如下错误：

```shell
warning: File "/home/xyc/MIT-6.S081-Operation-System/.gdbinit" auto-loading has been declined by your `auto-load safe-path' set to "$debugdir:$datadir/auto-load".
To enable execution of this file add
        add-auto-load-safe-path /home/xyc/MIT-6.S081-Operation-System/.gdbinit
line to your configuration file "/home/xyc/.config/gdb/gdbinit".
```

按照提示操作，将`add-auto-load-safe-path /home/xyc/MIT-6.S081-Operation-System/.gdbinit`加到`/home/xyc/.config/gdb/gdbinit`即可。
