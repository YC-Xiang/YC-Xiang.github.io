---
date: 2024-08-28T10:55:27+08:00
title: "DRM(6) -- Plane"
tags:
  - DRM
categories:
  - DRM
---

在扫描输出过程中，一个平面（plane）代表一个图像源，CRTC 可以对平面进行混合（blend）或叠加（overlaid）显示。plane 从 drm_framebuffer 获取输入数据。平面本身定义了图像的裁剪（cropping）和缩放（scaling）方式，以及它在 display pipeline 可见区域中的位置。平面还可以具有额外的属性来指定平面像素的定位和混合方式，比如旋转（rotation）或 Z 轴位置（Z-position）。所有这些属性都存储在 drm_plane_state 中。

plane 由 `struct drm_plane` 表示，使用`drm_universal_plane_init()` 初始化。

每个 plane 都有一个类型，参考 `enum drm_plane_type`。

每个 CRTC 都必须要有一个 primary plane。

# Data structure and api

```c++
struct drm_plane {
	struct drm_device *dev;
	struct list_head head;
	char *name;
	struct drm_modeset_lock mutex;
	struct drm_mode_object base;
	uint32_t possible_crtcs;
	uint32_t *format_types;
	unsigned int format_count;
	uint64_t *modifiers;
	unsigned int modifier_count;
	const struct drm_plane_funcs *funcs;
	struct drm_object_properties properties;
	enum drm_plane_type type;
	unsigned index;
	const struct drm_plane_helper_funcs *helper_private;
	struct drm_plane_state *state;
	struct drm_property *alpha_property;
	struct drm_property *zpos_property;
	struct drm_property *rotation_property;
	struct drm_property *blend_mode_property;
	struct drm_property *color_encoding_property;
	struct drm_property *color_range_property;
	struct drm_property *scaling_filter_property;
	struct drm_property *hotspot_x_property;
	struct drm_property *hotspot_y_property;
};
```

`possible_crtcs`: 该 plane 支持的 crtc 类型，在 plane 初始化时传入。

`format_types`: plane 支持的 pixel format 数组，在 plane 初始化时传入。

`format_count`: format_types 数组的大小。

`modifiers`: plane 支持的 modifier 数组，在 plane 初始化时传入。

`modifier_count`: modifiers 数组的大小。

```c++
struct drm_plane_state {
	struct drm_plane *plane;
	struct drm_crtc *crtc;
	struct drm_framebuffer *fb;
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
	struct drm_crtc_commit *commit;
	struct drm_atomic_state *state;
	bool color_mgmt_changed : 1;
};
```

`crtc`: 通过 drm_atomic_set_crtc_for_plane 绑定的 crtc

`fb`: 通过 drm_atomic_set_fb_for_plane 绑定的 fb

`crtc_x/y/w/h`: 设置该 plane 可见区域的起始位置和大小。

`src_x/y/w/h`: 从 framebuffer 中取 pixels 的区域起始位置和大小

