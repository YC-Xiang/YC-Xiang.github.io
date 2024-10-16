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

为了支持 vblank，底层 drvier 需要调用 drm_vblank_init()初始化，另外需要实现 drm_crtc_funcs.enable_vblank 和 drm_crtc_funcs.disable_vblank 两个回调函数，并且在 vblank 中断中调用 drm_crtc_handle_vblank()。

# 数据结构

```c
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

`inmodeset`: 表示是否在 modeset 过程中，1 vblank is disabled, 0 vblank is enabled.

```c
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

`sequence`: 硬件 vblank 应该在该数量 trigger

# 函数

drm_crtc_arm_vblank_event: 需要保证在 atomic commit 所有硬件改动完成后，才会触发 vblank 中断。

drm_crtc_send_vblank_event
