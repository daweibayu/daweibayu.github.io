---
layout: post
title:  "进程间通讯"
author: "daweibayu"
tags: Android
excerpt_separator: <!--more-->
---

<!--more-->

Android：Binder


## 缘由

前些天面试，聊到进程间通讯，大家都知道 Android 进程间通讯最常见的方式是 Binder，而 Binder 使用 mmap，也就是内存映射，但是对于内存映射的具体实现，双方略有不同意见。感觉这里也蛮有趣，大家都背贯了八股，但是对于具体，却都是模棱两可，所以感觉这个议题很有写一篇的必要。

## 通讯

通讯的最终目的：**发送方A** 把 **数据α** 给到 **接收方B**。一般计算机语境中，通讯分为两种，线程间通讯与进程间通讯。  
线程间通讯相对简单。以 Android 举例，其逻辑就是在线程 Ta 的执行流中把数据 α 放到线程 Tb 的 message queue 中，然后线程 Tb 的 Looper 轮询到 α，然后开始执行，从而完成此次通讯。谈线程的时候因为都可以认为是同一地址空间，所以没那么多概念。  
Linux 下常见的进程通讯方式有 Socket、管道、信号、消息队列、共享内存等。其原理自然也是类似，不同点就是进程 Pa 与进程 Pb 在不同的虚拟地址空间内，由操作系统隔离，从而不能像线程一样直接操作对方的 message queue。各种概念就由此多了起来，比如虚拟地址、物理地址、内核态、用户态等等。


## Android 进程间通讯

这里就以 Android 的进程间通讯为例，最上层一直到最下层，看一下具体怎么实现，这里只关注主线，挂一漏万，其他非主线逻辑，诸位看官可以自行搜索。

我们知道 Broadcast 是可以做到跨进程通讯的，就以 Broadcast 为例，我们看看具体是怎么实现的。照旧，不扯淡，直接看代码

1. 日常发送广播代码如下：

```kotlin
Intent().also { intent ->
    intent.setAction("com.example.broadcast.MY_NOTIFICATION")
    intent.putExtra("data", "Notice me senpai!")
    sendBroadcast(intent)
}
```

2. ContextWrapper.java 中的实现：

```java
@Override
public void sendBroadcast(Intent intent) {
    mBase.sendBroadcast(intent);
}
```

3. 其中 Context 的实现类为 ContextImpl.java，sendBroadcast 实现如下：

```java
@Override
public void sendBroadcast(Intent intent) {
    warnIfCallingFromSystemProcess();
    String resolvedType = intent.resolveTypeIfNeeded(getContentResolver());
    try {
        intent.prepareToLeaveProcess(this);
        ActivityManager.getService().broadcastIntentWithFeature(
                mMainThread.getApplicationThread(), getAttributionTag(), intent, resolvedType,
                null, Activity.RESULT_OK, null, null, null, null /*excludedPermissions=*/,
                null, AppOpsManager.OP_NONE, null, false, false, getUserId());
    } catch (RemoteException e) {
        throw e.rethrowFromSystemServer();
    }
}
```

所有逻辑都在 `ActivityManager.getService().broadcastIntentWithFeature()` 中，按照我们常规了解到的知识而言：

* ServiceManager 是用于处理 binder 通讯的，单独一个进程
* SystemServer 包含各种服务，包含 ActivityManagerService、WindowManagerService、InputManagerService 等，单独一个进程
* app 进程当然也是一个单独进程

所以单独这一行代码 `ActivityManager.getService().broadcastIntentWithFeature` 貌似就跨了三个进程，所以事实如何呢？这行代码分为两部分，一部分是 getService，另一部分是 broadcastIntentWithFeature()，我们挨个去看

### getService

这里的函数调用主线为：
- ActivityManager::getService
  - ServiceManager::getService
    - ServiceManager::rawGetService
      - BinderInternal::getContextObject

