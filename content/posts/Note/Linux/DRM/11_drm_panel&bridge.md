---
date: 2024-10-15T17:22:56+08:00
title: "DRM -- Panel and Bridge"
tags:
  - DRM
categories:
  - DRM
draft:
  - true
---

# Panel and Bridge

## 情况 1：设备树存在 panel 节点

imx6ull-dhcom-pdk2.dts 中的存在 panel 节点，对应 panel-simple.c:

```dts
&lcdif {
	status = "okay";

	port {
		display_out: endpoint {
			remote-endpoint = <&panel_in>;
		};
	};
};

panel {
	compatible = "auo,g101evn010";
	power-supply = <&ldo4_ext>;
	backlight = <&lcd_backlight>;

	port {
		panel_in: endpoint {
			remote-endpoint = <&display_out>;
		};
	};
};
```

这种情况比较简单，在底层 driver 中调用**drm_of_find_panel_or_bridge**找到设备树中的 panel 节点，和 panel driver 匹配，找到 panel driver 注册的 **drm_panel** 结构体。

再调用**devm_drm_panel_bridge_add**, 分配 panel_bridge, 注册一个固定的**drm_bridge**。

最后调用**drm_bridge_attach**, 调用到 bridge->funcs->attach, 即 panel_bridge_attach, 注册 connector，以及把 connector attach 到 encoder。

## 情况 2：设备树存在 bridge 节点, connector 节点

da850-lcdk.dts 中存在 bridge 和 connector 节点，对应 simple-bridge.c:

```dts
&lcdc {
	compatible = "ti,da850-tilcdc";
	status = "okay";

	port {
		lcdc_out_vga: endpoint {
			remote-endpoint = <&vga_bridge_in>;
		};
	};
};

vga-bridge {
	compatible = "ti,ths8135";
	#address-cells = <1>;
	#size-cells = <0>;

	ports {
		#address-cells = <1>;
		#size-cells = <0>;

		port@0 {
			reg = <0>;

			vga_bridge_in: endpoint {
				remote-endpoint = <&lcdc_out_vga>;
			};
		};

		port@1 {
			reg = <1>;

			vga_bridge_out: endpoint {
				remote-endpoint = <&vga_con_in>;
			};
		};
	};
};

vga {
	compatible = "vga-connector";

	ddc-i2c-bus = <&i2c0>;

	port {
		vga_con_in: endpoint {
			remote-endpoint = <&vga_bridge_out>;
		};
	};
};
```

底层 driver 仍然调用**drm_of_find_panel_or_bridge**函数，因为 panel 节点不存在，所以会找到设备树中的 bridge 节点，和 bridge driver 匹配，找到 bridge driver 注册的 **drm_bridge** 结构体。

再调用**drm_bridge_attach**, 调用到 bridge->funcs->attach, 即 simple_bridge_attach。

在这个函数中又会调用一次 drm_bridge_attach，不过这个时候传入的 bridge 为设备树中 port@1 中的 remote-endpoint 节点，对应的是 connector。

最后注册 connector，以及把 connector attach 到 encoder。

## 数据结构

```c
struct panel_bridge {
	struct drm_bridge bridge;
	struct drm_connector connector;
	struct drm_panel *panel;
	u32 connector_type;
};
```

panel bridge 是有 drm_panel 注册后，需要固定注册一个 drm_bridge。

```c
struct drm_panel {
	struct device *dev;
	struct backlight_device *backlight;
	const struct drm_panel_funcs *funcs;
	int connector_type;
	struct list_head list;
	struct list_head followers;
	struct mutex follower_lock;
	bool prepare_prev_first; // 确保在调用panel prepare之前，mipi dsi driver初始化完成, 主动置起该flag
	bool prepared; // panel是否prepared,在drm_panel_prepare中置起
	bool enabled; // panel是否enable,在drm_panel_enable中置起
};
```

```c
struct drm_panel_funcs {
	int (*prepare)(struct drm_panel *panel);
	int (*enable)(struct drm_panel *panel);
	int (*disable)(struct drm_panel *panel);
	int (*unprepare)(struct drm_panel *panel);
	int (*get_modes)(struct drm_panel *panel,
			 struct drm_connector *connector);
	enum drm_panel_orientation (*get_orientation)(struct drm_panel *panel);
	int (*get_timings)(struct drm_panel *panel, unsigned int num_timings,
			   struct display_timing *timings);
	void (*debugfs_init)(struct drm_panel *panel, struct dentry *root);
};
```

`prepare`: optinal, 做一些 setup 工作, 在 drm_panel_prepare 中调用

`enable`: optional, enable panel, 打开背光等，在 drm_panel_enable 中调用

`disable`: optional, 在 drm_panel_disable 中调用

`unprepare`: optional, 和 prepare 相反

`get_modes`: mandatory, 把 panel 支持的 mode 加入到 connector->probed_modes 中。在 drm_panel_get_modes 中调用，如果 drvier 绑定了 panel，那么 connector 的 get_modes 回调，直接调用 drm_panel_get_modes 就可以

`get_orientation`: optional, 从设备树或者 edid 中获取 panel 的方向

`get_timings`: optional, 返回 panel driver 中 fixed timing，这个回调看起来没怎么用到

