# 数据结构

Platform 设备是可以通过 CPU bus 直接寻址的设备, 内核在设备模型 bus, device 和 driver 的基础上,
进行了进一步封装, 抽象了 platform_bus, platform_device, platform_driver.

platform 相关的实现在 `include/linux/platform_device.h`, `drivers/base/platform.c`中.

```c
struct platform_device {
	const char	*name;
	int		id;
	bool		id_auto;
	struct device	dev;
	u64		platform_dma_mask;
	struct device_dma_parameters dma_parms;
	u32		num_resources;
	struct resource	*resource;

	const struct platform_device_id	*id_entry;
};
```

name: platform device 的名称.

dev: 指向底层的 device.

id: 可以手动设置, 也可设置为宏 PLATFORM_DEVID_NONE 表示没有 id, pdev->dev->name 直接设置为 pdev->name. 设置为宏 PLATFORM_DEVID_AUTO 表示自动生成 id.  
id_auto: 当 id 设置为 PLATFORM_DEVID_AUTO, id_auto 被置为 true, 在注册 platform_device 时可以自动生成 id.

platform_dma_mask: 设置 dma 掩码, 通过 setup_pdev_dma_masks() api.  
dma_parms:

num_resources: 资源数量, linux 中资源包括 I/O, Memory, Register, IRQ, DMA 等.  
resource: 资源数组.

```c
struct platform_driver {
	int (*probe)(struct platform_device *);
	int (*remove)(struct platform_device *);
	void (*remove_new)(struct platform_device *);
	void (*shutdown)(struct platform_device *);
	int (*suspend)(struct platform_device *, pm_message_t state);
	int (*resume)(struct platform_device *);
	struct device_driver driver;
	const struct platform_device_id *id_table;
};
```

# API

platform device API:

```c
int platform_device_register(struct platform_device *);
void platform_device_unregister(struct platform_device *);

struct resource *platform_get_resource(struct platform_device *,
					      unsigned int, unsigned int);
extern struct resource *platform_get_mem_or_io(struct platform_device *,
					       unsigned int);
extern int platform_get_irq(struct platform_device *, unsigned int);

int platform_add_devices(struct platform_device **, int);
struct platform_device *platform_device_register_full(
		const struct platform_device_info *pdevinfo);

extern int platform_device_add(struct platform_device *pdev);
extern void platform_device_del(struct platform_device *pdev);
extern void platform_device_put(struct platform_device *pdev);


```

注册和注销相关:

platform_device_register/platform_device_unregister: 注册/注销 platform_device.  
platform_add_devices: 一下子注册多个 platform_device.  
platform_device_register_full: 动态分配注册 platform_device.
platform_device_register_resndata: 在 platform_device_register_full 上再抽象了一层.

获取资源相关:

platform_get_resource: 获取 platform_device 的资源.  
platform_get_mem_or_io: 获取 platform_device 的内存或 I/O 资源.  
platform_get_irq: 获取 platform_device 的 IRQ.

在 device_add, device_del, device_put 的基础上, 抽象了 platform_device_add, platform_device_del, platform_device_put.

platform_device_add: 添加 platform_device.
platform_device_del: 删除 platform_device.
platform_device_put: 释放 platform_device.

platform driver API:

```c
int platform_driver_register(struct platform_driver *);
void platform_driver_unregister(struct platform_driver *);

#define platform_driver_probe(drv, probe) \
	__platform_driver_probe(drv, probe, THIS_MODULE)
int __platform_driver_probe(struct platform_driver *driver,
		int (*probe)(struct platform_device *), struct module *module);

#define platform_create_bundle(driver, probe, res, n_res, data, size) \
struct platform_device *__platform_create_bundle(
	struct platform_driver *driver, int (*probe)(struct platform_device *),
	struct resource *res, unsigned int n_res,
	const void *data, size_t size, struct module *module);

static inline void *platform_get_drvdata(const struct platform_device *pdev);
static inline void platform_set_drvdata(struct platform_device *pdev, void *data);
```

注册和注销相关:

platform_driver_register/platform_driver_unregister: 注册/注销 platform_driver.  
platform_driver_probe: 注册 platform_driver, 并调用传入的 probe 函数.
platform_create_bundle: 注册所有, 包括 platform_device 和 platform_driver.

platform_get_drvdata/platform_set_drvdata: 获取/设置 struct platform_device->device 的私有 device_data.

# 初始化及内部解析

Platform 模块的初始化是由 drivers/base/platform.c 中 platform_bus_init 接口完成的:
