# Media Controller devices

### 5.1.1 Abstract media device model

entity 用来抽象一个 media hardware block, 可以表示一个物理硬件设备(CMOS sensor), 也可以表示一个逻辑硬件设备(isp pipeline)等.

pad 用来在 entity 之间传输数据, 数据从 source pad 流向 sink pad.

link 是两个 pad 之间 point-to-point 的连接.

### 5.1.2 Media device

`struct media_device`

注册 media_device 需要调用: `media_device_init()`, `media_device_register()`.

注销: `media_device_unregister()`, `media_device_cleanup()`.

```c++
struct media_device {
	struct device *dev;
	struct media_devnode *devnode;

	char model[32];
	char driver_name[32];
	char serial[40];
	char bus_info[32];
	u32 hw_revision;

	u64 topology_version;

	u32 id;
	struct ida entity_internal_idx;
	int entity_internal_idx_max;

	struct list_head entities;
	struct list_head interfaces;
	struct list_head pads;
	struct list_head links;

	struct list_head entity_notify;

	struct mutex graph_mutex;
	struct media_graph pm_count_walk;

	void *source_priv;
	int (*enable_source)(struct media_entity *entity,
			     struct media_pipeline *pipe);
	void (*disable_source)(struct media_entity *entity);

	const struct media_device_ops *ops;

	struct mutex req_queue_mutex;
	atomic_t request_id;
};
```

dev: 底层的 device parent.  
devnode: media device node 相关.  
driver_name: 可选的 driver 名称, 如果没设置则为 dev->driver->name.  
model: device model 名称. sysfs 中显示的名称.  
serial: 设备序列号, usb 设备才会设置.  
bus_info: device location. 比如 rk isp 这个 field 为 platform:rkisp1.  
hw_revision: 硬件版本. pci 和 usb 设备才会设置.  
topology_version: 拓扑版本, 每次 media_gobj 创建或者注销的时候都会增加 1.  
id: 内部维护的一个 media_gobj id 号, 每次创建都增加 1.  
entity_internal_idx:  
entity_internal_idx_max:  
entities: 注册的 entities 链表.  
interfaces: 注册的 interfaces 链表.  
pads: 注册的 pads 链表.  
links: 注册的 links 链表.  
entity_notify: 注册的 entity_notify 回调函数链表.  
graph_mutex:  
pm_count_walk:  
source_priv: 下面两个回调需要用到的 driver private data.  
enable_source: 使能 source entity 的回调函数, 没看到哪边使用了该回调.  
disable_source: 禁用 source entity 的回调函数, 没看到哪边使用了该回调.  
ops: 可选的 media_device_ops 操作函数.  
req_queue_mutex:  
request_id: 用于生成 request id, 每次 alloc request 的时候增加 1.

```c++
struct media_device_ops {
	int (*link_notify)(struct media_link *link, u32 flags,
			   unsigned int notification);
	struct media_request *(*req_alloc)(struct media_device *mdev);
	void (*req_free)(struct media_request *req);
	int (*req_validate)(struct media_request *req);
	void (*req_queue)(struct media_request *req);
};
```

link_notify: 当 link 状态发生变化时, 调用该回调函数.  
req_alloc: 当需要分配比 struct media_request 更大的结构体时实现该回调, 目前没看到有实现的.  
req_free: 和 req_alloc 相反.  
req_validate: validate request.  
req_queue: queue a validated request.

### 5.1.3 Entities

`struct media_entity`, 通常嵌入在更大的 v4l2_subdev, vide_device 结构体中.

注册: `media_entity_pads_init()`, `media_device_register_entity()`.

在 video_register_deivce()和 v4l2_device_register_subdev()中都会调用 media_device_register_entity()来注册 entity.

在初始化前需要设置好 entity->function, functions 定义在`media.h`中, 并且实现好 entity->ops

注销: `media_device_unregister_entity()`.

```c++
struct media_entity {
	struct media_gobj graph_obj;	/* must be first field in struct */
	const char *name;
	enum media_entity_type obj_type;
	u32 function;
	unsigned long flags;

	u16 num_pads;
	u16 num_links;
	u16 num_backlinks;
	int internal_idx;

	struct media_pad *pads;
	struct list_head links;

	const struct media_entity_operations *ops;

	int use_count;

	union {
		struct {
			u32 major;
			u32 minor;
		} dev;
	} info;
};
```

