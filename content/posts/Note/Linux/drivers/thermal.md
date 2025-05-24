# Data Structure

```c++
struct thermal_zone_device {
	int id;
	char type[THERMAL_NAME_LENGTH];
	struct device device;
	struct attribute_group trips_attribute_group;
	struct thermal_attr *trip_temp_attrs;
	struct thermal_attr *trip_type_attrs;
	struct thermal_attr *trip_hyst_attrs;
	enum thermal_device_mode mode;
	void *devdata;
	struct thermal_trip *trips;
	int num_trips;
	unsigned long trips_disabled;	/* bitmap for disabled trips */
	unsigned long passive_delay_jiffies;
	unsigned long polling_delay_jiffies;
	int temperature;
	int last_temperature;
	int emul_temperature;
	int passive;
	int prev_low_trip;
	int prev_high_trip;
	atomic_t need_update;
	struct thermal_zone_device_ops *ops;
	struct thermal_zone_params *tzp;
	struct thermal_governor *governor;
	void *governor_data;
	struct list_head thermal_instances;
	struct ida ida;
	struct mutex lock;
	struct list_head node;
	struct delayed_work poll_queue;
	enum thermal_notify_event notify_event;
	bool suspended;
};
```

`id`: thermal zone unique id。

`type`: thermal zone 的名称。

`temperature`: 当前 thermal zone 的温度。

`last_temperature`: 上一次统计的 thermal zone 温度。

# DTS

rk3399.dtsi:

```txt
cpu_l0: cpu@0 {
	device_type = "cpu";
	compatible = "arm,cortex-a53";
	reg = <0x0 0x0>;
	#cooling-cells = <2>;
	dynamic-power-coefficient = <100>;
	cpu-idle-states = <&CPU_SLEEP &CLUSTER_SLEEP>;
};

tsadc: tsadc@ff260000 {
	compatible = "rockchip,rk3399-tsadc";
	#thermal-sensor-cells = <1>;
};

thermal_zones: thermal-zones {
	cpu_thermal: cpu-thermal {
		polling-delay-passive = <100>;
		polling-delay = <1000>;
		thermal-sensors = <&tsadc 0>;

		trips {
			cpu_alert0: cpu_alert0 {
				temperature = <70000>;
				hysteresis = <2000>;
				type = "passive";
			};
			cpu_alert1: cpu_alert1 {
				temperature = <75000>;
				hysteresis = <2000>;
				type = "passive";
			};
			cpu_crit: cpu_crit {
				temperature = <95000>;
				hysteresis = <2000>;
				type = "critical";
			};
		};

		cooling-maps {
			map0 {
				trip = <&cpu_alert0>;
				cooling-device =
					<&cpu_b0 THERMAL_NO_LIMIT THERMAL_NO_LIMIT>,
					<&cpu_b1 THERMAL_NO_LIMIT THERMAL_NO_LIMIT>;
			};
			map1 {
				trip = <&cpu_alert1>;
				cooling-device =
					<&cpu_l0 THERMAL_NO_LIMIT THERMAL_NO_LIMIT>,
					<&cpu_l1 THERMAL_NO_LIMIT THERMAL_NO_LIMIT>,
					<&cpu_l2 THERMAL_NO_LIMIT THERMAL_NO_LIMIT>,
					<&cpu_l3 THERMAL_NO_LIMIT THERMAL_NO_LIMIT>,
					<&cpu_b0 THERMAL_NO_LIMIT THERMAL_NO_LIMIT>,
					<&cpu_b1 THERMAL_NO_LIMIT THERMAL_NO_LIMIT>;
			};
		};
	};

	gpu_thermal: gpu-thermal {
		polling-delay-passive = <100>;
		polling-delay = <1000>;
		thermal-sensors = <&tsadc 1>;

		trips {
			gpu_alert0: gpu_alert0 {
				temperature = <75000>;
				hysteresis = <2000>;
				type = "passive";
			};
			gpu_crit: gpu_crit {
				temperature = <95000>;
				hysteresis = <2000>;
				type = "critical";
			};
		};

		cooling-maps {
			map0 {
				trip = <&gpu_alert0>;
				cooling-device =
					<&gpu THERMAL_NO_LIMIT THERMAL_NO_LIMIT>;
			};
		};
	};
};
```

在 thermal sensor 节点，需要指明 `#thermal-sensor-cells = <n>`, n 为 0 或 1.

当 thermal zone 中只有一个 thermal sensor 时，可以省略 thermal-sensors 的 index：`thermal-sensors = <&tsadc>;`。当有超过一个 thermal sensor 时，需要在第二个参数指明 index，`thermal-sensors = <&tsadc 0>;`

在 cooling device 节点，需要指明 `#cooling-cells = <n>`。

# Cooling device

cooling device driver 通过下面的三个 api 注册 cooling device，并挂入 thermal_cdev_list 全局链表：

```c++
thermal_cooling_device_register();
thermal_of_cooling_device_register();
devm_thermal_of_cooling_device_register();
```

设备树 cooling-maps 下面的每个 map 代表一个 trip 和各个 cooling devices 的绑定关系。

注册 thermal zone 时，会遍历 thermal_cdev_list 全局链表，将 cooling deivce 和 trip 绑定在一起，每一个绑定用 struct thermal_instance 表示，挂在 thermal_zone_device->thermal_instances 链表中。

```c++
thermal_of_zone_register();
	of_ops->bind = thermal_of_bind;
```

```c++
thermal_of_zone_register();
	of_ops->bind = thermal_of_bind;

thermal_zone_device_register_with_trips();
	bind_tz();
		list_for_each_entry(pos, &thermal_cdev_list, node) { // 遍历所有 cooling device
			ret = tz->ops->bind(tz, pos); // 进入 thermal_of_bind()
		}

thermal_of_bind();
	thermal_of_for_each_cooling_maps(); // 对 cooling-maps 下面的每个 map 调用 __thermal_of_bind

__thermal_of_bind();
	thermal_zone_bind_cooling_device(); // 一个 cooling device 绑定一个 trip, 注册 struct thermal_instance，挂入全局链表
```

# Hysteresis 机制

向上触发：温度上升时，直接使用 trip.temperature 作为触发阈值

向下恢复：温度下降时，使用 trip.temperature - trip.hysteresis 作为恢复阈值

假设有一个 trip point 设置为 90°C，hysteresis 为 5°C：

- 升温过程：温度达到 90°C 时触发热保护
- 降温过程：温度必须降到 85°C 以下才会取消热保护
- 防抖效果：温度在 85°C-90°C 之间波动时，不会频繁触发/取消事件

# 温度更新流程

达到 trip 温度，触发中断，在中断函数中调用 thermal_zone_device_update：

```c++
thermal_zone_device_update();
	__thermal_zone_device_update();

void __thermal_zone_device_update(struct thermal_zone_device *tz,
				  enum thermal_notify_event event)
{
	int count;

	update_temperature(tz); // 更新当前 thermal zone 的温度

	__thermal_zone_set_trips(tz); // 重新设定上下两档触发温度中断的 trips

	tz->notify_event = event;

	for (count = 0; count < tz->num_trips; count++)
		handle_thermal_trip(tz, count); // 找到触发当前中断的 trip，发送 netlink event，并根据不同 trip 类型调用相关回调

	monitor_thermal_zone(tz);
}
```
