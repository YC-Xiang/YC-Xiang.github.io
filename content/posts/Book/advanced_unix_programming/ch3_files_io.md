## 3.2 文件描述符

文件描述符 0,1,2 分别表示标准输入，标准输出，标准错误，`STDIN_FILENO`, `STDOUT_FILENO`, `STDERR_FILENO`。

## 3.3 open, openat

open/openat 函数可以打开或者创建一个文件。

```c++
#include <fcntl.h>

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
#include <fcntl.h>

int creat(const char *path, mode_t mode);
```

等效于 `open(path, O_WRONLY | O_CREAT | O_TRUNC, mode)`

## 3.5 close

close 函数关闭一个打开的文件。

```c++
#include <unistd.h>

int close(int fd);
```

当一个进程终止时，内核会自动关闭它所有打开的文件。

## 3.6 lseek

lseek 显式设置文件偏移量，当打开一个文件，除非指定
