---
date: 2024-09-26T10:02:27+08:00
title: Debug DRM driver in QEMU with Buildroot support
tags:
  - DRM
categories:
  - DRM
---

# Buildroot 编译 kernel 和 rootfs

我们想在 QEMU 中进行 debug，那么首先需要准备好 kernel image 和 rootfs。这边我们采用的是 buildroot 的方式来创建我们需要的 kernel 和 rootfs。

下载 buildroot 2024:

```sh
wget https://buildroot.org/downloads/buildroot-2024.02.6.tar.gz
```

</br>

接着进入 buildroot 目录，执行 make qemu_x86_64_defconfig，使用 qemu 的配置：

```sh
cd buildroot-2024.02.6/
make qemu_x86_64_defconfig
```

</br>

对该 config 文件作如下改动：

```txt
# support custom linux source code
BR2_PACKAGE_OVERRIDE_FILE="board/qemu/x86_64/custom_override.mk"
BR2_LINUX_KERNEL_CUSTOM_VERSION=y
BR2_LINUX_KERNEL_CUSTOM_VERSION_VALUE="linux-6.10"
BR2_LINUX_KERNEL_CUSTOM_CONFIG_FILE="board/qemu/x86_64/custom_linux.config"

# GDB support
BR2_PACKAGE_HOST_GDB=y # cross gdb for host machine
BR2_PACKAGE_GDB=y
BR2_PACKAGE_GDB_SERVER=y

BR2_TOOLCHAIN_BUILDROOT_CXX=y
BR2_DEBUG_3=y
BR2_ENABLE_DEBUG=y # enable debug symbol in packages
BR2_OPTIMIZE_0=y

BR2_PACKAGE_LIBDRM=y
BR2_PACKAGE_DRMTEST=y # custom drm test package
```

改动目的主要是：

1. 使用我们自己 download 的 linux 源码，linux-6.10 是我们外部 linux 源码的目录名。
2. 开启 GDB 支持。
3. 编译 libdrm 和 drmtest，为了后面调试 drm driver。

</br>

