---
layout: post
title:  "MacOS 下载 Android 源码"
author: "daweibayu"
tags: Android
excerpt_separator: <!--more-->
---

<!--more-->

* [MacOS 下载 Android 源码](/2018-09-03/androidSrcDownloadInMac)
* [MacOS 10.13 编译 Android 源码](/2018-09-04/androidSrcBuildOnMac)
* [Android 源码目录结构解析](/2018-09-05/androidSrcPath)
* [Android 手机 Root、debug 模式（修改源码方式）](/2018-09-06/androidRootDebug)

## 相关资源

[AOSP](https://source.android.com/) （Android Open Source Project） [代码库](https://android.googlesource.com/)

如果感觉英语费劲的话可以选择一下语言

## 具体流程

### 关于大小写不敏感的问题

MacOS 是大小写不敏感的，比如输入如下命令：

```shell
xxxxs-MacBook-Pro:Desktop xxx$ mkdir abc
xxxxs-MacBook-Pro:Desktop xxx$ mkdir Abc
mkdir: Abc: File exists
```

但 Linux 确是大小写敏感的，避免出现问题，所以要在 Mac 上创建区分大小写的磁盘映像，执行命令如下（执行完后会生成 `android.dmg.sparseimage` 文件）：

```shell
hdiutil create -type SPARSE -fs 'Case-sensitive Journaled HFS+' -size 80g ~/android.dmg
```

具体映像大小以及映像位置可以自由选择，这里我设置的大小是80G，位置是  `~/android.dmg`，一下执行命令均默认此位置

### 挂载此磁盘映像

可以在 `~/.bash_profile` 中添加挂载、卸载映像函数（添加函数并放到 `bash_profile` 里主要为了方便日后使用）

```shell
# mount the android file image
mountAndroid() { hdiutil attach ~/android.dmg -mountpoint /Volumes/android; }
# unmount the android file image
umountAndroid() { hdiutil detach /Volumes/android; }
```

添加完成后执行如下：

```shell
source ~/.bash_profile
mountAndroid
```

### 下载 `repo` 工具（一个 python 文件）

先创建一个目录（官方指导推荐放到 `~/bin`，放到其他目录也一样）

```shell
mkdir ~/bin
```

修改  `~/.bash_profile`  ，添加如下代码：

```shell
export PATH=$PATH:~/bin
```

然后下载 `repo` 工具并设置为可执行文件

```shell
source ~/.bash_profile
curl https://storage.googleapis.com/git-repo-downloads/repo > ~/bin/repo
chmod a+x ~/bin/repo
```

### 下载源代码

切记源码要位于已经挂载了的大小写敏感的映像下，所以要先切换到  `/Volumes/android/`  下

```shell
cd /Volumes/android/
```

在 [分支目录](https://source.android.com/setup/start/build-numbers#source-code-tags-and-builds)中选择要下载的分支，因为手头只有一台 Nexus 5，所以选择了  `android-6.0.1_r77`

新建文件夹并切换到相应文件夹

```shell
mkdir android-6.0.1_r77
cd android-6.0.1_r77
```

如果要完成下载太浪费时间了，应该是30多G的样子（2017 年下载的完整版本就已经近 30G 了），所以这里默认只选择 master 分支下的最新的完整版（--depth=1 为限制不拉取其余历史）

```shell
repo init --depth=1 -u https://android.googlesource.com/platform/manifest -b android-6.0.1_r77
repo sync -c
```

下载大小大概在 6G 的样子，然后就是等着下载完成就可以进行编译啦
