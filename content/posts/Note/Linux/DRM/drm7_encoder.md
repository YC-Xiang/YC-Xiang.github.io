---
date: 2024-09-18T17:22:56+08:00
title: "Drm -- Encoder"
tags:
  - DRM
categories:
  - DRM
draft:
  - true
---

```c
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

```c
struct drm_encoder {
  struct drm_device *dev;
  struct list_head head;
  struct drm_mode_object base;
  char *name;
  int encoder_type; /// DRM_MODE_ENCODER_<foo>
  unsigned index;
  uint32_t possible_crtcs;
  uint32_t possible_clones;
  struct drm_crtc *crtc;
  struct list_head bridge_chain;
  const struct drm_encoder_funcs *funcs;
  const struct drm_encoder_helper_funcs *helper_private;
  struct dentry *debugfs_entry;
};
```

```c
struct drm_encoder_helper_funcs {
  void (*dpms)(struct drm_encoder *encoder, int mode);
  enum drm_mode_status (*mode_valid)(struct drm_encoder *crtc,
             const struct drm_display_mode *mode);
  bool (*mode_fixup)(struct drm_encoder *encoder,
         const struct drm_display_mode *mode,
         struct drm_display_mode *adjusted_mode);
  void (*prepare)(struct drm_encoder *encoder);
  void (*commit)(struct drm_encoder *encoder);
  void (*mode_set)(struct drm_encoder *encoder,
       struct drm_display_mode *mode,
       struct drm_display_mode *adjusted_mode);
  void (*atomic_mode_set)(struct drm_encoder *encoder,
        struct drm_crtc_state *crtc_state,
        struct drm_connector_state *conn_state);
  enum drm_connector_status (*detect)(struct drm_encoder *encoder,
              struct drm_connector *connector);
  void (*atomic_disable)(struct drm_encoder *encoder,
             struct drm_atomic_state *state);
  void (*atomic_enable)(struct drm_encoder *encoder,
            struct drm_atomic_state *state);
  void (*disable)(struct drm_encoder *encoder);
  void (*enable)(struct drm_encoder *encoder);
  int (*atomic_check)(struct drm_encoder *encoder,
          struct drm_crtc_state *crtc_state,
          struct drm_connector_state *conn_state);
};
```

`dpms`: optional, legacy support.

`mode_valid`: optional, 检查 userspace 传入的 drm_display_mode 是否满足 encoder 的约束

`mode_fixup`: optional, 修复 drm_display_mode 保存到 adjusted_mode

`prepare`: optional, legacy support.

`commit`: optional, legacy support.
