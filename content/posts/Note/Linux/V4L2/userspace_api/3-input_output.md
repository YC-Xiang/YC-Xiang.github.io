---
date: 2025-07-28T11:04:36+08:00
title: "V4L2 Userspace API -- Input/Output"
tags:
  - V4L2
categories:
  - V4L2
---

V4L2 API 定义了几种不同的方法来读取或写入设备。

## 3.1 Read/Write

driver capabilities 需要置起 V4L2_CAP_READWRITE。

不会传递 frame counter 和 timestamp, 最简单的 I/O 方法。

## 3.2 Streaming I/O (Memory Mapping)

driver capabilities 需要置起 V4L2_CAP_STREAMING。

app 需要在 ioctl VIDIOC_REQBUFS 中把 memory type 设置为 V4L2_MEMORY_MMAP。

文档中有使用 mmap() 的例子。

## 3.3 Streaming I/O (User Pointers)

driver capabilities 需要置起 V4L2_CAP_STREAMING。

app 需要在 ioctl VIDIOC_REQBUFS 中把 memory type 设置为 V4L2_MEMORY_USERPTR。

## 3.4 Streaming I/O (DMA buffer importing)

driver capabilities 需要置起 V4L2_CAP_STREAMING。

app 需要在 ioctl VIDIOC_REQBUFS 中把 memory type 设置为 V4L2_MEMORY_DMABUF。

## 3.5 Buffers

streaming 的时候不能修改 control 和 format，需要 streamoff 之后并且归还所有 allocated buffers 才能重新设置。

``txt
VIDIOC_STREAMOFF
VIDIOC_REQBUFS(0)
VIDIOC_S_EXT_CTRLS
VIDIOC_S_FMT
VIDIOC_REQBUFS(n)
VIDIOC_QBUF
VIDIOC_STREAMON

```
