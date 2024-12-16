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

**Debug**: no optimization and full debug information.

**Release**: typically full optimization and no debug information.

**RelWithDebInfo**: 有优化 + debug info.

**MinRizeRel**: 优化 size.

## 13.1.1 Single Configuration Generators

像 make, ninja, 每个 build directory 只支持一种 build type, 需要在编译时指定 cache variable CMAKE_BUILD_TYPE:

```cmake
cmake -G Ninja -DCMAKE_BUILD_TYPE:STRING=Debug ../source
cmake --build .
```

一种可能的文件布局方式:

![](https://xyc-1316422823.cos.ap-shanghai.myqcloud.com/20241216112812.png)

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

一般 release build type 的后缀为空, debug 可以加 d 或者 debug 这样的后缀.

# Chapter 14. Compiler And Linker Essentials

这一章讨论如何控制 compiler 和 linker 的行为.

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

当前 target 需要链接的所有库. 列出的库可以是:

- 指向库的绝对地址.
- 库名称, 不带 platform 相关的前缀(lib)和后缀(.a, .so, .dll).
- cmake library target.

**LINK_FLAGS**

传递给 target 的 linker flags. 只对 executable, share library, module library 有用, 对 static library 不起作用.

**STATIC_LIBRARY_FLAGS**

只对 static library 起作用的 linker flags.

</br>

和 compiler property 不同, 这里只有 LINK_LIBRARIES 有对应的 interface property INTERFACE_LINK_LIBRARIES.

### 14.1.3 Target Property Commands

前面两节提到的 target property 通常不直接操作, 而是通过 cmake 的命令来操作:

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

**BEFORE**: 在已有的 include_directory 值头部追加, 默认行为是在尾部追加.

**SYSTEM**: 带 SYSTEM 关键字的 include path 会被视为系统头文件路径。系统 includ 目录会在普通 include 目录之后搜索。

假设你有一个第三方库的头文件，其中可能包含一些编译器在普通用户代码头文件中会发出警告的代码结构（比如一些不标准但在该库环境下合理的用法）。通过将该库的头文件路径标记为 SYSTEM，可以减少这些不必要的警告。

注意 imported target 的 INTERFACE_INCLUDE_DIRECTORIES 默认会被 consuming target 视为 SYSTEM 路径.

</br>

```cmake
target_include_directories(
    MyTarget
  PUBLIC
    $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>
    $<INSTALL_INTERFACE:include>
)
```

一种常见的用法, `$<BUILD_INTERFACE:…>`和`$<INSTALL_INTERFACE:…>` generator expression 允许为 build 和 install 指定不同路径.

</br>

```cmake
target_compile_definitions(targetName
  <PRIVATE|PUBLIC|INTERFACE> item1 [item2 ...]
  [<PRIVATE|PUBLIC|INTERFACE> item3 [item4 ...]]
  ...
)
```

增加 compile definition. each item having the form VAR or VAR=VALUE.

</br>

```cmake
target_compile_options(targetName [BEFORE]
  <PRIVATE|PUBLIC|INTERFACE> item1 [item2 ...]
  [<PRIVATE|PUBLIC|INTERFACE> item3 [item4 ...]]
  ...
)
```

增加 compiler flags.

## 14.2 Directory Properties And Commands

老的 cmake 版本, 通常使用 directory properties 而不是 target properties. 现在已不推荐使用. 命令如下:

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

通过 Variables 来修改 compiler 和 linker 的 flags, 已不推荐使用.

```txt
CMAKE_<LANG>_FLAGS
CMAKE_<LANG>_FLAGS_<CONFIG>
CMAKE_<TARGETTYPE>_LINKER_FLAGS
CMAKE_<TARGETTYPE>_LINKER_FLAGS_<CONFIG>
```

TARGETTYPE 有 EXE, STATIC, SHARED, MODULE

e.g.

```cmake
set(CMAKE_CXX_FLAGS "-Wall -Werror")
```

# Chapter 15. Language Requirements

设置 language requirements, 通常有两种方法, 一种直接设置变量, 另一种允许项目指定它们需要的语言特性，并让 CMake 选择适当的语言标准。

## 15.1 Setting The Language Standard Directly

相关的 target properties:

`<LANG>_STANDARD`

C 标准有 90, 99, 11.

`<LANG>_STANDARD_REQUIRED`

为 true 时, `<LANG>_STANDARD`没达到要求时才会报错.

`<LANG>_EXTENSIONS`

一些编译器支持自己对语言标准的扩展.

</br>

设置命令为:

```cmake
# Require C++11 and disable extensions for all targets
set(CMAKE_CXX_STANDARD 11)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)
```

这三条命令一般同时设置, 跟在 project() 后面.

## 15.2 Setting The Language Standard By Feature Requirements

通过 target_compile_features()命令控制 COMPILE_FEATURE 和 INTERFACE_COMPILE_FEATURE property.

```cmake
target_compile_features(targetName
  <PRIVATE|PUBLIC|INTERFACE> feature1 [feature2 ...]
  [<PRIVATE|PUBLIC|INTERFACE> feature3 [feature4 ...]]
  ...
)
```

e.g.

```cmake
target_compile_features(targetName PUBLIC cxx_std_14)
```

可支持的 feature 查阅 cmake 文档 `CMAKE_<LANG>_KNOWN_FEATURES` 和`CMAKE_<LANG>_COMPILE_FEATURES`变量.

# Chapter 16. Target Types

## 16.1 Executables

介绍比之前更多的几种形式:

```cmake
add_executable(targetName [WIN32] [MACOSX_BUNDLE]
  [EXCLUDE_FROM_ALL]
  source1 [source2 ...])
add_executable(targetName IMPORTED [GLOBAL])
add_executable(aliasName ALIAS targetName)
```

IMPORTED: 可以利用外部已经存在的 executable 来创建 cmake target. 注意 imported target 不能 install, 这是有区别的地方.

GLOBAL: 使该 target 的 scope 为 global.

ALIAS: 别名. 只能指向 real target, 不可以嵌套, 给 alias target 再起别名. 同时也不能 install 和 export.

## 16.2 Libraries

```cmake
add_library(targetName [STATIC | SHARED | MODULE | OBJECT]
  [EXCLUDE_FROM_ALL]
  source1 [source2 ...])

add_library(aliasName ALIAS otherTarget)
```

</br>

**Imported Library**

```cmake
add_library(targetName (STATIC | SHARED | MODULE | OBJECT | UNKNOWN)
  IMPORTED [GLOBAL]
)
```

import library 需要指定 IMPORTED_LOCATION property 表示路径, 如果是 object library 还需要指定 IMPORTED_OBJECTS.

```cmake
# Assume FOO_LIB holds the location of the library but its type is unknown
add_library(mysteryLib UNKNOWN IMPORTED)
set_target_properties(mysteryLib PROPERTIES
  IMPORTED_LOCATION ${FOO_LIB}
)

# Imported object library, Windows example shown
add_library(myObjLib OBJECT IMPORTED)
set_target_properties(myObjLib PROPERTIES
  IMPORTED_OBJECTS /some/path/obj1.obj # These .obj files would be .o
  /some/path/obj2.obj # on most other platforms
)
# Regular executable target using imported object library.
# Platform differences are already handled by myObjLib.
add_executable(myExe $<TARGET_SOURCES:myObjLib>)
```

</br>

**Interface Library**

header-only libraries，**一个应用场景**是, 不需要链接物理库, 但 header search
paths，compiler definitions 等需要传递到使用该头文件库的任何内容.

```cmake
add_library(targetName INTERFACE [IMPORTED [GLOBAL]])
```

```cmake
add_library(myHeaderOnlyToolkit INTERFACE)
target_include_directories(myHeaderOnlyToolkit
  INTERFACE /some/path/include
)
target_compile_definitions(myHeaderOnlyToolkit
  INTERFACE COOL_FEATURE=1)
add_executable(myApp ...)
target_link_libraries(myApp PRIVATE myHeaderOnlyToolkit)
```

当 myApp 被编译时, 会有 /some/path/include 头文件搜索路径, 还会有 COOL_FEATURE=1 的编译器定义.

</br>

**另一个应用场景**是为链接更大的库集提供便利.

```cmake
# Regular library targets
add_library(algo_fast ...)
add_library(algo_accurate ...)
add_library(algo_beta ...)
# Convenience interface library
add_library(algo_all INTERFACE)
target_link_libraries(algo_all INTERFACE
  algo_fast
  algo_accurate
  $<$<BOOL:${ENABLE_ALGO_BETA}>:algo_beta>
)
# Other targets link to the interface library
# instead of each of the real libraries
add_executable(myApp ...)
target_link_libraries(myApp PRIVATE algo_all)
```

添加 IMPORTED 关键字来生成 INTERFACE IMPORTED 库有时会引起混淆。当导出或安装 INTERFACE 库以便在项目外部使用时，通常会出现这种组合. 不同关键字的区别对 interface 库的影响如下:

![](https://xyc-1316422823.cos.ap-shanghai.myqcloud.com/20241202151146.png)

## 16.3 Promoting Imported Targets

如果没有 IMPORTED 关键字, 那么 imported targets 是局部可见的.

cmake 提供了一个关键字 IMPORTED_GLOBAL 来提升 imported library 的 scope 到 global.

```cmake
# Imported library created with local visibility.
# This could be in an external file brought in
# by an include() call rather than in the same
# file as the lines further below.
add_library(builtElsewhere STATIC IMPORTED)
set_target_properties(builtElsewhere PROPERTIES
  IMPORTED_LOCATION /path/to/libSomething.a)
# Promote the imported target to global visibility
set_target_properties(builtElsewhere PROPERTIES
  IMPORTED_GLOBAL TRUE)
```

# Chapter 17. Custom Tasks

cmake 支持实现一些的任务.

## 17.1 Custom Targets

cmake 除了添加 executable 和 library target, 还可以创建自定义的 target:

```cmake
add_custom_target(targetName [ALL]
  [command1 [args1...]]
  [COMMAND command2 [args2...]]
  [DEPENDS depends1...]
  [BYPRODUCTS [files...]]
  [WORKING_DIRECTORY dir]
  [COMMENT comment]
  [VERBATIM]
  [USES_TERMINAL]
  [SOURCES source1 [source2...]]
)
```

**ALL**: 表示 all target 依赖于当前的 custom target. 如果没有 ALL, 那么必须主动去 request, 或者有其他 target 依赖于该 custom target 才会编译.

**COMMAND**: command1 第一条指令可以不加 COMMAND 前缀, 但推荐还是加上.

**DEPENDS**: 依赖的文件, 这边不可以放依赖的 target. 这边需要是依赖文件的绝对路径.

**BYPRODUCTS**: 用于列出作为运行命令的一部分而创建的其他文件.

**WORKING_DIRECTORY**: 默认为`${CMAKE_CURRENT_BINARY_DIR}`, 可以修改为绝对路径或者基于当前 binary dir 的相对路径.

**COMMENT**: 在运行 commands 之前的一段 log, 注意不是所有 generator 都支持.

**VERBATIM**: 转义只由 cmake 解析, 不会由 platform 进一步解析, 推荐加上.

**USES_TERMINAL**: 指示 CMake 让命令直接访问终端.

**SOURCES**: 仅用来将一些文件和 target 关联起来, 不影响 build 过程.

## 17.2 Adding Build Steps To An Existing Target

有时候不需要自定义一个 target, 而只需要在 build 某个 target 时, 增加一些额外的命令:

```cmake
add_custom_command(TARGET targetName buildStage
  COMMAND command1 [args1...]
  [COMMAND command2 [args2...]]
  [WORKING_DIRECTORY dir]
  [BYPRODUCTS files...]
  [COMMENT comment]
  [VERBATIM]
  [USES_TERMINAL]
)
```

选项和 add_custom_target 类似, 其中 buildStage 必须是:

**PRE_BUILD**: 只有 visual studio 支持.

**PRE_LINK**: 在源码编译之后, 链接之前执行.

**POST_BUILD**: 在 build 完后执行.

一般常用 POST_BUILD, 而 PRE_BUILD 和 PRE_LINK 不常用.

e.g.

```cmake
add_executable(myExe main.cpp)
add_custom_command(TARGET myExe POST_BUILD
  COMMAND script1 $<TARGET_FILE:myExe>
)
# Additional command which will run after the above from a different directory
add_custom_command(TARGET myExe POST_BUILD
  COMMAND writeHash $<TARGET_FILE:myExe>
  BYPRODUCTS ${CMAKE_BINARY_DIR}/verify/myExe.md5
  WORKING_DIRECTORY ${CMAKE_BINARY_DIR}/verify
)
```

## 17.3 Commands That Generate Files

add_custom_command()还有一个作用, 通过运行命令来创建文件，不依赖于某个 target.

```cmake
add_custom_command(OUTPUT output1 [output2...]
  COMMAND command1 [args1...]
  [COMMAND command2 [args2...]]
  [WORKING_DIRECTORY dir]
  [BYPRODUCTS files...]
  [COMMENT comment]
  [VERBATIM]
  [USES_TERMINAL]
  [APPEND]
  [DEPENDS [depends1...]
  [MAIN_DEPENDENCY depend]
  [IMPLICIT_DEPENDS <lang1> depend1
  [<lang2> depend2...]]
  [DEPFILE depfile]
)
```

// FIXME: 这里的一些依赖没看懂

**MAIN_DEPENDENCY**:

**IMPLICIT_DEPENDS**, **DEPFILE**: 大部分 generator 都不支持.

对于 TARGET 和 OUTPUT 两种形式, 如果要多次调用 add_custom_command()来追加 commands 之类的, 有如下区别;

OUTPUT 形式, 后面追加调用的 add_custom_command()必须加 APPEND 关键字, 并且第一个 OUTPUT file 必须相同. 追加的 add_custom_command()中只有 COMMAND 和 DEPENDS 有效, 其他关键字都失效.

TARGET 形式, 不用加 APPEND, COMMENT, WORKING_DIRECTORY 之类的关键字也不会失效.

## 17.4 Configure Time Tasks

add_custom_target()和 add_custom_command()是在 build 阶段定义执行 commands 的, execute_process 可以在 configure 阶段执行命令.

```cmake
execute_process(COMMAND command1 [args1...]
  [COMMAND command2 [args2...]]
  [WORKING_DIRECTORY directory]
  [RESULT_VARIABLE resultVar]
  [RESULTS_VARIABLE resultsVar]
  [OUTPUT_VARIABLE outputVar]
  [ERROR_VARIABLE errorVar]
  [OUTPUT_STRIP_TRAILING_WHITESPACE]
  [ERROR_STRIP_TRAILING_WHITESPACE]
  [INPUT_FILE inFile]
  [OUTPUT_FILE outFile]
  [ERROR_FILE errorFile]
  [OUTPUT_QUIET]
  [ERROR_QUIET]
  [TIMEOUT seconds]
)
```

**RESULT_VARIABLE**: 最后一条命令执行的成功/失败保存到该变量, 变量值为 0 表示成功.

**RESULTS_VARIABLE**: 每条命令执行的成功/失败保存到 list 中.

**OUTPUT_VARIABLE**: 最后一条命令的输出保存到该变量.

**ERROR_VARUABLE**: 标准错误流保存到该变量.

**OUTPUT_STRIP_TRAILING_WHITESPACE**, **ERROR_STRIP_TRAILING_WHITESPACE**: 删除上面两条的行尾空格, 可以加上.

**OUTPUT_FILE**: 后一条命令的输出保存到该文件.

**ERROR_FILE**: 标准错误流保存到该文件.

**INPUT_FILE**: 第一条命令的输入流指定一个文件.

**OUTPUT_QUIET**: 不关心 output 输出.

**ERROR_QUIET**: 不关心 error 输出.

**TIMEOUT**: 设置命令的超时时间. 是否超时的结果保存到 RESULT_VARIABLE 中, 需要主动检查, 不会导致 cmake 出错.

## 17.5 Platform Independent Commands

不同平台的命令不同, cmake 抽象出了许多命令来屏蔽这些差异. 可以通过`cmake -E help`来查看所有支持的命令. 常用的有:

- compare_files
- copy
- copy_directory
- copy_if_different
- echo
- env
- make_directory
- md5sum
- remove
- remove_directory
- rename
- tar
- time
- touch

e.g.

`${CMAKE_COMMAND}` 对应 cmake 可执行文件.

```cmake
# Platform independent equivalent
add_custom_target(myCleanup
  COMMAND "${CMAKE_COMMAND}" -E remove_directory "${discardDir}")
```

</br>

cmake 支持通过-P 选项传入 cmake 文件(包含一系列 cmake commands).

```shell
cmake [options] -P filename
cmake -DOPTION_A=1 -DOPTION_B=foo -P myCustomScript.cmake
```

# Chapter 18. Working With Files

操作文件和目录, 主要是 file()命令.

## 18.1 Manipulating Paths

**获取文件路径/文件名:**

```cmake
get_filename_component(outVar input component [CACHE])
```

component 的取值有:

**DIRECTORY/PATH**: 提取文件路径

**NAME**: 提取整个文件名

**NAME_WE**: 提取文件名.前面的部分

**EXT**: 提取文件名.之后的部分

```cmake
set(input /some/path/foo.bar.txt)
get_filename_component(path1 ${input} DIRECTORY) # /some/path
get_filename_component(path2 ${input} PATH) # /some/path
get_filename_component(fullName ${input} NAME) # foo.bar.txt
get_filename_component(baseName ${input} NAME_WE) # foo
get_filename_component(extension ${input} EXT) # .bar.txt
```

</br>

**获取绝对路径:**

```cmake
get_filename_component(outVar input component [BASE_DIR dir] [CACHE])
```

input 如果是相对路径, 如果 BASE_DIR 存在, 那么会根据 BASE_DIR 计算, 否则根据当前目录计算绝对路径.

input 如果是绝对路径, BASE_DIR 会被忽略.

component 的取值有:

**ABSOLUTE**: 计算 input 的绝对路径, 不考虑 symbol link 展开.

**REALPATH**: 计算 input 的绝对路径, 考虑 symbol link 展开.

</br>

file()命令提供相反的操作, 可以计算相对路径.

```cmake
set(basePath /base)
set(fooBarPath /base/foo/bar)
set(otherPath /other/place)
file(RELATIVE_PATH fooBar ${basePath} ${fooBarPath})
file(RELATIVE_PATH other ${basePath} ${otherPath})
# The variables now have the following values:
# fooBar = foo/bar
# other = ../other/place
```

## 18.2 Copying Files

在 configure 阶段拷贝文件:

```cmake
configure_file(source destination [COPYONLY | @ONLY] [ESCAPE_QUOTES])
```

source 和 destination 是目标文件和目的文件, 可以是绝对路径或者相对路径, 相对路径分别基于 CMAKE_CURRENT_SOURCE_DIR 和 CMAKE_CURRENT_BINARY_DIR.

**COPYONLY**: source 文件中的 cmake 变量(${...}和@...@)不会被展开到 destination 文件.

**@ONLY**: source 文件中的${...}不会被展开, @...@会被展开.

**ESCAPE_QUOTES**: source 文件中的`\`也会被拷贝到 destination 文件, 而不是解释成转符号.

</br>

如果不需要替换, 也可以使用 file()命令:

```cmake
file(<COPY|INSTALL> fileOrDir1 [fileOrDir2...]
  DESTINATION dir
  [NO_SOURCE_PERMISSIONS | USE_SOURCE_PERMISSIONS |
  [FILE_PERMISSIONS permissions...]
  [DIRECTORY_PERMISSIONS permissions...]]
  [FILES_MATCHING]
  [PATTERN pattern | REGEX regex] [EXCLUDE]
  [PERMISSIONS permissions...]
  [...]
)
```

如果要拷贝目录下的内容, 要加上`/`, 否则拷贝的是目录名.

```cmake
file(COPY base/srcDir DESTINATION destDir) # --> destDir/srcDir
file(COPY base/srcDir/ DESTINATION destDir) # --> destDir
```

</br>

COPY 和 INSTALL 的区别是, COPY 拷贝原有文件的权限, 而 INSTALL 不是.
可以通过 NO_SOURCE_PERMISSIONS, USE_SOURCE_PERMISSIONS, FILE_PERMISSIONS, DIRECTORY_PERMISSIONS 主动修改:

```cmake
file(COPY whoami.sh
  DESTINATION .
  FILE_PERMISSIONS OWNER_READ OWNER_WRITE OWNER_EXECUTE
  GROUP_READ GROUP_EXECUTE
  WORLD_READ WORLD_WRITE
)
```

</br>

**REGEX**: 正则表达式匹配.

**EXCLUDE**: 排除某些文件.

**PATTERN**: 通配符.

e.g.

```cmake
file(COPY someDir
  DESTINATION .
  FILES_MATCHING
  REGEX .*_private\\.h EXCLUDE
  PATTERN *.h
  PATTERN *.sh
  PERMISSIONS OWNER_READ OWNER_WRITE OWNER_EXECUTE
)
```

configure_file()和 file()都是在 configure 阶段拷贝, 要在 build 阶段拷贝, 可以使用 cmake command mode:

```cmake
cmake -E copy file1 [file2...] destination
cmake -E copy_if_different file1 [file2...] destination # 如果文件已存在, 不拷贝
```

## 18.3 Reading And Writing Files Directly

文件写入:

```cmake
file(WRITE fileName content)
file(APPEND fileName content)
```

```cmake
file(GENERATE
  OUTPUT outFile
  INPUT inFile | CONTENT content
  [CONDITION expression]
)
```

和 file(WRITE...)类似, 但可以增加一个条件.

INPUT 和 CONTENT 只能出现一个.

CONDITION 是满足条件才会写入.

e.g.

```cmake
# Generate unique files for all but Release
file(GENERATE
  OUTPUT ${CMAKE_CURRENT_BINARY_DIR}/outfile-$<CONFIG>.txt
  INPUT ${CMAKE_CURRENT_SOURCE_DIR}/input.txt.in
  CONDITION $<NOT:$<CONFIG:Release>>
)

# Embedded content, bracket syntax does not
# prevent the use of generator expressions
file(GENERATE
  OUTPUT ${CMAKE_CURRENT_BINARY_DIR}/details-$<CONFIG>.txt
  CONTENT [[
Built as "$<CONFIG>" for platform "$<PLATFORM_ID>".
]])
```

</br>

文件读取:

```cmake
file(READ fileName outVar
  [OFFSET offset] [LIMIT byteCount] [HEX]
)
```

读取 file 的内容, 保存到 outVar 字符串.

**OFFSET**: 选择偏移量.

**LIMIT**: 限制读取的最大字节数.

**HEX**: 将内容转换为十六进制保存.

</br>

如果希望逐行分解文件内容, 则 STRINGS 的形式更方便, 每行以字符串的形式存储, 组成列表.

```cmake
file(STRINGS fileName outVar
  [LENGTH_MAXIMUM maxBytesPerLine]
  [LENGTH_MINIMUM minBytesPerLine]
  [LIMIT_INPUT maxReadBytes]
  [LIMIT_OUTPUT maxStoredBytes]
  [LIMIT_COUNT maxStoredLines]
  [REGEX regex]
)
```

**LENGTH_MAXIMUM**, **LENGTH_MINIMUM**: 排除每行长度大于/小于特定字节数的字符串.

**LIMIT_INPUT**: 限制读取的总字节数.

**LIMIT_OUTPUT**: 限制存储的总字节数.

**LIMIT_COUNT**: 限制总行数.

**REGEX**: 正则表达式.

## 18.4. File System Manipulation

```cmake
file(RENAME source destination)
file(REMOVE files...) # 删文件
file(REMOVE_RECURSE filesOrDirs...) # 删目录
file(MAKE_DIRECTORY dirs...)

```

检索文件:

```cmake
file(GLOB outVar
  [LIST_DIRECTORIES true|false]
  [RELATIVE path]
  [CONFIGURE_DEPENDS] # Requires CMake 3.12 or later
  expressions...
)
file(GLOB_RECURSE outVar
  [LIST_DIRECTORIES true|false]
  [RELATIVE path]
  [FOLLOW_SYMLINKS]
  [CONFIGURE_DEPENDS] # Requires CMake 3.12 or later
  expressions...
)
```

> 注意不要用这两种检索方法来收集源文件/头文件/build 过程的输入文件等, 原因是如果这些文件添加或者删除, cmake 不会自动重新 configure, 而 build 过程不会知道这些改动.

## 18.5. Downloading And Uploading

支持从 URL download/upload 文件.

```cmake
file(DOWNLOAD url fileName [options...])
file(UPLOAD fileName url [options...])
```

# Chapter 19. Specifying Version Details

cmake 项目版本控制.

## 19.1. Project Version

支持在 project()中指定项目 version:

```cmake
project(FooBar VERSION 2.4.7)
```

会创建如下变量:

![](https://xyc-1316422823.cos.ap-shanghai.myqcloud.com/20241206135345.png)

cmake 3.12 之后还引入了额外一组变量:

![](https://xyc-1316422823.cos.ap-shanghai.myqcloud.com/20241206135432.png)

推荐使用 prijectName_VERSION_XXX 这组变量, 是全局唯一的, 不会随着多次调用 project()命令而改变.

## 19.2. Source Code Access To Version Details

让源码了解 cmake 中的版本信息.

利用 configure_file 将 .in 文件中的 cmake version 变量展开, 生成源文件编译.

foobar_version.h

```cpp
#include <string>
std::string getFooBarVersion();
unsigned getFooBarVersionMajor();
unsigned getFooBarVersionMinor();
unsigned getFooBarVersionPatch();
unsigned getFooBarVersionTweak();
```

foobar_version.cpp.in

```c
#include "foobar_version.h"
std::string getFooBarVersion()
{
  return "@FooBar_VERSION@";
}
unsigned getFooBarVersionMajor()
{
  return @FooBar_VERSION_MAJOR@;
}
unsigned getFooBarVersionMinor()
{
  return @FooBar_VERSION_MINOR@ +0;
}
unsigned getFooBarVersionPatch()
{
  return @FooBar_VERSION_PATCH@ +0;
}
unsigned getFooBarVersionTweak()
{
  return @FooBar_VERSION_TWEAK@ +0;
}

```

CMakeLists.txt

```cmake
cmake_minimum_required(VERSION 3.0)
project(FooBar VERSION 2.4.7)
configure_file(foobar_version.cpp.in foobar_version.cpp @ONLY)
add_library(foobar_version STATIC ${CMAKE_CURRENT_BINARY_DIR}/foobar_version.cpp)
add_executable(foobar main.cpp)
target_link_libraries(foobar PRIVATE foobar_version)
add_library(fooToolkit mylib.cpp)
target_link_libraries(fooToolkit PRIVATE foobar_version)

```

## 19.3 Source Control Commits

让源码可以访问 git 信息, 比如当前 commit 的哈希值:

foobar_version.cpp.in

```cpp
std::string getFooBarGitHash()
{
return "@FooBar_GIT_HASH@";
}
```

CMakeLists.txt

```cmake
cmake_minimum_required(VERSION 3.0)
project(FooBar VERSION 2.4.7)

find_package(Git REQUIRED)
execute_process(
 COMMAND ${GIT_EXECUTABLE} rev-parse HEAD
 RESULT_VARIABLE result
 OUTPUT_VARIABLE FooBar_GIT_HASH
 OUTPUT_STRIP_TRAILING_WHITESPACE
)
if(result)
message(FATAL_ERROR "Failed to get git hash: ${result}")
endif()
configure_file(foobar_version.cpp.in foobar_version.cpp @ONLY)
```

# Chapter 20. Libraries

## 20.1 Build Basics

```cmake
add_library(targetName [STATIC | SHARED | MODULE | OBJECT]
  [EXCLUDE_FROM_ALL]
  source1 [source2 ...])
```

## 20.2 Linking Static Libraries

## 20.3 Shared Library Versioning

如果动态库只在项目内部使用, 那么可以不提供版本信息. 如果动态库需要发布, 需要带上版本信息.

Linux 动态库的例子:

```txt
libmystuff.so.2.4.3
libmystuff.so.2 --> libmystuff.so.2.4.3
libmystuff.so --> libmystuff.so.2
```

动态库的版本由 VERSION 和 SOVERSION 两个 target property 控制. VERSION 将设置 major.minor.patch, SOVERSION 只设置 major.

```cmake
add_library(mystuff SHARED source1.cpp ...)
set_target_properties(mystuff PROPERTIES
 VERSION 2.4.3
 SOVERSION 2
)
```

## 20.4 Interface Compatibility

# Chapter 21. Toolchains And Cross Compiling

## 21.1 Toolchain Files

指定 toolchain file:

```shell
cmake -DCMAKE_TOOLCHAIN_FILE=myToolchain.cmake path/to/source
```

## 21.2 Defining The Target System

描述 Target System 最基本的变量有:

**CMAKE_SYSTEM_NAME**

target platform 的类型. Linux, Windows 等, 裸机嵌入式设备可以设置为 Generic.

**CMAKE_SYSTEM_PROCESSOR**

target architecture 类型.

**CMAKE_SYSTEM_VERSION**

target system 的版本.

## 21.3 Tool Selection

`CMAKE_<LANG>_COMPILER`: 来设置 compiler 路径. 如果 `CMAKE_<LANG>_COMPILER` 没设置, 可以设置环境变量, 比如 C compiler 设置`CC`.

当 `CMAKE_<LANG>_COMPILER` 设置好后, cmake 会自动 detect `CMAKE_<LANG>_COMPILER_ID` 如 GNU, Clang, 和 `CMAKE_<LANG>_COMPILER_VERSION` 编译器版本.

`CMAKE_<LANG>_FLAGS`, `CMAKE_<LANG>_FLAGS_<CONFIG>`, `CMAKE_<TARGETTYPE>_LINKER_FLAGS` and `CMAKE_<TARGETTYPE>_LINKER_FLAGS_<CONFIG>` compiler 和 linker 默认的 flags 也会随着 compile path 确定下来. 可以通过设置 XXX_INIT 来 append flags.

```cmake
set(CMAKE_C_COMPILER gcc)
set(CMAKE_CXX_COMPILER g++)
set(extraOpts "-Wall -Wextra")
set(CMAKE_C_FLAGS_DEBUG_INIT ${extraOpts})
set(CMAKE_CXX_FLAGS_DEBUG_INIT ${extraOpts})
```

## 21.4 System Roots

大部分情况，设置好工具链就足够了，但有的项目可能需要访问 target 平台上的 libraries, headers.
这时候需要设置`CMAKE_SYSROOT`, target 的根目录.

## 21.5 Compiler Checks

try_compile()

//TODO: 什么作用

# Chapter 22. Apple Features

skip
