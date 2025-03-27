---
title: xv6_lab1 Unix utilities
date: 2023-12-30
tags:
  - xv6 OS
categories:
  - Project
draft: true
---

# memo

检查所有 questions 正确性：

```shell
make grade
```

# Boot xv6

```shell
git clone git://g.csail.mit.edu/xv6-labs-2023
cd xv6-labs-2023
```

```shell
make qemu
```

# sleep

Makefile

```makefile
UPROGS=\
	$U/_sleep\
```

user/sleep.c

```c++
#include "kernel/types.h"
#include "user/user.h"

int main(int argc, char *argv[]) {
  int sum = 0;
  int ret = 0;
  if (argc == 1) {
    printf("Usage: sleep + <time>\n");
    exit(1);
  }
  for (int i = 1; i < argc; i++) {
    sum += atoi(argv[i]);
  }
  ret = sleep(sum);
  if (ret < 0) {
    printf("sleep system call return wrong!\n");
  }
  exit(0);
}
```

检查正确性:

```shell
$ ./grade-lab-util sleep
$ make GRADEFLAGS=sleep grade # 等价
== Test sleep, no arguments == sleep, no arguments: OK (0.9s)
== Test sleep, returns == sleep, returns: OK (0.9s)
== Test sleep, makes syscall == sleep, makes syscall: OK (1.1s)
```

# pingpong

Makefile

```makefile
UPROGS=\
	$U/_pingpong\
```

pingpong.c

```c++
#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"

int main(int argc, char *argv[]) {
  int p[2];
  char buf[2];
  char *parmsg = "a";
  char *chimsg = "b";

  pipe(p);

  if (fork() == 0) {
    if (read(p[0], buf, 1) != 1) // block等待父进程write
    {
      fprintf(2, "Can't read from parent!\n");
      exit(1);
    }
    close(p[0]); // pipe创建的文件描述符，用完需要close

    printf("%d: received ping\n", getpid());

    if (write(p[1], chimsg, 1) != 1) {
      fprintf(2, "Can't write to parent!\n");
      exit(1);
    }
    close(p[1]);

    exit(0);

  } else {
    if (write(p[1], parmsg, 1) != 1) {
      fprintf(2, "Can't write to child!\n");
      exit(1);
    }

    close(p[1]);

    if (read(p[0], buf, 1) != 1) // block等待子进程write
    {
      fprintf(2, "Can't read from child!\n");
      exit(1);
    }
    close(p[0]);

    printf("%d: received pong\n", getpid());

    exit(0);
  }
}
```

```shell
$ pingpong
4: received ping
3: received pong
```

```shell
./grade-lab-util pingpong
== Test pingpong == pingpong: OK (1.3s)
    (Old xv6.out.pingpong failure log removed)
```

# prime

# Find

```c++

```

# xargs
