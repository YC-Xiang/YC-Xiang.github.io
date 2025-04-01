# DRM Internals

## Driver Initialization

driver 首先需要静态初始化一个 `struct drm_driver` 结构体，然后调用 `drm_dev_alloc()` 来
创建一个 `struct drm_device` 结构体，初始化必要的 fields 后，最后调用 `drm_dev_register()` 注册 drm_device.

## Driver Information

在 `struct drm_driver` 结构体中的.major, .minor, .patchlevel, .name, .desc .date 字段用于描述 driver 的基本信息。

通过 `DRM_IOCTL_VERSION` 和 `DRM_IOCTL_SET_VERSION` 两个 ioctl 可以获取和设置 driver 基本信息。

## Module Initialization

在 `drm_module.h` 中提供了一些封装的 api 来注册 module platform/pci driver:

```c++
drm_module_pci_driver(__pci_drv);
drm_module_platform_driver(__platform_drv);
```

## Device Instance and Driver Handling

这里有一个 driver 的 template, 具体查看官方文档。

如果需要支持热插拔 drm 设备，比如 USB, DT overlay unload, 需要使用 `drm_dev_unplug()` 代替 `drm_dev_unregister()`.

为了防止热插拔设备被拔出 (unplugged) 后，drm 设备资源被访问，需要在访问 drm 设备资源时，使用 `drm_dev_enter()` 和 `drm_dev_exit()` 来保护。
如果 drm device 状态是 unplugged, 则 drm_dev_enter() 会返回 false.

## Driver load

利用 component library 来实现 a logical device consists of a pile of independent hardware blocks 的情况。

drm device driver 在 .bind 函数中按这个流程初始化：devm_drm_dev_alloc() -> component_bind_all() -> drm_dev_register().

传入 component_bind_all() 的 void *data, 需要是 struct drm_device, 不要传 driver 私有数据。

## Open/Close, File Operations and IOCTLs

file operations 必须保存到 drm_driver.ops 中。

可以利用 `DEFINE_DRM_GEM_FOPS()`, `DEFINE_DRM_GEM_DMA_FOPS()` 这两个宏。

## DRM print

`drm_print.h`

```c++
void log_some_info(struct drm_printer *p)
{
        drm_printf(p, "foo=%d\n", foo);
        drm_printf(p, "bar=%d\n", bar);
}

// debugfs 用法
#ifdef CONFIG_DEBUG_FS
void debugfs_show(struct seq_file *f)
{
        struct drm_printer p = drm_seq_file_printer(f);
        log_some_info(&p);
}
#endif

/// driver 内部用法
void some_other_function(...)
{
        struct drm_printer p = drm_info_printer(drm->dev);
        log_some_info(&p);
}
```

修改 drm.debug 的值可以调整打印等级。

同时也可以在 sysfs 中修改 `echo 0xf > /sys/module/drm/parameters/debug`

## Data Structures and api

### drm_device

```c++
struct drm_device {
	int if_version; // drm api 版本
	struct kref ref; // 引用计数
	struct device *dev; // 对应的 bus 上的 device
	struct {
		struct list_head resources;
		void *final_kfree;
		spinlock_t lock;
	} managed; // 通过 ref 控制的 resource 链表
	const struct drm_driver *driver; // 对应的 drm_driver
	void *dev_private; // 已废弃，推荐的做法是把 drm_device 嵌入具体 driver 更大的 device 结构体中。
	struct drm_minor *primary; // /dev/dri/card0 节点，显示控制
	struct drm_minor *render; // /dev/dri/renderD128 节点，用于 GPU 图形渲染
	struct drm_minor *accel; // /dev/dri/accel0 节点，用于 GPU 计算加速，比如 CUDA, OpenCL 等，常用于 machine learning, deep learning. 这部分 driver 在 drivers/accel/目录下.
	bool registered; // 表示是否注册
	struct drm_master *master; // drm_device 的 master
	u32 driver_features; // driver 可以通过清除某些位来限制功能，drm device 这边的 driver_features 默认应该是全开的
	bool unplugged; // 热插拔设备是否被拔出的表示
	struct inode *anon_inode; /// 匿名 inode, 用于 mmap 匿名映射？
	char *unique; // drm device 的名称，在 drm_dev_init() 中设置为 device name
	struct mutex struct_mutex; // 用不上了，只有 intel i915 driver 会使用
	struct mutex master_mutex; // 给 drm_minor.master 和 drm_file.is_master 使用
	atomic_t open_count; // drm file 被打开的次数
	struct mutex filelist_mutex; // 保护下面的 filelist
	struct list_head filelist; // userspace clients 打开的 drm file 链表
	struct list_head filelist_internal; // kernel clients 打开的 drm file 链表
	struct mutex clientlist_mutex; // 保护下面的 clientlist
	struct list_head clientlist;// kernel clients 链表
	bool vblank_disable_immediate; // TODO: 理清这个的作用。当 refcount 到 0, vblank interrupt 会被立即禁止
	struct drm_vblank_crtc *vblank; // vblank 结构体。每个 crtc 各一个。
	spinlock_t vblank_time_lock;
	spinlock_t vbl_lock;
	// 硬件 vblank 计数器寄存器最大值，如果设为 0, drm core 会自动推测时间段内的 vblanks 数量。如果不为 0, 还需要实现 drm_ctrc_funcs.get_vblank_counter()
	u32 max_vblank_count;
	struct list_head vblank_event_list; // vblank event 链表
	spinlock_t event_lock; // 保护 vblank_event_list
	unsigned int num_crtcs; // 当前 device 有多少个 crtc, 在 drm_vblank_init() 中设置，可以传入 mode_config-> num_crtc
	struct drm_mode_config mode_config; // 当前的 mode config 配置模式。
	struct mutex object_name_lock; // GEM information
	struct idr object_name_idr; // GEM information
	struct drm_vma_offset_manager *vma_offset_manager; // GEM information
	struct drm_vram_mm *vram_mm; // vram memory manager
	enum switch_power_state switch_power_state; // switcheroo driver 才用得到，处理双显卡系统
	struct drm_fb_helper *fb_helper; // fbdev emulation, 通过 drm_fb_helper_init() 和 drm_fb_helper_fini() 初始化和销毁。
	struct dentry *debugfs_root; // debugfs 的根节点
};
```

### drm_driver

```c++
struct drm_driver {
	int (*open) (struct drm_device *, struct drm_file *);
	void (*postclose) (struct drm_device *, struct drm_file *);
	void (*lastclose) (struct drm_device *);
	void (*debugfs_init)(struct drm_minor *minor);
	struct drm_gem_object *(*gem_create_object)(struct drm_device *dev,
						    size_t size);
	struct drm_gem_object * (*gem_prime_import)(struct drm_device *dev,
				struct dma_buf *dma_buf);
	struct drm_gem_object *(*gem_prime_import_sg_table)(
				struct drm_device *dev,
				struct dma_buf_attachment *attach,
				struct sg_table *sgt);
	int (*dumb_create)(struct drm_file *file_priv,
			   struct drm_device *dev,
			   struct drm_mode_create_dumb *args);
	int (*dumb_map_offset)(struct drm_file *file_priv,
			       struct drm_device *dev, uint32_t handle,
			       uint64_t *offset);
	void (*show_fdinfo)(struct drm_printer *p, struct drm_file *f);
	int major;
	int minor;
	int patchlevel;
	char *name;
	char *desc;
	char *date;
	u32 driver_features;
	const struct drm_ioctl_desc *ioctls;
	int num_ioctls;
	const struct file_operations *fops;
};
```

open: 当 drm_file 被打开后调用的回调函数，可以使用 DEFINE_DRM_GEM_DMA_FOPS 中的 drm_open.  
可以设置一些 driver-private 的 data structure, 比如 buffer allocators, execution contexts.  
注意，因为 kms 只能由一个 master 的 drm_file 设置，所以不可以在 open 回调中设置 kms 相关的 resources.  
postclose: 当 drm_file 被关闭后调用的回调函数。和 open 相反。  
lastclose: 当最后一个 client 关闭 drm_file 后调用的回调函数。  
debugfs_init: 创建 driver-specific debugfs 文件。  
gem_create_object: 创建 GEM object 的回调，在 CMA 和 SHMEM GEM helper 中使用。  
gem_prime_import: GEM driver 的 prime import 回调。不设置的话默认为 drm_gem_prime_import().  
gem_prime_import_sg_table: GEM import sg table 回调。  
dumb_create: user space 通过 ioctl 创建 dumb buffer 的回调函数。  
dumb_map_offset: 创建 mmap 需要的 offset.默认实现为 drm_gem_create_mmap_offset()  
show_fdinfo: 打印 drm_file 的 fdinfo.  
major, minor, patchlevel: 用于描述 driver 的版本。  
name, desc, date: 用于描述 driver 的名称，描述，和日期。  
driver_features: 用于描述 driver 的功能，enum drm_driver_feature.  
ioctls: 用于描述 driver 的 ioctl 函数。  
num_ioctls: ioctl 函数的数量。  
fops: 用于描述 driver 的 file operations.

## APIs

`drm_drv.h`, `drm_drv.c`

```c++
devm_drm_dev_alloc(parent, driver, type, member);
void drm_dev_unplug(struct drm_device *dev);
bool drm_dev_is_unplugged(struct drm_device *dev);
bool drm_core_check_all_features(const struct drm_device *dev, u32 features);
bool drm_core_check_feature(const struct drm_device *dev, enum drm_driver_feature feature);
bool drm_drv_uses_atomic_modeset(struct drm_device *dev);
bool drm_dev_enter(struct drm_device *dev, int *idx);
void drm_dev_exit(int idx);
void drm_dev_get(struct drm_device *dev);
void drm_dev_put(struct drm_device *dev);
int drm_dev_register(struct drm_device *dev, unsigned long flags);
void drm_dev_unregister(struct drm_device *dev);
```

devm_drm_dev_alloc() 和 drm_dev_alloc() 都是用于分配 drm_device 并初始化，推荐使用前者，后者已废弃。

drm_dev_unplug 设置 drm_device 的 unplugged 为 true, 并调用 drm_dev_unregister() 注销设备。
drm_dev_is_unplugged 返回 drm_device 的 unplugged 状态。

drm_core_check_all_features() 可以一次检查 drm device 多个支持的 feature.  
drm_core_check_feature() 检查 drm device 是否支持某个 feature.  
drm_drv_uses_atomic_modeset() 检查 drm device 是否自持 atomic modeset feature.

drm_dev_enter(), drm_dev_exit(): 保护临界区，unplugged 状态无法访问。

drm_dev_get(): 增加 drm device 引用计数。
drm_dev_put(): 减少 drm device 引用计数，到 0 调用 drm_dev_release.

drm_dev_register(): 注册 drm device.  
drm_dev_unregister(): 注销 drm device.
