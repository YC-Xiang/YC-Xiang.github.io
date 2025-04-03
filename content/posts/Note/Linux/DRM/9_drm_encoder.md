---
date: 2024-09-18T17:22:56+08:00
title: "DRM -- Encoder"
tags:
  - DRM
categories:
  - DRM
draft:
  - true
---

encoder 从 CRTC 拿到 pixel data，并将其转化为 connector 需要的 format。

encoder 通过 `drm_encoder_init()` 初始化，`drm_encoder_cleanup()` 清除。

```c++
struct drm_encoder_funcs {
  void (*reset)(struct drm_encoder *encoder);
  void (*destroy)(struct drm_encoder *encoder);
  int (*late_register)(struct drm_encoder *encoder);
  void (*early_unregister)(struct drm_encoder *encoder);
  void (*debugfs_init)(struct drm_encoder *encoder, struct dentry *root);
};
```

`reset`: reset encoder

`destroy`: drm_encoder_cleanup

`late_register`: 和 crtc 相关接口类似
`early_unregister`: 同上

`debugfs_init`: 注册 debugfs_init

```c++
struct drm_encoder {
  struct drm_device *dev;
  struct list_head head;
  struct drm_mode_object base;
  char *name;
  int encoder_type; /// DRM_MODE_ENCODER_<foo>
  unsigned index;
  uint32_t possible_crtcs;
  uint32_t possible_clones;
  struct list_head bridge_chain;
  const struct drm_encoder_funcs *funcs;
  const struct drm_encoder_helper_funcs *helper_private;
  struct dentry *debugfs_entry;
};
```

possible_clones: 用于定义哪些 encoder 可以与当前 encoder 同时工作，在多显示输出的场景中有用，比如需要在多个显示器上显示相同内容时。如果单显示输出置为 0 即可。

```c++
struct drm_encoder_helper_funcs {
  enum drm_mode_status (*mode_valid)(struct drm_encoder *crtc,
             const struct drm_display_mode *mode);
  void (*atomic_mode_set)(struct drm_encoder *encoder,
        struct drm_crtc_state *crtc_state,
        struct drm_connector_state *conn_state);
  void (*atomic_disable)(struct drm_encoder *encoder,
             struct drm_atomic_state *state);
  void (*atomic_enable)(struct drm_encoder *encoder,
            struct drm_atomic_state *state);
  int (*atomic_check)(struct drm_encoder *encoder,
          struct drm_crtc_state *crtc_state,
          struct drm_connector_state *conn_state);
};
```

mode_valid: optional, 检查 userspace 传入的 drm_display_mode 是否满足 encoder 的约束

atomic_mode_set:

atomic_disable: disable encoder.

atomic_enable: enable encoder.
