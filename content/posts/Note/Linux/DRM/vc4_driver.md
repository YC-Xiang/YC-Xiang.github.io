---
date: 2024-08-20T17:02:27+08:00
title: DRM -- Hardware Driver
tags:
  - DRM
categories:
  - DRM
draft: true
---

# VC4 DRM driver

**vc4_hvs.c**: Hardware Video Scaler, 对 pixels 进行一些类似 translation, scaling, colorspace conversion, and compositing 的操作.

**vc4_vec.c**: vec encoder, 用来生成 PAL or NTSC 格式的 video output.

**vc4_hdmi.c**: hdmi module.

**vc4_dpi.c**: dpi module.

**vc4_dsi.c**: dsi module.

**vc4_txp.c**:

**vc4_crtc.c**: crtc module.

**vc4_v3d.c**: 3D 渲染模块.

## vc4_drv.c

```c
drm_firmware_drivers_only();
	video_firmware_drivers_only();
		return video_nomodeset;
```

其中 video_nomodeset 是一个 bool 类型, 如果在 kernel 命令行中传入"nomodeset", 那么会置为 true.
drm drvier register 函数会返回-ENODEV.

```c
static int __init disable_modeset(char *str)
{
	video_nomodeset = true;

	pr_warn("Booted with the nomodeset parameter. Only the system framebuffer will be available\n");

	return 1;
}

__setup("nomodeset", disable_modeset);
```
