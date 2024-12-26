# Data Structure

```c
struct dma_buf_ops {
    bool cache_sgt_mapping;
    int (*attach)(struct dma_buf *, struct dma_buf_attachment *);
    void (*detach)(struct dma_buf *, struct dma_buf_attachment *);
    int (*pin)(struct dma_buf_attachment *attach);
    void (*unpin)(struct dma_buf_attachment *attach);
    struct sg_table * (*map_dma_buf)(struct dma_buf_attachment *,
		     enum dma_data_direction);
    void (*unmap_dma_buf)(struct dma_buf_attachment *,
		  struct sg_table *,
		  enum dma_data_direction);
    void (*release)(struct dma_buf *);
    int (*begin_cpu_access)(struct dma_buf *, enum dma_data_direction);
    int (*end_cpu_access)(struct dma_buf *, enum dma_data_direction);
    int (*mmap)(struct dma_buf *, struct vm_area_struct *vma);
    int (*vmap)(struct dma_buf *dmabuf, struct iosys_map *map);
    void (*vunmap)(struct dma_buf *dmabuf, struct iosys_map *map);
};
```

`cache_sgt_mapping`, `.pin`, `.unpin`是用来 dynamic dma-buf mapping 的, 暂时不关注.

`attach`: 建立一个 dma-buf 与 device 的连接关系，这个连接关系被存放在新创建的 dma_buf_attachment 对象中，供后续调用 dma_buf_map_attachment() 使用.

`detach`: 断开 dma-buf 与 device 的连接关系，并释放 dma_buf_attachment 对象.

`map_dma_buf`: 两件事, 1.获取 dma_buf 内存 buffer 的 sg_table, 2. 同步 cache.

`unmap_dma_buf`: 与 map_dma_buf 相反.

`release`: 释放 dma-buf 对象.

`begin_cpu_access`: 如果在 map_dma_buf 之后 cpu 还需要访问 buffer, 那么在 CPU 对 dma-buf 的访问前, 需要调用来 invalid cache.

`end_cpu_access`: 如果在 map_dma_buf 之后 cpu 还需要访问 buffer, 那么在 CPU 对 dma-buf 的访问后, 需要调用来 flush cache.

`mmap`: 将 dma-buf 映射到用户空间.

`vmap`: 映射 dma-buf 到 kernel 虚拟地址.

`vunmap`: 与 vmap 相反.

# API

```c
int dma_buf_fd(struct dma_buf *dmabuf, int flags); // exporter从dma buf导出为fd
struct dma_buf *dma_buf_get(int fd); // importer从fd获取到dma buf, 并增加引用计数

void dma_buf_put(struct dma_buf *dmabuf); // 减少dma buf的引用计数
void get_dma_buf(struct dma_buf *dmabuf) // 增加dma buf的引用计数
```

importer 拿到 dma-buf 后就可以利用下面的 api:

```c
struct dma_buf_attachment *dma_buf_attach(struct dma_buf *dmabuf,
					  struct device *dev)
void dma_buf_detach(struct dma_buf *dmabuf, struct dma_buf_attachment *attach);
struct sg_table *dma_buf_map_attachment(struct dma_buf_attachment *attach,
		    enum dma_data_direction direction);
void dma_buf_unmap_attachment(struct dma_buf_attachment *attach,
		struct sg_table *sg_table,
		enum dma_data_direction direction);
int dma_buf_begin_cpu_access(struct dma_buf *dma_buf,
			     enum dma_data_direction dir);
int dma_buf_end_cpu_access(struct dma_buf *dma_buf,
	       enum dma_data_direction dir);
int dma_buf_mmap(struct dma_buf *, struct vm_area_struct *,
	 unsigned long);
int dma_buf_vmap(struct dma_buf *dmabuf, struct iosys_map *map);
void dma_buf_vunmap(struct dma_buf *dmabuf, struct iosys_map *map);
```

# Exporter

实现一个 exporter 驱动, 首先需要实现`dma_buf_ops`结构体, 实现回调函数, 以`drm_prime.c`这个 export driver 为例:

```c
static const struct dma_buf_ops drm_gem_prime_dmabuf_ops =  {
	.cache_sgt_mapping = true,
	.attach = drm_gem_map_attach,
	.detach = drm_gem_map_detach,
	.map_dma_buf = drm_gem_map_dma_buf,
	.unmap_dma_buf = drm_gem_unmap_dma_buf,
	.release = drm_gem_dmabuf_release,
	.mmap = drm_gem_dmabuf_mmap,
	.vmap = drm_gem_dmabuf_vmap,
	.vunmap = drm_gem_dmabuf_vunmap,
};
```

其中.map_dma_buf, .unmap_dma_buf, .release 是必须要实现的.

</br>

接着需要实现`dma_buf_export_info`结构体. 这边也可以借助 DEFINE_DMA_BUF_EXPORT_INFO 宏.

```c
struct dma_buf_export_info exp_info = {
	.exp_name = KBUILD_MODNAME, /* white lie for debug */
	.owner = dev->driver->fops->owner,
	.ops = &drm_gem_prime_dmabuf_ops,
	.size = obj->size,
	.flags = flags,
	.priv = obj,
	.resv = obj->resv,
};
```

最后利用`dma_buf_export()`填充 `struct dma_buf`, 通过`dma_buf_fd()`函数生成 fd.

# Importer

## Userspace importer

通过 exporter export 出来的 dma-buf fd, 调用 ioctl 获取 dma-buf.

之后就可以使用上面的 importer api 了.

## Kernel space importer

通过`dma_buf_get(int fd)`从 fd 获取`struct dma_buf`.

之后就可以使用上面的 importer api 了.
