libisp api:

global api:

- rts_av_isp_init/cleanup
- rts_av_isp_start/stop
- rts_av_isp_get_status
- rts_av_isp_register/unregister/get_algo
- rts_av_isp_bind/unbind_algo
- rts_av_isp_register/unregister/get/check_sensor
- rts_av_isp_bind/unbind_sensor
- rts_av_isp_register_iq

mipi out api:

- rts_av_isp_set_mipiout
- rts_av_isp_get_mipiout

v4l2 control:

- rts_isp_v4l2_query_ctrl
- rts_isp_v4l2_query_menu
- rts_isp_v4l2_g_ctrl
- rts_isp_v4l2_s_ctrl
- rts_isp_v4l2_query_ext_ctrl
- rts_isp_v4l2_g_ext_ctrls
- rts_isp_v4l2_s_ext_ctrls
- rts_isp_v4l2_try_ext_ctrls

sensor:

private mask:

3A Setting:

3A Statis:

IQ tuning:

other:

```c++
struct isp_core {
	struct isp_mod_hash_table hash;
	struct isp_notify notify;
	struct v4l2_ctrl_handler ctrl_handler;
	struct isp_statis statis;
	struct isp_iq iq;

	int initialized:1;
	int running:1;
};
```

hash: hash table, 用来保存 modules.
notify:
ctrl_hander:
statis:
iq:
initialized: isp_core_init() 之后置 1.
running: isp_core_start() 之后置 1.

```c++
struct isp_mod {
	uint32_t id;
	const char *name;
	uint32_t owner_id;

	/* do not directly use these callbacks, use api in this file */
	int (*init)(struct isp_mod *mod);
	int (*cleanup)(struct isp_mod *mod);
	int (*hardware_init)(struct isp_mod *mod);
	int (*hardware_cleanup)(struct isp_mod *mod);
	int (*add_ctrl)(struct isp_mod *mod, void *phandler);
	int (*need_block)(struct isp_mod *mod, uint32_t id, void *data);
	struct isp_mod_action_info *exec_actions;
	size_t exec_actions_num;
	struct isp_mod_action_info *info_actions;
	size_t info_actions_num;
	struct isp_mod_action_info *notify_actions;
	size_t notify_actions_num;
	size_t size; /* size of the struct which we embedded in */
	int virtual:1;

	void *owner; /* auto assigned by framework after registered */

	/* private, only used for internal */
	struct isp_mod *next;
	int initialized:1;
	int hardware_initialized:1;
	int ctrl_added:1;
};
```

# Poll + Eventfd mechanism

核心结构体：

```c++
struct isp_poll {
	int efd;
	uint64_t time;
	/* if timer num become too large, change to heap or rbtree */
	struct avl_tree timers;
	struct isp_list pendings;
	struct isp_list watchers;

	struct isp_work_queue wq;

	isp_io_handle_t trig;

	int enable:1;
};
```

efd: epoll file descriptor, 用来监视 I/O events.
time:
timers: avl tree 用来管理 timer.
pendings:
watchers:
trig:

**Poll creation**

通过 isp_poll_create 创建 epoll:

```c++
int isp_poll_create(isp_poll_t *pp)
```

**I/O registration**

```c++
int isp_io_init(isp_io_handle_t *io, isp_poll_t p, int fd, isp_io_cb cb)
```

**Timer Registration**

```c++
int isp_timer_init(isp_timer_handle_t *timer, isp_poll_t p, isp_timer_cb cb, void *data)
```

eventfd 是一种轻量级进程间通信机制。

eventfd 比传统的管道、FIFO 或消息队列更轻量级：

- 只需要一个文件描述符
- 内核开销小
- 不需要额外的缓冲区管理

eventfd 可以与 epoll 无缝结合，可以像普通文件描述符一样被 epoll 监控。

Trigger 机制：

eventfd 被用作触发机制，用于唤醒事件循环。

工作队列通知：

eventfd 被用于工作队列完成通知，当工作线程完成任务时，通知主线程处理结果。

# Notify mechanism

# UDS mechanism

UDS (Unix Domain Socket) 通信机制是一个完整的客户端 - 服务器架构，用于实现进程间通信。

**服务器端流程**

系统启动时，调用 isp_uds_stream_alloc 创建 UDS 流
UDS 流通过 uds_listen 创建 Unix Domain Socket 服务器，监听 /var/run/rtsisp.sock
设置 uds_accept 作为读回调函数，等待客户端连接
当客户端连接时，uds_accept 接受连接，创建客户端流
客户端流处理客户端请求，执行相应的操作，并发送响应

**客户端端流程**

客户端通过 isp_uds_message_simple 或 isp_uds_message_process 发送请求
这些函数从连接池获取一个连接（如果没有可用连接，则创建新连接）
设置消息序列号，发送消息，等待响应
接收响应，检查序列号，处理响应数据
将连接返回到连接池（如果连接出错，则关闭连接）

连接时的回调链：epoll_wait → stream_callback → uds_accept
读写消息时的回调链：epoll_wait → stream_callback → stream_service_client

# Message mechanism

```c++
struct rts_isp_msg_hdr {
	__u32 sequence; /* set by internal */
	__u32 msg_len;
	__u32 ret_len;
	__u32 isp_id;
	__u32 mod_id;
	__u32 action;
	__s32 ret_val;
	__u16 reloc_pos;
	__u16 reloc_num;
};
```

sequence: 当前的 frame count  
msg_len: header + msg data 总长度  
ret_len:
isp_id:
mod_id: isplib 中的某个 mod  
action: mod 对应的 action 函数名称  
ret_val:
reloc_pos:
reloc_num:
