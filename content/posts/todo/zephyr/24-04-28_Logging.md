---
date: 2024-04-30T16:42:41+08:00
title: 'Zephyr -- Logging'
draft: true
tags:
- Zephyr
categories:
- Zephyr OS
---

# Usage

## Logging in a module

一共有四种等级的Log, error, warning, info, debug。error等级最低，debug等级最高。

分为global log level(`CONFIG_LOG_DEFAULT_LEVEL`)和per module log level。其中global的配置只能提高module log level等级，而不能降低。

Module(Drivers)在使用API前必须define `LOG_LEVEL`宏。

e.g.

```c
#define LOG_LEVEL CONFIG_BT_BAS_LOG_LEVEL
#include <zephyr/logging/log.h>
LOG_MODULE_REGISTER(bas);
```

如果一个module有**多个文件**，需要在一个文件中定义`LOG_MODULE_REGISTER(foo)`, 其他文件需要`LOG_MODULE_DECLARE(foo)`来声明属于那个module。

`LOG_MODULE_REGISTER()`和`LOG_MODULE_DECLARE()`，可以填入第二个参数，表示compile time log level，如果没有则是默认的`CONFIG_LOG_DEFAULT_LEVEL`。

</br>

可以使用Template Kconfig来创建local log level configuration.

比如在spi/Kconfig中：

```c
module = SPI
module-str = spi
source "subsys/logging/Kconfig.template.log_config"
```

会自动定义`CONFIG_SPI_LOG_LEVEL`

## Logging in a module instance

如果一个module有多个instance，要在不同的instance中打印log，需要在driver config结构体中定义`LOG_INSTANCE_PTR_DECLARE`

```c
#include <zephyr/logging/log_instance.h>

struct foo_object {
     LOG_INSTANCE_PTR_DECLARE(log);
     uint32_t id;
}
```

// todo:

发现有两种写法，一种是：

```c
#define LOG_LEVEL CONFIG_BT_BAS_LOG_LEVEL
#include <zephyr/logging/log.h>
LOG_MODULE_REGISTER(bas);
```

另外一种直接:

```c
#include <zephyr/logging/log.h>
LOG_MODULE_REGISTER(bas, CONFIG_BT_BAS_LOG_LEVEL);
```
