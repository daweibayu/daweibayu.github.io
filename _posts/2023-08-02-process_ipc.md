---
layout: post
title:  "进程间通讯"
author: "daweibayu"
tags: Android
excerpt_separator: <!--more-->
---

<!--more-->


## 缘由

前些天面试，聊到进程间通讯，大家都知道 Android 进程间通讯最常见的方式是 Binder，而 Binder 使用 mmap，也就是内存映射，但是对于内存映射的具体实现，双方略有不同意见。感觉这里也蛮有趣，大家都背贯了八股，但是对于具体，却都是模棱两可，所以感觉这个议题很有写一篇的必要。

## 通讯

通讯的最终目的：`发送方A` 把 `数据α` 给到 `接收方B`。一般计算机语境中，通讯分为两种，线程间通讯与进程间通讯。  

线程间通讯相对简单。以 Android 举例，其逻辑就是在`线程 Ta`的执行流中把`数据 α `放到`线程 Tb`的 `message queue` 中，然后`线程 Tb` 的 `Looper` 轮询到 `数据α`，然后开始执行，从而完成此次通讯。谈线程的时候因为都可以认为是同一地址空间，所以没那么多概念。  

Linux 下常见的进程通讯方式有 Socket、管道、信号、消息队列、共享内存等。其原理自然也是类似，不同点就是`进程 Pa` 与`进程 Pb`在不同的虚拟地址空间内，由操作系统隔离，从而不能像线程一样直接操作对方的 message queue。各种概念就由此多了起来，比如虚拟地址、物理地址、内核态、用户态等等。


## Android 进程间通讯

这里就以 Android 的进程间通讯为例，以深度优先方式从最上层一直到最下层，看一下具体是怎么实现的，这里只关注主线，挂一漏万，其他非主线逻辑，诸位看官可以自行搜索。

我们知道 Broadcast 是可以做到跨进程通讯的，就以 Broadcast 为例，我们看看具体是怎么实现的。照旧，不扯淡，直接看代码

日常发送广播代码如下：

```kotlin
Intent().also { intent ->
    intent.setAction("com.example.broadcast.MY_NOTIFICATION")
    intent.putExtra("data", "Notice me senpai!")
    sendBroadcast(intent)
}
```

ContextWrapper.java 中的实现：

```java
@Override
public void sendBroadcast(Intent intent) {
    mBase.sendBroadcast(intent);
}
```

其中 Context 的实现类为 ContextImpl.java，sendBroadcast 实现如下：

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

* ServiceManager 用于处理 binder 通讯，单独一个进程
* SystemServer 包含各种服务，包含 ActivityManagerService、WindowManagerService、InputManagerService 等，单独一个进程
* app 进程当然也是一个单独进程

所以单独这一行代码 `ActivityManager.getService().broadcastIntentWithFeature` 貌似就跨了三个进程，所以事实如何呢？这行代码分为两部分，一部分是 getService，另一部分是 broadcastIntentWithFeature()，我们挨个去看

### getService

这里的函数调用主线为：
- ActivityManager::getService
  - ServiceManager::getService
    - ServiceManager::rawGetService
      - BinderInternal::getContextObject

