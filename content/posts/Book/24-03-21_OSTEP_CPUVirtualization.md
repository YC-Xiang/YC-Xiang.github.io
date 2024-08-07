---
title: Operating Systems Three Easy Pieces(OSTEP) - CPU Virtualization
date: 2024-03-21 14:22:28
tags:
- OSTEP
categories:
- Book
---

这篇文章是阅读OSTEP 3~11的笔记，主要讲的是Virtualization中Virtualize CPU的部分。

# Chapter4 Process

## 4.2 Process API

一般OS会提供以下的进程API来操作进程：

- **Create**: 创建进程。
- **Destory**: 结束进程。
- **Wait**: Wait a process to stop running. 等待进程结束。
- **Miscellaneous Control**: Suspend/Resume... 休眠，唤醒等等。
- **Status**: 查看进程状态。

## 4.3 Process Creation

1. 首先OS将存储在disk or SSD的program程序加载进memory内存。
这边有两种方式，一种是在运行前把code和static data全部加载进内存。现代操作系统一般会使用第二种方式，**懒加载**，只加载即将使用的code和data。
2. 分配栈。
3. 分配堆。
4. 分配三个文件描述符，标准输入0，标准输出1，错误2。

## 4.4 Process Status

进程的状态有：

- **Running**: 正在使用CPU执行指令。
- **Ready**: 进程就绪态。
- **Blocked**: 比如进程在和disk IO交互，这时会把CPU让出给其他进程使用，进入阻塞态。

![](https://xyc-1316422823.cos.ap-shanghai.myqcloud.com/20240321160042.png)

## 4.5 Data Struct

PCB, Process Control Block 用来描述进程的数据结构。

参考xv6中描述进程的数据结构：

```c
struct proc {
	char *mem; // Start of process memory
	uint sz; // Size of process memory
	char *kstack; // Bottom of kernel stack
	// for this process
	enum proc_state state; // Process state
	int pid; // Process ID
	struct proc *parent; // Parent process
	void *chan; // If !zero, sleeping on chan
	int killed; // If !zero, has been killed
	struct file *ofile[NOFILE]; // Open files
	struct inode *cwd; // Current directory
	struct context context; // Switch here to run process
	struct trapframe *tf; // Trap frame for the current interrupt
};
```

# Chapter5 Process API

## 5.1 fork() system call

`pid_t fork(void)`

fork系统调用用来创建进程。子进程返回0，父进程返回子进程PID。

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

int main(int argc, char *argv[]) {
	printf("hello (pid:%d)\n", (int) getpid());
	int rc = fork();
	if (rc < 0) {
		// fork failed
		fprintf(stderr, "fork failed\n");
		exit(1);
	} else if (rc == 0) {
		// child (new process)
		printf("child (pid:%d)\n", (int) getpid());
	} else {
		// parent goes down this path (main)
		printf("parent of %d (pid:%d)\n", rc, (int) getpid());
}
return 0;
}
```

```shell
prompt> ./p1
hello (pid:29146)
parent of 29147 (pid:29146) # 这条和下面一条出现顺序随机
child (pid:29147)
prompt>
```

## 5.2 wait() system call

`pid_t wait(int *wstatus)`

wait系统调用会block等待子进程结束。`wstatus`可以传入NULL，也可以传入一个指针，通过进一步其他的API来获取子进程状态。

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <sys/wait.h>
int main(int argc, char *argv[]) {
	printf("hello (pid:%d)\n", (int) getpid());
	int rc = fork();
	if (rc < 0) { // fork failed; exit
		fprintf(stderr, "fork failed\n");
		exit(1);
	} else if (rc == 0) { // child (new process)
		printf("child (pid:%d)\n", (int) getpid());
	} else { // parent goes down this path
		int rc_wait = wait(NULL);
		printf("parent of %d (rc_wait:%d) (pid:%d)\n", rc, rc_wait, (int) getpid());
	}
return 0;
}
```

```sh
prompt> ./p2
hello (pid:29266)
child (pid:29267) # 这条和吓一条顺序是确定的
parent of 29267 (rc_wait:29267) (pid:29266)
prompt>
```

## 5.3 exec() system call

`exec()`系列系统调用，直接在当前进程加载另一个program, 运行另一个进程，不返回。

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <string.h>
#include <sys/wait.h>

int main(int argc, char *argv[]) {
	printf("hello (pid:%d)\n", (int) getpid());
	int rc = fork();
	if (rc < 0) { // fork failed; exit
		fprintf(stderr, "fork failed\n");
		exit(1);
	} else if (rc == 0) { // child (new process)
		printf("child (pid:%d)\n", (int) getpid());
		char *myargs[3];
		myargs[0] = strdup("wc"); // program: "wc"
		myargs[1] = strdup("p3.c"); // arg: input file
		myargs[2] = NULL; // mark end of array
		execvp(myargs[0], myargs); // runs word count
		printf("this shouldn’t print out");
	} else { // parent goes down this path
		int rc_wait = wait(NULL);
		printf("parent of %d (rc_wait:%d) (pid:%d)\n", rc, rc_wait, (int) getpid());
	}
	return 0;
}

```

```sh
prompt> ./p3
hello (pid:29383)
child (pid:29384)
29 107 1030 p3.c
parent of 29384 (rc_wait:29384) (pid:29383)
prompt>
```

## Others

`kill()`, `signal()`, `pipe()`

# Chapter6 Limited Direct Execution

## 6.1 Problem#1 Restricted Operations

User space要与kernel space隔离，通过system call的方式来访问硬件。

OS启动，以及user程序system call与kernel交互流程：

![](https://xyc-1316422823.cos.ap-shanghai.myqcloud.com/20240321175447.png)

## 6.2 Problem#2 Switching Between Processes

- Cooperative Approach：协作式，等process自己主动交出CPU控制权。
- Non-cooperative Approach: 抢占式，OS通过timer interrupt，给每个process一定的时间片执行，到了timer的时间就要交出CPU控制权。

**Context Switch**

进程A和进程B进行切换的上下文交换过程：

> 注意每个进程都有自己的kernel stack。

![](https://xyc-1316422823.cos.ap-shanghai.myqcloud.com/20240322113305.png)

## Summary

1. CPU跑OS需要支持**user mode**和**kernel mode**。
2. user mode使用**system call** trap into kernel。
3. kernel启动过程中准备好了**trap table**。
4. OS完成system call后，通过**return-from-trap**指令返回user code。
5. kernel利用**timer interrupt**来防止一个用户进程一直占用CPU。
6. 进程间交换需要**context switch**。

# Chapter7 Scheduling: Introduction

几个衡量性能的指标：

转换时间=完成时间-到达时间  
$T_{turnaround}=T_{completion}-T_{arrival}$

响应时间=开始执行时间-到达时间  
$T_{response}=T_{firstrun}-T_{arrival}$

## 7.3 FIFO

先进先出原则，如果进程一起到来，按先后顺序执行。

![](https://xyc-1316422823.cos.ap-shanghai.myqcloud.com/20240322170741.png)

存在的问题是，如果前面的进程运行时间长，平均的turnaround时间就会变得很长：

![](https://xyc-1316422823.cos.ap-shanghai.myqcloud.com/20240322170605.png)

## 7.4 Shortest Job first(SJF)

先执行时间短的进程。

![](https://xyc-1316422823.cos.ap-shanghai.myqcloud.com/20240322170759.png)

存在的问题是，如果几个进程不是同时到来，先执行到时间长的进程，仍然有和FIFO调度一样的问题：

![](https://xyc-1316422823.cos.ap-shanghai.myqcloud.com/20240322170536.png)

## 7.5 Shortest Time-to-Completion First(STCF)

在SJF调度上加入抢占式机制。当有新进程到来时，调度器会判断谁的执行时间更短，来执行时间更短的进程。

![](https://xyc-1316422823.cos.ap-shanghai.myqcloud.com/20240322170714.png)

## 7.7 Round Robin

Response time比前面的调度算法都好。

每个进程执行一段时间后切换。要考虑context switch的消耗，选择合适的时间片。

![](https://xyc-1316422823.cos.ap-shanghai.myqcloud.com/20240322171956.png)

## 7.8 Incorporating I/O

执行IO的时候，调度别的进程。

![](https://xyc-1316422823.cos.ap-shanghai.myqcloud.com/20240322174407.png)

# Chapter8 Sheduling: The Multi-Level Feedback Queue(MLFQ)

MLFQ算法会维护一系列**Queues**, 拥有不同的优先级。

- **Rule1**: Priority(A)>Priority(B), 运行进程A.
- **Rule2**: Priority(A)=Priority(B), A和B以RR(Round Robin)规则运行.

![](https://xyc-1316422823.cos.ap-shanghai.myqcloud.com/20240323151420.png)

- **Rule3**: 一个进程刚到来时，属于最高优先级。
- **Rule4a**: 如果进程用完了自己的allotment, 会降低一个优先级。
- **Rule4b**: 如果进程在用完allotment前放弃了CPU(比如进行IO操作)，会停留在当前优先级，allotment重置。(Figure 8.3b)

Examples:

![](https://xyc-1316422823.cos.ap-shanghai.myqcloud.com/20240323152425.png)

![](https://xyc-1316422823.cos.ap-shanghai.myqcloud.com/20240323152443.png)

**目前为止MLFQ存在的问题**

- Starvation饥饿问题。如果有太多短时的进程，运行时间长的进程会拿不到CPU。
- Game the scheduler欺骗调度器。一个恶意的用户程序，可以在每次即将用完allotment时，进行一次IO操作，这样又可以停留在最高优先级了。

## Priority boost

为了解决饥饿问题，增加第五条规则：

- **Rule5**: 经过固定时间S, 把所有进程的优先级都调到最高。

这个时间S也称作voo-doo constants，比如把voo-doo constants设置为100ms:

![](https://xyc-1316422823.cos.ap-shanghai.myqcloud.com/20240323155652.png)

## Better Accounting

为了解决Game the scheduler的问题，修改Rule 4a和4b，不再是计算单次使用CPU的alloment，而是：

- **Rule4**: 当一个进程在某一优先级的总时间用完后，降低一个优先级。

![](https://xyc-1316422823.cos.ap-shanghai.myqcloud.com/20240323161225.png)

## Tuning MLFQ

一些参数是可以调整的，Queue的数量，time slice，allotment，priority boost time。

一种可以优化的做法是越高优先级的队列，time slice越短。

![](https://xyc-1316422823.cos.ap-shanghai.myqcloud.com/20240324095045.png)

# Chapter9 Scheduling: Proportional Share

比例份额调度(proportinal-share)，也称公平份额调度(fair-share)。

## 9.2 Lottery Scheduling

每个进程分配一个Ticket，生成一个位于0\~sum_tickets随机数winner，如下图如果winner位于0\~99，进程A执行; 100~149，进程B执行; 150~400，进程C执行。

![](https://xyc-1316422823.cos.ap-shanghai.myqcloud.com/20240325214031.png)

实现代码：

```c
// counter: used to track if we’ve found the winner yet
int counter = 0;

// winner: call some random number generator to
// get a value >= 0 and <= (totaltickets - 1)
int winner = getrandom(0, totaltickets);

// current: use this to walk through the list of jobs
node_t *current = head;
while (current) {
counter = counter + current->tickets;
if (counter > winner)
break; // found the winner
current = current->next;
}
// ’current’ is the winner: schedule it...
```

Lottery scheduling存在的问题有，如果job length很短的话会存在不公平的现象。还有一个是如何分配tickets。参考9.4和9.5章节。

## 9.6 Stride Scheduling

仍然给A,B,C分别分配tickets100,50,250，在Stride scheduling中需要一个总目标数，比如10000，用1000计算A,B,C的步长，10000/100=100, 10000/50=200, 10000/250=40。

Basic idea: at any given time, pick the process to run that has the lowest pass value so far; when you run a process, increment its pass counter by its stride.

```c
curr = remove_min(queue); // pick client with min pass
schedule(curr); // run for quantum
curr->pass += curr->stride; // update pass using stride
insert(queue, curr); // return curr to queue
```

![](https://xyc-1316422823.cos.ap-shanghai.myqcloud.com/20240325215738.png)

Pass Value相同时随机选择进程Run。

Stride Scheduling存在的问题有，如果新来一个进程，那么这个进程的pass value会是0，会持续运行该进程，占据CPU一段时间。

## 9.7 The Linux Completely Fair Scheduler(CFS)

所有竞争的进程公平地分配CPU使用。

每个进程运行会累计**virtual runtime**, CPU会选择vruntime最小的进程来run。

存在的问题有，如果CPU切换太快，context switch对系统的性能损耗太大。如果CPU切换太慢，进程间的公平性又会降低。

CFS算法为了解决上述问题，提供了一些控制参数：

- **sched_latency**: context switch之前进程应该跑多久，典型值为48ms。如果此时有n个进程一起在running，sched_latency=48/n。如下图所示，一开始有4个进程一起跑，sched_latency比较短，后面只有两个进程在跑，sched_latency变长。

![](https://xyc-1316422823.cos.ap-shanghai.myqcloud.com/20240326153422.png)

如果有太多进程一起在running，那么sched_latency就会被分的很小，为了解决这个问题，提出了第二个控制参数：

- **min_granularity**: 典型值6ms。进程最短的运行时间，不能再被分割。

### Weighting(Niceness)

Unix系统可以通过修改**nice**值来调整进程权重，可选值为-20~+19。

根据权重再调整进程的time_slice。

![](https://xyc-1316422823.cos.ap-shanghai.myqcloud.com/20240326164317.png)

n为总进程数，k为当前进程。weight：

![](https://xyc-1316422823.cos.ap-shanghai.myqcloud.com/20240326164432.png)

vruntime计算也需要调整：

![](https://xyc-1316422823.cos.ap-shanghai.myqcloud.com/20240326164849.png)

### Using Red-Black Trees

进程用链表查找太慢了，CFS使用红黑树来管理正在running的进程。
