---
date: 2024-05-06T09:38:31+08:00
title: 'Zephyr -- Polling and Events'
tags:
- Zephyr
categories:
- Zephyr OS
---

# Poll

轮询 API 允许单个线程同时等待一个或多个条件得到满足，而无需主动单独查看每个条件。

比如可以同时等待如下条件被满足：

- a semaphore becomes available
- a kernel FIFO contains data ready to be retrieved
- a kernel message queue contains data ready to be retrieved
- a kernel pipe contains data ready to be retrieved
- a poll signal is raised

使用poll api前需要将poll events都放入一个数组。

## Implementation

### k_poll()

静态初始化：

```c
struct k_poll_event events[4] = {
    K_POLL_EVENT_STATIC_INITIALIZER(K_POLL_TYPE_SEM_AVAILABLE,
                                    K_POLL_MODE_NOTIFY_ONLY,
                                    &my_sem, 0),
    K_POLL_EVENT_STATIC_INITIALIZER(K_POLL_TYPE_FIFO_DATA_AVAILABLE,
                                    K_POLL_MODE_NOTIFY_ONLY,
                                    &my_fifo, 0),
    K_POLL_EVENT_STATIC_INITIALIZER(K_POLL_TYPE_MSGQ_DATA_AVAILABLE,
                                    K_POLL_MODE_NOTIFY_ONLY,
                                    &my_msgq, 0),
    K_POLL_EVENT_STATIC_INITIALIZER(K_POLL_TYPE_PIPE_DATA_AVAILABLE,
                                    K_POLL_MODE_NOTIFY_ONLY,
                                    &my_pipe, 0),
};
```

runtime初始化：

```c
void k_poll_event_init(struct k_poll_event *event, uint32_t type,
		       int mode, void *obj)

struct k_poll_event events[4];
void some_init(void)
{
    k_poll_event_init(&events[0],
                      K_POLL_TYPE_SEM_AVAILABLE,
                      K_POLL_MODE_NOTIFY_ONLY,
                      &my_sem);

    k_poll_event_init(&events[1],
                      K_POLL_TYPE_FIFO_DATA_AVAILABLE,
                      K_POLL_MODE_NOTIFY_ONLY,
                      &my_fifo);

    k_poll_event_init(&events[2],
                      K_POLL_TYPE_MSGQ_DATA_AVAILABLE,
                      K_POLL_MODE_NOTIFY_ONLY,
                      &my_msgq);

    k_poll_event_init(&events[3],
                      K_POLL_TYPE_PIPE_DATA_AVAILABLE,
                      K_POLL_MODE_NOTIFY_ONLY,
                      &my_pipe);

    // tags are left uninitialized if unused
}
```

**type**必须和object类型匹配。比如`K_POLL_TYPE_SEM_AVAILABLE`和`my_sem`。在`kernel.h`中共找到如下支持的type:

- K_POLL_TYPE_SIGNAL
- K_POLL_TYPE_SEM_AVAILABLE
- K_POLL_TYPE_DATA_AVAILABLE
- K_POLL_TYPE_FIFO_DATA_AVAILABLE
- K_POLL_TYPE_MSGQ_DATA_AVAILABLE
- K_POLL_TYPE_PIPE_DATA_AVAILABLE

**mode**必须传入`K_POLL_MODE_NOTIFY_ONLY`。
**state**必须是`K_POLL_STATE_NOT_READY`，state初始化会在初始化函数中自动完成。

```c
__syscall int k_poll(struct k_poll_event *events, int num_events,
		     k_timeout_t timeout);

void do_stuff(void)
{
    for(;;) {

        rc = k_poll(events, 2, K_FOREVER);

        if (events[0].state == K_POLL_STATE_SEM_AVAILABLE) {
            k_sem_take(events[0].sem, 0);
        } else if (events[1].state == K_POLL_STATE_FIFO_DATA_AVAILABLE) {
            data = k_fifo_get(events[1].fifo, 0);
            // handle data
        } else if (events[2].state == K_POLL_STATE_MSGQ_DATA_AVAILABLE) {
            ret = k_msgq_get(events[2].msgq, buf, K_NO_WAIT);
            // handle data
        } else if (events[3].state == K_POLL_STATE_PIPE_DATA_AVAILABLE) {
            ret = k_pipe_get(events[3].pipe, buf, bytes_to_read, &bytes_read, min_xfer, K_NO_WAIT);
            // handle data
	}
        events[0].state = K_POLL_STATE_NOT_READY;
        events[1].state = K_POLL_STATE_NOT_READY;
        events[2].state = K_POLL_STATE_NOT_READY;
        events[3].state = K_POLL_STATE_NOT_READY;
    }
}
```

初始化完后，就可以调用`k_poll()`阻塞等待，在经过timeout时间后，如果返回值为0说明等到了某个事件发生，再根据event成员的state变量是否为available来判断各自的事件是否发生。
如果在loop中调用`k_poll()`，需要在下次循环开始前把state再次设置为`NOT_READY`。

### k_poll_signal_raise()

poll_signal可以看作是轻量级的二值信号量。

使用前需要调用`K_POLL_SIGNAL_INITIALIZER()` or `k_poll_signal_init()`单独初始化。

```c
struct k_poll_signal signal;
void do_stuff(void)
{
    k_poll_signal_init(&signal);
}
```

e.g.

```c
struct k_poll_signal signal;

// thread A
void do_stuff(void)
{
    k_poll_signal_init(&signal);

    struct k_poll_event events[1] = {
        K_POLL_EVENT_INITIALIZER(K_POLL_TYPE_SIGNAL,
                                 K_POLL_MODE_NOTIFY_ONLY,
                                 &signal),
    };

    k_poll(events, 1, K_FOREVER);

    if (events.signal->result == 0x1337) {
        // A-OK!
    } else {
        // weird error
    }
}

// thread B
void signal_do_stuff(void)
{
    k_poll_signal_raise(&signal, 0x1337);
}
```

使用 k_poll() 来合并多个线程，每个线程将在一个对象上挂起，从而节省可能大量的堆栈空间。

## Poll实现源码分析

// Todo:

# Events

用一个32bits的值，每个bit记录不同的events。

## Event Implementation

初始化

```c
struct k_event my_event;

k_event_init(&my_event); // 或
K_EVENT_DEFINE(my_event);
```

### Setting Events

Setting api是将传入的第二个参数直接设置为所有的events。

```c
void input_available_interrupt_handler(void *arg)
{
    k_event_set(&my_event, 0x001);
}
```

### Posting Evnets

Posting api传入的第二个参数是增加的events。

```c
void input_available_interrupt_handler(void *arg)
{
    k_event_post(&my_event, 0x120);
}
```

### Waiting for events

`k_event_wait`如果有一个监控的事件发生就返回。
`k_event_wait_all`等所有监控的事件发生才返回。

```c
void consumer_thread(void)
{
    uint32_t  events;

    events = k_event_wait(&my_event, 0xFFF, false, K_MSEC(50));
    if (events == 0) {
        printk("No input devices are available!");
    } else {
        /* Access the desired input device(s) */
        ...
    }
    ...
}
```

## Event实现源码分析

// Todo:
