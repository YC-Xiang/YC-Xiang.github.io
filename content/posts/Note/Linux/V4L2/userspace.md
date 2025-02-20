# 123

根据 determine_valid_ioctls()函数, video rx 设备支持的 ioctl 有:

```c
VIDIOC_QUERYCAP // 要定义ops->vidioc_querycap
VIDIOC_G_PRIORITY
VIDIOC_S_PRIORITY
VIDIOC_G_FREQUENCY // ops->vidioc_g_frequency
VIDIOC_S_FREQUENCY // ops->vidioc_s_frequency
VIDIOC_LOG_STATUS // ops->vidioc_log_status
VIDIOC_DQEVENT // ops->vidioc_subscribe_event
VIDIOC_SUBSCRIBE_EVENT // ops->vidioc_subscribe_event
VIDIOC_UNSUBSCRIBE_EVENT // ops->vidioc_unsubscribe_event
```

在定义了 vdev->ctrl_handler 的情况下:

```c
VIDIOC_QUERYCTRL // ops->vidioc_queryctrl
VIDIOC_QUERY_EXT_CTRL // ops->vidioc_query_ext_ctrl
VIDIOC_G_CTRL // ops->vidioc_g_ctrl || ops->vidioc_g_ext_ctrls
VIDIOC_S_CTRL // ops->vidioc_s_ctrl || ops->vidioc_s_ext_ctrls
VIDIOC_G_EXT_CTRLS // ops->vidioc_g_ext_ctrls
VIDIOC_S_EXT_CTRLS // ops->vidioc_s_ext_ctrls
VIDIOC_TRY_EXT_CTRLS // ops->vidioc_try_ext_ctrls
VIDIOC_QUERYMENU // ops->vidioc_querymenu
```

```c
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
```

# Video for Linux API

## 1.2 Querying Capabilities

查询 v4l2 设备支持的功能, 返回 `struct v4l2_capability` 结构体. 所有 app 程序在 open 后都要执行.

```c
struct v4l2_capability caps;
ioctl(fd, VIDIOC_QUERYCAP, &caps);
```

```c
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

driver: 驱动名称.  
card: 设备名称.  
bus_info: bus 名称, 比如 platform:xxx  
version: kernel 版本.  
capabilities: 物理设备支持的功能. 看着和 device_caps 差不多, 多一个 V4L2_CAP_DEVICE_CAPS?  
device_caps: 设备支持的功能.

## 1.3 Application Priority

## 1.4 Video Inputs and Outputs

```c
struct v4l2_input input;
ioctl(fd, VIDIOC_G_INPUT, &index); // 获取input index
ioctl(fd, VIDIOC_G_OUTPUT, &index); // 获取output index
ioctl(fd, VIDIOC_S_INPUT, &index); // 选择input index
ioctl(fd, VIDIOC_S_OUTPUT, &index); // 选择output index
ioctl(fd, VIDIOC_ENUMINPUT, &input); // 传入input.index, 获取第index个input的信息
ioctl(fd, VIDIOC_ENUMOUTPUT, &output); // 传入output.index, 获取第index个output的信息
```

对于 media controller, input/ouput 另外控制, 这里只会有一个 input 和 output.

## 1.5 Audio Inputs and Outputs

audio 相关

## 1.6 Tuners and Modulators

调谐器, 调制器.

## 1.7 Video Standards

tv 相关

## 1.8 Digital Video (DV) Timings

tv 相关.

## 1.9 User Controls

## 7.14 ioctl VIDIOC_ENUM_FMT

Enumerate image formats.

app 初始化 struct v4l2_fmtdesc 结构体, 传入 type, mbus_code, index, 其他由内核填充并返回.

```c
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

## 7.31 ioctl VIDIOC_G_FMT, VIDIOC_S_FMT, VIDIOC_TRY_FMT

Get or set the data format, try a format.

```c
struct v4l2_format {
	__u32	 type;
	union {
		struct v4l2_pix_format		pix;     /* V4L2_BUF_TYPE_VIDEO_CAPTURE */
		struct v4l2_pix_format_mplane	pix_mp;  /* V4L2_BUF_TYPE_VIDEO_CAPTURE_MPLANE */
		// ... other types format
	} fmt;
};
```

```c
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

width: picture width in pixels.  
height: picture height in pixels. 如果field是V4L2_FIELD_TOP/BOTTOM/ALTERNATE,
则height代表field中的height pixels, 否则是frame中的height pixels.  
pixelformat: picture format in fourcc code.  
field: enum v4l2_field 场的类型.  
bytesperline: 每行的字节数.  
sizeimage: 图像总字节数.  
colorspace: enum v4l2_colorspace.  
priv: app设置为V4L2_PIX_FMT_PRIV_MAGIC, 那么后面的extended fields才会有效.  
flags: 有V4L2_PIX_FMT_FLAG_PREMUL_ALPHA和V4L2_PIX_FMT_FLAG_SET_CSC.  
ycbcr_enc: enum v4l2_ycbcr_encoding.  
hsv_enc: enum v4l2_hsv_encoding.  
quantization: enum v4l2_quantization.  
xfer_func: enum v4l2_xfer_func.

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

**VIDIOC_G_FMT**: 获取当前的 format.

**VIDIOC_S_FMT**: app 设置好所有 field.

**VIDIOC_TRY_FMT**: 和 VIDIOC_S_FMT 类似, 但是 driver 永远不会返回 EBUSY, 不推荐实现.

## 7.46 ioctl VIDIOC_QBUF, VIDIOC_DQBUF

Exchange a buffer with the driver.

## 7.47 ioctl VIDIOC_QUERYBUF

Query the status of a buffer.

填充 struct v4l2_buffer 结构体.

```c
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

## 7.52 ioctl VIDIOC_REQBUFS

Initiate Memory Mapping, User Pointer I/O or DMA buffer I/O.

```c
struct v4l2_requestbuffers {
	__u32			count;
	__u32			type;		/* enum v4l2_buf_type */
	__u32			memory;		/* enum v4l2_memory */
	__u32			capabilities;
	__u8			flags;
	__u8			reserved[3];
};
```

count: app 传入要 request 的 buf 数量, driver 返回实际的 buf 数量.
type: app 传入的 v4l2_buf_type.
memory: usersapce 设置为 V4L2_MEMORY_MMAP/V4L2_MEMORY_DMABUF/V4L2_MEMORY_USERPTR.
capabilities: driver 返回的 capabilities.
flags:

# Media Controller API
