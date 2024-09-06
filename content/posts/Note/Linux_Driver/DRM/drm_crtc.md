---
date: 2024-09-04T15:00:18+08:00
title: "Drm_crtc"
tags:
  - DRM
categories:
  - DRM
hide:
  - true
---

# 数据结构

```c
struct drm_crtc {
    struct drm_device *dev;
    struct device_node *port;
    struct list_head head;
    char *name;
    struct drm_modeset_lock mutex;
    struct drm_mode_object base;
    struct drm_plane *primary; // for legacy IOCTL
    struct drm_plane *cursor; // for legacy IOCTL
    unsigned index;
    int cursor_x; // legacy
    int cursor_y; // legacy
    bool enabled; // legacy
    struct drm_display_mode mode;  // legacy
    struct drm_display_mode hwmode; // legacy
    int x; // legacy
    int y; // legacy
    const struct drm_crtc_funcs *funcs;
    uint32_t gamma_size; // legacy
    uint16_t *gamma_store; // legacy
    const struct drm_crtc_helper_funcs *helper_private;
    struct drm_object_properties properties;
    struct drm_property *scaling_filter_property;
    struct drm_crtc_state *state;
    struct list_head commit_list;
    spinlock_t commit_lock;
    struct dentry *debugfs_entry;
    struct drm_crtc_crc crc;
    unsigned int fence_context;
    spinlock_t fence_lock;
    unsigned long fence_seqno;
    char timeline_name[32];
    struct drm_self_refresh_data *self_refresh_data;
};
```

```c
struct drm_crtc_state {
    struct drm_crtc *crtc;
    bool enable;
    bool active;
    bool planes_changed : 1;
    bool mode_changed : 1;
    bool active_changed : 1;
    bool connectors_changed : 1;
    bool zpos_changed : 1;
    bool color_mgmt_changed : 1;
    bool no_vblank : 1;
    u32 plane_mask;
    u32 connector_mask;
    u32 encoder_mask;
    struct drm_display_mode adjusted_mode;
    struct drm_display_mode mode;
    struct drm_property_blob *mode_blob;
    struct drm_property_blob *degamma_lut;
    struct drm_property_blob *ctm;
    struct drm_property_blob *gamma_lut;
    u32 target_vblank;
    bool async_flip;
    bool vrr_enabled;
    bool self_refresh_active;
    enum drm_scaling_filter scaling_filter;
    struct drm_pending_vblank_event *event;
    struct drm_crtc_commit *commit;
    struct drm_atomic_state *state;
};
```

```c
struct drm_crtc_funcs {
    void (*reset)(struct drm_crtc *crtc);
    int (*cursor_set)(struct drm_crtc *crtc, struct drm_file *file_priv, uint32_t handle, uint32_t width, uint32_t height);
    int (*cursor_set2)(struct drm_crtc *crtc, struct drm_file *file_priv,uint32_t handle, uint32_t width, uint32_t height, int32_t hot_x, int32_t hot_y);
    int (*cursor_move)(struct drm_crtc *crtc, int x, int y);
    int (*gamma_set)(struct drm_crtc *crtc, u16 *r, u16 *g, u16 *b,uint32_t size, struct drm_modeset_acquire_ctx *ctx);
    void (*destroy)(struct drm_crtc *crtc);
    int (*set_config)(struct drm_mode_set *set, struct drm_modeset_acquire_ctx *ctx);
    int (*page_flip)(struct drm_crtc *crtc,struct drm_framebuffer *fb,struct drm_pending_vblank_event *event,uint32_t flags, struct drm_modeset_acquire_ctx *ctx);
    int (*page_flip_target)(struct drm_crtc *crtc,struct drm_framebuffer *fb,struct drm_pending_vblank_event *event,uint32_t flags, uint32_t target, struct drm_modeset_acquire_ctx *ctx);
    int (*set_property)(struct drm_crtc *crtc, struct drm_property *property, uint64_t val);
    struct drm_crtc_state *(*atomic_duplicate_state)(struct drm_crtc *crtc);
    void (*atomic_destroy_state)(struct drm_crtc *crtc, struct drm_crtc_state *state);
    int (*atomic_set_property)(struct drm_crtc *crtc,struct drm_crtc_state *state,struct drm_property *property, uint64_t val);
    int (*atomic_get_property)(struct drm_crtc *crtc,const struct drm_crtc_state *state,struct drm_property *property, uint64_t *val);
    int (*late_register)(struct drm_crtc *crtc);
    void (*early_unregister)(struct drm_crtc *crtc);
    int (*set_crc_source)(struct drm_crtc *crtc, const char *source);
    int (*verify_crc_source)(struct drm_crtc *crtc, const char *source, size_t *values_cnt);
    const char *const *(*get_crc_sources)(struct drm_crtc *crtc, size_t *count);
    void (*atomic_print_state)(struct drm_printer *p, const struct drm_crtc_state *state);
    u32 (*get_vblank_counter)(struct drm_crtc *crtc);
    int (*enable_vblank)(struct drm_crtc *crtc);
    void (*disable_vblank)(struct drm_crtc *crtc);
    bool (*get_vblank_timestamp)(struct drm_crtc *crtc,int *max_error,ktime_t *vblank_time, bool in_vblank_irq);
};
```

`.reset`:

# 函数

初始化 crtc 函数:

```c
drmm_crtc_alloc_with_planes(dev, type, member, primary, cursor, funcs, name, ...);
void *__drmm_crtc_alloc_with_planes(struct drm_device *dev,
				    size_t size, size_t offset,
				    struct drm_plane *primary,
				    struct drm_plane *cursor,
				    const struct drm_crtc_funcs *funcs,
				    const char *name, ...)
int drmm_crtc_init_with_planes(struct drm_device *dev, struct drm_crtc *crtc,
			       struct drm_plane *primary,
			       struct drm_plane *cursor,
			       const struct drm_crtc_funcs *funcs,
			       const char *name, ...)
```