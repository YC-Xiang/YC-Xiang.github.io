![](https://xyc-1316422823.cos.ap-shanghai.myqcloud.com/component.drawio.png)

# Aggregate Driver

实现结构体:

```c++
struct component_master_ops {
	int (*bind)(struct device *master);
	void (*unbind)(struct device *master);
};
```

在 probe 函数中调用 component_match_add()来填充 component match list, 最后调用 component_master_add_with_match() register aggregate driver, remove 函数中调用 component_master_del() 来 unregister。

```c++
void component_match_add(struct device *parent, struct component_match **matchptr, int (*compare)(struct device*, void*), void *compare_data)

int component_master_add_with_match(struct device *parent, const struct component_master_ops *ops, struct component_match *match)
```

# Components Driver

实现结构体：

```c++
struct component_ops {
	int (*bind)(struct device *comp, struct device *master,
		    void *master_data);
	void (*unbind)(struct device *comp, struct device *master,
		       void *master_data);
};
```

在 probe 函数中调用 component_add() register component driver, remove 函数中调用 component_del() 来 ungister。

```c++
int component_add(struct device *dev, const struct component_ops *ops);
```
