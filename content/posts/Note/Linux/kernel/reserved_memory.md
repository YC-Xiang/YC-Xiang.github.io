kernel boot 过程，设备树初始化 reserved-memory 节点：

```c++
early_init_fdt_scan_reserved_mem()
	fdt_scan_reserved_mem();
	fdt_init_reserved_mem();
```

```c++
static int __init fdt_scan_reserved_mem(void)
{
	int node, child;
	const void *fdt = initial_boot_params;

	node = fdt_path_offset(fdt, "/reserved-memory");

	fdt_for_each_subnode(child, fdt, node) {
		const char *uname;
		int err;

		if (!of_fdt_device_is_available(fdt, child))
			continue;

		uname = fdt_get_name(fdt, child, NULL);

		/// 存在 reg 属性的情况
		err = __reserved_mem_reserve_reg(child, uname);
		/// 不存在 reg 属性的情况，存在 size 属性
		if (err == -ENOENT && of_get_flat_dt_prop(child, "size", NULL))
			fdt_reserved_mem_save_node(child, uname, 0, 0);
	}
	return 0;
}

/// 先预留 rmem 的 base 和 size 都为 0
void __init fdt_reserved_mem_save_node(unsigned long node, const char *uname,
				      phys_addr_t base, phys_addr_t size)
{
	struct reserved_mem *rmem = &reserved_mem[reserved_mem_count];

	rmem->fdt_node = node;
	rmem->name = uname;
	rmem->base = base;
	rmem->size = size;

	reserved_mem_count++;
	return;
}
```

```c++
void __init fdt_init_reserved_mem(void)
{
	int i;

	for (i = 0; i < reserved_mem_count; i++) {
		struct reserved_mem *rmem = &reserved_mem[i];
		unsigned long node = rmem->fdt_node;
		int len;
		const __be32 *prop;
		int err = 0;
		bool nomap;

		nomap = of_get_flat_dt_prop(node, "no-map", NULL) != NULL;
		prop = of_get_flat_dt_prop(node, "phandle", &len);
		if (!prop)
			prop = of_get_flat_dt_prop(node, "linux,phandle", &len);
		if (prop)
			rmem->phandle = of_read_number(prop, len/4);

		if (rmem->size == 0)
			/// 在这里解析设备树，填充 rmem->base 和 rmem->size
			err = __reserved_mem_alloc_size(node, rmem->name,
						 &rmem->base, &rmem->size);
		if (err == 0) {
			/// 和 coherent.c, contiguous.c 中 RESERVEDMEM_OF_DECLARE 定义的 of_device_id 匹配
			/// 进入 rmem_dma_setup/rmem_cma_setup
			err = __reserved_mem_init_node(rmem);
			// ...
		}
	}
}

static int __init __reserved_mem_init_node(struct reserved_mem *rmem)
{
	extern const struct of_device_id __reservedmem_of_table[];
	const struct of_device_id *i;
	int ret = -ENOENT;

	/// 这边__reservedmem_of_table 是 coherent.c, contiguous.c 中通过 RESERVEDMEM_OF_DECLARE 定义的
	/// 如果 rmem 的 compatible 对应 shared-dma-pool，则进入 rmem_dma_setup 和 rmem_cma_setup
	for (i = __reservedmem_of_table; i < &__rmem_of_table_sentinel; i++) {
		reservedmem_of_init_fn initfn = i->data;
		const char *compat = i->compatible;

		if (!of_flat_dt_is_compatible(rmem->fdt_node, compat))
			continue;

		ret = initfn(rmem);
	}
	return ret;
}
```

`coherent.c`: dma memory

```c++
static int __init rmem_dma_setup(struct reserved_mem *rmem)
{
	unsigned long node = rmem->fdt_node;

	/// 如果有 reusable，那么是 cma memory，这里直接返回
	if (of_get_flat_dt_prop(node, "reusable", NULL))
		return -EINVAL;

#ifdef CONFIG_ARM
	/// arm platform dma memory 必须要有 no-map 属性
	if (!of_get_flat_dt_prop(node, "no-map", NULL)) {
		pr_err("Reserved memory: regions without no-map are not yet supported\n");
		return -EINVAL;
	}
#endif

#ifdef CONFIG_DMA_GLOBAL_POOL
	if (of_get_flat_dt_prop(node, "linux,dma-default", NULL)) {
		WARN(dma_reserved_default_memory,
		     "Reserved memory: region for default DMA coherent area is redefined\n");
		dma_reserved_default_memory = rmem;
	}
#endif

	rmem->ops = &rmem_dma_ops;
	pr_info("Reserved memory: created DMA memory pool at %pa, size %ld MiB\n",
		&rmem->base, (unsigned long)rmem->size / SZ_1M);
	return 0;
}
```

`contiguous.c`: cma memory

```c++
static int __init rmem_cma_setup(struct reserved_mem *rmem)
{
	unsigned long node = rmem->fdt_node;
	bool default_cma = of_get_flat_dt_prop(node, "linux,cma-default", NULL);
	struct cma *cma;
	int err;

	/// 如果没定义 reusable，或者有 no-map 属性，则返回错误
	if (!of_get_flat_dt_prop(node, "reusable", NULL) ||
	    of_get_flat_dt_prop(node, "no-map", NULL))
		return -EINVAL;

	err = cma_init_reserved_mem(rmem->base, rmem->size, 0, rmem->name, &cma);
	if (err) {
		pr_err("Reserved memory: unable to setup CMA region\n");
		return err;
	}
	/* Architecture specific contiguous memory fixup. */
	dma_contiguous_early_fixup(rmem->base, rmem->size);

	if (default_cma)
		dma_contiguous_default_area = cma;

	rmem->ops = &rmem_cma_ops;
	rmem->priv = cma;

	pr_info("Reserved memory: created CMA memory pool at %pa, size %ld MiB\n",
		&rmem->base, (unsigned long)rmem->size / SZ_1M);

	return 0;
}
```

可以看出 dma memory 和 cma memory 是互斥的，只会是其中一种。

</br>

driver 初始化 reserved memory, 以 dma memory 为例：

```c++
of_reserved_mem_device_init();
	rmem->ops->device_init(); // rmem_dma_device_init
```