看官可以根据这个主线去看下面的代码  
[frameworks/base/core/java/android/app/ActivityManager.java](https://cs.android.com/android/platform/superproject/main/+/main:frameworks/base/core/java/android/app/ActivityManager.java;l=4805) 中代码如下：

```java
public static IActivityManager getService() {
    return IActivityManagerSingleton.get();
}

@UnsupportedAppUsage
private static final Singleton<IActivityManager> IActivityManagerSingleton =
        new Singleton<IActivityManager>() {
            @Override
            protected IActivityManager create() {
                final IBinder b = ServiceManager.getService(Context.ACTIVITY_SERVICE);
                final IActivityManager am = IActivityManager.Stub.asInterface(b);
                return am;
            }
        };
```

[frameworks/base/core/java/android/os/ServiceManager.java](https://cs.android.com/android/platform/superproject/main/+/main:frameworks/base/core/java/android/os/ServiceManager.java;l=141) 中代码如下：

```java
@UnsupportedAppUsage
public static IBinder getService(String name) {
    try {
        IBinder service = sCache.get(name);
        if (service != null) {
            return service;
        } else {
            return Binder.allowBlocking(rawGetService(name));
        }
    } catch (RemoteException e) {
        Log.e(TAG, "error in getService", e);
    }
    return null;
}

private static IBinder rawGetService(String name) throws RemoteException {
    final long start = sStatLogger.getTime();

    final IBinder binder = getIServiceManager().getService(name);

    final int time = (int) sStatLogger.logDurationStat(Stats.GET_SERVICE, start);

    final int myUid = Process.myUid();
    final boolean isCore = UserHandle.isCore(myUid);

    final long slowThreshold = isCore
            ? GET_SERVICE_SLOW_THRESHOLD_US_CORE
            : GET_SERVICE_SLOW_THRESHOLD_US_NON_CORE;

    synchronized (sLock) {
        sGetServiceAccumulatedUs += time;
        sGetServiceAccumulatedCallCount++;

        final long nowUptime = SystemClock.uptimeMillis();

        // Was a slow call?
        if (time >= slowThreshold) {
            // We do a slow log:
            // - At most once in every SLOW_LOG_INTERVAL_MS
            // - OR it was slower than the previously logged slow call.
            if ((nowUptime > (sLastSlowLogUptime + SLOW_LOG_INTERVAL_MS))
                    || (sLastSlowLogActualTime < time)) {
                EventLogTags.writeServiceManagerSlow(time / 1000, name);

                sLastSlowLogUptime = nowUptime;
                sLastSlowLogActualTime = time;
            }
        }

        // Every GET_SERVICE_LOG_EVERY_CALLS calls, log the total time spent in getService().

        final int logInterval = isCore
                ? GET_SERVICE_LOG_EVERY_CALLS_CORE
                : GET_SERVICE_LOG_EVERY_CALLS_NON_CORE;

        if ((sGetServiceAccumulatedCallCount >= logInterval)
                && (nowUptime >= (sLastStatsLogUptime + STATS_LOG_INTERVAL_MS))) {

            EventLogTags.writeServiceManagerStats(
                    sGetServiceAccumulatedCallCount, // Total # of getService() calls.
                    sGetServiceAccumulatedUs / 1000, // Total time spent in getService() calls.
                    (int) (nowUptime - sLastStatsLogUptime)); // Uptime duration since last log.
            sGetServiceAccumulatedCallCount = 0;
            sGetServiceAccumulatedUs = 0;
            sLastStatsLogUptime = nowUptime;
        }
    }
    return binder;
}

@UnsupportedAppUsage
private static IServiceManager getIServiceManager() {
    if (sServiceManager != null) {
        return sServiceManager;
    }

    // Find the service manager
    sServiceManager = ServiceManagerNative.asInterface(Binder.allowBlocking(BinderInternal.getContextObject()));
    return sServiceManager;
}
```


[frameworks/base/core/java/com/android/internal/os/BinderInternal.java](https://cs.android.com/android/platform/superproject/main/+/main:frameworks/base/core/java/com/android/internal/os/BinderInternal.java;l=180)

```java
public static final native IBinder getContextObject();
```

getContextObject 为 native 函数，具体实现在在 [frameworks/base/core/jni/android_util_Binder.cpp](https://cs.android.com/android/platform/superproject/main/+/main:frameworks/base/core/jni/android_util_Binder.cpp;l=1160)

其下的函数调用主线如下，大家可以根据此主要链条参看 ProcessState 代码
- getContextObject
  - ProcessState::self()
    - ProcessState::init
      - sp<ProcessState>::make
        - ProcessState::ProcessState
          - mmap
  - ProcessState::getContextObject
    - getStrongProxyForHandle
        - ipc->transact



```c++
static jobject android_os_BinderInternal_getContextObject(JNIEnv* env, jobject clazz)
{
    sp<IBinder> b = ProcessState::self()->getContextObject(NULL);
    return javaObjectForIBinder(env, b);
}

```

[frameworks/native/libs/binder/ProcessState.cpp](https://cs.android.com/android/platform/superproject/main/+/main:frameworks/native/libs/binder/ProcessState.cpp;l=153) 代码如下：

```c++
sp<IBinder> ProcessState::getContextObject(const sp<IBinder>& /*caller*/)
{
    sp<IBinder> context = getStrongProxyForHandle(0);

    if (context) {
        // The root object is special since we get it directly from the driver, it is never
        // written by Parcell::writeStrongBinder.
        internal::Stability::markCompilationUnit(context.get());
    } else {
        ALOGW("Not able to get context object on %s.", mDriverName.c_str());
    }

    return context;
}


sp<IBinder> ProcessState::getStrongProxyForHandle(int32_t handle)
{
    sp<IBinder> result;

    AutoMutex _l(mLock);

    if (handle == 0 && the_context_object != nullptr) return the_context_object;

    handle_entry* e = lookupHandleLocked(handle);

    if (e != nullptr) {

        IBinder* b = e->binder;
        if (b == nullptr || !e->refs->attemptIncWeak(this)) {
            if (handle == 0) {

                IPCThreadState* ipc = IPCThreadState::self();

                CallRestriction originalCallRestriction = ipc->getCallRestriction();
                ipc->setCallRestriction(CallRestriction::NONE);

                Parcel data;
                status_t status = ipc->transact(
                        0, IBinder::PING_TRANSACTION, data, nullptr, 0);

                ipc->setCallRestriction(originalCallRestriction);

                if (status == DEAD_OBJECT)
                   return nullptr;
            }

            sp<BpBinder> b = BpBinder::PrivateAccessor::create(handle);
            e->binder = b.get();
            if (b) e->refs = b->getWeakRefs();
            result = b;
        } else {
            result.force_set(b);
            e->refs->decWeak(this);
        }
    }

    return result;
}

sp<ProcessState> ProcessState::self()
{
    return init(kDefaultDriver, false /*requireDefault*/);
}

sp<ProcessState> ProcessState::init(const char* driver, bool requireDefault) {
    if (driver == nullptr) {
        std::lock_guard<std::mutex> l(gProcessMutex);
        if (gProcess) {
            verifyNotForked(gProcess->mForked);
        }
        return gProcess;
    }

    [[clang::no_destroy]] static std::once_flag gProcessOnce;
    std::call_once(gProcessOnce, [&](){
        if (access(driver, R_OK) == -1) {
            ALOGE("Binder driver %s is unavailable. Using /dev/binder instead.", driver);
            driver = "/dev/binder";
        }

        if (0 == strcmp(driver, "/dev/vndbinder") && !isVndservicemanagerEnabled()) {
            ALOGE("vndservicemanager is not started on this device, you can save resources/threads "
                  "by not initializing ProcessState with /dev/vndbinder.");
        }


        int ret = pthread_atfork(ProcessState::onFork, ProcessState::parentPostFork,
                                 ProcessState::childPostFork);
        LOG_ALWAYS_FATAL_IF(ret != 0, "pthread_atfork error %s", strerror(ret));

        std::lock_guard<std::mutex> l(gProcessMutex);
        gProcess = sp<ProcessState>::make(driver);
    });

    if (requireDefault) {
        LOG_ALWAYS_FATAL_IF(gProcess->getDriverName() != driver,
                            "ProcessState was already initialized with %s,"
                            " can't initialize with %s.",
                            gProcess->getDriverName().c_str(), driver);
    }

    verifyNotForked(gProcess->mForked);
    return gProcess;
}

ProcessState::ProcessState(const char* driver)
      : mDriverName(String8(driver)),
        mDriverFD(-1),
        mVMStart(MAP_FAILED),
        mThreadCountLock(PTHREAD_MUTEX_INITIALIZER),
        mThreadCountDecrement(PTHREAD_COND_INITIALIZER),
        mExecutingThreadsCount(0),
        mWaitingForThreads(0),
        mMaxThreads(DEFAULT_MAX_BINDER_THREADS),
        mCurrentThreads(0),
        mKernelStartedThreads(0),
        mStarvationStartTimeMs(0),
        mForked(false),
        mThreadPoolStarted(false),
        mThreadPoolSeq(1),
        mCallRestriction(CallRestriction::NONE) {
    base::Result<int> opened = open_driver(driver);

    if (opened.ok()) {
        // mmap the binder, providing a chunk of virtual address space to receive transactions.
        mVMStart = mmap(nullptr, BINDER_VM_SIZE, PROT_READ, MAP_PRIVATE | MAP_NORESERVE,
                        opened.value(), 0);
        if (mVMStart == MAP_FAILED) {
            close(opened.value());
            // *sigh*
            opened = base::Error()
                    << "Using " << driver << " failed: unable to mmap transaction memory.";
            mDriverName.clear();
        }
    }

#ifdef __ANDROID__
    LOG_ALWAYS_FATAL_IF(!opened.ok(), "Binder driver '%s' could not be opened. Terminating: %s",
                        driver, opened.error().message().c_str());
#endif

    if (opened.ok()) {
        mDriverFD = opened.value();
    }
}
```

```c++
sp<T> sp<T>::make(Args&&... args) {
    T* t = new T(std::forward<Args>(args)...);
    sp<T> result;
    result.m_ptr = t;
    t->incStrong(t);
    return result;
}
```

由 std::call_once，我们可以看出，ProcessState 是一个单例。我们一直说 Binder 基于 mmap，当然最终实现也是如此，从代码一步一步最终也可以看出来。而且我们也都知道 mmap 是基于内存映射，即然都已经到这了，那我们就一杆子捅到底，看看 mmap 到底是怎么实现的。


#### mmap 实现


prebuilts/gcc/linux-x86/host/x86_64-linux-glibc2.17-4.8/sysroot/usr/include/i386-linux-gnu/sys/mman.h
```c++
extern void * __REDIRECT_NTH (mmap,
			      (void *__addr, size_t __len, int __prot,
			       int __flags, int __fd, __off64_t __offset),
			      mmap64);
```

bionic/libc/bionic/mmap.cpp

```c++
extern "C" void*  __mmap2(void*, size_t, int, int, int, size_t);

void* mmap64(void* addr, size_t size, int prot, int flags, int fd, off64_t offset) {
  if (offset < 0 || (offset & ((1UL << MMAP2_SHIFT)-1)) != 0) {
    errno = EINVAL;
    return MAP_FAILED;
  }

  // Prevent allocations large enough for `end - start` to overflow.
  size_t rounded = __BIONIC_ALIGN(size, page_size());
  if (rounded < size || rounded > PTRDIFF_MAX) {
    errno = ENOMEM;
    return MAP_FAILED;
  }

  return __mmap2(addr, size, prot, flags, fd, offset >> MMAP2_SHIFT);
}
```


未完待续



### broadcastIntentWithFeature

3. 通过查看 broadcastIntentWithFeature 函数可以知道，broadcastIntentWithFeature -> broadcastIntentLocked -> broadcastIntentLockedTraced -> enqueueBroadcastLocked(BroadcastQueue 的实现在 BroadcastQueueImpl.java 中)



## 其他

shmget vs mmap，这里需要说明的一点，严格意义上来说，mmap 是 memory map，翻译过来是内存映射，而共享内存是 shmem，实际是并不相同的。只不过 shmem 族的 api 是 System V 规范的，而 mmap 是 POSIX 规范的。shmem 更老支持也更广泛一点，mmap 会更简单一点。  
binder 不太适合数据量大的通讯，因为需要 Parcel  
shmget 没有提供同步机制  
binder(mmap)通过 mutex_lock 提供同步机制