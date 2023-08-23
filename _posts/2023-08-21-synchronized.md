---
layout: post
title:  "synchronized 实现"
author: "daweibayu"
tags: Android
excerpt_separator: <!--more-->
---

<!--more-->


### 前言

前边几篇文章分别介绍了线程的本质、进程间通讯，剩下的比较重要的一环就是关于线程安全的问题了。因为整个知识点太大，这里只拿 synchronized 为例，

其实关于 synchronized 相关的文章很多，但大多聚焦于什么内存模型、锁升级、volatie 区别等等，你抄我，我抄你，文字一大堆，图片一大堆，但就是没有代码。大家就只能似是而非的去记忆，为啥总有人说八股，其实也是源于此。这里的弊端溢于言表，但目前市场如此，也是可悲可叹。

所以此篇文章，重在突出代码，展示完整的代码调用链。从 framework 层一直到内核层，算是一个导引吧，方便读者可以直接定位不同层的代码位置。缺点就是只重主线，有去无回，缺少单层之间的完整逻辑，当然这部分以后有时间了会在补充（但是要是没时间嘛...）。


Attention:
* Android 并不是使用的标准 glibc，而是 [bionic](https://android.googlesource.com/platform/bionic/)，所以源码别找错了

### framework 层


### art 层：Monitor

synchronized 在 art 层的实现就是在代码段前后分别添加 MonitorEnter 与 MonitorExit，具体实现如下

art/runtime/monitor.cc

/Users/daweibayu/workspace/sourcecode/art/runtime/jni/jni_internal.cc

```
  static jint MonitorEnter(JNIEnv* env, jobject java_object) NO_THREAD_SAFETY_ANALYSIS {
    CHECK_NON_NULL_ARGUMENT_RETURN(java_object, JNI_ERR);
    ScopedObjectAccess soa(env);
    ObjPtr<mirror::Object> o = soa.Decode<mirror::Object>(java_object);
    o = o->MonitorEnter(soa.Self());
    if (soa.Self()->HoldsLock(o)) {
      soa.Env()->monitors_.Add(o);
    }
    if (soa.Self()->IsExceptionPending()) {
      return JNI_ERR;
    }
    return JNI_OK;
  }

  static jint MonitorExit(JNIEnv* env, jobject java_object) NO_THREAD_SAFETY_ANALYSIS {
    CHECK_NON_NULL_ARGUMENT_RETURN(java_object, JNI_ERR);
    ScopedObjectAccess soa(env);
    ObjPtr<mirror::Object> o = soa.Decode<mirror::Object>(java_object);
    bool remove_mon = soa.Self()->HoldsLock(o);
    o->MonitorExit(soa.Self());
    if (remove_mon) {
      soa.Env()->monitors_.Remove(o);
    }
    if (soa.Self()->IsExceptionPending()) {
      return JNI_ERR;
    }
    return JNI_OK;
  }
```


### native lib 层：pthread_mutex_lock

锁三种类型：
```
#define PTHREAD_MUTEX_NORMAL		0
#define PTHREAD_MUTEX_ERRORCHECK	1
#define PTHREAD_MUTEX_RECURSIVE		2
#define PTHREAD_MUTEX_DEFAULT		PTHREAD_MUTEX_NORMAL

#define  MUTEX_TYPE_BITS_NORMAL      MUTEX_TYPE_TO_BITS(PTHREAD_MUTEX_NORMAL)      // 普通锁，不做死锁检测
#define  MUTEX_TYPE_BITS_RECURSIVE   MUTEX_TYPE_TO_BITS(PTHREAD_MUTEX_RECURSIVE)   // 可重入锁
#define  MUTEX_TYPE_BITS_ERRORCHECK  MUTEX_TYPE_TO_BITS(PTHREAD_MUTEX_ERRORCHECK)  // 检错锁
// Use a special mutex type to mark priority inheritance mutexes.
#define  PI_MUTEX_STATE     MUTEX_TYPE_TO_BITS(3)     // PIMutex（Priority Inheritance mutex），支持优先级继承的锁
```

具体实现：

```
int pthread_mutex_lock(pthread_mutex_t* mutex_interface) {
    ...
    pthread_mutex_internal_t* mutex = __get_internal_mutex(mutex_interface);
    uint16_t old_state = atomic_load_explicit(&mutex->state, memory_order_relaxed);
    uint16_t mtype = (old_state & MUTEX_TYPE_MASK);
    // Avoid slowing down fast path of normal mutex lock operation.
    if (__predict_true(mtype == MUTEX_TYPE_BITS_NORMAL)) {
        uint16_t shared = (old_state & MUTEX_SHARED_MASK);
        // 如果是普通锁，并且获取锁成功，直接返回
        if (__predict_true(NonPI::NormalMutexTryLock(mutex, shared) == 0)) {
            return 0;
        }
    }
    if (old_state == PI_MUTEX_STATE) {
        PIMutex& m = mutex->ToPIMutex();
        // 如果是支持优先级继承的锁，并且获取锁成功，直接返回
        if (__predict_true(PIMutexTryLock(m) == 0)) {
            return 0;
        }
        // 如果锁处于 EBUSY 状态，则最终会调用 Futex 系统调用
        return PIMutexTimedLock(mutex->ToPIMutex(), false, nullptr);
    }
    if (__predict_false(IsMutexDestroyed(old_state))) {
        return HandleUsingDestroyedMutex(mutex_interface, __FUNCTION__);
    }

    // 针对不支持优先级继承的锁，调用 MutexLockWithTimeout 来完成锁操作
    return NonPI::MutexLockWithTimeout(mutex, false, nullptr);
}

static int MutexLockWithTimeout(pthread_mutex_internal_t* mutex, bool use_realtime_clock,
                                const timespec* abs_timeout_or_null) {
    uint16_t old_state = atomic_load_explicit(&mutex->state, memory_order_relaxed);
    uint16_t mtype = (old_state & MUTEX_TYPE_MASK);
    uint16_t shared = (old_state & MUTEX_SHARED_MASK);

    if ( __predict_true(mtype == MUTEX_TYPE_BITS_NORMAL) ) {
        // 普通锁
        return NormalMutexLock(mutex, shared, use_realtime_clock, abs_timeout_or_null);
    }

    // Do we already own this recursive or error-check mutex?
    pid_t tid = __get_thread()->tid;
    if (tid == atomic_load_explicit(&mutex->owner_tid, memory_order_relaxed)) {
        if (mtype == MUTEX_TYPE_BITS_ERRORCHECK) {
            return EDEADLK;
        }
        // 可重入锁
        return RecursiveIncrement(mutex, old_state);
    }

    const uint16_t unlocked           = mtype | shared | MUTEX_STATE_BITS_UNLOCKED;
    const uint16_t locked_uncontended = mtype | shared | MUTEX_STATE_BITS_LOCKED_UNCONTENDED;
    const uint16_t locked_contended   = mtype | shared | MUTEX_STATE_BITS_LOCKED_CONTENDED;

    // First, if the mutex is unlocked, try to quickly acquire it.
    // In the optimistic case where this works, set the state to locked_uncontended.
    if (old_state == unlocked) {
        // If exchanged successfully, an acquire fence is required to make
        // all memory accesses made by other threads visible to the current CPU.
        if (__predict_true(atomic_compare_exchange_strong_explicit(&mutex->state, &old_state,
                             locked_uncontended, memory_order_acquire, memory_order_relaxed))) {
            atomic_store_explicit(&mutex->owner_tid, tid, memory_order_relaxed);
            return 0;
        }
    }

    ScopedTrace trace("Contending for pthread mutex");

    while (true) {
        // 针对不同的状态做轮询判断
        if (old_state == unlocked) {
            // NOTE: We put the state to locked_contended since we _know_ there
            // is contention when we are in this loop. This ensures all waiters
            // will be unlocked.

            // If exchanged successfully, an acquire fence is required to make
            // all memory accesses made by other threads visible to the current CPU.
            if (__predict_true(atomic_compare_exchange_weak_explicit(&mutex->state,
                                                                     &old_state, locked_contended,
                                                                     memory_order_acquire,
                                                                     memory_order_relaxed))) {
                atomic_store_explicit(&mutex->owner_tid, tid, memory_order_relaxed);
                return 0;
            }
            continue;
        } else if (MUTEX_STATE_BITS_IS_LOCKED_UNCONTENDED(old_state)) {
            // We should set it to locked_contended beforing going to sleep. This can make
            // sure waiters will be woken up eventually.

            int new_state = MUTEX_STATE_BITS_FLIP_CONTENTION(old_state);
            if (__predict_false(!atomic_compare_exchange_weak_explicit(&mutex->state,
                                                                       &old_state, new_state,
                                                                       memory_order_relaxed,
                                                                       memory_order_relaxed))) {
                continue;
            }
            old_state = new_state;
        }

        // 校验时间
        int result = check_timespec(abs_timeout_or_null, true);
        if (result != 0) {
            return result;
        }
        // We are in locked_contended state, sleep until someone wakes us up.
        // 调用 __futex_wait_ex 实现
        if (RecursiveOrErrorcheckMutexWait(mutex, shared, old_state, use_realtime_clock,
                                           abs_timeout_or_null) == -ETIMEDOUT) {
            return ETIMEDOUT;
        }
        old_state = atomic_load_explicit(&mutex->state, memory_order_relaxed);
    }
}
```

```
static inline __always_inline int NormalMutexLock(pthread_mutex_internal_t* mutex,
                                                  uint16_t shared,
                                                  bool use_realtime_clock,
                                                  const timespec* abs_timeout_or_null) {
    if (__predict_true(NormalMutexTryLock(mutex, shared) == 0)) {
        return 0;
    }
    int result = check_timespec(abs_timeout_or_null, true);
    if (result != 0) {
        return result;
    }

    ScopedTrace trace("Contending for pthread mutex");

    const uint16_t unlocked           = shared | MUTEX_STATE_BITS_UNLOCKED;
    const uint16_t locked_contended = shared | MUTEX_STATE_BITS_LOCKED_CONTENDED;

    // We want to go to sleep until the mutex is available, which requires
    // promoting it to locked_contended. We need to swap in the new state
    // and then wait until somebody wakes us up.
    // An atomic_exchange is used to compete with other threads for the lock.
    // If it returns unlocked, we have acquired the lock, otherwise another
    // thread still holds the lock and we should wait again.
    // If lock is acquired, an acquire fence is needed to make all memory accesses
    // made by other threads visible to the current CPU.
    while (atomic_exchange_explicit(&mutex->state, locked_contended,
                                    memory_order_acquire) != unlocked) {
        if (__futex_wait_ex(&mutex->state, shared, locked_contended, use_realtime_clock,
                            abs_timeout_or_null) == -ETIMEDOUT) {
            return ETIMEDOUT;
        }
    }
    return 0;
}
```

```
static inline __always_inline int FutexWithTimeout(volatile void* ftx, int op, int value,
                                                   bool use_realtime_clock,
                                                   const timespec* abs_timeout, int bitset) {
  const timespec* futex_abs_timeout = abs_timeout;
  // pthread's and semaphore's default behavior is to use CLOCK_REALTIME, however this behavior is
  // essentially never intended, as that clock is prone to change discontinuously.
  //
  // What users really intend is to use CLOCK_MONOTONIC, however only pthread_cond_timedwait()
  // provides this as an option and even there, a large amount of existing code does not opt into
  // CLOCK_MONOTONIC.
  //
  // We have seen numerous bugs directly attributable to this difference.  Therefore, we provide
  // this general workaround to always use CLOCK_MONOTONIC for waiting, regardless of what the input
  // timespec is.
  timespec converted_monotonic_abs_timeout;
  if (abs_timeout && use_realtime_clock) {
    monotonic_time_from_realtime_time(converted_monotonic_abs_timeout, *abs_timeout);
    if (converted_monotonic_abs_timeout.tv_sec < 0) {
      return -ETIMEDOUT;
    }
    futex_abs_timeout = &converted_monotonic_abs_timeout;
  }

  return __futex(ftx, op, value, futex_abs_timeout, bitset);
}

int __futex_wait_ex(volatile void* ftx, bool shared, int value, bool use_realtime_clock,
                    const timespec* abs_timeout) {
  return FutexWithTimeout(ftx, (shared ? FUTEX_WAIT_BITSET : FUTEX_WAIT_BITSET_PRIVATE), value,
                          use_realtime_clock, abs_timeout, FUTEX_BITSET_MATCH_ANY);
}

int __futex_pi_lock_ex(volatile void* ftx, bool shared, bool use_realtime_clock,
                       const timespec* abs_timeout) {
  return FutexWithTimeout(ftx, (shared ? FUTEX_LOCK_PI : FUTEX_LOCK_PI_PRIVATE), 0,
                          use_realtime_clock, abs_timeout, 0);
}
```

```
static inline __always_inline int __futex(volatile void* ftx, int op, int value,
                                          const timespec* timeout, int bitset) {
  // Our generated syscall assembler sets errno, but our callers (pthread functions) don't want to.
  int saved_errno = errno;
  int result = syscall(__NR_futex, ftx, op, value, timeout, NULL, bitset);
  if (__predict_false(result == -1)) {
    result = -errno;
    errno = saved_errno;
  }
  return result;
}

```

调用 syscall __NR_futex 从而进入内核态

### kernel 层：Futex(Fast User-space Mutex)

__NR_futex
sys_futex_time32

所以可以简单理解:
1. 在 framework 层做的东西不多，就是在代码段前加了 monitorenter 和代码段后添加了 monitorexit
2. 在 art、native libs 层通过 pthread_mutex_lock 等函数族实现
3. 在 kernel 层通过 futex 实现


关于 futex 介绍的比较好的文章 [Futex 简述](http://blog.foool.net/2021/04/futex-%E7%BB%BC%E8%BF%B0/)
[手机平台上的用户空间锁概述](https://blog.csdn.net/feelabclihu/article/details/125814721)

