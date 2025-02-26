```c
struct rtscam_soc_dev {
        struct device *dev;
        void __iomem *base;
        unsigned long iostart;
        unsigned int iosize;
        int initialized;
        atomic_t init_count;
        const struct vb2_mem_ops *mem_ops;
        struct rtscam_sensor_fps sensor_fps;
        struct rtscam_video_device rvdev;
        struct rtscam_soc_slot_info slot_info[RTSCAM_MAX_STM_COUNT];
        struct rtscam_ge_device *mem_dev;
        struct rtscam_ge_device *cam_dev;
        struct rtscam_ge_device *ctrl_dev;
        struct rtscam_region td_config;
        struct rtscam_soc_icfg icfgs[RTSCAM_YUV_MAX_STRM_NUM];
        unsigned int icfg_count;
        struct rtscam_soc_rgbcfg rgbcfg;
        char name[PLATFORM_NAME_SIZE];
        kernel_ulong_t devtype;
        struct rtscam_mem_info *rtsmem;
        int pause_flag;
        unsigned long drop_frames;
        unsigned long drops[RTSCAM_MAX_STM_COUNT];
        int keep_user_setting;
        struct rtscam_subdev_t *subdev;
        struct rtscam_soc_video_in *video_in;
};
```

initialized: 是否初始化.
init_count: 初始化计数.
mem_ops: 内存操作回调函数, rtscam这边设置为自定义的rts_dma_contig_memops, linux内核可以直接设置为vb2_dma_contig_memops.
sensor_fps:
rvdev: 下面的 rts_video_device 设备.
slot_info: 每路流的 hw slot 信息.
mem_dev/cam_dev/ctrl_dev: ge device.  
td_config: 从设备树获取 td-config 节点, 不过没看到哪里有这个属性.  
icfgs: 从设备树获取的每路流的 y 和 uv 寄存器偏移地址和大小.  
icfg_count: 从设备树获取的 icfg 数量.  
rgbcfg: 从设备树获取的 rgb 相关寄存器偏移地址和大小.  
name: 设备名称.
devtype: chip 型号.  
rtsmem: 内存相关结构体.  
pause_flag: 所有 stream 状态. vendor ioctl control 可以设置起流停流.  
drop_frames: sysfs 可以设置 drop_frames, 有什么作用不太清楚.
drops: 每路流的 drop frames, 在 soc_s_stream() 时从 drop_frames 中复制过来.  
keep_user_setting:
subev: rtscam 中看起来就是 zoom device.  
video_in: video_in 设备.

```c
struct rtscam_video_stream {
	struct rtscam_video_device *icd;
	__u8 streamid;	/*initialized by user*/
	int video_nr;	/*initialized by user*/
	struct video_device *vdev;
	struct vb2_queue vb2_vidp;
	spinlock_t	lock;
	struct mutex stream_lock;
	struct mutex queue_lock;
	struct list_head capture;
	struct rtscam_video_format *user_formats;
	struct rtscam_video_fps fps;
	__u32 rts_code;
	__u32 user_format;
	__u32 user_width;
	__u32 user_height;
	__u32 bytesperline;
	__u32 sizeimage;
	unsigned long sequence;
	unsigned long frame_count;
	unsigned long skip_count;
	unsigned long overflow_count;
	unsigned long error_count;
	unsigned long abnormal_count;
	atomic_t use_count;
	struct file *memory_owner;
	unsigned long delay;
	unsigned long delta;
	int fov_status;
	__u8 vin_mode;
	__u32 ring_buf_height;
	__u8 streaming;
	unsigned int memory;
};
```

icd: back pointer to rtscam_video_device.  
stream_id: 流的id, 支持最多4路stream.  
video_nr: 设置流的最小的v4l2设备号, RGB在62~256范围内分配, yuv 在52~256范围内分配.  
vdev: 流对应的video_device.  
vb2_vidp: 流的vb2_queue.  
capture: 流自定义的链表, 用来保存qbuf的rtscam_video_buffer.  
user_formats: rts_camera支持的所有格式链表, 保存在m_rtscam_soc_formats, 在rtscam_register_format()中注册.  
fps:  
rtscode: userspace设置format的rts_code.  
user_format: userspace设置format的fourcc code.  
user_width/user_height: userspace设置format的width/height.  
bytesperline: userspace设置format的bytesperline, 每行字节数.  
sizeimage: userspace设置format的sizeimage, 一帧图像的字节数.  
sequence: 流的总frame count数.  
frame_count: 流的frame_count.  
skip_count: 流的skip_count.  
overflow_count: 流的overflow_count.  
error_count: 流的error_count.  
abnormal_count: 流的abnormal_count.  
use_count: 流的使用计数, 在open中+1, 在release中-1. 当stream的这个field好像没使用到.  
memory_owner: 在reqbufs时设定为当前file, 后面再复制给vma->vm_file.  
delay: 每路流之间处理的delay时间.  
delta: 每路流处理时间的差值.  
fov_status: fov mode.  
vin_mode: 默认为RTS_VIN_MODE_NORMAL, 可以通过ioctl VIDIOC_SET_VIN_MODE设置其他mode, 有RING_MEM mode.  
ring_buf_height: ring buffer的长度. 通过ioctl VIDIOC_SET_VIN_MODE设置.  
streaming: 表示是否在streaming, 在rtscam_video_streamon中置1.  
memory: 流的内存类型mmap/usrptr/dma_buf, 用户层通过ioctl VIDIOC_REQBUFS设置vb2->memory, 再拷贝到stream->memory.  

```c
struct rtscam_video_format {
	__u8 type;
	__u8 index;
	__u8 colorspace;
	__u8 field;
	const char name[32];
	__u32 fourcc;
	__u8 bpp;
	bool is_yuv;
	__u32 rts_code;
	__u8 initialized;
	__u8 frame_type;
	union {
		struct {
			struct rtscam_video_frame *frames;
		} discrete;
		struct {
			struct rtscam_frame_size max;
			struct rtscam_frame_size min;
			struct rtscam_frame_size step;
			struct rtscam_video_frmival frmival;
		} stepwise;
	};
	struct rtscam_video_format *next;
};
```

frame_type: 摄像头支持的分辨率类型的枚举值. enum rtscam_size_type, 支持RTSCAM_SIZE_DISCRETE/STEPWISE/CONTINUOUS.  
DISCRETE: 摄像头只支持特定的几种固定分辨率, 例如1920x1080, 1280x720等.  
STEPWISE: 摄像头支持在最小和最大分辨率之间以固定步长调整, 例如:最小 640x480,最大 1920x1080,步长 16x16.  
CONTINUOUS: 摄像头支持在最小和最大分辨率之间连续调整.

max: 最大分辨率的长和宽.
min: 最小分辨率的长和宽.
step: 长和宽的步长.
frmival:
