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

## 3.3 Streaming I/O (User Pointers)

## 3.4 Streaming I/O (DMA buffer importing)
