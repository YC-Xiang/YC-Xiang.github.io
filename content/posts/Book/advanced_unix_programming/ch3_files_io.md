## 3.2 文件描述符

文件描述符 0,1,2 分别表示标准输入，标准输出，标准错误，`STDIN_FILENO`, `STDOUT_FILENO`, `STDERR_FILENO`。

## 3.3 open, openat

open/openat 函数可以打开或者创建一个文件。

```c++
int open(const char *path, int oflag, ...);
int openat(int fd, const char *path, int oflag, ...);
```

path 是要打开或创建文件的名字，oflag 的选项有：

下面五个选项是互斥的，只能选择一个：

- O_RDONLY 只读打开
- O_WRONLY 只写打开
- O_RDWR 读写打开
- O_EXEC 只执行打开
- O_SEARCH 只搜索打开 (应用于目录)

下面的选项是可选的

- O_APPEND
- O_CLOEXEC
- O_CREAT
- O_DIRECTORY
- O_EXCL
- O_NOCTTY
- O_NOFOLLOW
- O_NONBLOCK
- O_SYNC
- O_TRUNC 如果此文件存在，而且为只写或读写打开，则将其长度截断为 0.
- O_TTY_INIT
- O_DSYNC
- O_RSYNC

`open` 和 `openat` 函数的区别在于 fd 参数。

- path 是绝对路径名，那么 fd 被忽略，openat 相当于 open 函数。
- path 是相对路径名，fd 指出了相对路径名在文件系统中的开始地址，fd 是通打开相对路径名所在的目录获取的。
- path 是相对路径名，fd 是特殊值 AT_FDCWD，这种情况下，路径名在当前工作目录获取。

## 3.4 creat

create 可以创建一个新文件。

```c++
int creat(const char *path, mode_t mode);
```

等效于 `open(path, O_WRONLY | O_CREAT | O_TRUNC, mode)`

## 3.5 close

close 函数关闭一个打开的文件。

```c++
int close(int fd);
```

当一个进程终止时，内核会自动关闭它所有打开的文件。

## 3.6 lseek

每个文件都有一个与其关联的“当前文件偏移量”。当打开一个文件，除非指定 O_APPEND 选项，否则偏移量默认为 0.

lseek 函数可以显式设置文件的偏移量。

```c++
off_t lseek(int fd, off_t offset, int whence);
```

offset 的含义和 whence 有关系，whence 可选值有：

- SEEK_SET: 将文件偏移量设置为距离文件开始处 offset 个字节。
- SEEK_CUR: 将文件偏移量设置为当前偏移量加 offset 个字节。offset 可正可负。
- SEEK_END: 将文件偏移量设置为文件长度加 offset。offset 可正可负。

若 lseek 成功执行，返回新的偏移量。

可用如下方式确定打开文件的偏移量，该方法还可以判断该文件是否可以设置偏移量，因为失败会返回 -1.

注意不要通过 lseek 是否返回负数来判断可不可以设置偏移量，而需要判断 -1，因为返回的偏移量可能是负的。

```c++
off_t currpos;
currpos = lseek(fd, 0, SEEK_CUR);

if (currpos == -1)
	printf("cannot seek\n");
```

## 3.7 read

```c++
ssize_t read(int fd, void *buf, size_t nbytes);
```

## 3.8 write

```c++
ssize_t write(int fd, const void *buf, size_t nbytes);
```

## 3.10 文件共享

## 3.11 原子操作

## 3.12 dup, dup2

用来复制一个现有的文件描述符：

```c++
int dup(int fd);
int dup2(int fd, int fd2);
```

dup 返回的文件描述符是当前可用的文件描述符的最小值。  
dup2 可以用 fd2 指定新描述符的值，如果 fd2 已经打开，则先将其关闭。如果 fd==fd2，那么直接返回 fd。

执行 dup 函数后，有两个 fd 指向文件表项。

![](https://xyc-1316422823.cos.ap-shanghai.myqcloud.com/20250714114317.png)

`dup(fd)` 等价于 `fcntl(fd, F_DUPFD, 0)`

`dup2(fd, fd2)` 等价于 `close(fd2)` + `fcntl(fd, F_DUPFD, fd2)`, 不过 dup2 是原子操作。

## 3.13 sync, fsync, fdatasync

磁盘 io 一般有缓冲区来实现延迟写，这三个函数用来保证磁盘上的内容和缓冲区一致。

```c++
int fsync(int fd);
int fdatasync(int fd);
void sync();
```

- sync: 只将所有修改过的缓冲区排入写队列，立即返回，不会阻塞等待磁盘操作结束。update 系统守护进程一般每 30s 就会调用一次 sync 函数。
- fsync: 只对某个 fd 生效，并且会阻塞等待磁盘操作完成。
- fdatasync: 和 fsync 类似，不过只影响数据部分。而 fsync 还会同步更新文件的属性。

## 3.14 fcntl

fcntl 用来改变已打开文件的属性。

```c++
int fcntl(int fd, int cmd, ...); /* int arg */
```

fcntl 函数有五种功能：

- 复制一个已有的描述符 (cmd = F_DUPFD 或 F_DUPFD_CLOEXEC)
- 获取/设置文件描述符标志 (cmd = F_GETFD 或 F_SETFD)
- 获取/设置文件状态标志 (cmd = F_GETFL 或 F_SETFL)
- 获取/设置异步 IO (cmd = F_GETFL 或 F_SETFL)
- 获取/设置记录锁 (cmd = F_GETLK, F_SETLK 或 F_SETLKW)

## 3.15 ioctl

```c++
int ioctl(int fd, int request, ...);
```

## 3.16 /dev/fd
