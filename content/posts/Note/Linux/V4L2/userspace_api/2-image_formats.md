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

**8-bit Bayer formats:**

每个采样 8 bits.

V4L2_PIX_FMT_SRGGB8(RGGB)  
V4L2_PIX_FMT_SGRBG8(GRBG)  
V4L2_PIX_FMT_SGBRG8(GBRG)  
V4L2_PIX_FMT_SBGGR8(BA81)

V4L2_PIX_FMT_SBGGR8:

奇数行由 BG 组成，偶数行由 GR 组成。

![](https://xyc-1316422823.cos.ap-shanghai.myqcloud.com/20251013135611.png)

</br>

**10-bit Bayer formats:**

每个采样 10 bits, 保存在 16-bit word 中，6 bit 高位填充为 0.

V4L2_PIX_FMT_SRGGB10 (RG10)  
V4L2_PIX_FMT_SGRBG10 (BA10)  
V4L2_PIX_FMT_SGBRG10 (GB10)  
V4L2_PIX_FMT_SBGGR10 (BG10)  

V4L2_PIX_FMT_SBGGR10:

![](https://xyc-1316422823.cos.ap-shanghai.myqcloud.com/20251013140023.png)

</br>

**10-bit packed Bayer formats:**

每个采样 10 bits, 每 4 个采样合并为 5 bytes。第 5 个 byte 保存前四个 pixel 的低两位。

V4L2_PIX_FMT_SRGGB10P (pRAA)  
V4L2_PIX_FMT_SGRBG10P (pgAA)  
V4L2_PIX_FMT_SGBRG10P (pGAA)  
V4L2_PIX_FMT_SBGGR10P (pBAA)

V4L2_PIX_FMT_SBGGR10P:

![](https://xyc-1316422823.cos.ap-shanghai.myqcloud.com/20251013140917.png)

</br>

**12-bit Bayer formats**

V4L2_PIX_FMT_SRGGB12 (RG12)  
V4L2_PIX_FMT_SGRBG12 (BA12)  
V4L2_PIX_FMT_SGBRG12 (GB12)  
V4L2_PIX_FMT_SBGGR12 (BG12)

每个采样 12bit, 保存在 16-bit word 中，4 bit 高位填充为 0.

</br>

**12-bit packed Bayer formats**

每个采样 12bits，每 2 个 pixels 合并为 3bytes. 第 3 个 byte 保存前两个 pixel 的低 4 位。

V4L2_PIX_FMT_SBGGR12P:

![](https://xyc-1316422823.cos.ap-shanghai.myqcloud.com/20251013141414.png)

</br>

**14-bit Bayer formats**

V4L2_PIX_FMT_SRGGB14 (RG14)  
V4L2_PIX_FMT_SGRBG14 (GR14)  
V4L2_PIX_FMT_SGBRG14 (GB14)  
V4L2_PIX_FMT_SBGGR14 (BG14)

每个采样 14bit, 保存在 16-bit word 中，2 bit 高位填充为 0.

</br>

**14-bit packed Bayer formats**

V4L2_PIX_FMT_SRGGB14P (pREE)  
V4L2_PIX_FMT_SGRBG14P (pgEE)  
V4L2_PIX_FMT_SGBRG14P (pGEE)  
V4L2_PIX_FMT_SBGGR14P (pBEE)

采样 14 位，每 4 个 sample 保存在 7 个 bytes 中。第 4-7 bytes 保存前 4 个 bytes 的低 6 位。

![](https://xyc-1316422823.cos.ap-shanghai.myqcloud.com/20251013141718.png)

</br>

**16-bit Bayer formats**

V4L2_PIX_FMT_SRGGB16 (RG16)  
V4L2_PIX_FMT_SGRBG16 (GR16)  
V4L2_PIX_FMT_SGBRG16 (GB16)  
V4L2_PIX_FMT_SBGGR16 (BYR2)

每个采样 16bits，保存在 16-bit word 中。

## 2.7 YUV Formats

// TODO:

## 2.13 Metadata Formats
