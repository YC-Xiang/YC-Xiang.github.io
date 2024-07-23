---
date: 2024-05-08T17:13:10+08:00
title: 'Zephyr -- WorkQueue'
tags:
- Zephyr
categories:
- Zephyr OS
---

工作队列是一个内核对象，它使用专用线程以先进先出的方式处理工作项。

WorkQueue通常被**中断**和**高优先级线程**用于把一些低优先度的任务放到低优先级的线程中执行。相当于一种中断下半部的实现，用来节省时间。

# Delayable work

首先启动一个timeout定时器，经过指定时间后才会把work item放入work queue。

# Triggered work

# System Workqueue

Kernel定义了一个全局系统工作队列，适用于任何应用程序或者kernel code。

尽量使用系统的work queue节省内存，除非满足不了需求，比如需要在work queue中增加blocking的操作，这样就可以新开一个work queue。

# How to Use Workqueues

## Define a Workqueue

定义自己的work queue:

```c
#define MY_STACK_SIZE 512
#define MY_PRIORITY 5

K_THREAD_STACK_DEFINE(my_stack_area, MY_STACK_SIZE);

struct k_work_q my_work_q;

k_work_queue_init(&my_work_q);

k_work_queue_start(&my_work_q, my_stack_area,
                   K_THREAD_STACK_SIZEOF(my_stack_area), MY_PRIORITY,
                   NULL);
```

## Sumbit a Work Item

Work Item用`k_work`来抽象，需要`k_work_init()`初始化，`k_work_submit()`加入**系统**工作队列或`k_work_submit_to_queue()`提交到指定的工作队列。

```c
struct device_info {
    struct k_work work;
    char name[16]
} my_device;

void my_isr(void *arg)
{
    ...
    if (error detected) {
        k_work_submit(&my_device.work);
    }
    ...
}

void print_error(struct k_work *item)
{
    struct device_info *the_device =
        CONTAINER_OF(item, struct device_info, work);
    printk("Got error on device %s\n", the_device->name);
}

/* initialize name info for a device */
strcpy(my_device.name, "FOO_dev");

/* initialize work item for printing device's error messages */
k_work_init(&my_device.work, print_error);
```

还有一些实用的API：

`k_work_busy_get()`: 查看work item状态，返回0表示还没scheduled, submitted, being executed。  
`k_work_is_pending()`: 返回true如果work scheduled, queued or running。  
`k_work_flush()`: 线程调用用来block等待work完成，如果work不是pending状态，立即返回。  
`k_work_cancel()`: 尝试阻止work开始执行，可能会失败。  
`k_work_cancel_sync()`: 通常用在k_work_cancel（在中断中调用，因为不是blocking api）后面，退出中断后这个函数会blocking来确保取消完成。

## Scheduling a Delayable Work Item

`k_work_schedule()`: 可以用来设定一个工作项在经过一定的延迟后执行。如果在调用这个函数时，工作项已经被安排在队列中（已经过了timeout等待延时），则它的行为取决于工作的状态：

如果该工作项当前已经在队列中等待执行，该调用将对现有安排没有影响。  
如果工作项不在队列中，k_work_schedule() 将根据指定的延迟时间安排工作。

`k_work_reschedule()`: 用于重新安排一个工作项。确保设定的延迟是从最近的调用开始计算的。主要用途是重置工作项的计时器：

如果工作项已经计划执行（无论是正在延迟阶段还是已经在队列中等待执行），k_work_reschedule() 会取消当前的计划并重新根据新的延迟时间安排工作。  
如果该工作项当前未被计划，则功能上和 k_work_schedule() 相同，会将其安排在将来某个时刻执行。

## Avoid Race Conditions

如果包含work item的结构体parent由多个线程或中断共享，比如这个线程sumit work item，工作队列中会改变parent->flag，该线程也会改变parent->flag，其他线程也可能需要访问该flag，这时候需要原子操作flag或者上锁保护。

```c
static void work_handler(struct work *work)
{
        struct work_context *parent = CONTAINER_OF(work, struct work_context, work_item);

        if (k_mutex_lock(&parent->lock, K_NO_WAIT) != 0) {
                /* NB: Submit will fail if the work item is being cancelled. */
                (void)k_work_submit(work);
                return;
        }

        /* do stuff under lock */
        k_mutex_unlock(&parent->lock);
        /* do stuff without lock */
}
```

# 源码分析

```c
struct k_work {
	/* Node to link into k_work_q pending list. */
	sys_snode_t node;

	/* The function to be invoked by the work queue thread. */
	k_work_handler_t handler;

	/* The queue on which the work item was last submitted. */
	struct k_work_q *queue;

	/* State of the work item.
	 *
	 * The item can be DELAYED, QUEUED, and RUNNING simultaneously.
	 *
	 * It can be RUNNING and CANCELING simultaneously.
	 */
	uint32_t flags;
};
```


```c
void k_work_init(struct k_work *work,
		  k_work_handler_t handler)
{
	// 设置中断handler
	*work = (struct k_work)Z_WORK_INITIALIZER(handler);
}

#define Z_WORK_INITIALIZER(work_handler) { \
	.handler = work_handler, \
}
```

work初始化，设置work handler。

</br>

```c
int k_work_busy_get(const struct k_work *work)
{
	k_spinlock_key_t key = k_spin_lock(&lock);
	int ret = work_busy_get_locked(work);

	k_spin_unlock(&lock, key);

	return ret;
}
```

这个函数就是返回k_work的state flag，包括running, canceling, queued, delayed, flushing。

</br>

```c
k_work_submit();
	k_work_submit_to_queue();
		z_work_submit_to_queue();
			submit_to_queue_locked();
		z_reschedule_unlocked(); // 主动调度其他线程

static int submit_to_queue_locked(struct k_work *work,
				  struct k_work_q **queuep)
{
	int ret = 0;

	if (flag_test(&work->flags, K_WORK_CANCELING_BIT)) {
		/* Disallowed */
		ret = -EBUSY;
	} else if (!flag_test(&work->flags, K_WORK_QUEUED_BIT)) {
		/* Not currently queued */
		ret = 1;

		/* If no queue specified resubmit to last queue.
		 */
		if (*queuep == NULL) {
			*queuep = work->queue;
		}

		/* If the work is currently running we have to use the
		 * queue it's running on to prevent handler
		 * re-entrancy.
		 */
		if (flag_test(&work->flags, K_WORK_RUNNING_BIT)) {
			__ASSERT_NO_MSG(work->queue != NULL);
			*queuep = work->queue;
			ret = 2;
		}

		int rc = queue_submit_locked(*queuep, work);

		if (rc < 0) {
			ret = rc;
		} else {
			flag_set(&work->flags, K_WORK_QUEUED_BIT);
			work->queue = *queuep;
		}
	} else {
		/* Already queued, do nothing. */
	}

	if (ret <= 0) {
		*queuep = NULL;
	}

	return ret;
}
```
