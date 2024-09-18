---
date: 2024-08-28T10:55:27+08:00
title: DRM Subsystem 4 -- Plane
tags:
  - DRM
categories:
  - DRM
hide:
  - true
---

# 数据结构

```c
struct drm_plane_state {
    struct drm_plane *plane; // backpointer指向plane
    struct drm_crtc *crtc; // 通过drm_atomic_set_crtc_for_plane绑定的crtc
    struct drm_framebuffer *fb; // 通过drm_atomic_set_fb_for_plane绑定的fb
    struct dma_fence *fence; //
    int32_t crtc_x; //
    int32_t crtc_y;
    uint32_t crtc_w, crtc_h;
    uint32_t src_x;
    uint32_t src_y;
    uint32_t src_h, src_w;
    int32_t hotspot_x, hotspot_y;
    u16 alpha;
    uint16_t pixel_blend_mode;
    unsigned int rotation;
    unsigned int zpos;
    unsigned int normalized_zpos;
    enum drm_color_encoding color_encoding;
    enum drm_color_range color_range;
    struct drm_property_blob *fb_damage_clips;
    bool ignore_damage_clips;
    struct drm_rect src, dst;
    bool visible;
    enum drm_scaling_filter scaling_filter;
    struct drm_crtc_commit *commit;
    struct drm_atomic_state *state;
    bool color_mgmt_changed : 1;
};
```

```c
struct drm_plane_helper_funcs {
	int (*prepare_fb)(struct drm_plane *plane,
			  struct drm_plane_state *new_state);
	void (*cleanup_fb)(struct drm_plane *plane,
			   struct drm_plane_state *old_state);
	int (*begin_fb_access)(struct drm_plane *plane, struct drm_plane_state *new_plane_state);
	void (*end_fb_access)(struct drm_plane *plane, struct drm_plane_state *new_plane_state);
	int (*atomic_check)(struct drm_plane *plane,
			    struct drm_atomic_state *state);
	void (*atomic_update)(struct drm_plane *plane,
			      struct drm_atomic_state *state);
	void (*atomic_enable)(struct drm_plane *plane,
			      struct drm_atomic_state *state);
	void (*atomic_disable)(struct drm_plane *plane,
			       struct drm_atomic_state *state);
	int (*atomic_async_check)(struct drm_plane *plane,
				  struct drm_atomic_state *state);
	void (*atomic_async_update)(struct drm_plane *plane,
				    struct drm_atomic_state *state);
	int (*get_scanout_buffer)(struct drm_plane *plane,
				  struct drm_scanout_buffer *sb);
	void (*panic_flush)(struct drm_plane *plane);
};
```

`prepare_fb`: 如果没实现，那么在 drm_atomic_helper_prepare_planes 中会调用 drm_gem_plane_helper_prepare_fb()代替。

`begin_fb_access`: 和 prepare_fb 类似，主要是给使用 shadow buffer 的 driver。

`atomic_check`: check plane specific constraints。可调用 drm_atomic_helper_check_plane_state()。

`atomic_update`: 更新 plane state。
