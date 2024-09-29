---
title: Buildroot
date: 2023-04-13 17:39:28
tags:
  - Buildroot
categories:
  - Notes
---

# Notes

`./lanuch` 最终的操作是

```shell
cd sdk_3921/buildroot-dist # 进入buildroot目录
make BR2_EXTERNAL=sdk_3921/platform/ O=sdk3921/out/rts3923_fpga/ rts3923_fpga_defconfig
```

```makefile
local.mk # BR2_PACKAGE_OVERRIDE_FILE 在platform/configs中定义
external.mk
    include common.mk
    	include package/*/*/*.mk
    	include package/*/*.mk
    include package/*/*/src.mki
    include package/*/*/gen.mki
    include post_pkg.mk
```

`platform/local.mk`: 指定自定义源码位置

在`local.mk`中还定义了：

`make rr`: Reconstruct rootfs

`make rp`: Rebuild external packages

## Managing the build and the configuration

### Out of tree build

在`buildroot Makefile`中有

```makefile
ifeq ($(O),$(CURDIR)/output)
CONFIG_DIR := $(CURDIR)
NEED_WRAPPER =
else
CONFIG_DIR := $(O)
NEED_WRAPPER = y
endif
```

可以看出，如果`O != buildroot_dist/output`, `CONFIG_DIR = sdk3921/out/rts3923_fpga/ `

在`CONFIG_DIR`目录下，保存着`.config`配置文件。

### Other building tips

Cleaning all the build output, but keeping the configuration file(删除 build/):

`make clean`

Cleaning everything, including the configuration file, and downloaded file if at the
default location (相当于删除了 build/和.config 一系列配置文件，需要重新 make menuconfig):

`make distclean`

## Buildroot source and build trees

### Build tree

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

## Managing the Linux kernel configuration

- `make linux-update-config`, to save a full config file
- `make linux-update-defconfig`, to save a minimal defconfig

## Root filesystem in Buildroot

copy rootfs overlays->execute post-build scripts->execute post-image scripts

`platform/board/rts3923/rootfs_overlay`

`platform/board/rts3923/post_build.sh`

`platform/board/rts3923/post_image.sh`

![](https://xyc-1316422823.cos.ap-shanghai.myqcloud.com/20230412162924.png)

## Advanced topics

### BR2_EXTERNAL

Ipcam sdk 在 make 的时候指定

`make BR2_EXTERNAL=sdk_3921/platform/ O=sdk3921/out/rts3923_fpga/ rts3923_fpga_defconfig`

Each external directory must contain:

- `external.desc`, which provides a name and description. The `$BR2_EXTERNAL_<NAME>_PATH` variable is available, where NAME is defined in `external.desc`.
- `Config.in`, configuration options that will be included in menuconfig（在 menuconfig external options 里）
- `external.mk`, will be included in the make logic

`make <pkg>-dirclean`, completely remove the package source code directory. The next make invocation will fully rebuild this package. 相当于直接删除`build/<pkg>`

`make <pkg>-rebuild`, force to re-execute the build and installation steps of the package.

`make <pkg>-reconfigure`, force to re-execute the configure, build and installation steps of the package.

# legacy

## 添加自己的软件包

### 配置 APP 对应的 Config.in 和 mk 文件

在 package 中新增目录 helloworld，并在里面添加 Config.in 和 helloworld.mk
**Config.in**

```txt
config BR2_PACKAGE_HELLOWORLD
bool "helloworld"
help
  This is a demo to add myown(fuzidage) package.
```

**helloworld.mk**

```makefile
HELLOWORLD_VERSION:= 1.0.0
HELLOWORLD_SITE:= $(BR2_EXTERNAL)/source/ipcam/helloworld
HELLOWORLD_SITE_METHOD:=local
HELLOWORLD_INSTALL_TARGET:=YES

$(eval $(cmake-package))

```

## 如何重新编译软件包

经过第一次完整编译后，如果我们需要对源码包重新配置，我们不能直接在 buildroot 上的根目录下直接 make，buildroot 是不知道你已经对源码进行重新配置，它只会将第一次编译出来的文件，再次打包成根文件系统镜像文件。

那么可以通过以下 2 种方式重新编译：

**1. 直接删除源码包,然后 make all**

例如我们要重新编译 helloworld，那么可以直接删除 output/build/helloworld 目录，那么当你 make 的时候，就会自动从 dl 文件夹下，解压缩源码包，并重新安装。这种效率偏低

**2. 进行 xxx-rebuild,然后 make all**

也是以 helloworld 为例子，我们直接输入 make helloworld-rebuild，即可对 build/helloworld/目录进行重新编译，然后还要进行 make all(或者 make helloworld)

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

`staging`:

`target`:

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

- 默认方法为 busybox`BR2_INIT_BUSYBOX`，启动文件`system/skeleton/etc/inittab`,主要任务是启动`/etc/init.d/rcS`
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

`make -s printvars VARS=''`: 可以打印编译过程中的变量，比如：

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
