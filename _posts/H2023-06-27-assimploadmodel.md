---
layout: post
title:  "assimp 加载 3d 模型"
author: "daweibayu"
tags: OpenGL
excerpt_separator: <!--more-->
---
 <!--more-->

 [assimp](https://github.com/assimp/assimp)，本文以 [tag v5.2.5](https://github.com/assimp/assimp/tree/v5.2.5) 代码为例

 其实也有人做了 java 的包装，[](https://github.com/kotlin-graphics/assimp)，我试了下，现在还是可用的，可以成功导入，但是看样子目前已经没有什么维护了，所以用着玩玩还行，如果是商业环境，就还是算了吧。


 安装 cmake、ndk

## 简介


## 编译

具体流程可以参考 [](https://assimp-docs.readthedocs.io/en/latest/about/quickstart.html#the-android-build)，上文是基于 [android-cmake](https://github.com/taka-no-me/android-cmake)


```
Could NOT find PkgConfig (missing: PKG_CONFIG_EXECUTABLE) 
```
```
No package 'minizip' found
```


```
cmake -DCMAKE_TOOLCHAIN_FILE=/Users/daweibayu/Library/Android/sdk/ndk/25.2.9519653/build/cmake/android.toolchain.cmake -DCMAKE_INSTALL_PREFIX=/assimp-3.0 -DANDROID_ABI=armeabi-v7a -DANDROID_NATIVE_API_LEVEL=android-10 -DANDROID_FORCE_ARM_BUILD=TRUE -DANDROID_STL=gnustl_static -DASSIMP_BUILD_OBJ_IMPORTER=TRUE -DASSIMP_BUILD_FBX_IMPORTER=TRUE ..
```


```
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

https://github.com/lammps/lammps/issues/3682



## 遇到的问题
```
/Users/daweibayu/workspace/opengl/assimp/code/PostProcessing/ValidateDataStructure.cpp:83:22: error: 'vsprintf' is deprecated: This function is provided for compatibility reasons only.  Due to security concerns inherent in the design of sprintf(3), it is highly recommended that you use vsnprintf(3) instead. [-Werror,-Wdeprecated-declarations]
    const int iLen = vsprintf(szBuffer, msg, args);
                     ^
/Library/Developer/CommandLineTools/SDKs/MacOSX13.3.sdk/usr/include/stdio.h:207:1: note: 'vsprintf' has been explicitly marked deprecated here
__deprecated_msg("This function is provided for compatibility reasons only.  Due to security concerns inherent in the design of sprintf(3), it is highly recommended that you use vsnprintf(3) instead.")
^
/Library/Developer/CommandLineTools/SDKs/MacOSX13.3.sdk/usr/include/sys/cdefs.h:215:48: note: expanded from macro '__deprecated_msg'
        #define __deprecated_msg(_msg) __attribute__((__deprecated__(_msg)))
                                                      ^
/Users/daweibayu/workspace/opengl/assimp/code/PostProcessing/ValidateDataStructure.cpp:98:22: error: 'vsprintf' is deprecated: This function is provided for compatibility reasons only.  Due to security concerns inherent in the design of sprintf(3), it is highly recommended that you use vsnprintf(3) instead. [-Werror,-Wdeprecated-declarations]
    const int iLen = vsprintf(szBuffer, msg, args);
                     ^
/Library/Developer/CommandLineTools/SDKs/MacOSX13.3.sdk/usr/include/stdio.h:207:1: note: 'vsprintf' has been explicitly marked deprecated here
__deprecated_msg("This function is provided for compatibility reasons only.  Due to security concerns inherent in the design of sprintf(3), it is highly recommended that you use vsnprintf(3) instead.")
^
/Library/Developer/CommandLineTools/SDKs/MacOSX13.3.sdk/usr/include/sys/cdefs.h:215:48: note: expanded from macro '__deprecated_msg'
        #define __deprecated_msg(_msg) __attribute__((__deprecated__(_msg)))

```