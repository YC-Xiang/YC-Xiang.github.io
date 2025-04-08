---
date: 2025-03-26T9:52:54+08:00
title: "V4L2 -- ioctl"
tags:
  - V4L2
categories:
  - V4L2
---

# ioctls

根据 determine_valid_ioctls() 函数，video rx 设备支持的 ioctl 有：

```c++
VIDIOC_QUERYCAP // 要定义 ops->vidioc_querycap
VIDIOC_G_PRIORITY
VIDIOC_S_PRIORITY
VIDIOC_G_FREQUENCY // ops->vidioc_g_frequency
VIDIOC_S_FREQUENCY // ops->vidioc_s_frequency
VIDIOC_LOG_STATUS // ops->vidioc_log_status
VIDIOC_DQEVENT // ops->vidioc_subscribe_event
VIDIOC_SUBSCRIBE_EVENT // ops->vidioc_subscribe_event
VIDIOC_UNSUBSCRIBE_EVENT // ops->vidioc_unsubscribe_event
```

在定义了 vdev->ctrl_handler 的情况下：

```c++
VIDIOC_QUERYCTRL // ops->vidioc_queryctrl
VIDIOC_QUERY_EXT_CTRL // ops->vidioc_query_ext_ctrl
VIDIOC_G_CTRL // ops->vidioc_g_ctrl || ops->vidioc_g_ext_ctrls
VIDIOC_S_CTRL // ops->vidioc_s_ctrl || ops->vidioc_s_ext_ctrls
VIDIOC_G_EXT_CTRLS // ops->vidioc_g_ext_ctrls
VIDIOC_S_EXT_CTRLS // ops->vidioc_s_ext_ctrls
VIDIOC_TRY_EXT_CTRLS // ops->vidioc_try_ext_ctrls
VIDIOC_QUERYMENU // ops->vidioc_querymenu
```

```c++
VIDIOC_ENUM_FMT // ops->vidioc_enum_fmt_vid_cap || ops->vidioc_enum_fmt_vid_overlay
VIDIOC_G_FMT // vidioc_g_fmt_vid_cap || vidioc_g_fmt_vid_cap_mplane || vidioc_g_fmt_vid_overlay
VIDIOC_S_FMT // 同上
VIDIOC_TRY_FMT // 同上
VIDIOC_OVERLAY
VIDIOC_G_FBUF
VIDIOC_S_FBUF
VIDIOC_G_JPEGCOMP
VIDIOC_S_JPEGCOMP
VIDIOC_G_ENC_INDEX
VIDIOC_ENCODER_CMD
VIDIOC_TRY_ENCODER_CMD
VIDIOC_DECODER_CMD
VIDIOC_TRY_DECODER_CMD
VIDIOC_ENUM_FRAMESIZES
VIDIOC_ENUM_FRAMEINTERVALS
VIDIOC_G_CROP // vidioc_g_selection
VIDIOC_CROPCAP // // vidioc_g_selection
VIDIOC_S_CROP // vidioc_s_selection
VIDIOC_G_SELECTION
VIDIOC_S_SELECTION

VIDIOC_REQBUFS
VIDIOC_QUERYBUF
VIDIOC_QBUF
VIDIOC_EXPBUF
VIDIOC_DQBUF
VIDIOC_CREATE_BUFS
VIDIOC_PREPARE_BUF
VIDIOC_STREAMON
VIDIOC_STREAMOFF

VIDIOC_ENUMSTD
VIDIOC_S_STD
VIDIOC_G_STD
VIDIOC_QUERYSTD
```

# Video for Linux API

```c++
static void rtsisp_hw_free_vreg(struct rtsisp_dev_info *dev_info)
{
	struct rtsisp_hw *hw = dev_info->hw;
	struct rtsisp_dev_hw_info *hw_info = &hw->hw_info[dev_info->dev_id];

	if (hw_info->vregs) {
		vfree(hw_info->vregs);
		hw_info->vregs = NULL;
	}

	if (hw->dev_num == 1 && hw->vregs_bitmap) {
		bitmap_free(hw->vregs_bitmap);
		hw->vregs_bitmap = NULL;
	}
}
```

## 1.2 Querying Capabilities

查询 v4l2 设备支持的功能，返回 `struct v4l2_capability` 结构体。所有 app 程序在 open 后都要执行。

```c++
struct v4l2_capability caps;
ioctl(fd, VIDIOC_QUERYCAP, &caps);
```

```c++
struct v4l2_capability {
	__u8	driver[16];
	__u8	card[32];
	__u8	bus_info[32];
	__u32   version;
	__u32	capabilities;
	__u32	device_caps;
	__u32	reserved[3];
};
```

driver: 驱动名称。
card: 设备名称。
bus_info: bus 名称，比如 platform:xxx  
version: kernel 版本。
capabilities: 物理设备支持的功能。看着和 device_caps 差不多，多一个 V4L2_CAP_DEVICE_CAPS?  
device_caps: 设备支持的功能。

## 1.3 Application Priority

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

audio 相关

## 1.6 Tuners and Modulators

调谐器，调制器。

## 1.7 Video Standards

tv 相关

## 1.8 Digital Video (DV) Timings

tv 相关。

## 1.9 User Controls

## 7.14 ioctl VIDIOC_ENUM_FMT

Enumerate image formats.

app 初始化 struct v4l2_fmtdesc 结构体，传入 type, mbus_code, index, 其他由内核填充并返回。

```c++
struct v4l2_fmtdesc {
	__u32		    index;             /* Format number      */
	__u32		    type;              /* enum v4l2_buf_type */
	__u32               flags;
	__u8		    description[32];   /* Description string */
	__u32		    pixelformat;       /* Format fourcc      */
	__u32		    mbus_code;		/* Media bus code    */
	__u32		    reserved[3];
};
```

index: app 要查询的 format index.  
type: capture 设备传 V4L2_BUF_TYPE_VIDEO_CAPTURE/V4L2_BUF_TYPE_VIDEO_CAPTURE_MPLANE.  
flags:  
description: drvier 返回的 format 的 description.  
pixelformat: driver 返回的 format(fourcc code).  
mbus_code: app 传入的 mbus_code, 可以来索引不同的 mbus 的 format.

## 7.25 ioctl VIDIOC_G_CTRL, VIDIOC_S_CTRL

get or set the value of control.

app 传入标准的 v4l2_ctrl id, driver 返回 value.

v4l2_ctrl id 参考 `v4l2-controls.h`.

```c++
struct v4l2_control {
	__u32		     id;
	__s32		     value;
};
```

## 7.29 ioctl VIDIOC_G_EXT_CTRLS, VIDIOC_S_EXT_CTRLS, VIDIOC_TRY_EXT_CTRLS

Get or set the value of several controls, try control values.

```c++
int ioctl(int fd, VIDIOC_G_EXT_CTRLS, struct v4l2_ext_controls *argp);
int ioctl(int fd, VIDIOC_S_EXT_CTRLS, struct v4l2_ext_controls *argp);
int ioctl(int fd, VIDIOC_TRY_EXT_CTRLS, struct v4l2_ext_controls *argp);
```

app 传入 count, which, controls, reserved, 并且初始化好所有的 v4l2_ext_control.

```c++
struct v4l2_ext_controls {
	union {
#ifndef __KERNEL__
		__u32 ctrl_class;
#endif
		__u32 which;
	};
	__u32 count;
	__u32 error_idx;
	__s32 request_fd;
	__u32 reserved[1];
	struct v4l2_ext_control *controls;
};
```

ctrl_class: 已废弃，使用 which.  
which: V4L2_CTRL_WHICH_CUR_VAL/V4L2_CTRL_WHICH_DEF_VAL/V4L2_CTRL_WHICH_REQUEST_VAL.  
count: controls 数组的数量。
error_idx: driver 返回发生错误的 control index.  
request_fd:  
reserved[1]: 设置为 0.  
controls: control 数组。

```c++
struct v4l2_ext_control {
	__u32 id;
	__u32 size;
	__u32 reserved2[1];
	union {
		__s32 value;
		__s64 value64;
		char __user *string;
		__u8 __user *p_u8;
		__u16 __user *p_u16;
		__u32 __user *p_u32;
		__s32 __user *p_s32;
		__s64 __user *p_s64;
		// ...
		void __user *ptr;
	};
}
```

id: control id.  
size:  通常为 0, 对于 pointer control 要设置为发送或接收的 payload 大小。
reserved2: 设置为 0.  
value: 要设置的 control value.

## 7.31 ioctl VIDIOC_G_FMT, VIDIOC_S_FMT, VIDIOC_TRY_FMT

Get or set the data format, try a format.

