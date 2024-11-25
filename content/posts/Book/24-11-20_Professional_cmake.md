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

**REPLACE** 从 input string(如果有多个 input 会组合到一起) 中找到 match string, 替换成 replacewith, 保存到 outVar.

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

注意 regex 中括号用来捕获组, replacewith 中可以用`\\1`, `\\2`来引用 regex 中的匹配项. 因为`\`需要转义,所以需要两个.

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

底层列表只是一个字符串，列表项由分号分隔.

列表各种操作:

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

其中 if 判断,对于没引号的常量(对大小写不敏感):

- true: 1, ON, YES, TRUE, Y or a non-zero number.
- false: 0, OFF, NO, FALSE, N, IGNORE, NOTFOUND, an empty string
  or a string that ends in -NOTFOUND.

如果不是上面提供的常量, 那么会进一步分析:

- 未加引号的变量名: 和 false 常量比较, 如果这些都不匹配, 则表达式的结果为真。未定义的变量为空字符串，也为 false.
- 加引号的变量名: cmake 3.1 后, 全部为 false.

> TODO: 除非特殊设定 见 Chapter12 policies

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
