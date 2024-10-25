**# 第一篇 **新学习路线、视频介绍、资料下载、开发板基础操作\*\*

# 第三篇 环境搭建与开发板操作

## 安装软件

### 2.6.2 下载 bsp

```shell
$ git clone https://e.coding.net/codebug8/repo.git # download reop
$ mkdir -p 100ask_imx6ull-sdk && cd 100ask_imx6ull-sdk
$ ../repo/repo init -u https://gitee.com/weidongshan /manifests.git -b linux-sdk -m imx6ull/100ask_imx6ull_linux4.9.88_release.xml --no-repo-verify
$ ../repo/repo sync -j4
```

Ubuntu22.04 python3.10 环境下 repo init 的时候会报错提示没有 formatter module，进入.repo/repo，打上这个 patch: https://gerrit-review.googlesource.com/c/git-repo/+/303282。

还需要修改 repo 文件中第一行的 python 为 python3: #!/usr/bin/env python3

### 2.6.3 配置交叉编译工具链

在.bashrc 中添加

```shell
export ARCH=arm
export CROSS_COMPILE=arm-buildroot-linux-gnueabihf-
export PATH=$PATH:/home/book/100ask_imx6ull-sdk/ToolChain/arm-buildroot-linux-gnueabihf_sdk-buildroot/bin
```

## 3.3 启动方式

- EMMC： 1.2.4 low, 3 high。
- SD/TF 卡：1.2.3 high, 4 low。
- USB： 3 low, 4 high。

## 3.4 串口连接

看图如何连接。

### 3.4.4 串口登录

输入 root 即可登录。

## 3.7 开发板挂载 Ubuntu 的 NFS 目录

```sh
[root@100ask:~]$ mount -t nfs -o nolock,vers=3 192.168.5.11:/home/book/nfs_rootfs /mnt
```

其中`192.168.5.11` 为 ubuntu 的桥接网卡 ip 地址。

## 3.8 FileZilla 在 Windows 和开发板之间传文件

## 3.9 使用 TFTP 传输文件

tftp 分为 server 和 client，server 开启 tftp 服务，client 可从 server 下载或者上传文件。

**Ubuntu 安装 tftp 服务：**

```sh
sudo apt-get install tftp-hpa tftpd-hpa
mkdir -p /home/book/tftpboot
sudo chmod 777 /home/book/tftpboot
sudo vim /etc/default/tftpd-hpa
```

在配置文件/etc/default/tftpd-hpa 中，添加以下字段设置 tftp 目录：

```vim
TFTP_DIRECTORY="/home/book/tftpboot"
TFTP_OPTIONS="-l -c -s"
```

```sh
sudo service tftpd-hpa restart # 启动tftp服务
ps -aux | grep "tftp" # 查看tftp服务是否已经启动
```

**Windows 安装 tftp 服务：**

下载 tftp64 工具。

允许应用通过 Windows 防火墙，把 tftp64 选上:

