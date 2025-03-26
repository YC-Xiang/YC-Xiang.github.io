---
date: 2024-09-02T09:49:20+08:00
title: "DRM -- Object and Property"
tags:
  - DRM
categories:
  - DRM
---

![](https://xyc-1316422823.cos.ap-shanghai.myqcloud.com/property.drawio.png)

# Property

CRTCs, planes, connectors 都有各自的 properties(字符串到值的映射)。Userspace 通过设置这些 properties，即可完成对显示参数的设置。

目前只有 CRTCs, planes, connectors 三者有 properties，因此 Userspace 只能对该三者的 properties 进行设置。

DRM 中定义了一系列 standard properties，这些 properties 在每个平台上都会创建，比如 connector 的 standard properties 会通过 drm_connector_create_standard_properties() 在 connector init 过程中自动创建，其他还有 specific 的 properties 需要底层 driver 调用特定的函数来创建，比如 drm_mode_create_dvi_i_properties() 可以创建 select subconnector property。

standard property 保存在 drm_device->mode_config 中，specific property 需要调用各自的创建函数来创建，保存在 drm_crtc/connector/plane 中。

```c
struct drm_property {
	struct list_head head; // property链表
	struct drm_mode_object base;
	uint32_t flags;
	char name[DRM_PROP_NAME_LEN]; // property名称
	uint32_t num_values;
	uint64_t *values;
	struct drm_device *dev;
	struct list_head enum_list;
};
```

`flags`: property flags，需要是以下选项之一:

`DRM_MODE_PROP_RANGE`: property 是一个范围，value 包括一个 unsinged minimum 和 unsigned maximum。

`DRM_MODE_PROP_SIGNED_RANGE`: property 有符号的一个范围。

`DRM_MODE_PROP_ENUM`: property 是枚举类型。

`DRM_MODE_PROP_BITMASK`: property 是 bitmask 类型。

`DRM_MODE_PROP_OBJECT`：value 数组保存的是 drm_mode_object 的 id，目前只有 FB_ID 和 CRTC_ID 是这种类型。

`DRM_MODE_PROP_BLOB`: 存放自定义的结构体数据，典型的如 MODE_ID。

下面两个 flag 可以和上面的组合使用：

`DRM_MODE_PROP_ATOMIC`: 表示该 porperty 只有在 drm 应用程序支持 atomic 操作时才可使用。

`DRM_MODE_PROP_IMMUTABLE`: 表示该 property userspace 是只读的，只有 kernel 可以修改。

`values`: 该 property 对应的 value 值，根据不同 flags，数组中存放不同的内容

</br>

```c
struct drm_property_blob {
	struct drm_mode_object base;
	struct drm_device *dev;
	struct list_head head_global;
	struct list_head head_file;
	size_t length;
	void *data;
};
```

blob property 用于存放一些 u64 放不下的大型结构体数据，比如"MODE_ID", blob 类型只能由 kernel 改写，userspace 不能改动。

</br>

DRM 中提供了一些 standard 的 properties 保存在 drm_mode_config 中，其他一些 specific properties 会保存到 drm_connector, drm_plane 各自的结构体中。

在 connector/plane/crtc 初始化过程中，会把 mode_config 中的 properties 通过 drm_object_attach_property()保存到各结构体的 xxx->obj->properties 中。但在 atomic driver 中，xxx->base->properties 中保存的都是默认值和只读的 properties，而不会更新其中的值(只会更新 drm_xxx_state 中的值)。具体可看 struct drm_object_properties 的注释。

下面列出一些常用的 properties:

## CRTC

`ACTIVE`: 用于代替 connector 中的 DPMS。

`MODE_ID`：crtc 选择哪一种 mode，0 表示 disable。

`OUT_FENCE_PTR`：// todo: fence 机制相关，目前不清楚。

## Plane

`type`: primary/overlay/cursor。

`FB_ID`: object 类型 property, 该 plane 的 id。

`CRTC_ID`: object 类型 property, 该 plane 对应的 crtc id。

`SRC_X`, `SRC_Y`: framebuffer 中 pixels source 的起始 x,y 坐标。

`SRC_W`, `SRC_H`: framebuffer 中 pixels source 的宽和高。

`CRTC_X`, `CRTC_Y`: 显示 destination 的起始 x,y 坐标。

`CRTC_W`, `CRTC_H`: 显示 destination 的宽和高。

`IN_FENCE_FD`：// todo: fence 机制相关，目前不清楚。

## Connector

drm_connector_create_standard_properties 函数前注释介绍了 connector 的 standard properties:

`EDID`: Extended Display Identification Data。BLOB+IMMUTABLE 类型 property，保存一些固有信息，kernel 可以通过 drm_get_edid() 获取 edid,并会调用 drm_connector_update_edid_property() 设置该 property，userspace 不可设置。

`DPMS`: Display Power Management Signaling。用来表示 connector power state。 legacy property, atomic driver 不必考虑，被 crtc 的 ACTIVE property 代替了。

`PATH`: BLOB+IMMUTABLE 类型，dp mst(dp multi-stream transport 多路显示) 需要的 property。

`TILE`: BLOB+IMMUTABLE 类型，用于标识当前 connector 是否应用于多屏拼接场景。kernel 通过 drm_connector_set_tile_property()来更新。

`link-status`: 连接状态，0:good, 1:bad。

`CRTC_ID`: object 类型 property, 该 connector 对应的 crtc id。

# Object

base KMS object。每个 drm mode 包括 drm_ctrc, drm_connector, drm_framebuffer 等结构体中都会有一个`drm_mode_object`数据结构，其中保存了该 crtc/connector/fb/plane 的 id 用于 id tracking 以及对应的 property 属性等。

</br>

drm_mode_object 结构体：

```c
struct drm_mode_object {
    uint32_t id;
    uint32_t type;
    struct drm_object_properties *properties;
    struct kref refcount;
    void (*free_cb)(struct kref *kref);
}
```

`id`: 用户空间操作的 id。

`type`: DRM_MODE_OBJECT_XXX, 包括 DRM_MODE_OBJECT_CRTC/CONNECTOR/ENCODER/MODE/PROPERTY/FB/BLOB/PLANE/ANY

<p class="note note-info">Userspace只能对CONNECTOR, CRTC, PLANE三种type的object property进行设置</p>

`*properties`: attach 到该 object 的 properties。

`refcount`: 如果 free_cb 回调存在的话，表示该 object 引用次数。

`free_cb`: 存在的话表示该 object 拥有 dynamic lifetime。

</br>

drm_object_properties 结构体：

```c
struct drm_object_properties {
	int count;
	struct drm_property *properties[DRM_OBJECT_MAX_PROPERTY];
	uint64_t values[DRM_OBJECT_MAX_PROPERTY];
};
```

`count`: 该 object 下面 attach 了多少个 properties。

`*properties[DRM_OBJECT_MAX_PROPERTY]`: properties 数组。

`values[DRM_OBJECT_MAX_PROPERTY]`: properties 对应的 value，注意 atomic driver 不会把 mutable properties (没有 DRM_MODE_PROP_IMMUTABLE flag) 的 value 存到这里，因此不要对可变的 properties 使用 drm_object_property_set_value()和 drm_object_property_get_value()函数。对于 atomic driver 来说，只有 default value 会保存在这里，也因此 IMMUTABLE(只读的) property 是可以调用前面两个函数的。

</br>

drm_mode_object 的一些操作函数：

```c
struct drm_mode_object *__drm_mode_object_find(struct drm_device *dev,
					       struct drm_file *file_priv,
					       uint32_t id, uint32_t type);
void drm_mode_object_put(struct drm_mode_object *obj);
void drm_mode_object_get(struct drm_mode_object *obj);
void drm_object_attach_property(struct drm_mode_object *obj,
				struct drm_property *property,
				uint64_t init_val);
int drm_object_property_set_value(struct drm_mode_object *obj,
				  struct drm_property *property, uint64_t val)
int drm_object_property_get_value(struct drm_mode_object *obj,
				  struct drm_property *property, uint64_t *val)
int drm_object_property_get_default_value(struct drm_mode_object *obj,
					  struct drm_property *property,
					  uint64_t *val)
```

`drm_mode_object_find`: 根据传入的 id 和 type,返回对应的 drm_mode_object。

`drm_mode_object_put`: refcount 减 1，如果等于 0 的话调用 object 的 free_cb 回调。

`drm_mode_object_get`: refcount 加 1。

`drm_object_attach_property`: 将 property attach 到 object->properties，drm_object_properties 的 value 初始值为 init_val

`drm_object_property_set_value`: 设置 object 对应的 property 的 value 为 val，atomic driver 不必调用这个接口。

`drm_object_property_get_value`：获取 object 对应 property 的 value 保存到 val。

`drm_object_property_get_default_value`: 获取 object 对应 property 的 value。只对 atomic driver 生效。