![](https://xyc-1316422823.cos.ap-shanghai.myqcloud.com/20241104172917.png)

当 SRC 与 CRTC 的 X/Y 不相等时，则实现了平移的效果；
当 SRC 与 CRTC 的 W/H 不相等时，则实现了缩放的效果；
当 SRC 与 FrameBuffer 的 W/H 不相等时，则实现了裁剪的效果；

`hotspot_x/y`:

`alpha`: 该 plane 的透明度，0x0 为全透明，0xff 为不透明。需要调用 drm_plane_create_alpha_property() 来创建该 property。

`pixel_blend_mode`: 当前 plane 和背景 blend 的模式。有三种 blend mode, DRM_MODE_BLEND_PIXEL_NONE, DRM_MODE_BLEND_PREMULTI, DRM_MODE_BLEND_COVERAGE。
三者计算公式不同，具体参考 drm_blend.c 注释。

`rotation`: 旋转/镜像 plane，需要通过 drm_plane_create_rotation_property() 创建该 property。

`zpos`: plane 的叠加优先顺序。需要通过 drm_plane_create_zpos_property() 或 drm_plane_create_zpos_immutable_property() 创建该 property。

`normalized_zpos`: 相比于 zpos 可以自行设定值，normalize 后的 zpos 在范围 0~N-1, N 为 plane 的数量。

`color_encoding`: 平面 YUV 编码，包括 BT601, BT709, BT2020。

`color_range`: 平面 YUV 颜色范围，包括 limited range, full range。

`fb_damage_clips`:

`ignore_damage_clips`:

`src/dst`: 经过 drm_atomic_helper_check_plane_state clip 后的 fb 源地址和 plane 目标地址。硬件编程最好使用这个属性，而不是前面的 crtc_x/y/w/h 和 src_x/y/w/h。

`visible`: 表示该 plane 是否可见，可在 plane atomic_check, atomic_update 中来检查一下该属性。

`scaling_filter`: 参考 enum drm_scaling_filter。大部分 driver 都不支持。

`color_mgmt_changed`: 表示 color management 属性是否被改变。注意这边 crtc_state 中也有一样的 color_mgmt_changed，在代码中看到一般都是操作 crtc_state 的 color_mgmt_changed。

```c++
struct drm_plane_funcs {
  void (*destroy)(struct drm_plane *plane);
  void (*reset)(struct drm_plane *plane);
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

`destroy`: optional, 在 drm_mode_config_cleanup() 中被调用，设置为 drm_plane_cleanup 即可。
如果使用的 drmm_xxx 初始化 plane 那么就不用设置该回调。

`reset`: optional，reset plane state。通用接口 drm_atomic_helper_plane_reset。
如果 driver 有 private plane state 结构体，那么需要使用自己的回调。

`atomic_duplicate_state`: 和 crtc 相关回调类似，drm_atomic_helper_plane_duplicate_state  

`atomic_destroy_state`: 同上，drm_atomic_helper_plane_destroy_state  

`atomic_set_property`: 同上  

`atomic_get_property`: 同上  

`late_register`: 同上  

`early_unregister`: 同上  

`atomic_print_state`: 同上

`format_mod_supported`: 检查某个 format 和某个 modifier 的**组合**是否支持，单独检查 format 是否支持在 drm_plane_check_pixel_format() 就做过了。

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
};
```

`prepare_fb`: optional hook, framebuffer 准备工作，包括 pin framebuffer backing storage, relocate into contiguous VRAM, flush cache 等。如果没实现，那么在 drm_atomic_helper_prepare_planes 中会自动调用 drm_gem_plane_helper_prepare_fb(). 如果有额外的任务，那么 driver 自行实现该回调，并在该回调中调用 drm_gem_plane_helper_prepare_fb。

`cleanup_fb`: optional hook, 清理 .prepare_fb 分配的 resource。

`begin_fb_access`: optional hook, 和 prepare_fb 类似。这两组回调用的 driver 目前很少。
`end_fb_access`: optional hook, free resources in begin_fb_access。

注意这边 prepare_fb/cleanup_fb 和 begin_fb_access/end_fb_access 这两组回调函数的区别在于 cleanup_fb 和 end_fb_access 的调用实时机不同。end_fb_access 在 drm_atomic_helper_commit_planes() 完成后就会调用，这时候还没 enable crtc，在 commit 完成后就释放。而 cleanup_fb 在 drm_atomic_helper_cleanup_planes() 所有完成才会被调用。

`atomic_check`: optional hook, check plane specific constraints。可在回调中调用 drm_atomic_helper_check_plane_state()。

`atomic_update`: 更新 plane state。

`atomic_enable`: enable plane。

`atomic_disable`: disable plane。

API：

```c++
int drm_universal_plane_init(struct drm_device *dev,
			     struct drm_plane *plane,
			     uint32_t possible_crtcs,
			     const struct drm_plane_funcs *funcs,
			     const uint32_t *formats,
			     unsigned int format_count,
			     const uint64_t *format_modifiers,
			     enum drm_plane_type type,
			     const char *name, ...);
#define drmm_universal_plane_alloc(dev, type, member, possible_crtcs, funcs, formats, \
				   format_count, format_modifiers, plane_type, name, ...)
static inline unsigned int drm_plane_index(const struct drm_plane *plane);
static inline struct drm_plane *drm_plane_find(struct drm_device *dev,
		struct drm_file *file_priv,
		uint32_t id);
#define drm_for_each_plane(plane, dev);
bool drm_any_plane_has_format(struct drm_device *dev,
			      u32 format, u64 modifier);
```
