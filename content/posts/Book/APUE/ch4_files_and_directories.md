## 4.2 stat, fstat, fstatat, lstat

```c++
int stat(const char *restrict pathname, struct stat *restrict buf);
int fstat(int fd, struct stat buf);
int lstat(const char* restrict pathname, struct stat *restrict buf);
int fstatat(int fd, const char *restrict pathname, struct stat *restrict buf, int flag);
```

- stat: 返回 pathname 文件的信息结构。
- fstat: 返回 fd 文件描述符的文件信息。
- lstat: 和 stat 类似，但如果 pathname 是一个符号链接，lstat 返回符号链接的有关信息，而不是符号链接指向的文件信息。
- fstatat 返回相对于 fd 文件描述符 pathname 路径名的文件信息。flag AT_SYMLINK_NOFOLLOW 表示不跟随符号链接展开。

stat 结构体不同系统实现可能不同：

```c++
struct stat {
	dev_t		st_dev;                 /* [XSI] ID of device containing file */ \
	mode_t		st_mode;                /* [XSI] Mode of file (see below) */ \
	nlink_t		st_nlink;               /* [XSI] Number of hard links */ \
	__darwin_ino64_t st_ino;                /* [XSI] File serial number */ \
	uid_t		st_uid;                 /* [XSI] User ID of the file */ \
	gid_t		st_gid;                 /* [XSI] Group ID of the file */ \
	dev_t		st_rdev;                /* [XSI] Device ID */ \
	__DARWIN_STRUCT_STAT64_TIMES \
	off_t		st_size;                /* [XSI] file size, in bytes */ \
	blkcnt_t	st_blocks;              /* [XSI] blocks allocated for file */ \
	blksize_t	st_blksize;             /* [XSI] optimal blocksize for I/O */ \
	__uint32_t	st_flags;               /* user defined flags for file */ \
	__uint32_t	st_gen;                 /* file generation number */ \
	__int32_t	st_lspare;              /* RESERVED: DO NOT USE! */ \
	__int64_t	st_qspare[2];           /* RESERVED: DO NOT USE! */ \
}

```

## 4.3 文件类型

- 普通文件
- 目录文件
- 块特殊文件 block special file. 提供对设备（如磁盘）带缓冲的访问。
- 字符特殊文件 character special file. 提供对设备不带缓冲的访问，系统中的所有设备要么是字符特殊文件，要么是块特殊文件
- FIFO
- 套接字 socket
- 符号链接 symbolic link

文件类型信息包含在 stat 结构的 st_mode 成员中。

```c++
S_ISREG() // 普通文件
S_ISDIR() // 目录文件
S_ISCHR() // 字符特殊文件
S_ISBLK() // 块特殊文件
S_ISFIFO() // FIFO
S_ISLINK() // 符号链接
S_ISSOCK() // 套接字
```
