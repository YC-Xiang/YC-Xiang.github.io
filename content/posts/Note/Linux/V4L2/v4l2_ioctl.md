---
date: 2025-06-19T15:15:36+08:00
title: "V4L2 -- ioctls"
tags:
  - V4L2
categories:
  - V4L2
---

https://docs.kernel.org/userspace-api/media/index.html

# Video for Linux API

# 7. Function reference

## 7.3 ioctl VIDIOC_CREATE_BUFS

Create buffers for **Memory Mapped** or **User Pointer** or **DMA Buffer I/O**.

可以用作 ioctl VIDIOC_REQBUFS 的替代。

```c++
struct v4l2_create_buffers {
	__u32			index;
	__u32			count;
	__u32			memory;
	struct v4l2_format	format;
	__u32			capabilities;
	__u32			flags;
	__u32			max_num_buffers;
	__u32			reserved[5];
};
```

app:

`count`: 要创建的 buffer 的数量。  
`memory`: 要创建的 buffer 的内存类型。enum v4l2_memory.  
`format`: 要创建的 buffer 的格式。通常需要先通过 VIDIOC_TRY_FMT 或者 VIDIOC_G_FMT 获取。  
`flags`: 要创建的 buffer 的 flags.  

kernel:

`index`: 返回创建的 buffers 中第一个 buffer 的 index。
`capabilities`: 返回创建的 buffers 的 capabilities.
`max_num_buffers`: 返回创建的 buffers 的最大数量。

## 7.4 ioctl VIDIOC_CROPCAP

Information about the video cropping and scaling abilities.

获取 driver 支持的可裁剪的区域大小。

```c++
struct v4l2_cropcap {
	__u32			type;
	struct v4l2_rect        bounds;
	struct v4l2_rect        defrect;
	struct v4l2_fract       pixelaspect;
};
```

app:

`type`: enum v4l2_buf_type.

kernel:

`bounds`: 返回 driver 支持的可裁剪的区域大小。  
`defrect`: 返回 driver 支持的默认的裁剪区域大小。  
`pixelaspect`: 返回 driver 支持的像素的宽高比，默认是 1:1，Other common values are 54/59 for PAL and SECAM, 11/10 for NTSC.

## 7.6 ioctl VIDIOC_DBG_G_REGISTER, VIDIOC_DBG_S_REGISTER

Read or write hardware registers

skip

## 7.7 ioctl VIDIOC_DECODER_CMD, VIDIOC_TRY_DECODER_CMD

Execute an decoder command

skip

## 7.8 ioctl VIDIOC_DQEVENT

Dequeue event

## 7.10 ioctl VIDIOC_ENCODER_CMD, VIDIOC_TRY_ENCODER_CMD

## 7.14 ioctl VIDIOC_ENUM_FMT

Enumerate image formats.

获取支持的 pixel format.

```c++
struct v4l2_fmtdesc {
	__u32		    index;
	__u32		    type;
	__u32               flags;
	__u8		    description[32];
	__u32		    pixelformat;
	__u32		    mbus_code;
};
```

app:

`index`: app 要查询的 format index.  
`type`: capture 设备传 V4L2_BUF_TYPE_VIDEO_CAPTURE/V4L2_BUF_TYPE_VIDEO_CAPTURE_MPLANE.  
`mbus_code`: app 传入的 mbus_code, MEDIA_BUS_FMT_XXX, 可以用来限制枚举的 formats, 只有 driver 置起 V4L2_CAP_IO_MC flag 才有效，否则为 0。

kernel:

`description`: drvier 返回的 format 的 description.  
`pixelformat`: driver 返回的 format(fourcc code).  

## 7.15 ioctl VIDIOC_ENUM_FRAMESIZES

Enumerate frame sizes

```c++
struct v4l2_frmsizeenum {
	__u32			index;
	__u32			pixel_format;
	__u32			type;

	union {
		struct v4l2_frmsize_discrete	discrete;
		struct v4l2_frmsize_stepwise	stepwise;
	};
};
```

app:

`index`: app 要查询的 frame size index.  
`pixel_format`: app 要查询的 pixel format.一般是 7.14 返回的 pixelformat.  

kernel:

`type`: frame size type, enum v4l2_frmsizetypes, 有 discrete, continuous, stepwise.  
`discrete`: 返回 discrete frame size, 是固定的长和宽。  
`stepwise`: 返回 stepwise frame size, 是可变的，有最小值，最大值，步长，continuous 类型步长为 1。

## 7.16 ioctl VIDIOC_ENUM_FRAMEINTERVALS

Enumerate frame intervals

```c++
struct v4l2_frmivalenum {
	__u32 index;
	__u32 pixel_format;
	__u32 width;
	__u32 height;
	__u32 type;

	union {
		struct v4l2_fract discrete;
		struct v4l2_frmival_stepwise stepwise;
	};
}
```

app:

`index`: app 要查询的 frame interval index.  
`pixel_format`: app 要查询的 pixel format，一般是 7.14 返回的 pixelformat.  
`width`: app 要查询的 width, 一般是 7.15 返回的 width.  
`height`: app 要查询的 height, 一般是 7.15 返回的 height.

kernel:

`type`: frame interval type, enum v4l2_frmivaltypes, 有 discrete, continuous, stepwise.  
`discrete`: 返回 discrete frame interval.  
`stepwise`: 返回 stepwise frame interval.

## 7.18 ioctl VIDIOC_ENUMINPUT

Enumerate video inputs

```c++
struct v4l2_input {
	__u32	     index;
	__u8	     name[32];
	__u32	     type;
	__u32	     audioset;
	__u32        tuner;
	v4l2_std_id  std;
	__u32	     status;
	__u32	     capabilities;
};
```

app:

`index`: app 要查询的 input index.

kernel:

`name`: driver 返回的 input name.  
`type`: driver 返回的 input type. V4L2_INPUT_TYPE_TUNER/CAMERA/TOUCH.  

## 7.19 ioctl VIDIOC_ENUMOUTPUT

## 7.20 ioctl VIDIOC_ENUMSTD, VIDIOC_SUBDEV_ENUMSTD

## 7.21 ioctl VIDIOC_EXPBUF

Export a buffer as a DMABUF file descriptor

dma buf export

// TODO:

## 7.24 ioctl VIDIOC_G_CROP, VIDIOC_S_CROP

Get or set the current cropping rectangle

```c++
struct v4l2_crop {
	__u32			type;
	struct v4l2_rect	c;
};
```

**VIDIOC_G_CROP**: 获取当前的 cropping rectangle.

app:

`type`: enum v4l2_buf_type.

kernel:

`c`: 返回当前的 cropping rectangle.

**VIDIOC_S_CROP**: 设置当前的 cropping rectangle.

app:

`type`: enum v4l2_buf_type.

`c`: 设置当前的 cropping rectangle.

## 7.25 ioctl VIDIOC_G_CTRL, VIDIOC_S_CTRL

get or set the value of control.

v4l2_ctrl id 参考 `v4l2-controls.h`.

```c++
struct v4l2_control {
	__u32		     id;
	__s32		     value;
};
```

**VIDIOC_G_CTRL**: 获取 control 的 value.

app:

`id`: v4l2_ctrl id.

kernel:

`value`: driver 返回 value.

</br>

**VIDIOC_S_CTRL**: 设置 control 的 value.

app:

`id`: v4l2_ctrl id.

`value`: 要设置的 value.

kernel:

## 7.29 ioctl VIDIOC_G_EXT_CTRLS, VIDIOC_S_EXT_CTRLS, VIDIOC_TRY_EXT_CTRLS

Get or set the value of several controls, try control values.

app 传入 count, which, controls, reserved, 并且初始化好所有的 v4l2_ext_control.

```c++
struct v4l2_ext_controls {
	__u32 which;
	__u32 count;
	__u32 error_idx;
	__s32 request_fd;
	struct v4l2_ext_control *controls;
};
```

which: V4L2_CTRL_WHICH_CUR_VAL/V4L2_CTRL_WHICH_DEF_VAL/V4L2_CTRL_WHICH_REQUEST_VAL.  
count: controls 数组的数量。
error_idx: driver 返回发生错误的 control index.  
request_fd:  
controls: control 数组。

```c++
struct v4l2_ext_control {
	__u32 id;
	__u32 size;
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
value: 要设置的 control value.

## 7.31 ioctl VIDIOC_G_FMT, VIDIOC_S_FMT, VIDIOC_TRY_FMT

Get or set the data format, try a format.

```c++
struct v4l2_format {
	__u32	 type;
	union {
		struct v4l2_pix_format		pix;
		struct v4l2_pix_format_mplane	pix_mp;
	} fmt;
};
```

app:

`type`: enum v4l2_buf_type.

kernel:

`fmt`: driver 返回的 format.

```c++
struct v4l2_pix_format {
	__u32			width;
	__u32			height;
	__u32			pixelformat;
	__u32			field;
	__u32			bytesperline;
	__u32			sizeimage;
	__u32			colorspace;
	__u32			priv;
	__u32			flags;
	union {
		__u32			ycbcr_enc;
		__u32			hsv_enc;
	};
	__u32			quantization;
	__u32			xfer_func;
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
}
```

plane_fmt: 每个平面具体的信息。  
num_planes: 平面数量。

```c++
struct v4l2_plane_pix_format {
	__u32		sizeimage;
	__u32		bytesperline;
}
```

sizeimage: 平面最大包含的字节数。  
bytesperline: 每个平面一行的字节数。

**VIDIOC_G_FMT**: 获取当前的 format.

**VIDIOC_S_FMT**: app 设置好所有 field.

**VIDIOC_TRY_FMT**: 和 VIDIOC_S_FMT 类似，但是不会改变 driver state, 不推荐实现。

## 7.37 ioctl VIDIOC_G_PARM, VIDIOC_S_PARM

Get or set streaming parameters

stream api???

```c++
struct v4l2_streamparm {
	__u32 type;
	union {
		struct v4l2_captureparm capture;
		struct v4l2_outputparm output;
		__u8 raw_data[200];
	} parm;
}
```

app:

`type`: enum v4l2_buf_type.

kernel:

`parm`: driver 返回的 streamparm.

## 7.39 ioctl VIDIOC_G_SELECTION, VIDIOC_S_SELECTION

// TODO:

## 7.45 ioctl VIDIOC_PREPARE_BUF

// TODO:

## 7.46 ioctl VIDIOC_QBUF, VIDIOC_DQBUF

Exchange a buffer with the driver.

## 7.47 ioctl VIDIOC_QUERYBUF

Query the status of a buffer.

```c++
struct v4l2_buffer {
	__u32			index;
	__u32			type;
	__u32			bytesused;
	__u32			flags;
	__u32			field;
	struct __kernel_v4l2_timeval timestamp;
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
	union {
		__s32		request_fd;
		__u32		reserved;
	};
};
```

app:

`type`: enum v4l2_buf_type.
`index`: buffer index.
`planes`: 指向用户层 struct v4l2_plane 数组。  
`length`: 对于 multi planes, 需要指定为 v4l2_plane 数组大小。

kernel:

`memory`: enum v4l2_memory.  
`byteused`: multi plane 为 0.  
`bytesused`: 已经使用的字节数。

## 7.48 ioctl VIDIOC_QUERYCAP

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

## 7.55 ioctl VIDIOC_STREAMON, VIDIOC_STREAMOFF

## 7.56 ioctl VIDIOC_SUBDEV_ENUM_FRAME_INTERVAL

Enumerate frame intervals

```c++
struct v4l2_subdev_frame_interval_enum {
	__u32 index;
	__u32 pad;
	__u32 code;
	__u32 width;
	__u32 height;
	struct v4l2_fract interval;
	__u32 which;
	__u32 stream;
	__u32 reserved[7];
}
```

app:

`index`: 要 get 的 frame interval index.  
`pad`: pad id.  
`code`: bus format code，一般是 7.58 中 get 的 code.  
`which`: 见 7.60 的 which.
`width`: frame width, 一般是 7.57 中 get 的 width.  
`height`: frame height, 一般是 7.57 中 get 的 height.

kernel:

`interval`: 返回 frame interval.

## 7.57 ioctl VIDIOC_SUBDEV_ENUM_FRAME_SIZE

Enumerate media bus frame sizes

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
`code`: bus format code, 一般是 7.58 中 get 的 code.  
`which`: 见 7.60 的 which.

kernel:

`min_width`, `max_width`, `min_height`, `max_height`: 如果是 discrete frame size，那么 min 和 max 值相同。

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
};


struct v4l2_mbus_framefmt {
	__u32			width;
	__u32			height;
	__u32			code;
	__u32			field;
	__u32			colorspace;
	union {
		__u16			ycbcr_enc; /* enum v4l2_ycbcr_encoding */
		__u16			hsv_enc; /* enum v4l2_hsv_encoding */
	};
	__u16			quantization;
	__u16			xfer_func;
	__u16			flags;
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
};

struct v4l2_fract {
	__u32   numerator;
	__u32   denominator;
};
```

