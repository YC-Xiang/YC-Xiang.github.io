---
date: 2024-03-29T15:38:09+08:00
title: 'USB_Reset_Suspend_Resume'
tags:
- USB
categories:
- Notes
---

# Reset

整个reset过程，host需要拉低D+的时间\(T_{DRST}\) 至少 > 10ms。

![](https://xyc-1316422823.cos.ap-shanghai.myqcloud.com/20240329105326.png)

High-speed capable hubs and devices在reset过程中还会执行一套reset protocol：

![](https://xyc-1316422823.cos.ap-shanghai.myqcloud.com/20240329104041.png)

1. Hub确认接的设备不是Low-speed, 如果是low-speed，不会有下面的reset握手协议。
2. Hub把D+拉低，进入SE0状态。
3. Device检测到SE0:
   1. 如果device是从**suspend**状态进入reset，那么device在不少于2.5us后进行high-speed detection handshake。
   2. 如果device是从**non-suspend full-speed**状态进入reset，那么device需要在2.5us~3ms之间进入handshake。
   3. 如果device是从**non-suspend high-speed**状态进入reset，那么device必须在3ms~3.125ms之间恢复full-spped，通过断开高速终端电阻以及重新接上D+的上拉电阻。接着在100us-875us内开始检测SE0状态，进入handshake。
4. Device往D-灌电流，形成一个800mv左右的ChripK信号, 该信号保持1~7ms。
5. Hub检测到ChripK信号后，100us内需要发送三对KJ(k-J-K-J-K-J)。每个单独的J或K都要保持40\~60us。KJ对打完之后，在500us内断开D+上拉电阻，使能高速终端电阻，进入高速状态。

# Suspend

Device有两种进入suspend的方式：

1. 如果连续3帧没有收到SOF包(1ms一帧)，device会进入suspend状态。
![](https://xyc-1316422823.cos.ap-shanghai.myqcloud.com/20240329155750.png)

2. host driver发送SET_PORT_FEATURE(PORT_SUSPEND)给hub，hub再发送给指定的device。

# Resume

![](https://xyc-1316422823.cos.ap-shanghai.myqcloud.com/20240329163231.png)

从suspend状态resume起来需要保持ChripK状态20ms以上，也有两种方式：

1. Device remote wakeup，device drive K-state 1~15ms。Hub会在1ms内，接管D+, D-,继续产生resume信号直到20ms。

2. host发送CLEAR_PORT_FEATURE( PORT_SUSPSEND)给hub，hub drive K-state 20ms。

# Reference

[Reset, Suspend, and Resume Commands](https://developerhelp.microchip.com/xwiki/bin/view/applications/usb/how-it-works/reset-suspend-resume/)

[USB全速设备的挂起、唤醒Resume](https://www.usbzh.com/article/detail-1214.html)

[USB2.0设备从全速模式到高速模式的识别过程及速率协商](https://www.usbzh.com/article/detail-672.html)

USB spec 7.1.7.5~7.1.7.7, 11.9
