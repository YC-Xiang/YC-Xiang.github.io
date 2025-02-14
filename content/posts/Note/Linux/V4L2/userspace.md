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

查询 v4l2 设备支持的功能, 返回`struct v4l2_capability`结构体. 所有 app 程序在 open 后都要执行.

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

# Media Controller API
