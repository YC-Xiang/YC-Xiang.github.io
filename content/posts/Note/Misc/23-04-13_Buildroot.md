---
title: Buildroot
date: 2023-04-13 17:39:28
tags:
  - Buildroot
categories:
  - Notes
---

## Config.in 语法

用 Kconfig 语言编写，用来配置 packages

必须以`BR2_PACKAGE_<PACKAGE>`开头

![](https://tva1.sinaimg.cn/large/008i3skNgy1gxwbwcmsauj30i303zt8t.jpg)

Config.in 是层级结构`package/<pkg>/Config.in`都被包含在`package/Config.in`

### menu/endmenu

menuconfig 中层级目录由`menu`来嵌套定义

```kbuild
menu "Base System"
source "$BR2_EXTERNAL_platform_PATH/package/example/Config.in"
source "$BR2_EXTERNAL_platform_PATH/package/fstools/Config.in"
endmenu

menu "Test Package"
source "$BR2_EXTERNAL_platform_PATH/package/foobar/Config.in"
endmenu

// Test Package在Base System下一级目录
menu "Base System"
menu "Test Package"
endmenu
endmenu
```

### if/endif

### choice/endchoice

### select、depends on

select 是一种自动依赖，如果 A select B，只要 A 被 enable，B 就会被 enable，而且不可 unselected

depends on 是一种用户定义的依赖，如果 A depends on B, A 只有在 B 被 enable 后才可见

- `make \<pkg\>-show-depend`: 查看 pkg 依赖的包
- `make \<pkg\>-show-rdepend`: 查看依赖 pkg 的包

## .mk 文件

```
xxx_SITE_METHOD = local
xxx_SITE = 本地源码库地址

xxx_SITE_METHOD = remote
xxx_SITE = 远程URL
```

Packages 可以被安装到不同目录：

- target 目录：`$(TARGET_DIR)`
- staging 目录：`$(STAGING_DIR)`
- images 目录：`$(BINARIES_DIR)`

分别由三个变量决定：

- `<pkg>_INSTALL_TARGET` , defaults to `YES`. If `YES`, then `<pkg>_INSTALL_TARGET_CMDS` will be called
- `<pkg>_INSTALL_STAGING` , defaults to `NO`. If `YES`, then `<pkg>_INSTALL_STAGING_CMDS` will be called
- `<pkg>_INSTALL_IMAGES` , defaults to `NO`. If `YES`, then `<pkg>_INSTALL_IMAGES_CMDS` will be called <br/><br/>
- Application Package 一般只要安装到 target
- Shared library 动态库必须安装到 target 与 staging
- header-based library 和 static-only library 静态库只安装到 staging
- bootloader 和 linux 要安装到 images

Config.in 文件不规定编译顺序，.mk 文件中的\<pkg\>\_DEPENDENCIES 可以规定编译顺序，\<pkg\>\_DEPENDENCIES 后面的软件包先编译。

## 参考

- [https://www.cnblogs.com/fuzidage/p/12049442.html](https://www.cnblogs.com/fuzidage/p/12049442.html)

# Buildroot User Manual

https://buildroot.org/downloads/manual/manual.html

# Chapter 4 Buildroot quick start

```sh
make menuconfig
make
```

make menuconfig 进入选择菜单，可以选择编译 kernel, bootloader, rootfs。

编译完成后内容会放到`output`目录下，其中：

`image`: 包括 kernel image, bootloader and root filesystem images

`build`:

`host`:

`staging`: 软链接，指向/host/\<toolchain\>/sysroot

`target`: 就是目标板的文件系统，和 staging 相比，developing files(header, etc.)被省略了，binaries are stripped, 去除了 debug info。

# PARTⅡ User Guide

# Chapter 6 Buildroot configuration

## 6.1 Cross-compilation toolchain

Buildroot 提供两种 toolchain:

- **internal** toolchain backend, `Buildroot toolchain` in menuconfig.
- **external** toolchain backend. `External toolchain` in menuconfig.

在 menuconfig`Toolchain Type`->`Toolchain` 中选择。

### 6.1.1 Internal toolchain backend

`make uclibc-menuconfig`: 可以修改

缺点是，修改内部工具链选项时，整个 toolchain 和 system 必须重新 rebuild，很耗时间。

### 6.1.2 External toolchain backend

## 6.2. /dev management

// todo:

## 6.3 init system

Buildroot 支持三种 init 方式，在`System configuration, Init system`中配置：

- 默认方法为 busybox`BR2_INIT_BUSYBOX`，利用 init 启动程序`/etc/inittab`,该文件主要任务是启动`/etc/init.d/rcS`。该 init 程序位于 package/busybox/inittab。
- systemV
- systemd

# Chapter 7 Configuration of other components

Buildroot 还可对下面的 components 进行配置，在配置前确保对应的 component 已经在 buildroot menuconfig 中 enable。

- BusyBox. 开启选项：`BR2_PACKAGE_BUSYBOX_CONFIG`, 配置菜单：`make busybox-menuconfig`
- uClibc. 开启选项：`BR2_UCLIBC_CONFIG`, 配置菜单：`make uclibc-menuconfig`
- Linux Kernel. linux config 文件：`BR2_LINUX_KERNEL_USE_CUSTOM_CONFIG`, 配置菜单：`make linux-menuconfig`
- Barebox. Barebox config 文件：`BR2_TARGET_BAREBOX_USE_CUSTOM_CONFIG`， 配置菜单：`make barebox-menuconfig`
- U-Boot. Uboot config 文件：`BR2_TARGET_UBOOT_USE_CUSTOM_CONFIG`， 配置菜单：`make uboot-menuconfig`

# Chapter 8 General Buildroot usage

`make clean`：只删除 build products。

`make distclean`: 删除所有 build products 和 configuration。

`make -s printvars VARS=`: 可以打印编译过程中的变量，比如：

```shell
$ make -s printvars VARS=BR2_EXTERNAL
BR2_EXTERNAL=/home/yucheng_xiang/sdk_3917/platform

$ make -s printvars VARS=BR2_EXTERNAL RAW_VARS=YES # 如果变量中还有变量，这种方式可以不展开
```

## 8.2 Understanding when a full rebuild is necessary

Full rebuild: `make clean all`

什么时候需要 full rebuild:

- architecture changed, 比如 binary format, floating point strategy 等
- toolchain changed
- rootfs skeleton changed. 但 rootfs overlay, 比如 post-build script 改动只需要 make 就可以

## 8.3 Understanding how to rebuild packages

`make <package>-dirclean`: 直接删除 package 目录。

`make <package>-rebuild`: 执行 make and make install, 因此只会 rebuild changed files。

`make <package>-reconfigure`: restart the configuration, compilation and installation of the package.

## 8.5 Building out-of-tree

`make O=/tmp/build menuconfig`

Or:

`cd /tmp/build; make O=$PWD -C path/to/buildroot menuconfig`

注意 O 后面的路径可以是相对或者绝对路径，相对路径是相对于 buildroot 根目录而不是当前目录。

## 8.6 Environment variables

`HOSTCC`, `BR2_DL_DIR`等一些 buildroot 自带的环境变量。

## 8.9 Graphing the dependencies between packages

`make graph-depends`：生成 packages 之间的依赖图，保存在 output/graphs/graph-depends.pdf。

`make <pkg>-graph-depends`: 生成某个 package 的依赖关系图。

## 8.10 Graphing the build duration

`make graph-build`: 生成每个 package 的 build 时间。

## 8.11 Graphing the filesystem size contribution of packages

`make graph-size`: 计算每个 packge 占 rootfs 的大小。

## 8.13 Advanced usage

### 8.13.1 Using the generated toolchain outside Buildroot

### 8.13.2 Using gdb in Buildroot

为了支持 gdb debug program。

- Internal toolchain 需要打开`BR2_PACKAGE_HOST_GDB`, `BR2_PACKAGE_GDB`, `BR2_PACKAGE_GDB_SERVER`，这保证了 host machine 使用的 cross gdb 和 target 使用的 gdb server 被编译。
- external toolchain 需要打开`BR2_TOOLCHAIN_EXTERNAL_GDB_SERVER_COPY`,把外部 toolchain 的 gdbserver 拷贝到 target，需要外部 toolchain 没有 cross gdb 和 gdbserver，那么启用和 internal toolchain 一样的选项。

调试 foo 程序，qemu 启动后，在 target 上:

```sh
gdbserver :2345 foo
```

在 host 上：

```sh
<buildroot>/output/host/bin/<tuple>-gdb -ix <buildroot>/output/staging/usr/share/buildroot/gdbinit foo

(gdb) target remote <target ip address>:2345
```

具体可参考 https://yc-xiang.github.io/posts/note/linux_driver/drm/qemu_debug_drm/

### 8.13.3 Using ccache in Buildroot

`BR2_CCACHE`

### 8.13.4 Location of downloaded packages

`BR2_DL_DIR`: 修改下载的 tarballs 位置。

### 8.13.5 Package-specific make targets

常用的 package make 命令:

The package build targets are (in the order they are executed):

- `make <package>-source`: Fetch the source (download the tarball, clone the source repository, etc)
- `make <p>-depends`: Build and install all dependencies required to build the package
- `make <p>-extract`: Put the source in the package build directory (extract the tarball, copy the source, etc)
- `make <p>-patch`: Apply the patches, if any
- `make <p>-configure`: Run the configure commands, if any
- `make <p>-build`: Run the compilation commands
- `make <p>-install`: Run installation commands

Additionally, there are some other useful make targets:

- `make <p>-show-depends`: Displays the first-order dependencies required to build the package. 只展示一层依赖。
- `make <p>-show-recursive-depends`: Recursively displays the dependencies required to build the package. 展示多层依赖，所有需要的软件包。
- `make <p>-graph-depends`: Generate a dependency graph of the package
- `make <p>-dirclean`: 删除整个 package 目录
- `make <p>-reinstall`: Re-run the install commands
- `make <p>-rebuild`: Re-run the build commands
- `make <p>-reconfigure`: Re-run the configure commands, then rebuild

### 8.13.6 Using Buildroot during development

`BR2_PACKAGE_OVERRIDE_FILE`: 该选项指明了 override file 的路径和名称。默认位置为`$(CONFIG_DIR)/local.mk`，通常和.config 在同一目录下。

在该 override file 中可以指定各 package source code 的覆盖路径。这样 buildroot 就不用从网上下载，解压 package 了。

```makefile
<pkg1>_OVERRIDE_SRCDIR = /path/to/pkg1/sources
<pkg2>_OVERRIDE_SRCDIR = /path/to/pkg2/sources
```

可以通过`<pkg>_OVERRIDE_SRCDIR_RSYNC_EXCLUSIONS`，在 rsync 时额外排除一些文件，防止拷贝过慢：

```makefile
WEBKITGTK_OVERRIDE_SRCDIR = /home/bob/WebKit
WEBKITGTK_OVERRIDE_SRCDIR_RSYNC_EXCLUSIONS = \
        --exclude JSTests --exclude ManualTests --exclude PerformanceTests \
        --exclude WebDriverTests --exclude WebKitBuild --exclude WebKitLibraries \
        --exclude WebKit.xcworkspace --exclude Websites --exclude Examples
```

# Chapter 9 Project-specific customization

## 9.1 Recommended directory structure

```txt
+-- board/
|   +-- <company>/
|       +-- <boardname>/
|           +-- linux.config
|           +-- busybox.config
|           +-- <other configuration files>
|           +-- post_build.sh
|           +-- post_image.sh
|           +-- rootfs_overlay/
|           |   +-- etc/
|           |   +-- <some files>
|           +-- patches/
|               +-- foo/
|               |   +-- <some patches>
|               +-- libbar/
|                   +-- <some other patches>
|
+-- configs/
|   +-- <boardname>_defconfig
|
+-- package/
|   +-- <company>/
|       +-- Config.in (if not using a br2-external tree)
|       +-- <company>.mk (if not using a br2-external tree)
|       +-- package1/
|       |    +-- Config.in
|       |    +-- package1.mk
|       +-- package2/
|           +-- Config.in
|           +-- package2.mk
|
+-- Config.in (if using a br2-external tree)
+-- external.mk (if using a br2-external tree)
+-- external.desc (if using a br2-external tree)
```

如果使用 br2-external tree, 那么\<company\>和\<boardname\>可以省略。

### 9.1.1 Implementing layered customizations

在 board/\<company\>/下面增加一层 common 层，可以对所有其他的 board 都 apply，这样就不用每个 board 都需要加相同的 config。

```txt
+-- board/
    +-- <company>/
        +-- common/
        |   +-- post_build.sh
        |   +-- rootfs_overlay/
        |   |   +-- ...
        |   +-- patches/
        |       +-- ...
        |
        +-- fooboard/
            +-- linux.config
            +-- busybox.config
            +-- <other configuration files>
            +-- post_build.sh
            +-- rootfs_overlay/
            |   +-- ...
            +-- patches/
                +-- ...
```

<p class="note note-info">是否需要设置 BR2_GLOBAL_PATCH_DIR="board/<company>/common/patches board/<company>/fooboard/patches"???</p>

## 9.2 Keeping customizations outside of Buildroot

使用 br2-external tree 的结构，源码放在 buildroot 外部，在编译时指定`BR2_EXTERNAL`

```shell
make BR2_EXTERNAL=/path/to/foo menuconfig
```

编译完成后会在 output 目录下生成`.br2-external.mk`文件，保存了一些 br2-external tree 的变量。

### 9.2.1 Layout of a br2-external tree

使用 br2-external tree 必须有下面三个文件：

- external.desc
- external.mk
- Config.in

可选的有：

- configs/ 自定义的板子 defconfig
- provides/ 可调整 package 的实现方式，比如 jpeg 可以新建 jepg.in 来选择 libjpeg 还是 jpeg-turbo。

</br>

**external.desc**, 需要提供 name 和 desc，比如：

```txt
name: bar_42
desc: xxxxxxxxx
```

会生成两个变量`BR2_EXTERNAL_$(NAME)_PATH`(指向 br2-external tree 路径) 和 `BR2_EXTERNAL_$(NAME)_DESC`

**external.mk**, makefile 的总入口，include 各种 package 的\*.mk，e.g.:

```txt
include $(sort $(wildcard $(BR2_EXTERNAL_BAR_42_PATH)/package/*/*.mk))
```

**Config.in**, Kconfig 总入口，source 各种 package 的 Config.in e.g.:

```txt
source "$BR2_EXTERNAL_BAR_42_PATH/package/package1/Config.in"
source "$BR2_EXTERNAL_BAR_42_PATH/package/package2/Config.in"
```

## 9.4 Storing the configuration of other components

`make linux-update-defconfig`: save config to `BR2_LINUX_KERNEL_CUSTOM_CONFIG_FILE`
`make busybox-update-config` to `BR2_PACKAGE_BUSYBOX_CONFIG`
`make uclibc-update-config` to `BR2_UCLIBC_CONFIG`
`make barebox-update-defconfig` to `BR2_TARGET_BAREBOX_CUSTOM_CONFIG_FILE`
`make uboot-update-defconfig` to `BR2_TARGET_UBOOT_CUSTOM_CONFIG_FILE`

## 9.5 Customizing the generated target filesystem

Root filesystem overlays: `BR2_ROOTFS_OVERLAY`

Post-build scripts: `BR2_ROOTFS_POST_BUILD_SCRIPT`

### 9.5.1 Setting file permissions and ownership and adding custom devices nodes

`BR2_ROOTFS_DEVICE_TABLE`

## 9.6 Adding custom user accounts

增加用户：`BR2_ROOTFS_USERS_TABLES`

## 9.7 Customization after the images have been created

`BR2_ROOTFS_POST_IMAGE_SCRIPT`, post-image script, 在所有 image 编译完成后执行的脚本。

## 9.8 Adding project-specific patches and hashes

首先会打全局的 patch，保存在:

`BR2_GLOBAL_PATCH_DIR/<packagename>/(<packageversion>/)`

再打每个 package 文件夹下的 patch:

`<number>-<description>.patch`, 按 number 的顺序打上 patch。

验证 package 完整性的 hash 文件路径和 patch 一样，文件名为`<packagename>.hash`

## 9.9 Adding project-specific packages

## 9.10 Quick guide to storing your project-specific customizations

参考网页

# My Notes

一些环境变量：

- `output/`对应`BASE_DIR`
- `output/build/`对应`BUILD_DIR`
- `output/host/`对应`HOST_DIR`
  - Contains both the tools built for the host (cross-compiler, etc.) and the sysroot of
    the toolchain
  - Host tools are directly in `host/`
  - The sysroot is in `host/<tuple>/sysroot/usr`E.g: `arm-unknown-linux-gnueabihf`
  - Variable for the sysroot: `STAGING_DIR`. `ouput`目录下的`staging`目录也是软连接到这的
- `output/target/`对应`TARGET_DIR`
  - Used to generate the final root filesystem images in` images/`
- `output/image/`对应`BINARIES_DIR`

# PartⅢ Developer guide

## Chapter 17 Adding support for a particular board

记录了怎么 porting 开发板，略。

## Chapter 18. Adding new packages to Buildroot

## 18.2 Config files

**Config.in:**

这边放置的给 target 的软件包。

```kconfig
config BR2_PACKAGE_LIBFOO
        bool "libfoo"
        help
          This is a comment that explains what libfoo is. The help text
          should be wrapped.

          http://foosoftware.org/libfoo/
```

**Config.in.host:**

这边放置的给 host 的软件包。

有两种情况：

1. 如果该软件包 host-xxx 只是在编译其他软件包时需要，那么不要创建 Config.in.host 文件，只需在.mk 中把 host-xxx 放到其他软件包的 \<package-name\>\_DEPENDENCIES 变量中。

2. 该软件包 host-xxx 需要用户在 menuconfig 中主动选择，那么需要创建 Config.in.host 文件。之后该选项会在 menuconfig->Host utilities 中看到。

语法和 Config.in 一样。

## 18.3 The .mk file

一般有五种不同的.mk 文件类型：

- generic packages
- autotools-based packages
- cmake-based packages
- python modules
- lua modules

## 18.4 The .hash file

对下载下来的文件进行 hash 验证，一般 hash 值 download 网站会提供。

如果 package 提供了版本号，那么一般放在`/package/libfoo/1.2.3/libfoo.hash`

```txt
# Hashes from: http://www.foosoftware.org/download/libfoo-1.2.3.tar.bz2.{sha1,sha256}:
sha1  486fb55c3efa71148fe07895fd713ea3a5ae343a  libfoo-1.2.3.tar.bz2
sha256  efc8103cc3bcb06bda6a781532d12701eb081ad83e8f90004b39ab81b65d4369  libfoo-1.2.3.tar.bz2

# md5 from: http://www.foosoftware.org/download/libfoo-1.2.3.tar.bz2.md5, sha256 locally computed:
md5  2d608f3c318c6b7557d551a5a09314f03452f1a1  libfoo-data.bin
sha256  01ba4719c80b6fe911b091a7c05124b64eeece964e09c058ef8f9805daca546b  libfoo-data.bin

# Locally computed:
sha256  ff52101fb90bbfc3fe9475e425688c660f46216d7e751c4bbdb1dc85cdccacb9  libfoo-fix-blabla.patch

# Hash for license files:
sha256  a45a845012742796534f7e91fe623262ccfb99460a2bd04015bd28d66fba95b8  COPYING
sha256  01b1f9f2c8ee648a7a596a1abe8aa4ed7899b1c9e5551bda06da6e422b04aa55  doc/COPYING.LGPL
```

## 18.5 The SNNFoo start script

采用 systemd 启动方式的系统，package 需要提供一个启动脚本。

## 18.6 Infrastructure for packages with specific build systems

### 18.6.1 generic-package tutorial

```makefile
01: ################################################################################
02: #
03: # libfoo
04: #
05: ################################################################################
06:
07: LIBFOO_VERSION = 1.0
08: LIBFOO_SOURCE = libfoo-$(LIBFOO_VERSION).tar.gz
09: LIBFOO_SITE = http://www.foosoftware.org/download
10: LIBFOO_LICENSE = GPL-3.0+
11: LIBFOO_LICENSE_FILES = COPYING
12: LIBFOO_INSTALL_STAGING = YES
13: LIBFOO_CONFIG_SCRIPTS = libfoo-config
14: LIBFOO_DEPENDENCIES = host-libaaa libbbb
15:
16: define LIBFOO_BUILD_CMDS
17:     $(MAKE) $(TARGET_CONFIGURE_OPTS) -C $(@D) all
18: endef
19:
20: define LIBFOO_INSTALL_STAGING_CMDS
21:     $(INSTALL) -D -m 0755 $(@D)/libfoo.a $(STAGING_DIR)/usr/lib/libfoo.a
22:     $(INSTALL) -D -m 0644 $(@D)/foo.h $(STAGING_DIR)/usr/include/foo.h
23:     $(INSTALL) -D -m 0755 $(@D)/libfoo.so* $(STAGING_DIR)/usr/lib
24: endef
25:
26: define LIBFOO_INSTALL_TARGET_CMDS
27:     $(INSTALL) -D -m 0755 $(@D)/libfoo.so* $(TARGET_DIR)/usr/lib
28:     $(INSTALL) -d -m 0755 $(TARGET_DIR)/etc/foo.d
29: endef
30:
31: define LIBFOO_USERS
32:     foo -1 libfoo -1 * - - - LibFoo daemon
33: endef
34:
35: define LIBFOO_DEVICES
36:     /dev/foo c 666 0 0 42 0 - - -
37: endef
38:
39: define LIBFOO_PERMISSIONS
40:     /bin/foo f 4755 foo libfoo - - - - -
41: endef
42:
43: $(eval $(generic-package))
```

7-11 行描述了 metadata，包括了 package version，下载网址，license。

12 行 LIBFOO_INSTALL_STAGING = YES，会安装一些东西到 staging space。会保证 20-24 行 LIBFOO_INSTALL_STAGING_CMDS 之间的命令被执行，如果是 library package 一般都需要。

13 行 libfoo-config 是一个 shell 脚本，// TODO: 这个设定没看懂干嘛的。

14 行 指定该 package 需要依赖的包。

16-29 行 LIBFOO_BUILD_CMDS 定义了 build package 时需要的操作，LIBFOO_INSTALL_STAGING_CMDS 定义了 install package 到 staging space 的操作，LIBFOO_INSTALL_TARGET_CMDS 定义了 install package 到 target space 的操作。

这些操作都依赖于`$(@D)`, 该变量为 package source code 的目录。

31-33 行 定义使用该 package 的 user。

35-37 行 定义该 package 使用的 device node。

39-41 行 设置该 package install 文件的权限。

43 行 表示是 generic package。

### 18.6.2 generic-package reference

```makefile
$(eval $(generic-package))
$(eval $(host-generic-package))
```

host-generic-package 是用来生成 host package 的，host-generic-package 必须在 generic-package 后面。
