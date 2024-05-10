---
date: 2024-05-07T17:51:10+08:00
title: 'Zephyr -- Data Passing'
tags:
- Zephyr
categories:
- Zephyr OS
---


# Message Queue

消息队列。
以异步方式在线程之间传输小数据项。
基于ring buffer实现。

```c
void k_msgq_init(struct k_msgq *msgq, char *buffer, size_t msg_size,uint32_t max_msgs);
// 和上面的区别是在函数内部动态分配了buffer内存(在堆上)
__syscall int k_msgq_alloc_init(struct k_msgq *msgq, size_t msg_size,
				uint32_t max_msgs);
// 如果调用的init函数是k_msgq_alloc_init，可以用cleanup函数释放掉buffer内存。
int k_msgq_cleanup(struct k_msgq *msgq);

__syscall int k_msgq_put(struct k_msgq *msgq, const void *data, k_timeout_t timeout);
__syscall int k_msgq_get(struct k_msgq *msgq, void *data, k_timeout_t timeout);
__syscall int k_msgq_peek(struct k_msgq *msgq, void *data);
```

## Msgq init

```c
struct data_item_type {
    uint32_t field1;
    uint32_t field2;
    uint32_t field3;
};

char my_msgq_buffer[10 * sizeof(struct data_item_type)];
struct k_msgq my_msgq;

k_msgq_init(&my_msgq, my_msgq_buffer, sizeof(struct data_item_type), 10);
```

## Writing to msgq

往消息队列中放数据，如果msgq满了无法放入，可以调用k_msgq_purge把msgq现存的所有的消息都丢弃。

```c
void producer_thread(void)
{
    struct data_item_type data;

    while (1) {
        /* create data item to send (e.g. measurement, timestamp, ...) */
        data = ...

        /* send data to consumers */
        while (k_msgq_put(&my_msgq, &data, K_NO_WAIT) != 0) {
            /* message queue is full: purge old data & try again */
            k_msgq_purge(&my_msgq);
        }

        /* data item was successfully added to message queue */
    }
}
```

## Reading from msgq

取出消息队列一条数据。

```c
void consumer_thread(void)
{
    struct data_item_type data;

    while (1) {
        /* get a data item */
        k_msgq_get(&my_msgq, &data, K_FOREVER);

        /* process data item */
        ...
    }
}
```

## Peaking into a msgq

查看消息队列头上的第一条数据。

```c
void consumer_thread(void)
{
    struct data_item_type data;

    while (1) {
        /* read a data item by peeking into the queue */
        k_msgq_peek(&my_msgq, &data);

        /* process data item */
        ...
    }
}
```

## 源码实现

```c
struct k_msgq {
	/** Message queue wait queue */
	_wait_q_t wait_q;
	/** Lock */
	struct k_spinlock lock;
	/** Message size */
	size_t msg_size;
	/** Maximal number of messages */
	uint32_t max_msgs;
	/** Start of message buffer */
	char *buffer_start;
	/** End of message buffer */
	char *buffer_end;
	/** Read pointer */
	char *read_ptr;
	/** Write pointer */
	char *write_ptr;
	/** Number of used messages */
	uint32_t used_msgs;

	Z_DECL_POLL_EVENT

	/** Message queue */
	uint8_t flags;
};
```

Init 函数主要是在初始化`k_msgq`结构体。

```c
void k_msgq_init(struct k_msgq *msgq, char *buffer, size_t msg_size,
		 uint32_t max_msgs)
{
	msgq->msg_size = msg_size; // 一条message的size
	msgq->max_msgs = max_msgs; // msgq最多放多少条msg
	msgq->buffer_start = buffer; // buffer起始地址
	msgq->buffer_end = buffer + (max_msgs * msg_size); // buffer存放最后一条msg的地址
	msgq->read_ptr = buffer; // 读指针
	msgq->write_ptr = buffer; // 写指针
	msgq->used_msgs = 0; // 表示msgq中存放了多少条msg
	msgq->flags = 0; // 如果buffer内存是在堆上动态分配的话，flag=K_MSGQ_FLAG_ALLOC,目前只有这一个flag
	z_waitq_init(&msgq->wait_q); // wait_q就是双向链表，用来保存挂起的线程
	msgq->lock = (struct k_spinlock) {};
#ifdef CONFIG_POLL
	sys_dlist_init(&msgq->poll_events);
#endif	/* CONFIG_POLL */
}
```

```c
int z_impl_k_msgq_put(struct k_msgq *msgq, const void *data, k_timeout_t timeout)
{
	// assert为真才能继续执行，这个条件表示在中断中，只有K_NO_WAIT可以，其他阻塞等待的timeout都会失败。
	__ASSERT(!arch_is_in_isr() || K_TIMEOUT_EQ(timeout, K_NO_WAIT), "");

	struct k_thread *pending_thread;
	k_spinlock_key_t key;
	int result;

	key = k_spin_lock(&msgq->lock);

	if (msgq->used_msgs < msgq->max_msgs) {
		/* message queue isn't full */
		pending_thread = z_unpend_first_thread(&msgq->wait_q);
		// 如果有线程在等待msg的input（这边对应get函数msgq没有数据可读的情况，倍加入waitq），直接把要写入的data拷贝到该pending thread的swap_data中，然后调度该线程。
		if (pending_thread != NULL) {
			/* give message to waiting thread */
			(void)memcpy(pending_thread->base.swap_data, data,
			       msgq->msg_size);
			/* wake up waiting thread */
			arch_thread_return_value_set(pending_thread, 0);
			z_ready_thread(pending_thread);
			z_reschedule(&msgq->lock, key);
			return 0;
		} else { // 没有线程在等待msg的input，就先放进msgq。
			/* put message in queue */
			__ASSERT_NO_MSG(msgq->write_ptr >= msgq->buffer_start &&
					msgq->write_ptr < msgq->buffer_end);
			// 拷贝一条msg到写指针
			(void)memcpy(msgq->write_ptr, data, msgq->msg_size);
			// 写指针后移一条msg大小
			msgq->write_ptr += msgq->msg_size;
			// 如果写指针达到buffer尾部，跳转到buffer开头
			if (msgq->write_ptr == msgq->buffer_end) {
				msgq->write_ptr = msgq->buffer_start;
			}
			// 记录msgq中放入的msg数量
			msgq->used_msgs++;
#ifdef CONFIG_POLL
			handle_poll_events(msgq, K_POLL_STATE_MSGQ_DATA_AVAILABLE);
#endif /* CONFIG_POLL */
		}
		result = 0;
	} else if (K_TIMEOUT_EQ(timeout, K_NO_WAIT)) {
		/* don't wait for message space to become available */
		// msgq满了，并且设置了NO_WAIT,就直接返回错误
		result = -ENOMSG;
	} else {
		/* wait for put message success, failure, or timeout */

		_current->base.swap_data = (void *) data;
		// msgq满了，把当前线程加入wait_q，进入睡眠，等待timeout时间后唤醒线程，判断是否其他线程从waitq中唤醒过该线程，如果唤醒过会把z_pend_curr的返回值设置为0，否则返回错误。
		result = z_pend_curr(&msgq->lock, key, &msgq->wait_q, timeout);
		return result;
	}

	k_spin_unlock(&msgq->lock, key);

	return result;
}
```

```c
int z_impl_k_msgq_get(struct k_msgq *msgq, void *data, k_timeout_t timeout)
{
	__ASSERT(!arch_is_in_isr() || K_TIMEOUT_EQ(timeout, K_NO_WAIT), "");

	k_spinlock_key_t key;
	struct k_thread *pending_thread;
	int result;

	key = k_spin_lock(&msgq->lock);

	if (msgq->used_msgs > 0U) {
		/* take first available message from queue */
		// msgq中有数据，直接取走。
		(void)memcpy(data, msgq->read_ptr, msgq->msg_size);
		msgq->read_ptr += msgq->msg_size;
		if (msgq->read_ptr == msgq->buffer_end) {
			msgq->read_ptr = msgq->buffer_start;
		}
		msgq->used_msgs--;

		/* handle first thread waiting to write (if any) */
		// 如果msgq数据满了，那么会有在waitq中挂起的线程，把该线程要写入的data放进msgq中，这边对应上面put函数msgq满的情况一起看。
		pending_thread = z_unpend_first_thread(&msgq->wait_q);
		if (pending_thread != NULL) {
			/* add thread's message to queue */
			__ASSERT_NO_MSG(msgq->write_ptr >= msgq->buffer_start &&
					msgq->write_ptr < msgq->buffer_end);
			(void)memcpy(msgq->write_ptr, pending_thread->base.swap_data,
			       msgq->msg_size);
			msgq->write_ptr += msgq->msg_size;
			if (msgq->write_ptr == msgq->buffer_end) {
				msgq->write_ptr = msgq->buffer_start;
			}
			msgq->used_msgs++;

			/* wake up waiting thread */
			// 因为取出的waitq中的线程可能还在阻塞，这里调度该线程。
			arch_thread_return_value_set(pending_thread, 0);
			z_ready_thread(pending_thread);
			z_reschedule(&msgq->lock, key);

			return 0;
		}
		result = 0;
	} else if (K_TIMEOUT_EQ(timeout, K_NO_WAIT)) {
		/* don't wait for a message to become available */
		result = -ENOMSG;
	} else {
		/* wait for get message success or timeout */
		_current->base.swap_data = data;
		// msgq中没有数据可读，把当前线程加入wait_q中，等待put函数处理。
		result = z_pend_curr(&msgq->lock, key, &msgq->wait_q, timeout);
		return result;
	}

	k_spin_unlock(&msgq->lock, key);

	return result;
}
```

# Pipe
