---
date: 2025-07-28T11:05:36+08:00
title: "V4L2 Userspace API -- interfaces"
tags:
  - V4L2
categories:
  - V4L2
---

## 4.1 Video Capture Interface

## 4.5 Video Memory-To-Memory Interface

// TODO:

## 4.12 Event Interface

// TODO:

## 4.13 Sub-device Interface

### 4.13.1 Controls

大部分 V4L2 controls 都由 sub-device hardware 实现，drivers 通常会把所有 controls 合并起来，通过 video device nodes 暴露给 userspace。

复杂的设备有时会在不同的硬件中实现相同的 control，例如对比度调整、白平衡。这时可以通过 sub-device 的 node 将 control 暴露出去，区别是通过不同的 sub-device 来实现某个功能的调整。

### 4.13.3 Pad-level Formats

图像的 format 通常使用 format 和 selection ioctl 在 video capture/output devices 之间协商。driver 负责将 video pipeline 中每个 block 都设置好该配置。

然而对于一些复杂的设备，相同的 pipeline output size 可以通过不同的硬件配置达到。比如图像缩放可以通过 sensor 或者 isp 实现。

sensor 实现缩放一般质量更低，但可以实现更高的帧率。根据应用场景不同 (图像质量 or 速度)，pipeline 必须要下不同的配置，app 必须对 pipeline 中的每一个点去下配置。

实现了 media api 的 driver，可以把 pad-level image format configuration 暴露给 app,app 可以通过 `VIDIOC_SUBDEV_G_FMT` 和 `VIDIOC_SUBDEV_S_FMT` 来实现对每个 pad 的 format 控制。

app 需要保证每个 connected pad 的 size&format 相同，否则会在 `VIDIOC_STREAMON` 时返回 EPIPE.

#### 4.13.3.1 Format Negotiation

可接受的 pad format 通常依赖于外部的一些参数，比如其他 pads 的 format, active links, controls. 找到一个 video pipeline 中所有 pads formats 的组合，能够满足 driver 和 app 的要求，不能够只依赖于 formats enumeration, 需要一个 format 协商的机制。

format negotiation 的核心是 get/set format operations. 通过将`VIDIOC_SUBDEV_G_FMT` 和 `VIDIOC_SUBDEV_S_FMT` ioctl 的 `which` 参数设置为 `V4L2_SUBDEV_FORMAT_TRY`, 设置 try formats 不会影响到 device state。

try format 不属于 device state, 而是保存在 sub-device 的 file handle 中。`VIDIOC_SUBDEV_G_FMT` 会返回 sub-device file handle 最后一次的 try format. 这样几个 app 同时 querying sub-device format 不会互相干扰。

app 使用 `VIDIOC_SUBDEV_S_FMT` 来查看一个具体的 format 是否被支持。driver 验证该 format，可能还会修改该 format 并返回修改之后的 format 值。

driver 自动在 sub-devices 中传播 format, 当一个 try/active format 设置到一个 pad, 该 sub-device 其他的 pads 可以被 driver 自由修改，但要遵守以下规则：

- format 应该从 sink pads 传播到 source pads。修改 source pads 不应该传播到 sink pads.
- 当 sink pads 被修改时，使用 variable scaling factor 缩放 frames 的 sub-device, 应该要把 scale factor reset 到默认值。如果支持 1:1 缩放比例，意味着 source pads 应该被 reset 成和 sink pads formats 一样。

Formats 不会在 links 之间传播。因为涉及到了多个 file handle. app 必须小心地设置好 link 两段的 formats。

Format Negotiation Example:

![](https://xyc-1316422823.cos.ap-shanghai.myqcloud.com/20250730151841.png)

1. Initial state. sensor pad format 设置为默认的 3MP. media bus code 设置为默认的 V4L2_MBUS_FMT_SGRBG8_1X8。
2. app 设置 frontend sink pad format size to 2048x1536, media bus code V4L2_MBUS_FMT_SGRBG_1X8. driver 将 sink pad format 传播到 source pad。
3. app 设置 scaler sink pad format size to 2046x1534, media bus code V4L2_MBUS_FMT_SGRBG_1X8. driver 将 sink pad size 传播到 sink pad compose selection rectangle。format 传播到 scaler source pad。
4. app 设置 scaler sink pad compose selection rectangle size to 1280x960, driver 将 size 传播到 scaler source pad format。

#### 4.13.3.2 Selections: cropping, scaling and composition

对于 **sink pads**:

crop 直接应用于当前 pad format, 当前的 sink pad format 代表从 pipeline 上一个 block 拿到的 image size。crop rectangle 代表 sub device 中要传播给后面的 sub-image.

scaling 是可选的，如果 sub device 支持的话，对于 sink pad 通过 `VIDIOC_SUBDEV_S_SELECTION` + `V4L2_SEL_TGT_COMPOSE` 来缩放 crop rectangle. 如果 subdev 支持 scaling，但不支持 composing，那么 top 和 left 必须设置为 0.

对于 **source pads**:

crop 操作和 sink pad 类似，但是操作的对象是 sink pad 的 compose rectangle。

#### 4.13.3.3 Types of selection targets

**Acutal targets**

真实硬件配置的 rectangle 范围。

**BOUNDS targets**

包含所有 valid actual rectangles 的最小 rectangle。因为有的 sensor pixel array 是十字形，圆形，所以 BOUNDS 范围是一个矩最小矩形，把这些范围框起来。所以最大的 size 可能比 BOUNDS 矩形小。

#### 4.13.3.4 Order of configuration and format propagation

配置顺序为：

1. Sink pad format
2. Sink pad crop selection
3. Sink pad compose selection
4. Source pad crop selection. 在 sink pad compose selection 上做。
5. Source pad format。width 和 height 不用设置，直接用 source pad crop selection 的 size。

三个例子：

sub device 只有 sink pad 支持 crop:

![](https://xyc-1316422823.cos.ap-shanghai.myqcloud.com/20250730164739.png)

sub device 支持先 cropping, 再 scaling, 最后再 cropping 到两个 source pads：

![](https://xyc-1316422823.cos.ap-shanghai.myqcloud.com/20250730164937.png)

sub device 支持两个 sink pads 的 cropping, scaling, composing。再从 composed 图像中 crop 出两路 stream 给两个 source pads.

![](https://xyc-1316422823.cos.ap-shanghai.myqcloud.com/20250730165107.png)

# 8. Common selection definitions

sub device 不支持 `V4L2_SEL_TGT_COMPOSE_DEFAULT` 和 `V4L2_SEL_TGT_COMPOSE_PADDED`.

**selection flags:**

- `V4L2_SEL_FLAG_GE`: Suggest the driver it should choose greater or equal rectangle (in size) than was requested. 如果没有这个 flag，driver 会选择 the closest possible rectangle。
- `V4L2_SEL_FLAG_LE`: Suggest the driver it should choose lesser or equal rectangle (in size) than was requested.
- `V4L2_SEL_FLAG_KEEP_CONFIG`: 禁止在 sub device 中向后传播 selection 配置。