```c++
struct v4l2_format {
	__u32	 type;
	union {
		struct v4l2_pix_format		pix;     /* V4L2_BUF_TYPE_VIDEO_CAPTURE */
		struct v4l2_pix_format_mplane	pix_mp;  /* V4L2_BUF_TYPE_VIDEO_CAPTURE_MPLANE */
		// ... other types format
	} fmt;
};
```

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
		__u32			ycbcr_enc;
		__u32			hsv_enc;
	};
	__u32			quantization;	/* enum v4l2_quantization */
	__u32			xfer_func;	/* enum v4l2_xfer_func */
};
```

width: picture width in pixels.  
height: picture height in pixels. 如果 field 是 V4L2_FIELD_TOP/BOTTOM/ALTERNATE,
则 height 代表 field 中的 height pixels, 否则是 frame 中的 height pixels.  
pixelformat: picture format in fourcc code.  
field: enum v4l2_field 场的类型。
bytesperline: 每行的字节数。
sizeimage: 图像总字节数。
colorspace: enum v4l2_colorspace.  
priv: app 设置为 V4L2_PIX_FMT_PRIV_MAGIC, 那么后面的 extended fields 才会有效。
flags: 有 V4L2_PIX_FMT_FLAG_PREMUL_ALPHA 和 V4L2_PIX_FMT_FLAG_SET_CSC.  
ycbcr_enc: enum v4l2_ycbcr_encoding.  
hsv_enc: enum v4l2_hsv_encoding.  
quantization: enum v4l2_quantization.  
xfer_func: enum v4l2_xfer_func.

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
}
```

**VIDIOC_G_FMT**: 获取当前的 format.

**VIDIOC_S_FMT**: app 设置好所有 field.

**VIDIOC_TRY_FMT**: 和 VIDIOC_S_FMT 类似，但是 driver 永远不会返回 EBUSY, 不推荐实现。

## 7.46 ioctl VIDIOC_QBUF, VIDIOC_DQBUF

Exchange a buffer with the driver.

## 7.47 ioctl VIDIOC_QUERYBUF

Query the status of a buffer.

填充 struct v4l2_buffer 结构体。

```c++
struct v4l2_buffer {
	__u32			index;
	__u32			type; // enum v4l2_buf_type
	__u32			bytesused;
	__u32			flags;
	__u32			field;
#ifdef __KERNEL__
	struct __kernel_v4l2_timeval timestamp;
#else
	struct timeval		timestamp;
#endif
	struct v4l2_timecode	timecode;
	__u32			sequence;

	__u32			memory;
	union {
		__u32           offset;
		unsigned long   userptr;
		struct v4l2_plane *planes;
		__s32		fd;
	} m;
	__u32			length;
	__u32			reserved2;
	union {
		__s32		request_fd;
		__u32		reserved;
	};
};
```

index: buffer index.

## 7.49 ioctls VIDIOC_QUERYCTRL, VIDIOC_QUERY_EXT_CTRL and VIDIOC_QUERYMENU

Enumerate controls and menu control items.

```c++
int ioctl(int fd, VIDIOC_QUERYCTRL, struct v4l2_queryctrl *argp);
int ioctl(int fd, VIDIOC_QUERY_EXT_CTRL, struct v4l2_query_ext_ctrl *argp);
int ioctl(int fd, VIDIOC_QUERYMENU, struct v4l2_querymenu *argp);
```

```c++
struct v4l2_queryctrl {
	__u32		     id;
	__u32		     type;	/* enum v4l2_ctrl_type */
	__u8		     name[32];	/* Whatever */
	__s32		     minimum;	/* Note signedness */
	__s32		     maximum;
	__s32		     step;
	__s32		     default_value;
	__u32                flags;
	__u32		     reserved[2];
};
```

id: control id. app 设置。
type: enum v4l2_ctrl_type. driver 返回。
name:
minimum:
maximum:
step:
default_value:
flags: control flags. driver 返回。

```c++
struct v4l2_querymenu {
	__u32		id;
	__u32		index;
	union {
		__u8	name[32];	/* Whatever */
		__s64	value;
	};
	__u32		reserved;
}
```

id: control id, app 设定。
index: 该 menu 选项中的第 index 个，app 设定。
name: driver 返回的 menu control string, V4L2_CTRL_TYPE_MENU 类型 control 适用。
value: driver 返回的 menu control value, V4L2_CTRL_TYPE_INTEGER_MENU 类型 control 适用。

## 7.52 ioctl VIDIOC_REQBUFS

Initiate Memory Mapping, User Pointer I/O or DMA buffer I/O.

```c++
struct v4l2_requestbuffers {
	__u32			count;
	__u32			type;		/* enum v4l2_buf_type */
	__u32			memory;		/* enum v4l2_memory */
	__u32			capabilities;
	__u8			flags;
	__u8			reserved[3];
};
```

count: app 传入要 request 的 buf 数量，driver 返回实际的 buf 数量。
type: app 传入的 v4l2_buf_type.
memory: usersapce 设置为 V4L2_MEMORY_MMAP/V4L2_MEMORY_DMABUF/V4L2_MEMORY_USERPTR.
capabilities: driver 返回的 capabilities.
flags:

## 7.57 ioctl VIDIOC_SUBDEV_ENUM_FRAME_SIZE

```c++
struct v4l2_subdev_frame_size_enum {
	__u32 index;
	__u32 pad;
	__u32 code;
	__u32 min_width;
	__u32 max_width;
	__u32 min_height;
	__u32 max_height;
	__u32 which;
	__u32 stream;
	__u32 reserved[7];
};
```

app:

`index`: 要 get 的 frame size index.  
`pad`: pad id.  
`code`: bus format code.  
`which`: 见 7.60 的 which.

kernel:

`min_width`, `max_width`, `min_height`, `max_height`: disCrete frame size
的 min 和 max 值相同。

## 7.58 ioctl VIDIOC_SUBDEV_ENUM_MBUS_CODE

Enumerate media bus formats.

```c++
struct v4l2_subdev_mbus_code_enum {
	__u32 pad;
	__u32 index;
	__u32 code;
	__u32 which;
	__u32 reserved[8];
}
```

app:

`pad`: pad id.  
`index`: 要 get 的 mbus code index.  
`which`: 见 7.60 的 which.

kernel:

`code`: 返回 bus format, MEDIA_BUS_FMT_XXX.

## 7.60 ioctl VIDIOC_SUBDEV_G_FMT, VIDIOC_SUBDEV_S_FMT

Get or set the data format on a subdev pad.

```c++
struct v4l2_subdev_format {
	__u32 which;
	__u32 pad;
	struct v4l2_mbus_framefmt format;
	__u32 reserved[8];
};


struct v4l2_mbus_framefmt {
	__u32			width;
	__u32			height;
	__u32			code;
	__u32			field;
	__u32			colorspace;
	union {
		/* enum v4l2_ycbcr_encoding */
		__u16			ycbcr_enc;
		/* enum v4l2_hsv_encoding */
		__u16			hsv_enc;
	};
	__u16			quantization;
	__u16			xfer_func;
	__u16			flags;
	__u16			reserved[10];
};
```

**VIDIOC_SUBDEV_S_FMT：**

app：

`which`: Format to modified, 传入 enum v4l2_subdev_format_whence,
有 V4L2_SUBDEV_FORMAT_TRY 和 V4L2_SUBDEV_FORMAT_ACTIVE 两个值。前者用来 try format,
后者 apply to hardware.  
`pad`: pad id.  
`format`: 要设置的 struct v4l2_mbus_framefmt.

## 7.61 ioctl VIDIOC_SUBDEV_G_FRAME_INTERVAL, VIDIOC_SUBDEV_S_FRAME_INTERVAL

Get or set the frame interval on a subdev pad

```c++
struct v4l2_subdev_frame_interval {
	__u32 pad;
	struct v4l2_fract interval;
	__u32 stream;
	__u32 reserved[8];
};

struct v4l2_fract {
	__u32   numerator;
	__u32   denominator;
};
```

app:

`pad`: pad number.  
`interval`: 要设置的 struct v4l2_fract.  
`stream`: stream identifier.

## 7.66 ioctl VIDIOC_SUBSCRIBE_EVENT, VIDIOC_UNSUBSCRIBE_EVENT

Subscribe or unsubscribe event

```c++
struct v4l2_event_subscription {
	__u32				type;
	__u32				id;
	__u32				flags;
	__u32				reserved[5];
};
```

type: 要订阅的 event 类型，V4L2_EVENT_XXX.  
id: 根据 type 不同类型，有的需要 event source id.
flags: V4L2_EVENT_SUB_FL_XXX.

# Media Controller API
