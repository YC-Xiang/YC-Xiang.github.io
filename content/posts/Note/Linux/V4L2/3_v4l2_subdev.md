## 2.7 V4L2 sub-devices

许多 drivers 需要和 sub-devices 交流, 这些子设备可以处理 audio, video muxing, encoding, decoding 等等.  
对于 webcams, 常见的 sub-devices 有 sensors, camera controllers.

通过 `v4l2_subdev_init(sd, &ops)` 来初始化 v4l2_subdev. 接着需要设置 `sd->name`.

如果需要和 media framework 聚合, 那么需要初始化 media_entity 成员, 如果 entity 有 pads 还需要调用`media_entity_pads_init`.

```c
struct v4l2_subdev {
#if defined(CONFIG_MEDIA_CONTROLLER)
	struct media_entity entity;
#endif
	struct list_head list;
	struct module *owner;
	bool owner_v4l2_dev;
	u32 flags;
	struct v4l2_device *v4l2_dev;
	const struct v4l2_subdev_ops *ops;
	const struct v4l2_subdev_internal_ops *internal_ops;
	struct v4l2_ctrl_handler *ctrl_handler;
	char name[52];
	u32 grp_id;
	void *dev_priv;
	void *host_priv;
	struct video_device *devnode;
	struct device *dev;
	struct fwnode_handle *fwnode;
	struct list_head async_list;
	struct list_head async_subdev_endpoint_list;
	struct v4l2_async_notifier *subdev_notifier;
	struct list_head asc_list;
	struct v4l2_subdev_platform_data *pdata;
	struct mutex *state_lock;

	struct led_classdev *privacy_led;
	struct v4l2_subdev_state *active_state;
	u64 enabled_streams;
};
```

owner_v4l2_dev: 如果和 v4l2_dev->dev 的 owner 一致, 则为 true.  
flags: V4L2_SUBDEV_FL_IS_I2C 表示是 I2C 设备, V4L2_SUBDEV_FL_IS_SPI 表示是 SPI 设备,
V4L2_SUBDEV_FL_HAS_DEVNODE 表示需要 device node, V4L2_SUBDEV_FL_HAS_EVENTS 表示会生成 events.
V4L2_SUBDEV_FL_STREAMS 表示支持 multiplexed streams.
v4l2_dev: 指向 v4l2_device.  
ops: subdev 的操作函数.  
internal_ops: subdev 的内部操作函数.  
ctrl_handler: v4l2 控制句柄.
grp_id: 该 subdev 属于哪个 subdev group, 由 driver 自定义.

e.g.

```c
static const struct v4l2_subdev_pad_ops rkisp1_isp_pad_ops = {
	...
};

static const struct v4l2_subdev_video_ops rkisp1_isp_video_ops = {
	...
};

static const struct v4l2_subdev_core_ops rkisp1_isp_core_ops = {
	...
};

static const struct v4l2_subdev_ops rkisp1_isp_ops = {
	.core = &rkisp1_isp_core_ops,
	.video = &rkisp1_isp_video_ops,
	.pad = &rkisp1_isp_pad_ops,
	...
};

v4l2_subdev_init(sd, &rkisp1_isp_ops);
```

### 2.7.1 Subdev registration

V4L2 子设备注册的方式有两种.

第一种是由 bridge driver 来注册 subdevices. 适用于 internal subdevices, 比如 video data processing units,
camera sensor.

第二种是 bridge driver 和 subdevice 异步注册. 适用于 subdevices 的信息不是在 bridge driver 中,
而是比如在设备树中定义的 i2c 设备.

#### 2.7.1.1 Registering synchronous sub-devices

和 bridge driver 同步注册.

注册: `v4l2_device_register_subdev(v4l2_dev, sd)`

注销: `v4l2_device_unregister_subdev(sd)`

#### 2.7.1.2 Registering asynchronous sub-devices

和 bridge driver 异步注册.

注册: `v4l2_async_register_subdev(v4l2_dev, sd)`

注销: `v4l2_async_unregister_subdev(sd)`

通过异步注册的 subdev, 会被挂入全局的 global list, 等待 bridge driver 统一注册.

#### 2.7.1.3 Asynchronous sub-device notifiers

Bridge driver 需要注册一个 notifier object.

注册: `v4l2_async_nf_register()`

注销: `v4l2_async_nf_unregister()`, 在释放 unregistered notifier 前, 还需要调用`v4l2_async_nf_cleanup()`.

注册 notifier 前, bridge driver 首先需要调用`v4l2_async_nf_init()`, 接着可以通过`v4l2_async_nf_add_fwnode()`, `v4l2_async_nf_add_fwnode_remote()` 和 `v4l2_async_nf_add_i2c()` 来获取 bridge device 需要的 connection descriptor.

#### 2.7.1.4 Asynchronous sub-device notifier for sub-devices

async sub-device 也可以注册一个 asyncchronous notifier, 比如用于注册 sensor 依赖的一些设备, flash LED, lens focus.

通过 `v4l2_async_subdev_nf_init()` 初始化.

#### 2.7.1.5 Asynchronous sub-device registration helper for camera sensor drivers

对于 camera sensor, 有一个 helper function 用来注册 subdev, 同时还会注册 sensor 的异步 notifier, 用来
异步注册 lens, flash devices 等.

`v4l2_async_register_subdev_sensor()`

#### 2.7.1.7 Asynchronous sub-device notifier callbacks

V4L2 core 会利用 connection descriptors 来匹配异步注册的 subdevices.

connection match 之后调用.bound()回调, 所有 connections 都 bound 之后调用.complete()回调.

connection remove 之后调用.unbind()回调.

### 2.7.2 Calling subdev operations

调用某个 subdev 的 ops 回调, 使用如下宏:

`v4l2_subdev_call()`

调用所有或某组 subdev 的 ops:

`v4l2_device_call_all()`

`v4l2_device_call_until_err()`

如果该 subdev 要通知上层的 v4l2_device, 可以调用`v4l2_device_notify()`

## 2.8 V4L2 sub-device userspace API

`/dev` 下的 `v4l-subdevX` 节点, 对应于 v4l2_subdev. 如果需要生成 subdev 设备节点, subdev 需要在注册前
置起 V4L2_SUBDEV_FL_HAS_DEVNODE flag. v4l2 driver 通过 v4l2_device_register_subdev_nodes()来注册所有的 subdev 节点.

这些 subdev device nodes 支持下列 ioctl:

`VIDIOC_QUERYCTRL`, `VIDIOC_QUERYMENU`, `VIDIOC_G_CTRL`, `VIDIOC_S_CTRL`, `VIDIOC_G_EXT_CTRLS`,
`VIDIOC_S_EXT_CTRLS`, `VIDIOC_TRY_EXT_CTRLS`.

subdev 这些 ioctl 和 v4l2 device 是一致的.

`VIDIOC_DQEVENT`, `VIDIOC_SUBSCRIBE_EVENT`, `VIDIOC_UNSUBSCRIBE_EVENT`

subdev event 相关的 ioctl 和 v4l2 device 也是一致的. 需要在注册 subdev 前置起 V4L2_SUBDEV_FL_HAS_EVENTS.

## 2.9 Read-only sub-device userspace API

如果不希望通过 subdev device node 修改参数, 可以设置为只读的.

v4l2 driver 用 v4l2_device_register_ro_subdev_nodes()代替 v4l2_device_register_subdev_nodes()注册 subdev 节点.

## 2.10 I2C sub-device drivers

i2c subdev 的一些 helper function 在`v4l2-common.h`中.

```c
void v4l2_i2c_subdev_init(struct v4l2_subdev *sd, struct i2c_client *client,
		const struct v4l2_subdev_ops *ops);
struct v4l2_subdev *v4l2_i2c_new_subdev(struct v4l2_device *v4l2_dev,
		struct i2c_adapter *adapter, const char *client_type,
		u8 addr, const unsigned short *probe_addrs);
struct v4l2_subdev *v4l2_i2c_new_subdev_board(struct v4l2_device *v4l2_dev,
		struct i2c_adapter *adapter, struct i2c_board_info *info,
		const unsigned short *probe_addrs);
```

v4l2_i2c_subdev_init(): 初始化 v4l2_subdev.

v4l2_i2c_new_subdev(): 注册 v4l2_subdev 和 i2c_client.

v4l2_i2c_new_subdev_board(): v4l2_i2c_new_subdev 更底层的实现.

## 2.11 Centrally managed subdev active state

// TODO:

## 2.13 data structures and functions

APIs:

`v4l2-async.h`

```c
void v4l2_async_nf_init(struct v4l2_async_notifier *notifier,
			struct v4l2_device *v4l2_dev);
void v4l2_async_subdev_nf_init(struct v4l2_async_notifier *notifier,
			       struct v4l2_subdev *sd);

int v4l2_async_nf_register(struct v4l2_async_notifier *notifier);
void v4l2_async_nf_unregister(struct v4l2_async_notifier *notifier);
void v4l2_async_nf_cleanup(struct v4l2_async_notifier *notifier);

#define v4l2_async_nf_add_fwnode(notifier, fwnode, type);
#define v4l2_async_nf_add_fwnode_remote(notifier, ep, type);
#define v4l2_async_nf_add_i2c(notifier, adapter, address, type);

int v4l2_async_subdev_endpoint_add(struct v4l2_subdev *sd,
				   struct fwnode_handle *fwnode);
struct v4l2_async_connection *
v4l2_async_connection_unique(struct v4l2_subdev *sd);


#define v4l2_async_register_subdev(sd);
void v4l2_async_unregister_subdev(struct v4l2_subdev *sd);
int v4l2_async_register_subdev_sensor(struct v4l2_subdev *sd);
```

`v4l2-subdev.h`

```c

```
