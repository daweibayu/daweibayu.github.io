---
layout: post
title:  "3D 模型加载（Assimp）"
author: "daweibayu"
tags: opengl
excerpt_separator: <!--more-->
---

<!--more-->

## 前言

代码完全可以参考 [AssimpAndroid](https://github.com/anandmuralidhar24/AssimpAndroid)，对应 Anand’s blog [Use Assimp to load a 3D model](http://www.anandmuralidhar.com/blog/android/assimp/)，这是原创的作者，但因为工程已经是七年前了，年久失修，如果要 run 起来，需要修改的地方太多。  

我参考了 [TestAssimp](https://github.com/a422070876/TestAssimp)，该工程也已经有三年了，Android studio 直接导入的话也会有些问题，需要指定老版本 ndk ```ndkVersion '20.0.5594570'``` 才行，不过至少可以 run 并看到显示效果。

我在 TestAssimp 的基础上做了代码重构，代码可见 [AndroidAssimp by daweibayu](https://github.com/daweibayu/AndroidAssimp)，一是感觉原有程序代码框架略有问题，二是为了学习 opengl 并把 C++ 捡起来。

## 环境
```
Android Studio Flamingo | 2022.2.1 Patch 1
gradle 8.0
OpenJDK 17
```

## 建立 native C++ 工程
不多说，通过 Android studio 直接按照指示创建就可以，注意要提前将 ndk 等依赖先下载好

## 3d 模型下载

可以在 [free3d](https://free3d.com/) 上搜索自己喜欢的`免费`模型，注意要下载 `obj` 格式，当然其他格式也可以用 Assimp 导入，但是目前该工程只支持了 obj。当然也可以直接使用我工程中用的资源或者 AssimpAndroid 中的资源。

在使用前推荐先了解下 [obj 文件解析](/2023-07-07/fileobj)、[mtl 文件解析](/2023-07-07/filemtl)。一是方便了解代码，二是下载的资源中如果有`错误内容`，也方便修改。

将对应的 obj、mtl、jpg 文件 copy 到 assets 文件夹下

## 导入三方库

### 导入 assimp

具体编译可以参看 [assimp 编译](/2023-06-27/assimpcompile)
1. 将 `include` 文件夹下的 `assimp` 完整 copy 到工程中 `cpp` 文件夹下的 `include` 中（当然你要放到别的文件夹下也是可以的）
2. 将 `lib` 下的 `libassimp.so` 动态库 copy 到工程中 `jniLibs` 下的 `arm64-v8a` 中（因为上文中编译的是 arm64-v8a 的库）
3. 修改 CMakeLists.txt
```
include_directories(${CMAKE_CURRENT_SOURCE_DIR}/include)
add_library(assimp SHARED IMPORTED)
set_target_properties(assimp PROPERTIES IMPORTED_LOCATION ${libs}/arm64-v8a/libassimp.so)
target_link_libraries(androidassimp assimp ...)
```

### 导入 glm

[glm](https://github.com/g-truc/glm)，最新版本 [0.9.9.8](https://github.com/g-truc/glm/tree/0.9.9.8)
因为 glm 是 **header only**，所以也就没有必要编译打包了，直接源码接入就完事了

1. 将下载下来的代码中的 `glm-0.9.9.8/glm` 完整 copy 到工程中 `cpp` 文件夹下的 `include` 中
2. 我这里将 glm 文件夹名字修改为 `glm-0.9.9.8`，为了方便版本标识
3. 修改 CMakeLists.txt
```
aux_source_directory (${CMAKE_SOURCE_DIR}/include/glm-0.9.9.8/ GLM)
aux_source_directory (${GLM}/gtc GTC)
aux_source_directory (${GLM}/gtx GTX)
aux_source_directory (${GLM}/detail DETAIL)

target_include_directories(androidassimp PRIVATE ${GLM} ${GTC} ${DETAIL} ${GTX})
```

### 导入 opencv

opencv 库在这里只用来读取和处理纹理图片

1. 在 [opencv](https://opencv.org/releases/) 官网下载，当前最新版本是 4.8.0，但保险起见，还是选择 4.7.0 吧
2. 将 `OpenCV-android-sdk/sdk/native/jni/include/` 下的 `opencv2` 文件夹完整 copy 到工程中 `cpp` 文件夹下的 `include` 中
3. 将 `OpenCV-android-sdk/sdk/native/libs/arm64-v8a/` 下的 `libopencv_java4.so` copy 到工程中 `jniLibs` 下的 `arm64-v8a` 中
4. 修改 CMakeLists.txt
```
add_library(opencv SHARED IMPORTED)
set_target_properties(opencv PROPERTIES IMPORTED_LOCATION ${libs}/arm64-v8a/libopencv_java4.so)
target_link_libraries(androidassimp opencv ...)
```


未完待续