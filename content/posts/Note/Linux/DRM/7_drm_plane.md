---
date: 2024-08-28T10:55:27+08:00
title: DRM -- Plane
tags:
  - DRM
categories:
  - DRM
---

# 数据结构

```c++
struct drm_plane{

}
```

```c++
struct drm_plane_state {
    struct drm_plane *plane; // backpointer指向plane
    struct drm_crtc *crtc; // 通过drm_atomic_set_crtc_for_plane绑定的crtc
    struct drm_framebuffer *fb; // 通过drm_atomic_set_fb_for_plane绑定的fb
    struct dma_fence *fence;
    int32_t crtc_x;
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

`crtc_x/y/w/h`: 设置该 plane 可见区域的起始位置和大小。

`src_x/y/w/h`: 从 framebuffer 中取 pixels 的可见区域起始位置和大小

![](https://xyc-1316422823.cos.ap-shanghai.myqcloud.com/20241104172917.png)

当 SRC 与 CRTC 的 X/Y 不相等时，则实现了平移的效果；
当 SRC 与 CRTC 的 W/H 不相等时，则实现了缩放的效果；
当 SRC 与 FrameBuffer 的 W/H 不相等时，则实现了裁剪的效果；

`hotspot_x/y`:

`alpha`: 该 plane 的透明度,0x0 为全透明，0xff 为不透明。需要调用 drm_plane_create_alpha_property()来创建该 property。

`pixel_blend_mode`: 当前 plane 和背景 blend 的模式。有三种 blend mode, DRM_MODE_BLEND_PIXEL_NONE, DRM_MODE_BLEND_PREMULTI, DRM_MODE_BLEND_COVERAGE。
三者计算公式不同，具体参考 drm_blend.c 注释。

`rotation`: 旋转/镜像 plane，需要通过 drm_plane_create_rotation_property()创建该 property。

`zpos`: plane 的叠加优先顺序。需要通过 drm_plane_create_zpos_property() 或 drm_plane_create_zpos_immutable_property()创建该 property。

`normalized_zpos`: 相比于 zpos 可以自行设定值，normalize 后的 zpos 在范围 0~N-1, N 为 plane 的数量。

`color_encoding`: 设置非 RGB 格式的颜色编码，包括 BT601, BT709, BT2020。

`color_range`: 设置非 RGB 格式的颜色范围，包括 limited range, full range。

`fb_damage_clips`:

`ignore_damage_clips`:

`src/dst`: 经过 drm_atomic_helper_check_plane_state clip 后的 fb 源地址和 plane 目标地址。硬件编程最好使用这个属性，而不是前面的 crtc_x/y/w/h 和 src_x/y/w/h。

`visible`: 表示该 plane 是否可见，可在 plane atomic_check, atomic_update 中来检查一下该属性。

`scaling_filter`: 参考 enum drm_scaling_filter。大部分 driver 都不支持。

`color_mgmt_changed`: 表示 color management 属性是否被改变。注意这边 crtc_state 中也有一样的 color_mgmt_changed,在代码中看到一般都是操作 crtc_state 的 color_mgmt_changed。

```c++
struct drm_plane_funcs {
  int (*update_plane)(struct drm_plane *plane,
          struct drm_crtc *crtc, struct drm_framebuffer *fb,
          int crtc_x, int crtc_y,
          unsigned int crtc_w, unsigned int crtc_h,
          uint32_t src_x, uint32_t src_y,
          uint32_t src_w, uint32_t src_h,
          struct drm_modeset_acquire_ctx *ctx);
  int (*disable_plane)(struct drm_plane *plane,
           struct drm_modeset_acquire_ctx *ctx);
  void (*destroy)(struct drm_plane *plane);
  void (*reset)(struct drm_plane *plane);
  int (*set_property)(struct drm_plane *plane,
          struct drm_property *property, uint64_t val);
  struct drm_plane_state *(*atomic_duplicate_state)(struct drm_plane *plane);
  void (*atomic_destroy_state)(struct drm_plane *plane,
             struct drm_plane_state *state);
  int (*atomic_set_property)(struct drm_plane *plane,
           struct drm_plane_state *state,
           struct drm_property *property,
           uint64_t val);
  int (*atomic_get_property)(struct drm_plane *plane,
           const struct drm_plane_state *state,
           struct drm_property *property,
           uint64_t *val);
  int (*late_register)(struct drm_plane *plane);
  void (*early_unregister)(struct drm_plane *plane);
  void (*atomic_print_state)(struct drm_printer *p,
           const struct drm_plane_state *state);
  bool (*format_mod_supported)(struct drm_plane *plane, uint32_t format,
             uint64_t modifier);
};

```

`update_plane`: legacy support，ioctrl setplane 会调用到，直接用 drm_atomic_helper_update_plane
`disable_plane`: legacy support, 直接用 drm_atomic_helper_disable_plane

`destroy`: 和 crtc 相关回调类似, drm_plane_cleanup  
`reset`: 同上, drm_atomic_helper_plane_reset  
`set_preperty`: 同上  
`atomic_duplicate_state`: 同上, drm_atomic_helper_plane_duplicate_state  
`atomic_destroy_state`: 同上, drm_atomic_helper_plane_destroy_state  
`atomic_set_property`: 同上  
`atomic_get_property`: 同上  
`late_register`: 同上  
`early_unregister`: 同上  
`atomic_print_state`: 同上

`format_mod_supported`: 检查 format 和 modifier 是否支持。

```c++
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

`prepare_fb`: optional hook, 准备 framebuffer，包括 flush cache 等。如果没实现，那么在 drm_atomic_helper_prepare_planes 中会调用 drm_gem_plane_helper_prepare_fb()代替  
`cleanup_fb`: optional hook, free resources in prepare fb

`begin_fb_access`: optional hook, 和 prepare_fb 类似，主要是给使用 shadow buffer 的 driver  
`end_fb_access`: optional hook, free resources in begin_fb_access

`atomic_check`: optional hook, check plane specific constraints。可在回调中调用 drm_atomic_helper_check_plane_state()。

`atomic_update`: 更新 plane state。

# Format Modifiers