![](https://xyc-1316422823.cos.ap-shanghai.myqcloud.com/20241024224300.png)

windows 做 server，停留在这个界面即可。

![](https://xyc-1316422823.cos.ap-shanghai.myqcloud.com/20241024224535.png)

**开发板通过 tftp 传输文件：**

**开发板做 client**，从 ubuntu/windows server 获取和上传文件：

Busybox 中 tftp 的用法：

```sh
$ tftp [option] ... host [port]

# -g表示下载文件(get)
# -p表示上传文件(put)
# -l表示本地文件名(local file)
# -r表示远程主机的文件名(remote file)
```

```sh
[root@100ask:~]$ tftp -g -r zImage 192.168.5.11 # 192.168.5.11为ubuntu ip地址，从tftp设置目录下载zImage
[root@100ask:~]$ tftp -p -l zImage 192.168.5.11 # 192.168.5.11为ubuntu ip地址，上传zImage到tftp目录
```

</br>

**开发板做 server**，ubuntu/windows 做 client 主动获取和上传文件：

Uboot 中启动 tftp server:

```sh
tftpsrv
tftpsrv 0x80400000 # 将收到的文件保存到内存0x80400000
```

Windows: 利用 tftp64 的 client get/put 文件。

Ubuntu:

```sh
tftp 10.0.13.5 -m binary -c put u-boot.bin
```

Ubuntu 下 tftp 命令参考：

```txt
connect：连接到远程tftp服务器
mode：文件传输模式
put：上传文件
get：下载文件
quit：退出
verbose：显示详细的处理信息
tarce：显示包路径
status：显示当前状态信息
binary：二进制传输模式
ascii：ascii传送模式
rexmt：设置包传输的超时时间
timeout：设置重传的超时时间
help：帮助信息
?：帮助信息
```

# Notes

Buildroot 编译：

```shell
make 100ask_imx6ull_pro_ddr512m_systemV_qt5_defconfig
```

## 烧写系统

把要烧写的文件放进 `\02_开发工具\100ask_imx6ull_pro开发板系统烧写工具\files\` 目录下。

# 遇到的问题

在 ubuntu22.04 系统下编译遇到了一系列问题，大多数是因为版本不兼容，懒得换系统了，因此直接对出错的 package 进行修改。

1

Ubuntu22.04 编译 1.4.18 版本的 M4 Package 会出错，版本不对应，需要把 m4 版本修改为 1.4.19, 参考: https://stackoverflow.com/questions/69719688/buildroot-error-when-building-with-ubuntu-21-10

2

fakeroot package 也是同样的问题, 用https://git.busybox.net/buildroot/commit/?id=f45925a951318e9e53bead80b363e004301adc6f提供的buildroot fakeroot 版本替换。

3

编译到 glib package 的时候出现错误'cmake_root': temp_parser.get_cmake_var('MESON_CMAKE_ROOT')[0], 参考https://forums.100ask.net/t/topic/8074, 把系统的 cmake 卸载后编译没出现问题了。

4

编译到 qt5base-5.12.8 时出现错误：

```shell
100ask_imx6ull-sdk/Buildroot_2020.02.x/output/build/qt5base-5.12.8/src/corelib/global/qendian.h:332:35: error: ‘numeric_limits’ is not a member of ‘std’
  332 |     { return QSpecialInteger(std::numeric_limits<T>::max()); }
      |                                   ^~~~~~~~~~~~~~
```

在每个报未定义的文件中加上`#include <limits>`

5

编译 kernel 出错：

```shell
usr/bin/ld: scripts/dtc/dtc-parser.tab.o:(.bss+0x10): multiple definition of 'yylloc'; scripts/dtc/dtc-lexer.lex.o:(.bss+0x0): first defined here
```

把`linux-origin_master/scripts/dtc/dtc-lexer.lex.c`中的`YYLTYPE yylloc`修改为`extern YYLTYPE yylloc`

legacy

## 2.3 使用 buildroot 编译完整系统

```shell
# 查看分区表
cat buildroot/board/100ask/nxp-imx6ull/genimage.cfg
```

## 2.4 烧写更新开发板 LINUX 系统&裸机系统

> 使用 2.8 NFS 烧写更方便

设置为 usb 启动(4 跳高)。运行 100ASK_IMX6ULL Flashing Tool

（02_100ask_imx6ull_pro_2020.02.29_v2.0\01_Tools）

可以选择烧写到不同的设备，烧写系统，烧写裸机文件。

烧写整个系统文件来自`files/emmc.img`，可以用自己编译好的文件代替，改名为`emmc.img`

内核：`zimage`

设备树：`100ask_imx6ull-14x14.dtb`

Uboot：`u-boot-dtb.imx`

## 2.6 开发板 Windows ubuntu 三者网络互通方式

ping 不通 windows，看看 windows 防火墙关了没。

虚拟机：添加网络适配器，桥接模式，复制物理网络连接状态。

ubuntu 虚拟机和开发板 nfs：

mount -t nfs -o nolock,vers=3 192.168.31.51:/home/xyc/nfs /mnt

## 2.8 单独编译更新 kernel + dtb+内核模块+Uboot

### NFS 更新 kernel+dtb+modules

Ubuntu:

> 也可以进入 Buildroot 分别编译

```shell
cd /home/xyc/100ask_imx6ull-sdk/Linux-4.9.88
make mrproper # Remove all generated files + config + various backup files
make 100ask_imx6ull_defconfig
make zImage -j4 # kernel
make dtbs # dtb
cp arch/arm/boot/zImage ~/nfs # 生成的内核镜像在arch/arm/boot
cp arch/arm/boot/dts/100ask_imx6ull-14x14.dtb ~/nfs # 设备树在arch/arm/boot/dts/
make modules # 编译模块
make INSTALL_MOD_PATH=/home/xyc/nfs modules_install #安装模块
```

开发板：

```shell
cp /mnt/zImage /boot
cp /mnt/*.dtb /boot
cp /mnt/lib/modules /lib -rfd
```

### uboot

Ubuntu:

```shell
cd Uboot-2017.03
make distclean
make mx6ull_14x14_evk_defconfig
make

cp u-boot-dtb.imx ~/nfs/
```

开发板：

```shell
echo 0 > /sys/block/mmcblk1boot0/force_ro # 取消emmc写保护
cd ~
cp /mnt/u-boot-dtb.imx .
dd if=u-boot-dtb.imx of=/dev/mmcblk1boot0 bs=512 seek=2 #写uboot
echo 1 > /sys/block/mmcblk1boot0/force_ro
```

\*\*
