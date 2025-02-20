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

user_formats: rts_camera支持的所有格式, 保存在m_rtscam_soc_formats, 会在

user_format: 当前userspace设置的format格式, fourcc code.
user_width/user_height: 当前userspace设置的width/height.
bytesperline:
sizeimage:


streaming: 表示是否在streaming, 在rtscam_video_streamon中置1.


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
