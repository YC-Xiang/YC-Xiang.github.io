---
date: 2024-09-02T09:49:20+08:00
title: "DRM -- Object and Property"
tags:
  - DRM
categories:
  - DRM
---

# Modeset Base Object Abstraction

![](https://xyc-1316422823.cos.ap-shanghai.myqcloud.com/20250327095631.png)

所有 KMS objects 的 base structure 是 `struct drm_mode_object`, 用来追踪 property.

property 可以通过 `drm_object_attach_property()` attach 到不同的 object 上。

## Object

每个 drm mode 包括 drm_crtc, drm_connector, drm_framebuffer 等结构体中都会有一个 `drm_mode_object` 数据结构，其中保存了该 crtc/connector/fb/plane 的 id, 用于 id tracking. 还有对应的 property 属性等。

</br>

drm_mode_object 结构体：

```c++
struct drm_mode_object {
    uint32_t id;
    uint32_t type;
    struct drm_object_properties *properties;
    struct kref refcount;
    void (*free_cb)(struct kref *kref);
}
```

`id`: 用户空间操作的 id.

`type`: DRM_MODE_OBJECT_XXX, 包括 DRM_MODE_OBJECT_CRTC/CONNECTOR/ENCODER/MODE/PROPERTY/FB/BLOB/PLANE/ANY

<p class="note note-info"> Userspace 只能对 CONNECTOR, CRTC, PLANE 三种 type 的 object property 进行设置 </p>

`properties`: drm object 用来追踪 property 的结构体。

`refcount`: 如果 free_cb 回调存在的话，表示该 object 引用次数。

`free_cb`: 存在的话表示该 object 拥有 dynamic lifetime.

</br>

drm_object_properties 结构体：

```c++
struct drm_object_properties {
	int count;
	struct drm_property *properties[DRM_OBJECT_MAX_PROPERTY];
	uint64_t values[DRM_OBJECT_MAX_PROPERTY];
};
```

`count`: 该 object 下面 attach 了多少个 properties.

`properties[DRM_OBJECT_MAX_PROPERTY]`: properties 数组。

`values[DRM_OBJECT_MAX_PROPERTY]`: properties 对应的 value.

注意 atomic driver 不会把 mutable properties (没有 DRM_MODE_PROP_IMMUTABLE flag) 的 value 存到这里，对于 atomic driver 不要对可变的 properties 使用 drm_object_property_set_value() 和 drm_object_property_get_value() 函数。对于 atomic driver 来说，只有 default value 会保存在这里，因此 IMMUTABLE(只读的) property 是可以调用前面两个函数的。

atomic driver 通过 drm_crtc/plane/connector_funcs.get/set_property() 函数把 property 存放到对应的 state structure 中。

</br>

drm_mode_object 的一些操作函数：

```c++
struct drm_mode_object *drm_mode_object_find(struct drm_device *dev,
		struct drm_file *file_priv,
		uint32_t id, uint32_t type)
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

`drm_mode_object_find`: 根据传入的 id 和 type, 返回对应的 drm_mode_object.

`drm_mode_object_put`: refcount 减 1, 如果等于 0 的话调用 object 的 free_cb 回调。

`drm_mode_object_get`: refcount 加 1.

`drm_object_attach_property`: 将 property attach 到 `object->properties`, drm_object_properties 的 value 初始值为 init_val

`drm_object_property_set_value`: 设置 object 对应的 property 的 value 为 val, atomic driver 不要调用这个接口。

`drm_object_property_get_value`：获取 object 对应 property 的 value 保存到 val. atomic driver 不要调用这个接口。

`drm_object_property_get_default_value`: 获取 object 对应 property 的 value. 只对 atomic driver 生效。

## Property

对于 atomic modeset，properties 是 userspace 唯一设置 modeset configuration 的方法。

CRTCs, planes, connectors 都有各自的 properties(字符串到值的映射). Userspace 通过设置这些 properties, 即可完成对显示参数的设置。

目前只有 CRTCs, planes, connectors 三者有 properties, 因此 Userspace 只能对该三者的 properties 进行设置。

DRM 中定义了一系列 standard properties, 这些 properties 在每个平台上都会创建，
比如 connector 的 standard properties 会通过 drm_connector_create_standard_properties()
在 connector init 过程中自动创建，其他还有 specific 的 properties 需要底层 driver 调用特定的函数来创建，
比如 drm_mode_create_dvi_i_properties() 可以创建 select subconnector property.

全局唯一的 property 保存在 `drm_device->mode_config` 中，
其他每个 crtc/plane/connector 不同的 property 放在各自的 drm_crtc/plane/connector 结构体中。

```c++
struct drm_property {
	struct list_head head;
	struct drm_mode_object base;
	uint32_t flags;
	char name[DRM_PROP_NAME_LEN];
	uint32_t num_values;
	uint64_t *values;
	struct drm_device *dev;
	struct list_head enum_list;
};
```

`head`: 保存在 mode_config.property_list 链表中的节点。

`flags`: property flags, 需要是以下选项之一：

DRM_MODE_PROP_RANGE: property 是一个范围，value 包括一个 unsigned minimum 和 unsigned maximum.  
DRM_MODE_PROP_SIGNED_RANGE: property 有符号的一个范围。  
DRM_MODE_PROP_ENUM: property 是枚举类型。  
DRM_MODE_PROP_BITMASK: property 是 bitmask 类型。  
DRM_MODE_PROP_OBJECT: value 数组保存的是 drm_mode_object 的 id, 目前只有 FB_ID 和 CRTC_ID 是这种类型。  
DRM_MODE_PROP_BLOB: 存放自定义的结构体数据，典型的如 MODE_ID.

下面两个 flag 可以和上面的组合使用：

DRM_MODE_PROP_ATOMIC: 表示该 property 只有在 drm 应用程序支持 atomic 操作时才可使用。
DRM_MODE_PROP_IMMUTABLE: 表示该 property userspace 是只读的，只有 kernel 可以修改。

`num_values`: value array 的长度。

`values`: 该 property 对应的 value 值，根据不同 flags, 数组中存放不同的内容。

`enum_list`: 对于 enum 和 bitmask 类型，保存的 struct drm_property_enum 链表。

</br>

```c++
struct drm_property_enum {
    uint64_t value;
    struct list_head head;
    char name[DRM_PROP_NAME_LEN];
};
```

该结构体对应 enum 和 bitmask property 的某个 entry。

value: enum entry 的值，对于 bitmask, value 保存的是 bitshift。  
head: 保存在 drm_property.enum_list 的链表节点。

</br>

```c++
struct drm_property_blob {
	struct drm_mode_object base;
	struct drm_device *dev;
	struct list_head head_global;
	struct list_head head_file;
	size_t length;
	void *data;
};
```

blob property 用于存放一些 u64 放不下的大型结构体数据，比如 "MODE_ID", blob 类型只能由 kernel 改写，userspace 不能改动。

`head_global`: mode_config.blob_list 链表中的节点。  
`head_file`: file_priv.blobs 链表中的节点。  
`length`: 该 blob 属性的 size。  
`data`: 真正的 data structure。

</br>

下面列出 CRTC, plane, connector 常用的 properties:

### CRTC

`ACTIVE`: 用于代替 connector 中的 DPMS.

`MODE_ID`: crtc 选择哪一种 display mode, 0 表示 disable.

`OUT_FENCE_PTR`：// todo: fence 机制相关，目前不清楚。

### Plane

`type`: primary/overlay/cursor.

`FB_ID`: object 类型 property, 该 plane 的 id.

`CRTC_ID`: object 类型 property, 该 plane 对应的 crtc id.

`SRC_X`, `SRC_Y`: framebuffer 中 pixels source 的起始 x, y 坐标。

`SRC_W`, `SRC_H`: framebuffer 中 pixels source 的宽和高。

`CRTC_X`, `CRTC_Y`: 显示 destination 的起始 x, y 坐标。

`CRTC_W`, `CRTC_H`: 显示 destination 的宽和高。

`IN_FENCE_FD`：// todo: fence 机制相关，目前不清楚。

### Connector

drm_connector_create_standard_properties 函数前注释介绍了 connector 的 standard properties:

init 过程中自动注册的 properties:

`EDID`: Extended Display Identification Data. BLOB+IMMUTABLE 类型 property, 保存一些固有信息，kernel 可以通过 drm_get_edid() 获取 edid, 并会调用 drm_connector_update_edid_property() 设置该 property, userspace 不可设置。

`DPMS`: Display Power Management Signaling. 用来表示 connector power state. legacy property, atomic driver 不必考虑，被 crtc 的 ACTIVE property 代替了。

`PATH`: BLOB+IMMUTABLE 类型，dp mst(dp multi-stream transport 多路显示) 需要的 property.

`TILE`: BLOB+IMMUTABLE 类型，用于标识当前 connector 是否应用于多屏拼接场景。kernel 通过 drm_connector_set_tile_property() 来更新。

`link-status`: 连接状态，0: good, 1: bad.

`non_desktop`:

`HDR_OUTPUT_METADATA`:

`CRTC_ID`: object 类型 property, 该 connector 对应的 crtc id.

需要主动调用函数注册的 properties:

`Content Protection`:

`HDCP Content Type`:

`max bpc`

# IOCTL

## DRM_IOCTL_MODE_OBJ_GETPROPERTIES

app 传入 drm object 的 id 和 type, 得到该 object 的 property 数量和 property id 数组指针，property value 数组指针。

```c++
struct drm_mode_obj_get_properties {
	__u64 props_ptr;
	__u64 prop_values_ptr;
	__u32 count_props;
	__u32 obj_id;
	__u32 obj_type;
};
```

app 传入：

obj_id: drm object id.  
obj_type: drm object type.  

kernel 返回：

props_ptr: property id 数组指针。  
pro_values_ptr: property value 数组指针。  
count_props: property 数量。

## DRM_IOCTL_MODE_GETPROPERTY

app 传入 property id, 返回某个 property 的信息。

```c++
struct drm_mode_get_property {
	__u64 values_ptr;
	__u64 enum_blob_ptr;
	__u32 prop_id;
	__u32 flags;
	char name[DRM_PROP_NAME_LEN];
	__u32 count_values;
	__u32 count_enum_blobs;
};
```

app 传入：

prop_id: property id.

kernel 返回：

flags: property flag.  
name: property 的名称。  
values_ptr: value 的数组指针。  
count_values: value 的数量。  
enum_blob_ptr: 当 property 是 enum 或 bitmask 的时候，返回 drm_property_enum 结构体的数组指针。  
count_enum_blobs: drm_property_enum 的数量。

![](https://xyc-1316422823.cos.ap-shanghai.myqcloud.com/property.drawio.png)
