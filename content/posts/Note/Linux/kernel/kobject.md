# 概述

include/linux/kobject.h

lib/kobject.c

Kobject 是基本数据类型，每个 Kobject 都会在"/sys/“文件系统中以目录的形式出现。

Ktype 代表 Kobject 的属性操作集合 (由于通用性，多个 Kobject 可能共用同一个属性操作集，因此把 Ktype 独立出来了).

Kset 是一个特殊的 Kobject（因此它也会在"/sys/“文件系统中以目录的形式出现），它用来集合相似的 Kobject（这些 Kobject 可以是相同属性的，也可以不同属性的）。

# 数据结构

kobject:

```c++
struct kobject {
	const char		*name;
	struct list_head	entry;
	struct kobject		*parent;
	struct kset		*kset;
	const struct kobj_type	*ktype;
	struct kernfs_node	*sd; /* sysfs directory entry */
	struct kref		kref;

	unsigned int state_initialized:1;
	unsigned int state_in_sysfs:1;
	unsigned int state_add_uevent_sent:1;
	unsigned int state_remove_uevent_sent:1;
	unsigned int uevent_suppress:1;
};
```

name: kobject 的名称, 也是在 sysfs 中的目录名.

entry: 用于将 kobject 链接到 kset 的链表节点.

parent: 指向父 kobject.

kset: 指向 kset. 如果 kobject 存在，且没有指定 parent，则会把 Kset 作为 parent（别忘了 Kset 是一个特殊的 Kobject）。

ktype: 该 Kobject 属于的 kobj_type. 每个 Kobject 必须有一个 ktype, 否则 Kernel 会提示错误.

sd: 该 Kobject 在 sysfs 中的表示.

kref: 引用计数.

state_initialized: 表示 kobject 是否已经初始化.  
state_in_sysfs: 表示 kobject 是否在 sysfs 中.  
state_add_uevent_sent:  
state_remove_uevent_sent:  
uevent_suppress: 如果该字段为 1，表示忽略所有上报的 uevent 事件。

</br>

Kset:

```c++
struct kset {
	struct list_head list;
	spinlock_t list_lock;
	struct kobject kobj;
	const struct kset_uevent_ops *uevent_ops;
} __randomize_layout;
```

list/list_lock: 保存该 kset 下所有的 kobject 的链表.

kobj: 该 kset 自己的 kobject（kset 是一个特殊的 kobject，也会在 sysfs 中以目录的形式体现）.

uevent_ops: 该 kset 的 uevent 操作集合.

</br>

ktype:

```c++
struct kobj_type {
	void (*release)(struct kobject *kobj);
	const struct sysfs_ops *sysfs_ops;
	const struct attribute_group **default_groups;
	const struct kobj_ns_type_operations *(*child_ns_type)(const struct kobject *kobj);
	const void *(*namespace)(const struct kobject *kobj);
	void (*get_ownership)(const struct kobject *kobj, kuid_t *uid, kgid_t *gid);
};
```

release: 通过该回调函数，可以将包含该种类型 kobject 的数据结构的内存空间释放掉.

sysfs_ops: 该 kobject 的 sysfs 操作集合.

default_groups: 该 kobject 的默认属性组.

child_ns_type: 和文件系统（sysfs）的命名空间有关.

namespace: 和文件系统（sysfs）的命名空间有关.

get_ownership: 该 kobject 的拥有者.

# 函数

kobject api:

```c++
int kobject_add(struct kobject *kobj, struct kobject *parent, const char *fmt, ...);
int kobject_init_and_add(struct kobject *kobj, const struct kobj_type *ktype, struct kobject *parent, const char *fmt, ...);
struct kobject kobject_create_and_add(const char *name, struct kobject *parent);
struct kobject *kobject_get(struct kobject *kobj);
void kobject_put(struct kobject *kobj);
```

Kobject 使用流程:

1. 定义一个 struct kset 类型的指针，并在初始化时为它分配空间，添加到内核中
2. 根据实际情况，定义自己所需的数据结构原型，该数据结构中包含有 Kobject
3. 定义一个适合自己的 ktype，并实现其中回调函数
4. 在需要使用到包含 Kobject 的数据结构时，动态分配该数据结构，并分配 Kobject 空间，添加到内核中
5. 每一次引用数据结构时，调用 kobject_get 接口增加引用计数；引用结束时，调用 kobject_put 接口，减少引用计数
6. 当引用计数减少为 0 时，Kobject 模块调用 ktype 所提供的 release 接口，释放上层数据结构以及 Kobject 的内存空间

</br>

分配 Kobject 的方式有两种:

1. 通过 kmalloc 分配 kobject 所在的结构体内存, 定义好 struct kobj_type 以及其中的 release 回调函数等, 之后通过 kobject_init()和 kobject_add()添加进 kernel.

![](https://xyc-1316422823.cos.ap-shanghai.myqcloud.com/20241231145254.png)

2. 使用 kobject_create 创建, 其中包含了分配内存已经内置了一个 struct kobj_type.

![](https://xyc-1316422823.cos.ap-shanghai.myqcloud.com/20241231152009.png)

</br>

kobject_get(), kobject_put() 可以增加和减少 kobject 的计数, 当计数为 0 时, 调用 kobject_release 释放资源.

![](https://xyc-1316422823.cos.ap-shanghai.myqcloud.com/20241231154009.png)

</br>

最后 kset 也是一种特殊的 kobject, 提供了单独的 api 来初始化 kset. 也是两种方式, 第一种通过自行分配 kset 所在结构体内存, 调用 kset_register(). 第二种通过 kset_create_and_add() 创建, 其中包含了分配内存并且内置了一个 struct kobj_type.

```c++
void kset_init(struct kset *kset);
int kset_register(struct kset *kset);
void kset_unregister(struct kset *kset);
kset_create_and_add(const char *name, const struct kset_uevent_ops *u,
					       struct kobject *parent_kobj);
static inline struct kset *kset_get(struct kset *k);
static inline void kset_put(struct kset *k);
```

![](https://xyc-1316422823.cos.ap-shanghai.myqcloud.com/20241231155118.png)

# 参考

http://www.wowotech.net/device_model/kobject.html
