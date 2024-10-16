---
date: 2024-08-19T17:00:27+08:00
title: MIPI DPI 协议介绍
tags:
  - MIPI
categories:
  - Video
draft: true
---

# Reference

MIPI Alliance Standard for Display Pixel Interface
(DPI-2)

# Overview

MIPI DPI(Display Pixel Interface)协议。分辨率最大支持到 800\*480。

Parallel RGB 屏遵从的就是 MIPI DPI 协议。

# 4 Display Architectures and Interface Constructions

显示设备一般有 4 种结构:

**Type1**: includes a display device, display driver IC, full-frame memory, registers, timing controller, non-volatile memory and control interface.

![](https://xyc-1316422823.cos.ap-shanghai.myqcloud.com/20240814153605.png)

**Type2**: includes a display device, display driver IC, **partial-frame memory**, registers, timing controller, non-volatile memory, control interface and **video stream interface**.

![](https://xyc-1316422823.cos.ap-shanghai.myqcloud.com/20240814153619.png)

**Type3**: includes a display device, display driver IC, ~~partial-frame memory~~, registers, timing controller, non-volatile memory, control interface and video stream interface

![](https://xyc-1316422823.cos.ap-shanghai.myqcloud.com/20240814153645.png)

**Type4**: includes a display device, display driver IC, ~~partial-frame memory~~, registers, timing controller, ~~non-volatile memory~~, control lines and video stream interface.

RGB 屏属于 Type4。

![](https://xyc-1316422823.cos.ap-shanghai.myqcloud.com/20240814153657.png)

# 5 Display Pixel Interface Interoperability

Host 和显示屏之间需要连接 power signals 和 interface singals。

![](https://xyc-1316422823.cos.ap-shanghai.myqcloud.com/20240814152622.png)

其中 Host 需要实现 24-bit 的 data bus width，可调整输出为 16-bit/18-bit。

只有 Type4 需要 SD(shutdown), CM(color mode) 控制信号线。

# 6 Interface Signal Description

解释上图中各信号线的作用：

Power signals:

- \(V\_{DD}\)：Power supply
- \(V\_{DDI}\): I/F logic level supply
- \(AGND\): Analog GND
- \(DGND\): Digital GND

Interface signals:

- \(V\_{sync}\): Vertical sync
- \(H\_{sync}\): Horizontal sync
- DE: Data Enable
- PCLK: pixel clock
- D[15:0], D[17:0], D[23:0]: pixel data
- SD: shutdown
- CM: color mode

# 7 Programmable Timing Parameters

Vsync 表示某一帧的开始，Hsync 表示每一行的开始。

显示屏会在 PCLK 上升沿采样。

![](https://xyc-1316422823.cos.ap-shanghai.myqcloud.com/20240814160720.png)

每帧的周期为 Vsync+VBP+VAdr+VFP, 每行的周期为 Hsync+HBP+HAdr+HFP。

DPI 协议对这些参数的要求有：

![](https://xyc-1316422823.cos.ap-shanghai.myqcloud.com/20240814161328.png)

# 8 Interface Color Coding

Table 4 展示了 16-bit, 18-bit, 24-bit RGB 数据在 D[23:0]上的传输编码。16-bit 可以有三种配置，18-bit 有两种，24-bit 只有一种。

这边 Table 参考 spec。

# 9 Interface Electrical Characteristics

主要是一些电气信号的规范，略。

# 11 Type2 and Type3 Display Architecture Control Interfaces

Type2,3 参考 MIPI DBI 的 TypeC 类型。

# 12 Type4 Architecute Shutdown and Color Mode signals

# 12.1 Shutdown for Type4 Architecture

SD 拉高进入 sleep mode，拉低唤醒。时序参考 spec。

# 12.2 Color Mode for Type4 Architecture

CM 拉高从 full-color mode -> 8-color mode。  
CM 拉低从 8-color mode -> full-color mode。

时序参考 spec。

# Driver IC

ST7262
HX8264
