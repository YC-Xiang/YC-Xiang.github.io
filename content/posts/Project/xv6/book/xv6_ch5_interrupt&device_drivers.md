deivce interrupt 的入口在`kernel/trap.c` 的 `devintr`函数.

# 5.1 Code: Console input

ns16550 uart driver 寄存器相关, 每个占 1 个 byte:

LSR:

RHR:

THR:

xv6 main 函数通过`kernel/console.c`中的`consoleinit`函数来初始化 uart 硬件.
