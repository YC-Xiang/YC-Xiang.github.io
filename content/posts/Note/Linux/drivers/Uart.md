---
title: Uart Subsystem
date: 2023-05-15 22:25:00
tags:
- Linux driver
categories:
- Linux driver
---

## Reference

https://www.kernel.org/doc/Documentation/serial/driver

## Introduction

波特率 115200，bps 每秒传输的 bit 数。

每一位 1/115200 秒，传输 1byte 需要 10 位（start, data, stop）,那么每秒能传 11520byte。

115200，8n1。8:data，n:校验位不用，1：停止位。

![](https://xyc-1316422823.cos.ap-shanghai.myqcloud.com/20230522151159.png)

## TTY 体系中设备节点的差别

不关心终端是真实的还是虚拟的，都可以通过/dev/tty 找到当前终端。

**/dev/console**

内核的打印信息可以通过 cmdline 来选择打印到哪个设备。

console=ttyS0 console=tty \
console=ttyS0 时，/dev/console 就是 ttyS0 \
console=ttyN 时，/dev/console 就是/dev/ttyN \
console=tty 时，/dev/console 就是前台程序的虚拟终端 \
console=tty0 时，/dev/console 就是前台程序的虚拟终端 \
console 有多个取值时，使用最后一个取值来判断。

**/dev/tty 和/dev/tty0 区别**

`/dev/tty`表示当前进程的控制终端，也就是当前进程与用户交互的终端。

`/dev/tty0`则是当前所使用虚拟终端的一个别名

## Linux 串口应用编程

https://digilander.libero.it/robang/rubrica/serial.htm

```c++

struct termios options;

open("/dev/ttyS1", O_RDWR | O_NOCTTY | O_NDELAY)// O_NOCTTY: 不用作控制终端 O_NDELAY: 使 I/O 变成非阻塞模式


fcntl(fd, F_SETFL, 0): //读数据时，没有数据则阻塞等待
fcntl(fd, F_SETFL, FNDELAY): //读数据时不等待，没有数据就返回 0


/* c_cflag: Control Options */
options.c_cflag |= (CLOCAL | CREAD); // 必须打开 Enable the receiver and set local mode

options.c_cflag &= ~CSIZE; /* Mask the character size bits */
options.c_cflag |= CS8;    /* Select 8 data bits */

cfsetispeed(&options, B19200); //设置 input output 波特率
cfsetospeed(&options, B19200);

options.c_cflag &= ~PARENB // no parity
options.c_cflag &= ~CSTOPB

options.c_cflag &= ~CNEW_RTSCTS; // RTS CTS


/* c_lflag: Local Options */
options.c_lflag |= (ICANON | ECHO | ECHOE); // 选择规范输入 Canonical Input
options.c_lflag &= ~(ICANON | ECHO | ECHOE | ISIG); // 选择原始输入 Raw Input

/* c_iflag: Input Options*/
opt.c_iflag &= ~INPCK;

/* c_oflag: Input Options*/
options.c_oflag &= ~OPOST; // raw output. When the OPOST option is disabled, all other option bits in c_oflag are ignored.

tcsetattr(fd, TCSANOW, &options);
```

VMIN: 读数据时的最小字节数，没读到这些数据就不返回

VTIME: 等待第一个数据的时间，比如 VTIME=1，表示 10 秒内一个数据都没有的话就返回，如果 10 秒内至少读到一个字节，就继续等待，完全读到 VMIN 个数据返回。VTIME=0 表示一直等待。

**Timeouts are ignored in canonical input mode or when the \*NDELAY\* option is set on the file via \*open\* or \*fcntl\*.**

raw mode 可以设置 VMIN 和 VTIME，canonical input mode 不用设置。

## 8250 Uart driver

### 几个重要的结构体

**uart_driver**：通过`int uart_register_driver(struct uart_driver *drv)`注册进内核。

```c++
static struct uart_driver serial8250_reg = {
	.owner			= THIS_MODULE, ///拥有该 uart_driver 的模块，一般为 THIS_MODULE
	.driver_name	= "serial", ///串口驱动名，串口设备文件名以驱动名为基础
	.dev_name		= "ttyS", /// 串口设备名
	.major			= TTY_MAJOR,
	.minor			= 64,
	.cons			= SERIAL8250_CONSOLE, /// 其对应的 console.若该 uart_driver 支持 serial console，否则为 NULL
};

serial8250_reg.nr = UART_NR; /// 支持最大的串口数量 UART_NR == 3
```

**uart_port**：对应一个串口设备，通过`int uart_add_one_port
(struct uart_driver *drv,struct uart_port *port)`向该驱动添加`uart_port`\
**uart_ops**：串口操作函数，为 uart_port 的成员。\
**uart_8250_port**：8250 serial driver 抽象出来的结构体，包含了**uart_port**以及别的 8250 需要的成员。\
**dw8250_data**：dw 820 serial driver 抽象出来的结构体

注意到`uart_port`和`uart_ops`中有相同的回调函数，这里的调用顺序为：首先`uart_ops`会检查`uart_port`是否实现了相同的回调函数，如果实现了则调用`uart_port`，否则调用`uart_ops`。具体可以参考`8250_dw.c`中的`p->set_termios = dw8250_set_termios`和`8250_port.c`中的`.set_termios  = serial8250_set_termios`

```c++
struct uart_8250_port {
	struct uart_port	port;
	struct timer_list	timer;		/* "no irq" timer */
	struct list_head	list;		/* ports on this IRQ */
	u32			capabilities;	/* port capabilities */
	u16			bugs;		/* port bugs */
	unsigned int		tx_loadsz;	/* transmit fifo load size */
	unsigned char		acr;
	unsigned char		fcr;
	unsigned char		ier;
	unsigned char		lcr;
	unsigned char		mcr;
	unsigned char		cur_iotype;	/* Running I/O type */
	unsigned int		rpm_tx_active;
	unsigned char		canary;		/* non-zero during system sleep
						 *   if no_console_suspend
						 */
	unsigned char		probe;
	struct mctrl_gpios	*gpios;
#define UART_PROBE_RSA	(1 << 0)

	/*
	 * Some bits in registers are cleared on a read, so they must
	 * be saved whenever the register is read but the bits will not
	 * be immediately processed.
	 */
#define LSR_SAVE_FLAGS UART_LSR_BRK_ERROR_BITS
	u16			lsr_saved_flags;
	u16			lsr_save_mask;
#define MSR_SAVE_FLAGS UART_MSR_ANY_DELTA
	unsigned char		msr_saved_flags;

	struct uart_8250_dma	*dma;
	const struct uart_8250_ops *ops;

	u32			(*dl_read)(struct uart_8250_port *up);
	void			(*dl_write)(struct uart_8250_port *up, u32 value);
	struct uart_8250_em485 *em485;
	void			(*rs485_start_tx)(struct uart_8250_port *);
	void			(*rs485_stop_tx)(struct uart_8250_port *);
	struct delayed_work overrun_backoff;
	u32 overrun_backoff_time_ms;
};
```

timer: 使用 polling mode 时的 timer。\
list: ISA shared interrupts 需要把 shared interrupts 挂入一个链表。\
tx_loadsz: tx fifo load size.\
capabilities： UART_CAP_XXX.\
bugs: UART_BUG_XXX.

### 8250_core.c

- 通过 uart_register_driver() 注册 uart_driver
- 通过 serial8250_register_ports()->uart_add_one_port(). 注册 8250 所有的 uart_port（后面更底层的 8250_dw.c 会注册 uart_port 替换）
- 通过 univ8250_console_init() 注册 univ8250_console

```c++
//8250_core.c
struct uart_8250_port *up = &serial8250_ports[i];
struct uart_port *port = &up->port;

port->line = i;
port->ops = &serial8250_pops;
port->has_sysrq = IS_ENABLED(CONFIG_SERIAL_8250_CONSOLE);
up->cur_iotype = 0xFF;
port->ops = &univ8250_port_ops;

up->dl_read = default_serial_dl_read;
up->dl_write = default_serial_dl_write;
port->serial_in = io_serial_in;
port->serial_out = io_serial_out;
up->cur_iotype = p->iotype; /// p->iotype=0
univ8250_port_ops = *base_ops; /// 相当于又赋值回来 port->ops = &serial8250_pops;


serial8250_init(); /// 内核打印：Serial: 8250/16550 driver, 3 ports, IRQ sharing disabled
	serial8250_isa_init_ports();
    	serial8250_init_port(); // 8250_port.c
		serial8250_set_defaults();
    		set_io_from_upio();
    uart_register_driver();
    	tty_register_driver();
    serial8250_register_ports(); /// 注册三个 uart_port(serial8250_ports) 进 uart_driver.
		uart_add_one_port(drv, &up->port);
			uart_configure_port(); // 会直接返回 因为 port->iobase 没初始化
	serial8250_probe();


console_initcall(univ8250_console_init);
	serial8250_isa_init_ports
    register_console
```

APIs:

```c++
int serial8250_register_8250_port(const struct uart_8250_port *); // 注册 8250 port
void serial8250_unregister_port(int line); // 注销 8250 port
void serial8250_suspend_port(int line);
void serial8250_resume_port(int line);
```

### 8250_port.c

作为 8250_core.c 的 backend。

- 实现 uart_port->uart_ops callbacks.
- 实现 univ8250_console 的 callbacks.

### 8250_dw.c

在 8250_core 之后注册。

```c++
// 8250_dw.c
struct uart_8250_port uart = {}, *up = &uart;
struct uart_port *p = &up->port;

p->type		= PORT_8250;
p->flags	= UPF_SHARE_IRQ | UPF_FIXED_PORT | UPF_BOOT_AUTOCONF; //最后一个在之后 append
p->dev		= dev;
p->iotype	= UPIO_MEM;
p->set_ldisc	= dw8250_set_ldisc;
p->set_termios	= dw8250_set_termios; /// 在 dw8250_quirks 中又 p->set_termios=NULL
p->membase = devm_ioremap(dev, regs->start, resource_size(regs));
p->private_data = &data->data;
p->iotype = UPIO_MEM32;
p->serial_in = dw8250_serial_in32;
p->serial_out = dw8250_serial_out32;
p->line = id;
p->set_termios = NULL;
port->type = PORT_16550A;
up->capabilities = UART_CAP_FIFO;

dw8250_probe();
	dw8250_quirks();
    serial8250_register_8250_port();
		serial8250_find_match_or_unused();
        uart_remove_one_port();
            unregister_console(); // 内核打印:printk: console [ttyS1] disabled
		serial8250_set_defaults();
        uart_add_one_port();
			uart_configure_port();
				port->ops->config_port(port, flags); // 调用到 8250_port.c 中 uart_ops 的 config_port
					autoconfig();
						autoconfig_16550a();
				uart_report_port();// 打印：...ttyS1 at MMIO 0x18810100 (irq = 13, base_baud = 1500000) is a 16550A
				register_console();// 内核打印:printk: console [ttyS1] enabled
					try_enable_new_console();
						newcon->setup(); // 这里调用到 8250_core.c 中 univ8250_console.setup 创建 console
					unregister_console();//内核打印:bootconsole [earlycon0] disabled


// 中断处理
p->handle_irq	= dw8250_handle_irq;
dw8250_handle_irq();
	serial8250_handle_irq();
		serial8250_rx_chars(); // 读数据
		serial8250_tx_chars(); // 发数据

```

### Serial_core.c

注册一个`tty driver`, 提供了`static const struct tty_operations uart_ops` `static const struct tty_port_operations uart_port_ops`供上层调用。

## Printk

以 linux5.10 为例。（注意到最新的 linux6.3 printk 流程不一样了）

```c++
//kernel/printk/printk.c
printk;
	vprintk_func // printk_safe.c
		vprintk_default
			vprintk_emit
				vprintk_store // 把要打印的信息保存在 log_buf 中
        			log_output
        		console_unlock
        			call_console_drivers
        				con->write(con, text, len); // 调用 console driver 的 write 函数

// 8250 consle
static struct console univ8250_console = {
	.name		= "ttyS",
	.write		= univ8250_console_write,
	.device		= uart_console_device,
	.setup		= univ8250_console_setup,
	.match		= univ8250_console_match,
	.flags		= CON_PRINTBUFFER | CON_ANYTIME,
	.index		= -1,
	.data		= &serial8250_reg,
};

univ8250_console_write();
	serial8250_console_write();
		uart_console_write();
			putchar();//即 serial8250_console_putchar();
				serial_port_out(port, UART_TX, ch); // 往 UART_TX 寄存器写数据
					up->serial_out(up, offset, value); // 调用到 8250_dw.c 中写寄存器函数
```

## Early printk

打开`CONFIG_EARLY_PRINTK`

如果在设备树 cmdline 中添加了`earlyprintk`，会进入`/arch/arm/kernel/early_printk.c` 中`early_param(" earlyprintk", setup_early_printk);`指定的 setup_early_printk 函数。可以看到这个 earlyprintk 是在 arch arm 上实现的，比如其他 risc-v 是没有的 (通过 earlycon 实现)。

```c++
setup_early_printk();
	register_console(&early_console_dev); // 注册了一个 console，printk 会调用 console 的.write 回调函数

early_console_write();
	early_write();
		printascii(); // arch/arm/kernel/debug.S
			addruart_current
                addruart
                	ldr	\rp, =CONFIG_DEBUG_UART_PHYS // 需要在 linux menuconfig 中手动设定 earlyprink 的 Uart 串口地址
                	ldr	\rv, =CONFIG_DEBUG_UART_VIRT
            waituarttxrdy
            senduart
            busyuart
```

```c++
register_console
    	pr_info("%sconsole [%s%d] enabled\n",
		(newcon->flags & CON_BOOT) ? "boot" : "" ,
		newcon->name, newcon->index);
```

## Earlycon

- command line 中`earlycon`如果不带参数，参数在下面的`stdout-path`中，所以要解析设备树。

使用"/chosen"下的"stdout-path"找到节点，根据节点的`"compatible"`找到对应的`OF_EARLYCON_DECLARE`，进入 setup 函数。

- 如果`earlycon=xxx`含参数，无需设备树。

可以利用`EARLYCON_DECLARE`宏，根据`name(xxx)`，找到对应的 setup 函数。

例如`earlycon-riscv-sbi.c`中`EARLYCON_DECLARE(sbi, early_sbi_setup);` 只需在设备树中指定`earlycon=sbi`
