---
title: Framebuffer
date: 2024-08-21 16:45:28
tags:
  - DRM
categories:
  - Video
draft: true
---

# Kconfig 配置

## Capemgr 配置

`linux-menuconfig:`

`CONFIG_CAPE_REALTEK`
`CONFIG_OF_OVERLAY`
`CONFIG_FW_LOADER`

`echo lcd_tianma_tm050rdh03 > sys/devices/platform/rts_capemgr/slots`

## LCD 配置

`menuconfig:`

`CONFIG_BR2_PACKAGE_LCD`

在`BR2_PACKAGE_LCD_LIST`即`Supported lcd display list`中加入 lcd 屏的名称`tianma_tm050rdh03`。(这里看`graphic/lcd/CMakeList.txt`，会寻找我们加入的屏名称，编译对应的 dts 到/lib/firmware)

`linux-menuconfig:`

`CONFIG_FB_RTS`
