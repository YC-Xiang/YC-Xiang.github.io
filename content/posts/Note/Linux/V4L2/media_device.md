# Media Controller devices

## 5.1.1 Abstract media device model

entity 用来抽象一个 media hardware block, 可以表示一个物理硬件设备(CMOS sensor), 也可以表示一个逻辑硬件设备(isp pipeline)等.

pad 用来在 entity 之间传输数据, 数据从 source pad 流向 sink pad.

## 5.1.2 Media device

`struct media_device`

注册 media_device 需要调用: `media_device_init()`, `media_device_register()`.

注销: `media_device_unregister()`, `media_device_cleanup()`.

```c
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

	/* notify callback list invoked when a new entity is registered */
	struct list_head entity_notify;

	/* Serializes graph operations. */
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

model: device model 名称.

serial:

bus_info: device location. 比如 rk isp 这个 field 为 platform:rkisp1.

hw_revision: 硬件版本.

## 5.1.3 Entities

`struct media_entity`

注册: `media_entity_pads_init()`, `media_device_register_entity()`.

注销: `media_device_unregister_entity()`.

```c
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

## 5.1.4 Interfaces

`struct media_interface`

目前只有一种类型的 interface: device node. 以`struct media_intf_devnode`表示.

注册: `media_devnode_create()`.

注销: `media_devnode_remove()`.

## 5.1.5 Pads

`struct media_pad`

```c
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

media_entity: 指向该 pad 所属的 entity.

index: 当前 pad 在 entity 中的索引.

num_links: 该 pad 的 link 数量.

sig_type: 该 pad 的信号类型.

flags: 该 pad 的标志. 有 MEDIA_PAD_FL_SINK, MEDIA_PAD_FL_SOURCE, MEDIA_PAD_FL_MUST_CONNECT.

pipe: 该 pad 所属的 pipeline.

## 5.1.6 Links

`struct media_link`

有两种类型的 link:

1. pad to pad link.

创建: `media_create_pad_link()`

注销: `media_entity_remove_links()`

2. interface to entity link.

创建: `media_create_intf_link()`

注销: `media_remove_intf_links()`

```c
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

## 5.1.7 Graph traversal

`struct media_graph`
