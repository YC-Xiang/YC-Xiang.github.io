# v4l2-async.c

# v4l2-common.c

```c++
v4l_err()/v4l_warn()/v4l_info()/v4l_dbg(); // for v4l-i2c drivers
v4l2_err()/v4l2_warn()/v4l2_info()/v4l2_dbg(); // for v4l2_device and v4l2_subdev
// v4l2_ctrl_query_fill();
// v4l2_i2c_new_subdev();
v4l2_i2c_subdev_set_name();

```

# v4l2-dev.c

```c++
v4l2_prio_init();
v4l2_prio_change();
v4l2_prio_open();
v4l2_prio_close();
v4l2_prio_max();
v4l2_prio_check();

media_entity_to_video_device();
// to_video_device();
> video_register_device();/video_unregister_device();
// video_register_device_no_warn();
> video_device_alloc();/video_device_release();
> video_device_release_empty();
> v4l2_disable_ioctl();
> video_get_drvdata();/video_set_drvdata(); // get/set video device private data
> video_devdata(); // 获取 file 对应的 video device
> video_drvdata(); // video_get_drvdata(video_devdata(file));
> video_device_node_name(); // 获取 device name
> video_is_registered(); // 检查 video device 是否已经注册

> video_device_pipeline_start();__video_device_pipeline_start();
> video_device_pipeline_stop();__video_device_pipeline_stop();
> video_device_pipeline_alloc_start(); // 多了 allocate pipeline 的操作，不过一般 pipeline 会嵌在 driver 自定义结构体中，随着上层一起分配。
// video_device_pipeline();
```

# v4l2-device.c

```c++
v4l2_device_get();v4l2_device_put();
```
