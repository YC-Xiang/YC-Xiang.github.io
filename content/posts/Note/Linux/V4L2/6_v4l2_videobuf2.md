```c
struct vb2_queue {
	unsigned int			type;
	unsigned int			io_modes;
	struct device			*dev;
	unsigned long			dma_attrs;
	unsigned int			bidirectional:1;
	unsigned int			fileio_read_once:1;
	unsigned int			fileio_write_immediately:1;
	unsigned int			allow_zero_bytesused:1;
	unsigned int		   quirk_poll_must_check_waiting_for_buffers:1;
	unsigned int			supports_requests:1;
	unsigned int			requires_requests:1;
	unsigned int			uses_qbuf:1;
	unsigned int			uses_requests:1;
	unsigned int			allow_cache_hints:1;
	unsigned int			non_coherent_mem:1;

	struct mutex			*lock;
	void				*owner;

	const struct vb2_ops		*ops;
	const struct vb2_mem_ops	*mem_ops;
	const struct vb2_buf_ops	*buf_ops;

	void				*drv_priv;
	u32				subsystem_flags;
	unsigned int			buf_struct_size;
	u32				timestamp_flags;
	gfp_t				gfp_flags;
	u32				min_queued_buffers;
	u32				min_reqbufs_allocation;

	struct device			*alloc_devs[VB2_MAX_PLANES];

	/* private: internal use only */
	struct mutex			mmap_lock;
	unsigned int			memory;
	enum dma_data_direction		dma_dir;
	struct vb2_buffer		**bufs;
	unsigned long			*bufs_bitmap;
	unsigned int			max_num_buffers;

	struct list_head		queued_list;
	unsigned int			queued_count;

	atomic_t			owned_by_drv_count;
	struct list_head		done_list;
	spinlock_t			done_lock;
	wait_queue_head_t		done_wq;

	unsigned int			streaming:1;
	unsigned int			start_streaming_called:1;
	unsigned int			error:1;
	unsigned int			waiting_for_buffers:1;
	unsigned int			waiting_in_dqbuf:1;
	unsigned int			is_multiplanar:1;
	unsigned int			is_output:1;
	unsigned int			is_busy:1;
	unsigned int			copy_timestamp:1;
	unsigned int			last_buffer_dequeued:1;

	struct vb2_fileio_data		*fileio;
	struct vb2_threadio_data	*threadio;

	char				name[32];
};
```

type: buffer的类型, enum v4l2_buf_type, vendor driver初始化时设置.  
io_modes: 支持的io模式, enum vb2_io_modes, vendor driver初始化时设置.  
dma_attrs: dma arrtributes, 可以使用dma-mapping.h中的一些宏设置.  
bidirectional: 是否支持dma双向传输.  
fileio_read_once/fileio_write_immediately:  
min_queued_buffers: 由driver初始化, 最少需要的buffer数.  
min_reqbufs_allocation: 由vb2_core_queue_init()初始化, 等于min_queued_buffers+1.  
alloc_devs:

min_queued_buffers是在driver队列中的buffer, 多的一个是userspace正在使用的buffer.  

done_list: 准备好dequeue给userspace的buffer链表, 通过vb2_buffer_done()添加到链表中.  

streaming: 是否在streaming, 在调用streamon ioctl后会置1.  
start_streaming_called: 是否调用过start_streaming.  
error:
waiting_for_buffers: 正在dqbuf ioctrl时会被置1.
waiting_in_dqbuf:
is_multiplanar:
is_output:
is_busy:
copy_timestamp:
last_buffer_dequeued:

</br>

用户层交互的buffer结构体:

```c
struct v4l2_buffer {
	__u32			index;
	__u32			type;
	__u32			bytesused;
	__u32			flags;
	__u32			field;
#ifdef __KERNEL__
	struct __kernel_v4l2_timeval timestamp;
#else
	struct timeval		timestamp;
#endif
	struct v4l2_timecode	timecode;
	__u32			sequence;
	__u32			memory;
	union {
		__u32           offset;
		unsigned long   userptr;
		struct v4l2_plane *planes;
		__s32		fd;
	} m;
	__u32			length;
	__u32			reserved2;
	union {
		__s32		request_fd;
		__u32		reserved;
	};
};
```

```c
struct vb2_buffer {
	struct vb2_queue	*vb2_queue;
	unsigned int		index;
	unsigned int		type;
	unsigned int		memory;
	unsigned int		num_planes;
	u64			timestamp;
	struct media_request	*request;
	struct media_request_object	req_obj;
	enum vb2_buffer_state	state;
	unsigned int		synced:1;
	unsigned int		prepared:1;
	unsigned int		copied_timestamp:1;
	unsigned int		skip_cache_sync_on_prepare:1;
	unsigned int		skip_cache_sync_on_finish:1;
	struct vb2_plane	planes[VB2_MAX_PLANES];
	struct list_head	queued_entry;
	struct list_head	done_entry;
};
```
