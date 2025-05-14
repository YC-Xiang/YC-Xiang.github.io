---
title: Spi Subsystem
date: 2023-05-22 17:40:28
tags:
- Linux driver
categories:
- Linux driver
---

# SPI 协议介绍

CPOL:表示 SPICLK 的初始电平，0 为低电平，1 为高电平

CPHA:表示相位，即第一个还是第二个时钟沿采样数据，0 为第一个时钟沿，1 为第二个时钟沿

| CPOL | CPHA | 模式 | 含义                                           |
| ---- | ---- | ---- | ---------------------------------------------- |
| 0    | 0    | 0    | SPICLK 初始电平为低电平，在第一个时钟沿采样数据 |
| 0    | 1    | 1    | SPICLK 初始电平为低电平，在第二个时钟沿采样数据 |
| 1    | 0    | 2    | SPICLK 初始电平为高电平，在第一个时钟沿采样数据 |
| 1    | 1    | 3    | SPICLK 初始电平为高电平，在第二个时钟沿采样数据 |

我们常用的是模式 0 和模式 3，因为它们都是在上升沿采样数据，不用去在乎时钟的初始电平是什么，只要在上升沿采集数据就行。

![](https://xyc-1316422823.cos.ap-shanghai.myqcloud.com/20230522180216.png)

# SPI driver

## spi_controller

```c++
struct spi_controller {
	struct device	dev;
	struct list_head list;
	s16			bus_num;
	u16			num_chipselect;
	u16			dma_alignment;
	u32			mode_bits;
	u32			buswidth_override_bits;
	u32			bits_per_word_mask;
	u32			min_speed_hz;
	u32			max_speed_hz;
	u16			flags;
	bool			devm_allocated;
	union {
		bool			slave;
		bool			target;
	};
	size_t (*max_transfer_size)(struct spi_device *spi);
	size_t (*max_message_size)(struct spi_device *spi);
	struct mutex		io_mutex;
	struct mutex		add_lock;
	spinlock_t		bus_lock_spinlock;
	struct mutex		bus_lock_mutex;
	bool			bus_lock_flag;
	int			(*setup)(struct spi_device *spi);
	int (*set_cs_timing)(struct spi_device *spi);
	int			(*transfer)(struct spi_device *spi,
						struct spi_message *mesg);
	void			(*cleanup)(struct spi_device *spi);
	bool			(*can_dma)(struct spi_controller *ctlr,
					   struct spi_device *spi,
					   struct spi_transfer *xfer);
	struct device *dma_map_dev;
	struct device *cur_rx_dma_dev;
	struct device *cur_tx_dma_dev;
	bool				queued;
	struct kthread_worker		*kworker;
	struct kthread_work		pump_messages;
	spinlock_t			queue_lock;
	struct list_head		queue;
	struct spi_message		*cur_msg;
	struct completion               cur_msg_completion;
	bool				cur_msg_incomplete;
	bool				cur_msg_need_completion;
	bool				busy;
	bool				running;
	bool				rt;
	bool				auto_runtime_pm;
	bool				cur_msg_mapped;
	bool                            fallback;
	bool				last_cs_mode_high;
	s8				last_cs[SPI_CS_CNT_MAX];
	u32				last_cs_index_mask : SPI_CS_CNT_MAX;
	struct completion               xfer_completion;
	size_t				max_dma_len;
	int (*optimize_message)(struct spi_message *msg);
	int (*unoptimize_message)(struct spi_message *msg);
	int (*prepare_transfer_hardware)(struct spi_controller *ctlr);
	int (*transfer_one_message)(struct spi_controller *ctlr,
				    struct spi_message *mesg);
	int (*unprepare_transfer_hardware)(struct spi_controller *ctlr);
	int (*prepare_message)(struct spi_controller *ctlr,
			       struct spi_message *message);
	int (*unprepare_message)(struct spi_controller *ctlr,
				 struct spi_message *message);
	union {
		int (*slave_abort)(struct spi_controller *ctlr);
		int (*target_abort)(struct spi_controller *ctlr);
	};
	void (*set_cs)(struct spi_device *spi, bool enable);
	int (*transfer_one)(struct spi_controller *ctlr, struct spi_device *spi,
			    struct spi_transfer *transfer);
	void (*handle_err)(struct spi_controller *ctlr,
			   struct spi_message *message);
	const struct spi_controller_mem_ops *mem_ops;
	const struct spi_controller_mem_caps *mem_caps;
	struct gpio_desc	**cs_gpiods;
	bool			use_gpio_descriptors;
	s8			unused_native_cs;
	s8			max_native_cs;
	struct spi_statistics __percpu	*pcpu_statistics;
	struct dma_chan		*dma_tx;
	struct dma_chan		*dma_rx;
	void			*dummy_rx;
	void			*dummy_tx;
	int (*fw_translate_cs)(struct spi_controller *ctlr, unsigned cs);
	bool			ptp_sts_supported;
	unsigned long		irq_flags;
	bool			queue_empty;
	bool			must_async;
	bool			defer_optimize_message;
};
```

`list`: spi controller global list。

`bus_num`: SoC 上某个 spi controller 的编号。

`num_chipselect`: 用来表示 spi slaves 的编号，每个 slave 都有一个 chipselect signal。

`dma_alignment`: 有些 spi controller 对 dma buffer 有 alignment 要求，vendor driver 设置。

`mode_bits`: 用来表示 spi controller 支持的 mode，在 uapi/linux/spi/spi.h 中定义，vendor driver 设置。

`bits_per_word_mask`: 用来表示 spi controller 支持的 bits_per_word，vendor driver 设置。

`min/max_speed_hz`: 用来表示 spi controller 支持的 min/max speed，vendor driver 设置。

`flags`: 额外的 flags，支持的有：

`transfer`: 主要任务是向 queue 中增加 spi message, 由 spi core 提供，默认实现为 spi_queued_transfer。

`can_dma`: 是否支持 dma.

`queued`: 如果支持 spi core 的 queue 机制，transfer 函数由 spi core 提供，则 spi core 把 queued 置为 1.

`transfer_one_message`: 如果 vendor driver 没实现，默认实现为 spi_transfer_one_message.

```c++
#define SPI_CONTROLLER_HALF_DUPLEX	BIT(0)	/* Can't do full duplex */
#define SPI_CONTROLLER_NO_RX		BIT(1)	/* Can't do buffer read */
#define SPI_CONTROLLER_NO_TX		BIT(2)	/* Can't do buffer write */
#define SPI_CONTROLLER_MUST_RX		BIT(3)	/* Requires rx */
#define SPI_CONTROLLER_MUST_TX		BIT(4)	/* Requires tx */
#define SPI_CONTROLLER_GPIO_SS		BIT(5)	/* GPIO CS must select slave */
#define SPI_CONTROLLER_SUSPENDED	BIT(6)	/* Currently suspended */
#define SPI_CONTROLLER_MULTI_CS		BIT(7)
```

`max_transfer_size/max_message_size`: spi controller 支持的 max transfer size/max message size 回调，vendor driver 设置。

`setup`: 设定 spi controller mode, clk 等等。

`set_cs_timing`: 设定 cs timing 的回调。

## spi_device

## spi_transfer, spi_message

在 SPI 子系统中，用 spi_transfer 结构体描述一个传输，用 spi_message 管理整个传输。可以构造多个 spi_transfer 结构体，把它们放入一个 spi_message 里面。

SPI 传输时，发出 N 个字节，就可以同时得到 N 个字节。

- 即使只想读 N 个字节，也必须发出 N 个字节：可以发出 0xff
- 即使只想发出 N 个字节，也会读到 N 个字节：可以忽略读到的数据。

# SPI 设备树

**SPI Master**

必须的属性如下：

- address-cells：这个 SPI Master 下的 SPI 设备，需要多少个 cell 来表述它的片选引脚
- size-cells：必须设置为 0
- compatible：根据它找到 SPI Master 驱动

可选的属性如下：

- cs-gpios：SPI Master 可以使用多个 GPIO 当做片选，可以在这个属性列出那些 GPIO
- num-cs：片选引脚总数

**SPI Device**

必须的属性如下：

- compatible：根据它找到 SPI Device 驱动
- reg：用来表示它使用哪个片选引脚
- spi-max-frequency：必选，该 SPI 设备支持的最大 SPI 时钟

可选的属性如下：

- spi-cpol：这是一个空属性 (没有值)，表示 CPOL 为 1，即平时 SPI 时钟为低电平
- spi-cpha：这是一个空属性 (没有值)，表示 CPHA 为 1)，即在时钟的第 2 个边沿采样数据
- spi-cs-high：这是一个空属性 (没有值)，表示片选引脚高电平有效
- spi-3wire：这是一个空属性 (没有值)，表示使用 SPI 三线模式
- spi-lsb-first：这是一个空属性 (没有值)，表示使用 SPI 传输数据时先传输最低位 (LSB)
- spi-tx-bus-width：表示有几条 MOSI 引脚；没有这个属性时默认只有 1 条 MOSI 引脚
- spi-rx-bus-width：表示有几条 MISO 引脚；没有这个属性时默认只有 1 条 MISO 引脚
- spi-rx-delay-us：单位是毫秒，表示每次读传输后要延时多久
- spi-tx-delay-us：单位是毫秒，表示每次写传输后要延时多久

