---
date: 2024-11-20T21:14:33+08:00
title: "Professional CMake: A Practical Guide"
tags:
  - CMake
categories:
  - Book
---

# Chapter 4. Building Simple Targets

## 4.1 Executables

```cmake
add_executable(targetName [WIN32] [MACOSX_BUNDLE]
[EXCLUDE_FROM_ALL]
source1 [source2 ...]
)
```

## 4.2 Defining Libraries

```cmake
add_library(targetName [STATIC | SHARED | MODULE]
[EXCLUDE_FROM_ALL]
source1 [source2 ...]
)
```

STATIC: .a 静态库

SHARED: .so 动态库

MODULE: 可以在运行时加载的库, 类似于各种软件的 plugins.

可以不指定 type, 那么由 cmake 变量 BUILD_SHARED_LIBS 决定默认类型. 如果为 true 则是
shared library, 否则为 static.

## 4.3 Linking Targets

```cmake
target_link_libraries(targetName
<PRIVATE|PUBLIC|INTERFACE> item1 [item2 ...]
[<PRIVATE|PUBLIC|INTERFACE> item3 [item4 ...]]
...
)
```

A 依赖于 B(A is linked to B), Libraries 之间的依赖关系有下面三种:

PRIVATE: 只有 A 依赖于 B, 其他链接到 A 的 targets 不知道 B 的存在.

PUBLIC: 其他链接到 A 的 targets 也会依赖于 B.

INTERFACE: A 不依赖于 B, 链接到 A 的 targets 依赖于 B.

## 4.4 Linking Non-targets

target_link_libraries 除了可以传入现存的 cmake targets(通过 add_executable 和
add_library 创建的 targets), 还可以传:

- Full path to a library file: 库文件(.a, .so)的绝对路径.
- Plain library name: 库文件名称, 比如 foo 对应-lfoo, linker 会自己从库路径中找.
- Link flag: 可以传 flags 给 linker command.

# Chapter 5. Variables

```cmake
set(varName value... [PARENT_SCOPE])
unset(varName)
```

PARENT_SCOPE 用来提升变量的作用域, 这个后面章节会讲.

cmake 所有变量都是 string, 不用加引号, 除非中间有空格.

如果提供了多个 value, 那么就是 cmake 中的 lists, 每个 value 由`;`隔开, 组成最终一个字符串.

```cmake
set(myVar a b c) # myVar = "a;b;c"
set(myVar a;b;c) # myVar = "a;b;c"
set(myVar "a b c") # myVar = "a b c"
set(myVar a b;c) # myVar = "a;b;c"
set(myVar a "b c") # myVar = "a;b c"
```