app:

`pad`: pad number.  
`interval`: 要设置的 struct v4l2_fract.  
`stream`: stream api 相关。

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

`type`: 要订阅的 event 类型，V4L2_EVENT_XXX.  
`id`: 根据 type 不同类型，有的需要 event source id.
`flags`: V4L2_EVENT_SUB_FL_XXX.

# Media Controller API

## 5.4 ioctl MEDIA_IOC_DEVICE_INFO

获取 media device 的信息

## 5.5 ioctl MEDIA_IOC_G_TOPOLOGY

新版 api，获取 entity，interface，pads，links。

```c++
struct media_v2_topology {
	__u64 topology_version;

	__u32 num_entities;
	__u64 ptr_entities;

	__u32 num_interfaces;
	__u64 ptr_interfaces;

	__u32 num_pads;
	__u64 ptr_pads;

	__u32 num_links;
	__u64 ptr_links;
}
```

kernel:

`topology_version`: topology 版本。  
`num_entities`: entites 数量。  
`ptr_entities`：struct media_v2_entity 数组。  
`num_interfaces`: interfaces 数量。  
`ptr_interfaces`: struct media_v2_interface 数组。  
`num_pads`: pads 数量。  
`ptr_pads`: struct media_v2_pad 数组。  
`num_links`: links 数量。  
`ptr_links`: struct media_v2_link 数组。

## 5.6 ioctl MEDIA_IOC_ENUM_ENTITIES

Enumerate entities and their properties

用户层传入 entity 的 id，返回 entity 信息

```c++
struct media_entity_desc {
	__u32 id;
	char name[32];
	__u32 type;
	__u32 flags;
	__u16 pads;
	__u16 links;

	__u32 reserved[4];

	union {
		struct {
			__u32 major;
			__u32 minor;
		} dev;
		__u8 raw[184];
	};
}
```

app 传入：

`id`: 如果与上了 MEDIA_ENT_ID_FLAG_NEXT flag，则返回下一个 entity。

kernel 返回：

`id`: entity 真正的 id 号。  
`name`  
`type`  
`flags`  
`pads`  
`links`  
`type`

## 5.7 ioctl MEDIA_IOC_ENUM_LINKS

Enumerate all pads and links for a given entity

```c++
struct media_links_enum {
	__u32 entity;
	struct media_pad_desc __user *pads;
	struct media_link_desc __user *links;
	__u32 reserved[4];
};
```

app 传入：

`entity`: entity id。

kernel 返回：

`pads`: struct media_pad_desc 数组

```c++
struct media_pad_desc {
	__u32 entity; /* entity ID */
	__u16 index; /* pad index */
	__u32 flags; /* pad flags */
}
```

`links`: struct media_link_desc 数组

```c++
struct media_link_desc {
	struct media_pad_desc source;
	struct media_pad_desc sink;
	__u32 flags;
}
```

## 5.8 ioctl MEDIA_IOC_SETUP_LINK

Modify the properties of a link

enable source pad 和 sink pad 的 link.

```c++
struct media_link_desc {
	struct media_pad_desc source;
	struct media_pad_desc sink;
	__u32 flags;
}
```

app:

`source`: source pad desc.  
`sink`: sink pad desc.  
`flags`: MEDIA_LNK_FL_ENABLE.