示例：

```c++
spi@f00 {
		#address-cells = <1>;
		#size-cells = <0>;
		compatible = "fsl,mpc5200b-spi","fsl,mpc5200-spi";
		reg = <0xf00 0x20>;
		interrupts = <2 13 0 2 14 0>;
		interrupt-parent = <&mpc5200_pic>;

		ethernet-switch@0 {
			compatible = "micrel,ks8995m";
			spi-max-frequency = <1000000>;
			reg = <0>;
		};

		codec@1 {
			compatible = "ti,tlv320aic26";
			spi-max-frequency = <100000>;
			reg = <1>;
		};
	};
```

# spidev 的使用 (SPI 用户态 API)

- 内核驱动：`drivers\spi\spidev.c`
- 内核提供的测试程序：`tools\spi\spidev_fdx.c`
- 内核文档：`Documentation\spi\spidev`

## spidev 驱动程序分析

内核驱动：`drivers\spi\spidev.c`

```c++
spidev_write();
	spidev_sync_write();
		spidev_sync();

spidev_read();
	spidev_sync_read();
		spidev_sync();

spidev_ioctl();
// 在应用层通过 ioctl(fd, SPI_IOC_MESSAGE(x), xfer) 来调用，进行 spi 传输。参考 spidev_fdx.c
```

