---
date: 2024-08-28T13:40:27+08:00
title: "DRM -- VBlank"
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

`pipe`: 表示第几个 crtc，在 drm_vblank_init 中初始化。

`refcount`: vblank interrupt user/waiter 数量。

`max_vblank_count`: vblank register 最大的范围，如果不为 0 表示支持硬件 vblank count 计数，底层 driver 初始化该数值，并且 drm_crtc_funcs.get_vblank_counter 必须提供。

`inmodeset`: 表示是否在 modeset 过程中，1 vblank is disabled, 0 vblank is enabled.

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

在 userspace 进行 atomic commit 之后，会先进入 crtc_funcs->atomic_disable() 回调，在这里
调用 drm_crtc_vblank_off().

```c++
void drm_crtc_vblank_off(struct drm_crtc *crtc)
{
	struct drm_device *dev = crtc->dev;
	unsigned int pipe = drm_crtc_index(crtc);
	struct drm_vblank_crtc *vblank = &dev->vblank[pipe];
	struct drm_pending_vblank_event *e, *t;
	ktime_t now;
	u64 seq;

	spin_lock_irq(&dev->event_lock);

	spin_lock(&dev->vbl_lock);

	if (drm_core_check_feature(dev, DRIVER_ATOMIC) || !vblank->inmodeset)
		drm_vblank_disable_and_save(dev, pipe);

	wake_up(&vblank->queue);

	if (!vblank->inmodeset) {
		atomic_inc(&vblank->refcount);
		vblank->inmodeset = 1;
	}
	spin_unlock(&dev->vbl_lock);

	seq = drm_vblank_count_and_time(dev, pipe, &now);

	list_for_each_entry_safe(e, t, &dev->vblank_event_list, base.link) {
		if (e->pipe != pipe)
			continue;
		list_del(&e->base.link);
		drm_vblank_put(dev, pipe);
		send_vblank_event(dev, e, seq, now);
	}

	drm_vblank_cancel_pending_works(vblank);

	spin_unlock_irq(&dev->event_lock);

	vblank->hwmode.crtc_clock = 0;

	drm_vblank_flush_worker(vblank);
}

void drm_vblank_disable_and_save(struct drm_device *dev, unsigned int pipe)
{
	struct drm_vblank_crtc *vblank = &dev->vblank[pipe];
	unsigned long irqflags;

	assert_spin_locked(&dev->vbl_lock);

	spin_lock_irqsave(&dev->vblank_time_lock, irqflags);

	/// 如果之前没调用过 drm_crtc_vblank_on(), 那么可以直接返回。
	if (!vblank->enabled)
		goto out;

	drm_update_vblank_count(dev, pipe, false);
	__disable_vblank(dev, pipe);
	vblank->enabled = false;

out:
	spin_unlock_irqrestore(&dev->vblank_time_lock, irqflags);
}
```
