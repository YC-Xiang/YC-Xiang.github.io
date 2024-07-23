---
date: 2024-04-11T09:24:31+08:00
title: 'Zephyr -- Threads'
tags:
- Zephyr
categories:
- Zephyr OS
---

# Threads

`CONFIG_MULTITHREADING`打开Zephyr多线程功能。

Zephyr线程有下面几个关键的特性：

- Stack area。 线程的栈大小可修改。
- thread control block `k_thread`。用来保存线程的一些metadata。
- entry point function。线程开始执行运行的函数。
- scheduling priority。支持配置调度优先级。
- thread option。提供线程的一些特殊配置。
- execution mode。Supervisor/User mode。依赖于`CONFIG_USERSPACE`。

## Lifecycle

`k_thread_create()`: 创建线程。  
`k_thread_join()`: 阻塞等待线程终止。  
`k_thread_abort()`: 发生异常情况，线程可以由自己或其他线程来终结。  
`k_thread_suspend()`, `k_thread_resume()`: 线程suspend后只有通过resume才能重新调度。

## Thread States

![](https://xyc-1316422823.cos.ap-shanghai.myqcloud.com/20240411095150.png)

## Thread Stack objects

初始化线程栈相关属性，如果线程只在内核运行，用`K_KERNEL_STACK_XXX`。

如果是user space线程，用`K_THREAD_STACK_XXX`。

如果`CONFIG_USERSPACE`没打开，那么`K_THREAD_STACK`等于`K_KERNEL_STACK`。

## Thread Priorities

优先级数字越小，优先级越高。

cooperative thread可配置的优先级为 -`CONFIG_NUM_COOP_PRIORITIES`到-1。  
preemptible thread可配置的优先级为0到 (`CONFIG_NUM_PREEMPT_PRIORITIES`-1)。

可见cooperative thread的优先级肯定比preemptible thread高。

Cooperative thread是需要主动交出CPU控制权的，否则会一直执行该线程。  
Preemptible thread是根据时间片轮转调度的，会自动切换线程。

![](https://xyc-1316422823.cos.ap-shanghai.myqcloud.com/20240411101543.png)

## Meta-IRQ Priorities

// TODO:

## APIs

创建线程：

**方法1：**

```c
k_tid_t k_thread_create(struct k_thread *new_thread, k_thread_stack_t *stack,
			size_t stack_size, k_thread_entry_t entry,
			void *p1, void *p2, void *p3, int prio,
			uint32_t options, k_timeout_t delay)
```

- 线程栈必须使用 `K_THREAD_STACK_DEFINE` or `K_KERNEL_STACK_DEFINE`定义。
- 线程栈大小必须是传入给`K_THREAD_STACK` or `K_KERNEL_STACK`宏的大小。或者利用`K_THREAD_STACK_SIZEOF()`/`K_KERNEL_STACK_SIZEOF()`，对用`K_THREAD_STACK`/`K_KERNEL_STACK`创建线程返回的结构体。

e.g.

```c
#define MY_STACK_SIZE 500
#define MY_PRIORITY 5

extern void my_entry_point(void *, void *, void *);

K_THREAD_STACK_DEFINE(my_stack_area, MY_STACK_SIZE);
struct k_thread my_thread_data;

k_tid_t my_tid = k_thread_create(&my_thread_data, my_stack_area,
                                 K_THREAD_STACK_SIZEOF(my_stack_area),
                                 my_entry_point,
                                 NULL, NULL, NULL,
                                 MY_PRIORITY, 0, K_NO_WAIT);
```

**方法2：**

利用宏`K_THREAD_DEFINE`，在编译期定义。

```c
#define MY_STACK_SIZE 500
#define MY_PRIORITY 5

extern void my_entry_point(void *, void *, void *);

K_THREAD_DEFINE(my_tid, MY_STACK_SIZE,
                my_entry_point, NULL, NULL, NULL,
                MY_PRIORITY, 0, 0);
```

## Thread Options

上面的options可以传入的选项有（省略了一些不常用的）：

- `K_ESSENTIAL`: 表示这是基础线程，任何termination或aborting都会导致系统错误。
- `K_FP_REGS`: 线程使用CPU浮点计算，调度的时候会保存和恢复浮点计算寄存器组。
- `K_USER`: 如果`CONFIG_USERMODE` enable了，这个选项表示是用户级线程。
- `K_INHERIT_PERMS`: 如果`CONFIG_USERMODE` enable了，会继承父线程所有内核对象权限。

## Thread Custom Data

每个线程都有一个私有的32-bit数据，通过下面两个API读写。需要打开`CONFIG_THREAD_CUSTOM_DATA`开关。

`k_thread_custom_data_set()`  
`k_thread_custom_data_get()`

e.g.

```c
call_count = (uint32_t)k_thread_custom_data_get();
call_count++;
k_thread_custom_data_set((void *)call_count);
```

# 调度源码分析

Pend当前线程的API:

```c
int z_pend_curr(struct k_spinlock *lock, k_spinlock_key_t key,
	       _wait_q_t *wait_q, k_timeout_t timeout)
{
#if defined(CONFIG_TIMESLICING) && defined(CONFIG_SWAP_NONATOMIC)
	pending_current = _current;
#endif
	__ASSERT_NO_MSG(sizeof(sched_spinlock) == 0 || lock != &sched_spinlock);

	(void) k_spin_lock(&sched_spinlock);
	pend_locked(_current, wait_q, timeout); //如果timeout是K_FOREVER不会阻塞，直接调度到其他线程
	k_spin_release(lock);
	return z_swap(&sched_spinlock, key);
}
```

spin lock在单核系统中就是屏蔽一些HW中断。

```c
k_spin_lock();
	k.key = arch_irq_lock();

static ALWAYS_INLINE unsigned int arch_irq_lock(void)
{
	unsigned int key;
	unsigned int tmp;

	__asm__ volatile(
		"mov %1, %2;"
		"mrs %0, BASEPRI;"
		"msr BASEPRI_MAX, %1;"
		"isb;"
		: "=r"(key), "=r"(tmp)
		: "i"(_EXC_IRQ_DEFAULT_PRIO)
		: "memory");

	return key;
}
```

key是%0, tmp是%1, _EXC_IRQ_DEFAULT_PRIO是%2，该宏为(1<<5)。

首先将宏`_EXC_IRQ_DEFAULT_PRIO`传入`tmp`, 再将`BASEPRI`寄存器值传入`key`用于返回保存。  
最后把`tmp`中的值，即`_EXC_IRQ_DEFAULT_PRIO`写入``BASEPRI_MAX`寄存器，`BASEPRI_MAX`和`BASEPRI`是同一个寄存器，当指定`BASEPRI_MAX`表示只有在新值高于（即优先级数值更低）当前值的情况下，才会更新只更新`BASEPRI`。

系统不会处理大于等于BASEPRI设置的值的中断优先级的中断。

```c
/* On ARMv7-M and ARMv8-M Mainline CPUs, this function prevents regular
 * exceptions (i.e. with interrupt priority lower than or equal to
 * _EXC_IRQ_DEFAULT_PRIO) from interrupting the CPU. NMI, Faults, SVC,
 * and Zero Latency IRQs (if supported) may still interrupt the CPU.
 */
```

```c
pend_locked();
	add_to_waitq_locked(thread, wait_q);
	add_thread_timeout(thread, timeout);

static void add_to_waitq_locked(struct k_thread *thread, _wait_q_t *wait_q)
{
	unready_thread(thread); // 把当前thread从run queue中移除
	z_mark_thread_as_pending(thread); // 把thread_state设置为pending

	if (wait_q != NULL) {
		thread->base.pended_on = wait_q;
		z_priq_wait_add(&wait_q->waitq, thread); // 加入到waitq中
	}
}

static void add_thread_timeout(struct k_thread *thread, k_timeout_t timeout)
{
	if (!K_TIMEOUT_EQ(timeout, K_FOREVER)) {
		z_add_thread_timeout(thread, timeout);
	}
}
z_add_thread_timeout();
	z_thread_timeout();
		z_sched_wake_thread(thread, true); // 等待timeout之间后，唤醒当前线程
```

如果z_pend_curr传入的是NO_WAIT的话，当前线程会立刻进入睡眠，等待其他线程唤醒，并由其他线程设置return value。  
如果z_pend_curr传入的是一段时间，那么当前线程会进入睡眠建立一个timer，固定时间后唤醒，如果在睡眠期间别的线程完成了任务，会把return value设置为0并重新调度，否则超时唤醒timer,返回错误。

其他线程唤醒pending线程的过程：

```c
z_unpend_first_thread();
arch_thread_return_value_set();
z_ready_thread();
z_reschedule();
```
