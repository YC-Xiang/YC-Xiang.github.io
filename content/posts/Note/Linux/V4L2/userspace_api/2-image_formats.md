---
date: 2025-07-28T11:04:36+08:00
title: "V4L2 Userspace API -- Image Formats"
tags:
  - V4L2
categories:
  - V4L2
---

struct `v4l2_pix_format` 和 `v4l2_pix_format_mplane` 定义了 image in memory 的 format 和 layout。

## 2.1 Single-planar format structure

```c++
struct v4l2_pix_format {
	__u32			width;
	__u32			height;
	__u32			pixelformat;
	__u32			field;		/* enum v4l2_field */
	__u32			bytesperline;	/* for padding, zero if unused */
	__u32			sizeimage;
	__u32			colorspace;	/* enum v4l2_colorspace */
	__u32			priv;		/* private data, depends on pixelformat */
	__u32			flags;		/* format flags (V4L2_PIX_FMT_FLAG_*) */
	union {
		/* enum v4l2_ycbcr_encoding */
		__u32			ycbcr_enc;
		/* enum v4l2_hsv_encoding */
		__u32			hsv_enc;
	};
	__u32			quantization;	/* enum v4l2_quantization */
	__u32			xfer_func;	/* enum v4l2_xfer_func */
};
```

## 2.2 Multi-planar format structures

```c++
struct v4l2_plane_pix_format {
	__u32		sizeimage;
	__u32		bytesperline;
}
```

```c++
struct v4l2_pix_format_mplane {
	__u32				width;
	__u32				height;
	__u32				pixelformat;
	__u32				field;
	__u32				colorspace;

	struct v4l2_plane_pix_format	plane_fmt[VIDEO_MAX_PLANES];
	__u8				num_planes;
	__u8				flags;
	 union {
		__u8				ycbcr_enc;
		__u8				hsv_enc;
	};
	__u8				quantization;
	__u8				xfer_func;
	__u8				reserved[7];
} __attribute__ ((packed));
```

## 2.6 Raw Bayer Formats

// TODO:

## 2.7 YUV Formats

// TODO:

## 2.13 Metadata Formats
