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
