---
date: 2024-08-28T13:43:54+08:00
title: "Drm_atomic"
tags:
  - DRM
categories:
  - DRM
hide:
  - true
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

`allow_modeset`: 通过 userspace 传递 flag DRM_MODE_ATOMIC_ALLOW_MODESET

`legacy_cursor_update`：已废弃。

`async_update`: asynchronous plane update.

`duplicated`: 表示 atomic_state 是否有被复制过。
