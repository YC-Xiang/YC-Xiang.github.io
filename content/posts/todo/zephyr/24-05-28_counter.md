---
date: 2024-05-28T11:35:10+08:00
title: 'Zephyr -- Counter Driver'
tags:
- Zephyr
categories:
- Zephyr OS
draft: true
---

# Consumer

分周期性的timer和alarm两种。

周期性触发的timer:

初始化`struct counter_top_cfg`后调用`counter_set_top_value`。

```c++
top_cfg.flags = 0;
top_cfg.ticks = counter_us_to_ticks(timer_dev, (uint64_t)delay);
/* interrupt will be triggered periodically */
top_cfg.callback = timer_top_handler;
top_cfg.user_data = NULL;

/* set top value */
err = counter_set_top_value(timer_dev, &top_cfg);
if (err != 0) {
	shell_error(shctx, "%s: failed to set top value, err: %d", argv[ARGV_DEV], err);
	return err;
}
```

触发一次的alarm:

```c++
struct counter_alarm_cfg alarm_cfg;

counter_start();

// 配置alarm参数
alarm_cfg.flags = 0;
alarm_cfg.ticks = counter_us_to_ticks(counter_dev, DELAY);
alarm_cfg.callback = test_counter_interrupt_fn;
alarm_cfg.user_data = &alarm_cfg;

// 初始化alarm
counter_set_channel_alarm(counter_dev, ALARM_CHANNEL_ID,
					&alarm_cfg);

// 中断处理函数
static void test_counter_interrupt_fn(const struct device *counter_dev,
				      uint8_t chan_id, uint32_t ticks,
				      void *user_data)
{

}
```
