---
date: 2024-05-06T17:13:10+08:00
title: 'Zephyr -- WorkQueue'
draft: true
tags:
- Zephyr
categories:
- Zephyr OS
---


# Linked List

## Single-linked List

Zephyr的单向链表是带头尾节点的链表。

```c
struct _snode {
	struct _snode *next;
};

/** Single-linked list node structure. */
typedef struct _snode sys_snode_t; // 代表一个链表节点

struct _slist {
	sys_snode_t *head; // 头节点
	sys_snode_t *tail; // 尾节点
};

/** Single-linked list structure. */
typedef struct _slist sys_slist_t;
```

![](https://xyc-1316422823.cos.ap-shanghai.myqcloud.com/20240506140439.png)

```c
// 初始化链表
static inline void sys_slist_init(sys_slist_t *list);
// 返回链表头节点
static inline sys_snode_t *sys_slist_peek_head(sys_slist_t *list);
// 返回链表尾节点
static inline sys_snode_t *sys_slist_peek_tail(sys_slist_t *list);
// 返回当前node的下一个next节点
static inline sys_snode_t *sys_slist_peek_next(sys_snode_t *node);
// 把node插入到list的头节点位置
static inline void sys_slist_prepend(sys_slist_t *list,
				     sys_snode_t *node);
// 把node插入到list的尾节点位置
static inline void sys_slist_append(sys_slist_t *list,
				    sys_snode_t *node);
// 链表插入
// 把node插入到prev节点后面，从prev->next到prev->node->next
static inline void sys_slist_insert(sys_slist_t *list,
				    sys_snode_t *prev,
				    sys_snode_t *node);
// 链表删除
// 把prev后面的节点删除，
static inline void sys_slist_remove(sys_slist_t *list,
				    sys_snode_t *prev_node,
				    sys_snode_t *node);
```

## Flagged List

```c
typedef uint32_t unative_t;

struct _sfnode {
	unative_t next_and_flags; // 这边保存的是next指针地址+flags
};

/** Flagged single-linked list node structure. */
typedef struct _sfnode sys_sfnode_t;

/** @cond INTERNAL_HIDDEN */
struct _sflist {
	sys_sfnode_t *head;
	sys_sfnode_t *tail;
};

/** Flagged single-linked list structure. */
typedef struct _sflist sys_sflist_t;
```

实现基本和单向链表类似，提供相同功能的API接口。

不同点是node指针的低两位可以保存flags，通过`sys_sfnode_flags_get()` and `sys_sfnode_flags_set()`获取和设置flags。

## Double-linked List

双向循环链表。

![](https://xyc-1316422823.cos.ap-shanghai.myqcloud.com/20240506153341.png)

# Ring buffers

// Todo:

# Queue

```c
struct k_queue {
	sys_sflist_t data_q;
	struct k_spinlock lock;
	_wait_q_t wait_q;

	Z_DECL_POLL_EVENT
};
```

```c
__syscall void k_queue_init(struct k_queue *queue);

void z_impl_k_queue_init(struct k_queue *queue)
{
	sys_sflist_init(&queue->data_q);
	queue->lock = (struct k_spinlock) {};
	z_waitq_init(&queue->wait_q);

#if defined(CONFIG_POLL)
	sys_dlist_init(&queue->poll_events);
#endif
}
```

```c
void k_queue_append(struct k_queue *queue, void *data);

void k_queue_append(struct k_queue *queue, void *data)
{
	(void)queue_insert(queue, NULL, data, false, true);
}

static int32_t queue_insert(struct k_queue *queue, void *prev, void *data,
			    bool alloc, bool is_append)
{
	struct k_thread *first_pending_thread;
	k_spinlock_key_t key = k_spin_lock(&queue->lock);

	if (is_append) {
		prev = sys_sflist_peek_tail(&queue->data_q);
	}
	first_pending_thread = z_unpend_first_thread(&queue->wait_q);

	if (first_pending_thread != NULL) {
		prepare_thread_to_run(first_pending_thread, data);
		z_reschedule(&queue->lock, key);

		return 0;
	}

	/* Only need to actually allocate if no threads are pending */
	if (alloc) {
		struct alloc_node *anode;

		anode = z_thread_malloc(sizeof(*anode));
		if (anode == NULL) {
			k_spin_unlock(&queue->lock, key);

			SYS_PORT_TRACING_OBJ_FUNC_EXIT(k_queue, queue_insert, queue, alloc,
				-ENOMEM);

			return -ENOMEM;
		}
		anode->data = data;
		sys_sfnode_init(&anode->node, 0x1);
		data = anode;
	} else {
		sys_sfnode_init(data, 0x0);
	}

	sys_sflist_insert(&queue->data_q, prev, data);
	handle_poll_events(queue, K_POLL_STATE_DATA_AVAILABLE);
	z_reschedule(&queue->lock, key);

	return 0;
}
```
