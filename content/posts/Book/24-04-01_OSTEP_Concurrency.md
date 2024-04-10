---
date: 2024-04-01T09:50:33+08:00
title: 'Operating Systems Three Easy Pieces(OSTEP) - Concurrency'
tags:
- OSTEP
categories:
- Book
---

# 26. Concurrency: Introduction

线程和进程类似，但多线程程序，共享address space和data。

多线程各自独享寄存器组，切换进程的时候需要**上下文切换**。

进程上下文切换的时候把上一个进程的寄存器组保存到PCB(Process Control Block)中，线程切换需要定义一个或更多新的结构体TCBs(Thread Control Blocks)来保存每个线程的状态。

线程切换Page table不用切换，因为线程共享地址空间。

线程拥有自己的栈。

![](https://xyc-1316422823.cos.ap-shanghai.myqcloud.com/20240401101216.png)

## 26.1 Why use threads?

- Parallelism。单线程程序只能利用一个CPU核，多线程程序可以并行利用多个CPU核，提高效率。

- 防止I/O操作导致程序block。多线程程序可以在一个线程执行I/O操作的时候，其他线程继续利用CPU。

## 26.2~26.4 Problem of Shared data

创建两个线程，分别对全局变量counter加1e7, 最后结果会不等于2e7。

```c
#include <stdio.h>
#include <pthread.h>
#include "common.h"
#include "common_threads.h"

static volatile int counter = 0;

void *mythread(void *arg)
{
	printf("%s: begin\n", (char *) arg);
	int i;
	for (i = 0; i < 1e7; i++)
		counter = counter + 1;
	printf("%s: done\n", (char *) arg);
	return NULL;
}

int main(int argc, char *argv[])
{
	pthread_t p1, p2;
	printf("main: begin (counter = %d)\n", counter);
	Pthread_create(&p1, NULL, mythread, "A");
	Pthread_create(&p2, NULL, mythread, "B");
	// join waits for the threads to finish
	Pthread_join(p1, NULL);
	Pthread_join(p2, NULL);
	printf("main: done with both (counter = %d)\n", counter);
	return 0;
}
```

`pthread_create`: 创建线程。
`pthread_join`: 阻塞等待线程终止。

假设变量counter的地址为0x8049a1c，值为50, `counter = counter + 1`的汇编代码为：

```c
mov 0x8049a1c, %eax
add $0x1, %eax
mov %eax, 0x8049a1c
```

假设Thread1在执行到第二行`add $0x1, %eax`的时候被切换到Thread2, Thread2执行完，将51写入内存0x8049a1c中，再切回Thread1。这时因为context switch的原因，Thread1也不会重新执行第一行，从内存重新获取值，而是用之前保存在`%eax`中的51，再次写入内存0x9049a1c。这样发生了错误。

![](https://xyc-1316422823.cos.ap-shanghai.myqcloud.com/20240401162026.png)

这种情况被称作**race condition**或**data race**。会导致**race condition**的代码段(并行访问共享数据)被称作**critical section**临界区。

解决这种问题的方法是**mutual exclusion**互斥。一个线程在访问共享数据的时候，另一个线程无法访问。

## 26.5 The wish For Atomicity

**synchronization primitives** 同步原语。

# 27 Thread API

## 27.1 Thread Creation

```c
#include <pthread.h>
int pthread_create(pthread_t *thread, const pthread_attr_t *attr, void *(*start_routine)(void*), void *arg);
```

`pthread_t thread`: 需要传入一个`pthread_t`结构体地址。

`const pthread_attr_t *attr`: 用来指定一些属性，比如栈大小，线程调度优先级等。还需要调用`pthread_attr_init()`，使用默认属性直接传入NULL。

`(*start_routine)(void*)`: 线程调用的函数指针，函数名为start_routine。

`void *arg`: 传入函数指针的形参。

</br>

如果函数需要的形参是一个int：

`int pthread_create(..., void *(*start_routine)(int), int arg);`

如果函数的返回值是int:

`int pthread_create(..., int (*start_routine)(void *), void *arg);`

## 27.2 Thread Completion

```c
int pthread_join(pthread_t thread, void **value_ptr);
```

value_ptr: 指向线程的return value，不关注的话传入NULL。

</br>

永远不要在线程执行函数中返回在栈上分配的变量，比如：

```c
void *mythread(void *arg) {
	myarg_t *args = (myarg_t *) arg;
	printf("%d %d\n", args->a, args->b);
	myret_t oops; // ALLOCATED ON STACK: BAD!
	oops.x = 1;
	oops.y = 2;
	return (void *) &oops;
}
```

## 27.3 Locks

```c
int pthread_mutex_lock(pthread_mutex_t *mutex);
int pthread_mutex_unlock(pthread_mutex_t *mutex);
```

初始化锁有两种方式：

```c
pthread_mutex_t lock = PTHREAD_MUTEX_INITIALIZER;
```

或

```c
int rc = pthread_mutex_init(&lock, NULL); // 第二个参数为可选的属性
assert(rc == 0); // always check success!
```

使用方法：

```c
pthread_mutex_lock(&lock);
x = x + 1; // or whatever your critical section is
pthread_mutex_unlock(&lock);
```

用完后需要调用destory函数：`pthread mutex destroy()`。

一个线程调用`pthread_mutex_lock()`后，其他线程如果要访问临界区，都会阻塞等待。

下面两个API，第一个trylock会尝试获取一次锁，如果没得到，直接返回failure，不会阻塞等待。第二个timedlock会等待指定的一段时间后再返回failure。

```c
int pthread_mutex_trylock(pthread_mutex_t *mutex);
int pthread_mutex_timedlock(pthread_mutex_t *mutex, struct timespec *abs_timeout);
```

## 27.4 Condition Variables

条件变量。

```c
int pthread_cond_wait(pthread_cond_t *cond, pthread_mutex_t *mutex);
int pthread_cond_signal(pthread_cond_t *cond);
```

线程使用条件变量的时候首先需要拥有锁。
下面这段示例code，拿到锁之后，会等待全局变量ready非0，否则会阻塞在`Pthread_cond_wait(&cond, &lock)`。

```c
pthread_mutex_t lock = PTHREAD_MUTEX_INITIALIZER;
pthread_cond_t cond = PTHREAD_COND_INITIALIZER;

Pthread_mutex_lock(&lock);
while (ready == 0)
	Pthread_cond_wait(&cond, &lock);
Pthread_mutex_unlock(&lock);
```

某个其他线程来唤醒上面的线程：

```c
Pthread_mutex_lock(&lock);
ready = 1;
Pthread_cond_signal(&cond);
Pthread_mutex_unlock(&lock);
```

阻塞在`Pthread_cond_wait(&cond, &lock)`的线程会让出锁进入sleep，让其他线程能够获得锁来唤醒。

在线程之间需要使用条件变量，而不要使用简单的全局flag来进行逻辑判断。

# 28. Locks

## 28.2 Pthread Locks

POSIX库中用**mutex**来表示锁。

## 28.5 Controlling interrupts

在单处理器的系统上，可以通过关闭中断来达到锁的效果。类似如下：

```c
void lock() {
	DisableInterrupts();
}
void unlock() {
	EnableInterrupts();
}
```

缺点有

1. 需要给线程特权操作，这样对系统不安全，恶意线程可以一直占有锁不放开，这样其他线程会全部阻塞。
2. 在多核系统上无效，一个CPU核禁止中断，其他CPU核仍然可以访问临界区。
3. 禁止中断会导致这段时间内可能有用的中断会丢失。

## 28.6 A Failed Attempt: Just Using Loads/Stores

如果没有硬件的支持，想通过flag来达到锁的目的，例如：

```c
typedef struct __lock_t { int flag; } lock_t;

void init(lock_t *mutex) {
	// 0 -> lock is available, 1 -> held
	mutex->flag = 0;
}

void lock(lock_t *mutex) {
	while (mutex->flag == 1) // TEST the flag
	; // spin-wait (do nothing)
	mutex->flag = 1; // now SET it!
}

void unlock(lock_t *mutex) {
	mutex->flag = 0;
}
```

一个线程执行`lock()`后，全局`mutex->flag`被置1，其他所有线程试图通过`lock()`获取锁时，会阻塞spin-wait。

这样的实现也会有并发的问题，在while判断通过后切换了线程，这样两个线程可能同时都对flag置1：

![](https://xyc-1316422823.cos.ap-shanghai.myqcloud.com/20240408213043.png)

## 28.7 Spin Locks with Test-And-Set

接下来几小节会介绍几种实现锁的硬件支持指令，包括：

- Test-And-Set
- Compare-And-Swap
- Load-Linked and Store-Conditional
- Fetch-and-Add

硬件会支持一种TestAndSet**原子交换**指令，读出和写入内存地址是原子操作，不会被其他进程打断。一开始锁的flag为0，第一个线程调用TestAndSet会返回0，并且把flag置1，可以跳出循环。后面的线程调用TestAndSet会返回1，不断spin。

```c
int TestAndSet(int *old_ptr, int new) {
	int old = *old_ptr; // fetch old value at old_ptr
	*old_ptr = new; // store ’new’ into old_ptr
	return old; // return the old value
}
```

因此**自旋锁**利用原子交换，基本实现如下：

```c
typedef struct __lock_t {
	int flag;
} lock_t;

void init(lock_t *lock) {
	// 0: lock is available, 1: lock is held
	lock->flag = 0;
}

void lock(lock_t *lock) {
	while (TestAndSet(&lock->flag, 1) == 1)
	; // spin-wait (do nothing)
}

void unlock(lock_t *lock) {
	lock->flag = 0;
}
```

在lock的时候使用原子交换指令。

## 28.9 Compare-And-Swap

另一种硬件原语Compare-And-Swap。判断锁的flag是否和预期相同，相同则更新值。

```c
int CompareAndSwap(int *ptr, int expected, int new) {
	int original = *ptr;
	if (original == expected)
	*ptr = new;
	return original;
}
```

利用该硬件原语，同样可以实现自旋锁：

```c
void lock(lock_t *lock) {
	while (CompareAndSwap(&lock->flag, 0, 1) == 1)
	; // spin
}
```

## 28.10 Load-Linked and Store-Conditional

一些平台提供了实现临界区的指令，比如MIPS的Load-Linked and Store-Conditional。其他平台也有类似的指令。

Load-Linked(LL)链接加载和普通load指令没什么不同。

Store-Conditional(SC)条件存储是只有上一次链接加载的地址在期间没有更新过，才能成功。

```c
int LoadLinked(int *ptr) {
	return *ptr;
}

int StoreConditional(int *ptr, int value) {
	if (no update to *ptr since LL to this addr) {
		*ptr = value;
		return 1; // success!
	} else {
		return 0; // failed to update
	}
}
```

利用LL和SC实现锁：

```c
void lock(lock_t *lock) {
	while (1) {
		while (LoadLinked(&lock->flag) == 1)
			; // spin until it’s zero
		if (StoreConditional(&lock->flag, 1) == 1)
			return; // if set-to-1 was success: done
			// otherwise: try again
	}
}

void unlock(lock_t *lock) {
	lock->flag = 0;
}
```

## 28.11 Fetch-And-Add

```c
int FetchAndAdd(int *ptr) {
	int old = *ptr;
	*ptr = old + 1;
	return old;
}
```

利用该硬件原语，可以实现一种**Ticket Lock**，一种公平的锁。

```c
typedef struct __lock_t {
	int ticket;
	int turn;
} lock_t;

void lock_init(lock_t *lock) {
	lock->ticket = 0;
	lock->turn = 0;
}

void lock(lock_t *lock) {
	int myturn = FetchAndAdd(&lock->ticket);
	while (lock->turn != myturn)
	; // spin
}
void unlock(lock_t *lock) {
	lock->turn = lock->turn + 1;
}

```

## 28.14 Using Queues: Sleeping instead of Spinning

前面的方法让拿不到锁的线程一直自旋，或者直接让出CPU，都会浪费CPU，也不能防止饿死。


提供一种将等待锁的线程加入队列的方法。

```c
typedef struct __lock_t {
	int flag;
	int guard;
	queue_t *q;
} lock_t;

void lock_init(lock_t *m) {
	m->flag = 0;
	m->guard = 0;
	queue_init(m->q);
}

void lock(lock_t *m) {
	while (TestAndSet(&m->guard, 1) == 1)
		; //acquire guard lock by spinning
	if (m->flag == 0) {
		m->flag = 1; // lock is acquired
		m->guard = 0;
	} else {
		queue_add(m->q, gettid());
		m->guard = 0;
		park();
	}
}

void unlock(lock_t *m) {
	while (TestAndSet(&m->guard, 1) == 1)
		; //acquire guard lock by spinning
	if (queue_empty(m->q))
		m->flag = 0; // let go of lock; no one wants it
	else
		unpark(queue_remove(m->q)); // hold lock
	// (for next thread!)
	m->guard = 0;
}
```

`park()`: 让当前的线程进入sleep。
`unpark()`: 唤醒指定TID的线程。

首先第一个线程调用`lock()`，此时`m->guard==0`，因此不会自旋等待，会执行

```c
if (m->flag == 0)
{
	m->flag = 1; // lock is acquired
	m->guard = 0;
}
```

其他线程调用`lock()`，如果第一个进程执行了一半被中断，比如刚好执行到上面的`m->flag = 1`，而没有执行`m->guard = 0`, 那么其他线程会在`TestAndSet()`中自旋等待一小段时间，这只有几条指令的时间，因此自旋等待一会不是问题。

其他线程正常的流程会直接调用到:

```c
else
{
	queue_add(m->q, gettid());
	m->guard = 0;
	park();
}
```

把当前线程加入队列，并且进入sleep，因为进入的时候`TestAndSet()`把guard置为1了，其他线程在等待，在进入sleep前还需要把`m->guard`置0。

</br>

在当前进程用完锁后会调用`unlock()`, 如果此时还有其他线程正在等待锁，当前线程会执行到：

```c
	unpark(queue_remove(m->q)); // hold lock
```

直接将队列中下一个可以拥有锁的线程唤醒，因为此时锁的flag`m->flag`仍然为1，没有清过，所以此时拥有锁的线程为被唤醒的线程。

</br>

上面代码存在的一个问题有，如果第二个线程在即将调用`park()`前，发生了线程切换，切换到第一个线程，该线程释放了锁，再切回第二个线程调用`park()`，这样第二个线程可能永远睡下去。

为了解决这个问题，引入一个系统调用`setpark()`，表示自己即将要park，如果发生了线程调度，其他线程调用了unpark，那么该park会立即返回，而不会进入sleep。

```c
queue_add(m->q, gettid());
m->guard = 0;
park();

//修改为

queue_add(m->q, gettid());
setpark(); // new code
m->guard = 0;
```

## 28.16 Two-Phase Locks

真实的操作系统如Linux中，mutex lock一般会先自旋等待一段时间，如果没有获得锁则进入sleep。