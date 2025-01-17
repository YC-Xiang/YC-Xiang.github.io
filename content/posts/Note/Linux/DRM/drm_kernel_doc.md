# DRM Internals

## Driver Initialization

driver 首先需要静态初始化一个`struct drm_driver`结构体, 然后调用`drm_dev_alloc()`,
创建一个`struct drm_device`结构体, 最后调用`drm_dev_register()` 注册 drm_device.

## Driver Information

在`struct drm_driver`结构体中的.major, .minor, .patchlevel, .name, .desc .date 字段用于描述 driver 的基本信息.

通过`DRM_IOCTL_VERSION`和`DRM_IOCTL_SET_VERSION` 两个 ioctl 可以获取和设置 driver 基本信息.

```c
struct drm_device {
	int if_version; // drm api 版本
	struct kref ref; // 引用计数
	struct device *dev; // 对应的bus上的device
	struct {
		/** @managed.resources: managed resources list */
		struct list_head resources;
		/** @managed.final_kfree: pointer for final kfree() call */
		void *final_kfree;
		/** @managed.lock: protects @managed.resources */
		spinlock_t lock;
	} managed;
	const struct drm_driver *driver; // 对应的drm_driver
	void *dev_private; // 已废弃, 设置为NULL

	struct drm_minor *primary;
	struct drm_minor *render;
	struct drm_minor *accel;

	bool registered; // 表示是否注册

	struct drm_master *master; // drm_device的master

	/**
	 * @driver_features: per-device driver features
	 *
	 * Drivers can clear specific flags here to disallow
	 * certain features on a per-device basis while still
	 * sharing a single &struct drm_driver instance across
	 * all devices.
	 */
	u32 driver_features;

	/**
	 * @unplugged:
	 *
	 * Flag to tell if the device has been unplugged.
	 * See drm_dev_enter() and drm_dev_is_unplugged().
	 */
	bool unplugged;

	/** @anon_inode: inode for private address-space */
	struct inode *anon_inode;

	char *unique; // drm device的名称, 在drm_dev_init()中设置为device name

	/**
	 * @struct_mutex:
	 *
	 * Lock for others (not &drm_minor.master and &drm_file.is_master)
	 *
	 * TODO: This lock used to be the BKL of the DRM subsystem. Move the
	 *       lock into i915, which is the only remaining user.
	 */
	struct mutex struct_mutex;

	/**
	 * @master_mutex:
	 *
	 * Lock for &drm_minor.master and &drm_file.is_master
	 */
	struct mutex master_mutex;

	/**
	 * @open_count:
	 *
	 * Usage counter for outstanding files open,
	 * protected by drm_global_mutex
	 */
	atomic_t open_count;

	/** @filelist_mutex: Protects @filelist. */
	struct mutex filelist_mutex;
	/**
	 * @filelist:
	 *
	 * List of userspace clients, linked through &drm_file.lhead.
	 */
	struct list_head filelist;

	/**
	 * @filelist_internal:
	 *
	 * List of open DRM files for in-kernel clients.
	 * Protected by &filelist_mutex.
	 */
	struct list_head filelist_internal;

	/**
	 * @clientlist_mutex:
	 *
	 * Protects &clientlist access.
	 */
	struct mutex clientlist_mutex;

	/**
	 * @clientlist:
	 *
	 * List of in-kernel clients. Protected by &clientlist_mutex.
	 */
	struct list_head clientlist;

	/**
	 * @vblank_disable_immediate:
	 *
	 * If true, vblank interrupt will be disabled immediately when the
	 * refcount drops to zero, as opposed to via the vblank disable
	 * timer.
	 *
	 * This can be set to true it the hardware has a working vblank counter
	 * with high-precision timestamping (otherwise there are races) and the
	 * driver uses drm_crtc_vblank_on() and drm_crtc_vblank_off()
	 * appropriately. See also @max_vblank_count and
	 * &drm_crtc_funcs.get_vblank_counter.
	 */
	bool vblank_disable_immediate;

	/**
	 * @vblank:
	 *
	 * Array of vblank tracking structures, one per &struct drm_crtc. For
	 * historical reasons (vblank support predates kernel modesetting) this
	 * is free-standing and not part of &struct drm_crtc itself. It must be
	 * initialized explicitly by calling drm_vblank_init().
	 */
	struct drm_vblank_crtc *vblank;

	/**
	 * @vblank_time_lock:
	 *
	 *  Protects vblank count and time updates during vblank enable/disable
	 */
	spinlock_t vblank_time_lock;
	/**
	 * @vbl_lock: Top-level vblank references lock, wraps the low-level
	 * @vblank_time_lock.
	 */
	spinlock_t vbl_lock;

	/**
	 * @max_vblank_count:
	 *
	 * Maximum value of the vblank registers. This value +1 will result in a
	 * wrap-around of the vblank register. It is used by the vblank core to
	 * handle wrap-arounds.
	 *
	 * If set to zero the vblank core will try to guess the elapsed vblanks
	 * between times when the vblank interrupt is disabled through
	 * high-precision timestamps. That approach is suffering from small
	 * races and imprecision over longer time periods, hence exposing a
	 * hardware vblank counter is always recommended.
	 *
	 * This is the statically configured device wide maximum. The driver
	 * can instead choose to use a runtime configurable per-crtc value
	 * &drm_vblank_crtc.max_vblank_count, in which case @max_vblank_count
	 * must be left at zero. See drm_crtc_set_max_vblank_count() on how
	 * to use the per-crtc value.
	 *
	 * If non-zero, &drm_crtc_funcs.get_vblank_counter must be set.
	 */
	u32 max_vblank_count;

	/** @vblank_event_list: List of vblank events */
	struct list_head vblank_event_list;

	/**
	 * @event_lock:
	 *
	 * Protects @vblank_event_list and event delivery in
	 * general. See drm_send_event() and drm_send_event_locked().
	 */
	spinlock_t event_lock;

	/** @num_crtcs: Number of CRTCs on this device */
	unsigned int num_crtcs;

	/** @mode_config: Current mode config */
	struct drm_mode_config mode_config;

	/** @object_name_lock: GEM information */
	struct mutex object_name_lock;

	/** @object_name_idr: GEM information */
	struct idr object_name_idr;

	/** @vma_offset_manager: GEM information */
	struct drm_vma_offset_manager *vma_offset_manager;

	/** @vram_mm: VRAM MM memory manager */
	struct drm_vram_mm *vram_mm;

	/// switcheroo driver才用得到, 处理双显卡系统
	enum switch_power_state switch_power_state;

	/**
	 * @fb_helper:
	 *
	 * Pointer to the fbdev emulation structure.
	 * Set by drm_fb_helper_init() and cleared by drm_fb_helper_fini().
	 */
	struct drm_fb_helper *fb_helper;

	/**
	 * @debugfs_root:
	 *
	 * Root directory for debugfs files.
	 */
	struct dentry *debugfs_root;
};
```
