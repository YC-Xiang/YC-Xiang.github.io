---
date: 2024-10-10T11:05:54+08:00
title: "Drm -- EDID"
tags:
  - DRM
categories:
  - DRM
draft: true
---

# Introduction

EDID: Extended Display Identification Data.

EDID store some fixed information of the display，such as native resolution，pixel format, etc. Thus the host can fetch the information without configuring them manually.

EDID is transmitted from the device to your display adapter using a Dynamic Data Channel (DDC). DDC is a standard for communication between a monitor and a display adapter. The DDC uses a standard serial signaling scheme known as the I2C bus

# EDID format

![](https://xyc-1316422823.cos.ap-shanghai.myqcloud.com/20241011112935.png)

drm_edid.h 中对应的结构体为：

```c
struct edid {
	u8 header[8];
	/* Vendor & product info */
	union {
		struct drm_edid_product_id product_id;
		struct {
			u8 mfg_id[2];
			u8 prod_code[2];
			u32 serial; /* FIXME: byte order */
			u8 mfg_week;
			u8 mfg_year;
		} __packed;
	} __packed;
	/* EDID version */
	u8 version;
	u8 revision;
	/* Display info: */
	u8 input;
	u8 width_cm;
	u8 height_cm;
	u8 gamma;
	u8 features;
	/* Color characteristics */
	u8 red_green_lo;
	u8 blue_white_lo;
	u8 red_x;
	u8 red_y;
	u8 green_x;
	u8 green_y;
	u8 blue_x;
	u8 blue_y;
	u8 white_x;
	u8 white_y;
	/* Est. timings and mfg rsvd timings*/
	struct est_timings established_timings;
	/* Standard timings 1-8*/
	struct std_timing standard_timings[8];
	/* Detailing timings 1-4 */
	struct detailed_timing detailed_timings[4];
	/* Number of 128 byte ext. blocks */
	u8 extensions;
	/* Checksum */
	u8 checksum;
} __packed;
```

## Header

0x00, 0xff, 0xff, 0xff, 0xff, 0xff, 0xff, 0x00.

## Vendor & Product ID

![](https://xyc-1316422823.cos.ap-shanghai.myqcloud.com/20241015152641.png)

## EDID Structure Version & Revision

保存对应 edid spec 的版本号

![](https://xyc-1316422823.cos.ap-shanghai.myqcloud.com/20241015153059.png)

## Basic Display Parameters and Features

![](https://xyc-1316422823.cos.ap-shanghai.myqcloud.com/20241015153153.png)

### 0x14h

![](https://xyc-1316422823.cos.ap-shanghai.myqcloud.com/20241015153609.png)

###

# Reference

https://glenwing.github.io/docs/VESA-EEDID-A2.pdf
