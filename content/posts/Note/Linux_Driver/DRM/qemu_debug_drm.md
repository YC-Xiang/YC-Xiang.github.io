---
date: 2024-09-26T10:02:27+08:00
title: Debugging DRM driver in QEMU with Buildroot support
tags:
  - DRM
categories:
  - DRM
---

# Buildroot 编译 kernel 和 rootfs

进入 buildroot 目录，创建一份自定义的 config，因为要在 qemu x86_64 架构上跑，因此复制一份 qemu_x86_64_defconfig，在此基础上进行修改：

```sh
cd buildroot-2024.02.6/
cp configs/qemu_x86_64_defconfig configs/custom_qemu_x86_64_defconfig
```

</br>

该**config 文件**的改动主要是从使用 buildroot 自带的 linux 源码替换为我们自己 download 下来最新的 linux-6.10 源码。

```txt
BR2_PACKAGE_OVERRIDE_FILE="board/qemu/x86_64/custom_override.mk"
BR2_LINUX_KERNEL_CUSTOM_VERSION=y
BR2_LINUX_KERNEL_CUSTOM_VERSION_VALUE="linux-6.10"
BR2_LINUX_KERNEL_CUSTOM_CONFIG_FILE="board/qemu/x86_64/custom_linux.config"
```

</br>

创建 config 文件中指定的 board/qemu/x86_64/**custom_override.mk**，来指明 linux 源码路径，其中\$(TOPDIR)表示 buildroot 根目录，\$(BR2_LINUX_KERNEL_VERSION)是上面指定的 linux-6.10。

```makefile
LINUX_OVERRIDE_SRCDIR=$(TOPDIR)/../$(BR2_LINUX_KERNEL_VERSION)
```

</br>

创建 config 文件中指定的 linux config 文件**custom_linux.config**。这边也是从/board/qemu/x86_64/linux.config 拷贝过来。

注意这边 CONFIG_PCI 需要打开，否则后面挂载根文件会失败。

CONFIG_DEBUG_INFO 也需要打开，为了后面 gdb 调试 kernel。

</br>

最后在 buildroot 根目录下执行`make`编译，编译完成后执行`qemu-system-x86_64 -nographic -kernel bzImage -hda rootfs.ext2 -append "console=ttyS0 root=/dev/sda"`

# QEMU+GDB 调试内核
