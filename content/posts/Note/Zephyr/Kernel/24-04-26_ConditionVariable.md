---
date: 2024-04-26T11:06:31+08:00
title: 'Zephyr -- Condition Variables and Semaphores'
tags:
- Zephyr
categories:
- Zephyr OS
---

# Condition Variables

## Concept

条件变量基本上是一个线程队列，当某些执行状态不符合预期时，线程可以将自己放入该队列中。

`k_condvar_wait()`:

- Releases the last acquired mutex.
- Puts the current thread in the condition variable queue.

其他某个线程在更改所述状态时，可以唤醒一个（或多个）等待线程，通过`k_condvar_signal()` or `k_condvar_broadcast()`:

- Re-acquires the mutex previously released.
- Returns from k_condvar_wait().

## Implementation

### Defining a Condition Variable

```c++
struct k_condvar my_condvar;
k_condvar_init(&my_condvar); // 或
K_CONDVAR_DEFINE(my_condvar);
```

### Waiting on a Condition Variable

```c++
k_mutex_lock(&mutex, K_FOREVER);

/* block this thread until another thread signals cond. While
* blocked, the mutex is released, then re-acquired before this
* thread is woken up and the call returns.
*/
k_condvar_wait(&condvar, &mutex, K_FOREVER);
...
k_mutex_unlock(&mutex);
```

### Signaling a Condition Variable

```c++

k_mutex_lock(&mutex, K_FOREVER);

/*
* Do some work and fulfill the condition
*/
k_condvar_signal(&condvar);
k_mutex_unlock(&mutex);
```

## Zephyr源码分析

Init函数初始化条件变量的wait queue。

```c++
struct k_condvar {
	_wait_q_t wait_q;
};
```

```c++
int z_impl_k_condvar_init(struct k_condvar *condvar)
{
	z_waitq_init(&condvar->wait_q);

	return 0;
}
```

```c++
int z_impl_k_condvar_wait(struct k_condvar *condvar, struct k_mutex *mutex, k_timeout_t timeout)
{
	k_spinlock_key_t key;
	int ret;

	key = k_spin_lock(&lock);
	k_mutex_unlock(mutex);
	//将当前线程加入wait_q,
	ret = z_pend_curr(&lock, key, &condvar->wait_q, timeout);
	k_mutex_lock(mutex, K_FOREVER);

	return ret;
}
```

```c++
int z_impl_k_condvar_signal(struct k_condvar *condvar)
{
	k_spinlock_key_t key = k_spin_lock(&lock);

	// 唤醒wait_q优先级最高的线程
	struct k_thread *thread = z_unpend_first_thread(&condvar->wait_q);

	if (thread != NULL) {
		arch_thread_return_value_set(thread, 0);
		z_ready_thread(thread);
		// 重新调度被唤醒的这个线程
		z_reschedule(&lock, key);
	} else {
		k_spin_unlock(&lock, key);
	}

	return 0;
}
```

唤醒所有等待线程：
`int z_impl_k_condvar_broadcast(struct k_condvar *condvar)`

# Semaphores

## Concept

信号量有下面两个关键的属性：

A **count** that indicates the number of times the semaphore can be taken. A count of zero indicates that the semaphore is unavailable.

A **limit** that indicates the maximum value the semaphore’s count can reach.

## Implementation

### Defining a Semaphore

```c++
struct k_sem my_sem;
k_sem_init(&my_sem, 0, 1); // 或
K_SEM_DEFINE(my_sem, 0, 1);
```

### Giving a Semaphore

```c++
k_sem_give(&my_sem);
```

### Taking a Semaphore

```c++
if (k_sem_take(&my_sem, K_MSEC(50)) != 0) {
	printk("Input data not available!");
} else {
	/* fetch available data */
}
```

## Zephyr源码分析

```c++
struct k_sem {
	_wait_q_t wait_q;
	unsigned int count;
	unsigned int limit;

	sys_dlist_t poll_events;
};
```

```c++
int z_impl_k_sem_init(struct k_sem *sem, unsigned int initial_count,
		      unsigned int limit)
{
	/*
	 * Limit cannot be zero and count cannot be greater than limit
	 */
	CHECKIF(limit == 0U || limit > K_SEM_MAX_LIMIT || initial_count > limit) {
		return -EINVAL;
	}

	sem->count = initial_count;
	sem->limit = limit;

	z_waitq_init(&sem->wait_q);
#if defined(CONFIG_POLL)
	sys_dlist_init(&sem->poll_events);
#endif

	return 0;
}
```

```c++
int z_impl_k_sem_take(struct k_sem *sem, k_timeout_t timeout)
{
	int ret = 0;

	__ASSERT(((arch_is_in_isr() == false) ||
		  K_TIMEOUT_EQ(timeout, K_NO_WAIT)), "");

	k_spinlock_key_t key = k_spin_lock(&lock);

	// 成功拿到semaphore
	if (likely(sem->count > 0U)) {
		sem->count--;
		k_spin_unlock(&lock, key);
		ret = 0;
		goto out;
	}
	// timeout为NO_WAIT的话直接返回BUSY
	if (K_TIMEOUT_EQ(timeout, K_NO_WAIT)) {
		k_spin_unlock(&lock, key);
		ret = -EBUSY;
		goto out;
	}
	// 将当前线程移除kernel ready queue, 加入wait_q，进入pending状态等待其他线程唤醒。
	ret = z_pend_curr(&lock, key, &sem->wait_q, timeout);

out:
	return ret;
}
```

```c++
void z_impl_k_sem_give(struct k_sem *sem)
{
	k_spinlock_key_t key = k_spin_lock(&lock);
	struct k_thread *thread;
	bool resched = true;

	// 唤醒wait queue中优先级最高的线程
	thread = z_unpend_first_thread(&sem->wait_q);

	if (thread != NULL) {
		arch_thread_return_value_set(thread, 0);
		// 重新加入调度
		z_ready_thread(thread);
	} else {
		// 如果没有线程在等待semaphore，count++
		sem->count += (sem->count != sem->limit) ? 1U : 0U;
		resched = handle_poll_events(sem);
	}

	if (resched) {
		z_reschedule(&lock, key);
	} else {
		k_spin_unlock(&lock, key);
	}
}
```

唤醒所有等待线程，并将semaphore的count值设为0。  
`void z_impl_k_sem_reset(struct k_sem *sem)`

# Reference

[Zephyr线程阻塞和超时机制分析](https://lgl88911.github.io/2020/05/17/Zephyr%E7%BA%BF%E7%A8%8B%E9%98%BB%E5%A1%9E%E5%92%8C%E8%B6%85%E6%97%B6%E6%9C%BA%E5%88%B6%E5%88%86%E6%9E%90/)
