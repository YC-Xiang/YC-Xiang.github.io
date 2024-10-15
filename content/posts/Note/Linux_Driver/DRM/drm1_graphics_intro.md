---
date: 2024-08-20T17:00:27+08:00
title: DRM -- Graphic Introduction
tags:
  - DRM
categories:
  - DRM
hide:
  - true
---

# Reference

https://docs.kernel.org/gpu/index.html

introduction 中提供了一些关于 DRM 的讲座和 slides 材料。

https://bootlin.com/doc/training/graphics/graphics-slides.pdf

# Basic Theory and Concepts About Graphics

## YUV 数据格式

### 采样

YUV 格式，由一个 Y 的亮度分量（Luma）和 U（蓝色投影 Cb）和 V（红色投影 Cr）的色度分量（Chroma）表示。

![](https://xyc-1316422823.cos.ap-shanghai.myqcloud.com/20240821093649.png)

- 4:4:4 表示不降低色度（UV）通道的采样率。每个 Y 分量对应一组 UV 分量。
- 4:2:2 表示 2:1 水平下采样，没有垂直下采样。每两个 Y 分量共享一组 UV 分量。
- 4:2:0 表示 2:1 水平下采样，同时 2:1 垂直下采样。每四个 Y 分量共享一组 UV 分量。
- 4:1:1 表示 4:1 水平下采样，没有垂直下采样。每四个 Y 分量共享一组 UV 分量。

BT.601 中规定的 YUV 和 RGB 的转换公式：

\(R=Y+1.140*V-0.7\)
\(G=Y-0.343*U-0.711*V+0.526\)
\(B=Y+1.765*U-0.883\)

\(Y=0.299*R+0.587*G+0.114*B\)
\(U=-0.169*R-0.331*G+0.500*B\)
\(V=0.500*R-0.439*G-0.081\*B\)

### 存储格式

- **packed** 共同占一个 plane。
- **semi-planer** Y 占一个 plane，UV 一起占一个 plane。
- **planer** 每个分量分开存储，各占一个 plane。

![](https://xyc-1316422823.cos.ap-shanghai.myqcloud.com/20240822141334.png)

## FOURCC

FOURCC 用 4 个 char 字符来表示一种数据格式。比如在 linux drm 中`XRGB8888`被定义为`XR24`

```c
#define DRM_FORMAT_XRGB8888	fourcc_code('X', 'R', '2', '4') /* [31:0] x:R:G:B 8:8:8:8 little endian */
```

## Region Copy

又称 BitBlit，把图像从一个地址拷贝到另一个地址。

## Alpha Blending

每个 pixel 多存储一个 alpha channel 来表示透明度。alpha blending 表示将两张图片混合叠加，有多种合成操作，包括 over, in, out, atop, xor。

![](https://xyc-1316422823.cos.ap-shanghai.myqcloud.com/20240821110149.png)

以 over 为例，其中\(\alpha_0\)是结果的 alpha，\(C_0\)是运算结果。\(\alpha_a,\alpha_b\)是图像 A,B 的 Alpha 值，\(C_a, C_b\)是图像 A,B 中的像素。

![](https://xyc-1316422823.cos.ap-shanghai.myqcloud.com/20240821105806.png)

## Color Key

Colorkey 技术是作用在两个图像 alpha blending 的时候，对特殊色做特殊过滤。

## Scaling and interpolation

## Linear filtering and convolution

## Blur filters

## Dithering

用于提高低深度图片质量。

![](https://xyc-1316422823.cos.ap-shanghai.myqcloud.com/20240821111226.png)

# Hardware Aspects

## Display pipeline overview

![](https://xyc-1316422823.cos.ap-shanghai.myqcloud.com/20240821112934.png)

- Framebuffers
- Planes
- CRTC
- Encoder
- Connector
- Display

## Rendering hardware overview

Rendering 理解是相当于 GPU/2D accelerators 做的事情，对图像进行一些渲染处理。

- 基础像素处理 Basic pixel processing
  - 像素格式转换 pixel format conversion，dithering，scaling，blitting，blending
  - 由 Fixed-function hardware 处理。
- 复杂像素处理 Complex pixel processing
  - 各种计算操作
  - 由 programmable hardware(DSP)处理。
- 2D 向量绘制，2D vector drawing

# Software Aspects