```c
struct drm_bridge {
	struct drm_private_obj base;
	struct drm_device *dev;
	struct drm_encoder *encoder; // bridge连接的encoder
	struct list_head chain_node;
	struct device_node *of_node; // bridge在设备树中对应的节点
	struct list_head list;
	const struct drm_bridge_timings *timings;
	const struct drm_bridge_funcs *funcs;
	void *driver_private;
	enum drm_bridge_ops ops;
	int type; // bridge output的格式 DRM_MODE_CONNECTOR_*
	bool interlace_allowed; // bridge是否能处理interlace mode
	bool pre_enable_prev_first;
	struct i2c_adapter *ddc;
	struct mutex hpd_mutex;
	void (*hpd_cb)(void *data, enum drm_connector_status status);
	void *hpd_data;
};
```

```c
struct drm_bridge_funcs {
	int (*attach)(struct drm_bridge *bridge,
		      enum drm_bridge_attach_flags flags);
	void (*detach)(struct drm_bridge *bridge);
	enum drm_mode_status (*mode_valid)(struct drm_bridge *bridge,
					   const struct drm_display_info *info,
					   const struct drm_display_mode *mode);
	bool (*mode_fixup)(struct drm_bridge *bridge,
			   const struct drm_display_mode *mode,
			   struct drm_display_mode *adjusted_mode);
	void (*disable)(struct drm_bridge *bridge);
	void (*post_disable)(struct drm_bridge *bridge);
	void (*mode_set)(struct drm_bridge *bridge,
			 const struct drm_display_mode *mode,
			 const struct drm_display_mode *adjusted_mode);
	void (*pre_enable)(struct drm_bridge *bridge);
	void (*enable)(struct drm_bridge *bridge);
	void (*atomic_pre_enable)(struct drm_bridge *bridge,
				  struct drm_bridge_state *old_bridge_state);
	void (*atomic_enable)(struct drm_bridge *bridge,
			      struct drm_bridge_state *old_bridge_state);
	void (*atomic_disable)(struct drm_bridge *bridge,
			       struct drm_bridge_state *old_bridge_state);
	void (*atomic_post_disable)(struct drm_bridge *bridge,
				    struct drm_bridge_state *old_bridge_state);
	struct drm_bridge_state *(*atomic_duplicate_state)(struct drm_bridge *bridge);
	void (*atomic_destroy_state)(struct drm_bridge *bridge,
				     struct drm_bridge_state *state);
	u32 *(*atomic_get_output_bus_fmts)(struct drm_bridge *bridge,
					   struct drm_bridge_state *bridge_state,
					   struct drm_crtc_state *crtc_state,
					   struct drm_connector_state *conn_state,
					   unsigned int *num_output_fmts);
	u32 *(*atomic_get_input_bus_fmts)(struct drm_bridge *bridge,
					  struct drm_bridge_state *bridge_state,
					  struct drm_crtc_state *crtc_state,
					  struct drm_connector_state *conn_state,
					  u32 output_fmt,
					  unsigned int *num_input_fmts);
	int (*atomic_check)(struct drm_bridge *bridge,
			    struct drm_bridge_state *bridge_state,
			    struct drm_crtc_state *crtc_state,
			    struct drm_connector_state *conn_state);
	struct drm_bridge_state *(*atomic_reset)(struct drm_bridge *bridge);
	enum drm_connector_status (*detect)(struct drm_bridge *bridge);
	int (*get_modes)(struct drm_bridge *bridge,
			 struct drm_connector *connector);
	const struct drm_edid *(*edid_read)(struct drm_bridge *bridge,
					    struct drm_connector *connector);
	void (*hpd_notify)(struct drm_bridge *bridge,
			   enum drm_connector_status status);
	void (*hpd_enable)(struct drm_bridge *bridge);
	void (*hpd_disable)(struct drm_bridge *bridge);
	void (*debugfs_init)(struct drm_bridge *bridge, struct dentry *root);
};
```

`attach`: optional, attach bridge to encoder, invoked in **drm_bridge_attach**

`detach`: optional, detach bridge to encoder

`mode_valid`: optional, check display mode restraints in bridge

`mode_fixup`: optional, fix display mode and store into adjusted mode

`disable`: deprecated  
`post_disable`: deprecated  
`mode_set`: deprecated  
`pre_enable`: deprecated  
`enable`: deprecated

`atomic_pre_enable`: optional, enable bridge before the preceding element is enabled(like encoder enable function).

`atomic_enable`: optional, enable bridge after the preceding element is enabled

`atomic_disable`: optional, disable bridge before the preceding element is disabled

`atomic_post_disable`: optional, disable bridge after the preceding element is disabled

`atomic_duplicate_state`: mandatory, duplicate drm_bridge_state. If &drm_bridge_state is note subclassed, then use **drm_atomic_helper_bridge_duplicate_state**

`atomic_destroy_state`: mandatory, **drm_atomic_helper_bridge_destroy_state**

`atomic_get_output_bus_fmts`:

`atomic_get_input_bus_fmts`:

`atomic_check`: 检查

`atomic_reset`:

`detect`:

`get_modes`:

`edid_read`:

`hpd_notify`:

`hpd_enable`:

`hpd_disable`:
