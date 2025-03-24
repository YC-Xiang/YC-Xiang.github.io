## 2.2 Structure of a V4L driver

```txt
device instances
  |
  +-sub-device instances
  |
  \-V4L2 device nodes
      |
      \-filehandle instances
```

## 2.4 Video device

video device 用于抽象系统注册的 v4l2 /dev 设备节点，以便用户空间可以进行交互.

`v4l2-dev.h`:

```c
struct video_device {
#if defined(CONFIG_MEDIA_CONTROLLER)
	struct media_entity entity;
	struct media_intf_devnode *intf_devnode;
	struct media_pipeline pipe;
#endif
	const struct v4l2_file_operations *fops;

	u32 device_caps;

	/* sysfs */
	struct device dev;
	struct cdev *cdev;

	struct v4l2_device *v4l2_dev;
	struct device *dev_parent;

	struct v4l2_ctrl_handler *ctrl_handler;

	struct vb2_queue *queue;

	struct v4l2_prio_state *prio;

	/* device info */
	char name[64];
	enum vfl_devnode_type vfl_type;
	enum vfl_devnode_direction vfl_dir;
	int minor;
	u16 num;
	unsigned long flags;
	int index;

	/* V4L2 file handles */
	spinlock_t		fh_lock;
	struct list_head	fh_list;

	int dev_debug;

	v4l2_std_id tvnorms;

	/* callbacks */
	void (*release)(struct video_device *vdev);
	const struct v4l2_ioctl_ops *ioctl_ops;
	DECLARE_BITMAP(valid_ioctls, BASE_VIDIOC_PRIVATE);

	struct mutex *lock;
};
```

entity:

fops: v4l2 file operations.  
device_caps: 设备支持的能力, 在 videodev2.h 中定义.  
dev:
cdev:
v4l2_dev: 上层的 v4l2_device 设备.  
dev_parent: 上层的 device 设备, 如果为 NULL 在 video_register_device 中会指向 v4l2_dev->dev  
ctrl_handler: 如果为 NULL, 在 video_register_device 中会指向 v4l2_dev->ctrl_handler.  
queue: 和该 device node 相关的 vb2_queue 结构体.  
prio: 如果为 NULL, 在 video_register_device 中会指向 v4l2_dev->prio .  
name: video device name.  
vfl_type: enum vfl_devnode_type, v4l2 设备的类型, 在 video_register_device 中传入.  
vfl_dir: enum vfl_devnode_direction, 表示 v4l2 设备的方向, RX(capture device)/TX(output device)/M2M(mem2mem, codec).  
minor: device node 次设备号.  
num: video device node 的编号, /dev/videoX 的 X, 在 video_register_device 最后一个参数传入.  
flags: enum v4l2_video_device_flags, 一些辅助 flags.  
index: 一个物理设备对应多个 v4l2 设备节点, 分别的 index. 每次调用 video_register_device()都会增加 1.  
fh_lock: 用来 lock v4l2_fhs.  
fh_list: v4l2_fh 链表.  
dev_debug: userspace 通过 sysfs 设置的 debug level. /sys/class/video4linux/video0/dev_debug  
tvnorms: 支持的电视标准 PAL/NTSC/SECAM.  
release: 释放函数. 在 video_device 没有 subclass 的情况下可以使用 video_device_release().  
如果 video_device subclass 或者是 static 分配的, 不需要释放内存使用 video_device_release_empty().  
ioctl_ops: v4l2 file ioctl operations.  
valid_ioctls: 支持的 ioctrl bitmap.  
lock: 用来串行化 v4l2 device 的 unlock_ioctl.

其中一些 fields 需要我们手动去初始化, 包括 fops, device_caps, v4l2_dev, queue, name, vfl_dir, ioctl_ops, lock.

如果需要和 media framework 联合使用, 需要初始化 entity 成员, 并调用 media_entity_pads_init().

如果 v4l2 driver 使用 v4l2_ioctl_ops, 则 v4l2_file_operations 中的.unlock_ioctl 回调需要使用 video_ioctl2.

### 2.4.2 Video device registration

```c
err = video_register_device(vdev, VFL_TYPE_VIDEO, -1);
if (err) {
	video_device_release(vdev); /* or kfree(my_vdev); */
	return err;
}
```

不同的设备类型会生成不同的设备节点:

