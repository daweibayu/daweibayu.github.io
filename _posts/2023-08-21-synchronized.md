---
layout: post
title:  "synchronized"
author: "daweibayu"
tags: Android
excerpt_separator: <!--more-->
---

<!--more-->

前边几篇文章分别介绍了线程的本质、进程间通讯，剩下的比较重要的一环就是关于线程安全的问题了。因为整个知识点太大，这里只拿 synchronized 为例，依然如前，只关注主线，从 framework、jvm 一直到内核实现，一杆子捅到底。


这里需要注意的是 Android 并不是使用的标准 glibc，而是 [bionic](https://android.googlesource.com/platform/bionic/)，所以源码别找错了

### pthread_mutex_lock

```
int pthread_mutex_lock(pthread_mutex_t* mutex_interface) {
#if !defined(__LP64__)
    // Some apps depend on being able to pass NULL as a mutex and get EINVAL
    // back. Don't need to worry about it for LP64 since the ABI is brand new,
    // but keep compatibility for LP32. http://b/19995172.
    if (mutex_interface == nullptr) {
        return EINVAL;
    }
#endif

    pthread_mutex_internal_t* mutex = __get_internal_mutex(mutex_interface);
    uint16_t old_state = atomic_load_explicit(&mutex->state, memory_order_relaxed);
    uint16_t mtype = (old_state & MUTEX_TYPE_MASK);
    // Avoid slowing down fast path of normal mutex lock operation.
    if (__predict_true(mtype == MUTEX_TYPE_BITS_NORMAL)) {
        uint16_t shared = (old_state & MUTEX_SHARED_MASK);
        if (__predict_true(NonPI::NormalMutexTryLock(mutex, shared) == 0)) {
            return 0;
        }
    }
    if (old_state == PI_MUTEX_STATE) {
        PIMutex& m = mutex->ToPIMutex();
        // Handle common case first.
        if (__predict_true(PIMutexTryLock(m) == 0)) {
            return 0;
        }
        return PIMutexTimedLock(mutex->ToPIMutex(), false, nullptr);
    }
    if (__predict_false(IsMutexDestroyed(old_state))) {
        return HandleUsingDestroyedMutex(mutex_interface, __FUNCTION__);
    }
    return NonPI::MutexLockWithTimeout(mutex, false, nullptr);
}
```
```
static int (*ll_pthread_mutex_lock)(pthread_mutex_t *mutex)	= __pthread_mutex_lock;
```

__NR_futex
sys_futex_time32