接着创建 config 文件中指定的 board/qemu/x86_64/**custom_override.mk**，来指明 linux 源码路径，其中\$(TOPDIR)表示 buildroot 根目录，\$(BR2_LINUX_KERNEL_VERSION)是上面指定的 linux-6.10:

```makefile
LINUX_OVERRIDE_SRCDIR=$(TOPDIR)/../$(BR2_LINUX_KERNEL_VERSION)
```

</br>

接着再创建 config 文件中指定的 linux config 文件**custom_linux.config**。这边我们需要对 linux config 做一些修改，因此拷贝/board/qemu/x86_64/linux.config 到 custom_linux.config，在此基础上修改,新增一些 config:

```sh
CONFIG_DEBUG_INFO_DWARF5=y
CONFIG_DEBUG_INFO=y
CONFIG_DEBUG_FS=y
```

主要是为了编译 kernel 加入 debug symbol。

</br>

改动都完成后，在 buildroot 根目录执行`make qemu_x86_64_defconfig`

最后执行`make`编译，编译完成后，在`output/images/`目录下会生成 bzImage, rootfs.ext2, start-qemu.sh。分别对应 kernel image，rootfs，qemu 启动脚本。

# 创建 drmtest package

在 buildroot 中增加一个 drmtest package。主要是为了在最终生成的 rootfs 中加入 cross compiled 的 drmtest 测试程序。

这一部分参考 buildroot user manual 以及 reference 中 patch 代码。

# QEMU+GDB 调试内核

在 start-qemu.sh 脚本的-append 中加上 `nokaslr` 防止 kernel 地址随机化，以及最后加上`-s` 选项启动 gdbserver。

```sh
exec qemu-system-x86_64 -M pc -kernel bzImage -drive file=rootfs.ext2,if=virtio,format=raw -append "rootwait nokaslr root=/dev/vda console=tty1 console=ttyS0"  -net nic,model=virtio -net user ${EXTRA_ARGS} "$@" -s
```

</br>

使用 system mode 启动 qemu, 这样能调用 QEMU GUI:

```sh
./start-qemu.sh --use-system-mode
```

</br>

host machine 启动 gdb 并连接 target remote，打上我们需要的断点:

```sh
$ <buildroot>/output/host/bin/<tuple>-gdb -ix <buildroot>/output/staging/usr/share/buildroot/gdbinit vmlinux
(gdb) b drm_mode_atomic_ioctl
(gdb) target remote :1234
Remote debugging using :1234
(gdb) c
Continuing.
```

</br>

在 qemu 中启动 drm 测试程序：

```sh
$ cd /
$ ./bin/drmtest
```

</br>

这样就能触发我们的断点，可以愉快地开始调试了：

```sh
Breakpoint 1, drm_mode_atomic_ioctl (dev=0xffff8880026c7000, data=0xffffc900001b7dd8, file_priv=0xffff888002ecb800)
    at drivers/gpu/drm/drm_atomic_uapi.c:1370
1370    {
```

# QEMU+GDB 调试应用程序

如果我们想调试应用程序 drmtest，看应用程序的流程该如何做呢？

去掉之前 qemu 启动程序中的`-s`选项，加入`-net user,hostfwd=tcp::1234-:1234`

</br>

启动 qemu, 并在 qemu 中启动 gdbserver：

```sh
$ ./start-qemu.sh --use-system-mode
$ gdbserver :1234 ./bin/drmtest
```

</br>

host 启动 gdb，在 main 打上断点，continue 后即可调试：

```sh
<buildroot>/output/host/bin/<tuple>-gdb -ix <buildroot>/output/staging/usr/share/buildroot/gdbinit drmtest
(gdb) b main
(gdb) c
Continuing.
Breakpoint 1, main (argc=1, argv=0x7fffc71a9038) at main.c:1223
```

<p class="note note-info">在应用程序中调试syscall时，发现通过step无法进入kernel code，似乎无法同时支持调试user code和kernel code。不知道是否有什么方法可以同时调试。Stackoverflow上有相同的问题：https://stackoverflow.com/questions/26271901/is-it-possible-to-use-gdb-and-qemu-to-debug-linux-user-space-programs-and-kernel</p>

# VSCode 调试内核

我们可以通过 vscode 来更方便地调试。

首先需要下载扩展 Native Debug 来支持 cross gdb debug。

接着配置 launch.json:

```json
{
  "version": "0.2.0",
  "configurations": [
    {
      "name": "(gdb) Launch",
      "type": "gdb",
      "request": "attach",
      "executable": "/home/yucheng_xiang/buildroot-2024.02.6/output/build/linux-custom/vmlinux",
      "target": "localhost:1234",
      "remote": true,
      "cwd": "${workspaceRoot}",
      "gdbpath": "/home/yucheng_xiang/buildroot-2024.02.6/output/host/bin/x86_64-buildroot-linux-gnu-gdb"
    }
  ]
}
```

`type`: gdb, 表示使用 Native Debug 配置  
`request`: 使用 attach  
`executable`: 需要调试的 kernel image 路径  
`target`: 即 target remote 后面的 X.X.X.X: \<port\>
`remote`: true, 表示远程调试
`gdbpath`: cross gdb 路径

</br>

QEMU 启动文件中加入`-s`参数来支持 gdb 调试, 启动 qemu:

```sh
./start-qemu.sh --use-system-mode
```

</br>

VSCode 按 F5 启动调试，因为这个时候 kernel 是在 run 的，没法打断点，所以我们输入 interupt 让 kernel 先停下来，再打上我们需要的断点：

```sh
(gdb) interrupt
(gdb) b drm_mode_atomic_ioctl
(gdb) c
```

</br>

qemu 启动测试程序后就会进入我们的断点：

```sh
./drmtest
```

</br>

![](https://xyc-1316422823.cos.ap-shanghai.myqcloud.com/20240929172340.png)

接着就可以在 VSCode 中愉快地调试了。

# Reference

Buildroot 官方文档：  
https://buildroot.org/downloads/manual/manual.html

VSCode target remote 调试：  
https://stackoverflow.com/questions/38089178/is-it-possible-to-attach-to-a-remote-gdb-target-with-vscode

[My Implement Patch](https://xyc-1316422823.cos.ap-shanghai.myqcloud.com/0001-add-drmtest-config-and-source-code.patch)
