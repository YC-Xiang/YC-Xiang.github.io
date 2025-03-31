
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

flags: 可传入的 flags 参考 DRM_MODE_ATOMIC_FLAGS 宏，有 DRM_MODE_PAGE_FLIP_EVENT/DRM_MODE_PAGE_FLIP_ASYNC/DRM_MODE_ATOMIC_TEST_ONLY/DRM_MODE_ATOMIC_NONBLOCK/DRM_MODE_ATOMIC_ALLOW_MODESET.  
count_objs: 要设置的 drm object 数量。  
objs_ptr: 要设置的 object id 数组指针。  
count_props_ptr: 每个 object 要设置多少个 property 的数组指针。  
props_ptr: 要设置的 property id 数组指针。  
prop_values_ptr: 要设置的 property value 数组指针。  
user_data: 用户传入的一个 data 指针值。

kernel:
