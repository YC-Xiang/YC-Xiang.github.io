---
title: Linux Essentials
date: 2023-05-19 22:25:00
tags:
  - Linux
categories:
  - Linux
---

# 编译

使用 LLVM 编译内核 (需要安装 llvm 工具链):

```shell
make mrproper
make LLVM=1 ARCH=arm defconfig
make LLVM=1 ARCH=arm Image -j12
```

生成 compile_commands.json:

```shell
cd scripts/clang-tools/
./gen_compile_commands.py -d ../.. # linux根目录
mv compile_commands.json ../..
```

# NOT indexed

include <asm/xxx.h> 先找 arch/xxx/include/xxx.h，没有的话就找/include/asm-generic/xxx.h

从 dts 中获取 regs 地址并映射到 virtual address:

linux5.10: void \_\_iomem *devm_platform_get_and_ioremap_resource(struct platform_device *pdev, unsigned int index, struct resource \*\*res)

linux5.4: void \_\_iomem *devm_platform_ioremap_resource(struct platform_device *pdev, unsigned int index)

相当于 platform_get_resource + devm_request_mem_region **+** devm_ioremap

linux 链表相关操作：

```c
#define list_entry(ptr, type, member) \ /// list_entry作用和container_of相同
	container_of(ptr, type, member)

container_of(ptr, type, member)

// ptr:表示结构体中member的地址h(已知的)
// type:表示结构体类型 struct xxx
// member:表示结构体中的成员 yyy
// 返回结构体的首地址

/**
 * list_for_each_entry	-	iterate over list of given type
 * @pos:	the type * to use as a loop cursor. pos中的list_head被加入了第二个成员head中
 * @head:	the head for your list. 要循环的链表
 * @member:	the name of the list_head within the struct. pos中的list_head链表对象
 */
list_for_each_entry(pos, head, member)
```

# Initcalls

![](https://xyc-1316422823.cos.ap-shanghai.myqcloud.com/20230524150950.png)

- 图中 initcalls 优先级从上到下。这些都是只能用于 builtin 的 modules。loadable 的 modules 使用 module_init()。

- 使用 initcalls 会在目标文件 object file 中创建 ELF sections。

**module_init()**

本质是 device_initcall。

```c
#define module_init(x)	__initcall(x);
#define __initcall(fn) device_initcall(fn)
```

kernel 的`System.map`可以查看符号文件，其中`__initcall6_start`后的顺序就对应`device_initcall`的驱动加载顺序。

```c
#define pure_initcall(fn)		__define_initcall(fn, 0)

#define core_initcall(fn)		__define_initcall(fn, 1)
#define postcore_initcall(fn)		__define_initcall(fn, 2)
#define arch_initcall(fn)		__define_initcall(fn, 3)
#define subsys_initcall(fn)		__define_initcall(fn, 4)
#define fs_initcall(fn)			__define_initcall(fn, 5)
#define rootfs_initcall(fn)		__define_initcall(fn, rootfs)
#define device_initcall(fn)		__define_initcall(fn, 6)
#define late_initcall(fn)		__define_initcall(fn, 7)
```
