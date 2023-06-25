---
layout: post
title:  "Android 手机 Root、debug 模式（修改源码方式）"
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

前边三篇文章写了下载源码、编译和目录解析，当然费了这么大劲搞这玩意不是编译完了就完事了，我们得切切实实的用起来哇，不然岂不白浪费时间了。

相信 Android 开发者们总有需要 Root 手机的需求，而且大多数 Android 开发者对 Linux 层的东西也不是很熟悉，所以一般也都是网上去找 Root 教程。什么 3x0 一键 Root、疼训 Root 等，这里奉劝各位一句，千万别用，手机被 Root 了就相当于家里的大门没锁，这个时候可不是什么人都能往家领的，请神容易送神难，要是在 root 执行完后偷偷执行个脚本，那你真就没有任何隐私可言了（国内无良厂商的节操也不用我多说了吧...）。至于家喻户晓的 SuperSU，还被收购了。与其把身家性命放到别人手里，不如自己去直接 build 一个 Root 的 Android 系统嘛，让手机彻底掌握到自己手里嘛。

那接下来就是我们的正文了，build 一个 Root 的 Android 系统。

### Root 的基本原理

其实就一句话：将可执行文件 su 放入` /system/xbin`中。

正常的 Android 中 `/system/xbin` 文件夹下是没有 `su` 命令的，而且非 root 情况下 `/system/xbin` 又是没有权限访问且只读的，所以此时正常途径也不能把 `su` 放进去。这样才防止了正常使用下 root 的可能。第三方的 Root 工具就是通过漏洞、刷系统、recovery 等方式，不过这不是我们本文的重点，不赘述。

### 修改源文件

我们常见的判断依据是通常是手机中的两个配置文件：

* `/default.prop` （每次开机都会覆盖）
* `/system/build.prop` （开机重启后依然不变）

default.prop 内容一般为：

```shell
...
ro.secure=1
...
ro.debuggable=1
...
```

其中 `ro.secure` 用来表示是否 root，`ro.debuggable` 用来标示是否是 debug 模式。

`/system/build.prop` 内容一般为：
```shell
...
ro.build.type=userdebug
...
```

所以剩下的就简单了，我们找到源码中对应的位置修改掉就可以啦。
全局搜索得到如下结果（非 out 目录、且是设置此值或者设置默认值的地方）：

`build/core/main.mk`
```shell
...
ADDITIONAL_DEFAULT_PROPERTIES += ro.secure=1
ADDITIONAL_DEFAULT_PROPERTIES += security.perf_harden=1
...
...
# Set device insecure for non-user builds.
ADDITIONAL_DEFAULT_PROPERTIES += ro.secure=1
...
```

cts/tests/tests/os/src/android/os/cts/BuildTest.java
```shell
public void testIsSecureUserBuild() throws IOException {
 assertEquals("Must be a user build", "user", Build.TYPE);
 assertProperty("Must be a non-debuggable build", RO_DEBUGGABLE, "0");
 assertProperty("Must be a secure build", RO_SECURE, "1");
}
```

`system/core/adb/adb_main.cpp`
```shell
...
property_get("ro.secure", value, "1");
bool ro_secure = (strcmp(value, "1") == 0);

property_get("ro.debuggable", value, "");
bool ro_debuggable = (strcmp(value, "1") == 0);
...
```

那我们就该掉就好啦，具体修改如下：

`build/core/main.mk`
```shell
...
ADDITIONAL_DEFAULT_PROPERTIES += ro.secure=0
ADDITIONAL_DEFAULT_PROPERTIES += security.perf_harden=1
...
...
# Set device insecure for non-user builds.
ADDITIONAL_DEFAULT_PROPERTIES += ro.secure=0
...
```

cts/tests/tests/os/src/android/os/cts/BuildTest.java
```shell
public void testIsSecureUserBuild() throws IOException {
 assertEquals("Must be a user build", "user", Build.TYPE);
 # assertProperty("Must be a non-debuggable build", RO_DEBUGGABLE, "0");
 # assertProperty("Must be a secure build", RO_SECURE, "1");
}
```

`system/core/adb/adb_main.cpp`
```shell
...
property_get("ro.secure", value, "0");
bool ro_secure = (strcmp(value, "1") == 0);

property_get("ro.debuggable", value, "");
bool ro_debuggable = (strcmp(value, "1") == 0);
...
```

### 编译

以上修改是否可以成功呢？我也不是很确定，那我们就试试看咯

按照 [MacOS 10.13 编译 Android 源码](https://www.jianshu.com/p/122fff2d4e37)文中编译、刷机步骤再搞一下。

刷机成功后 `adb shell` 上去就已经是 root 了，那证明我们前边搞的都是没问题的，在验证下 `/default.prop` 与 `/system/build.prop`，也改过来了~~~~
