---
title: Pinctrl Subsystem
date: 2023-05-10 09:26:47
tags:
- Linux driver
categories:
- Linux driver
---

## Consumer

APIs:

```c++
bool pinctrl_gpio_can_use_line(unsigned gpio);
int pinctrl_gpio_request(unsigned gpio);
void pinctrl_gpio_free(unsigned gpio);
int pinctrl_gpio_direction_input(unsigned gpio);
int pinctrl_gpio_direction_output(unsigned gpio);
int pinctrl_gpio_set_config(unsigned gpio, unsigned long config);
```

pinctrl_gpio_can_use_line: 如果 dts 没有定义 gpio-ranges-group-names,
将 gpio index 映射到 pin index, 总是返回 true。\

统一驱动设备模型会处理 pin control：

```c++
platform_driver_register()
	driver_register();
		bus_add_driver();
			driver_attach();
				__driver_attach();
					device_driver_attach(drv, dev);
						driver_probe_device(drv, dev);
							really_probe(dev, drv);

	really_probe(struct device *dev, struct device_driver *drv)
		pinctrl_bind_pins(dev);
			devm_pinctrl_get(dev);
				pinctrl_lookup_state(dev->pins->p, PINCTRL_STATE_DEFAULT);
				pinctrl_lookup_state(dev->pins->p, PINCTRL_STATE_INIT);
				pinctrl_select_state(dev->pins->p, dev->pins->default_state);
```

## Provider

## Debug files

/sys/kernel/debug/pinctrl

## Reference

https://docs.kernel.org/driver-api/pin-control.html