graph_obj: 底层的 media_gobj 结构体.  
name: entity name.  
obj_type: entity type, 有 BASE/VIDEO_DEVICE/SUBDEV, BASE 表示没有嵌入在其他结构体中.  
function: entity function, 定义在 media.h MEDIA_ENT_F_XXX 中, 需要 vendor driver 指定某一功能.  
flags: entity flags, 定义在 media.h MEDIA_ENT_FL_XXX 中, 需要 vendor driver 指定 flag, 指定 flag 的 driver 不多.  
num_pads: entity source pad 和 sink pad 的数量.  
num_links: entity 的 link 数量, 包括 forward link, back link, enabled 和 disabled link.  
num_backlinks: entity back link 的数量.  
internal_idx: 分配的 id 号.  
pads: pads 链表.  
links: links 链表.  
ops: entity 操作函数, 需要 vendor driver 实现.  
use_count: entity 的使用计数.  
info: device node 信息, 为了向后兼容性.

### 5.1.4 Interfaces

`struct media_interface`

目前只有一种类型的 interface: device node. 以`struct media_intf_devnode`表示.

注册: `media_devnode_create()`.

注销: `media_devnode_remove()`.

```c++
struct media_interface {
	struct media_gobj		graph_obj;
	struct list_head		links;
	u32				type;
	u32				flags;
};
```

graph_obj: 底层的 media_gobj 结构体.  
links: 和该 interface 连接的 link 链表.  
type: MEDIA_INTF_T_XXX, 定义在 media.h 中, 在 video_register_media_controller()中设置.  
flags: 目前没有定义任何 interface flag.

### 5.1.5 Pads

`struct media_pad`

```c++
struct media_pad {
	struct media_gobj graph_obj;
	struct media_entity *entity;
	u16 index;
	u16 num_links;
	enum media_pad_signal_type sig_type;
	unsigned long flags;
	struct media_pipeline *pipe;
};
```

graph_obj: 底层的 media_gobj 结构体.  
entity: 指向该 pad 所属的 entity.  
index: 当前 pad 在 entity 中的索引.  
num_links: 该 pad 的 link 数量.  
sig_type: 该 pad 的信号类型.  
flags: 该 pad 的标志. 有 MEDIA_PAD_FL_SINK, MEDIA_PAD_FL_SOURCE, MEDIA_PAD_FL_MUST_CONNECT.  
pipe: 该 pad 所属的 pipeline.

### 5.1.6 Links

`struct media_link`

有两种类型的 link:

1. pad to pad link.

创建: `media_create_pad_link()`

注销: `media_entity_remove_links()`

2. interface to entity link.

创建: `media_create_intf_link()`

注销: `media_remove_intf_links()`

```c++
struct media_link {
	struct media_gobj graph_obj;
	struct list_head list;
	union {
		struct media_gobj *gobj0;
		struct media_pad *source;
		struct media_interface *intf;
	};
	union {
		struct media_gobj *gobj1;
		struct media_pad *sink;
		struct media_entity *entity;
	};
	struct media_link *reverse;
	unsigned long flags;
	bool is_backlink;
};
```

### 5.1.7 Graph traversal

media framework 提供了一些搜索定位 entities, links 的方法:

```c++
media_device_for_each_entity(); // 遍历所有entity
media_device_for_each_intf(); // 遍历所有interface
media_device_for_each_pad(); // 遍历所有pad
media_device_for_each_link(); // 遍历所有link

media_entity_find_link(); // 根据source pad和sink pad找到对应link
media_pad_remote_pad_first(); // 传入pad, 寻找和当前pad连接的remote pad
media_entity_remote_source_pad_unique(); // 传入entity, 寻找一个remote source pad
media_pad_remote_pad_unique() // 寻找传入pad的unique remote pad, 如果有不止一个remote pad返回错误
```

### 5.1.10 Pipelines and media streams

driver应该在上层结构体中 embed struct media_pipeline, 通过每个 media_pad 中的 media_pipeline 指针来访问.

```c++
struct media_pipeline {
	bool allocated;
	struct media_device *mdev;
	struct list_head pads;
	int start_count;
};
```

`media_pipeline_start()`

### 5.1.11 Link validation

### 5.1.12 Pipeline traversal

当一条 pipeline 通过 `media_pipeline_start()` 构建好后, 就可以通过 `media_pipeline_for_each_entity()`,
`media_pipeline_for_each_pad()` 来遍历 entity 和 pad 了.

```c++
media_pipeline_pad_iter iter;
struct media_pad *pad;

media_pipeline_for_each_pad(pipe, &iter, pad) {
    /* 'pad' will point to each pad in turn */
    // ...
}

// 遍历entity还需要init和cleanup的步骤.
media_pipeline_entity_iter iter;
struct media_entity *entity;
int ret;

ret = media_pipeline_entity_iter_init(pipe, &iter);
if (ret)
    ...;

media_pipeline_for_each_entity(pipe, &iter, entity) {
    /* 'entity' will point to each entity in turn */
    ...
}

media_pipeline_entity_iter_cleanup(&iter);
```
