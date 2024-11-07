# 第一篇 新学习路线、视频介绍、资料下载、开发板基础操作

# 第三篇 环境搭建与开发板操作

## 2.6 下载 bsp 和配置交叉编译工具链

```shell
$ git clone https://e.coding.net/codebug8/repo.git # download reop
$ mkdir -p 100ask_imx6ull-sdk && cd 100ask_imx6ull-sdk
$ ../repo/repo init -u https://gitee.com/weidongshan /manifests.git -b linux-sdk -m imx6ull/100ask_imx6ull_linux4.9.88_release.xml --no-repo-verify
$ ../repo/repo sync -j4
```

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

Ubuntu:

防止开发板挂载出现 permission denied 情况, 需要修改`/etc/exports`文件:

```sh
/srv/homes hostname1(rw,sync,no_subtree_check) hostname2(ro,sync,no_subtree_check)

# 挂载share目录
/home/yucheng_xiang/share *(rw,sync,no_root_squash)
```

修改完文件后,执行 `exportfs -rv`

开发板:

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

## 5.2 编译内核

make zImage + make dtbs

```sh
$ export ARCH=arm
book@100ask:~/100ask_imx6ull-sdk/Linux-4.9.88$ make mrproper
book@100ask:~/100ask_imx6ull-sdk/Linux-4.9.88$ make 100ask_imx6ull_defconfig
book@100ask:~/100ask_imx6ull-sdk/Linux-4.9.88$ make zImage -j4
book@100ask:~/100ask_imx6ull-sdk/Linux-4.9.88$ make dtbs
# 拷贝到nfs目录
book@100ask:~/100ask_imx6ull-sdk/Linux-4.9.88$ cp arch/arm/boot/zImage ~/nfs_rootfs
book@100ask:~/100ask_imx6ull-sdk/Linux-4.9.88$ cp arch/arm/boot/dts/100ask_imx6ull-1
4x14.dtb ~/nfs_rootfs
```

## 5.3 编译安装内核模块

make modules

```sh
book@100ask:~/100ask_imx6ull-sdk/Linux-4.9.88$ make modules
make INSTALL_MOD_PATH=/home/book/nfs_rootfs modules_install
```

## 5.4 安装内核和模块到开发板

```sh
cp /mnt/zImage /boot
cp /mnt/100ask_imx6ull-14x14.dtb /boot
cp /mnt/lib/modules /lib -rfd
sync
```

重启开发板后，就使用了新的 zImage, dtb, modules。

## 6.2 编译 bootloader

```sh
book@100ask: ~/100ask_imx6ull-sdk/Uboot-2017.03$ make distclean
book@100ask: ~/100ask_imx6ull-sdk/Uboot-2017.03$ make mx6ull_14x14_evk_defconfig
book@100ask: ~/100ask_imx6ull-sdk/Uboot-2017.03$ make
cp u-boot-dtb.imx ~/nfs_rootfs
```

将 bootloader 烧到 emmc 上:

```sh
[root@100ask:~] echo 0 > /sys/block/mmcblk1boot0/force_ro
[root@100ask:~] dd if=u-boot-dtb.imx of=/dev/mmcblk1boot0 bs=512 seek=2
[root@100ask:~] echo 1 > /sys/block/mmcblk1boot0/force_ro
```

## 6.5 Buildroot 构建 IMX6ULL Pro 版的根文件系统

有两个配置：

![](https://xyc-1316422823.cos.ap-shanghai.myqcloud.com/20241025224902.png)

选择第一个 core，还需要把 libsync 选上。

```sh
book@100ask:~/100ask_imx6ull-sdk/Buildroot_2020.02.x$ make clean
book@100ask:~/100ask_imx6ull-sdk/Buildroot_2020.02.x$ make 100ask_imx6ull_pro_ddr512
m_systemV_qt5_defconfig
book@100ask:~/100ask_imx6ull-sdk/Buildroot_2020.02.x$ make all -j4
```

![](https://xyc-1316422823.cos.ap-shanghai.myqcloud.com/20241025225011.png)

## 6.6 开发板使用 NFS 根文件系统

Buildroot 编译完成之后生成的 rootfs.tar.bz2, 可以解压之后放到 NFS 服务器上作为 NFS 文件系统供开发板使用。

将编译后得到的内核 zImage, 设备树文件 100ask_imx6ull-14x14.dtb, 放到 ubuntu 的 tftp 目录下。

将文件系统 rootfs.tar.bz2 解压到 Ubuntu 的/etc/exports 文件中指定的目录里，即
复制到/home/book/nfs_rootfs 目录下，并解压(注意：解压时要用 sudo)。`sudo tar -jxvf rootfs.tar.bz2`

开发板进入 uboot，执行：

```sh
=> setenv serverip 192.168.5.11 # 设置服务器的 IP 地址，这里指的是 Ubuntu 主机 IP
=> setenv ipaddr 192.168.5.9 # 设置开发板的 IP 地址。
=> setenv nfsroot /home/book/nfs_rootfs # 设置 nfs 文件系统所在目录。
=> run netboot # 设置完成后，运行网络启动系统命令
```

默认的 netargs 是 dhcp 获取 ip 地址，会进不去 kernel, 设置 netargs 为静态 ip:

```sh
netargs=setenv bootargs console=${console},${baudrate} root=/dev/nfs ip=192.168.5.9:192.168.5.11::255.255.255.0::eth0:off nfsroot=${serverip}:${nfsroot},v3,tcp
```

设置完成后，执行

```sh
run netboot
```

另外还需要在 nfs 文件系统中修改`nfs_rootfs/etc/network/interfaces`文件，也修改为静态 ip:

```txt
auto lo
iface lo inet loopback

auto eth0
iface eth0 inet static
    address 192.168.5.9
    netmask 255.255.255.0
    gateway 192.168.5.1
```

# Notes

## 7. 烧写系统

把要烧写的文件放进 `\02_开发工具\100ask_imx6ull_pro开发板系统烧写工具\files\` 目录下。
