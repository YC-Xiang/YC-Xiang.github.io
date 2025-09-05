---
date: 2024-08-28T13:40:27+08:00
title: "DRM(10) -- VBlank"
tags:
  - DRM
categories:
  - DRM
---

# Overview

一篇关于垂直同步 V-Sync 解释的文章：https://daily.elepover.com/2021/03/27/vsync/index.html

当 GPU 渲染的速度 > 显示器刷新的速度时，GPU 在显示器还来不及完成渲染完一帧时，就切换了 framebuffer，会导致出现撕裂现象。

帧率大于显示器刷新率时，启用垂直同步。  
帧率小于显示器刷新率时，禁用垂直同步。

</br>

为了支持 vblank，底层 drvier 需要调用 drm_vblank_init() 初始化，另外需要实现 drm_crtc_funcs.enable_vblank 和 drm_crtc_funcs.disable_vblank 两个回调函数，并且在 vblank 中断中调用 drm_crtc_handle_vblank()。

# 数据结构

每个 crtc 对应一个 struct drm_vblank_crtc 结构体。

```c++
struct drm_vblank_crtc {
	struct drm_device *dev;
	wait_queue_head_t queue;
	struct timer_list disable_timer;
	seqlock_t seqlock;
	atomic64_t count;
	ktime_t time;
	atomic_t refcount;
	u32 last;
	u32 max_vblank_count;
	unsigned int inmodeset;
	unsigned int pipe;
	int framedur_ns;
	int linedur_ns;
	struct drm_display_mode hwmode;
	bool enabled;
	struct kthread_worker *worker;
	struct list_head pending_work;
	wait_queue_head_t work_wait_queue;
};
```

`refcount`: vblank interrupt user/waiter 数量。

`max_vblank_count`: vblank register 最大的范围，如果不为 0 表示支持硬件 vblank count 计数，底层 driver 初始化该数值，并且 drm_crtc_funcs.get_vblank_counter 必须提供。

`inmodeset`: 表示是否在 modeset 过程中，1 vblank is disabled, 0 vblank is enabled.

`pipe`: 表示第几个 crtc，在 drm_vblank_init 中初始化。

`framedur_ns`: 一帧持续的纳秒。

`linedur_ns`: 一行持续的纳秒。

</br>

struct drm_pending_vblank_event 保存在 crtc_state->event 中

```c++
struct drm_pending_vblank_event {
	struct drm_pending_event base;
	unsigned int pipe;
	u64 sequence;
	union {
		struct drm_event base;
		struct drm_event_vblank vbl;
		struct drm_event_crtc_sequence seq;
	} event;
};
```

pipe: 属于哪一个 crtc id.  
sequence: 硬件 vblank 应该在该数量 trigger.  

```c++
struct drm_pending_event {
	struct completion *completion;
	void (*completion_release)(struct completion *completion);
	struct drm_event *event;
	struct dma_fence *fence;
	struct drm_file *file_priv;
	struct list_head link;
	struct list_head pending_link;
};
```

# function flow

init 函数中首先调用 drm_vblank_init():

```c++
int drm_vblank_init(struct drm_device *dev, unsigned int num_crtcs)
{
	int ret;
	unsigned int i;

	dev->vblank = drmm_kcalloc(dev, num_crtcs, sizeof(*dev->vblank), GFP_KERNEL);
	dev->num_crtcs = num_crtcs;

	for (i = 0; i < num_crtcs; i++) {
		struct drm_vblank_crtc *vblank = &dev->vblank[i];

		vblank->dev = dev;
		vblank->pipe = i;
		init_waitqueue_head(&vblank->queue);
		timer_setup(&vblank->disable_timer, vblank_disable_fn, 0);

		ret = drmm_add_action_or_reset(dev, drm_vblank_init_release,
					       vblank);
		ret = drm_vblank_worker_init(vblank);
	}

	return 0;
}
```

接着做 reset 的动作：

```c++
drm_mode_config_reset();
	crtc->funcs->reset(); // drm_atomic_helper_crtc_reset()
		drm_crtc_vblank_reset();

void drm_crtc_vblank_reset(struct drm_crtc *crtc)
{
	struct drm_device *dev = crtc->dev;
	unsigned int pipe = drm_crtc_index(crtc);
	struct drm_vblank_crtc *vblank = &dev->vblank[pipe];

	spin_lock_irq(&dev->vbl_lock);
	if (!vblank->inmodeset) {
		atomic_inc(&vblank->refcount);
		vblank->inmodeset = 1;
	}
	spin_unlock_irq(&dev->vbl_lock);
}
```

在 userspace 进行 atomic commit 之后，会先创建 vblank event.

struct drm_pending_vblank_event -> struct drm_pending_event -> struct drm_event

```c++
prepare_signaling();
	create_vblank_event();
		struct drm_pending_vblank_event *e = NULL;
		e = kzalloc(sizeof *e, GFP_KERNEL);
		e->event.base.type = DRM_EVENT_FLIP_COMPLETE;
		e->event.base.length = sizeof(e->event);
		e->event.vbl.crtc_id = crtc->base.id;
		e->event.vbl.user_data = user_data;
	crtc_state->event = e;
	drm_event_reserve_init(dev, file_priv, &e->base, &e->event.base);
		p->event = e; // drm_pending_event->event = e
		list_add(&p->pending_link, &file_priv->pending_event_list); // 把 drm_pending_event 挂入链表
		p->file_priv = file_priv;
drm_atomic_helper_setup_commit();
		new_crtc_state->event->base.completion = &commit->flip_done;
		new_crtc_state->event->base.completion_release = release_crtc_commit;
```

接着进入 crtc_funcs->atomic_disable() 回调，先调用 drm_crtc_vblank_off() 关掉 vblank 中断。

再调用 drm_crtc_vblank_on() 打开中断。

```c++
void drm_crtc_vblank_on(struct drm_crtc *crtc)
{
	struct drm_device *dev = crtc->dev;
	unsigned int pipe = drm_crtc_index(crtc);
	struct drm_vblank_crtc *vblank = drm_crtc_vblank_crtc(crtc);

	drm_reset_vblank_timestamp(dev, pipe);

	if (atomic_read(&vblank->refcount) != 0 || drm_vblank_offdelay == 0)
		drm_WARN_ON(dev, drm_vblank_enable(dev, pipe));
}


static int drm_vblank_enable(struct drm_device *dev, unsigned int pipe)
{
	struct drm_vblank_crtc *vblank = drm_vblank_crtc(dev, pipe);
	int ret = 0;


	if (!vblank->enabled) {
		ret = __enable_vblank(dev, pipe); // crtc->funcs->enable_vblank
		if (ret) {
			atomic_dec(&vblank->refcount);
		} else {
			drm_update_vblank_count(dev, pipe, 0);
			WRITE_ONCE(vblank->enabled, true);
		}
	}

	return ret;
}
```

进入 driver 中断，在中断中调用 drm_crtc_handle_vblank(), 这个函数的作用有唤醒 commit 过程后面的
drm_atomic_helper_wait_for_vblanks() 阻塞等待 vblank 的函数，并且向 userspace 发送通过 drm_crtc_arm_vblank_event()
添加到 vblank_event_list 中的 pending event。

```c++
drm_crtc_handle_vblank();
	wake_up(&vblank->queue); // 唤醒 drm_atomic_helper_wait_for_vblanks()
	// 对通过 drm_crtc_arm_vblank_event() 保存到 vblank_event_list 的 pending event 处理
	drm_handle_vblank_events(dev, pipe); 
```

还需要调用 drm_crtc_send_vblank_event() 向 userspace 发送 event 事件。

```c++
void drm_crtc_send_vblank_event(struct drm_crtc *crtc,
				struct drm_pending_vblank_event *e)
{
	struct drm_device *dev = crtc->dev;
	u64 seq;
	unsigned int pipe = drm_crtc_index(crtc);
	ktime_t now;

	if (drm_dev_has_vblank(dev)) {
		seq = drm_vblank_count_and_time(dev, pipe, &now);
	}
	e->pipe = pipe;
	send_vblank_event(dev, e, seq, now);
}

send_vblank_event();
	drm_send_event_timestamp_locked();
		drm_send_event_helper();
			if (e->completion) {
			// 这边是 drm_atomic_helper_setup_commit() 中设定的
			// new_crtc_state->event->base.completion = &commit->flip_done;
			complete_all(e->completion); 			
			e->completion_release(e->completion);
			e->completion = NULL;
	}
```

在完成 send_vblank_event() 之后，唤醒 drm_atomic_helper_wait_for_vblanks() ，结束当前的 atomic commit.

</br>

注意到各类驱动处理 vblank event 有两种方式：

第一种一般在 crtc_funcs->atomic_flush 中调用 drm_crtc_arm_vblank_event() 将 drm event 放入 pending_event_list 中，等到进入 isr 的时候，只需调用 drm_crtc_handle_vblank(), 会把 drm event 发送给 userspace。

第二种在 isr 中调用 drm_crtc_handle_vblank() + drm_crtc_send_vblank_event()，这时候 pending_event_list 为空因为之前没有调用过 drm_crtc_arm_vblank_event(), 所以需要手动调用 drm_crtc_send_vblank_event() 来发送 drm event 给 userspace。

drm_crtc_arm_vblank_event() 函数注释中写到，需要在整个 atomic commit sequence 完成后才能打开 vblank 中断，否则会出现竞态同步问题。
