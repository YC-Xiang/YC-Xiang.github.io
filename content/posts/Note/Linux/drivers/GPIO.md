---
title: GPIO Subsystem
date: 2023-05-12 10:19:00
tags:
- Linux driver
categories:
- Linux driver
---

# Kernel doc: General Purpose Input/Output (GPIO)

## GPIO Driver Interface

### Controller Drivers: gpio_chip

`struct gpio_chip` 抽象 gpio controller。

`gpiochip_add_data()` or `devm_gpiochip_add_data()`接口用来注册 gpio controller。
`gpiochip_remove()`释放 gpio controller。

`gpiochip_is_request()`在 gpio controller driver 中用于检测某个 gpio 是否被其他
chip 占用，没占用返回 NULL，占用返回 request 时传入的 string。

### GPIO electrical configuration

gpio_chip 的`.set_config`回调用于设置：

- Debouncing
- Single-ended modes (open drain/open source)
- Pull up and pull down resistor enablement

这些属性可以在 dts 中指定，`include/dt-bindings/gpio/gpio.h` `GPIO_PUSH_PULL`
`GPIO_LINE_OPEN_SOURCE` `GPIO_OPEN_DRAIN `...

可以设置为`gpiochip_generic_config()`会调用到`pinctrl_gpio_set_config()`->
`ops->pin_config_set`

### GPIO drivers providing IRQs

```c++
gpiod_to_irq(); // 传入 gpio_desc，返回 gpio 的 irq number(软件映射的，不是 irq hw id)
	gpio_chip_hwgpio();
		gc->to_irq();
			rts_gpio_to_irq();
				irq_linear_revmap();

```

#### Cascaded GPIO irqchips

1. CHAINED CASCADED GPIO IRQCHIPS：挺多 soc 上是这种做法，
打开`CONFIG_GPIOLIB_IRQCHIP`设置 girq->parent_handler。
gpio controller 注册过程中通过**irq_set_chained_handler**设置中断处理函数，
因此在中断处理函数中需要 chained_irq_enter，chained_irq_exit。
相当于级联中断处理器的做法。

```c++
static irqreturn_t foo_gpio_irq(int irq, void *data) /// 中断处理函数
    chained_irq_enter(...);
    generic_handle_irq(...);
    chained_irq_exit(...);
```

1. GENERIC CHAINED GPIO IRQCHIPS：rts3917 是这种做法，
通过**reuqest_irq**进入的 rts_irq_handler 中断处理函数。
发现的每一个 gpio 都进入 generic_handle_irq，
最后会到各自 irq_desc 中通过 request_irq 的中断处理函数。

```c++
static irqreturn_t rts_irq_handler(int irq, void *dev_id)
    for each detected GPIO IRQ
        generic_handle_irq(...);
```

1. NESTED THREADED GPIO IRQCHIPS：gpio expander 的做法，不深究。

### Infrastructure helpers for GPIO irqchips

GPIO 子系统有针对中断的一套框架，Kconfig 为`GPIOLIB_IRQCHIP`，
rts 没有用到就不分析了。可以看文档中具体的解释。

注意一点，方法一：如果 parent_handler
赋值了`girq->parent_handler = ftgpio_gpio_irq_handler`
应该就是上面`CHAINED CASCADED GPIO IRQCHIPS`的做法。

```c++
gpiochip_add_data();
    gpiochip_add_data_with_key();
    	gpiochip_add_irqchip();
    		irq_set_chained_handler_and_data(gc->irq.parents[i], gc->irq.parent_handler, data);
```

方法二：`girq->parent_handler = NULL`，直接在 driver 中`devm_request_threaded_irq`
对应`GENERIC CHAINED GPIO IRQCHIPS`的做法。

## GPIO Descriptor Consumer Interface

![](https://xyc-1316422823.cos.ap-shanghai.myqcloud.com/gpio.png)

```c++
struct gpio_desc *gpiod_get(struct device *dev, const char *con_id, enum gpiod_flags flags);
struct gpio_desc *gpiod_get_index(struct device *dev, const char *con_id, unsigned int idx, enum gpiod_flags flags);
// return NULL 如果没有 request 到 GPIO
struct gpio_desc *gpiod_get_optional(struct device *dev, const char *con_id, enum gpiod_flags flags);
```

## GPIO Mappings

### device tree

```c++
foo_device {
        compatible = "acme,foo";
        ...
        led-gpios = <&gpio 15 GPIO_ACTIVE_HIGH>, /* red */
                    <&gpio 16 GPIO_ACTIVE_HIGH>, /* green */
                    <&gpio 17 GPIO_ACTIVE_HIGH>; /* blue */

        power-gpios = <&gpio 1 GPIO_ACTIVE_LOW>;
};
```

对应

```c++
struct gpio_desc *red, *green, *blue, *power;

red = gpiod_get_index(dev, "led", 0, GPIOD_OUT_HIGH);
green = gpiod_get_index(dev, "led", 1, GPIOD_OUT_HIGH);
blue = gpiod_get_index(dev, "led", 2, GPIOD_OUT_HIGH);

power = gpiod_get(dev, "power", GPIOD_OUT_HIGH);
```

gpiod_set_value 设置的值是“逻辑值”，不一定等于物理值。

## Sysfs 接口

/sys/class/gpio/

echo 19 > export

/sys/class/gpio/gpioN/

/sys/kernel/debug/gpio 可以看哪些 gpio 被申请了

# Reference

GPIO 推挽和开漏输出：https://zhuanlan.zhihu.com/p/41942876
