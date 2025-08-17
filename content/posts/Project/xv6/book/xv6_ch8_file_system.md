# File System

## 8.1 Overview

File system 架构：

![](https://xyc-1316422823.cos.ap-shanghai.myqcloud.com/20250817094004.png)

- disk layer: 读写磁盘 block
- buffer cache layer: 缓存和同步磁盘数据
- logging layer: 把多个 blocks 打包到一个 transaction.
- inode layer: 包括 individual files, 每个都被表示为一个 inode.
- directory layer: 包括 directories, 每个都被表示为一个特殊的 inode.
- path name layer: 提供像/xv6/fs.c 这样的层级路径名。
- file descriptor layer: 使用 file descriptor 抽象一些 unix 资源 (pipes, devices, files)

disk hardware 一般以 block(或者叫 sector) 为单位，一个 block 通常是 512 bytes.

操作系统使用的 block size 可能与 disk 的 block size 不同，一般是 disk 的 block size 的整数倍。

xv6 使用`struct buf` 来保存 disk block 的数据，可能有时与 disk 中的 block 不同步。

</br>

xv6 将 disk 分成以下的结构：

![](https://xyc-1316422823.cos.ap-shanghai.myqcloud.com/20250817095803.png)

- block 0: file system 不使用，boot sector.
- block 1: super block, 保存 file system 的 metadata，包括 file system size in blocks, data blocks 数量，inodes 数量，log 中 blocks 的数量等。由一个名为 mkfs 的单独程序填充，该程序构建一个初始文件系统
- block 2 开始：log.
- inode: 保存 inode 数据。
- bitmap: 保存使用中的 data block bitmap。
- data: 保存 data block 数据。

## 8.2 Buffer cache layer

buffer cache 有两个任务：

1. 同步访问 disk block，以确保在内存中只有一个 block 的副本，并且一次只有一个内核线程使用该副本
2. 缓存 popular blocks，这样就不需要从 disk 重新读取，相关代码在 bio.c 中。

`bread`: 从 disk 读取一个 block 到 buffer cache.
`bwrite`: 将 buffer cache 中的 block 写入 disk.
`brelse`: kernel thread 在结束对 buffer 的访问时必须调用 brelse 来释放。

buffer cache 使用 LRU 算法来替换 block 块。

## 8.3 Code: Buffer cache

```c++
struct buf {
	int valid; // has data been read from disk?
	int disk; // does disk "own" buf?
	uint dev;
	uint blockno;
	struct sleeplock lock;
	uint refcnt;
	struct buf *prev; // LRU cache list
	struct buf *next;
	uchar data[BSIZE];
};
```

`binit` 函数初始化 the NBUF buffers.

![](https://xyc-1316422823.cos.ap-shanghai.myqcloud.com/20250817142045.png)

`bread` 函数获取一个 buf, 如果 buf 的 valid 为 0, 则从 disk 读取一个 block 到 buf, 并设置 valid 为 1.

`bwrite` 函数将 buffer cache 中的 block 写入 disk.

`brelse` 释放一个 buf, 并放到链表最前面。

![](https://xyc-1316422823.cos.ap-shanghai.myqcloud.com/20250817142909.png)

LRU 算法取 buffer 时，是从 head->prev, 从链表尾部开始拿出一个 refcnt 为 0 的 buffer, 该 buffer 就为 least recently used buffer.

## 8.4 Logging layer

## 8.5 Log design

## 8.6 Code: Logging

## 8.7 Code: Block allocator

`balloc` 分配一个 disk block.

`bfree` 释放一个 disk block.

## 8.8 Inode layer

## 8.9 Code: Inodes

术语 inode 可以有两个相关含义之一。它可能指的是包含 file size 和 list of data block number 的 on-disk data structure。或者 inode 可能指的是内存中的 inode，它包含磁盘上 inode 的副本以及内核中所需的额外信息。

on disk inode 由 `struct dinode` 表示。

```c++
struct dinode {
  short type;           // File type
  short major;          // Major device number (T_DEVICE only)
  short minor;          // Minor device number (T_DEVICE only)
  short nlink;          // Number of links to inode in file system
  uint size;            // Size of file (bytes)
  uint addrs[NDIRECT+1];   // Data block addresses
};
```

`type`: 区分文件、目录和特殊文件（设备）。0 表示磁盘上的 inode 是空闲的。

</br>

memory inode 由 `struct inode` 表示。

```c++
struct inode {
  uint dev;           // Device number
  uint inum;          // Inode number
  int ref;            // Reference count
  struct sleeplock lock; // protects everything below here
  int valid;          // inode has been read from disk?

  short type;         // copy of disk inode
  short major;
  short minor;
  short nlink;
  uint size;
  uint addrs[NDIRECT+1];
};
```

kernel 将 active inodes in memory 放到`itable`中管理。

## 8.10 Code: Inode content

inode content 包括 direct blocks 和 indirect blocks. 前者 d 的地址保存在 inode 的 addrs 数组中，共有 NDIRECT 12 个。indirect blocks 共有 256 个，需要访问 addrs 数组的最后一个成员来获取 block 地址。

![](https://xyc-1316422823.cos.ap-shanghai.myqcloud.com/20250817130503.png)

参考 `bmap` 函数的实现。

## 8.11 Code: directory layer

`struct dirent` 代表一个 directory entry, 包含一个 name 和 inode number.

```c++
struct dirent {
	ushort inum;
	char name[DIRSIZ];
};
```

## 8.12 Code: Path names

## 8.13 File descriptor layer

## 8.14 Code: System calls
