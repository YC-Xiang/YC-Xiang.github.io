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
```c

```


```c
struct counter_alarm_cfg alarm_cfg;

counter_start();

// 初始化alarm
alarm_cfg.flags = 0;
alarm_cfg.ticks = counter_us_to_ticks(counter_dev, DELAY);
alarm_cfg.callback = test_counter_interrupt_fn;
alarm_cfg.user_data = &alarm_cfg;

//
counter_set_channel_alarm(counter_dev, ALARM_CHANNEL_ID,
					&alarm_cfg);

static void test_counter_interrupt_fn(const struct device *counter_dev,
				      uint8_t chan_id, uint32_t ticks,
				      void *user_data)
{

}
```
