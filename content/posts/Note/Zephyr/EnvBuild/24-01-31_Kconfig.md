---
title: Zephyr -- Kconfig
date: 2024-01-31 15:40:28
tags:
- Zephyr
categories:
- Zephyr OS
---

## Setting Kconfig configuration values

生成的配置文件:
`zephyr/build/.config`: for CMake use.
`zephyr/build/zephyr/include/generated/autoconf.h`: for c file use.

所有的Kconfig配置会merge如下路径的Kconfig files：

1. `board/<arch>/<BOARD>/<BOARD>_defconfig`
2. CMake中定义的`CONFIG_XXX`
3. Application configuration(APP 目录下的Kconfig相关文件)

第三点Application configuration又会从如下路径获取Kconfig，默认使用`prj.conf`:

1. 如果定义了`CONF_FILE`, 会把该文件的Kconfig merge进来。`CONF_FILE`可以在如下定义
   1. App的`CMakeLists.txt`, 在`find_package(zephyr)`前定义。
   2. west直接传入`-DCONF_FILE=<conf file(s)>`
   3. From the CMake variable cache
2. 如果未定义`CONF_FILE`, 如果存在`prj_<BOARD>.conf`，merge进`prj.conf`。
3. 如果存在`board/<BOARD>.conf`，merge进`prj.conf`。
4. 如果存在`board/<BOARD>_<revision>.conf`，merge进`prj.conf`。
5. merge必须有的`prj.conf`。

如果`board/`下的`<BOARD>_defconfig`和`APP/`下的Kconfig冲突了，以APP的为准。