## spidev 应用程序分析

内核提供的测试程序：`tools\spi\spidev_fdx.c`

使用方法：

spidev_fdx [-h] [-m N] [-r N] /dev/spidevB.D

h: 打印用法

m N：先写 1 个字节 0xaa，再读 N 个字节，**注意：**不是同时写同时读

r N：读 N 个字节

## spidev 缺点

使用 read、write 函数时，只能读、写，这是半双工方式。

使用 ioctl 可以达到全双工的读写。

但是 spidev 有 2 个缺点：

- 不支持中断
- 只支持同步操作，不支持异步操作：就是 read/write/ioctl 这些函数只能执行完毕才可返回

# Data structure and APIs

```c++
// include/linux/spi/spi.h
// 简易函数
static inline int spi_write(struct spi_device *spi, const void *buf, size_t len);
static inline int spi_read(struct spi_device *spi, void *buf, size_t len);
extern int spi_write_then_read(struct spi_device *spi, const void *txbuf, unsigned n_tx, void *rxbuf, unsigned n_rx);
static inline ssize_t spi_w8r8(struct spi_device *spi, u8 cmd);
static inline ssize_t spi_w8r16(struct spi_device *spi, u8 cmd);
static inline ssize_t spi_w8r16be(struct spi_device *spi, u8 cmd);

// 复杂函数
extern int spi_async(struct spi_device *spi, struct spi_message *message);
extern int spi_sync(struct spi_device *spi, struct spi_message *message);
static inline int spi_sync_transfer(struct spi_device *spi, struct spi_transfer *xfers, unsigned int num_xfers);
```

```c++
spi_write()
    spi_sync_transfer()
        spi_sync()
	    // 这里根据 queue_empty 决定 sync/async 传输，如果 queue_empty 则 sync, 否则 async
	    __spi_transfer_message_noqueue()/spi_async_locked()

// sync 处理
__spi_transfer_message_noqueue()
    __spi_pump_transfer_message()
        spi_map_msg()
	ctlr->transfer_one_message(ctlr, msg) // spi_transfer_one_message
	    spi_set_cs()
	    ctlr->transfer_one() // vendor driver 实现

// async 处理
spi_async_locked()
    ctlr->transfer() // spi_queued_transfer
        list_add_tail(&msg->queue, &ctlr->queue)
	kthread_queue_work(ctlr->kworker, &ctlr->pump_messages) // 进入内核线程，执行 spi_pump_messages_work
	// return

spi_pump_messages()
    __spi_pump_transfer_message(); // 这里就和 sync 处理一样了

```
