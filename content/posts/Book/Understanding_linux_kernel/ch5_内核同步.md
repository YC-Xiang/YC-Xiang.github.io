# 同步原语

## per cpu 变量

把内核变量声明为 per cpu variable. per cpu variable 主要是数据结构的数组，系统的每个 CPU 对应数组的一个元素。

```c++
DEFINE_PER_CPU(type, name) // 静态分配一个 per cpu 数组，数组名为 name，结构类型为 type。
per_cpu(name, cpu) // 为 cpu 选择一个 per cpu 元素，cpu 参数为 index, 数组名称为 name
get_cpu_var(name) // 先禁用内核抢占，然后在 per cpu 数组
put_cpu_var(name) // 启用内核抢占
alloc_percpu(type) // 动态分配 type 类型数据结构的 per cpu 数组，返回它的地址
free_percpu(pointer)
per_cpu_ptr(pointer, cpu)
```

## 原子操作
