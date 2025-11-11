# v4l2-async.c

# v4l2-common.h

```c++
v4l_err()/v4l_warn()/v4l_info()/v4l_dbg(); // for v4l-i2c drivers
v4l2_err()/v4l2_warn()/v4l2_info()/v4l2_dbg(); // for v4l2_device and v4l2_subdev
// v4l2_ctrl_query_fill();
// v4l2_i2c_new_subdev();
v4l2_i2c_subdev_set_name();

```

# v4l2-ctl.h

# v4l2-dev.h

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
> video_drvdata(); // video_get_drvdata());
> video_device_node_name(); // 获取 device name
> video_is_registered(); // 检查 video device 是否已经注册

> video_device_pipeline_start();__video_device_pipeline_start(); // 在 start streaming 中调用
> video_device_pipeline_stop();__video_device_pipeline_stop(); // 在 stop streaming 中调用
> video_device_pipeline_alloc_start(); // 多了 allocate pipeline 的操作，不过一般 pipeline 会嵌在 driver 自定义结构体中，随着上层一起分配。
// video_device_pipeline();
```

# v4l2-device.h

```c++
v4l2_device_get();v4l2_device_put();
> v4l2_device_register();/v4l2_device_unregister();
v4l2_device_set_name();
// v4l2_device_disconnect(); // 给 usb 设备使用
> v4l2_device_register_subdev();/v4l2_device_unregister_subdev();
> v4l2_device_register_subdev_nodes(); // 注册所有 subdev 的 device node，通常在.complete 中调用
// v4l2_device_register_ro_subdev_nodes(); // 没人用
v4l2_subdev_notify(); // subdev 通知 v4l2 device
// v4l2_device_supports_requests(); // 内部使用
// v4l2_device_for_each_subdev(); // 遍历所有的 subdev, 只有 pci 和 usb 设备调用

// v4l2_device_call_all(); // 只有 pci 和 usb 设备使用
> v4l2_device_call_until_err(); // call 某个 group id 中所有 subdev 的某个 op, 直到发生错误
// v4l2_device_mask_call_all();/v4l2_device_mask_call_until_err(); // 没人用
// v4l2_device_has_op();/v4l2_device_mask_has_op() // 没人用
```

# v4l2-event.h

# v4l2-fh.h

```c++
v4l2_fh_init();/v4l2_fh_exit();
v4l2_fh_add();/v4l2_fh_del();
v4l2_fh_open();/v4l2_fh_release():
v4l2_fh_is_singular();
v4l2_fh_is_singular_file();
```

# v4l2-fwnode.h

```c++
v4l2_fwnode_endpoint_parse();
```

# v4l2-mc.h

```c++
~ v4l2_mc_create_media_graph(); // pci 和 usb 设备使用
~ v4l_enable_media_source();/v4l_disable_media_source();/v4l_vb2q_enable_media_source();
> v4l2_create_fwnode_links_to_pad();
~ v4l2_create_fwnode_links();
~ v4l2_pipeline_pm_get();v4l2_pipeline_pm_put(); // linux-6.17 中已经被废弃
~ v4l2_pipeline_link_notify(); // linux-6.17 中已经被废弃
```

`v4l2_create_fwnode_links_to_pad`: 这个函数搜索 fwnode endpoint connections 从 source subdev 到 single sink pad.
在 sink device 的 v4l2-async notifier bound callback 中调用，来创建 link 从 source subdev 到 sink pad. 如果要使用这个函数，必须实现 .get_fwnode_pad media operation

`v4l2_create_fwnode_links`: 这个函数搜索 fwnode endpoint connections 在 source subdev 和 sink subdev 之间，把他们转化为 media link.
在 sink device 的 v4l2-async notifier bound callback 中调用。如果要使用这个函数，必须实现 .get_fwnode_pad media operation

# v4l2-mediabus.h

```c++
v4l2_fill_pix_format();
v4l2_fill_mbus_format();
v4l2_fill_pix_format_mplane();
v4l2_fill_mbus_format_mplane();
```

# v4l2-mem2mem.h

```c++
> v4l2_m2m_get_curr_priv(); // 返回当前 running instance 的 driver private data
> v4l2_m2m_get_vq(); // 
// v4l2_m2m_try_schedule();
> v4l2_m2m_job_finish(); // 当前 instance 完成任务后需要调用，把 device 释放出来。需要在.device_run 之后调用
// v4l2_m2m_buf_done_and_job_finish(); // 给 stateless encoder 使用
> v4l2_m2m_buf_done();
// v4l2_m2m_clear_state(); // clear encoding/decoding state
// v4l2_m2m_mark_stopped(); // set current encoding/decodfing as stopped
// v4l2_m2m_dst_buf_is_last(); // encoding/decoding session draining management state
// v4l2_m2m_has_stopped();
// v4l2_m2m_is_last_draining_src_buf();
// v4l2_m2m_last_buffer_done();
> v4l2_m2m_suspend();/v4l2_m2m_resume();

