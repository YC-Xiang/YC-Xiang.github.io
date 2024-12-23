---
date: 2024-11-30T11:09:33+08:00
title: "Professional CMake: A Practical Guide Part III The Bigger Picture"
tags:
  - CMake
categories:
  - Book
---

# Chapter 23. Finding Things

中等规模以上的项目除了本身的项目之外, 可能还依赖于其他东西. 比如 a particular library or tool, location of a specific configuration file or a header for a library. 甚至项目可能需要找一个 package, 其中定义了一系列内容, 包括 targets, functions, variables...

`find_...`命令提供了搜索 file、library 或 progaram，甚至 package 的能力.

## 23.1 Finding Files and Paths

```cmake
find_file(outVar
  name | NAMES name1 [name2...]
  [HINTS path1 [path2...] [ENV var]...]
  [PATHS path1 [path2...] [ENV var]...]
  [PATH_SUFFIXES suffix1 [suffix2 ...]]
  [NO_DEFAULT_PATH]
  [NO_PACKAGE_ROOT_PATH]
  [NO_CMAKE_PATH]
  [NO_CMAKE_ENVIRONMENT_PATH]
  [NO_SYSTEM_ENVIRONMENT_PATH]
  [NO_CMAKE_SYSTEM_PATH]
  [CMAKE_FIND_ROOT_PATH_BOTH |
  ONLY_CMAKE_FIND_ROOT_PATH |
  NO_CMAKE_FIND_ROOT_PATH]
  [DOC "description"]
)
```

搜索顺序按如下表格:

