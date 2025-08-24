## 3.2 进程描述符及任务结构

内核把进程的列表存放在任务队列 task list 的双向循环链表中。链表中的每一项都是类型为 struct `task_struct` 的结构体，称为进程描述符，定义在`linux/sched.h`中。

### 3.2.1 分配进程描述符

linux 通过 slab 分配器分配 task_struct 结构，在栈底创建一个新的结构体 struct `thread_info`, 在`<asm/thread_info.h>`中定义。

### 3.2.2 进程描述符的存放

内核通过 PID 来标识每个进程，PID 类型为 `pid_t`, 其实就是 int 类型。

### 3.2.3 进程状态

struct `task_struct` 的 __state 成员表示进程的状态，只有五种状态：

- TASK_RUNNING: 进程可执行，或者正在执行，或者在运行队列中等待执行。
- TASK_INTERRUPTIBLE: 进程正在睡眠 (被阻塞), 等待某些条件的达成。
- TASK_UNINTERRUPTIBLE: 不可中断的睡眠状态
- TASK_TRACED: 被其他进程跟踪的进程
- TASK_STOPPED: 进程停止执行

![](https://xyc-1316422823.cos.ap-shanghai.myqcloud.com/20250824141117.png)

### 3.2.4 设置当前进程状态

`set_task_state(task, state)` 函数用于设置进程的状态。

`set_current_state(state)` 函数用于设置当前进程的状态。

> 最新的 linux kernel 中没有 set_task_state() 了。

### 3.2.5 进程上下文

### 3.2.6 进程家族树

Unix 系统的进程之间存在一个明显的继承关系。所有进程都是 PID 为 1 的 init 进程的后代。

每个进程必有一个父进程。每个进程也可以拥有 0 个或多个子进程。保存在 struct task_struct 的 `parent` 和 `children` 成员中。

## 3.3 进程创建

`fork()` 和 `exec()`

首先通过 `fork()` 拷贝当前进程创建一个子进程。

// TODO:

## 3.4 线程在 Linux 中的实现

Linux 把所有线程都当做进程来实现。内核没有准备特别的调度算法或定义特别的数据结构来表示线程。相反，线程仅仅被视为一个与其他进程共享某些资源的进程。

每个线程也拥有自己的 struct `task_struct`, 所以在内核中，它看起来就像是一个普通的进程。

### 3.4.1 创建线程

// TODO:

### 3.4.2 内核线程

内核经常需要在后台执行一些操作，这种任务可以通过内核线程完成。

内核线程和普通进程的主要区别在于，内核线程没有独立的地址空间 (指向地址空间的 mm 指针被设置为 NULL)。它们只在内核中运行，不会切换到用户空间去。

内核是通过从 kthreadd 内核进程中衍生出所有的内核线程的，创建内核线程的 api:

```c++
struct task_struct *kthread_create(int (*threadfn)(void *data), void *data, const char *namefmt, ...);
```

如果不通过 wake_up_process(), 则内核线程不会主动运行。

创建一个线程并让它运行起来，可以通过调用`kthread_run()`, 就是 kthread_create() 和 wake_up_process() 的组合。

内核线程启动后就一直运行直到调用`do_exit()`退出。或者内核的其他部分调用`kthread_stop()`退出，传递给 kthread_stop() 的参数为要退出的 task_struct.

## 3.5 进程终结

// TODO:
