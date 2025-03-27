# Data Structure

```c++
struct software_node {
	const char *name;
	const struct software_node *parent;
	const struct property_entry *properties;
};
```

```c++

struct swnode {
	struct kobject kobj;
	struct fwnode_handle fwnode;
	const struct software_node *node;
	int id;

	/* hierarchy */
	struct ida child_ids;
	struct list_head entry;
	struct list_head children;
	struct swnode *parent;

	unsigned int allocated:1;
	unsigned int managed:1;
};
```

# Provider

software_node_register()注册单个 software node, software_node_register_node_group()注册一组 software nodes.
都是将 software nodes 挂入一个全局链表.

```c++
// 定义属性
static const struct property_entry my_device_properties[] = {
    PROPERTY_ENTRY_U32("reg", 0),
    PROPERTY_ENTRY_U32("my_property", 1234),
    { }
};

// 定义 software node
static const struct software_node my_device_node = {
    .name = "my-device",
    .properties = my_device_properties,
};

// 定义 software node 组
static const struct software_node *my_device_node_group[] = {
    &my_device_node,
    NULL
};

// 注册 software node 组
software_node_register_node_group(my_device_node_group);
```

device_add_software_node() 这个 api 可以设置 dev->fwnode = &swnode->fwnode.
这样后续 consumer 可以通过 dev_fwnode()获取到 fwnode_handle.

```c++
int device_add_software_node(struct device *dev, const struct software_node *node);
int device_create_managed_software_node(struct rts_device *dev,
					const struct property_entry *properties,
					const struct software_node *parent);
```

# Consumer

**方法 1**

通过 software_node_register_node_group 注册的 software nodes 需要传入 software node 到 software_node_fwnode()来获取 fwnode_handle, 才能使用 fwnode_xxx api.

```c++
struct fwnode_handle *fwnode;

static const struct software_node max17047_node = {
	.name = "max17047",
	.properties = max17047_properties,
};

fwnode = software_node_fwnode(&max17047_node);
```

也可以通过 name 来查找 software_node, 再通过 software_node_fwnode 获取 fwnode:

```c++
struct fwnode_handle *fwnode;
struct software_node *swnode;

swnode = software_node_find_by_name(NULL, "intel-xhci-usb-sw");
fwnode = software_node_fwnode(swnode);
```

**方法 2**

通过 device_add_software_node()和 device_create_managed_software_node()创建的 software node, 会设置 dev->fwnode, 将 device 和 fwnode_handle 联系起来.

这样直接通过 dev->fwnode 利用 fwnode_xxx api 就可以调用到底层 software_node 的回调函数.

</br>

通过 device_for_each_child_node 遍历 device 的 children node, 再通过 fwnode_property_read_u32 读取属性.

```c++
struct fwnode_handle *child;

device_for_each_child_node(dev, child) {
	if (fwnode_property_read_u32(child, "reg", &ch->reg)) {
		dev_err(dev, "channel without reg\n");
		fwnode_handle_put(child);
		return ERR_PTR(-EINVAL);
	}
}
```

通过 dev_fwnode 获取 fwnode, 再通过 fwnode_property_read_u32 读取属性.

```c++
struct fwnode_handle *fwnode;
fwnode = dev_fwnode(dev);

fwnode_property_read_u32(fwnode, "reg", &reg);
fwnode_property_read_u32(fwnode, "my_property", &my_property);
```
