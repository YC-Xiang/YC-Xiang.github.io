
# Color Management

`drm_color_mgmt.c`, `drm_color_mgmt.h`

color management or color space 通过 drm_crtc 和 drm_plane 的相关 properties 来实现控制。

drm_crtc_enable_color_mgmt()  用来 create crtc 相关的 degamma lut, ctx, gamma lut 这些 properties，在 crtc 初始化过程中调用。

还有一个初始化函数 drm_mode_crtc_set_gamma_size() 可以用来 support legacy gamma 相关接口。

drm_plane_create_color_properties() create plane 相关的 yuv color coding, color range properties，在 plane 初始化过程中调用。

## Properties

具体看 drm_crtc_state 和 drm_plane_state 中对应的 fields.

crtc:

`DEGAMMA_LUT`, `DEGAMMA_LUT_SIZE`, `CTM`, `GAMMA_LUT`, `GAMMA_LUT_SIZE`

plane:

`COLOR_ENCODING`, `COLOR_RANGE`

## Userspace

之后 userspace 可以通过设置这些 properties 修改到 struct drm_crtc_state 和 struct drm_plane_state 中相关的 fields.
这样 driver 在硬件回调函数中可以获取这些 fields 从而控制硬件改动。

userspace 的例子：

```c++
struct drm_color_lut gamma_lut[256];

memset(gamma_lut, 0, sizeof(gamma_lut));
util_smpte_fill_lut(info->ncolors, gamma_lut);
drmModeCreatePropertyBlob(dev->fd, gamma_lut, sizeof(gamma_lut), &blob_id);

add_property_optional(dev, crtc_id, "GAMMA_LUT", blob_id);
```

# Blend

`drm_blend.c`, `drm_blend.h`

blend 通过 drm_plane 的 alpha, rotation, zpos, blend_mode 几个 properties 来控制。

## Properties

具体看 drm_plane_state 中对应的 fields.

`alpha`, `rotation`, `zpos`, `pixel blend mode`.

# Damage tracking

允许指定 framebuffer 中实际发生变化的矩形区域列表，只更新这些变化的区域，而不是整个 framebuffer。

减少数据传输量，一般用于网络或 USB 传输的虚拟设备，降低带宽需求和处理开销。

也有支持部分刷新的硬件设备。

## Properties

`FB_DAMAGE_CLIPS`
