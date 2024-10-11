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

## drm_crtc

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

## drm_crtc_state

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

`enable`: gate all other state。控制 crtc 是否需要 enable，控制 resource assignment

`active`: 控制 crtc hardware state， userspace 通过 "ACTIVE" property 设置

`XXX_changed`: 用于控制 atomic commit flow，在底层 driver .atomic_check 回调中可以修改，以及可以通过 drm_atomic_get_new_crtc_state()获取到 new state 的 XXX_changed 来控制对应的操作 flow  
`plane_changed`: 在 drm_atomic_helper_check_planes 中更新，只要 plane_state->crtc 不为 NULL，则 plane_changed 为 true  
`mode_changed`: 在 drm_atomic_helper_check_modeset 中更新。当 old 和 new crtc_state->mode 或 crtc_state->enable 改动时，置为 true。底层 driver 可在 plane/crtc .atomic_check 回调中修改该 flag 表示是否需要 full modeset  
`active_changed`: 在 drm_atomic_helper_check_modeset 中更新。当 old 和 new crtc_state->active 或 crtc_state->active 改动时，置为 true  
`connectors_changed`: 底层 driver 在 drm_encoder_helper_funcs.atomic_check 中更新  
`zpos_changed`:  
`color_mgmt_changed`: 在 drm_atomic_crtc_set_property 中更新，表示 userspace 传入的 color LUT 是否更新

`no_blank`: 不支持 vblank 的 driver 会在 drm_atomic_helper_check_modeset 中自动置起

`plane/connector/encoder_mask`: 连接到该 crtc 的 plane/connector/encoder mask

`adjusted_mode`: 底层 driver 最终使用的 mode 参数  
`mode`: userspace request 的 mode 参数

`degamma_lut`: ctm 前的 color LUT
`ctm`: Color transformation matrix  
`gamma_lut`: ctm 后的 color LUT

## drm_crtc_funcs

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

`reset`: optional，reset crtc。通用接口 drm_atomic_helper_crtc_reset

`cursor_set`: optional, **deprecated** legacy support  
`cursor_set2`: optional, **deprecated** legacy support  
`cursor_move`: optional, **deprecated** legacy support

`gamma_set`: optional, legacy support for color LUT。atomic driver 直接修改 gamma_lut_property 即可。

`destroy`: optional, 在 drm_mode_config_cleanup()中被调用，设置为 drm_crtc_cleanup 即可。

`set_config`: **mandatory hook**, legacy 设置 crtc 的入口，atomic driver 设置为 drm_atomic_helper_set_config。

`page_flip`: optional , legacy interface page flip 入口，atomic driver 直接设置为 drm_atomic_helper_page_flip。  
`page_flip_target`: optional, 和 page_flip 类似，可以额外指定 target。

`set_property`: optional, legacy support。

`atomic_duplicate_state`: **mandatory hook**, 底层 driver 没有 subclass drm_crtc_state 的直接设置为 drm_atomic_helper_crtc_duplicate_state。否则自定义函数分配 drm_crtc_state，再调用\_\_drm_atomic_helper_crtc_duplicate_state()。  
`atomic_destroy_state`: **mandatory hook**， 设置为 drm_atomic_helper_crtc_destroy_state。

`atomic_set_property`: optional, 设置 driver-private property。在 drm_atomic_crtc_set_property 中被调用。
`atomic_get_property`: optional, 获取 driver-private property。在 drm_atomic_crtc_get_property 中被调用。

`late_register`: optional, 在这个回调函数中注册 userspace crtc 相关的 debugfs 接口。  
`early_unregister`: optional, unregister 上面的接口。

`set_crc_source`: optional, userspace 通过 open /sys/kernel/debug/dri/0/crtc-N/crc/data 来设置 CRC enable/disable，以及 CRC 源。  
`verify_crc_source`: optional, userspace 通过往 /sys/kernel/debug/dri/0/crtc-N/crc/control 写入"auto"等字符串，来检查某个 CRC 源，并返回 value_cnt，每个 drm_crtc_crc_enrty 有几个 CRC。  
`get_crc_sources`: optional, userspace 通过 open /sys/kernel/debug/dri/0/crtc-N/crc/control 获取 all the available sources for CRC generation。

`atomic_print_state`: 如果底层 driver subclass 了 drm_crtc_state，需要把自定义的 state 打印出来。

`get_vblank_counter`: optional, 获取当前硬件的 vblank 数量，如果为 NULL, kernel 会根据 timestamp 计算在中断中错过的 vblank 事件。

`enable_vblank`: 对于支持 vblank 的 driver **必须要实现**，在这里 enable vblank interrupt。  
`disable_vblank`: 同理，用来 disable vblank interrupt。

`get_vblank_timestamp`: optional, vblank interval end precise timestamp。

## drm_crtc_helper_funcs

```c
struct drm_crtc_helper_funcs {
	void (*dpms)(struct drm_crtc *crtc, int mode);
	void (*prepare)(struct drm_crtc *crtc);
	void (*commit)(struct drm_crtc *crtc);
	enum drm_mode_status (*mode_valid)(struct drm_crtc *crtc,
					   const struct drm_display_mode *mode);
	bool (*mode_fixup)(struct drm_crtc *crtc,
			   const struct drm_display_mode *mode,
			   struct drm_display_mode *adjusted_mode);

	int (*mode_set)(struct drm_crtc *crtc, struct drm_display_mode *mode,
			struct drm_display_mode *adjusted_mode, int x, int y,
			struct drm_framebuffer *old_fb);
	void (*mode_set_nofb)(struct drm_crtc *crtc);
	int (*mode_set_base)(struct drm_crtc *crtc, int x, int y,
			     struct drm_framebuffer *old_fb);

	int (*mode_set_base_atomic)(struct drm_crtc *crtc,
				    struct drm_framebuffer *fb, int x, int y,
				    enum mode_set_atomic);
	void (*disable)(struct drm_crtc *crtc);
	int (*atomic_check)(struct drm_crtc *crtc,
			    struct drm_atomic_state *state);
	void (*atomic_begin)(struct drm_crtc *crtc,
			     struct drm_atomic_state *state);
	void (*atomic_flush)(struct drm_crtc *crtc,
			     struct drm_atomic_state *state);
	void (*atomic_enable)(struct drm_crtc *crtc,
			      struct drm_atomic_state *state);
	void (*atomic_disable)(struct drm_crtc *crtc,
			       struct drm_atomic_state *state);
	bool (*get_scanout_position)(struct drm_crtc *crtc,
				     bool in_vblank_irq, int *vpos, int *hpos,
				     ktime_t *stime, ktime_t *etime,
				     const struct drm_display_mode *mode);
};
```

`dpms`: optional, **legacy support**. atomic driver 不用实现，用 atomic_enable/disable 接口代替。  
`prepare`: optional, **legacy support**. 用来 disable crtc，atomic driver 不用实现，用 atomic_disable 代替。  
`commit`: optional, **legacy support**. 用来 enable crtc，atomic driver 不用实现，用 atomic_enable 代替。

`mode_valid`: 检查 userspace 传入的 drm_display_mode 是否合法，返回 enum drm_mode_status。

`mode_fixup`: optional, 修改 userspace 传入的 drm_display_mode，把修改后的参数保存到 adjusted_mode。

`mode_set`: deprecated, **legacy support**.  
`mode_set_base`: deprecated, **legacy support**.

`mode_set_nofb`: optional, 设置 display mode，包括所有的时序 timing。注意，需要支持 runtime PM 的 driver 不能使用这个回调（进 suspend 后寄存器可能会 reset 而不会 restore），需要将所有的操作移到 atomic_enable 中。

`disable`: optional, atomic driver 使用 atomic_disable 代替。

`atomic_check`: optional, 检查 plane-update 相关的 CRTC 限制。在函数 drm_atomic_helper_check_planes 中被调用。

`atomic_begin`: optional, atomic update 前做一些准备工作。

`atomic_flush`: optional, 一般会调用 drm_crtc_send_vblank_event，在 page flip 之后发送 vblank event。

`atomic_enable`: optional, enable crtc。

`atomic_disable`: optional, disable crtc。
