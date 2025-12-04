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
- 目录文件，只有内核可以直接写目录文件。
- 块特殊文件 block special file. 提供对设备（如磁盘）带缓冲的访问。
- 字符特殊文件 character special file. 提供对设备不带缓冲的访问，系统中的所有设备要么是字符特殊文件，要么是块特殊文件
- FIFO，有时也叫做命名管道 named pipe。
- 套接字 socket
- 符号链接 symbolic link

文件类型信息包含在 stat 结构的 st_mode 成员中，可以用下列的宏确定文件类型，这些宏的参数都是 stat 中的 st_mode。

```c++
S_ISREG() // 普通文件
S_ISDIR() // 目录文件
S_ISCHR() // 字符特殊文件
S_ISBLK() // 块特殊文件
S_ISFIFO() // FIFO
S_ISLINK() // 符号链接
S_ISSOCK() // 套接字
```

## 4.4 设置用户 ID 和设置组 ID

// TODO:

## 4.5 文件访问权限

每个文件有 9 个访问权限位：

- S_IRUSR 用户读
- S_IWUSR 用户写
- S_IXUSR 用户执行
- S_IRGRP 组读
- S_IWGRP 组写
- S_IXGRP 组执行
- S_IROTH 其他读
- S_IWOTH 其他写
- S_IXOTH 其他执行

## 4.6 新文件和目录的所有权

// TODO:

## 4.7 函数 access 和 faccessat

当用 open 打开一个文件时，内核以进程的有效用户 ID 和有效组 ID 为基础执行其访问权限测试。

而 access 和 faccessat 函数按照实际用户 ID 和实际组 ID 进行访问权限测试。

```c++
int access(const char *pathname, int mode);
int faccessat(int fd, const char *pathname, int mode, int flag);
```

如果测试文件已经存在，mode 为 F_OK, 否则 mode 为 R_OK, W_OK, X_OK 或者他们的或。

faccessat 计算相对于打开目录 (fd) 的 pathname。

## 4.8 umask

umask 为进程设置文件模式创建屏蔽字，并返回之前的值。

> umask 设置的权限位，再调用类似 create 函数创建文件，会自动屏蔽该权限位

```c++
mode_t umask(mode_t cmask);
```

其中 cmask 为 4.5 中列出的 9 个常量的若干位。

## 4.9 函数 chmod, fchmod 和 fchmodat

这三个函数可以改变现有文件的访问权限。

```c++
int chmod(const char *pathname, mode_t mode);
int fchmod(int fd, mode_t mode);
int fchmodat(int fd, const char *pathname, mode_t mode, int flag);
```

其中 mode 为 4.5 中的 9 个，加上

- S_ISUID 执行时设置用户 ID
- S_ISGID 执行时设置组 ID
- S_ISVTX 保存正文 (现代系统用不上了)
- S_IRWXU 用户读 + 写 + 执行
- S_IRWXG 组读 + 写 + 执行
- S_IRWXO 其他读 + 写 + 执行

## 4.10 黏着位

## 4.11 函数 chown, fchown, fchownat 和 lchown

用于更改文件的用户 ID 和组 ID.

```c++
int chown(const char *pathname, uid_t owner, gid_t group);
int fchown(int fd, uid_t owner, gid_t group);
int fchownat(int fd, const char *pathname, uid_t owner, gid_t group, int flag);
int lchown(const char *pathname, uid_t owner, gid_t group);
```

lchown 和 fchownat 会修改符号链接本身的所有者，chown 和 fchown 只会改变符号链接指向文件的所有者。

## 4.12 文件长度

stat 的 st_size 成员。只对普通文件，目录文件和符号链接有意义。

## 4.13 文件截断

```c++
int truncate(const char *pathname, off_t length);
int ftruncate(int fd, off_t length);
```

## 4.14 文件系统

skip

## 4.15 函数 link, linkat, unlink, unlinkat 和 remove
