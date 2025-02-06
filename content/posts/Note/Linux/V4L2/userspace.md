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
