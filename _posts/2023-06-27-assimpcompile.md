---
layout: post
title:  "assimp 编译"
author: "daweibayu"
tags: opengl
excerpt_separator: <!--more-->
---
<!--more-->


## 简介


 [assimp](https://github.com/assimp/assimp)，一个支持多种 3d 格式模型的导入库 
 本文以 [tag v5.2.3](https://github.com/assimp/assimp/tree/v5.2.3) 代码为例

 其实也有人做了 java 的包装，[kotlin assimp](https://github.com/kotlin-graphics/assimp)，我试了下，现在还是可用的，可以成功导入，但是看样子目前已经没有什么维护了，所以用着玩玩还行，如果是商业环境，就还是算了吧。

关于 assimp 的具体使用请看 [3D 模型加载](/2023-07-10/3dmodelloader)

## 环境
```
macOS 13.3.1
ndk 25.2.9519653
cmake version 3.26.4
```

## 安装 cmake、ndk


## 编译

**把具体的 shell 命令写出来，纯粹是为了初学者方便，其中都是我 pc 中的环境路径， 各位看客注意替换**

```shell
# 在 ~/.bash_profile 中添加 NDK_TOOLCHAIN_FILE，供 cmake 使用
export NDK_PATH=/Users/daweibayu/Library/Android/sdk/ndk/25.2.9519653
export NDK_TOOLCHAIN_FILE=$NDK_PATH/build/cmake/android.toolchain.cmake
```

```bash
# 下载代码
daweibayu@daweibayudeMac opengl % git clone git@github.com:assimp/assimp.git
```

```bash
# 从 master 分支切到 tags/v5.2.5
daweibayu@daweibayudeMac-mini assimp % git checkout tags/v5.2.5 -b v5.2.5
```

```bash
# 新建编译文件夹，避免与当前源码混和
daweibayu@daweibayudeMac opengl % cd assimp 
daweibayu@daweibayudeMac assimp % mkdir build
daweibayu@daweibayudeMac assimp % cd build 
```

```bash
# 由 cmake 生成 makefile，具体参数参看后文
daweibayu@daweibayudeMac build % cmake -DCMAKE_TOOLCHAIN_FILE=$NDK_TOOLCHAIN_FILE -DCMAKE_INSTALL_PREFIX=/assimp_525 -DANDROID_ABI=arm64-v8a -DANDROID_NATIVE_API_LEVEL=24 -DANDROID_FORCE_ARM_BUILD=TRUE -DANDROID_STL=c++_shared -DASSIMP_BUILD_ALL_IMPORTERS_BY_DEFAULT=false -DASSIMP_BUILD_OBJ_IMPORTER=TRUE -DASSIMP_NO_EXPORT=true -DASSIMP_BUILD_TESTS=false -DCMAKE_BUILD_TYPE=Release ..
```

```bash
# 生成 makefile 后直接 make
daweibayu@daweibayudeMac build % make -j8
```

```bash
# 将编译后的文件安装到 tmp 文件夹下
daweibayu@daweibayudeMac build % make install DESTDIR=/tmp/assimp/ 
```

## 导入 Android 工程

请参看 [3D 模型加载](/2023-07-10/3dmodelloader)


## 其他
### 参数
cmake 一些参数的注释：
```c
CMake build options
The cmake-build-environment provides options to configure the build. The following options can be used:

ASSIMP_HUNTER_ENABLED (default OFF): Enable Hunter package manager support.
BUILD_SHARED_LIBS (default ON): Generation of shared libs (dll for windows, so for Linux). Set this to OFF to get a static lib.
ASSIMP_BUILD_FRAMEWORK (default OFF, MacOnly): Build package as Mac OS X Framework bundle.
ASSIMP_DOUBLE_PRECISION (default OFF): All data will be stored as double values.
ASSIMP_OPT_BUILD_PACKAGES (default OFF): Set to ON to generate CPack configuration files and packaging targets.
ASSIMP_ANDROID_JNIIOSYSTEM (default OFF): Android JNI IOSystem support is active.
ASSIMP_NO_EXPORT (default OFF): Disable Assimp's export functionality.
ASSIMP_BUILD_ZLIB (default OFF): Build our own zlib.
ASSIMP_BUILD_ALL_EXPORTERS_BY_DEFAULT (default ON): Build Assimp with all exporter senabled.
ASSIMP_BUILD_ALL_IMPORTERS_BY_DEFAULT (default ON): Build Assimp with all importer senabled.
ASSIMP_BUILD_ASSIMP_TOOLS (default ON): If the supplementary tools for Assimp are built in addition to the library.
ASSIMP_BUILD_SAMPLES (default OFF): If the official samples are built as well (needs Glut).
ASSIMP_BUILD_TESTS (default ON): If the test suite for Assimp is built in addition to the library.
ASSIMP_COVERALLS (default OFF): Enable this to measure test coverage.
ASSIMP_INSTALL (default ON): Install Assimp library. Disable this if you want to use Assimp as a submodule.
ASSIMP_WARNINGS_AS_ERRORS (default ON): Treat all warnings as errors.
ASSIMP_ASAN (default OFF): Enable AddressSanitizer.
ASSIMP_UBSAN (default OFF): Enable Undefined Behavior sanitizer.
ASSIMP_BUILD_DOCS (default OFF): Build documentation using Doxygen. OBSOLETE, see https://github.com/assimp/assimp-docs
ASSIMP_INJECT_DEBUG_POSTFIX (default ON): Inject debug postfix in .a/.so/.lib/.dll lib names
ASSIMP_IGNORE_GIT_HASH (default OFF): Don't call git to get the hash.
ASSIMP_INSTALL_PDB (default ON): Install MSVC debug files.
USE_STATIC_CRT (default OFF): Link against the static MSVC runtime libraries.
ASSIMP_BUILD_DRACO (default OFF): Build Draco libraries. Primarily for glTF.
ASSIMP_BUILD_ASSIMP_VIEW (default ON, if DirectX found, OFF otherwise): Build Assimp view tool (requires DirectX).
```

参数请按照自己的需求定义
比如因为示例中只使用了 obj 文件，所以设置为 `-DASSIMP_BUILD_ALL_IMPORTERS_BY_DEFAULT=false` 和 `-DASSIMP_BUILD_OBJ_IMPORTER=TRUE`  
因为用于 Android 手机，所以设置 `-DANDROID_ABI=arm64-v8a`
需要注意的是，对于 NDK r18 以上版本，gnustl_static 已经不在支持，在 [ndk r18 Changelog](https://github.com/android/ndk/wiki/Changelog-r18) 中有`gnustl, gabi++, and stlport have been removed`，所有要设定为 `-DANDROID_STL=c++_shared`(当然并不一定是动态库)


### 选择分支

master 分支可以正常编译，但是导入后缺失引用，具体 [issue utf8.h: No such file or directory](https://github.com/assimp/assimp/issues/5005)。本文选用了 v5.2.5 分支，注意，因为要编译的是 arm64-v8a，如果也要编译 armeabi-v7a，如果编译失败，请试一下其他分支。