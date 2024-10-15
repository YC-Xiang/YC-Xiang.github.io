---
date: 2024-08-28T13:40:27+08:00
title: "Drm -- KMS Core"
tags:
  - DRM
categories:
  - DRM
hide:
  - true
---

# 数据结构

## drm_mode_config_funcs

mode setting functions, 该结构体需要在底层 driver 中初始化。

```c
struct drm_mode_config_funcs {
    struct drm_framebuffer *(*fb_create)(struct drm_device *dev,
    		struct drm_file *file_priv, const struct drm_mode_fb_cmd2 *mode_cmd);
    const struct drm_format_info *(*get_format_info)(const struct drm_mode_fb_cmd2 *mode_cmd);
    void (*output_poll_changed)(struct drm_device *dev);
    enum drm_mode_status (*mode_valid)(struct drm_device *dev, const struct drm_display_mode *mode);
    int (*atomic_check)(struct drm_device *dev, struct drm_atomic_state *state);
    int (*atomic_commit)(struct drm_device *dev,struct drm_atomic_state *state, bool nonblock);
    struct drm_atomic_state *(*atomic_state_alloc)(struct drm_device *dev);
    void (*atomic_state_clear)(struct drm_atomic_state *state);
    void (*atomic_state_free)(struct drm_atomic_state *state);
};
```

`fb_create`: 创建 framebuffer 的回调函数，在 userspace 调用 drmModeAddFB2()之后会调用到。.fb_create 回调有两个 drm 通用的函数，drm_gem_fb_create()和 drm_gem_fb_create_with_dirty()可以直接使用。

`get_format_info`: optional hook. 如果实现了该回调，会在 drm_get_format_info() 函数中返回 driver 自定义的 pixel format。

`output_poll_changed`: deprecated hook。

`mode_valid`: device 范围的 constraints 可以在这里检查，crtc/encoder/bridge/connector 有各自的 mode_valid 回调。

`atomic_check`: 在进行 atomic_commit 前进行的检查。drm 提供了 helper function: **drm_atomic_helper_check()**，在其中会再分别调用 connector/plane/crtc funcs 中的 atomic_check 回调。可以直接使用该函数，或者在该函数上再包装一层，检查一些全局的限制。

`atomic_commit`: 应用所有的 property 修改，提交 commit。helper function: drm_atomic_helper_commit。

`atomic_state_alloc`: 在调用 drm_atomic_state_alloc()时，创建自定义的 xxx_atomic_state 结构体，subclass drm_atomic_state。已废弃，被 drm_private_state and drm_private_obj 代替。

`atomic_state_clear`: 同上，已废弃。

`atomic_state_free`: 同上，已废弃。

## drm_mode_config_helper_funcs

```c
struct drm_mode_config_helper_funcs {
	void (*atomic_commit_tail)(struct drm_atomic_state *state);
	int (*atomic_commit_setup)(struct drm_atomic_state *state);
};
```

`atomic_commit_tail`: optional hook, 不实现的话默认会调用 drm_atomic_helper_commit_tail。

`atomic_commit_setup`: optional hook, 用于在 atomic commit setup 最后增加一些对 drm_private_obj 的操作。

## drm_mode_config

mode_config，整个 graphics 的配置。

