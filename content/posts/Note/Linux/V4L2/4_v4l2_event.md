## 2.14 V4L2 events

V4L2 events 提供了一种方法给 userspace 传递 events.

```c++
struct v4l2_subscribed_event {
	struct list_head	list;
	u32			type;
	u32			id;
	u32			flags;
	struct v4l2_fh		*fh;
	struct list_head	node;
	const struct v4l2_subscribed_event_ops *ops;
	unsigned int		elems;
	unsigned int		first;
	unsigned int		in_use;
	struct v4l2_kevent	events[] __counted_by(elems);
};
```

list: 加入到 v4l2_fh->subscribed list 的链表节点。  
type: event type, 在 videodev2.h 中定义。  
id: event 的 control id, 根据 event 的 (type, id) 二元组就能找到对应的 event.  
flags: 从 userspace 传入的 v4l2_event_subscription->flags 中拷贝过来。  
fh: 订阅该 event 的 file handler(v4l2_fh).  
node: 加入到 v4l2_ctrl->ev_subs 的链表节点。  
ops: event 的回调函数。  
elems: event arrays 中的 event 数量。  
first: 最早的 event index.  
in_use: queued events 的数量。  
events: 相同 type 的 event 数组。

```c++
struct v4l2_subscribed_event_ops {
	int  (*add)(struct v4l2_subscribed_event *sev, unsigned int elems);
	void (*del)(struct v4l2_subscribed_event *sev);
	void (*replace)(struct v4l2_event *old, const struct v4l2_event *new);
	void (*merge)(const struct v4l2_event *old, struct v4l2_event *new);
};
```

add: 在 event 被添加到 subscribed list 时调用。
del: 在 event 被从 subscribed list 中删除时调用。
replace: 将旧的 event 替换为新的 event, 只有在 elems=1, 即 events[]数组长度为 1 时才可以使用。
merge: 将最旧的 event 替换为第二旧的 event, 在 elems>1 时使用。

### 2.14.1 Event subscription

通过 `v4l2_event_subscribe(fh, sub, elems, ops)` 订阅 events.

<div align="center">
<img src="https://xyc-1316422823.cos.ap-shanghai.myqcloud.com/20250211143545.png" width="30%">
</div>

### 2.14.2 Unsubscribing events

通过 `v4l2_event_unsubscribe(fh, sub)` 取消订阅 events.

### 2.14.3 Check if there's a pending event

`v4l2_event_pending(fh)`

### 2.14.4 How events work
