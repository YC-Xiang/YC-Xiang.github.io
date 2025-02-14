# DRM Internals

## Driver Initialization

driver 首先需要静态初始化一个`struct drm_driver`结构体, 然后调用`drm_dev_alloc()`来
创建一个`struct drm_device`结构体, 初始化必要的 fields 后, 最后调用`drm_dev_register()` 注册 drm_device.

## Driver Information

```c
struct drm_driver {
	int (*load) (struct drm_device *, unsigned long flags);
	int (*open) (struct drm_device *, struct drm_file *);
	void (*postclose) (struct drm_device *, struct drm_file *);
	void (*lastclose) (struct drm_device *);
	void (*unload) (struct drm_device *);
	void (*release) (struct drm_device *);
	void (*master_set)(struct drm_device *dev, struct drm_file *file_priv,
			   bool from_open);
	void (*master_drop)(struct drm_device *dev, struct drm_file *file_priv);
	void (*debugfs_init)(struct drm_minor *minor);
	struct drm_gem_object *(*gem_create_object)(struct drm_device *dev,
						    size_t size);
	int (*prime_handle_to_fd)(struct drm_device *dev, struct drm_file *file_priv,
				uint32_t handle, uint32_t flags, int *prime_fd);
	int (*prime_fd_to_handle)(struct drm_device *dev, struct drm_file *file_priv,
				int prime_fd, uint32_t *handle);
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

load: 已废弃.  
unload: 已废弃.  
release: 已废弃.  
open: 当 drm_file 被打开后调用的回调函数, 可以使用 DEFINE_DRM_GEM_DMA_FOPS 中的 drm_open.  
可以设置一些 driver-private 的 data structure, 比如 buffer allocators, execution contexts.  
注意, 因为 kms 只能由一个 master 的 drm_file 设置, 所以不可以在 open 回调中设置 kms 相关的 resources.  
postclose: 当 drm_file 被关闭后调用的回调函数. 和 open 相反.  
lastclose: 当最后一个 client 关闭 drm_file 后调用的回调函数.  
master_set: 只有 vmwgfx 会使用.  
master_drop: 只有 vmwgfx 会使用.  
debugfs_init: 创建 driver-specific debugfs 文件.  
gem_create_object: 创建 GEM object 的回调, 在 CMA 和 SHMEM GEM helper 中使用.  
prime_handle_to_fd: 只有 vmwgfx 会使用.  
prime_fd_to_handle: 只有 vmwgfx 会使用.  
gem_prime_import: GEM driver 的 prime import 回调. 不设置的话默认为 drm_gem_prime_import().  
gem_prime_import_sg_table: GEM import sg table 回调.  
dumb_create: user space 通过 ioctl 创建 dumb buffer 的回调函数.  
dumb_map_offset: 创建 mmap 需要的 offset.默认实现为 drm_gem_create_mmap_offset()  
show_fdinfo: 打印 drm_file 的 fdinfo.  
major, minor, patchlevel: 用于描述 driver 的版本.  
name, desc, date: 用于描述 driver 的名称, 描述, 和日期.  
driver_features: 用于描述 driver 的功能.  
ioctls: 用于描述 driver 的 ioctl 函数.  
num_ioctls: ioctl 函数的数量.  
fops: 用于描述 driver 的 file operations.

在`struct drm_driver`结构体中的.major, .minor, .patchlevel, .name, .desc .date 字段用于描述 driver 的基本信息.

通过`DRM_IOCTL_VERSION`和`DRM_IOCTL_SET_VERSION` 两个 ioctl 可以获取和设置 driver 基本信息.

## Device Instance and Driver Handling

### Display driver example

```c
struct driver_device {
	struct drm_device drm;
	void *userspace_facing;
	struct clk *pclk;
};

static const struct drm_driver driver_drm_driver = {
	[...]
};

