---
layout: post
title:  "FileProvider 使用"
author: "daweibayu"
tags: Android
excerpt_separator: <!--more-->
---

<!--more-->

前一段时间有用户报点击图片会 crash，以前运行没什么问题，给用户要了堆栈后才发现是不兼容问题。
具体就是 7.0 有一些不兼容更新，具体详见 [7.0 Behavior Changes](https://developer.android.com/about/versions/nougat/android-7.0-changes.html#sharing-files) 。关于 FileProvider 可以参见 [FileProvider](https://developer.android.com/reference/android/support/v4/content/FileProvider.html)。

这里只介绍使用方式，具体原理性的东西稍后有时间再单独介绍。
三个步骤:
1. 在 AndroidManifest.xml 添加 provider 声明：

{% highlight c++ %}
<application .>
    <provider
        android:name="android.support.v4.content.FileProvider"
        android:authorities="<authorities>"
        android:exported="false"
        android:grantUriPermissions="true">
        <meta-data
            android:name="android.support.FILE_PROVIDER_PATHS"
            android:resource="@xml/<file-path>" />
    </provider>
</application>
{% endhighlight %}

注意其中的两个地方：一个是 <authorities>，一个是<file-path>，这两个地方的值可以自定义。

2. 在 res 文件夹下，新建文件夹 xml（与 drawable、layout 等并列），在 xml 文件夹中新建文件 <file-path>.xml，注意这里的值要与 1 中的 <file-path> 一致。
并修改其中内容为：
{% highlight c++ %}
<?xml version="1.0" encoding="utf-8"?>
<paths xmlns:android="http://schemas.android.com/apk/res/android">
    <cache-path name="cache_path" path=""/>
    <external-cache-path name="external-cache-path" path=""/>
    <files-path name="files_path" path=""/>
    <external-files-path name="external-files-path" path=""/>
</paths>
{% endhighlight %}

3. 在需要使用的地方添加：
```java
Intent intent = new Intent(Intent.ACTION_VIEW);
File file = new File(<local-file-path>);
if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.N) {
  intent.setFlags(Intent.FLAG_GRANT_READ_URI_PERMISSION);
  Uri contentUri = FileProvider.getUriForFile(context, <authorities>, imageFile);
  intent.setDataAndType(contentUri, "image/*");
} else {
  intent.setDataAndType(Uri.fromFile(imageFile), "image/*");
}
startActivity(intent);
```
注意替换上边的 <local-file-path>，替换为自己的文件路径即可。还有 <authorities>，这个值要与 第 1 步中的 <authorities> 相同。

然后就可以正常使用了。