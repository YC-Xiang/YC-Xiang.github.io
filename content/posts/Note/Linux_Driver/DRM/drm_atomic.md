---
date: 2024-08-28T13:43:54+08:00
title: "Drm -- Atomic"
tags:
  - DRM
categories:
  - DRM
draft: true
---

```c
struct drm_crtc_commit {
    struct drm_crtc *crtc;
    struct kref ref;
    struct completion flip_done;
    struct completion hw_done;
    struct completion cleanup_done;
    struct list_head commit_entry;
    struct drm_pending_vblank_event *event;
    bool abort_completion;
};
```

`flip_done`: 显示 buffer 切换后置起。drm_atomic_helper_setup_commit 中，new_crtc_state->event->base.completion = &commit->flip_done。在 drm_crtc_send_vblank_event 中 complete_all(e->completion)。

`hw_done`: 所有硬件操作都完成后置起。在 drm_atomic_helper_commit_hw_done 中 complete_all(hw_done)

`cleanup_done`: clean 工作完成后置起。在 drm_atomic_helper_commit_cleanup_done 中 complete_all(cleanup_done)

三者的顺序为 hw_done->flip_done->cleanup_done

```c
struct drm_atomic_state {
    struct kref ref;
    struct drm_device *dev;
    bool allow_modeset : 1;
    bool legacy_cursor_update : 1;
    bool async_update : 1;
    bool duplicated : 1;
    struct __drm_planes_state *planes;
    struct __drm_crtcs_state *crtcs;
    int num_connector;
    struct __drm_connnectors_state *connectors;
    int num_private_objs;
    struct __drm_private_objs_state *private_objs;
    struct drm_modeset_acquire_ctx *acquire_ctx;
    struct drm_crtc_commit *fake_commit;
    struct work_struct commit_work;
};
```

`allow_modeset`: 通过 userspace 传递的 flag DRM_MODE_ATOMIC_ALLOW_MODESET

`legacy_cursor_update`：已废弃。

`async_update`: asynchronous plane update.

`duplicated`: 表示 atomic_state 是否有被复制过。

# Driver Private State

## 数据结构

```c

```

## 函数流程

drm_atomic_private_obj_init，底层 driver 调用来初始化 private object
