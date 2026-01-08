---
title: PWM Subsystem
date: 2023-03-21 16:00:00
tags:
  - Linux driver
categories:
  - Linux driver
---

# PWM 子系统

# PWM 原理

![](https://xyc-1316422823.cos.ap-shanghai.myqcloud.com/RISC-V%E4%B8%AD%E6%96%87%E6%89%8B%E5%86%8C/Untitled.png)

利用微处理器的数字输出来对模拟电路进行控制的一种非常有效的技术，其本质是一种对模拟信号电平进行数字编码的方法。在嵌入式设备中，PWM 多用于控制马达、LED、振动器等模拟器件.

- 脉冲周期（T），单位是时间，ns, us ,ms。
- 脉冲频率（f），单位是赫兹（Hz），与脉冲周期成倒数关系，f=1/T。
- 脉冲宽度（W），简称“脉宽”，是脉冲高电平持续的时间。单位是时间，ns, us, ms。
- 占空比（D），脉宽除以脉冲周期的值。

W = ton

T = ton + toff = 1/f

D = ton / (ton+ toff) = ton / T

# PWM consumer

```c++
/* include/linux/pwm.h */
int pwm_config(struct pwm_device *pwm, int duty_ns, int period_ns);
int pwm_enable(struct pwm_device *pwm);
void pwm_disable(struct pwm_device *pwm);
int pwm_apply_state(struct pwm_device *pwm, const struct pwm_state *state);
```

`pwm_config`，用于控制 PWM 输出信号的频率和占空比，其中频率是以周期（`period_ns`）的形式配置的，占空比是以有效时间（`duty_ns`）的形式配置的。

`pwm_enable/pwm_disable`，用于控制 PWM 信号输出与否。

`pwm_apply_state`需要定义一个`pwm_state`，可以一下子修改`period/duty_cycle/polarity/enabled`。

上面的 API 都以`struct pwm_device`类型的指针为操作句柄，该指针抽象了一个 PWM 设备，那么怎么获得 PWM 句柄呢？使用如下的 API：

```c++
 /* include/linux/pwm.h */
struct pwm_device *pwm_get(struct device *dev, const char *con_id);
struct pwm_device *of_pwm_get(struct device_node *np, const char *con_id);
void pwm_put(struct pwm_device *pwm);
struct pwm_device *devm_pwm_get(struct device *dev, const char *con_id);
struct pwm_device *devm_of_pwm_get(struct device *dev, struct device_node *np, const char *con_id);
void devm_pwm_put(struct device *dev, struct pwm_device *pwm);
```

`pwm_get/devm_pwm_get`，从指定设备（dev）的 DTS 节点中，获得对应的 PWM 句柄。可以通过 con_id 指定一个名称，或者会获取和该设备绑定的第一个 PWM 句柄。设备的 DTS 文件需要用这样的格式指定所使用的 PWM device：

```c++
bl: backlight {       
pwms = <&pwm 0 5000000 PWM_POLARITY_INVERTED>;       
pwm-names = "backlight";
};
```

如果“con_id”为 NULL，则返回 DTS 中“pwms”字段所指定的第一个 PWM device；如果“con_id”不为空，如是“backlight”，则返回和“pwm-names ”字段所指定的 name 对应的 PWM device。

上面“pwms”字段各个域的含义如下：  
1）`&pwm`，对 DTS 中 pwm 节点的引用；  
2）`0`，pwm device 的设备号，具体需要参考 SOC 以及 pwm driver 的实际情况；  
3）`5000000`，PWM 信号默认的周期，单位是纳秒（ns）；  
4）`PWM_POLARITY_INVERTED`，可选字段，是否提供由 pwm driver 决定，表示 pwm 信号的极性，若为 0，则正常极性，若为`PWM_POLARITY_INVERTED`，则反转极性。

`of_pwm_get/devm_of_pwm_get`，和`pwm_get/devm_pwm_get`类似，区别是可以指定需要从中解析 PWM 信息的`device node`，而不是直接指定 device 指针。

# PWM provider

### 2.1 pwm_chip

抽象 PWM 控制器

在一个 SOC 中，可以同时支持多路 PWM 输出，以便同时控制多个 PWM 设备。这样每一路 PWM 输出，可以看做一个 PWM 设备（struct pwm_device）

```c++
/**
 * struct pwm_chip - abstract a PWM controller
 * @dev: device providing the PWMs
 * @ops: callbacks for this PWM controller
 * @base: number of first PWM controlled by this chip
 * @npwm: number of PWMs controlled by this chip
 * @of_xlate: request a PWM device given a device tree PWM specifier
 * @of_pwm_n_cells: number of cells expected in the device tree PWM specifier
 * @list: list node for internal use
 * @pwms: array of PWM devices allocated by the framework
 */
struct pwm_chip {
	struct device *dev;
	const struct pwm_ops *ops;
	int base; // 动态指定pwm起始软件编号。旧的pwm_request会使用。在底层driver指定，传入-1即可。
	unsigned int npwm;

	struct pwm_device * (*of_xlate)(struct pwm_chip *pc,
					const struct of_phandle_args *args);
	unsigned int of_pwm_n_cells;

	/* only used internally by the PWM framework */
	struct list_head list;
	struct pwm_device *pwms;
};
```

`dev`，该 pwm chip 对应的设备，一般由 pwm driver 对应的 platform 驱动指定。必须提供！

`ops`，操作 PWM 设备的回调函数，后面会详细介绍。必须提供！

`npwm`，该 pwm chip 可以支持的 pwm channel（也可以称作 pwm device 由 struct pwm_device 表示）个数，kernel 会根据该 number，分配相应个数的 struct pwm_device 结构，保存在 pwms 指针中。必须提供！

`pwms`，保存所有 pwm device 的数组，kernel 会自行分配，不需要 driver 关心。

`base`，在将该 chip 下所有 pwm device 组成 radix tree 时使用，只有旧的 pwm_request 接口会使用，因此忽略它吧，编写 pwm driver 不需要关心。

`of_pwm_n_cells`，该 PWM chip 所提供的 DTS node 的 cell，一般是 2 或者 3，例如：为 3 时，consumer 需要在 DTS 指定 pwm number、pwm period 和 pwm flag 三种信息（如 2.1 中的介绍）；为 2 时，没有 flag 信息。

`of_xlate`，用于解析 consumer 中指定的、pwm 信息的 DTS node 的回调函数（如 2.1 中介绍的，pwms = <&pwm 0 5000000 PWM_POLARITY_INVERTED>）。

注 2：一般情况下，of_pwm_n_cells 取值为 3，或者 2（不关心极性），of_xlate 则可以使用 kernel 提供的 of_pwm_xlate_with_flags（解析 of_pwm_n_cells 为 3 的 chip）或者 of_pwm_simple_xlate（解析 of_pwm_n_cells 为 2 的情况）。

### 2.2 pwm_ops

```c++
struct pwm_ops {
	int (*request)(struct pwm_chip *chip, struct pwm_device *pwm);
	void (*free)(struct pwm_chip *chip, struct pwm_device *pwm);
	int (*capture)(struct pwm_chip *chip, struct pwm_device *pwm,
		       struct pwm_capture *result, unsigned long timeout);
	int (*apply)(struct pwm_chip *chip, struct pwm_device *pwm,
		     const struct pwm_state *state);
	void (*get_state)(struct pwm_chip *chip, struct pwm_device *pwm,
			  struct pwm_state *state);
	struct module *owner;

	/* Only used by legacy drivers 目前rts使用的是legacy接口 */
	int (*config)(struct pwm_chip *chip, struct pwm_device *pwm,
		      int duty_ns, int period_ns);
	int (*set_polarity)(struct pwm_chip *chip, struct pwm_device *pwm,
			    enum pwm_polarity polarity);
	int (*enable)(struct pwm_chip *chip, struct pwm_device *pwm);
	void (*disable)(struct pwm_chip *chip, struct pwm_device *pwm);
};
```

这些回调函数的操作对象是具体的`pwm device`，包括：

`config`，配置`pwm device`的频率、占空比。必须提供！

`enable/disable`，使能/禁止 pwm 信号输出。必须提供！

`request/free`，不再使用。

`set_polarity`，设置 pwm 信号的极性。可选，具体需要参考`of_pwm_n_cells`的定义。

### 2.3 pwm device

```c++
struct pwm_device {
	const char *label;
	unsigned long flags;
	unsigned int hwpwm; // pwm device对应的hardware pwm number，可用于寄存器的寻址操作
	unsigned int pwm; // chip->base + hwpwm
	struct pwm_chip *chip;
	void *chip_data;

	struct pwm_args args;
	struct pwm_state state;
};
```

### 2.4 **pwmchip_add/pwmchip_remove**

初始化完成后的 pwm chip 可以通过 pwmchip_add 接口注册到 kernel 中：

```c++
  1: int pwmchip_add(struct pwm_chip *chip);
  2: int pwmchip_remove(struct pwm_chip *chip);
```

# API 使用指南

### 3.2 provider 编写 PWM driver 的步骤

1）创建代表该 pwm driver 的 DTS 节点，并提供 platform device 有关的资源信息，例如：

2）定义一个`pwm chip`变量。

3）注册相应的 platform driver，并在 driver 的.probe()接口中，初始化 pwm chip 变量，至少要包括如下字段：

`dev`，使用 platform device 中的 dev 指针即可；`npwm；ops`，至少包括`config、enable、disable`三个回调函数。

**如果该 pwm chip 支持额外的 flag（如 PWM 极性，或者自定义的 flag），将 PWM cell 指定为 3（of_pwm_n_cells），of_xlate 指定为 of_pwm_xlate_with_flags。**

4）每当 consumer 有 API 调用时，kernel 会以 pwm device 为参数，调用 pwm driver 提供的 pwm ops，相应的回调函数可以从 pwm device 中取出 pwm number（该 number 的意义 driver 自行解释），并操作对应的寄存器即可。

### 3.3 **consumer 使用 PWM 的步骤**

1）查看 pwm provider 所提供的 pwm dts binding 信息（一般会在“Documentation/devicetree/bindings/pwm”目录中），并以此在该 device 所在的 dts node 中添加“pwms ”以及“pwm-names ”相关的配置。例如：

```c++
/* arch\arm\boot\dts\imx23-evk.dts */

backlight {       
compatible = "pwm-backlight";        
pwms = <&pwm 2 5000000>;       
brightness-levels = <0 4 8 16 32 64 128 255>;       
default-brightness-level = <6>;
};
```

2）在 driver 的 probe 接口中，调用**devm_pwm_get**接口，获取**pwm device**句柄，并保存起来。

3）devm_pwm_get 成功后，该 pwm 信号已经具备初始的周期和极性。后续根据需要，可以调用**pwm_config**更改该 pwm 信号的周期、占空比。

4）driver 可以根据需要，调用**pwm_enable/pwm_disable**接口，打开或者关闭 pwm 信号的输出。

# Reference

原理介绍：[https://zhuanlan.zhihu.com/p/374083276](https://zhuanlan.zhihu.com/p/374083276)

Driver 解析：[http://www.wowotech.net/comm/pwm_overview.html](http://www.wowotech.net/comm/pwm_overview.html)
