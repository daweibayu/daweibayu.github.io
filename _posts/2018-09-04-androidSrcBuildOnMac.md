---
layout: post
title:  "MacOS 10.13 编译 Android 源码"
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

原来下载下来编译其实还挺简单的，直接照官方指导执行就行了，结果现在再编译就各种问题，搞了大半天才算彻底搞定，主要是因为系统、软件等版本问题，记录下，避免以后别人再采坑了。

以下顺序不要搞错，不然会来回返工反而费时费力：
* 检查 jdk 版本
* 检查 Xcode 版本
* 检查 Xcode 命令行工具是否安装
* 安装 MacPorts
* 安装 Make、git、GPG 等
* 编译

* * * * * *

### 环境准备

#### 切换 Java 版本到 1.7

关于 java sdk 的版本要求可以看 [jdk 版本要求](https://source.android.com/setup/build/requirements)

我要编译的是 Android 6.0，需要的 java 版本是 1.7，但是我平时使用的是 1.8，所以可以在 `~/.bash_profile` 添加如下代码：

```shell
export JAVA_7_HOME="$(/usr/libexec/java_home -v 1.7)"
export JAVA_8_HOME="$(/usr/libexec/java_home -v 1.8)"

#默认JDK 8
export JAVA_HOME=$JAVA_8_HOME

alias jdk7="export JAVA_HOME=$JAVA_7_HOME"
alias jdk8="export JAVA_HOME=$JAVA_8_HOME"
```

当编译 Android 版本时直接命令行执行如下命令就可以直接切换到 Java 1.7 了：

```shell
jdk7
```

#### Xcode 降级

对于 Xcode 9.0 版本的童鞋需要降级到 8.+ 版本才可使用，不然报错如下：

```shell
In file included from external/libcxxabi/src/cxa_exception.cpp:18:
external/libcxx/include/cstdlib:159:44: error: declaration conflicts with target of using declaration already in scope
inline _LIBCPP_INLINE_VISIBILITY long abs(  long __x) _NOEXCEPT {return labs(__x);}
 ^
/Applications/Xcode.app/Contents/Developer/Toolchains/XcodeDefault.xctoolchain/usr/include/c++/v1/stdlib.h:111:44: note: target of using declaration
inline _LIBCPP_INLINE_VISIBILITY long abs(  long __x) _NOEXCEPT {return labs(__x);}
 ^
external/libcxx/include/cstdlib:134:9: note: using declaration
using ::abs;
 ^
external/libcxx/include/cstdlib:161:44: error: declaration conflicts with target of using declaration already in scope
inline _LIBCPP_INLINE_VISIBILITY long long abs(long long __x) _NOEXCEPT {return llabs(__x);}
```
![xcode_downgrade_tip.png](https://upload-images.jianshu.io/upload_images/2829180-f96fddd73b004ebe.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

具体上下文可参见[XCode 9 - new LLVM headers causing errors](https://forums.developer.apple.com/thread/87814)，流程就是如下两步：

*  完整卸载 Xcode 9，依次删除如下文件（或者使用其他工具）
1.  /Applications/Xcode.app
2.  /Library/Preferences/com.apple.dt.Xcode.plist
3.  ~/Library/Preferences/com.apple.dt.Xcode.plist
4.  ~/Library/Caches/com.apple.dt.Xcode
5.  ~/Library/Application Support/Xcode
6.  ~/Library/Developer

* [下载 Xcode 8.3.3](https://developer.apple.com/download/more/)，解压缩并拖入  `/Application`  中后点击安装

#### 安装 Xcode 命令行工具，执行命令如下：

```shell
xcode-select --install
```

#### 安装 MacPorts

[MacPorts 官网](https://www.macports.org/install.php) 下载安装

请确保 `/opt/local/bin` 在路径中显示在`/usr/bin` 前面。否则，请将以下内容添加到 `~/.bash_profile` 文件中：

```shell
export PATH=/opt/local/bin:$PATH
```

#### 安装 Make、git、GPG 等

```shell
POSIXLY_CORRECT=1 sudo port install gmake libsdl git gnupg2
```

* * * * * *

### 编译

然后就是编译啦，在源文件根目录（此处我的示例就是  `/Volumes/android/android-6.0.1_r77`）依次执行如下命令：

#### 清空以前编译操作的遗留的输出

```shell
make clobber
```

#### 初始化环境

```shell
source build/envsetup.sh
```

#### 选择要编译的目标：

```shell
lunch
```

```shell
Lunch menu... pick a combo:
 1\. aosp_arm-eng
 2\. aosp_arm64-eng
 3\. aosp_mips-eng
 4\. aosp_mips64-eng
 5\. aosp_x86-eng
 6\. aosp_x86_64-eng
 7\. aosp_deb-userdebug
 8\. aosp_flo-userdebug
 9\. full_fugu-userdebug
 10\. aosp_fugu-userdebug
 11\. mini_emulator_arm64-userdebug
 12\. m_e_arm-userdebug
 13\. mini_emulator_mips-userdebug
 14\. mini_emulator_x86-userdebug
 15\. mini_emulator_x86_64-userdebug
 16\. aosp_flounder-userdebug
 17\. aosp_angler-userdebug
 18\. aosp_bullhead-userdebug
 19\. aosp_hammerhead-userdebug
 20\. aosp_hammerhead_fp-userdebug
 21\. aosp_shamu-userdebug
 Which would you like? [aosp_arm-eng]
```

然后选择你想执行的，我选的是  `aosp_hammerhead-userdebug`，然后 Enter 键执行选择

#### 编译

```shell
make -j8
```
![make_completed.png](https://upload-images.jianshu.io/upload_images/2829180-16af0d503617329f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

硬件配置：2015顶配 MBP，Intel i7 2.5GHz、4核、16G内存、500 ssd，用时 1小时 11分钟


#### 刷机

先转到 FastBoot 模式，执行命令如下：
```shell
adb reboot bootloader
```
跳转到输出 img 的目录：
```shell
cd 源码位置/out/target/product/hammerhead/
```
然后就是刷机啦（这样刷机数据可都是会丢失的哈，注意用测试机）：
```shell
fastboot -w flashall
```


* * * * * *

### MacOS 10.13 常见的一些错误：

#### gnupg2

```shell
Error: gnupg has been deprecated. If you absolutely want to stay on the classic version, install the gnupg1 port. All other users are recommended to install gnupg2.
```

如文中所示，gnupg 已经被废弃了，但是有可能任然需要用，所以安装的时候要安装 gnupg2

```shell
POSIXLY_CORRECT=1 sudo port install gnupg2
```

#### MacOS sdk

```shell
build/core/combo/mac_version.mk:38: *****************************************************
build/core/combo/mac_version.mk:39: * Can not find SDK 10.6 at /Developer/SDKs/MacOSX10.6.sdk
build/core/combo/mac_version.mk:40: *****************************************************
build/core/combo/mac_version.mk:41: *** Stop.. Stop.
```

![macos_sdk_tip.png](https://upload-images.jianshu.io/upload_images/2829180-390a56500bce0ca7.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

顾名思义，就是缺少了 MacOS sdk，直接[非官方](https://github.com/phracker/MacOSX-SDKs/releases)下载就可以了，下载到的文件是  `MacOSX10.11.sdk.tar.xz`  文件

* 先解压缩：

```shell
tar -xf MacOSX10.11.sdk.tar.xz
```

* 然后将解压得到的  `MacOSX10.11.sdk`文件夹复制到`/Applications/Xcode.app/Contents/Developer/Platforms/MacOSX.platform/Developer/SDKs`文件夹下

#### Xcode 命令行工具没有安装或者版本问题

```shell
Yacc: libmcldScript <= frameworks/compile/mclinker/lib/Script/ScriptParser.yy
prebuilts/misc/darwin-x86/bison/bison -d -o out/host/darwin-x86/obj/STATIC_LIBRARIES/libmcldScript_intermediates/ScriptParser.cpp frameworks/compile/mclinker/lib/Script/ScriptParser.yy
frameworks/compile/mclinker/lib/Script/ScriptParser.yy:28.1-5: invalid directive: `%code'
frameworks/compile/mclinker/lib/Script/ScriptParser.yy:28.7-14: syntax error, unexpected identifier
make: *** [out/host/darwin-x86/obj/STATIC_LIBRARIES/libmcldScript_intermediates/ScriptParser.cpp] Error 1
make: *** Waiting for unfinished jobs....
```

![xcode_commandline_tip.png](https://upload-images.jianshu.io/upload_images/2829180-92333565af92fa77.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


 如果没有安装执行如下命令安装：

 ```shell
 xcode-select --install
 ```

 如果已经安装但仍有问题，可以删除后重新安装：

 ```shell
 sudo rm -rf /Library/Developer/CommandLineTools
 ```
 ```shell
 xcode-select --install
 ```

#### bison

```shell
make: *** [out/host/darwin-x86/obj/EXECUTABLES/aidl_intermediates/aidl_language_y.cpp] Abort trap: 6
make: *** Waiting for unfinished jobs....
```

![bison_error_tip.png](https://upload-images.jianshu.io/upload_images/2829180-4ef21d8d44beb3fd.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

具体原因可见 [build aosp on Mac OS 10.13 failed](http://android.2317887.n4.nabble.com/build-aosp-on-Mac-OS-10-13-failed-td435701.html)，原因就是直接下载下来的源码中的 bison 版本中包含一个 bug，官方也已[修复](https://android-review.googlesource.com/c/platform/external/bison/+/517740)，所以直接下载官方的最新版本替换即可

* [官网](https://ftp.gnu.org/gnu/bison/) 下载最新版本 [bison-3.1.tar.xz](https://ftp.gnu.org/gnu/bison/bison-3.1.tar.xz)
*  解压缩
*  在解压缩的到文件的根目录下直接执行

```shell
./configure
make
make install
```

安装成功后复制最新的可执行文件  `/bison 所在的位置/src/bison`到  `/Android 源码所在位置/android-6.0.1_r77/prebuilts/misc/darwin-x86/bison/`