static int driver_probe(struct platform_device *pdev)
{
	struct driver_device *priv;
	struct drm_device *drm;
	int ret;

	priv = devm_drm_dev_alloc(&pdev->dev, &driver_drm_driver,
				  struct driver_device, drm);
	if (IS_ERR(priv))
		return PTR_ERR(priv);
	drm = &priv->drm;

	ret = drmm_mode_config_init(drm);
	if (ret)
		return ret;

	priv->userspace_facing = drmm_kzalloc(..., GFP_KERNEL);
	if (!priv->userspace_facing)
		return -ENOMEM;

	priv->pclk = devm_clk_get(dev, "PCLK");
	if (IS_ERR(priv->pclk))
		return PTR_ERR(priv->pclk);

	// Further setup, display pipeline etc

	platform_set_drvdata(pdev, drm);

	drm_mode_config_reset(drm);

	ret = drm_dev_register(drm);
	if (ret)
		return ret;

	drm_fbdev_{...}_setup(drm, 32);

	return 0;
}

// This function is called before the devm_ resources are released
static int driver_remove(struct platform_device *pdev)
{
	struct drm_device *drm = platform_get_drvdata(pdev);

	drm_dev_unregister(drm);
	drm_atomic_helper_shutdown(drm)

	return 0;
}

// This function is called on kernel restart and shutdown
static void driver_shutdown(struct platform_device *pdev)
{
	drm_atomic_helper_shutdown(platform_get_drvdata(pdev));
}

static int __maybe_unused driver_pm_suspend(struct device *dev)
{
	return drm_mode_config_helper_suspend(dev_get_drvdata(dev));
}

static int __maybe_unused driver_pm_resume(struct device *dev)
{
	drm_mode_config_helper_resume(dev_get_drvdata(dev));

	return 0;
}

static const struct dev_pm_ops driver_pm_ops = {
	SET_SYSTEM_SLEEP_PM_OPS(driver_pm_suspend, driver_pm_resume)
};