// 不使用 ioctl helpers 的话 vendor driver 可以自己包装下面一系列函数
v4l2_m2m_reqbufs();
v4l2_m2m_querybuf();
v4l2_m2m_qbuf();/v4l2_m2m_dqbuf();
v4l2_m2m_prepare_buf();
v4l2_m2m_create_bufs();
v4l2_m2m_expbuf();
v4l2_m2m_streamon();v4l2_m2m_streamoff();
v4l2_m2m_poll();
v4l2_m2m_mmap();

// v4l2_m2m_update_start_streaming_state();/v4l2_m2m_update_stop_streaming_state(); // encoder/decoder
// v4l2_m2m_encoder_cmd();
// v4l2_m2m_decoder_cmd();
> v4l2_m2m_init();/v4l2_m2m_release(); // 分配/释放 struct v4l2_m2m_dev, 通常在 probe()/remove() 中调用
> v4l2_m2m_register_media_controller();/v4l2_m2m_unregister_media_controller(); // 注册/注销 media controller
> v4l2_m2m_ctx_init();/v4l2_m2m_ctx_release(); // 分配 m2m ctx, 通常在 open()/release() 中调用
// v4l2_m2m_set_src_buffered();/v4l2_m2m_set_dst_buffered(); // 没人用
> v4l2_m2m_buf_queue();  // 把 buffer 加入到 buffer list
v4l2_m2m_num_src_bufs_ready();/v4l2_m2m_num_dst_bufs_ready(); // 返回 src/dst 中 ready buf 的数量
> v4l2_m2m_next_src_buf();/v4l2_m2m_next_dst_buf(); // 拿下一个 src/dst ready buf
// v4l2_m2m_last_src_buf();v4l2_m2m_last_dst_buf(); // encoder/decoder 用的
v4l2_m2m_for_each_src_buf();v4l2_m2m_for_each_dst_buf(); // 遍历 src/dst ready bufs
v4l2_m2m_for_each_src_buf_safe();/v4l2_m2m_for_each_dst_buf_safe(); // 和上面差不多
v4l2_m2m_get_src_vq();/v4l2_m2m_get_dst_vq(); // 获取 src/dst vb2_queue
> v4l2_m2m_src_buf_remove();v4l2_m2m_dst_buf_remove(); // 从 src/dst buf list 中 remove 一个 buf
v4l2_m2m_src_buf_remove_by_buf();v4l2_m2m_dst_buf_remove_by_buf(); // remove 某个特定的 buf
// v4l2_m2m_src_buf_remove_by_idx();v4l2_m2m_dst_buf_remove_by_idx(); // 没人用
> v4l2_m2m_buf_copy_metadata(); // copy metadata from output buffer to capture buffer

v4l2_m2m_request_queue(); // request api 相关

// ioctl helpers
v4l2_m2m_ioctl_reqbufs();
v4l2_m2m_ioctl_create_bufs();
v4l2_m2m_ioctl_querybuf();
v4l2_m2m_ioctl_expbuf();
v4l2_m2m_ioctl_qbuf();
v4l2_m2m_ioctl_dqbuf();
v4l2_m2m_ioctl_prepare_buf();
v4l2_m2m_ioctl_streamon();
v4l2_m2m_ioctl_streamoff();
// v4l2_m2m_ioctl_encoder_cmd();
// v4l2_m2m_ioctl_decoder_cmd();
// v4l2_m2m_ioctl_try_encoder_cmd();
// v4l2_m2m_ioctl_try_decoder_cmd();
// v4l2_m2m_ioctl_stateless_try_decoder_cmd();
// v4l2_m2m_ioctl_stateless_decoder_cmd();
v4l2_m2m_fop_mmap();
v4l2_m2m_fop_poll();
```

# v4l2-subdev.h

```c++
v4l2_subdev_get_pad_format();
v4l2_subdev_get_pad_crop();
v4l2_subdev_get_pad_compose();
v4l2_set_subdevdata();v4l2_get_subdevdata();
v4l2_set_subdev_hostdata();v4l2_get_subdev_hostdata();
v4l2_subdev_get_fwnode_pad_1_to_1();
v4l2_subdev_link_validate_default();
v4l2_subdev_link_validate();
v4l2_subdev_has_pad_interdep();
v4l2_subdev_init_finalize();
v4l2_subdev_cleanup();
v4l2_subdev_lock_state();/v4l2_subdev_unlock_state();
v4l2_subdev_get_unlocked_active_state();
v4l2_subdev_get_locked_active_state();
v4l2_subdev_lock_and_get_active_state();
v4l2_subdev_get_fmt();
v4l2_subdev_set_routing();
for_each_active_route();
v4l2_subdev_set_routing_with_fmt();
v4l2_subdev_state_get_stream_format();
v4l2_subdev_state_get_stream_crop();
v4l2_subdev_state_get_stream_compose();
v4l2_subdev_routing_find_opposite_end();
v4l2_subdev_state_get_opposite_stream_format();
v4l2_subdev_state_xlate_streams();
v4l2_subdev_routing_validate();
v4l2_subdev_enable_streams();/v4l2_subdev_disable_streams();
v4l2_subdev_s_stream_helper();
v4l2_subdev_init();
v4l2_subdev_call();
v4l2_subdev_notify_event();
```
