## 6.1 Races

## 6.2 Code: Locks

Xv6 有两种锁，spinlocks 和 sleep-locks.

```c++
struct spinlock {
	uint locked; // 0 表示可以获取锁，1 表示锁被占用
	char *name; // 锁的名称
	struct cpu *cpu; // 持有锁的 CPU
};
```

```c++
void acquire(struct spinlock *lk)
{
	push_off(); // disable interrupts to avoid deadlock.
	if (holding(lk)) // 如果当前进程已经持有锁，则 panic，不可重入
		panic("acquire");

	while (__sync_lock_test_and_set(&lk->locked, 1) != 0)
		;

	__sync_synchronize();
	lk->cpu = mycpu();
}
```

__sync_lock_test_and_set 是 C 库的一个原子操作，在 risc-v 上，它会被编译成 `amoswap` 原子指令。

__sync_lock_test_and_set(&lk->locked, 1) 的工作原理：

1. 原子性地读取 lk->locked 的当前值
2. 将 lk->locked 设置为 1（表示锁被占用）
3. 返回 lk->locked 的原始值

- 如果返回 0：说明锁之前是空闲的（0），现在成功获取了锁，跳出循环
- 如果返回 1：说明锁之前就被占用（1），继续自旋等待

```c++
void release(struct spinlock *lk)
{
	if (!holding(lk))
		panic("release");

	lk->cpu = 0;
	
	__sync_synchronize();
	__sync_lock_release(&lk->locked);

	pop_off(); // enable interrupts again
}
```

__sync_lock_release 原子地将 lk->locked 设置为 0。

## 6.3 Code: Using locks

## 6.4 Deadlock and lock ordering

## 6.5 Re-entrant locks

通过使用可重入锁（也称为递归锁）可以避免一些死锁和锁排序问题。

其思想是，如果锁由进程持有，并且如果该进程试图再次获得锁，那么内核可以允许这样做（因为该进程已经拥有锁），而不是像 xv6 内核那样调用 panic。

## 6.6 Locks and interrupt handlers

## 6.7 Instruction and memory ordering

__sync_synchronize() 内存屏障，确保在它之前的所有内存操作都完成，在它之后的所有内存操作都开始。
