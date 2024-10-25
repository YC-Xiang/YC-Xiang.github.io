---
date: 2024-08-28T09:55:27+08:00
title: DRM -- FrameBuffer
tags:
  - DRM
categories:
  - DRM
---

# Framebuffer

帧缓冲区是抽象的内存对象，提供了扫描到 CRTC 的像素源。应用程序通过 IOCTL`DRM_IOCTL_MODE_ADDFB(2)`来创建 framebuffer，会返回一个不透明句柄，该句柄可以传递给 KMS CRTC、来控制 plane configuration 和 page flip 功能。

## 数据结构

```c
struct drm_framebuffer {
    struct drm_device *dev;
    struct list_head head; // framebuffer链表，可能有多个fb
    struct drm_mode_object base;
    char comm[TASK_COMM_LEN]; // allocate fb的进程名
    const struct drm_format_info *format; // fb pixel format
    const struct drm_framebuffer_funcs *funcs;
    // 一行多少bytes, 会从用户空间的drm_mode_fb_cmd2拷贝过来
    unsigned int pitches[DRM_FORMAT_MAX_PLANES];
    // framebuffer和actual pixel data的offset，也从drm_mode_fb_cmd2拷贝过来
    unsigned int offsets[DRM_FORMAT_MAX_PLANES];
    uint64_t modifier; // 从drm_mode_fb_cmd2的modifier拷贝过来，DRM_FORMAT_MOD_XXX
    unsigned int width; // framebuffer宽
    unsigned int height; // framebuffer高
    int flags; // DRM_MODE_FB_INTERLACED, DRM_MODE_FB_MODIFIERS
    struct list_head filp_head;
    struct drm_gem_object *obj[DRM_FORMAT_MAX_PLANES];
};
```

```c
struct drm_format_info {
	u32 format; // FOURCC格式，DRM_FORMAT_*
	u8 depth; // color depth, legacy field, 设置为0。
	u8 num_planes; // Number of color planes (1 to 3)
	union {
		// 每个plane的bytes per pixel, legacy field。
		u8 cpp[DRM_FORMAT_MAX_PLANES];
		// 每个plane的bytes per block。用于单个pixel不是byte对齐的情况。
		// block大小用下面的block_w和block_h来描述。
		u8 char_per_block[DRM_FORMAT_MAX_PLANES];
	};
	u8 block_w[DRM_FORMAT_MAX_PLANES]; // block width占几个bytes
	u8 block_h[DRM_FORMAT_MAX_PLANES]; // block height占几个bytes
	u8 hsub; // 行采样因子
	u8 vsub; // 列采样因子，比如yuv422那么hsub=2, vsub=1
	bool has_alpha; // pixel format中是否含有alpha
	bool is_yuv; // 是不是yuv格式
	bool is_color_indexed; // 是不是color_indexed格式，即伪彩，存index进color LUT查找对应颜色
};
```

```c
struct drm_framebuffer_funcs {
	void (*destroy)(struct drm_framebuffer *framebuffer);
	int (*create_handle)(struct drm_framebuffer *fb,
			     struct drm_file *file_priv,
			     unsigned int *handle);
	// 有些硬件在fb内容更新后不会主动刷新内容到屏幕上。
	// userspace需要通过DRM_IOCTL_MODE_DIRTYFB ioctl调用到dirty函数来刷新屏幕的某块区域。
	int (*dirty)(struct drm_framebuffer *framebuffer,
		     struct drm_file *file_priv, unsigned flags,
		     unsigned color, struct drm_clip_rect *clips,
		     unsigned num_clips);
};
```

## 注册 framebuffer

用户空间通过`drmModeAddFB2()`或`drmModeAddFB2WithModifiers()`函数来注册 framebuffer，需要传入`struct drm_mode_fb_cmd2`，除了`fb_id`是返回的参数，其他都是需要传入的参数：

```c
struct drm_mode_fb_cmd2 {
	__u32 fb_id; // id
	__u32 width; // 宽
	__u32 height; // 高
	__u32 pixel_format; // FourCC 格式 DRM_FORMAT_XXXX drm_fourcc.h
	__u32 flags; // 可传入 DRM_MODE_FB_INTERLACED 和 DRM_MODE_FB_MODIFIERS
	__u32 handles[4]; // 内存 handle, 从 DRM_IOCTL_MODE_CREATE_DUMB ioctrl 获得
	__u32 pitches[4]; // 某个 plane 一行的 bytes 数
	__u32 offsets[4]; // 某个 plane 和 fb 起始地址的 offset, 比如yuv420 planer格式，那么第二个plane和fb是有offset的
	// DRM_FORMAT_MOD_XXX，自定义的fourcc格式，4个modifier都要和modifier[0]相同。
	// 另外DRM_MODE_FB_MODIFIERS 需要被置起，所有plane的modifier需要相同。
	// 底层driver一般还需要实现get_format_info回调来获取自定义format
	__u64 modifier[4];
};
```

```c
int drmModeAddFB2WithModifiers(int fd, uint32_t width,
		uint32_t height, uint32_t pixel_format, const uint32_t bo_handles[4],
		const uint32_t pitches[4], const uint32_t offsets[4],
		const uint64_t modifier[4], uint32_t *buf_id, uint32_t flags)

drm_mode_addfb2_ioctl();
	drm_mode_addfb2();
		drm_internal_framebuffer_create();
			framebuffer_check();
			dev->mode_config.funcs->fb_create();
```

`.fb_create`回调有两个 drm 通用的函数，`drm_gem_fb_create`和`drm_gem_fb_create_with_dirty`。

```c
struct drm_framebuffer *drm_gem_fb_create(struct drm_device *dev,
		struct drm_file *file, const struct drm_mode_fb_cmd2 *mode_cmd)
{
	return drm_gem_fb_create_with_funcs(dev, file, mode_cmd,
					    &drm_gem_fb_funcs);
}

struct drm_framebuffer * drm_gem_fb_create_with_dirty(struct drm_device *dev,
		struct drm_file *file, const struct drm_mode_fb_cmd2 *mode_cmd)
{
	return drm_gem_fb_create_with_funcs(dev, file, mode_cmd,
					    &drm_gem_fb_funcs_dirtyfb);
}
```

可以看出区别是传入`drm_gem_fb_create_with_func`的`drm_framebuffer_funcs`不同，多实现了一个`.dirty`回调,该回调的作用见上方。

```c
static const struct drm_framebuffer_funcs drm_gem_fb_funcs = {
	.destroy	= drm_gem_fb_destroy,
	.create_handle	= drm_gem_fb_create_handle,
};

static const struct drm_framebuffer_funcs drm_gem_fb_funcs_dirtyfb = {
	.destroy	= drm_gem_fb_destroy,
	.create_handle	= drm_gem_fb_create_handle,
	.dirty		= drm_atomic_helper_dirtyfb,
};
```

```c
drm_gem_fb_create_with_funcs();
	drm_gem_fb_init_with_funcs();
		drm_gem_fb_init();
			fb->obj[i] = obj[i];
			drm_helper_mode_fill_fb_struct(); /// 根据userspace传入的mode_cmd，填充fb结构体
				fb->dev = dev;
				fb->format = drm_get_format_info(dev, mode_cmd);
				fb->width = mode_cmd->width;
				fb->height = mode_cmd->height;
				for (i = 0; i < 4; i++) {
					fb->pitches[i] = mode_cmd->pitches[i];
					fb->offsets[i] = mode_cmd->offsets[i];
				}
				fb->modifier = mode_cmd->modifier[0];
				fb->flags = mode_cmd->flags;
			drm_framebuffer_init(); /// 填充fb->funcs和drm_mode_object
				fb->funcs = funcs;
				strcpy(fb->comm, current->comm);
				list_add(&fb->head, &dev->mode_config.fb_list);
				__drm_mode_object_add();

```

## afbc framebuffer

AFBC 是一种专有的无损图像压缩协议和格式。它提供细粒度的随机访问，并最大限度地减少 IP 块之间传输的数据量。

通过使用 drm_fourcc.h 中定义的 AFBC 格式修饰符(DRM_FORMAT_MOD_ARM_AFBC)，可以在支持它的驱动程序上启用 AFBC。

![](https://xyc-1316422823.cos.ap-shanghai.myqcloud.com/20241022154831.png)

在 SoC 中使用 AFBC 时，视频处理器只需以压缩格式写出视频流，GPU 则读取它们并且仅在片上内存中解压缩它们。完全相同的优化将应用到用于屏幕的输出缓冲。无论是 GPU 还是视频处理器生成最终的帧缓冲，它们都会被压缩，因此显示处理器将以 AFBC 格式读取它们并且仅在移到显示内存中时进行解压缩。

https://community.arm.com/arm-community-blogs/b/graphics-gaming-and-vr-blog/posts/mali-v500-video-processor-reducing-memory-bandwidth-with-afbc