![](https://xyc-1316422823.cos.ap-shanghai.myqcloud.com/20241206173114.png)

**Package root variables**

仅适用于在 Find Module 中调用 find_file() 的情况.

**Cache variables (CMake-specific)**

CMAKE_PREFIX_PATH, CMAKE_INCLUDE_PATH and CMAKE_FRAMEWORK_PATH.

CMAKE_PREFIX_PATH 会从`<prefix>/include`中查找, 另外两个直接在变量路径中查找.

CMAKE_INCLUDE_PATH 适用于 find_file()和 find_path(). CMAKE_FRAMEWORK_PATH 适用于前两者+find_library().

**Environment variables (CMake-specific)**

CMAKE_PREFIX_PATH, CMAKE_INCLUDE_PATH and CMAKE_FRAMEWORK_PATH 三个同名环境变量.

**Environment variables (system-specific)**

INCLUDE 和 PATH 两个环境变量.

**Cache variables (platform-specific)**

CMAKE_SYSTEM_PREFIX_PATH, CMAKE_SYSTEM_INCLUDE_PATH and CMAKE_SYSTEM_FRAMEWORK_PATH.

会由 platform toolchain 自动设置.

**HINTS and PATHS**

上面讨论的变量都是由项目之外设置的，但是 HINTS 和 PATHS 选项是项目本身应该增加的额外搜索路径.

PATHS 一般是固定的路径, 而 HINTS 一般是由其他变量计算展开得到的. HINTS 的路径优先级比 PATHS 高.

</br>

图片中 skip option 中的选项可以用来跳过搜索某些路径. NO_DEFAULT_PATH 可以跳过除了 HINTS 和 PATHS 的所有搜索路径.

</br>

**PATH_SUFFIXES**: 会增加搜索每个搜索路径下的子文件夹.

</br>

find_file()还有一种缩略形式, 可以只搜索指定的路径:

```cmake
find_file(outVar name [path1 [path2...]])
```

</br>

如果想更改 find_file 的搜索顺序, 先搜索我们想搜索的路径, 如果不存在, 再搜索默认的路径, 可以使用如下多次调用的方式:

```cmake
find_file(FOO_HEADER foo.h PATHS /opt/foo/include NO_DEFAULT_PATH)
find_file(FOO_HEADER foo.h)
```

</br>

CMAKE_FIND_ROOT_PATH_BOTH, ONLY_CMAKE_FIND_ROOT_PATH, NO_CMAKE_FIND_ROOT_PATH 三个变量可以改变搜索顺序:

![](https://xyc-1316422823.cos.ap-shanghai.myqcloud.com/20241209110236.png)

## 23.1.2 Cross-compilation Controls

如果是交叉编译, 那么需要修改搜索的根目录, 通过 CMAKE_FIND_ROOT_PATH 或者 CMAKE_SYSROOT(只能在 toolchain file 中修改).

## 23.2 Finding Paths

搜索目录, find_path(), 和 find_file()其他参数一模一样.

## 23.3 Finding Programs

和 find_file()基本一致, 不同点有:

**Cache variables (CMake-specific)**

对于 CMAKE_PREFIX_PATH, find_file() append 的是 include/, 而 find_program() append 的是 bin/, sbin/

CMAKE_INCLUDE_PATH 由 CMAKE_PROGRAM_PATH 代替.

CMAKE_FRAMEWORK_PATH 由 CMAKE_APPBUNDLE_PATH 代替.

**Environment variables (system-specific)**

只搜索 PATH, 不搜索 INCLUDE.

**General**

多了一个 NAMES_PER_DIR 选项, 可以在 NAMES 列出来的文件中, 倒过来搜索.

## 23.4 Finding Libraries

和 find_file()基本一致, 不同点有:

**Cache variables (CMake-specific)**

对于 CMAKE_PREFIX_PATH, append 的是 lib/

**Environment variables (system-specific)**

PATH 和 LIB

**General**

多了和 find_program()一样的 NAMES_PER_DIR.

## 23.5 Finding Packages

cmake 定义 package 有两种方式, 分别是 module 和 config details.

find_package()提供了两种形式, 一种 short term 支持寻找 module 和 config, 另一种 long term 只支持寻找 config.

short term 通常应该是首选, 因为它更简单. 然而, 长格式提供了更多的搜索控制, 使它在某些情况下灵活.

short term 形式如下:

```cmake
find_package(packageName
  [version [EXACT]]
  [QUIET] [REQUIRED]
  [[COMPONENTS] component1 [component2...]]
  [OPTIONAL_COMPONENTS component3 [component4...]]
  [MODULE]
  [NO_POLICY_SCOPE]
)
```

version: 指定最低的 package version. EXACT: 需要匹配准确的 version.

QUIET: 如果没找到 package, 不会打印错误信息.

REQUIRED: 如果没找到 package, 会打印错误信息.

COMPONENTS: 指定 package 中必须的组件.

OPTIONAL_COMPONENTS: 指定 package 中可选的组件.

MODULE 和 NO_POLICY_SCOPE 这两个关键字不推荐使用.

e.g.

例如，下面的调用需要找到 Qt 5.9 或更高版本，并且 Gui 组件是必选，DBus 模块是可选。

```cmake
find_package(Qt5 5.9 REQUIRED
  COMPONENTS Gui
  OPTIONAL_COMPONENTS DBus
)
# 当出现 REQUIRED 选项时，可以省略 COMPONENTS 关键字，并将强制性组件放置在 REQUIRED 之后.
find_package(Qt5 5.9 REQUIRED Gui Widgets Network)
```

https://blog.csdn.net/zhanghm1995/article/details/105466372

FindXXX.cmake for non-native CMake software.

XXX-config.cmake for CMake-based software.

</br>

module 的 find_package()会寻找`Find<packageName>.cmake`, 而 config 会找`<packageName>Config.cmake`或者`<lowercasePackageName>-config.cmake`.

### 23.5.1 Package Registries

CMake 支持一种 package 注册表形式，来处理 package 分散在不同的地方.

可以通过 export(PACKAGE packageName) 将 package 添加到 user registry:

```txt
~/.cmake/packages/<packageName>/
```

### 23.5.2 FindPkgConfig

适用于没提供 cmake config file, 而是提供了 pkg-config file.

# Chapter 24. Testing

# Chapter 25. Installing

## 25.1 Directory Layout

### 25.1.1 Relative Layout

cmake 的 GNUInstallDirs 模块提供了非常方便的方法来使用标准目录布局。`include(GNUInstallDirs)`命令会提供一系列目录`CMAKE_INSTALL_<dir>`, dir 的取值有:

BINDIR: 直接运行的可执行文件、脚本和符号链接的位置。默认为 bin

SRINDIR: 与 BINDIR 相似，不过是针对有系统管理权限的情况。默认为 sbin

LIBDIR: 库和编译文件的路径。根据主机/目标平台，默认设置为 lib

LIBEXECDIR: 不直接由用户调用的可执行文件，但可以通过启动脚本或位于 BINDIR 中的符号链接的方式运行。默认为 libexec

INCLUDEDIR: 头文件目录。默认为 include

DATAROOTDIR

DATADIR

MANDIR

DOCDIR

### 25.1.2 Base Install Location

**CMAKE_INSTALL_PREFIX** 变量控制 install 目录, Unix 默认安装目录为/usr/local.

For add-on packages, 推荐安装到`/opt/<package>`或`/opt/<provider>/<package>`目录下.

**CMAKE_STAGING_PREFIX** 变量可以用于交叉编译的场景, 多一个 install 的备份目录.

**DESTDIR** 直接用于 build tool, 控制安装目录.

```shell
make DESTDIR=/home/me/staging install
env DESTDIR=/home/me/staging ninja install
```

## 25.2 Installing Targets

```cmake
install(TARGETS targets...
  [EXPORT exportName]
  [CONFIGURATIONS configs...]
  # One or more blocks of the following
  [ [entityType]
  DESTINATION dir
  [PERMISSIONS permissions...]
  [NAMELINK_ONLY | NAMELINK_SKIP]
  [COMPONENT component]
  [NAMELINK_COMPONENT component] # CMake 3.12 or later only
  [EXCLUDE_FROM_ALL]
  [OPTIONAL]
  [CONFIGURATIONS configs...]
  ]...
  # Special case
  [INCLUDES DESTINATION incDirs...]
)
```

**entityType** 指定如何安装目标的不同部分, 选项有:

RUNTIME: install executable binaries.

LIBRARY: install shared library.

ARCHIVE: install static library.

OBJECTS: install objects in object library.

PUBLIC_HEADER: install files in PUBLIC_HEADER property.

PRIVATE_HEADER: install files in PRIVATE_HEADER property.

RESOURCE: install files in RESOURCE property.

e.g.

```cmake
install(TARGETS mySharedLib myStaticLib
 RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR}
 LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
 ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR}
)
```

如果只有一种类型, 可以省略 entityType.

```cmake
# Targets are both executables, so specifying the entity type isn't needed
install(TARGETS exe1 exe2
 DESTINATION ${CMAKE_INSTALL_BINDIR}
)
```

**PERMISSION** 设置目标安装后的权限, 可选值有:

![](https://xyc-1316422823.cos.ap-shanghai.myqcloud.com/20241217100505.png)

</br>

安装库文件时,

```cmake
libmyShared.so.1.3.2
libmyShared.so.1 --> libmyShared.so.1.3.2
libmyShared.so --> libmyShared.so.1
```

**NAMELINK_ONLY**: 只安装 libmyShared.so

**NAMELINK_SKIP**: 安装 libmyShared.so.1.3.2 和 libmyShared.so.1

NAMELINK_ONLY 和 NAMELINK_SKIP 不能共存, 需要分开安装.

**COMPONENT**: 指定 install component name, 其中规定了一系列 install rule.

```cmake
install(TARGETS myShared myStatic
   RUNTIME
       DESTINATION ${CMAKE_INSTALL_BINDIR}
       COMPONENT MyProj_Runtime
   LIBRARY
       DESTINATION ${CMAKE_INSTALL_LIBDIR}
       NAMELINK_SKIP
       COMPONENT MyProj_Runtime
   ARCHIVE
     DESTINATION ${CMAKE_INSTALL_LIBDIR}
     COMPONENT MyProj_Development
)

install(TARGETS myShared
   LIBRARY
     DESTINATION ${CMAKE_INSTALL_LIBDIR}
     NAMELINK_ONLY
     COMPONENT MyProj_Development
)
```

</br>

**CONFIGURATIONS**: 根据 build type 来选择不同的安装方式

```cmake
install(TARGETS myStatic
 ARCHIVE
 DESTINATION ${CMAKE_INSTALL_LIBDIR}/Debug
 CONFIGURATIONS Debug
)
install(TARGETS myStatic
 ARCHIVE
 DESTINATION ${CMAKE_INSTALL_LIBDIR}/Release
 CONFIGURATIONS Release RelWithDebInfo MinSizeRel
)
```

CONFIGURATIONS 关键字还可以位于所有参数的前面，并作为没提供配置的默认值的默认值。下面的示例中，除了为 debug 和 release 安装的 ARCHIVE 外，所有的块都只为 release 安装。

```cmake
install(TARGETS myShared myStatic
 CONFIGURATIONS Release
 RUNTIME
 DESTINATION ${CMAKE_INSTALL_BINDIR}
 LIBRARY
 DESTINATION ${CMAKE_INSTALL_LIBDIR}
 ARCHIVE
 DESTINATION ${CMAKE_INSTALL_LIBDIR}
 CONFIGURATIONS Debug Release
)
```

### 25.2.1 Interface Properties

任何链接到 foo 的内容都会在头文件搜索路径中添加一个 anotherDir. 当 foo 安装时，可以打包并部署到完全不同的机器上。显然 anotherDir 的路径将不再有意义.

```cmake
add_library(foo STATIC ...)
target_include_directories(foo
 INTERFACE ${CMAKE_CURRENT_BINARY_DIR}/somewhere
${MyProject_BINARY_DIR}/anotherDir
)
install(TARGETS foo
 DESTINATION ...
)

```

构建时使用头文件路径 xxx，安装时使用头文件路径 yyy:

```cmake
include(GNUInstallDirs)
target_include_directories(foo
 INTERFACE
 $<BUILD_INTERFACE:${CMAKE_CURRENT_BINARY_DIR}/somewhere>
 $<BUILD_INTERFACE:${MyProject_BINARY_DIR}/anotherDir>
 $<INSTALL_INTERFACE:${CMAKE_INSTALL_INCLUDEDIR}>
)
```

这样为每个 target 都指定 install 后的头文件路径比较麻烦, 因为安装后 target 共享相同的头文件搜索路径, 可以在 install 命令中统一指定.

```cmake
add_library(myStatic STATIC ...)
add_library(myHeaderOnly INTERFACE ...)
target_include_directories(myStatic
 PUBLIC $<BUILD_INTERFACE:${CMAKE_CURRENT_BINARY_DIR}/static_exports>
)
target_include_directories(myHeaderOnly
 INTERFACE $<BUILD_INTERFACE:${CMAKE_CURRENT_LIST_DIR}>
)
install(TARGETS myStatic myHeaderOnly
 ARCHIVE
   DESTINATION ${CMAKE_INSTALL_LIBDIR}
 INCLUDES
   DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}
)

```

### 25.2.2. RPATH

linux 平台使用`LD_LIBRARY_PATH`环境变量来查找动态库.

可以使用 RPATH 在 cmake 中指定动态库查找路径, 这样就可以不依赖于环境变量.

RPATH 种类有, CMAKE_BUILD_RPATH, CMAKE_INSTALL_RPATH 等.

e.g.

```cmake
set(CMAKE_INSTALL_RPATH $ORIGIN $ORIGIN/../lib)
```

## 25.3 Installing Exports

安装导出信息, xxx.cmake. 这样其他项目可以通过 find_package()来获取相关信息.

```cmake
install(EXPORT exportName
  DESTINATION dir
  [FILE name.cmake]
  [NAMESPACE namespace]
  [PERMISSIONS permissions...]
  [EXPORT_LINK_INTERFACE_LIBRARIES]
  [COMPONENT component]
  [EXCLUDE_FROM_ALL]
  [CONFIGURATIONS configs...]
)
```

## 25.4 Installing Files And Directories

安装文件:

```cmake
install(<FILES | PROGRAMS> files...
  DESTINATION dir
  [RENAME newName]
  [PERMISSIONS permissions...]
  [COMPONENT component]
  [EXCLUDE_FROM_ALL]
  [OPTIONAL]
  [CONFIGURATIONS configs...]
)
```

install(FILES) 和 install(PROGRAMS) 的区别是，如果没有 PERMISSIONS ，后者会在默认情况下添加执行权限。

</br>

安装目录:

```cmake
install(DIRECTORY dirs...
 DESTINATION dir
 [FILE_PERMISSIONS permissions... | USE_SOURCE_PERMISSIONS]
 [DIRECTORY_PERMISSIONS permissions...]
 [COMPONENT component]
 [EXCLUDE_FROM_ALL]
 [OPTIONAL]
 [CONFIGURATIONS configs...]
 [MESSAGE_NEVER]
 [FILES_MATCHING]
# The following block can be repeated as many times as needed
 [ [PATTERN pattern | REGEX regex]
 [EXCLUDE]
 [PERMISSIONS permissions...] ]
)
```

## 25.5 Custom Install Logic

```cmake
install(SCRIPT fileName | CODE cmakeCode
  [COMPONENT component]
  [EXCLUDE_FROM_ALL]
)
```

e.g.

```cmake
install(CODE [[ message("Starting custom script") ]]
  SCRIPT myCustomLogic.cmake
  CODE [[ message("Finished custom script") ]]
  COMPONENT MyProj_Runtime
)
```

# Chapter 26. Packaging

cpack 打包

// TODO:

# Chapter 27. External Content

引入外部项目 ExternalProject, FetchContent 两个 module.

// TODO:

# Chapter 28. Project Organization

// TODO:
