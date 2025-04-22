---
date: 2024-09-18T17:22:44+08:00
title: "DRM(7) -- Connector"
tags:
  - DRM
categories:
  - DRM
---

DRM connector 是对显示接收器 (display sink) 的抽象，包括固定的 panels, 或者其他任何可以显示像素的东西。

通过 `drm_connector_init()` 和 `drm_connector_register()` 初始化和注册。

connector 使用前必须 attach 到 encoder 上，对于 encoder 和 connector 1:1 的情况，在初始化流程中调用 `drm_connector_attach_encoder()`.

# Data Structure

```c++
struct drm_connector {
	struct drm_device *dev;
	struct device *kdev;
	struct device_attribute *attr;
	struct fwnode_handle *fwnode;
	struct list_head head;
	struct list_head global_connector_list_entry;
	struct drm_mode_object base;
	char *name;
	struct mutex mutex;
	unsigned index;
	int connector_type;
	int connector_type_id;
	bool interlace_allowed;
	bool doublescan_allowed;
	bool stereo_allowed;
	bool ycbcr_420_allowed;
	enum drm_connector_registration_state registration_state;
	struct list_head modes;
	enum drm_connector_status status;
	struct list_head probed_modes;
	struct drm_display_info display_info;
	const struct drm_connector_funcs *funcs;
	struct drm_property_blob *edid_blob_ptr;
	struct drm_object_properties properties;
	uint8_t polled;
	const struct drm_connector_helper_funcs *helper_private;
	struct drm_cmdline_mode cmdline_mode;
	enum drm_connector_force force;
	u64 epoch_counter;
	u32 possible_encoders;
	struct drm_encoder *encoder;
	struct i2c_adapter *ddc;
	struct dentry *debugfs_entry;
	struct drm_connector_state *state;
};
```

`connector_type`: DRM_MODE_CONNECTOR_XXX。

`connector_type_id`: 自动分配的 connector id。

`interlace_allowed`: driver 设置，只在 drm_helper_probe_single_connector_modes() 中验证 display mode 用到。  
`doublescan_allowed`:同上  
`stereo_allowed`:同上  
`ycbcr_420_allowed`:同上

`registration_state`: connector 的注册状态，在 drm_connector_register 和 unregister 中修改。

`modes`: .fill_modes(drm_helper_probe_single_connector_modes)->__drm_helper_update_and_validate->drm_connector_list_update
把 probe_modes 复制 modes

`status`: connector 的状态，由 connector->funcs->detect 返回，如果没实现则在 probe 过程中设置为 connector_status_connected。

`probed_modes`: 在 connector->funcs->get_modes 中把支持的 display mode 加入 probed_modes 链表。

`display_info`: non hot-pluggable displays 需要在 panel driver 中主动设置。hpd connector 会自动通过 edid 获取。

`polled`: 有三个宏，DRM_CONNECTOR_POLL_HPD: connector 能主动 detect 到 hotplug 并发出 hotplug event。
DRM_CONNECTOR_POLL_CONNECT：需要 polling 是否发生了 connect。
DRM_CONNECTOR_POLL_DISCONNECT：需要 polling 是否发生了 disconnect。如果 polled 设置为 0，表示不支持检测 connection status 变化。

`cmdline_mode`: kernel cmdline 可以设定 connector 参数。

`force`: connector force mode, 可以通过 kernel cmdline, sysfs 设置。

`possible_encoders`: drm_connector_attach_encoder() 中设置。

`state`: drm_connector_state。

</br>

drm_display_info 只列出了和 fixed panel 相关的 fields:

```c++
struct drm_display_info {
	unsigned int width_mm;
	unsigned int height_mm;
	unsigned int bpc;
	int panel_orientation;
	const u32 *bus_formats;
	unsigned int num_bus_formats;
	u32 bus_flags;
}
```

```c++
struct drm_connector_funcs {
  void (*reset)(struct drm_connector *connector);
  enum drm_connector_status (*detect)(struct drm_connector *connector,
              bool force);
  void (*force)(struct drm_connector *connector);
  int (*fill_modes)(struct drm_connector *connector, uint32_t max_width, uint32_t max_height);
  int (*late_register)(struct drm_connector *connector);
  void (*early_unregister)(struct drm_connector *connector);
  void (*destroy)(struct drm_connector *connector);
  struct drm_connector_state *(*atomic_duplicate_state)(struct drm_connector *connector);
  void (*atomic_destroy_state)(struct drm_connector *connector,
             struct drm_connector_state *state);
  int (*atomic_set_property)(struct drm_connector *connector,
           struct drm_connector_state *state,
           struct drm_property *property,
           uint64_t val);
  int (*atomic_get_property)(struct drm_connector *connector,
           const struct drm_connector_state *state,
           struct drm_property *property,
           uint64_t *val);
  void (*atomic_print_state)(struct drm_printer *p,
           const struct drm_connector_state *state);
  void (*oob_hotplug_event)(struct drm_connector *connector,
          enum drm_connector_status status);
  void (*debugfs_init)(struct drm_connector *connector, struct dentry *root);
};
```

`reset`: optional，reset connector。通用接口 drm_atomic_helper_connector_reset。

`detect`: optional, legacy support, 检测 connector 是否 attached，如果没提供该回调，那么默认 connector 一直是 attached 的。
atomic driver 实现 helper function 中的.detect_ctx 即可。

`force`: optional, 当 connector 被 userspace force 进入一个状态时调用，更新 encoder 的状态。

`fill_modes`: mandatory, 获取支持的 display modes，在 userspace getconnector ioctl 时会调用到，
通用接口 drm_helper_probe_single_connector_modes。

`late_register`: optional, 用来注册一些 userspace 自定义的 sysfs, debugfs 接口  
`early_unregister`: optional, 注销 late_register 中的接口

`destroy`: 通用接口 drm_connector_cleanup。如果使用 drmm_connector_init() 则不用指定。

`atomic_duplicate_state`: mandatory, 通用接口 drm_atomic_helper_connector_duplicate_state  
`atomic_destroy_state`: mandatory, 通用接口 drm_atomic_helper_connector_destroy_state

`atomic_set_property`: optional, 设置 driver 自定义的 property
`atomic_get_property`: optional, 获取 driver 自定义的 property

`atomic_print_state`: optional, 打印 driver 自定义的 state

`oob_hotplug_event`: optional

`debugfs_init`: 创建 connector 相关的 debugfs

```c++
struct drm_connector_helper_funcs {
  int (*get_modes)(struct drm_connector *connector);
  int (*detect_ctx)(struct drm_connector *connector,
        struct drm_modeset_acquire_ctx *ctx,
        bool force);
  enum drm_mode_status (*mode_valid)(struct drm_connector *connector,
             struct drm_display_mode *mode);
  int (*mode_valid_ctx)(struct drm_connector *connector,
            struct drm_display_mode *mode,
            struct drm_modeset_acquire_ctx *ctx,
            enum drm_mode_status *status);
  struct drm_encoder *(*best_encoder)(struct drm_connector *connector);
  struct drm_encoder *(*atomic_best_encoder)(struct drm_connector *connector,
               struct drm_atomic_state *state);
  int (*atomic_check)(struct drm_connector *connector,
          struct drm_atomic_state *state);
  void (*atomic_commit)(struct drm_connector *connector,
            struct drm_atomic_state *state);
  int (*prepare_writeback_job)(struct drm_writeback_connector *connector,
             struct drm_writeback_job *job);
  void (*cleanup_writeback_job)(struct drm_writeback_connector *connector,
              struct drm_writeback_job *job);
  void (*enable_hpd)(struct drm_connector *connector);
  void (*disable_hpd)(struct drm_connector *connector);
};
```

`get_modes`: mandatory, 获取 connector 支持的所有 display mode, 有两种方法，一种通过 edid,
另一种通过 fixed specific modes 来填充 drm_display_mode 结构体，通过 drm_add_edid_modes() 或 drm_mode_probed_add()
保存进 &drm_connector.probed_modes list. 在.fill_modes(drm_helper_probe_single_connector_modes)->drm_helper_probe_get_modes 中被调用。

`detect_ctx`: optional, 判断 connector 状态，代替 drm_connector_funcs 中的.detect 回调

`mode_valid`: optional, 检查 connector 从 fixed panel, edid 获取到的 modes 有效性。
drm_helper_probe_single_connector_modes()->__drm_helper_update_and_validate()->drm_mode_validate_pipeline()->drm_connector_mode_valid()

`mode_valid_ctx`: optional, mode_valid 的 atomic 版本

`best_encoder`: 如果 connector 通过 drm_connector_attach_encoder() attach 了多个 possible encoders, 那么需要实现该回调来选择某个 encoder。

`atomic_best_encoder`: best_encoder 的 atomic 版本。

`atomic_check`: 检查 connector state。

`atomic_commit`: 给 writeback connectors commit hardware 用的。

`prepare_writeback_job`:

`cleanup_writeback_job`:

`enable_hpd`:

`disable_hpd`:

```c++
struct drm_connector_state {
	struct drm_connector *connector;
	struct drm_crtc *crtc;
	struct drm_encoder *best_encoder;
	enum drm_link_status link_status;
	struct drm_atomic_state *state;
	struct drm_crtc_commit *commit;
	struct drm_tv_connector_state tv;
	bool self_refresh_aware;
	enum hdmi_picture_aspect picture_aspect_ratio;
	unsigned int content_type;
	unsigned int hdcp_content_type;
	unsigned int scaling_mode;
	unsigned int content_protection;
	enum drm_colorspace colorspace;
	struct drm_writeback_job *writeback_job;
	u8 max_requested_bpc;
	u8 max_bpc;
	enum drm_privacy_screen_status privacy_screen_sw_state;
	struct drm_property_blob *hdr_output_metadata;
};
```

# Hotplug

检查 connector 状态的方式有两种，第一种 polling，第二种为 hotplug。

首先需要 connector 初始化时，指定 connector.polled, DRM_CONNECTOR_POLL_HPD 表示支持 hotplug,connector 可以生成 hotplug event。
DRM_CONNECTOR_POLL_CONNECT 和 DRM_CONNECTOR_POLL_DISCONNECT 表示周期性 polling connector status。
如果 polled 为 0，那么表示不支持检测 connector 状态。

## 方式 1 polling

在 driver init 中调用 drm_kms_helper_poll_init 即可。在 polling 中检测到 connector status 改变就会向 userspace 发送 uevent。

## 方式 2 Hotplug Interrupt

仍然需要首先调用 drm_kms_helper_poll_init，其中会调用到 connector_helper_funcs->enable_hpd 来 enable hpd。

后面需要在 hotplug 中断处理函数中调用 drm_helper_hpd_irq_event(处理多个 connector)
或 drm_connector_helper_hpd_irq_event(单个 connector) 来进行 hotplug processing，向 userspace 发送 uevent。

# WriteBack connector

用于将显示内容写入内存缓冲区。

用于屏幕录制、截图等场景。支持视频编码和流媒体传输。

与普通 connector 的区别：

普通 connector：将内容输出到显示设备（如显示器）。

writeback connector：将内容输出到内存缓冲区。

所以，writeback connector 本质上是一个"虚拟"的 connector，它不连接到实际的显示设备，而是将显示内容捕获到内存中，
为各种需要访问显示内容的应用程序提供支持。
