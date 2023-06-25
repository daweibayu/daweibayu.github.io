---
layout: post
title:  "Android 源码目录结构解析"
author: "daweibayu"
tags: Android
excerpt_separator: <!--more-->
---

<!--more-->

* [MacOS 下载 Android 源码](/2018-09-03/androidSrcDownloadInMac)
* [MacOS 10.13 编译 Android 源码](/2018-09-04/androidSrcBuildOnMac)
* [Android 源码目录结构解析](/2018-09-05/androidSrcPath)
* [Android 手机 Root、debug 模式（修改源码方式）](/2018-09-06/androidRootDebug)

### 前言

前边我们已经介绍如何下载源码了，下载下来后我们也不能大眼瞪小眼，你不认它，它也不认识你，那岂不就白浪费时间下载了。下载的目的一方面是为了方便我们阅读源码，另一方面我们还可以修改源码编译属于自己定制的系统嘛。这个时候了解源码的目录结构就是第一步的工作了。废话不多说，我们先看一下源码的目录结构。

### 目录结构

> [abi](#1)
> [art](#2)
> [bionic](#3)
> [bootable](#4)
> [build](#5)
> [cts](#6)
> [dalvik](#7)
> [developers](#8)
> [development](#9)
> [device](#10)
> [docs](#11)
> [external](#12)
> [frameworks](#13)
> [hardware](#14)
> [libcore](#15)
> [libnativehelper](#16)
> [ndk](#17)
> [out](#18)
> [packages](#19)
> [pdk](#20)
> [platform_testing](#21)
> [prebuilts](#22)
> [sdk](#23)
> [system](#24)
> [tools](#25)
> [vendor](#26)


### abi（Application Binary Interface）

具体请看[官方文档](https://developer.android.com/ndk/guides/abis)


### art（Android runtime）

具体请看[ART and Dalvik](https://source.android.com/devices/tech/dalvik/)


### bionic

一些基础库（以下只是列举了几个，并非全部）
> libm（library math）
> libc（library c）：在 glibc 的基础上做了裁剪与修改的，为了规避GNU GPL等商业行为的约束
> [libstdc++](https://developer.android.com/ndk/guides/cpp-support)（library standard C++）：并非完整版，只做了简单支持
> linker：装载链接相关库


### bootable

> recovery

bootable 下仅包含 recovery 此文件夹，其实就是启动 Android recovery 模式相关的代码


### build

Android Build 系统，用来定制各种编译规则。主要由 makefile 组成。
比如在编译时要执行的  ```source build/envsetup.sh```  就位于 build 下。
推荐一篇以前看过的介绍 Build 比较好的文章 [理解 Android Build 系统](https://www.ibm.com/developerworks/cn/opensource/os-cn-android-build/index.html)


### cts（Compatibility Test Suite）

一个自动化测试工具 [CTS](https://source.android.com/compatibility/cts/)。确保 make 出来的系统没问题，注意如果要是修改了源码的话相关的 testcase 也是要修改的。


### dalvik

dalvik 虚拟机，与 art 有千丝万缕的关系，具体也可以看 [ART and Dalvik](https://source.android.com/devices/tech/dalvik/)


### developers

主要是一些可运行的 Android 示例项目，可以单独拉出来运行。


### development

仍然是一些工具性的东西，全部子文件夹如下：
> apps
> build
> cmds
> docs
> host
> ide
> libraries
> ndk
> perftests
> samples
> scripts
> sdk
> sdk_overlay
> sys-img
> testrunner
> tools
> tutorials

* apps 中包含了一些并没有在系统中部署的应用
* ndk 中就是和 ndk 相关的东西
* samples 中则是一些示例 app
* 像 MonkeyTest 相关代码位于 development/cmds/monkey 中
* eclipse、emacs、intellij、xcode的配置信息位于 development/ide 中
emulator, simulator, and stuff for the NDK and SDK


### device

包含不同品牌手机独有的设备信息，具体目录如下：
> asus
> common
> generic
> google
> htc
> huawei
> lge
> moto
> sample


### docs

> source.android.com 

仅包含此文件夹，该文件夹下相关文件就是生成 source.android.com 站点的具体素材及代码


### external

一些开源的第三方组件，这里仅列了一下大家比较熟悉的如glide、junit、okhttp、sqlite 等
> aac
> apache-http
> bison
> chromium-webview
> easymock
> glide
> google-breakpad
> google-fonts
> jpeg
> junit
> lldb
> llvm
> ltrace
> markdown
> okhttp
> opencv
> proguard
> protobuf
> robolectric
> scrypt
> selinux
> smali
> sqlite
> strace
> tcpdump
> valgrind
> webrtc
> zlib


### frameworks

这就是 Android 中大家熟悉的 Frameworks，应用程序框架层啦，全部子文件夹如下：
> av
> base
> compile
> data-binding
> ex
> mff
> minikin
> ml
> multidex
> native
> opt
> rs
> support
> volley
> webview
> wilhelm

* Android support 包 com.android.support:support-v4、v7 等都位于 frameworks/support 文件夹下
* webview 就位于 frameworks/webview 文件夹下
* 各种 Service，比如ActivityManagerService、SystemService、WindowManagerService、InputManagerService等就位于 frameworks/base 文件夹下
* keystore、opengl 等也位于 frameworks/base 文件夹下


### hardware

主要包含了 android [HAL](https://source.android.com/devices/architecture/hal)（硬件抽象层）相关代码。硬件抽象层介于 Linux内核驱动程序与 Android 系统之间。对 Linux 驱动进行了封装，使操作系统级别可以忽略底层实现的细节。


### libcore

一些核心库


### libnativehelper

JNI 相关的一些类


### ndk

原生开发工具包


### out

编译完后输出的所有相关文件都位于此文件夹下，包括生成的各种 img 就位于 out/target/product/hammerhead 下


### packages

各种内置的 apk、ContentProvider、输入法、壁纸等，所有文件夹如下：
> apps
> experimental
> inputmethods
> providers
> screensavers
> services
> wallpapers

*  蓝牙、浏览器、相机、邮件、音乐、NFC 等都位于 packages/apps 下面
* MediaProvider、DownloadProvider、MmsProvider等都位于 packages/providers 下
* 壁纸相关位于 packages/wallpapers 下


### pdk（Platform Development Kit）

平台开发套件，仅包含了一些供硬件抽象层开发使用的必要组件，供一些 OEM 厂商用来适配及测试最新的Android 系统，加快第三方厂商的更新速度。
加快OEM厂商的update速度


### platform_testing

平台相关的一些测试用例


### prebuilts

一些预构建成二进制的库 [prebuilts](https://developer.android.com/ndk/guides/prebuilts)
其中关于 build 时 bison 问题的主角就位于 prebuilts/misc/darwin-x86 下的 bison。


### sdk

看了下里边挺多被废弃的代码，所以我也吃不准这个文件夹的意义何在，所以暂时先不写了


### system

Android 的部分系统源码及一些工具，主要是在各种 java 启动程序起来前的部分。工具比如 adb、fastboot、keystore 等，其他如 mkbootimg、init 进程等。


### tools

工具，近包含 fat32lib 与 gradle，具体文件目录如下
> external
>> fat32lib
>> gradle


### vendor

包含不同供应商的私有的二进制库，仅包含如下三个文件夹：
> broadcom
> lge
> qcom


## 参考资料
[elinux Master-android](https://elinux.org/Master-android)

