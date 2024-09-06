---
date: 2024-08-20T17:01:27+08:00
title: DRM Subsystem 2 -- Userspace API
tags:
  - DRM
categories:
  - DRM
hide:
  - true
---

# ioctl

首先给出`IOCTL`的 API, 在下面附上对应的`libdrm`的 API。

## Driver identification and capabilities

`include/uapi/drm/drm.h`

1.获取 `drm_driver` 定义的 major/minor/patchlevel:

```c
struct drm_version version = { ... };
ioctl(drm_fd, DRM_IOCTL_VERSION, &version);

// libdrm
drmVersionPtr drmGetVersion(int fd);
```

2.获取 driver 支持的 capability，传入需要判断的 capability`get_cap.capability`，返回`get_cap.value`。

```c
struct drm_get_cap get_cap = { 0 };
get_cap.capability = DRM_CAP_DUMB_BUFFER;
ioctl(drm_fd, DRM_IOCTL_GET_CAP, &get_cap);

// libdrm
int drmGetCap(int fd, uint64_t capability, uint64_t *value);
```

3.通知 kernel, client 支持的 capability，分别传入需要设置的 capability 和 value, kernel 会把支持的属性保存到`struct drm_file`中。

```c
struct drm_set_client_cap client_cap = { 0 };
client_cap.capability = DRM_CLIENT_CAP_UNIVERSAL_PLANES;
client_cap.value = 1;
ioctl(drm_fd, DRM_IOCTL_SET_CLIENT_CAP, &client_cap);

// libdrm
int drmSetClientCap(int fd, uint64_t capability, uint64_t value);
```

4.用户空间支持多个 client 同时打开 drm device node，但只有一个**master** client 可以 config display(KMS)。通过如下两个 ioctrl set 和 drop master client。

```c
ioctl(drm_fd, DRM_IOCTL_SET_MASTER, NULL);
ioctl(drm_fd, DRM_IOCTL_DROP_MASTER, NULL);

// libdrm
int drmSetMaster(int fd);
int drmDropMaster(int fd);
```

## Memory Management

1.根据传入的 width height 和 bpp，分配一个 dumb buffer，返回 handle, pitch and size:

```c
struct drm_mode_create_dumb create_dumb = { ... };
ioctl(drm_fd, DRM_IOCTL_MODE_CREATE_DUMB, &create_dumb);

// libdrm
int drmModeCreateDumbBuffer(int fd, uint32_t width, uint32_t height, uint32_t bpp,
			uint32_t flags, uint32_t *handle, uint32_t *pitch,
			uint64_t *size);
```

2.销毁 dumb buffer:

```c
struct drm_mode_destroy_dumb destroy_dumb = { .handle = ..., };
ioctl(drm_fd, DRM_IOCTL_MODE_DESTROY_DUMB, &destroy_dumb);

// libdrm
int drmModeDestroyDumbBuffer(int fd, uint32_t handle);
```

3.创建 mmap 的 offset 参数，保存到 map_dump.offset:

```c
struct drm_mode_map_dumb map_dumb = { .handle = ..., };
ioctl(drm_fd, DRM_IOCTL_MODE_MAP_DUMB, &map_dumb);

// libdrm
int drmModeMapDumbBuffer(int fd, uint32_t handle, uint64_t *offset);
```

4.mmap dumb buffer 到 userspace:

```c
map = mmap(NULL, create_dumb.size, PROT_READ | PROT_WRITE, MAP_SHARED,
drm_fd, map_dumb.offset);

//libdrm
int drmMap(int fd, drm_handle_t handle, drmSize size, drmAddressPtr address);
```

5.munmap buffer:

```c
munmap(map, create_dumb.size);

// libdrm
int drmUnmap(drmAddress address, drmSize size);
```

## FourCC

`include/uapi/drm/drm_fourcc.h`

## KMS resources probing

1.获取 fb, crtc, connector, encoder 的数量以及各自的 id 保存到数组中:

```c
struct drm_mode_card_res res = { ... };
ioctl(drm_fd, DRM_IOCTL_MODE_GETRESOURCES, &res);
// u32 crtc_id = res->crtcs[0]
// u32 conn_id = res->connectors[0]

// libdrm
drmModeResPtr drmModeGetResources(int fd);
```

2.获取 plane 的数量和 id 数组：

```c
struct drm_mode_get_plane_res res = { ... };
ioctl(drm_fd, DRM_IOCTL_MODE_GETPLANERESOURCES, &res);

// libdrm
drmModePlaneResPtr drmModeGetPlaneResources(int fd);
```

3.传入`connector_id`，获取 connector 参数：

```c
struct drm_mode_get_connector get_connector = { .connector_id = ... };
ioctl(drm_fd, DRM_IOCTL_MODE_GETCONNECTOR, &get_connector);

// libdrm
drmModeConnectorPtr _drmModeGetConnector(int fd, uint32_t connector_id, int probe);
```

4.传入`encoder_id`，获取 encoder 参数：

```c
struct drm_mode_get_encoder get_encoder = { .encoder_id = ... };
ioctl(drm_fd, DRM_IOCTL_MODE_GETENCODER, &get_encoder);

// libdrm
drmModeEncoderPtr drmModeGetEncoder(int fd, uint32_t encoder_id);
```

5.注册和销毁 framebuffer，drm 最多支持 4 个 plane：

```c
struct drm_mode_fb_cmd2 fb_cmd2 = { ... };
ret = ioctl(drm_fd, DRM_IOCTL_MODE_ADDFB2, &fb_cmd2);

unsigned int fb_id = fb_cmd2.fb_id;
ret = ioctl(drm_fd, DRM_IOCTL_MODE_RMFB, &fb_id);

// libdrm
int drmModeAddFB2(int fd, uint32_t width, uint32_t height,
		uint32_t pixel_format, const uint32_t bo_handles[4],
		const uint32_t pitches[4], const uint32_t offsets[4],
		uint32_t *buf_id, uint32_t flags);
```

6.传入`ctrc_id`，获取 controller 参数：

```c
struct drm_mode_crtc crtc = { .crtc_id = ... };
ret = ioctl(drm_fd, DRM_IOCTL_MODE_GETCRTC, &crtc);

// libdrm
drmModeCrtcPtr drmModeGetCrtc(int fd, uint32_t crtcId);
```

设置 controller 参数：

```c
struct drm_mode_crtc crtc = { .crtc_id = ... };
ret = ioctl(drm_fd, DRM_IOCTL_MODE_SETCRTC, &crtc);

// libdrm
drmModeSetCrtc(int fd, uint32_t crtcId, uint32_t bufferId,
		   uint32_t x, uint32_t y, uint32_t *connectors, int count,
		   drmModeModeInfoPtr mode)
```
