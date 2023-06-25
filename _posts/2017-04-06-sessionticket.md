---
layout: post
title:  "Android 启用 SessionTicket"
author: "daweibayu"
tags: Android
excerpt_separator: <!--more-->
---

<!--more-->

貌似还没有看到 Android 关于 SessionTicket，前些天正好用到，这里写一下。
关于协议相关的，这里就不赘述了，感兴趣的可以去看 [RFC](https://www.ietf.org/rfc/rfc5077.txt)

这里先把具体代码列一下，然后我们在细说。
``` java
SSLSessionCache sessionCache = new SSLSessionCache(getApplicationContext());
SSLCertificateSocketFactory socketFactory = (SSLCertificateSocketFactory)SSLCertificateSocketFactory.getDefault(5000, sessionCache);
Socket socket = socketFactory.createSocket();
if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.JELLY_BEAN_MR1) {
 socketFactory.setUseSessionTickets(socket, true);
}
```

需要注意的地方：

* SSLSessionCache、SSLCertificateSocketFactory 可以做成单例，这里只是方便展示，所以直接 new 的对象（当然不搞单例也没什么问题）。
* SSLCertificateSocketFactory.getDefault(5000, sessionCache) 中的 5000 仅为个人经验值，大家可以根据自己的需要设置不同值。

其中涉及到的主要是两个类 [SSLSessionCache.java](https://github.com/android/platform_frameworks_base/blob/master/core/java/android/net/SSLSessionCache.java)、[SSLCertificateSocketFactory.java](https://github.com/android/platform_frameworks_base/blob/master/core/java/android/net/SSLCertificateSocketFactory.java)

## 关于 SessionTicket：

我这抓包截了几张给大家看一下：

这一张是正常的 TLS 握手，可以看到其中 NO.92 中的长度为 3278：
![ServerHelloWithCertificate.png](/assets/images/bae3e1999f29aa1f.webp)

展开后可以看到下图，其中 Certificate 占了 2795：
![CertificateLeangth.png](/assets/images/151e724dfffce13c.webp)

具体的 SessionTicket 是在第一张图中的 NO.96 发送过来的，具体展开可以看到 SessionTicket 的具体内容，如图：
![SessionTicket.png](/assets/images/6de6c2263fd459ac.webp)


SSLSessionCache 负责上述 Session 的缓存。当已经有缓存了以后，再次 Client Hello 会把 SessionTicket 附上，所以图一中的 Client Hello 长度只有 237，下图的 ClientHello 为 571，而 Server Hello 只有 191 了：
![ServerHello.png](/assets/images/009e31f4065360db.webp)

至于 SessionTicket 怎么生成: [阮一峰 - 图解SSL/TLS协议](http://www.ruanyifeng.com/blog/2014/09/illustration-ssl.html)