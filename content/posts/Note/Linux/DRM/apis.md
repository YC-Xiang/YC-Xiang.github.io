基于 linux 6.6 drm api

注释掉的 api 表示暂时不关心

# drm_atomic_helper.ch

```c++
drm_atomic_helper_check_modeset();
// drm_atomic_helper_check_wb_encoder_state();
drm_atomic_helper_check_plane_state();
drm_atomic_helper_check_planes();
drm_atomic_helper_check_crtc_primary_plane();
drm_atomic_helper_check();
drm_atomic_helper_commit_tail();
drm_atomic_helper_commit_tail_rpm();
drm_atomic_helper_commit();
drm_atomic_helper_async_check();
drm_atomic_helper_async_commit();
drm_atomic_helper_wait_for_fences();
drm_atomic_helper_wait_for_vblanks();
drm_atomic_helper_wait_for_flip_done();
drm_atomic_helper_update_legacy_modeset_state();
drm_atomic_helper_calc_timestamping_constants();
drm_atomic_helper_commit_modeset_disables();
drm_atomic_helper_commit_modeset_enables();
drm_atomic_helper_prepare_planes();
drm_atomic_helper_unprepare_planes();
drm_atomic_helper_commit_planes();
drm_atomic_helper_cleanup_planes();
drm_atomic_helper_commit_planes_on_crtc();
drm_atomic_helper_disable_planes_on_crtc();
drm_atomic_helper_swap_state();

/* nonblocking commit helpers */
drm_atomic_helper_setup_commit();
drm_atomic_helper_wait_for_dependencies();
drm_atomic_helper_fake_vblank();
drm_atomic_helper_commit_hw_done();
drm_atomic_helper_commit_cleanup_done();

/* implementations for legacy interfaces */
drm_atomic_helper_update_plane();
drm_atomic_helper_disable_plane();
drm_atomic_helper_set_config();
drm_atomic_helper_disable_all();
drm_atomic_helper_shutdown();
drm_atomic_helper_duplicate_state();
drm_atomic_helper_suspend();
drm_atomic_helper_commit_duplicated_state();
drm_atomic_helper_resume();
drm_atomic_helper_page_flip();
drm_atomic_helper_page_flip_target();

#define drm_atomic_crtc_for_each_plane();
#define drm_atomic_crtc_state_for_each_plane();
#define drm_atomic_crtc_state_for_each_plane_state();
inline drm_atomic_plane_enabling();
inline drm_atomic_plane_disabling();
drm_atomic_helper_bridge_propagate_bus_fmt();
```

# drm_atomic_state_helper.ch

```c++
__drm_atomic_helper_crtc_state_reset();
__drm_atomic_helper_crtc_reset();
drm_atomic_helper_crtc_reset();
__drm_atomic_helper_crtc_duplicate_state();
drm_atomic_helper_crtc_duplicate_state();
__drm_atomic_helper_crtc_destroy_state();
drm_atomic_helper_crtc_destroy_state();

__drm_atomic_helper_plane_state_reset();
__drm_atomic_helper_plane_reset();
drm_atomic_helper_plane_reset();
__drm_atomic_helper_plane_duplicate_state();
drm_atomic_helper_plane_duplicate_state();
__drm_atomic_helper_plane_destroy_state();
drm_atomic_helper_plane_destroy_state();

__drm_atomic_helper_connector_state_reset();
__drm_atomic_helper_connector_reset();
drm_atomic_helper_connector_reset();
// drm_atomic_helper_connector_tv_reset();
// drm_atomic_helper_connector_tv_check();
// drm_atomic_helper_connector_tv_margins_reset();

__drm_atomic_helper_connector_duplicate_state();
drm_atomic_helper_connector_duplicate_state();

__drm_atomic_helper_connector_destroy_state();
drm_atomic_helper_connector_destroy_state();
__drm_atomic_helper_private_obj_duplicate_state();

__drm_atomic_helper_bridge_duplicate_state();
drm_atomic_helper_bridge_duplicate_state();
drm_atomic_helper_bridge_destroy_state();
__drm_atomic_helper_bridge_reset();
drm_atomic_helper_bridge_reset();
```

# drm_atomic.ch

```c++
inline drm_crtc_commit_get()
inline drm_crtc_commit_put()
drm_atomic_state_alloc();
drm_atomic_state_clear();
drm_atomic_state_get()
__drm_atomic_state_free();
inline drm_atomic_state_put()
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
```
