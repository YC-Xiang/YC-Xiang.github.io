# 概述

bus 模块的功能包括:

- bus 的注册和注销
- 本 bus 下有 device 或者 device_driver 注册到内核时的处理
- 本 bus 下有 device 或者 device_driver 从内核注销时的处理
- device_drivers 的 probe 处理
- 管理 bus 下的所有 device 和 device_driver

# 数据结构

```c
struct bus_type {
	const char		*name;
	const char		*dev_name;
	const struct attribute_group **bus_groups;
	const struct attribute_group **dev_groups;
	const struct attribute_group **drv_groups;

	int (*match)(struct device *dev, struct device_driver *drv);
	int (*uevent)(const struct device *dev, struct kobj_uevent_env *env);
	int (*probe)(struct device *dev);
	void (*sync_state)(struct device *dev);
	void (*remove)(struct device *dev);
	void (*shutdown)(struct device *dev);

	int (*online)(struct device *dev);
	int (*offline)(struct device *dev);

	int (*suspend)(struct device *dev, pm_message_t state);
	int (*resume)(struct device *dev);

	int (*num_vf)(struct device *dev);

	int (*dma_configure)(struct device *dev);
	void (*dma_cleanup)(struct device *dev);

	const struct dev_pm_ops *pm;

	bool need_parent_lock;
};
```

name : bus 的名称.

dev_name:

</br>

```c
struct subsys_private {
	struct kset subsys;
	struct kset *devices_kset;
	struct list_head interfaces;
	struct mutex mutex;

	struct kset *drivers_kset;
	struct klist klist_devices;
	struct klist klist_drivers;
	struct blocking_notifier_head bus_notifier;
	unsigned int drivers_autoprobe:1;
	const struct bus_type *bus;
	struct device *dev_root;

	struct kset glue_dirs;
	const struct class *class;

	struct lock_class_key lock_key;
};
```

和 struct driver_private 类似, bus 的私有数据.

subsys, devices_kset, drivers_kset: 三个 kset, 分别代表 bus, device, driver.

klist_devices, klist_drivers: 两个链表, 保存了本 bus 下所有的 device 和 device_driver 的指针.

bus: 上层 bus 的指针.

class: 属于的 class 指针.

# Bus 注册

```c
int bus_register(const struct bus_type *bus);
```

初始化 struct subsys_private, 并注册到内核.

# Device 和 Device Driver 的添加

```c
int bus_add_device(struct device *dev);
int bus_add_driver(struct device_driver *drv);
```

将 device 和 driver 挂入 bus 的 klist_devices 和 klist_drivers 链表中.

在 bus_add_driver 中还会调用 driver_attach, 遍历 bus 链表上的 devices, 调用 bus->match 函数, 检查 device 和 driver 是否匹配, 如果匹配的话调用 bus->probe 函数.

![](https://xyc-1316422823.cos.ap-shanghai.myqcloud.com/20250110140334.png)

# Driver 的 probe
