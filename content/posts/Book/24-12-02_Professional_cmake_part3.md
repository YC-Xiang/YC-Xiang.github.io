---
date: 2024-11-29T11:09:33+08:00
title: "Professional CMake: A Practical Guide Part III The Bigger Picture"
tags:
  - CMake
categories:
  - Book
---

# Chapter 23. Finding Things

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

cmake 有两种 package, 分别是 module 和 config.

find_package()也有两种形式, 一种 short term 支持寻找 module 和 config, 另一种 long term 只支持寻找 config.

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

COMPONENTS: 必须的组件.

OPTIONAL_COMPONENTS: 可选的组件.

version: 指定要找的 package version. EXACT: 需要匹配准确的 version.

REQUIRED: 如果没找到 package, 会打印错误信息.

QUIET: 如果没找到 package, 不会打印错误信息.

MODULE 和 NO_POLICY_SCOPE 这两个关键字不推荐使用.

e.g.

```cmake
find_package(Qt5 5.9 REQUIRED
  COMPONENTS Gui
  OPTIONAL_COMPONENTS DBus
)
# 如果有REQUIRED关键字,没有optional components, 那么COMPNENTS可以省略
find_package(Qt5 5.9 REQUIRED Gui Widgets Network)
```

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

include(GNUInstallDirs)会提供一系列目录`CMAKE_INSTALL_<dir>`, dir 的取值有:

BINDIR

SRINDIR

LIBDIR

LIBEXECDIR

INCLUDEDIR

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

entityType 选项有:

RUNTIME: install executable binaries.

LIBRARY: install shared library.

ARCHIVE: install static library.

OBJECTS: install objects in object library.

PUBLIC_HEADER: install files in PUBLIC_HEADER property.

PRIVATE_HEADER: install files in PRIVATE_HEADER property.

RESOURCE: install files in RESOURCE property.

# Chapter 26. Packaging

# Chapter 27. External Content

# Chapter 28. Project Organization
