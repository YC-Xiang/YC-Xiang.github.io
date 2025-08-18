基于 linux 6.6 drm api

注释掉的 api 表示暂时不关心，带 > 的 api 表示常常在 driver 中调用，其他都是在 drm internal 中内部调用，或不常使用。

# drm_atomic_helper.c

```c++
drm_atomic_helper_check_modeset(); // drm_atomic_helper_check() 中调用
// drm_atomic_helper_check_wb_encoder_state();
drm_atomic_helper_check_plane_state(); // 在 plane atomic check 中调用
drm_atomic_helper_check_planes(); // drm_atomic_helper_check() 中调用
drm_atomic_helper_check_crtc_primary_plane(); // 检查 crtc 是否绑定了 primary plane。一般给 simple drm 只有一个 primary plane 的 driver 使用。
drm_atomic_helper_check(); // drm_mode_config_funcs->atomic_check 的 helper func
drm_atomic_helper_commit_tail(); // drm_mode_config_helper_funcs.atomic_commit_tail 的 helper 函数。
drm_atomic_helper_commit_tail_rpm(); // rpm 版本
drm_atomic_helper_commit(); // drm_mode_config_funcs->atomic_commit 的 helper 函数。
// drm_atomic_helper_async_check();
// drm_atomic_helper_async_commit();
drm_atomic_helper_wait_for_fences(); // ??? dma fence 相关
drm_atomic_helper_wait_for_vblanks(); // 在 drm_atomic_helper_commit_tail() 中调用
drm_atomic_helper_wait_for_flip_done(); // 可以代替 wait_for_vblanks. vendor driver 自行在 atomic commit 回调中调用，用于等待所有的 crtc 完成。
drm_atomic_helper_update_legacy_modeset_state(); // drm_atomic_helper_commit_modeset_disables() 中调用
drm_atomic_helper_calc_timestamping_constants(); // drm_atomic_helper_commit_modeset_disables() 中调用
drm_atomic_helper_commit_modeset_disables(); // drm_atomic_helper_commit_tail() 中调用
drm_atomic_helper_commit_modeset_enables(); // drm_atomic_helper_commit_tail() 中调用
drm_atomic_helper_prepare_planes(); // drm_atomic_helper_commit() 中调用
drm_atomic_helper_unprepare_planes(); // drm_atomic_helper_commit() 中调用
drm_atomic_helper_commit_planes(); // drm_atomic_helper_commit_tail() 中调用
drm_atomic_helper_cleanup_planes(); // drm_atomic_helper_commit_tail() 中调用
// drm_atomic_helper_commit_planes_on_crtc();
drm_atomic_helper_disable_planes_on_crtc(); // ??? 一些 vendor driver 在 crtc 的 atomic_disable 中调用来 disable planes
drm_atomic_helper_swap_state(); // drm_atomic_helper_commit() 中调用
drm_atomic_helper_setup_commit(); // drm_atomic_helper_commit() 中调用
drm_atomic_helper_wait_for_dependencies(); // drm_atomic_helper_commit() 中调用
drm_atomic_helper_fake_vblank(); // drm_atomic_helper_commit_tail() 中调用
drm_atomic_helper_commit_hw_done(); // drm_atomic_helper_commit_tail() 中调用
drm_atomic_helper_commit_cleanup_done(); // drm_atomic_helper_commit() 中调用

/* implementations for legacy interfaces */
drm_atomic_helper_update_plane(); // drm_plane_funcs->update_plane
drm_atomic_helper_disable_plane(); // drm_plane_funcs->disable_plane
drm_atomic_helper_set_config(); // drm_crtc_funcs->set_config
drm_atomic_helper_disable_all(); // 在 drm_atomic_helper_suspend() 和 drm_atomic_helper_shutdown() 中调用
drm_atomic_helper_shutdown();
drm_atomic_helper_duplicate_state(); // drm_atomic_helper_suspend() 中调用
drm_atomic_helper_suspend();
drm_atomic_helper_commit_duplicated_state(); // drm_atomic_helper_resume() 中调用
drm_atomic_helper_resume();
drm_atomic_helper_page_flip(); // drm_crtc_funcs->page_flip
// drm_atomic_helper_page_flip_target();

drm_atomic_crtc_for_each_plane();
drm_atomic_crtc_state_for_each_plane();
drm_atomic_crtc_state_for_each_plane_state();
drm_atomic_helper_bridge_propagate_bus_fmt();
```

如果 vendor driver 使用提供的 helper function `drm_atomic_helper_check()` 和 `drm_atomic_helper_commit()`，那么其他的 helper function 大部分不需要 vendor driver 再去调用，除了：

- drm_atomic_helper_check_plane_state()
- drm_atomic_helper_disable_planes_on_crtc()

`drm_atomic_helper_check_modeset`: 在 drm_mode_config_funcs->atomic_check 中调用。检查 atomic state 是否是 physical possible 的。
如果在 plane 的 atomic_check 函数中设置了 mode_changed=true (表示 plane 的更新需要 full modeset，比如调整 pclk, 内存带宽等等)，则需要再 plane check 之后再次调用该函数。

// TODO: add comments on each function

# drm_atomic_state_helper.c

```c++
__drm_atomic_helper_crtc_state_reset(); // __drm_atomic_helper_crtc_reset() 中调用
__drm_atomic_helper_crtc_reset(); // vendor subclass drm_crtc_state 时，在 drm_crtc_funcs->reset 中调用
drm_atomic_helper_crtc_reset(); // vendor 直接使用 drm_crtc_state 时，drm_crtc_funcs->reset
__drm_atomic_helper_crtc_duplicate_state(); // vendor subclass drm_crtc_state 时，在 drm_crtc_funcs->atomic_duplicate_state 中调用
drm_atomic_helper_crtc_duplicate_state(); // vendor 直接使用 drm_crtc_state 时，drm_crtc_funcs->atomic_duplicate_state
__drm_atomic_helper_crtc_destroy_state(); // vendor subclass drm_crtc_state 时，在 drm_crtc_funcs->rts_crtc_atomic_destroy_state 中调用
drm_atomic_helper_crtc_destroy_state(); // vendor 直接使用 drm_crtc_state 时，drm_crtc_funcs->rts_crtc_atomic_destroy_state

__drm_atomic_helper_plane_state_reset(); // __drm_atomic_helper_plane_reset 中调用
__drm_atomic_helper_plane_reset(); // vendor subclass drm_plane_state 时，在 drm_plane_funcs->reset 中调用
drm_atomic_helper_plane_reset(); // vendor 直接使用 drm_plane_state 时，drm_plane_funcs->reset
__drm_atomic_helper_plane_duplicate_state(); // 同上
drm_atomic_helper_plane_duplicate_state(); // 同上
__drm_atomic_helper_plane_destroy_state(); // 同上
drm_atomic_helper_plane_destroy_state(); // 同上

__drm_atomic_helper_connector_state_reset(); // 同上
__drm_atomic_helper_connector_reset(); // 同上
drm_atomic_helper_connector_reset(); // 同上
// drm_atomic_helper_connector_tv_reset();
// drm_atomic_helper_connector_tv_check();
// drm_atomic_helper_connector_tv_margins_reset();
__drm_atomic_helper_connector_duplicate_state(); // 同上
drm_atomic_helper_connector_duplicate_state(); // 同上
__drm_atomic_helper_connector_destroy_state(); // 同上
drm_atomic_helper_connector_destroy_state(); // 同上

__drm_atomic_helper_private_obj_duplicate_state(); // 同上

__drm_atomic_helper_bridge_duplicate_state(); // 同上
drm_atomic_helper_bridge_duplicate_state(); // 同上
drm_atomic_helper_bridge_destroy_state(); // 同上
__drm_atomic_helper_bridge_reset(); // 同上
drm_atomic_helper_bridge_reset(); // 同上
```

# drm_atomic.c

```c++
drm_crtc_commit_get()
drm_crtc_commit_put()

drm_atomic_state_alloc();
drm_atomic_state_clear();
drm_atomic_state_get()
drm_atomic_state_put()
drm_atomic_state_init();
drm_atomic_state_default_clear();
drm_atomic_state_default_release();

drm_atomic_get_crtc_state();
drm_atomic_get_plane_state();
drm_atomic_get_connector_state();

drm_atomic_private_obj_init();
drm_atomic_private_obj_fini();

drm_atomic_get_private_obj_state();
drm_atomic_get_old_private_obj_state();
drm_atomic_get_new_private_obj_state();

drm_atomic_get_old_connector_for_encoder();
drm_atomic_get_new_connector_for_encoder();
drm_atomic_get_old_crtc_for_encoder();
drm_atomic_get_new_crtc_for_encoder();

drm_atomic_get_existing_crtc_state();
drm_atomic_get_old_crtc_state();
drm_atomic_get_new_crtc_state();
// same for plane/connector

```

vendor driver 会使用的 api 有：

- drm_atomic_get_crtc/plane/connector/private_obj_state
- drm_atomic_private_obj_init/fini

# drm_auth.c

```c++
drm_master_get();
drm_file_get_master();
drm_master_put();
drm_is_current_master();
drm_master_create();
```

# drm_blend.c

```c++
drm_rotation_90_or_270(); // 判断 rotation 是否为 90 or 270 度
drm_plane_create_alpha_property(); // 创建 plane alpha 属性
drm_plane_create_rotation_property(); // 创建 plane rotation 属性
drm_rotation_simplify(); // 排除 plane 不支持的 rotation
drm_plane_create_zpos_property();  // 创建 plane zpos 属性
drm_plane_create_zpos_immutable_property(); // 创建 plane immutable zpos 属性
drm_atomic_normalize_zpos(); // 把 userspace 设置的 zpos 统一到 0~N-1, N 为 planes 的数量。
drm_plane_create_blend_mode_property(); // 创建 plane blend mode 属性
```

# drm_bridge_connector.c

```c++
> drm_bridge_connector_init();
```

注册 connector.

# drm_bridge.c

```c++
drm_bridge_add()/drm_bridge_remove(); // 加入/删除 bridge list, 在 bridge driver 中调用
devm_drm_bridge_add();
drm_bridge_attach(); // attach bridge 到 encoder/previous bridge, 在 bridge driver/vendor driver 中调用
of_drm_find_bridge(); // 
drm_bridge_get_next_bridge()/drm_bridge_get_prev_bridge();
drm_bridge_chain_get_first_bridge();
drm_for_each_bridge_in_chain();
drm_bridge_chain_mode_fixup();
drm_bridge_chain_mode_valid();
drm_bridge_chain_mode_set();
drm_atomic_bridge_chain_check();
drm_atomic_bridge_chain_enable()/drm_atomic_bridge_chain_disable();
drm_atomic_bridge_chain_post_disable();
drm_atomic_bridge_chain_pre_enable();
drm_atomic_helper_bridge_propagate_bus_fmt();
drm_bridge_detect();
drm_bridge_get_modes();
drm_bridge_edid_read();
drm_bridge_get_edid();
drm_bridge_hpd_enable()/drm_bridge_hpd_disable();
drm_bridge_hpd_notify();
drm_bridge_is_panel();
drm_panel_bridge_add()/drm_panel_bridge_remove();
drm_panel_bridge_add_typed();
drm_panel_bridge_set_orientation();
devm_drm_panel_bridge_add();
devm_drm_panel_bridge_add_typed();
drmm_panel_bridge_add();
drm_panel_bridge_connector();
devm_drm_of_get_bridge();
drmm_of_get_bridge();
```

# drm_client_modeset.c

# drm_client.c

# drm_color_mgmt.c

```c++
drm_color_lut_extract();
// drm_color_ctm_s31_32_to_qm_n();
drm_crtc_enable_color_mgmt();
drm_mode_crtc_set_gamma_size();
drm_color_lut_size();
drm_plane_create_color_properties();
drm_color_lut_check();
```

# drm_connector.c

# drm_crtc_helper.c

该文件中的 api 大部分都不是给 atomic driver 使用的。

```c++
drm_crtc_helper_atomic_check(); // crtc atomic_check 的 helper 函数
```

# drm_crtc.c

```c++
> drm_crtc_from_index(); // 获取指定 index 的 crtc
> drm_crtc_init_with_planes();
> drm_crtc_cleanup();
> drmm_crtc_init_with_planes();
> drmm_crtc_alloc_with_planes();
> drm_crtc_mask(); // 返回当前 crtc 的 index mask
> drm_for_each_crtc();
// drm_for_each_crtc_reverse();
// drm_crtc_create_scaling_filter_property();
```

# drm_damage_helper.c

# drm_debugfs.c

```c++
> drm_debugfs_create_files();
> drm_debugfs_remove_files();
> drm_debugfs_add_file();
> drm_debugfs_add_files();
```

drm_debugfs_create_files (传统方式)  
立即创建：调用时立即在指定的 debugfs 目录中创建文件  
作用域：与 drm_minor 绑定，每个 minor 都有独立的文件  
生命周期管理：需要手动管理，调用 drm_debugfs_remove_files 清理  

drm_debugfs_add_files (现代方式)  
延迟创建：添加到待创建列表，在 drm_debugfs_init 时统一创建  
作用域：与 drm_device 绑定，设备级别的调试文件  
生命周期管理：自动管理，使用 drmm_kzalloc 分配，设备销毁时自动清理  

优先使用 drm_debugfs_add_files：现代化的接口，自动内存管理

仅在需要自定义目录结构时使用 drm_debugfs_create_files：需要特殊的 debugfs 层次结构

# drm_drv.c

```c++
> devm_drm_dev_alloc();/drm_dev_alloc();
> drm_dev_register();/drm_dev_unregister();
> drm_dev_get();drm_dev_put();
drm_put_dev(); // drm_dev_unregister() + drm_dev_put()
> drm_dev_enter();/drm_dev_exit(); // device critical section, 如果 unplugged 状态，直接返回
> drm_dev_unplug();
drm_dev_is_unplugged();
> drm_firmware_drivers_only();
drm_core_check_feature();
drm_drv_uses_atomic_modeset();
```

# drm_encoder.c

```c++
> drm_encoder_init();
> drmm_encoder_init();
> drmm_encoder_alloc();
drmm_plain_encoder_alloc();
drm_encoder_index();
> drm_encoder_mask();
drm_encoder_crtc_ok();
drm_encoder_find();
> drm_encoder_cleanup();
> drm_for_each_encoder_mask();
> drm_for_each_encoder();
```

# drm_fb_dma_helper.c

# drm_fb_helper.c

# drm_fbdev_dma.c

# drm_fbdev_generic.c

# drm_file.c

# drm_flip_work.c

# drm_format_helper.c

```c++
drm_fb_clip_offset(); // 返回 clipping rectangle 的左顶点在 framebuffer 中的 offset
drm_fb_memcpy();
drm_fb_swab();
drm_fb_xrgb8888_to_rgb332();
drm_fb_xrgb8888_to_rgb565();
drm_fb_xrgb8888_to_xrgb1555();
drm_fb_xrgb8888_to_argb1555();
drm_fb_xrgb8888_to_rgba5551();
drm_fb_xrgb8888_to_rgb888();
drm_fb_xrgb8888_to_argb8888();
drm_fb_xrgb8888_to_xrgb2101010();
drm_fb_xrgb8888_to_argb2101010();
drm_fb_xrgb8888_to_gray8();
drm_fb_blit();
drm_fb_xrgb8888_to_mono();
drm_fb_build_fourcc_list();
```

这些变换函数，一般是给 mipi dbi 屏用的。

# drm_fourcc.c

```c++
drm_format_info();
drm_get_format_info();
// drm_mode_legacy_fb_format();
// drm_driver_legacy_fb_format();
drm_format_info_block_width();
drm_format_info_block_height();
drm_format_info_bpp();
drm_format_info_min_pitch();
```

# drm_prime.c

# drm_plane.c

```c++
drm_plane_state_src();/drm_plane_state_dst();
> drm_universal_plane_init();
> drm_plane_cleanup();
> drmm_universal_plane_alloc();
> drm_plane_index();
> drm_plane_mask();
drm_plane_from_index();
drm_plane_force_disable();
drm_mode_plane_set_obj_prop();
drm_plane_find();
> drm_for_each_plane_mask();
> drm_for_each_plane();
drm_any_plane_has_format();
drm_plane_enable_fb_damage_clips();
drm_plane_get_damage_clips_count();
drm_plane_get_damage_clips();
drm_plane_create_scaling_filter_property();
```

# drm_print.c

```c++
drm_info();
drm_notice();
drm_warn();
drm_err();
drm_info_once();
drm_notice_once();
drm_warn_once();
drm_err_once();
```

# drm_probe_helper.c

```c++
drm_helper_probe_single_connector_modes();
drm_kms_helper_poll_init();/drm_kms_helper_poll_fini();
drm_helper_hpd_irq_event();
drm_connector_helper_hpd_irq_event();
drm_kms_helper_hotplug_event();
drm_kms_helper_connector_hotplug_event();
drm_kms_helper_poll_enable();drm_kms_helper_poll_disable();
drm_kms_helper_poll_reschedule();
drm_kms_helper_is_poll_worker();
// drm_crtc_helper_mode_valid_fixed();
// drm_connector_helper_get_modes_from_ddc();
// drm_connector_helper_get_modes_fixed();
// drm_connector_helper_get_modes();
// drm_connector_helper_tv_get_modes();
```

# drm_property.c

```c++
drm_property_create();/drm_property_destroy();
> drm_property_create_enum();
> drm_property_create_bitmask();
> drm_property_create_range();
> drm_property_create_signed_range();
drm_property_create_object();
drm_property_create_bool();
> drm_property_add_enum();
> drm_property_create_blob();
> drm_property_blob_get();/drm_property_blob_put();
drm_property_lookup_blob();
drm_property_replace_global_blob();
> drm_property_replace_blob();
```

# drm_rect.c

```c++
drm_rect_intersect();
drm_rect_clip_scaled();
drm_rect_calc_hscale();/drm_rect_calc_vscale();
drm_rect_rotate();
drm_rect_rotate_inv();
```

# drm_simple_kms_helper.c

```c++
drm_simple_encoder_init();
drmm_simple_encoder_alloc();
drm_simple_display_pipe_attach_bridge();
drm_simple_display_pipe_init();
```

# drm_syncobj.c

# drm_vblank_work.c

# drm_vblank.c
