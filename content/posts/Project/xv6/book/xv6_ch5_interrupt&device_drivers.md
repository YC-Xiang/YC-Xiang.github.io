deivce interrupt 的入口在`kernel/trap.c` 的 `devintr`函数。

# 5.1 Code: Console input

ns16550 uart driver 寄存器相关，每个占 1 个 byte:

- LSR: line status register, 保存 uart 硬件的状态。
- RHR: receive holding register, 保存接收到的数据。
- THR: transmit holding register, software 往里写，uart hardware 就会发送。

xv6 main 函数通过 `kernel/console.c` 中的 `consoleinit` 函数来初始化 uart 硬件，使能 uart 中断。

xv6 shell 通过在 `init.c` 中打开的 console file descriptor 从 console 中读取数据。通过 `read` system call 调用到 `consoleread` 回调函数。`consoleread` 函数等待 input, 并保存到 `cons.buf`. 如果 user 还没有输入完完整的一行，reading process 会进入 sleep.

当用户输入一个 character, 就会触发一个 uart 中断，进入 trap handler 后跳转到 `devintr`, 查看 RISC-V 的 `scause` 寄存器判断是否是从 external device 传来的，然后通过 PLIC 判断是哪个 device 的中断，如果是 uart 中断最后进入 `uartintr`。

`uartintr` 把 uart hardware receive register 中的所有数据都读出来，然后调用 `consoleintr`. `consoleintr` 的作用是累计 input characters 到 `cons.buf` 直到遇到换行符，当新行到来时，会 wakeup 在等待的 `consoleread`. 一旦被唤醒 consoleread 会从 cons.buf 中拿出一行 copy 到 userspace.

## 5.2 Code: Console output

write 系统调用，进入`consolewrite`, 再进入`uartputc`, 将要发送的字符放入 buffer, 最后调用 `uartstart` 发送字符。

每发送一个字符都会产生一个中断，通过`uartintr` -> `uartstart` 检查上面的流程是否完成了传输。如果一个进程写了多个 bytes, 通常第一个 byte 会通过`uartputc`->`uartstart` 发送，然后产生一个中断，然后后面的 bytes 都会通过`uartintr`->`uartstart` 发送。

# 5.4 Timer interrupts

RISC-V 要求 timer 中断需要在 machine mode 中处理，而不能在 supervisor mode 中处理。

machine mode 的 code 在 `start.c` 中，`main` 函数之前。在 `timerinit` 中设置 timer 的 interrupt handler 为 `timervec`.

因为 timer 中断可能打断任何 user 和 kernel code, kernel 又没办法在临界区禁止 machine mode 的 timer 中断。所以 timer 中断必须要保证不破坏 kernel 一些共享的数据结构。

因此 xv6 在 machine mode timer interrupt 中只 raise 一个 software interrupt, 这样把 timer interrupt 剩余部分移交到 kernel 中处理，这样 kernel 可以像普通中断一样处理 timer 中断，也可以 disable timer interrupt 处理了。
