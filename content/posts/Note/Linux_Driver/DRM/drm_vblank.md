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
