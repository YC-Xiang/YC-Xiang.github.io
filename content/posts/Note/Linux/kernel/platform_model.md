# Bus

```c


```

# Platform Device

Platform 设备是可以通过 CPU bus 直接寻址的设备, 内核在设备模型 bus, device 和 driver 的基础上,
进行了进一步封装, 抽象了 platform_bus, platform_device, platform_driver.

include/linux/platform_device.h, drivers/base/platform.c

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

	/* MFD cell pointer */
	struct mfd_cell *mfd_cell;

	/* arch specific additions */
	struct pdev_archdata	archdata;
};
```

id_entry:

mfd_cell: mfd 设备相关, 不关注.

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
	bool prevent_deferred_probe;
	bool driver_managed_dma;
};
```

platform device API:

```c
int platform_device_register(struct platform_device *);
void platform_device_unregister(struct platform_device *);

struct resource *platform_get_resource(struct platform_device *,
					      unsigned int, unsigned int);
extern int platform_get_irq(struct platform_device *, unsigned int);

int platform_add_devices(struct platform_device **, int);
struct platform_device *platform_device_register_full(
		const struct platform_device_info *pdevinfo);
int platform_device_add(struct platform_device *pdev)

static inline void *platform_get_drvdata(const struct platform_device *pdev);
static inline void platform_set_drvdata(struct platform_device *pdev, void *data);
```

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
```

# Platform Driver