static struct platform_driver driver_driver = {
	.driver = {
		[...]
		.pm = &driver_pm_ops,
	},
	.probe = driver_probe,
	.remove = driver_remove,
	.shutdown = driver_shutdown,
};
module_platform_driver(driver_driver);
```

如果需要支持热插拔 drm 设备, 比如 USB, DT overlay unload, 需要使用`drm_dev_unplug()` 代替`drm_dev_unregister()`.

为了防止热插拔设备被拔出(unplugged)后, drm 设备资源被访问, 需要在访问 drm 设备资源时, 使用`drm_dev_enter()` 和`drm_dev_exit()` 来保护.
如果 drm device 状态是 unplugged, 则 drm_dev_enter()会返回 false.

## Data Structures

### drm_device

```c
struct drm_device {
	int if_version; // drm api 版本
	struct kref ref; // 引用计数
	struct device *dev; // 对应的bus上的device
	struct {
		struct list_head resources;
		void *final_kfree;
		spinlock_t lock;
	} managed; // 通过ref控制的resource链表
	const struct drm_driver *driver; // 对应的drm_driver
	void *dev_private; // 已废弃, 推荐的做法是把drm_device嵌入具体driver更大的device结构体中.
	struct drm_minor *primary; // /dev/dri/card0 节点, 显示控制
	struct drm_minor *render; // /dev/dri/renderD128 节点, 用于GPU图形渲染
	struct drm_minor *accel; // /dev/dri/accel0 节点, 用于GPU计算加速, 比如CUDA, OpenCL等, 常用于machine learning, deep learning. 这部分driver在drivers/accel/目录下.
	bool registered; // 表示是否注册
	struct drm_master *master; // drm_device的master
	u32 driver_features; // driver可以通过清除某些位来限制功能, drm device这边的driver_features默认应该是全开的
	bool unplugged; // 热插拔设备是否被拔出的表示
	struct inode *anon_inode; /// 匿名inode, 用于mmap匿名映射?
	char *unique; // drm device的名称, 在drm_dev_init()中设置为device name
	struct mutex struct_mutex; // 用不上了, 只有intel i915 driver会使用
	struct mutex master_mutex; // 给drm_minor.master和drm_file.is_master使用
	atomic_t open_count; // drm file被打开的次数
	struct mutex filelist_mutex; // 保护下面的filelist
	struct list_head filelist; // userspace clients打开的drm file链表
	struct list_head filelist_internal; // kernel clients打开的drm file链表
	struct mutex clientlist_mutex; // 保护下面的clientlist
	struct list_head clientlist;// kernel clients链表
	bool vblank_disable_immediate; // TODO: 理清这个的作用. 当refcount到0, vblank interrupt会被立即禁止
	struct drm_vblank_crtc *vblank; // vblank 结构体. 每个crtc各一个.
	spinlock_t vblank_time_lock;
	spinlock_t vbl_lock;
	// 硬件vblank计数器寄存器最大值, 如果设为0, drm core会自动推测时间段内的vblanks数量.如果不为0, 还需要实现drm_ctrc_funcs.get_vblank_counter()
	u32 max_vblank_count;
	struct list_head vblank_event_list; // vblank event链表
	spinlock_t event_lock; // 保护vblank_event_list
	unsigned int num_crtcs; // 当前device有多少个crtc, 在drm_vblank_init()中设置, 可以传入mode_config->num_crtc
	struct drm_mode_config mode_config; // 当前的 mode config 配置模式.
	struct mutex object_name_lock; // GEM information
	struct idr object_name_idr; // GEM information
	struct drm_vma_offset_manager *vma_offset_manager; // GEM information
	struct drm_vram_mm *vram_mm; // vram memory manager
	enum switch_power_state switch_power_state; // switcheroo driver才用得到, 处理双显卡系统
	struct drm_fb_helper *fb_helper; // fbdev emulation, 通过drm_fb_helper_init()和drm_fb_helper_fini()初始化和销毁.
	struct dentry *debugfs_root; // debugfs的根节点
};
```

### drm_driver

## APIs

`drm_drv.h`, `drm_drv.c`

```c
devm_drm_dev_alloc(parent, driver, type, member);
struct drm_device *drm_dev_alloc(const struct drm_driver *driver, struct device *parent);
void drm_dev_unplug(struct drm_device *dev);
bool drm_dev_is_unplugged(struct drm_device *dev);
bool drm_core_check_all_features(const struct drm_device *dev, u32 features);
bool drm_core_check_feature(const struct drm_device *dev, enum drm_driver_feature feature);
bool drm_drv_uses_atomic_modeset(struct drm_device *dev);
void drm_put_dev(struct drm_device *dev);
bool drm_dev_enter(struct drm_device *dev, int *idx);
void drm_dev_exit(int idx);
void drm_dev_get(struct drm_device *dev);
void drm_dev_put(struct drm_device *dev);
int drm_dev_register(struct drm_device *dev, unsigned long flags);
void drm_dev_unregister(struct drm_device *dev);
```

devm_drm_dev_alloc() 和 drm_dev_alloc() 都是用于分配 drm_device 并初始化, 推荐使用前者, 后者已废弃.

drm_dev_unplug 设置 drm_device 的 unplugged 为 true, 并调用 drm_dev_unregister()注销设备.  
drm_dev_is_unplugged 返回 drm_device 的 unplugged 状态.

drm_core_check_all_features() 可以一次检查 drm device 多个支持的 feature.  
drm_core_check_feature() 检查 drm device 是否支持某个 feature.  
drm_drv_uses_atomic_modeset() 检查 drm device 是否自持 atomic modeset feature.

drm_put_dev(): 已废弃.

drm_dev_enter(), drm_dev_exit(): 保护临界区, unplugged 状态无法访问.

drm_dev_get(): 增加 drm device 引用计数.  
drm_dev_put(): 减少 drm device 引用计数, 到 0 调用 drm_dev_release.

drm_dev_register:
