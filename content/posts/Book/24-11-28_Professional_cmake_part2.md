---
date: 2024-11-29T11:09:33+08:00
title: "Professional CMake: A Practical Guide Part II Builds In Depth"
tags:
  - CMake
categories:
  - Book
---

# Chapter 13. Build Type

## 13.1 Build Type Basics

cmake 有以下几种 build type, 不同的 tpye 会导致 compiler 和 linker flags 不同:

Debug: no optimization and full debug information.

Release: typically full optimization and no debug information.

RelWithDebInfo: 有优化 + debug info.

MinRizeRel: 优化 size.

## 13.1.1 Single Configuration Generators

像 make, ninja, 每个 build directory 只支持一种 build type, 需要在编译时指定 cache variable CMAKE_BUILD_TYPE:

```cmake
cmake -G Ninja -DCMAKE_BUILD_TYPE:STRING=Debug ../source
cmake --build .
```

## 13.1.2 Multiple Configuration Generators

类似 Xcode and Visual Studio, 不关注, 跳过.

## 13.2 Common Errors

不要在 CMakeLists.txt 中使用 CMAKE_BUILD_TYPE 来判断 build type, 而使用`$<CONFIG:…>`:

```cmake
# WARNING: Do not do this!
if(CMAKE_BUILD_TYPE STREQUAL "Debug")
  # Do something only for debug builds
endif()
```

## 13.3 Custom Build Types

通过设置 CMAKE_BUILD_TYPE 增加自定义的 build type:

```cmake
set_property(CACHE CMAKE_BUILD_TYPE PROPERTY
  STRINGS Debug Release Profile)
```

</br>

定义不同 build type 的 compiler 和 linker flags. 其中 `<CONFIG>` 为 build type
如果没有 CONFIG 的话会对所有 build type 都生效:

`CMAKE_<LANG>_FLAGS_<CONFIG>`

`CMAKE_<TARGETTYPE>_LINKER_FLAGS_<CONFIG>`

e.g.

```cmake
set(CMAKE_C_FLAGS_PROFILE "-p -g -O2" CACHE STRING "")
set(CMAKE_CXX_FLAGS_PROFILE "-p -g -O2" CACHE STRING "")
set(CMAKE_EXE_LINKER_FLAGS_PROFILE "-p -g -O2" CACHE STRING "")
set(CMAKE_SHARED_LINKER_FLAGS_PROFILE "-p -g -O2" CACHE STRING "")
set(CMAKE_STATIC_LINKER_FLAGS_PROFILE "-p -g -O2" CACHE STRING "")
set(CMAKE_MODULE_LINKER_FLAGS_PROFILE "-p -g -O2" CACHE STRING "")
```

</br>

另外一个有用的变量为`CMAKE_<CONFIG>_POSTFIX`. 生成对应 build type 的 target 时, 会带该后缀, 这样在 build directory 中不同的 build type 生成的文件就不会冲突了.

一般 release build type 的后缀为空.

# Chapter 14. Compiler And Linker Essentials

## 14.1 Target Properties

### 14.1.1 Compiler Flags

控制当前 target 的 compiler flags 最重要的 target properties 如下:

**INCLUDE_DIRECTORIES**

头文件搜索目录, 绝对路径. 相当于 compiler 的`-I`选项.

**COMPILE_DEFINITIONS**

定义的宏, 相当于 compiler 中的`-DXXX`和`-DXXX=XXX`.

**COMPILE_OPTIONS**

其他的 compiler flags 放这里.

</br>

注意以上三个 target property 还分别有一个同名的 property, 但是以 INTERFACE\_开头.

区别是 INTERFACE\_开头的不是给当前 target 使用的, 而是给 link 当前 target 的 target 使用的.

### 14.1.2 Linker Flags

控制当前 target 的 linker flags 的 target properties 如下:

**LINK_LIBRARIES**

当前 target 需要链接的所有库. 列出的库可以是: 1. 指向库的绝对地址. 2. 库名称, 不带 platform 相关的前缀和后缀. 3. cmake library target.

**LINK_FLAGS**

传递给 target 的 linker flags. 只对 executable, share library, module library 有用, 对 static library 不起作用.

**STATIC_LIBRARY_FLAGS**

只对 static library 起作用的 linker flags.

</br>

和 compiler property 不同, 这里只有 LINK_LIBRARIES 有对应的 interface property INTERFACE_LINK_LIBRARIES.

### 14.1.3 Target Property Commands

```cmake
target_link_libraries(targetName
  <PRIVATE|PUBLIC|INTERFACE> item1 [item2 ...]
  [<PRIVATE|PUBLIC|INTERFACE> item3 [item4 ...]]
  ...
)
```

PRIVATE 增加 libs 到 LINK_LIBRARIES 属性, INTERFACE 增加 libs 到 INTERFACE_LINK_LIBRARIES, PUBLIC 同时增加.

下面的命令同理.

</br>

```cmake
target_include_directories(targetName [SYSTEM] [BEFORE]
  <PRIVATE|PUBLIC|INTERFACE> dir1 [dir2 ...]
  [<PRIVATE|PUBLIC|INTERFACE> dir3 [dir4 ...]]
  ...
)
```

BEFORE: 在已有的 include_directory 值头部追加, 默认行为是在尾部追加.

// FIXME: 没太懂这个 SYSTEM 的作用

SYSTEM: 编译器将被告知这些目录是某些平台上的系统包含目录。这可能会抑制警告或跳过依赖项计算中的包含头文件。此外，不管指定的顺序如何，系统包含目录都会在普通包含目录之后搜索。

注意 imported target 的 INTERFACE_INCLUDE_DIRECTORIES 会被 consuming target 视为 SYSTEM 路径.

</br>

```cmake
target_compile_definitions(targetName
  <PRIVATE|PUBLIC|INTERFACE> item1 [item2 ...]
  [<PRIVATE|PUBLIC|INTERFACE> item3 [item4 ...]]
  ...
)
```

each item having the form VAR or VAR=VALUE.

</br>

```cmake
target_compile_options(targetName [BEFORE]
  <PRIVATE|PUBLIC|INTERFACE> item1 [item2 ...]
  [<PRIVATE|PUBLIC|INTERFACE> item3 [item4 ...]]
  ...
)
```

## 14.2 Directory Properties And Commands

老的 cmake 版本, 通常使用 directory properties 而不是 target properties. 现在已不推荐使用.

```cmake
include_directories([AFTER | BEFORE] [SYSTEM] dir1 [dir2...])
add_definitions(-DSomeSymbol /DFoo=Value ...)
remove_definitions(-DSomeSymbol /DFoo=Value ...)
add_compile_definitions(SomeSymbol Foo=Value ...)
add_compile_options(opt1 [opt2 ...])
link_libraries(item1 [item2 ...] [ [debug | optimized | general] item] ...)
link_directories(dir1 [dir2 ...])
```

## 14.3 Compiler And Linker Variables

补充 13.3 的内容, 在某些情况下，用户希望添加自己的编译器或链接器标志。他们可能希望添加更多的警告选项，打开特殊的编译器特性，如调试开关等。对于这些情况，可以使用 variable 代替 property 更为合适。

```txt
CMAKE_<LANG>_FLAGS
CMAKE_<LANG>_FLAGS_<CONFIG>
CMAKE_<TARGETTYPE>_LINKER_FLAGS
CMAKE_<TARGETTYPE>_LINKER_FLAGS_<CONFIG>
```

TARGETTYPE 有 EXE, STATIC, SHARED, MODULE
