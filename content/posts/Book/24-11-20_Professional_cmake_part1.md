---
date: 2024-11-20T21:14:33+08:00
title: "Professional CMake: A Practical Guide Part I Fundamentals"
tags:
  - CMake
categories:
  - Book
---

# Reference

[Professional CMake A Practical Guide](https://crascit.com/professional-cmake/) version 1.0.0

2018 年出版, 基于 cmake 3.12 版本(2018-7-17).

目前该书出到了 19th Edition, 支持到 cmake 3.30, 不断更新中.

# Chapter 1. Introduction

From Wikipedia: CMake is a free, cross-platform, software development tool for building applications via compiler-independent instructions. It also can automate testing, packaging and installation. It runs on a variety of platforms and supports many programming languages.

![](https://xyc-1316422823.cos.ap-shanghai.myqcloud.com/20241210175607.png)

Build 部分由其他不同的 build tool 负责, 比如 make, ninja, Visual Studio, XCode...

# Chapter 2. Setting Up a Project

## 2.1 In-source Builds

source directory 和 build directory 相同.

## 2.2 Out-of-source Builds

build 目录单独出来, 可以有多个 build 目录, 比如一个给 debug version, 一个给 release version.

![](https://xyc-1316422823.cos.ap-shanghai.myqcloud.com/20241210203803.png)

## 2.3 Generating Project Files

运行 cmake:

```cmake
mkdir build
cd build
cmake -G "Unix Makefiles" ../source
```

如果不提供-G 选项, 会根据 host platform 自动选择 generator.

</br>

新版本的 cmake 可以使用-S, -B 指定 source 和 build 目录:

```cmake
~/package $ cmake -S source -B build
```

</br>

```shell
-- Configuring done
-- Generating done
-- Build files have been written to: /some/path/build
```

分 configure 和 generate 两个阶段, configure 阶段读取 CMakeLists.txt, generate 阶段创建 project files.

## 2.4 Running The Build Tool

经过上面 cmake 的命令后, 此时可以运行 build tool 了, 比如进入 build 目录执行`make`.

或者，cmake 也支持直接调用 build tool:

```shell
cmake --build /some/path/build --target MyApp
```

--target 可以省略, cmake 会编译默认的 target.

# Chapter 3. A Minimal Project

所有的 CMake 项目都是从一个名为 CMakeLists.txt 的文件开始的，通常放置在 source tree 的顶部。

```cmake
cmake_minimum_required(VERSION 3.2)
project(MyApp)
add_executable(myExe main.cpp)
```

arguments 以空格分开, 或者空行.

```cmake
add_executable(myExe
  main.cpp
  src1.cpp
  src2.cpp
)
```

cmake 命令大小写不敏感, 但通常小写.

```cmake
add_executable(myExe main.cpp)
ADD_EXECUTABLE(myExe main.cpp)
Add_Executable(myExe main.cpp)
```

## 3.1 Managing CMake versions

CMakeLists.txt 第一行一般指定 cmake 最低要求的版本, 如果 cmake 低于该版本, 会中止报错.

```cmake
cmake_minimum_required(VERSION major.minor[.patch[.tweak]])
```

## 3.2 The project() Command

```cmake
project(projectName
  [VERSION major[.minor[.patch[.tweak]]]]
  [LANGUAGES languageName ...]
)
```

project name 是必须的, VERSION 用来指定版本, 会生成一些版本相关的变量. LANGUAGES 用来指定编程语言,
如果不指定, 默认为 C 和 CXX.

## 3.3 Building A Basic Executable

```cmake
add_executable(targetName source1 [source2 ...])
```

# Chapter 4. Building Simple Targets

## 4.1 Executables

更详细的

```cmake
add_executable(targetName [WIN32] [MACOSX_BUNDLE]
[EXCLUDE_FROM_ALL]
source1 [source2 ...]
)
```

EXCLUDE_FROM_ALL: exclude target from the "all" target. 这样需要 cmake 命令主动去 build the target.

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

可以不指定 type, 那么由 cmake 变量 BUILD_SHARED_LIBS 决定默认类型. 如果为 true 则是 shared library, 否则为 static library.

## 4.3 Linking Targets

```cmake
target_link_libraries(targetName
<PRIVATE|PUBLIC|INTERFACE> item1 [item2 ...]
[<PRIVATE|PUBLIC|INTERFACE> item3 [item4 ...]]
...
)
```

A 链接 B, Libraries 之间的依赖关系有下面三种:

PRIVATE: 只有 A 链接 B, 其他链接 A 的 targets 不知道 B 的存在.

PUBLIC: 其他链接 A 的 targets 也会链接 B.

INTERFACE: A 不链接 B, 其他链接 A 的 targets 链接 B.

![](https://xyc-1316422823.cos.ap-shanghai.myqcloud.com/20241217115341.png)

## 4.4 Linking Non-targets

target_link_libraries 中的 item 除了可以传入 cmake targets, 还可以传:

Full path to a library file: 库文件(.a, .so)的绝对路径.

Plain library name: 库文件名称, 比如 foo 对应-lfoo, linker 会自己从库路径中找.

Link flag: 可以传 flags 给 linker command.

# Chapter 5. Variables

## 5.1 Variable Basics

```cmake
set(varName value... [PARENT_SCOPE])
unset(varName)
```

PARENT_SCOPE 用来提升变量的作用域, 这个后面章节会讲.

CMake 将所有变量都作为字符串处理, 不用加引号, 除非中间有空格.

如果提供了多个 value, 那么就是 cmake 中的 lists, 每个 value 由`;`隔开, 组成最终一个字符串.

```cmake
set(myVar a b c) # myVar = "a;b;c"
set(myVar a;b;c) # myVar = "a;b;c"
set(myVar "a b c") # myVar = "a b c"
set(myVar a b;c) # myVar = "a;b;c"
set(myVar a "b c") # myVar = "a;b c"
```

变量的值通过 ${myVar} 获得, 可以递归使用. 未定义的变量是空字符串.

```cmake
set(foo ab) # foo = "ab"
set(bar ${foo}cd) # bar = "abcd"
set(baz ${foo} cd) # baz = "ab;cd"
set(myVar ba) # myVar = "ba"
set(big "${${myVar}r}ef") # big = "${bar}ef" = "abcdef"
set(${foo} xyz) # ab = "xyz"
set(bar ${notSetVar}) # bar = ""
```

字符串不限于单行, 它们可以包含嵌入的换行符. 它们还可以包含引号, 需要用反斜杠转义.

```cmake
set(myVar "goes here")
set(multiLine "First line ${myVar}
Second line with a \"quoted\" word")
```

为了防止转义和变量替换, 可以使用类似 lua 的`[[...]]`语法, `[[`中间可以加任意`=`:

```cmake
# Simple multi-line content with bracket syntax,
# no = needed between the square bracket markers
set(multiLine [[
First line
Second line
]])
# Bracket syntax prevents unwanted substitution
set(shellScript [=[
#!/bin/bash
[[ -n "${USER}" ]] && echo "Have USER"
]=])
# Equivalent code without bracket syntax
set(shellScript
"#!/bin/bash
[[ -n \"\${USER}\" ]] && echo \"Have USER\"
")
```

解除变量, 下面两句是等价的:

```cmake
set(myVar)
unset(myVar)
```

## 5.2 Environment Variables

cmake 支持设置环境变量, 不过只在 cmake 执行时有效.

```cmake
set(ENV{PATH} "$ENV{PATH}:/opt/myDir")
```

## 5.3 Cache Variables

除了常规变量之外, cmake 还支持缓存变量. 缓存变量存储在 CMakeCache.txt 的特殊文件中, 并且在 cmake 运行期间持久存在. 一旦设置好了, 缓存变量就会保持不变, 直到显式地将它们从缓存中删除.

```cmake
set(varName value... CACHE type "docstring" [FORCE])
```

cache variable 的 type 只影响 cmake GUI, 有下面几种类型:

- BOOL: ON/OFF, TRUE/FALSE, 1/0...
- FILEPATH: 文件路径
- PATH: 目录路径
- STRING: 字符串
- INTERNAL: 内部变量, 不会在 GUI 中显示

</br>

设置 BOOL cache variable 有一种简化方式:

```cmake
option(optVar helpString [initialValue])
```

如果 initialValue 没有提供, 那么默认为 OFF.

相当于:

```cmake
set(optVar initialValue CACHE BOOL helpString)
```

注意, set()指令, 对 cache variable 不会覆盖, 是一种 set-if-not-set 的机制, 除非指定 **FORCE** 关键字可以覆盖.

## 5.4 Manipulating Cache Variables

cache variable 可以通过命令行设置, 相当于 set()中设置了 CACHE 和 FORCE, type 和 INTERNAL 类似, docstring 为空.

```cmake
cmake -D myVar:type=someValue ...

cmake -D foo:BOOL=ON ...
cmake -D "bar:STRING=This contains spaces" ...
cmake -D hideMe=mysteryValue ...
cmake -D helpers:FILEPATH=subdir/helpers.txt ...
cmake -D helpDir:PATH=/opt/helpThings ...

cmake -U 'help*' -U foo ... # 删除变量
```

## 5.4.2 cmake GUI

// TODO:

## 5.5 Debugging Variables And Diagnostics

```cmake
message([mode] msg1 [msg2]...)
```

mode 的选择有:

- STATUS: 输出的消息带两个连字符
- WARNING: warning 颜色, 不会停止 cmake
- AUTHOR_WARNING: 需要 cmake 命令行传入-Wdev
- SEND_ERROR: cmake 不会立即停止, configure 会完成, generation 不会执行
- FATAL_ERROR: cmake 会直接停止
- DEPRECATION: 记录弃用消息

</br>

variable_watch 监视某个变量, 任何读取和修改都会被 log.

```cmake
variable_watch(myVar [command])
```

额外的 command 可以传入 cmake function/macro, 在每次读或者修改变量时都会执行.

## 5.6 String Handling

string 函数.

**FIND** 从 input string 中查找第一个找到的 sub string, 返回 index 保存到 outVar, REVERSE 表示反向查找.

```cmake
string(FIND inputString subString outVar [REVERSE])

set(longStr abcdefabcdef)
set(shortBit def)
string(FIND ${longStr} ${shortBit} fwdIndex)
string(FIND ${longStr} ${shortBit} revIndex REVERSE)
message("fwdIndex = ${fwdIndex}, revIndex = ${revIndex}")

fwdIndex = 3, revIndex = 9
```

</br>

**REPLACE** 替换子字符串, 从 input string(如果有多个 input 会组合到一起) 中找到 match string, 替换成 replacewith, 保存到 outVar.

```cmake
string(REPLACE matchString replaceWith outVar input [input...])
```

</br>

**REGEX** 支持正则替换.

```cmake
string(REGEX MATCH regex outVar input [input...])
string(REGEX MATCHALL regex outVar input [input...])
string(REGEX REPLACE regex replaceWith outVar input [input...])
```

MATCH: 返回正则匹配到的第一个, 保存到 outVar
MATCHALL: 返回正则匹配到的所有, 以 list 的形式保存到 outVar
REPLACE: 将 input 以正则匹配替换

注意 regex 中括号用来捕获组, replacewith 中可以用`\\1`, `\\2`来引用 regex 中的匹配项. 因为`\`也需要转义,所以需要两个`\\`.

```cmake
set(longStr abcdefabcdef)
string(REGEX MATCHALL "[ace]" matchVar ${longStr})
string(REGEX REPLACE "([de])" "X\\1Y" replVar ${longStr})
message("matchVar = ${matchVar}")
message("replVar = ${replVar}")

matchVar = a;c;e;a;c;e
replVar = abcXdYXeYfabcXdYXeYf
```

</br>

string 函数还支持一些操作:

```cmake
string(SUBSTRING input index length outVar) # 从index开始获取length长度的string
string(LENGTH input outVar) # 获取input长度
string(TOLOWER input outVar) # 转小写
string(TOUPPER input outVar) # 转大写
string(STRIP input outVar) # 删除空格
```

## 5.7 Lists

列表只是一个用分号分隔的列表项的单个字符串，cmake 提供了 list() 来简化操作.

**LENGTH** 获取 list 长度, **GET** 获取 list 元素

```cmake
list(LENGTH listVar outVar)
list(GET listVar index [index...] outVar)

set(myList a b c) # Creates the list "a;b;c"
list(LENGTH myList len)
message("length = ${len}")
list(GET myList 2 1 letters)
message("letters = ${letters}")

length = 3
letters = c;b
```

</br>

**APPEND** 在 list 尾部增加元素, **INSERT** 在 list 指定 index 增加元素.

```cmake
list(APPEND listVar item [item...])
list(INSERT listVar index item [item...])

set(myList a b c)
list(APPEND myList d e f)
message("myList (first) = ${myList}")
list(INSERT myList 2 X Y Z)
message("myList (second) = ${myList}")

myList (first) = a;b;c;d;e;f
myList (second) = a;b;X;Y;Z;c;d;e;f

```

</br>

**FIND** 查找指定元素的 index.

```cmake
list(FIND myList value outVar)

set(myList a b c d e)
list(FIND myList d index)
message("index = ${index}")

index = 3
```

</br>

还有一些其他操作:

```cmake
list(REMOVE_ITEM myList value [value...]) # 从list中删除指定value
list(REMOVE_AT myList index [index...]) # 删除指定index value
list(REMOVE_DUPLICATES myList) # 删除重复value

list(REVERSE myList) # 翻转list
list(SORT myList) # 排序list
```

## 5.8 Math

数学运算 支持 ` + - * / % | & ^ ~ << >> * / %.`

```cmake
math(EXPR outVar mathExpr)

set(x 3)
set(y 7)
math(EXPR z "(${x}+${y}) / 2")
message("result = ${z}")

result = 5
```

# Chapter 6. Flow Control

## 6.1 The if() Command

### 6.1.1 Basic Expressions

```cmake
if(expression1)
  # commands ...
elseif(expression2)
  # commands ...
else()
  # commands ...
endif()
```

其中 if 判断,对于未引用的常量(对大小写不敏感):

- true: 1, ON, YES, TRUE, Y or a non-zero number.
- false: 0, OFF, NO, FALSE, N, IGNORE, NOTFOUND, an empty string or a string that ends in -NOTFOUND.

如果不是上面提供的常量, 那么会进一步分析:

- 未加引号的变量名: 和 false 常量比较, 如果这些都不匹配, 则表达式的结果为真。未定义的变量为空字符串，也为 false.
- 加引号的变量名: cmake 3.1 后, 全部为 false.

### 6.1.2 Logic Operators

支持 AND OR NOT

```cmake
if(NOT expression)
if(expression1 AND expression2)
if(expression1 OR expression2)
```

### 6.1.3 Comparison Tests

三种类型的比较, numeric, string, version. 不同类型的比较符不同.

```cmake
if(value1 OPERATOR value2)
```

![](https://xyc-1316422823.cos.ap-shanghai.myqcloud.com/20241125172954.png)

cmake 会自动识别比较的类型:

```cmake
if(2 GREATER 1)
if("23" EQUAL 23)
set(val 42)
if(${val} EQUAL 42)
if("${val}" EQUAL 42)

if(1.2 VERSION_EQUAL 1.2.0)
if(1.2 VERSION_LESS 1.2.3)
if(1.2.3 VERSION_GREATER 1.2 )
if(2.0.1 VERSION_GREATER 1.9.7)
if(1.8.2 VERSION_LESS 2 )
```

注意字符串和数字也可以进行比较, 行为不确定.

</br>

**MATCHES**关键字支持正则匹配:

```cmake
if(value MATCHES regex)

if("Hi from ${who}" MATCHES "Hi from (Fred|Barney).*")
  message("${CMAKE_MATCH_1} says hello")
endif()
```

`${CMAKE_MATCH_<n>}`是捕获组.

### 6.1.4 File System Tests

支持文件系统相关的判断.

```cmake
if(EXISTS pathToFileOrDir)
if(IS_DIRECTORY pathToDir)
if(IS_SYMLINK fileName)
if(IS_ABSOLUTE path)
if(file1 IS_NEWER_THAN file2) # 需要绝对路径
```

文件系统操作符不支持任何${}变量展开.

### 6.1.5 Existence Tests

最后一类 if 表达式支持测试是否存在各种 cmake 实例.

```cmake
if(DEFINED name)
if(COMMAND name)
if(POLICY name)
if(TARGET name)
if(TEST name) # Available from CMake 3.4 onwards
```

**DEFINED**: 判断变量是否存在, 环境变量也可以

```cmake
if(DEFINED SOMEVAR) # Checks for a CMake variable
if(DEFINED ENV{SOMEVAR}) # Checks for an environment variable
```

**COMMAND**: 判断 function/macro 是否存在

**POLICY**: 通常是 CMPXXXX

**TARGET**: 判断由 add_executable(), add_library()创建的 target 是否存在

**TEST**: 判断由 add_test()创建的 test 是否存在

## 6.2 Looping

### 6.2.1 foreach()

两种形式, 第一种:

```cmake
foreach(loopVar arg1 arg2 ...)
  # ...
endforeach()
```

argN 是各种 value, loopVar 是循环变量

</br>

第二种:

```cmake
foreach(loopVar IN [LISTS listVar1 ...] [ITEMS item1 ...])
  # ...
endforeach()
```

相比第一种可以传入 list.

</br>

example:

```cmake
set(list1 A B)
set(list2)
set(foo WillNotBeShown)
foreach(loopVar IN LISTS list1 list2 ITEMS foo bar)
  message("Iteration for: ${loopVar}")
endforeach()
```

</br>

也支持类似 C 的循环:

```cmake
foreach(loopVar RANGE start stop [step])
foreach(loopVar RANGE value) # 相当于 foreach(loopVar RANGE 0 value)
```

### 6.2.2 while()

```cmake
while(condition)
  # ...
endwhile()
```

### 6.2.3 Interrupting Loops

支持 break()和 continue()

# Chapter 7. Using Subdirectories

## 7.1 add_subdirectory()

```cmake
add_subdirectory(sourceDir [ binaryDir ] [ EXCLUDE_FROM_ALL ])
```

### 7.1.1 Source And Binary Directory Variables

和 source, build directory 相关的 cmake 自带变量:

**CMAKE_SOURCE_DIR**: 顶层 CMakeLists.txt 所在目录.

**CMAKE_BINARY_DIR**: 顶层 Build 目录.

**CMAKE_CURRENT_SOURCE_DIR**: 当前处理的 CMakeLists.txt 目录.是通过 add_subdirectory()延伸的目录.

**CMAKE_CURRENT_BINARY_DIR**: 当前处理的 CMakeLists.txt 所在 build 目录中的位置. 是通过 add_subdirectory()延伸的目录.

### 7.1.2 Scope

- calling scope 定义的变量, current scope 可见.
- current scope 定义的变量, calling scope 不可见.
- current scope 修改 calling scope 的变量只在 current scope 有效, 离开 current scope 变量值恢复.

如果想在 current scope 修改 calling scope 的变量, 可以在 set 命令中指定**PARENT_SCOPE**参数.
注意这样只会修改 calling scope 变量的值, 而 current scope 的值不会被修改.

e.g.

```cmake
# CMakeLists.txt
set(myVar foo)
message("Parent (before): myVar = ${myVar}")
add_subdirectory(subdir)
message("Parent (after): myVar = ${myVar}")

## subdir/CMakeLists.txt
message("Child (before): myVar = ${myVar}")
set(myVar bar PARENT_SCOPE)
message("Child (after): myVar = ${myVar}")

Parent (before): myVar = foo
Child (before): myVar = foo
Child (after): myVar = foo # current scope的值不会被改
Parent (after): myVar = bar
```

## 7.2 include()

```cmake
include(fileName [OPTIONAL] [RESULT_VARIABLE myVar] [NO_POLICY_SCOPE])
include(module [OPTIONAL] [RESULT_VARIABLE myVar] [NO_POLICY_SCOPE])
```

第一条和 add_subdirectory()有些类似, 区别有:

- include() 传入的一般是以.cmake 结尾的文件名, 而 add_subdirectory()是包含 CMakeLists.txt 的目录.
- include() 不会创建新的 variable scope.
- 两者都会创建新的 policy scope, 但 include()可以通过 NO_POLICY_SCOPE 关键字不创建.
- include() 不会改变 CMAKE_CURRENT_SOURCE_DIR 和 CMAKE_CURRENT_BINARY_DIR.

在 include()后, CMAKE_CURRENT_SOURCE_DIR 仍然保持为调用 include()的文件目录路径.
而 CMAKE_CURRENT_LIST_DIR 可以更新到 include 文件的目录, 因此更推荐使用 CMAKE_CURRENT_LIST_DIR.

**CMAKE_CURRENT_LIST_DIR**: 当前文件的绝对路径.

**CMAKE_CURRENT_LIST_FILE**: 当前文件的绝对路径+文件名.

**CMAKE_CURRENT_LIST_LINE**: 当前文件的行号.

## 7.3 Ending Processing Early

return()可以中止处理当前文件, 返回上一级 caller.

在 cmake include 文件中可以 通过 return 像 C 头文件一样防止重复包含:

```cmake
if(DEFINED cool_stuff_include_guard)
  return()
endif()
set(cool_stuff_include_guard 1)
# ...
```

cmake 3.1 后直接使用:

```cmake
include_guard()
```

# Chapter 8. Functions and Macros

## 8.1 The Basics

```cmake
function(name [arg1 [arg2 [...]]])
  # Function body (i.e. commands) ...
endfunction()
macro(name [arg1 [arg2 [...]]])
  # Macro body (i.e. commands) ...
endmacro()
```

## 8.2 Argument Handling Essentials

function 和 macro 的区别是, function 的参数相当于定义了变量, 而 macro 参数是字符串,
后面调用的参数都是字符串替换.

func 和 macro 都自带的参数有:

ARGC: 参数个数

ARGV: 参数列表, 包括列出来的参数和没列出来的可变参数

ARGN: 参数列表, 只包括没列出来的可变参数

ARG{x}: 引用第 x 个参数

关于 ARGN 的作用, 参考例子:

```cmake
function(add_mytest targetName)
  add_executable(${targetName} ${ARGN})
  target_link_libraries(${targetName} PRIVATE foobar)
  add_test(NAME ${targetName}
  COMMAND ${targetName}
  )
endfunction()

add_mytest(smallTest small.cpp)
add_mytest(bigTest big.cpp algo.cpp net.cpp)
```

用 ${ARGN} 来表示所有的 unnamed \*.cpp 文件.

</br>

注意在宏中使用 ${ARGN} 会导致意外的结果,下面有个例子:

```cmake
# WARNING: This macro is misleading
macro(dangerous)
    # Which ARGN?
    foreach(arg IN LISTS ARGN)
      message("Argument: ${arg}")
    endforeach()
endmacro()

function(func)
  dangerous(1 2)
endfunction()

func(3)
```

这边的输出是 3, 因为 macro 只是替换, 这边的 ARGN 会是 func 函数中提供的 3.

```shell
Argument: 3
```

## 8.3 Keyword Arguments

cmake_parse_arguments 函数用来处理复杂的参数解析。

```cmake
cmake_parse_arguments(prefix
  noValueKeywords
  singleValueKeywords
  multiValueKeywords
  argsToParse)
```

**argsToParse** 通常传入${ARGN}.

**noValueKeywords** 为独立的关键字参数, 类似于开关.

**singleValueKeywords** 在关键字后需要一个额外的参数.

**multiValueKeywords** 在关键字后需要零个或额外多个参数.

在调用 cmake_parse_arguments 前需要设定好 prefix, noValueKeywords, singleValueKeywords, multiValueKeywords. prefix 作用是 cmake_parse_arguments 会生成带 prefix 前缀的相关变量, 例子如下:

```cmake
function(func)
    set(prefix ARG)
    set(noValues ENABLE_NET COOL_STUFF)
    set(singleValues TARGET)
    set(multiValues SOURCES IMAGES)

    cmake_parse_arguments(${prefix}
                        "${noValues}"
                        "${singleValues}"
                        "${multiValues}"
                        ${ARGN})

    message("Option summary:")
    foreach(arg IN LISTS noValues)
        if(${${prefix}_${arg}})
	        message(" ${arg} enabled")
        else()
            message(" ${arg} disabled")
        endif()
    endforeach()
endfunction()

func(SOURCES foo.cpp bar.cpp TARGET myApp ENABLE_NET)
func(COOL_STUFF TARGET dummy IMAGES here.png there.png gone.png)
```

输出为:

```cmake
Option summary:
  ENABLE_NET enabled
  COOL_STUFF disabled
  TARGET = myApp
  SOURCES = foo.cpp;bar.cpp
  IMAGES =
Option summary:
  ENABLE_NET disabled
  COOL_STUFF enabled
  TARGET = dummy
  SOURCES =
  IMAGES = here.png;there.png;gone.png
```

## 8.4 Scope

因为 cmake 的 function 没有返回值, 所以如果要修改上一层 scope 的变量, 那么需要在 set()命令中加上 PARENT_SCOPE 关键字.

```cmake
function(func resultVar1 resultVar2)
    set(${resultVar1} "First result" PARENT_SCOPE)
    set(${resultVar2} "Second result" PARENT_SCOPE)
endfunction()
```

function 和 macro 很大的一个区别是, function 创建了新的 scope, 而 macro 不会. 所以最好不要在 macro 中使用 PARENT_SCOPE, 可能和期望的行为不一致.

同理, 如果在 macro 中使用 return()会和 function 不同, function 是退出当前 function,
而 macro 替换后, return()可能会把上一级的 scope 退出掉.

## 8.5 Overriding Commands

重名的 `function()`可以覆盖, 旧函数会被替换成`__function()`

# Chapter 9. Properties

## 9.1 General Property Commands

property 是依附在某个实体上的属性, 主要是 set_property() 和 get_property()

```cmake
set_property(entitySpecific
 [APPEND] [APPEND_STRING]
 PROPERTY propName [value1 [value2 [...]]])
```

**entitySpecific** 必须是以下几种类型:

```txt
GLOBAL
DIRECTORY [dir]
TARGET [target1 [target2 [...]]]
SOURCE [source1 [source2 [...]]]
INSTALL [file1 [file2 [...]]]
TEST [test1 [test2 [...]]]
CACHE [var1 [var2 [...]]]
```

APPEND 和 APPEND_STRING 是两个可选的选项, 区别如下:

![](https://xyc-1316422823.cos.ap-shanghai.myqcloud.com/20241128095305.png)

</br>

获取 property 的 value 值.

```cmake
get_property(resultVar entitySpecific
 PROPERTY propName
 [DEFINED | SET | BRIEF_DOCS | FULL_DOCS])
```

几个可选参数:

**DEFINED**: 返回 boolean, 表示该 property 是否用 define_property() 定义过.

**SET**: 返回 boolean, 表示该 property 是否用 set_property() 设置过.

**BRIEF_DOCS**: 获取用 define_property() 定义的 BRIEF_DOCS.

**FULL_DOCS**: 获取用 define_property() 定义的 FULL_DOCS.

除了 SET, 另外三个使用的都不多.

</br>

```cmake
define_property(entityType
  PROPERTY propName [INHERITED]
  BRIEF_DOCS briefDoc [moreBriefDocs...]
  FULL_DOCS fullDoc [moreFullDocs...])
```

这个命令不设置 property 的值, 只提供文档和表示是否从别处继承.

如果使用了 INHERITED, 并且该属性没有在指定的范围内设置, 那么 get_property()将连接到父作用域.

## 9.2 Global Properties

全局属性通常和 build 相关, cmake 提供了 get_cmake_property()来获取全局 property:

```cmake
get_cmake_property(resultVar property)
```

property 可以是某个具体的 property 名, 或者是 cmake 定义的 property:

**VARIABLES**: 返回所有普通变量 list

**CACHE_VARIABLES**: 返回所有缓存变量 list

**COMMANDS**: 返回所有 cmake command, 自定义的 function, macro

**MACROS**: 返回所有自定义的 macro

**COMPONENTS**: 返回所有由 install()定义的 component

## 9.3 Directory Properties

CMake 提供了专门的命令来设置和获取目录属性, 比通用的 set_property(DIRECTORY...) 更简洁.

```cmake
set_directory_properties(PROPERTIES prop1 val1 [prop2 val2 ...])

get_directory_property(resultVar [DIRECTORY dir] property) # [DIRECTORY dir]未指定的话就是当前目录
get_directory_property(resultVar [DIRECTORY dir] DEFINITION varName) # 很少用
```

## 9.4 Target Properties

CMake 提供了专门的命令来设置和获取 target 属性.

```cmake
set_target_properties(target1 [target2...]
  PROPERTIES
  propertyName1 value1
  [propertyName2 value2] ... )
get_target_property(resultVar target propertyName)
```

## 9.5 Source Properties

CMake 提供了专门的命令来设置和获取单个 source 文件属性.

```cmake
set_source_files_properties(file1 [file2...]
  PROPERTIES
propertyName1 value1
[propertyName2 value2] ... )
get_source_file_property(resultVar sourceFile propertyName)
```

用的比较少, 如果文件的 property 修改了编译 flag, 那么会导致所有目标都需要重新构建, 带来极大的性能损失.

## 9.6 Cache Variable Properties

看起来主要是给 cmake GUI 用的, 跳过了.

## 9.7 Other Property Types

test 相关 property.

```cmake
set_tests_properties(test1 [test2...]
  PROPERTIES
  propertyName1 value1
  [propertyName2 value2] ... )
get_test_property(resultVar test propertyName)
```

# Chapter 10. Generator Expressions

运行 CMake 时，developers 通常将其视为一个单一的步骤，读取项目的 CMakeLists.txt 文件并生成相关的 build tool project files (Unix Makefile 或 Ninja input files...)

但其实分两个步骤, configure 和 generate.

configure 步骤读取和处理 CMakeLists.txt.

generate 步骤创建 build tool project files.

```cmake
-- Configuring done
-- Generating done
-- Build files have been written to: /some/path/build
```

当需要在 generate 甚至 install 阶段才展开的逻辑, 就需要用 Generator Expressions 生成器表达式.

## 10.1 Simple Boolean Logic

```cmake
$<1:...> # 返回...的内容
$<0:...> # 返回空字符串
$<BOOL:...> # ...为false返回0, 其他返回1

$<AND:expr[,expr...]> # 以下三个都返回逻辑运算结果0/1
$<OR:expr[,expr...]>
$<NOT:expr>

$<IF:expr,val1,val0> # expr为true返回val1, false返回val0
```

还支持 string, number, version 的比较:

```cmake
$<STREQUAL:string1,string2>
$<EQUAL:number1,number2>
$<VERSION_EQUAL:version1,version2>
$<VERSION_GREATER:version1,version2>
$<VERSION_LESS:version1,version2>
```

支持 build type 的判断, arg 是当前 build type 的话返回 1:

```cmake
$<CONFIG:arg>

target_link_libraries(myApp PRIVATE
 $<IF:$<CONFIG:Debug>,checkedAlgo,fastAlgo>
)
```

## 10.2 Target Details

另一种常见用法通过 generator expressions 来获取 target 信息:

```cmake
$<TARGET_PROPERTY:target,property>
$<TARGET_PROPERTY:property>
```

第一种形式从指定 target 中获取指定的 property.

第二种形式从 generator expressions 所在的 target 中检索指定的 property。

</br>

除了 TARGET_PROPERTY, 还有一些其他的表达式类型可以获取 target info:

`$<TARGET_FILE:target>`: 获取当前 target 的 binary path + name

`$<TARGET_FILE_NAME:target>`: 获取当前 target 的 binary name

`$<TARGET_FILE_DIR:target>`: 获取当前 target 的 binary path

这三个表达式在 build 阶段自定义 build rules, 拷贝文件时很有用.

</br>

object library, 可以通过`$<TARGET_OBJECTS:…>`来获取 source files.

```cmake
# Define an object library
add_library(objLib OBJECT src1.cpp src2.cpp)
# Define two executables which each have their own source
# file as well as the object files from objLib
add_executable(app1 app1.cpp $<TARGET_OBJECTS:objLib>)
add_executable(app2 app2.cpp $<TARGET_OBJECTS:objLib>)
```

## 10.3 General Information

除了可以获取 target information, 还可以获取 compiler, platform, build type 等等的 generator expressions:

`$<CONFIG>`: 判断 build tpye.

`$<PLATFORM_ID>`: 判断 platform 类型.

`$<C_COMPILER_VERSION>, $<CXX_COMPILER_VERSION>`: 判断编译器 version.

`$<LOWER_CASE:…>, $<UPPER_CASE:…>`: 大小写转换.

# Chapter 11. Modules

module 是在 cmake 核心语言特性上构建的 CMake 代码预构块。提供丰富的功能(定义一些自定义函数...)，项目可以用来完成各种各样的目标。

加载 module 有两种方式:

第一种通过 include, include(FooBar), cmake 就会查找对应的 FooBar.cmake 文件, 大小写敏感.

```cmake
include(module [OPTIONAL] [RESULT_VARIABLE myVar] [NO_POLICY_SCOPE])
```

</br>

查找 module 文件, 首先会到 CMAKE_MODULE_PATH 中查找, 如果没找到接着会到 cmake 内部的 module 目录查找.
可以将自定义的 modules 放在一个目录中, 然后加到 CMAKE_MODULE_PATH 中去.

```cmake
list(APPEND CMAKE_MODULE_PATH "${CMAKE_SOURCE_DIR}/cmake")
```

![](https://xyc-1316422823.cos.ap-shanghai.myqcloud.com/20241224140954.png)

</br>

第二种是通过 find_package(), 具体会在 第 23 章介绍.

```cmake
find_package(PackageName)
```

和 include 不同的点有, 以上述命令为例, find_package()会查找 FindPackageName.cmake 而不是 PackageName.cmake.

下面的章节介绍一些 cmake 内置的一些 modules.

## 11.1 Useful Development Aids

比如 CMakePrintHelpers module 提供了两个封装的打印函数用来打印 property 和 variable:

```cmake
cmake_print_properties([TARGETS target1 [target2...]]
  [SOURCES source1 [source2...]]
  [DIRECTORIES dir1 [dir2...]]
  [TESTS test1 [test2...]]
  [CACHE_ENTRIES var1 [var2...]]
  PROPERTIES property1 [property2...]
)
```

相当于是 get_property()和 message()的封装.

```cmake
add_executable(myApp main.c)
add_executable(myAlias ALIAS myApp)
add_library(myLib STATIC src.cpp)
include(CMakePrintHelpers)
cmake_print_properties(TARGETS myApp myib myAlias
  PROPERTIES TYPE ALIASED_TARGET)
```

输出结果：

```cmake
Properties for TARGET myApp:
  myApp.TYPE = "EXECUTABLE"
  myApp.ALIASED_TARGET = <NOTFOUND>
Properties for TARGET myLib:
  myLib.TYPE = "STATIC_LIBRARY"
  myLib.ALIASED_TARGET = <NOTFOUND>
Properties for TARGET myAlias:
  myAlias.TYPE = "EXECUTABLE"
  myAlias.ALIASED_TARGET = "myApp"
```

</br>

打印变量：

```cmake
cmake_print_variables(var1 [var2...])
```

```cmake
set(foo "My variable")
unset(bar)
include(CMakePrintHelpers)
cmake_print_variables(foo bar CMAKE_VERSION)
```

输出结果：

```cmake
foo="My variable" ; bar="" ; CMAKE_VERSION="3.8.2"
```

## 11.2 Endianness

TestBigEndian module 可以判断大小端:

```cmake
include(TestBigEndian)
test_big_endian(isBigEndian)
message("Is target system big endian: ${isBigEndian}")
```

## 11.3 Checking Existance and Support

一些用来 check 编译, 链接，运行结果的 modules.

```cmake
include(CheckCSourceCompiles)
check_c_source_compiles(code resultVar [FAIL_REGEX regex])

include(CheckCSourceRuns)
check_c_source_runs(code resultVar)

include(CheckCCompilerFlag)
check_c_compiler_flag(flag resultVar)

include(CheckSymbolExists)
check_symbol_exists(symbol headers resultVar)
```

// TODO:

# Chapter 12. Policies

## 12.1 Policy control

当项目中有一些部分(比如 imported lib)需要不同版本的 cmake version 时, 可以通过 cmake_policy()修改:

```cmake
cmake_policy(VERSION major[.minor[.patch[.tweak]]])
```

e.g.

```cmake
cmake_minimum_required(VERSION 3.7)
project(WithLegacy)
# Uses recent CMake features
add_subdirectory(modernDir)
# Imported from another project, relies on old behavior
cmake_policy(VERSION 2.8.11)
add_subdirectory(legacyDir)
```

cmake 3.12 后可以支持一定范围的 cmake 版本:

```cmake
cmake_minimum_required(VERSION 3.7...3.12)
cmake_policy(VERSION 3.7...3.12)
```

CMake 还提供了使用 SET 单独控制每个行为变化:

```cmake
cmake_policy(SET CMPxxxx NEW)
cmake_policy(SET CMPxxxx OLD)
```

## 12.2 Policy scope

可以通过 push, pop 操作来设置当前的 policy:

```cmake
cmake_policy(PUSH)
cmake_policy(POP)

# Save existing policy state
cmake_policy(PUSH)
# Set some policies to OLD to preserve a few old behaviors
cmake_policy(SET CMP0060 OLD) # Library path linking behavior
cmake_policy(SET CMP0021 OLD) # Tolerate relative INCLUDE_DIRECTORIES
# Do various processing here...
# Restore earlier policy settings
cmake_policy(POP)
```

add_subdirectory(), include(), find_package()三个命令, 相当于进入时隐式地调用 push(), 退出时调用 pop()
