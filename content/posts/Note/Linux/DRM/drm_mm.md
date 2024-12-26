---
date: 2024-12-10T14:16:27+08:00
title: "Drm -- Memory Management"
tags:
  - DRM
categories:
  - DRM
---

# Introduction

DRM 核心包括两个内存管理器，分别是 Translation Table Manager(TTM) 和 Graphics Execution Manager(GEM).

TTM 是第一个被开发的 DRM 内存管理器。它提供了一个单一的用户空间 API 来满足所有硬件的需求，支持统一内存体系结构(Unified Memory Architecture，UMA)设备和具有专用视频 RAM (即大多数离散显卡)的设备。这导致了一大段复杂的代码，结果很难用于驱动程序开发。

由于 TTM 的复杂性，GEM 最初是由英特尔(Intel)赞助的一个项目。GEM 没有提供每个图形内存相关问题的解决方案，而是确定了驱动程序之间的公共代码，并创建了一个支持库来共享它。GEM 的初始化和执行要求比 TTM 简单，但没有视频 RAM 管理功能，因此仅限于 UMA 设备。

# GEM Initialization

在 `struct drm_driver` driver feature 中设置 `DRIVER_GEM` bit.

中间层会自动调用`drm_gem_init()`来完成 GEM 的初始化.

# GEM Objects Creation

GEM object 由结构体 `struct drm_gem_object`表示, 通过`drm_gem_oject_init()`初始化, 利用 shmem 来分配 anonymous pageable memory.

如果 hardware 需要 physical contiguous system memory(通常是嵌入式设备需求), 那么可以不需要使用 shmem, 而是通过`drm_gem_private_object_init()`初始化.

# GEM Objects Lifetime

通过`drm_gem_object_get()`和`drm_gem_object_put()`来获取和释放 GEM reference.

当最后一次引用被释放时, 会调用`drm_gem_object_funcs`的 free 函数.

# GEM Ojects Naming

userspace 和 kernel 层有多种方式来引用 GEM objects, 比如 local handlers, global names or file descriptors.

**handle**

通过`drm_gem_handle_create()`创建 GEM object handle, 通过`drm_gem_object_lookup()`来获取 GEM object handle.

**name**

GEM name 和 GEM handle 类似, 和 handle 的区别是, handle is local to drm file while name is not. 所以 GEM name 可以在不同进程中来引用 GEM object. 通过 DRM_IOCTL_GEM_FLINK 和 DRM_IOCTL_GEM_OPEN 来在 name 和 handle 之间转换.

**dma buf file descriptors**

GEM also supports buffer sharing with **dma-buf** file descriptors through PRIME. 上面通过 global GEM name 的方式适用于老的 lagacy userspace api, 通过 dma-buf 的方式在进程间共享内存是更好的方式.

# GEM Objects Mapping

因为 mmap 系统调用不能直接映射 GEM object, 因为 GEM object 没有自己的 file handle.

因此需要对 DRM file handle 进行 mmap, 在映射之前，GEM 对象必须与 fake offset 关联。为此，驱动程序必须对 object 调用 `drm_gem_create_mmap_offset()` 来创建 fake offset。

GEM core 提供了`drm_gem_mmap()`来处理 object mapping. 会根据 offset value 查找 GEM object, 设置 VMA 操作为 struct drm_drvier 的 `gem_vm_ops`. 注意, drm_gem_mmap()不将内存映射到用户空间，而是依赖于驱动程序提供的 fault handler 来分别映射页面。

为了使用 drm_gem_mmap, 必须预先填充 struct drm_drvier 的 gem_vm_ops field.

# Core Structure

```c
struct drm_gem_object {
	struct kref refcount;
	unsigned handle_count;
	struct drm_device *dev;
	struct file *filp;
	struct drm_vma_offset_node vma_node;
	size_t size;
	int name;
	struct dma_buf *dma_buf;
	struct dma_buf_attachment *import_attach;
	struct dma_resv *resv;
	struct dma_resv _resv;
	struct {
		struct list_head list;
	} gpuva;
	const struct drm_gem_object_funcs *funcs;
	struct list_head lru_node;
	struct drm_gem_lru *lru;
};
```

`refcount`: gem object 的引用计数.

`handle_count`:

`dev`: 该 gem object 属于的 drm device.

`filp`: SHMEM file node used as backing storage for swappable buffer objects. GEM 还支持 driver private objects with driver-specific backing storage(contiguous DMA memory, special reserved blocks) 此时 filp 为 NULL.

`vma_node`:

```c
struct drm_gem_object_funcs {
	void (*free)(struct drm_gem_object *obj);
	int (*open)(struct drm_gem_object *obj, struct drm_file *file);
	void (*close)(struct drm_gem_object *obj, struct drm_file *file);
	void (*print_info)(struct drm_printer *p, unsigned int indent,
			   const struct drm_gem_object *obj);
	struct dma_buf *(*export)(struct drm_gem_object *obj, int flags);
	int (*pin)(struct drm_gem_object *obj);
	void (*unpin)(struct drm_gem_object *obj);
	struct sg_table *(*get_sg_table)(struct drm_gem_object *obj);
	int (*vmap)(struct drm_gem_object *obj, struct iosys_map *map);
	void (*vunmap)(struct drm_gem_object *obj, struct iosys_map *map);
	int (*mmap)(struct drm_gem_object *obj, struct vm_area_struct *vma);
	int (*evict)(struct drm_gem_object *obj);
	enum drm_gem_object_status (*status)(struct drm_gem_object *obj);
	size_t (*rss)(struct drm_gem_object *obj);
	const struct vm_operations_struct *vm_ops;
};
```

`free`: mandatory, 销毁 GEM object.

`open`: optional, 在 GEM handle creation 时调用.

`close`: optional, 在 GEM handle realse 时调用.

`print_info`: optional, 如果 subclass 了 drm_gem_object 可以用来打印其他成员.

`export`: optional, 用来 export backing buffer 为 dma-buf, 如果没实现则会调用 drm_gem_prime_export().

`pin`: optional, 固定 memory 中的 backing buffer.

`unpin`: optional, 与 pin 相反.

`get_sg_table`:

`vmap`: optional, 映射 buffer 到 kernel 虚拟地址.

`vunmap`: optional, 与 vmap 相反.

`mmap`: optional, 映射 buufer 到 user 虚拟地址.

`evict`: optional, 将 gem object 从 memory 中逐出, 没看到哪边实现了这个回调.

`status`: 返回 enum drm_gem_object_status.

`rss`: return resident size of GEM object memory. 只有 panfrost driver 实现了.

`vm_ops`: mmap 用到的 virtual memory operations.

# Userspace API

和 GEM 相关的 ioctl 有:

```c
DRM_IOCTL_DEF(DRM_IOCTL_GEM_CLOSE, drm_gem_close_ioctl, DRM_RENDER_ALLOW),
DRM_IOCTL_DEF(DRM_IOCTL_GEM_FLINK, drm_gem_flink_ioctl, DRM_AUTH),
DRM_IOCTL_DEF(DRM_IOCTL_GEM_OPEN, drm_gem_open_ioctl, DRM_AUTH),
DRM_IOCTL_DEF(DRM_IOCTL_MODE_CREATE_DUMB, drm_mode_create_dumb_ioctl, 0),
DRM_IOCTL_DEF(DRM_IOCTL_MODE_MAP_DUMB, drm_mode_mmap_dumb_ioctl, 0),
DRM_IOCTL_DEF(DRM_IOCTL_MODE_DESTROY_DUMB, drm_mode_destroy_dumb_ioctl, 0),
```

**DRM_IOCTL_GEM_FLINK**

传入 gem object handle, 给 gem object 分配 name.

**DRM_IOCTL_GEM_OPEN**

通过 name 来获取 gem object handle.

# Kernel layer

## gem_dma_helper

gem_dma_helper 是给设备提供连续内存的方案.

对于第一类设备, 通过 IOMMU 来访问 memory bus 的, gem_dma_helper 通过传统的 page-based allocator 分配 buffer, 物理地址可能是不连续的. 但是在 IOMMU IOVA space 看起来是连续的, 所以可以使用.

对于其他设备, gem_dma_helper 依赖于 CMA , 提供物理地址连续的内存 buffer. 看起来嵌入式设备一般对应这种.

## gem_shmem_helper

利用 anonymous pageable memory 分配内存 buffer.

## gem_vram_helper

Used for framebuffer devices with dedicated memory. 用于自带 vram 的设备.
