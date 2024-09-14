---
date: 2024-08-20T17:01:27+08:00
title: DRM Subsystem 2 -- Userspace Application
tags:
  - DRM
categories:
  - DRM
hide:
  - true
---

参考 https://github.com/dvdhrm/docs/tree/master/drm-howto 中的 modeset-atomic.c

**首先 open drm 设备节点：**

```c
fd = open("/dev/dri/card0", O_RDWR | O_CLOEXEC);
```

</br>

**设置 client 的 capability：**

```c
drmSetClientCap(fd, DRM_CLIENT_CAP_UNIVERSAL_PLANES, 1);
drmSetClientCap(fd, DRM_CLIENT_CAP_ATOMIC, 1);
```

第一行设置 drm file_priv->universal_planes=1。  
第二行设置 file_priv->atomic=1, file_priv->universal_planes=1, file_priv->aspect_ratio_allowed = 1。

</br>

**获取 client 的 capability:**

```c
drmGetCap(fd, DRM_CAP_DUMB_BUFFER, &cap);
drmGetCap(fd, DRM_CAP_CRTC_IN_VBLANK_EVENT, &cap);
```

第一行，如果 drm_driver 提供了 dumb_create 回调，则返回 1，一般都会提供。
第二行，必定返回 1，现在 driver 都会支持 Vblank 事件。

</br>

**获取 resources:**

```c
drmModeGetResources();
drmIoctl(fd, DRM_IOCTL_MODE_GETRESOURCES, &res);
```

得到 fbs/crtcs/connectors/encoders 的数量，以及每个对应的 id，还有显示支持的最大最小长宽，填充结构体：

```c
typedef struct _drmModeRes {
	int count_fbs;
	uint32_t *fbs;
	int count_crtcs;
	uint32_t *crtcs;
	int count_connectors;
	uint32_t *connectors;
	int count_encoders;
	uint32_t *encoders;
	uint32_t min_width, max_width;
	uint32_t min_height, max_height;
} drmModeRes, *drmModeResPtr;
```

**获取 connector:**

根据上面的 connectors id，获取 connector

```c
drmModeGetConnector(fd, res->connectors[i]);
drmIoctl(fd, DRM_IOCTL_MODE_GETCONNECTOR, &conn);
```

填充该结构体：

```c
typedef struct _drmModeConnector {
	uint32_t connector_id; // 传入的connector id
	uint32_t encoder_id; // 对应的encoder id
	uint32_t connector_type; // 在connector_init中分配的DRM_MODE_CONNECTOR_XXX
	uint32_t connector_type_id; // 在connector_init中分配的一个unique id
	drmModeConnection connection;
	uint32_t mmWidth, mmHeight; //panel的物理长宽
	drmModeSubPixel subpixel;

	int count_modes;
	drmModeModeInfoPtr modes;

	int count_props;
	uint32_t *props; /**< List of property ids */
	uint64_t *prop_values; /**< List of property values */

	int count_encoders;
	uint32_t *encoders; /**< List of encoder ids */
} drmModeConnector, *drmModeConnectorPtr;
```
