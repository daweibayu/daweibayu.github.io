---
layout: post
title:  "线程的本质（art 层实现）"
author: "daweibayu"
tags: Android
excerpt_separator: <!--more-->
---
<!--more-->

ojluni：OpenJDK, java.lang / java.util / java.net / java.io 的缩写，就是OpenJDK核心库的意思。
Bionic库是Android的基础库之一，也是连接Android和Linux的桥梁。Bionic库中包含了很多基本系统功能接口，这些功能大部分来自 Linux，但是和标准的 Linux 之间有很多细微差别。同时Bionic库中增加了一些新的模块（像log、linker），服务于Android的上层代码。

[java_lang_Thread.cc](https://cs.android.com/android/platform/superproject/+/master:art/runtime/native/java_lang_Thread.cc)

[thread.cc](https://cs.android.com/android/platform/superproject/+/master:art/runtime/thread.cc)


[pthread.cpp](https://cs.android.com/android/platform/superproject/+/refs/heads/master:bionic/libc/bionic/pthread_create.cpp)




pthread_create(&new_pthread, &attr, gUseUserfaultfd ? Thread::CreateCallbackWithUffdGc: Thread::CreateCallback,
 child_thread);


