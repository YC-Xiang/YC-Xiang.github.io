---
date: 2024-04-28T09:27:10+08:00
title: 'Zephyr -- Watchdog Driver'
tags:
- Zephyr
categories:
- Zephyr OS
---

# Consumer

```c++
// config watchdog
static inline int wdt_install_timeout(const struct device *dev,
				      const struct wdt_timeout_cfg *cfg);
// enable watchdog
__syscall int wdt_setup(const struct device *dev, uint8_t options);
// disable watchdog
__syscall int wdt_disable(const struct device *dev);
// feed wathdog
__syscall int wdt_feed(const struct device *dev, int channel_id);
```

`wdt_install_timeout()`需要在`wdt_setup()`之前。

示例code:

```c++
//samples/drivers/watchdog/src/main.c

struct wdt_timeout_cfg wdt_config = {
	/* Reset SoC when watchdog timer expires. */
	.flags = WDT_FLAG_RESET_SOC,

	/* Expire watchdog after max window */
	.window.min = 0,
	.window.max = 1000,
};

static void wdt_callback(const struct device *wdt_dev, int channel_id)
{
	static bool handled_event;

	if (handled_event) {
		return;
	}

	wdt_feed(wdt_dev, channel_id);

	printk("Handled things..ready to reset\n");
	handled_event = true;
}

int main()
{
	// 需要driver支持watchdog interrupt
	wdt_config.callback = wdt_callback;
	wdt_channel_id = wdt_install_timeout(wdt, &wdt_config);
	wdt_setup(wdt, WDT_OPT);
}

```