![](https://xyc-1316422823.cos.ap-shanghai.myqcloud.com/20250210164308.png)

### 2.4.3 Video device debugging

往`/sys/class/video4linux/<devX>/dev_debug`写入数值:

![](https://xyc-1316422823.cos.ap-shanghai.myqcloud.com/20250210164947.png)

### 2.4.4 Video device cleanup

`video_unregister_device()`

别忘记 clean up media_entity:

`media_entity_cleanup (&vdev->entity)`

### 2.4.5 helper functions

```c
static inline void *video_get_drvdata(struct video_device *vdev)
static inline void video_set_drvdata(struct video_device *vdev, void *data)
struct video_device *video_devdata(struct file *file)
```

### 2.4.6 functions and data structures

一些重要的 APIs:

```c
int video_register_device(struct video_device *vdev,
				enum vfl_devnode_type type, int nr);
void video_unregister_device(struct video_device *vdev);
struct video_device *video_device_alloc(void);
```

## 2.5 V4L2 device

v4l2 用来抽象最顶层的 v4l2 设备, 包含一系列子设备.

通过`v4l2_device_register (dev, v4l2_dev)`注册, 在注册前需要调用 dev_set_drvdata()
设置 dev->drier_data 为包含 v4l2_device 的 driver-specific 结构体.

通过`v4l2_device_unregister()`注销, 如果是 hotpluggable 设备, 在此之前还需要调用`v4l2_device_disconnect()`

`v4l2-device.h`:

```c
struct v4l2_device {
	struct device *dev;
	struct media_device *mdev;
	struct list_head subdevs;
	spinlock_t lock;
	char name[36];
	void (*notify)(struct v4l2_subdev *sd,
			unsigned int notification, void *arg);
	struct v4l2_ctrl_handler *ctrl_handler;
	struct v4l2_prio_state prio;
	struct kref ref;
	void (*release)(struct v4l2_device *v4l2_dev);
};
```

dev: 底层 device 设备.  
mdev: 指向 media device.  
subdevs: 子设备列表.  
name: v4l2 设备名称.  
notify: 子设备调用的通知回调函数.  
ctrl_handler: v4l2 控制句柄.  
prio: 设备的优先级状态.  
ref: 引用计数.  
release: 释放回调函数.

</br>

APIs:

```c
void v4l2_device_get(struct v4l2_device *v4l2_dev);
int v4l2_device_put(struct v4l2_device *v4l2_dev);
int v4l2_device_register(struct device *dev, struct v4l2_device *v4l2_dev);
int v4l2_device_set_name(struct v4l2_device *v4l2_dev, const char *basename,
			 atomic_t *instance);
void v4l2_device_disconnect(struct v4l2_device *v4l2_dev);
void v4l2_device_unregister(struct v4l2_device *v4l2_dev);

#define v4l2_device_register_subdev(v4l2_dev, sd) \
	__v4l2_device_register_subdev(v4l2_dev, sd, THIS_MODULE)
int __v4l2_device_register_subdev(struct v4l2_device *v4l2_dev,
				struct v4l2_subdev *sd,
				struct module *module);
void v4l2_device_unregister_subdev(struct v4l2_subdev *sd);
int v4l2_device_register_subdev_nodes(struct v4l2_device *v4l2_dev);
int v4l2_device_register_ro_subdev_nodes(struct v4l2_device *v4l2_dev);
void v4l2_subdev_notify(struct v4l2_subdev *sd, unsigned int notification, void *arg);
bool v4l2_device_supports_requests(struct v4l2_device *v4l2_dev);
```

## 2.6 V4L2 File handlers

struct v4l2_fh 提供了一种方式, 使得 file 方便地处理 V4L2 中的一些 specific data.

初始化: `v4l2_fh_init()`, 必须在 driver 的 v4l2_file_operations->open()回调中调用,
会置起 video_device 的 V4L2_FL_USES_V4L2_FH flag.

在 userspace open device node 后, 调用到 v4l2_fh_open, 会把 file->private_data 设置为 fh.

许多情况下 v4l2_fh 都会嵌入在更大的结构体中, 这时需要调用

v4l2_fh_init()和 v4l2_fh_add 在.open()回调  
v4l2_fh_del()和 v4l2_fh_exit()在.release()回调

```c
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

```c
void v4l2_fh_init(struct v4l2_fh *fh, struct video_device *vdev);
void v4l2_fh_add(struct v4l2_fh *fh);
int v4l2_fh_open(struct file *filp);
void v4l2_fh_del(struct v4l2_fh *fh);
void v4l2_fh_exit(struct v4l2_fh *fh);
int v4l2_fh_release(struct file *filp);
int v4l2_fh_is_singular(struct v4l2_fh *fh);
int v4l2_fh_is_singular_file(struct file *filp);
```
