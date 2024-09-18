---
date: 2024-08-20T17:02:27+08:00
title: DRM Subsystem 3 -- Hardware Driver
tags:
  - DRM
categories:
  - DRM
hide:
  - true
---

# 中间层

`struct drm_driver`

# 底层 driver

## xxx_drv.c

首先初始化一个`struct drm_driver`

```c
DEFINE_DRM_GEM_DMA_FOPS(fops);

static const struct drm_driver mxsfb_driver = {
	.driver_features	= DRIVER_GEM | DRIVER_MODESET | DRIVER_ATOMIC,
	DRM_GEM_DMA_DRIVER_OPS,
	.fops	= &fops,
	.name	= "mxsfb-drm",
	.desc	= "MXSFB Controller DRM",
	.date	= "20160824",
	.major	= 1,
	.minor	= 0,
};
```

其中`.driver_feature`支持的属性有：

```c
enum drm_driver_feature {

	DRIVER_GEM = BIT(0), // GEM 内存管理，一般都需要选上
	DRIVER_MODESET = BIT(1), // KMS, kernel mode setting
	DRIVER_RENDER = BIT(3), // dedicated render node
	DRIVER_ATOMIC = BIT(4), // 支持所有modesetting userspace API。
	DRIVER_SYNCOBJ = BIT(5), // 支持&drm_syncobj，用来主动同步
	DRIVER_SYNCOBJ_TIMELINE = BIT(6),
	DRIVER_COMPUTE_ACCEL = BIT(7), // 支持compute acceleration。和DRIVER_RENDER,DRIVER_MODESET选项互斥，如果同时支持modeset和compute_accel那么就需要实现两个driver。
	DRIVER_GEM_GPUVA = BIT(8), // 支持user定义的GPU VA bindings for GEM objects
	DRIVER_CURSOR_HOTSPOT = BIT(9), // 支持cursor hotspot information
	// 下面这些都是legacy driver的属性
	DRIVER_USE_AGP = BIT(25), // AGP support, 不清楚作用，已不使用
	DRIVER_LEGACY = BIT(26), // 使用shadow attach的legacy driver，已淘汰
	DRIVER_PCI_DMA = BIT(27), // Legacy PCI DMA supprot, 已不使用
	DRIVER_SG = BIT(28), // Legacy scatter/gather DMA support, 已不使用
	DRIVER_HAVE_DMA	= BIT(29), // Legacy DMA support, 已不使用
	DRIVER_HAVE_IRQ	= BIT(30), // Legacy irq support, 已不使用
};
```

在 driver probe 函数中调用`devm_drm_dev_alloc()`函数分配`struct drm_device`空间，`drm_dev_alloc()`中又会调用`drm_dev_init()`来进行一系列初始化`struct drm_device`。

```c
static int mxsfb_probe(struct platform_device *pdev)
{
	struct drm_device *drm;

	// 初始化drm_device，填充一些common的成员
	// 用devm_drm_dev_alloc代替
	drm_dev_alloc(&mxsfb_driver, &pdev->dev);

	// device_get_match_data获取of_device_id中对应的.data
	// 底层driver初始化，填充drm_device一些和具体driver相关的成员
	mxsfb_load(drm, device_get_match_data(&pdev->dev));

	// drm_device初始化完成后调用
	drm_dev_register(drm, 0);

	drm_fbdev_dma_setup(drm, 32);
}

static int mxsfb_load(struct drm_device *drm,
		      const struct mxsfb_devdata *devdata)
{
	drmm_mode_config_init(drm);
	mxsfb_kms_init(mxsfb);
	drm_vblank_init(drm, drm->mode_config.num_crtc);

	drm_crtc_vblank_off(&mxsfb->crtc);
	mxsfb_attach_bridge(mxsfb);

	drm_mode_config_reset(drm);
	drm_kms_helper_poll_init(drm);
	drm_helper_hpd_irq_event(drm);
}
```

## xxx_kms.c

```c
int mxsfb_kms_init(struct mxsfb_drm_private *mxsfb)
{
	// 初始化primary plane
	drm_plane_helper_add(&mxsfb->planes.primary,
				&mxsfb_plane_primary_helper_funcs);
	drm_universal_plane_init(mxsfb->drm, &mxsfb->planes.primary, 1,
				       &mxsfb_plane_funcs,
				       mxsfb_primary_plane_formats,
				       ARRAY_SIZE(mxsfb_primary_plane_formats),
				       mxsfb_modifiers, DRM_PLANE_TYPE_PRIMARY,
				       NULL);
	// 初始化overlay plane
	...

	//
}
```
