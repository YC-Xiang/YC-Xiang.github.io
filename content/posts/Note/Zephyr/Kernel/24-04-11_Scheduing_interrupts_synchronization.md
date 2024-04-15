---
date: 2024-04-11T09:24:31+08:00
title: 'Scheduing, Interrupts and Synchronization'
tags:
- Zephyr
categories:
- Zephyr OS
---

# Threads

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

cooperative thread可配置的优先级为 `CONFIG_NUM_COOP_PRIORITIES`到-1。

preemptible thread可配置的优先级为0到 `CONFIG_NUM_PREEMPT_PRIORITIES`到1。

可见cooperative thread的优先级肯定比preemptible thread高。

![](https://xyc-1316422823.cos.ap-shanghai.myqcloud.com/20240411101543.png)

## Meta-IRQ Priorities

// TODO:

## Thread Options

`K_ESSENTIAL`: 表示这是基础线程，任何termination或aborting都会导致系统

`K_SSE_REGS`

## APIs

```c
k_tid_t k_thread_create(struct k_thread *new_thread, k_thread_stack_t *stack,
			size_t stack_size, k_thread_entry_t entry,
			void *p1, void *p2, void *p3, int prio,
			uint32_t options, k_timeout_t delay)
```
