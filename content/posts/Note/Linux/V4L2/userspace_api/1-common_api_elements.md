---
date: 2025-07-28T11:04:36+08:00
title: "V4L2 Userspace API -- Common API Elements"
tags:
  - V4L2
categories:
  - V4L2
---

# Common API Elements

## 1.1 Opening and Closing Devices

### 1.1.1 Controlling a hardware peripheral via V4L2

需要使用 media controller 的设备称为以 MC-centric 设备。通过 V4L2 device node 完全控制的设备 称为 video-node-centric 设备。

可以通过 ioctl `VIDIOC_QUERYCAP` 检查 `device_caps` field, 如果有 `V4L2_CAP_IO_MC`, 则是 MC-centric 的。

MC-centric 设备需要通过 media controller api 来 configure pipeline。

video-node-centric 设备也可能提供 media controller 和 sub-device interface. 但在这种情况下，media controller 和 sub-device
只是可读的，用来提供信息，所有的 configuration 都由 video node 来下。

### 1.1.2 V4L2 Device Node Naming

V4L2 的 device node 主设备号为 81，次设备号是动态分配的，除非设定 `CONFIG_VIDEO_FIXED_MINOR_RANGES` 来指定范围。

### 1.1.3 Related Devices

### 1.1.4 Multiple Opens

### 1.1.5 Shared Data Streams

V4L2 driver 不支持在多个 app 中读写相同的 data stream.

## 1.2 Querying Capabilities

## 1.3 Application Priority

`VIDIOC_G_PRIORITY`, `VIDIOC_S_PRIORITY`

## 1.4 Video Inputs and Outputs

```c++
struct v4l2_input input;
ioctl(fd, VIDIOC_G_INPUT, &index); // 获取 input index
ioctl(fd, VIDIOC_G_OUTPUT, &index); // 获取 output index
ioctl(fd, VIDIOC_S_INPUT, &index); // 选择 input index
ioctl(fd, VIDIOC_S_OUTPUT, &index); // 选择 output index
ioctl(fd, VIDIOC_ENUMINPUT, &input); // 传入 input.index, 获取第 index 个 input 的信息
ioctl(fd, VIDIOC_ENUMOUTPUT, &output); // 传入 output.index, 获取第 index 个 output 的信息
```

对于 media controller, input/ouput 另外控制，这里只会有一个 input 和 output.

## 1.5 Audio Inputs and Outputs

skip

## 1.6 Tuners and Modulators

skip

## 1.7 Video Standards

skip

## 1.8 Digital Video(DV) Timings

skip

## 1.9 User Controls

有一些使用 v4l2 controls 的例子。

## 1.10 Extended Controls API

参考 `VIDIOC_G_EXT_CTRLS`, `VIDIOC_S_EXT_CTRLS` and `VIDIOC_TRY_EXT_CTRLS`。

创建一个属于相同 control class 的 v4l2_ext_control 数组。

## 1.26 Single- and multi-planar APIs

multi-planar API 调用可以处理所有 single-planar 格式（只要它们以 multi-planar API 结构传递），而 single-planar API 无法处理 multi-planar 格式。

需要区别 single 和 multi-planar 的 API 有：

`VIDIOC_QUERYCAP`, `VIDIOC_G_FMT, VIDIOC_S_FMT, VIDIOC_TRY_FMT`, `VIDIOC_QBUF, VIDIOC_DQBUF, VIDIOC_QUERYBUF`, `VIDIOC_REQBUFS`

## 1.27 Cropping, composing and scaling -- the SELECTION API

## 1.27.2 Selection targets

![](https://xyc-1316422823.cos.ap-shanghai.myqcloud.com/20250730111038.png)

### 1.27.3 Configuration

即使设备不支持 cropping 和 composing，cropping 和 composing target 的 rectangles 也会被定义。在这种情况下，它们的大小和位置将被固定。如果设备不支持 scaling，则 cropping 和 composing rectangles 的大小相同。

#### 1.27.3.1 Configuration of video capture

一般先配置 cropping target, 再配置 composing target。

crop:

- `V4L2_SEL_TGT_CROP_BOUNDS`: crop 的边界范围，对应图中 CROP_BOUNDS
- `V4L2_SEL_TGT_CROP_DEFAULT`: driver 默认的 crop 区域，对应图中 CROP_DEFAULT
- `V4L2_SEL_TGT_CROP`: 设置的 crop 区域，对应图中 CROP_ACTIVE

compose:

- `V4L2_SEL_TGT_COMPOSE_BOUNDS`: compose 的边界，width 和 height 为 VIDIOC_S_FMT 中的 image size
- `V4L2_SEL_TGT_COMPOSE_DEFAULT`: 默认的 compose 区域，一般和 bound 相同。
- `V4L2_SEL_TGT_COMPOSE`: 设置的 compose 区域。
- `V4L2_SEL_TGT_COMPOSE_PADDED`: hw 读写的 compose 区域。

#### 1.27.3.3 Scaling control

通过 crop 和 compose rectangle 的 width 和 height 可以判断出是否有 scaling，如果不相等就是有 scaling。

### 1.27.4 Comparison with old cropping API

使用 selection api 代替旧的 crop api。

## 1.28 Image Cropping, Insertion and Scaling -- the CROP API

// TODO: 被 selection api 代替

## 1.29 Streaming Parameters

可以实现一种 streaming io, 通过 `VIDIOC_G_PARM`, `VIDIOC_S_PARM` 设置参数。
