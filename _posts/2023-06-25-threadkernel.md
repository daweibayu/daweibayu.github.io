---
layout: post
title:  "线程的本质（内核层实现）"
author: "daweibayu"
tags: Android
excerpt_separator: <!--more-->
---
 <!--more-->
 
[kthread.c](https://android.googlesource.com/kernel/common/+/refs/heads/android-gs-bluejay-5.10-android13/kernel/kthread.c)

[arm64 process.c](https://android.googlesource.com/kernel/common/+/refs/heads/android-gs-bluejay-5.10-android13/arch/arm64/kernel/process.c)

[sched.h](https://android.googlesource.com/kernel/common/+/refs/heads/android-gs-bluejay-5.10-android13/include/linux/sched.h)