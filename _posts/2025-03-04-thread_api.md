---
layout: post
title:  "Thread API"
author: "daweibayu"
tags: Android
excerpt_separator: <!--more-->
---

<!--more-->



最近在面试，遇到不少要手撕 Thread 相关的面试题，发现确实有不少东西还是模棱两可，所以写本文梳理一下线程相关逻辑。本文不重在线程相关的原理，而是使用。为了让本文更专注一点，所以只列 API。

## pthread

不管是 Kotlin、Java 还是其他 JVM 语言，本身的线程都是由 Native 支持，所以先看一下 Native 的 API。

所谓的 pthread 即是 posix(Portable Operating System Interface) thread，相关的协议规范于 [IEEE Std 1003](https://standards.ieee.org/ieee/1003.1/7700/)，文档本身收费，其接口定义则由 [The Open Group](https://www.opengroup.org/) 维护，好在这个不收费，其中关于 pthread 的 api 位于 [pthread.h](https://pubs.opengroup.org/onlinepubs/9699919799/basedefs/pthread.h.html)。不论是哪种（NPTL、LinuxThreads、FreeBSD、Windows 等）实现，最终都要遵循这一规范。

和线程切换等相关的 api 如下：

```
int   pthread_create(pthread_t *, const pthread_attr_t *, void *(*)(void *), void *);
int   pthread_detach(pthread_t);
int   pthread_equal(pthread_t, pthread_t);
void  pthread_exit(void *);
int   pthread_cancel(pthread_t);
int   pthread_join(pthread_t, void **);


int   pthread_mutex_init(pthread_mutex_t *, const pthread_mutexattr_t *);
int   pthread_mutex_lock(pthread_mutex_t *);
int   pthread_mutex_destroy(pthread_mutex_t *);
int   pthread_mutex_trylock(pthread_mutex_t *);
int   pthread_mutex_unlock(pthread_mutex_t *);

int   pthread_cond_destroy(pthread_cond_t *);
int   pthread_cond_init(pthread_cond_t *, const pthread_condattr_t *);
int   pthread_cond_timedwait(pthread_cond_t *, pthread_mutex_t *, const struct timespec *);
int   pthread_cond_wait(pthread_cond_t *, pthread_mutex_t *);
int   pthread_cond_signal(pthread_cond_t *);
int   pthread_cond_broadcast(pthread_cond_t *);
int   pthread_once(pthread_once_t *, void (*)(void));
```



## Java Thread

Java 层和线程直接相关的 api 如下（suspend、resume、stop、destroy 等均以废弃）：

`Thread.java`
```java
public final void join(long millis)
public void interrupt()
public synchronized void start()
public static void sleep(long millis)
public static native void yield();
```

`Object.java`
```java
 public final native void wait(long timeout, int nanos)
 @FastNative public final native void notifyAll();
 @FastNative public final native void notify();
```

* `start` 通过 `pthread_create` 来创建内核线程；  
* `join` 是通过 `pthread_join` 来阻塞调用线程；  
* `yield` 是通过  `sched_yield` 来实现非阻塞且无延迟的让出 cpu [sched_yield](https://pubs.opengroup.org/onlinepubs/9699919799/)；  
* `sleep` 通过 `pthread_cond_timedwait` 结合高精度时钟（ CLOCK_REALTIME 或 CLOCK_MONOTONIC ）实现限时等待；  
* `wait` 是通过 `pthread_cond_timedwait(&_cond, &_mutex, &ts)` 来等带线程；  
* `notify` 则是通过 `pthread_cond_signal(&_cond)` 来唤醒一个等待的线程；  
* `notifyAll` 则是通过 `pthread_cond_broadcast(&_cond)` 来唤醒一个等待的线程；  


```C++
void ObjectMonitor::wait(jlong millis) {
    pthread_mutex_lock(&_mutex);  // 加锁
    if (millis == 0) {
        pthread_cond_wait(&_cond, &_mutex);  // 无限期等待
    } else {
        struct timespec ts;
        clock_gettime(CLOCK_REALTIME, &ts);
        ts.tv_sec += millis / 1000;
        ts.tv_nsec += (millis % 1000) * 1000000;
        pthread_cond_timedwait(&_cond, &_mutex, &ts);  // 超时等待
    }
    pthread_mutex_unlock(&_mutex);  // 释放锁
}
```

所以其实弄懂 pthread api 的作用，其实就知道 Java 层线程的边界在哪里，所以接下来详细介绍 pthread 的 api。


ps：这中间其实还有 art 层 `thread.cc` 的桥接，这个过程其实也会限制 Java 层调用的边界


## 详解 pthread api


* `pthread_create` 用于创建线程
* `pthread_detach` 用于分离线程
* `pthread_equal` 对比两个线程是否为同一个实体
* `pthread_exit` 主动终止当前线程
* `pthread_cancel` 向另一个线程发送取消请求
* `pthread_join` 阻塞等待指定线程结束，并回收其资源




pthread_mutex_init、pthread_mutex_lock、pthread_mutex_destroy、pthread_mutex_trylock、pthread_mutex_unlock
pthread_cond_destroy、pthread_cond_init、pthread_cond_timedwait、pthread_cond_wait、pthread_cond_signal、pthread_cond_broadcast、pthread_once