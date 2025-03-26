---
date: 2024-08-20T17:01:27+08:00
title: DRM -- Userspace Application
tags:
  - DRM
categories:
  - DRM
---

参考 https://github.com/dvdhrm/docs/tree/master/drm-howto 中的 modeset-atomic.c

# Modeset prepare

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

</br>

**获取 connector:**

根据上面的 connectors id，获取 connector

```c
drmModeGetConnector(fd, res->connectors[i]);
drmIoctl(fd, DRM_IOCTL_MODE_GETCONNECTOR, &conn);
```

填充 connector 结构体：

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

</br>

**创建自定的 property blob：**

```c
drmModeCreatePropertyBlob(fd, &out->mode, sizeof(out->mode),
				      &out->mode_blob_id);
DRM_IOCTL(fd, DRM_IOCTL_MODE_CREATEPROPBLOB, &create);
```

</br>

**根据 connector 对应的 encoder id，获取 encoder, 再根据 encoder id 找到对应的 crtc id 保存起来：**

```c
drmModeGetEncoder(fd, conn->encoder_id);
drmIoctl(fd, DRM_IOCTL_MODE_GETENCODER, &enc);
```

填充 encoder 结构体：

```c
typedef struct _drmModeEncoder {
	uint32_t encoder_id;
	uint32_t encoder_type; // DRM_MODE_ENCODER_XXX
	uint32_t crtc_id;
	uint32_t possible_crtcs;
	uint32_t possible_clones;
} drmModeEncoder, *drmModeEncoderPtr;
```

</br>

**获取 plane resource, 包括 plane id 数组和 plane count:**

```c
drmModeGetPlaneResources(fd);
drmIoctl(fd, DRM_IOCTL_MODE_GETPLANERESOURCES, &res);
```

</br>

**根据 plane id 获取 plane：**

```c
drmModePlanePtr plane = drmModeGetPlane(fd, plane_id);
drmIoctl(fd, DRM_IOCTL_MODE_GETPLANE, &ovr);
```

**获取 plane,connector,crtc 的 properties：**

```c
modeset_get_object_properties(fd, connector, DRM_MODE_OBJECT_CONNECTOR);
modeset_get_object_properties(fd, crtc, DRM_MODE_OBJECT_CRTC);
modeset_get_object_properties(fd, plane, DRM_MODE_OBJECT_PLANE);

drmIoctl(fd, DRM_IOCTL_MODE_OBJ_GETPROPERTIES, &properties); // 获取crtc/plane/connector所有properties的id
drmIoctl(fd, DRM_IOCTL_MODE_GETPROPERTY, &prop); // 传入某个prop id，返回value
```

**创建 dumb buffer 以及 add framebuffer, 最后 mmap framebuffer:**

```c
drmIoctl(fd, DRM_IOCTL_MODE_CREATE_DUMB, &creq);
drmModeAddFB2(fd, buf->width, buf->height, DRM_FORMAT_XRGB8888,
			    handles, pitches, offsets, &buf->fb, 0);
DRM_IOCTL(fd, DRM_IOCTL_MODE_ADDFB2, &f)

buf->map = mmap(0, buf->size, PROT_READ | PROT_WRITE, MAP_SHARED,
		fd, mreq.offset);
```

# Modeset draw

绘制好需要下一帧显示的 framebuffer：

```c
modeset_paint_framebuffer(iter);
//...
*(uint32_t*)&buf->map[off] =
		(out->r << 16) | (out->g << 8) | out->b;
```

修改好各种 properties 后进行 atomic commit:

```c
drmModeAtomicAlloc();
set_drm_object_property(req, &out->connector, "CRTC_ID", out->crtc.id);
set_drm_object_property(req, plane, "SRC_X", 0);
//...
flags = DRM_MODE_ATOMIC_ALLOW_MODESET | DRM_MODE_PAGE_FLIP_EVENT;
drmModeAtomicCommit(fd, req, flags, NULL);
```

接着在 main 中 polling drm event 事件(通过 read drm fd)，等到 kernel 的 page flip done event 会进入 userspace 设置的回调函数中：

```c
drmEventContext ev;
ev.version = 3;
ev.page_flip_handler2 = modeset_page_flip_event; // kernel返回flip完成event后，会进入该回调

drmHandleEvent(fd, &ev);
```

在 modeset_page_flip_event 函数中，继续绘制下一帧，double buffer 中的另一块 framebuffer 接着再 atomic commit：

```c
modeset_page_flip_event();
	modeset_draw_out();

modeset_paint_framebuffer(out);
drmModeAtomicAlloc();
modeset_atomic_prepare_commit(fd, out, req);
flags = DRM_MODE_PAGE_FLIP_EVENT | DRM_MODE_ATOMIC_NONBLOCK;
drmModeAtomicCommit(fd, req, flags, NULL);
```
