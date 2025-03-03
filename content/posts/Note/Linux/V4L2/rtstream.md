//TODO: isp 架构图

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

```c
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

hash: hash table, 用来保存modules.
notify:
ctrl_hander:
statis:
iq:
initialized: isp_core_init() 之后置 1.
running: isp_core_start() 之后置 1.

```c
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

# Poll mechanism

核心结构体:

```c
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

efd: epoll file descriptor, 用来监视I/O events.
time:
timers: avl tree 用来管理timer.
pendings:
watchers:
trig:

**Poll creation**

通过isp_poll_create创建epoll:

```c
int isp_poll_create(isp_poll_t *pp)
```

**I/O registration**

```c
int isp_io_init(isp_io_handle_t *io, isp_poll_t p, int fd, isp_io_cb cb)
```

**Timer Registration**

```c
int isp_timer_init(isp_timer_handle_t *timer, isp_poll_t p, isp_timer_cb cb, void *data)
```

# Notify mechanism
