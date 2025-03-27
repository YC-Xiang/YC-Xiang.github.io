## 2.6 V4L2 File handlers

struct v4l2_fh 提供了一种方式, 使得 file 方便地处理 V4L2 中的一些 specific data.

初始化: `v4l2_fh_init()`, 必须在 driver 的 v4l2_file_operations->open()回调中调用,
会置起 video_device 的 V4L2_FL_USES_V4L2_FH flag.

在 userspace open device node 后, 调用到 v4l2_fh_open, 会把 file->private_data 设置为 fh.

许多情况下 v4l2_fh 都会嵌入在更大的结构体中, 这时需要调用

v4l2_fh_init()和 v4l2_fh_add 在.open()回调  
v4l2_fh_del()和 v4l2_fh_exit()在.release()回调

```c++
struct v4l2_fh {
	struct list_head	list;
	struct video_device	*vdev;
	struct v4l2_ctrl_handler *ctrl_handler;
	enum v4l2_priority	prio;

	wait_queue_head_t	wait;
	struct mutex		subscribe_lock;
	struct list_head	subscribed;
	struct list_head	available;
	unsigned int		navailable;
	u32			sequence;

	struct v4l2_m2m_ctx	*m2m_ctx;
};
```

list: 链接到 video_device 的 fh_list.  
vdev: 指向 video_device.  
ctrl_handler: 指向 video_device 的 ctrl_handler.  
prio: file handler 的优先级.  
wait: 进程等待队列.  
subscribe_lock: 串行化 subscribe list.  
subscribed: subscribe list, 保存该文件订阅的 event 类型, 挂载 struct v4l2_subscribed_event->list.  
available: 存储待处理的 event list, 挂载 struct v4l2_kevent->list.  
navailable: 待处理事件的数量.  
sequence: 给每个 event 分配的序列号.
m2m_ctx:

流程:

应用程序通过 VIDIOC_SUBSCRIBE_EVENT ioctl 订阅感兴趣的 events.
驱动程序将事件类型添加到 subscribed list.
当事件发生时，驱动将事件添加到 available list.
应用程序通过 VIDIOC_DQEVENT ioctl 获取事件.
如果没有可用事件，进程会在 wait 队列上等待.

APIs:

```c++
void v4l2_fh_init(struct v4l2_fh *fh, struct video_device *vdev);
void v4l2_fh_add(struct v4l2_fh *fh);
int v4l2_fh_open(struct file *filp);
void v4l2_fh_del(struct v4l2_fh *fh);
void v4l2_fh_exit(struct v4l2_fh *fh);
int v4l2_fh_release(struct file *filp);
int v4l2_fh_is_singular(struct v4l2_fh *fh);
int v4l2_fh_is_singular_file(struct file *filp);
```