看官可以根据这个主线去看下面的代码 [frameworks/base/core/java/android/app/ActivityManager.java](https://cs.android.com/android/platform/superproject/main/+/main:frameworks/base/core/java/android/app/ActivityManager.java;l=4805) 中代码如下：

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

接着往下看 [frameworks/base/core/java/com/android/internal/os/BinderInternal.java](https://cs.android.com/android/platform/superproject/main/+/main:frameworks/base/core/java/com/android/internal/os/BinderInternal.java;l=180) 实现：


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

由 std::call_once，我们可以看出，ProcessState 是一个单例。我们一直说 Binder 基于 mmap，当然最终实现也是如此，从代码一步一步最终也可以看出来。而且我们也都知道 mmap 是基于内存映射，但内存映射到底是怎么实现的呢？我们接着往下看


#### mmap 内核层实现


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

linux/fs.h（这个头文件对于理解 Linux 的文件体系还是蛮重要的，建议有时间有兴趣的可以好好看一下）

```c++
struct file_operations {
	struct module *owner;
	loff_t (*llseek) (struct file *, loff_t, int);
	ssize_t (*read) (struct file *, char __user *, size_t, loff_t *);
	ssize_t (*write) (struct file *, const char __user *, size_t, loff_t *);
	ssize_t (*read_iter) (struct kiocb *, struct iov_iter *);
	ssize_t (*write_iter) (struct kiocb *, struct iov_iter *);
	int (*iopoll)(struct kiocb *kiocb, bool spin);
	int (*iterate) (struct file *, struct dir_context *);
	int (*iterate_shared) (struct file *, struct dir_context *);
	__poll_t (*poll) (struct file *, struct poll_table_struct *);
	long (*unlocked_ioctl) (struct file *, unsigned int, unsigned long);
	long (*compat_ioctl) (struct file *, unsigned int, unsigned long);
	int (*mmap) (struct file *, struct vm_area_struct *);
	unsigned long mmap_supported_flags;
	int (*open) (struct inode *, struct file *);
	int (*flush) (struct file *, fl_owner_t id);
	int (*release) (struct inode *, struct file *);
	int (*fsync) (struct file *, loff_t, loff_t, int datasync);
	int (*fasync) (int, struct file *, int);
	int (*lock) (struct file *, int, struct file_lock *);
	ssize_t (*sendpage) (struct file *, struct page *, int, size_t, loff_t *, int);
	unsigned long (*get_unmapped_area)(struct file *, unsigned long, unsigned long, unsigned long, unsigned long);
	int (*check_flags)(int);
	int (*flock) (struct file *, int, struct file_lock *);
	ssize_t (*splice_write)(struct pipe_inode_info *, struct file *, loff_t *, size_t, unsigned int);
	ssize_t (*splice_read)(struct file *, loff_t *, struct pipe_inode_info *, size_t, unsigned int);
	int (*setlease)(struct file *, long, struct file_lock **, void **);
	long (*fallocate)(struct file *file, int mode, loff_t offset,
			  loff_t len);
	void (*show_fdinfo)(struct seq_file *m, struct file *f);
#ifndef CONFIG_MMU
	unsigned (*mmap_capabilities)(struct file *);
#endif
	ssize_t (*copy_file_range)(struct file *, loff_t, struct file *,
			loff_t, size_t, unsigned int);
	loff_t (*remap_file_range)(struct file *file_in, loff_t pos_in,
				   struct file *file_out, loff_t pos_out,
				   loff_t len, unsigned int remap_flags);
	int (*fadvise)(struct file *, loff_t, loff_t, int);

	ANDROID_KABI_RESERVE(1);
	ANDROID_KABI_RESERVE(2);
	ANDROID_KABI_RESERVE(3);
	ANDROID_KABI_RESERVE(4);
} __randomize_layout;
```

其中对于 mmap 的映射关系绑定在 drivers/android/binder.c，struct file_operations 的 mmap 在其他逻辑里还会绑定 hpet_mmap、cached_mmap、kfd_mmap、radeon_mmap、vmw_mmap、siw_mmap、vb2_fop_mmap、open_dice_mmap、sg_mmap、ashmem_mmap、usbdev_mmap、incfs_file_mmap、sock_mmap、had_pcm_mmap、kvm_device_mmap 等等

```c++

const struct file_operations binder_fops = {
	.owner = THIS_MODULE,
	.poll = binder_poll,
	.unlocked_ioctl = binder_ioctl,
	.compat_ioctl = compat_ptr_ioctl,
	.mmap = binder_mmap,
	.open = binder_open,
	.flush = binder_flush,
	.release = binder_release,
};

static int binder_mmap(struct file *filp, struct vm_area_struct *vma)
{
	struct binder_proc *proc = filp->private_data;

	if (proc->tsk != current->group_leader)
		return -EINVAL;

	binder_debug(BINDER_DEBUG_OPEN_CLOSE,
		     "%s: %d %lx-%lx (%ld K) vma %lx pagep %lx\n",
		     __func__, proc->pid, vma->vm_start, vma->vm_end,
		     (vma->vm_end - vma->vm_start) / SZ_1K, vma->vm_flags,
		     (unsigned long)pgprot_val(vma->vm_page_prot));

	if (vma->vm_flags & FORBIDDEN_MMAP_FLAGS) {
		pr_err("%s: %d %lx-%lx %s failed %d\n", __func__,
		       proc->pid, vma->vm_start, vma->vm_end, "bad vm_flags", -EPERM);
		return -EPERM;
	}
	vma->vm_flags |= VM_DONTCOPY | VM_MIXEDMAP;
	vma->vm_flags &= ~VM_MAYWRITE;

	vma->vm_ops = &binder_vm_ops;
	vma->vm_private_data = proc;

	return binder_alloc_mmap_handler(&proc->alloc, vma);
}
```

这里出现了结构体 struct vm_area_struct，需要标注一下，因为关于 linux 下虚拟内存的的管理就是靠的这个结构体，这里就不翻译了，大家可以看英文注释。  
对内而言，是一段连续的虚拟内存，通过 vm_start - vm_end 标记。  
整体而言，可以通过 vm_next、vm_prev 以链表形式串起来，也可以通过 vm_rb 以红黑树形式串起来。  
struct vm_area_struct 用于描述用户态的虚拟地址空间，而 struct vm_struct 用于描述内核态的虚拟地址空间，struct page 用于描述物理内存页。  

```c++
/*
 * This struct describes a virtual memory area. There is one of these
 * per VM-area/task. A VM area is any part of the process virtual memory
 * space that has a special rule for the page-fault handlers (ie a shared
 * library, the executable area etc).
 */
struct vm_area_struct {
	/* The first cache line has the info for VMA tree walking. */

	unsigned long vm_start;		/* Our start address within vm_mm. */
	unsigned long vm_end;		/* The first byte after our end address within vm_mm. */

	/* linked list of VM areas per task, sorted by address */
	struct vm_area_struct *vm_next, *vm_prev;

	struct rb_node vm_rb;

	/*
	 * Largest free memory gap in bytes to the left of this VMA.
	 * Either between this VMA and vma->vm_prev, or between one of the
	 * VMAs below us in the VMA rbtree and its ->vm_prev. This helps
	 * get_unmapped_area find a free area of the right size.
	 */
	unsigned long rb_subtree_gap;

	/* Second cache line starts here. */

	struct mm_struct *vm_mm;	/* The address space we belong to. */

	/*
	 * Access permissions of this VMA.
	 * See vmf_insert_mixed_prot() for discussion.
	 */
	pgprot_t vm_page_prot;
	unsigned long vm_flags;		/* Flags, see mm.h. */

	/*
	 * For areas with an address space and backing store,
	 * linkage into the address_space->i_mmap interval tree.
	 *
	 * For private anonymous mappings, a pointer to a null terminated string
	 * containing the name given to the vma, or NULL if unnamed.
	 */

	union {
		struct {
			struct rb_node rb;
			unsigned long rb_subtree_last;
		} shared;
		/*
		 * Serialized by mmap_sem. Never use directly because it is
		 * valid only when vm_file is NULL. Use anon_vma_name instead.
		 */
		struct anon_vma_name *anon_name;
	};

	/*
	 * A file's MAP_PRIVATE vma can be in both i_mmap tree and anon_vma
	 * list, after a COW of one of the file pages.	A MAP_SHARED vma
	 * can only be in the i_mmap tree.  An anonymous MAP_PRIVATE, stack
	 * or brk vma (with NULL file) can only be in an anon_vma list.
	 */
	struct list_head anon_vma_chain; /* Serialized by mmap_lock & page_table_lock */
	struct anon_vma *anon_vma;	/* Serialized by page_table_lock */

	/* Function pointers to deal with this struct. */
	const struct vm_operations_struct *vm_ops;

	/* Information about our backing store: */
	unsigned long vm_pgoff;		/* Offset (within vm_file) in PAGE_SIZE units */
	struct file * vm_file;		/* File we map to (can be NULL). */
	void * vm_private_data;		/* was vm_pte (shared mem) */

#ifdef CONFIG_SWAP
	atomic_long_t swap_readahead_info;
#endif
#ifndef CONFIG_MMU
	struct vm_region *vm_region;	/* NOMMU mapping region */
#endif
#ifdef CONFIG_NUMA
	struct mempolicy *vm_policy;	/* NUMA policy for the VMA */
#endif
	struct vm_userfaultfd_ctx vm_userfaultfd_ctx;
#ifdef CONFIG_SPECULATIVE_PAGE_FAULT
	seqcount_t vm_sequence;
	atomic_t vm_ref_count;		/* see vma_get(), vma_put() */
#endif

	ANDROID_KABI_RESERVE(1);
	ANDROID_KABI_RESERVE(2);
	ANDROID_KABI_RESERVE(3);
	ANDROID_KABI_RESERVE(4);
} __randomize_layout;
```

```c++
int binder_alloc_mmap_handler(struct binder_alloc *alloc, struct vm_area_struct *vma)
{
	int ret;
	const char *failure_string;
	struct binder_buffer *buffer;

	mutex_lock(&binder_alloc_mmap_lock);
	if (alloc->buffer_size) {
		ret = -EBUSY;
		failure_string = "already mapped";
		goto err_already_mapped;
	}
	alloc->buffer_size = min_t(unsigned long, vma->vm_end - vma->vm_start, SZ_4M);
	mutex_unlock(&binder_alloc_mmap_lock);

	alloc->buffer = (void __user *)vma->vm_start;

	alloc->pages = kcalloc(alloc->buffer_size / PAGE_SIZE,
			       sizeof(alloc->pages[0]),
			       GFP_KERNEL);
	if (alloc->pages == NULL) {
		ret = -ENOMEM;
		failure_string = "alloc page array";
		goto err_alloc_pages_failed;
	}

	buffer = kzalloc(sizeof(*buffer), GFP_KERNEL);
	if (!buffer) {
		ret = -ENOMEM;
		failure_string = "alloc buffer struct";
		goto err_alloc_buf_struct_failed;
	}

	buffer->user_data = alloc->buffer;
	list_add(&buffer->entry, &alloc->buffers);
	buffer->free = 1;
	binder_insert_free_buffer(alloc, buffer);
	alloc->free_async_space = alloc->buffer_size / 2;
	binder_alloc_set_vma(alloc, vma);
	mmgrab(alloc->vma_vm_mm);

	return 0;

err_alloc_buf_struct_failed:
	kfree(alloc->pages);
	alloc->pages = NULL;
err_alloc_pages_failed:
	alloc->buffer = NULL;
	mutex_lock(&binder_alloc_mmap_lock);
	alloc->buffer_size = 0;
err_already_mapped:
	mutex_unlock(&binder_alloc_mmap_lock);
	binder_alloc_debug(BINDER_DEBUG_USER_ERROR,
			   "%s: %d %lx-%lx %s failed %d\n", __func__,
			   alloc->pid, vma->vm_start, vma->vm_end,
			   failure_string, ret);
	return ret;
}
```

```c++
static inline void *kcalloc(size_t n, size_t size, gfp_t flags)
{
	return kmalloc_array(n, size, flags | __GFP_ZERO);
}

static inline void *kzalloc(size_t size, gfp_t flags)
{
	return kmalloc(size, flags | __GFP_ZERO);
}
```

这里可以看到，使用的是 kcalloc、kzalloc，再往下我就不追了，这两个都是基于 [slob](https://en.wikipedia.org/wiki/SLOB) (simple list of block) 的方式分配物理内存，其实看到很多文章中说 mmap 是用于映射文件到到内存，所以有效率问题。这么说在 Android 平台中，一定是错的。由这里的代码也可以看出，逻辑还是申请了一段物理内存，然后分别映射到用户空间，这也是为什么有所谓“一次写入”的特性。



### broadcastIntentWithFeature

3. 通过查看 broadcastIntentWithFeature 函数可以知道，broadcastIntentWithFeature -> broadcastIntentLocked -> broadcastIntentLockedTraced -> enqueueBroadcastLocked(BroadcastQueue 的实现在 BroadcastQueueImpl.java 中)



## 其他

shmget vs mmap，这里需要说明的一点，严格意义上来说，mmap 是 memory map，翻译过来是内存映射，而共享内存是 shmem，实际是并不相同的。只不过 shmem 族的 api 是 System V 规范的，而 mmap 是 POSIX 规范的。shmem 更老支持也更广泛一点，mmap 会更简单一点。  
binder 不太适合数据量大的通讯，因为需要 Parcel  
shmget 没有提供同步机制  
binder(mmap)通过 mutex_lock 提供同步机制