```c
struct drm_mode_config {
    struct mutex mutex;
    struct drm_modeset_lock connection_mutex;
    struct drm_modeset_acquire_ctx *acquire_ctx;
    struct mutex idr_mutex;
    struct idr object_idr;
    struct idr tile_idr;
    struct mutex fb_lock;
    int num_fb;
    struct list_head fb_list;
    spinlock_t connector_list_lock;
    int num_connector;
    struct ida connector_ida;
    struct list_head connector_list;
    struct llist_head connector_free_list;
    struct work_struct connector_free_work;
    int num_encoder;
    struct list_head encoder_list;
    int num_total_plane;
    struct list_head plane_list;
    struct raw_spinlock panic_lock;
    int num_crtc;
    struct list_head crtc_list;
    struct list_head property_list;
    struct list_head privobj_list;
    int min_width, min_height;
    int max_width, max_height;
    const struct drm_mode_config_funcs *funcs;
    bool poll_enabled;
    bool poll_running;
    bool delayed_event;
    struct delayed_work output_poll_work;
    struct mutex blob_lock;
    struct list_head property_blob_list;
    struct drm_property *edid_property;
    struct drm_property *dpms_property;
    struct drm_property *path_property;
    struct drm_property *tile_property;
    struct drm_property *link_status_property;
    struct drm_property *plane_type_property;
    struct drm_property *prop_src_x;
    struct drm_property *prop_src_y;
    struct drm_property *prop_src_w;
    struct drm_property *prop_src_h;
    struct drm_property *prop_crtc_x;
    struct drm_property *prop_crtc_y;
    struct drm_property *prop_crtc_w;
    struct drm_property *prop_crtc_h;
    struct drm_property *prop_fb_id;
    struct drm_property *prop_in_fence_fd;
    struct drm_property *prop_out_fence_ptr;
    struct drm_property *prop_crtc_id;
    struct drm_property *prop_fb_damage_clips;
    struct drm_property *prop_active;
    struct drm_property *prop_mode_id;
    struct drm_property *prop_vrr_enabled;
    struct drm_property *dvi_i_subconnector_property;
    struct drm_property *dvi_i_select_subconnector_property;
    struct drm_property *dp_subconnector_property;
    struct drm_property *tv_subconnector_property;
    struct drm_property *tv_select_subconnector_property;
    struct drm_property *legacy_tv_mode_property;
    struct drm_property *tv_mode_property;
    struct drm_property *tv_left_margin_property;
    struct drm_property *tv_right_margin_property;
    struct drm_property *tv_top_margin_property;
    struct drm_property *tv_bottom_margin_property;
    struct drm_property *tv_brightness_property;
    struct drm_property *tv_contrast_property;
    struct drm_property *tv_flicker_reduction_property;
    struct drm_property *tv_overscan_property;
    struct drm_property *tv_saturation_property;
    struct drm_property *tv_hue_property;
    struct drm_property *scaling_mode_property;
    struct drm_property *aspect_ratio_property;
    struct drm_property *content_type_property;
    struct drm_property *degamma_lut_property;
    struct drm_property *degamma_lut_size_property;
    struct drm_property *ctm_property;
    struct drm_property *gamma_lut_property;
    struct drm_property *gamma_lut_size_property;
    struct drm_property *suggested_x_property;
    struct drm_property *suggested_y_property;
    struct drm_property *non_desktop_property;
    struct drm_property *panel_orientation_property;
    struct drm_property *writeback_fb_id_property;
    struct drm_property *writeback_pixel_formats_property;
    struct drm_property *writeback_out_fence_ptr_property;
    struct drm_property *hdr_output_metadata_property;
    struct drm_property *content_protection_property;
    struct drm_property *hdcp_content_type_property;
    uint32_t preferred_depth, prefer_shadow;
    bool quirk_addfb_prefer_xbgr_30bpp;
    bool quirk_addfb_prefer_host_byte_order;
    bool async_page_flip;
    bool fb_modifiers_not_supported;
    bool normalize_zpos;
    struct drm_property *modifiers_property;
    struct drm_property *size_hints_property;
    uint32_t cursor_width, cursor_height;
    struct drm_atomic_state *suspend_state;
    const struct drm_mode_config_helper_funcs *helper_private;
};
```

`async_page_flip`: 不用等待 vsync，在 init 中初始化，目前 kernel 中只有少数 driver 像 amd, vc4 支持。

# 函数

`drmm_mode_config_init`, 在底层 driver 初始化中直接调用，初始化 drm_mode_config 结构体：

`drm_mode_config_reset`, 调用 plane, crtc, encoder, connector 的 reset 回调，可以在底层 driver 的 load/resume code 中来 reset 硬件和软件状态。

`drm_mode_config_cleanup`, 调用 plane, crtc, encoder, connector 的 destroy 回调，free up 所有的资源，如果调用了`drmm_mode_config_init`，那么底层 driver 无需手动调用该 cleanup 函数。
