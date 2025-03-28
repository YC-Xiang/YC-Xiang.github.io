# DRM_IOCTL_MODE_OBJ_GETPROPERTIES

app 传入 drm object 的 id 和 type, 得到该 object 的 property 数量和 property id 数组指针, property value 数组指针.

```c++
struct drm_mode_obj_get_properties {
	__u64 props_ptr;
	__u64 prop_values_ptr;
	__u32 count_props;
	__u32 obj_id;
	__u32 obj_type;
};
```

app 传入:

obj_id: drm object id.  
obj_type: drm object type.  

kernel 返回:

props_ptr: property id 数组指针.  
pro_values_ptr: property value 数组指针.  
count_props: property 数量.  

# DRM_IOCTL_MODE_GETPROPERTY

app 传入 property id, 返回某个 property 的信息.

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

app 传入:

prop_id: property id.

kernel 返回:

flags: property flag.
name: property 的名称.
values_ptr: value 的数组指针.
count_values: value 的数量.
enum_blob_ptr: 当 property 是 enum 或 bitmask 的时候, 返回 drm_property_enum 结构体的数组指针.
count_enum_blobs: drm_property_enum 的数量.

# DRM_IOCTL_MODE_ATOMIC

```c++
struct drm_mode_atomic {
	__u32 flags;
	__u32 count_objs;
	__u64 objs_ptr;
	__u64 count_props_ptr;
	__u64 props_ptr;
	__u64 prop_values_ptr;
	__u64 reserved;
	__u64 user_data;
};
```

app:

flags: 可传入的 flags 参考 DRM_MODE_ATOMIC_FLAGS 宏, 有DRM_MODE_PAGE_FLIP_EVENT/DRM_MODE_PAGE_FLIP_ASYNC/DRM_MODE_ATOMIC_TEST_ONLY/DRM_MODE_ATOMIC_NONBLOCK/DRM_MODE_ATOMIC_ALLOW_MODESET.
count_objs: 要设置的 drm object 数量.  
objs_ptr: 要设置的 object id 数组指针.
count_props_ptr: 每个 object 要设置多少个 property 的数组指针.
props_ptr: 要设置的 property id 数组指针.
prop_values_ptr: 要设置的 property value 数组指针.
user_data: 用户传入的一个 data 指针值.

kernel:
