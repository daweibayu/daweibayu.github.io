---
layout: post
title:  "线程的本质（art 层实现）"
author: "daweibayu"
tags: Android
excerpt_separator: <!--more-->
---
<!--more-->

## 名词解释

ojluni：OpenJDK、 java.lang 、 java.util 、 java.net 、 java.io 的缩写。This relates to "luni" in Android source which stands for lang util net io。

Bionic：Bionic库是Android的基础库之一，也是连接Android和Linux的桥梁。Bionic库中包含了很多基本系统功能接口，这些功能大部分来自 Linux，但是和标准的 Linux 之间有很多细微差别。

## 源代码来源

我们都知道 Android 在 5.0 后切换到 art ([5.0 behavior change](https://developer.android.com/about/versions/lollipop/android-5.0-changes))，所以搜索相关代码时需要在 art 目录下寻找，避免找错。

* [runtime.cc](https://cs.android.com/android/platform/superproject/+/refs/heads/master:art/runtime/runtime.cc)
* [native_util.h](https://cs.android.com/android/platform/superproject/+/refs/heads/master:art/runtime/native/native_util.h)
* [java_long_thread.cc](https://cs.android.com/android/platform/superproject/+/master:art/runtime/native/java_lang_Thread.cc)
* [thread.cc](https://cs.android.com/android/platform/superproject/+/master:art/runtime/thread.cc)
* [pthread.cpp](https://cs.android.com/android/platform/superproject/+/refs/heads/master:bionic/libc/bionic/pthread_create.cpp)

关于 Android 源码的下载，可以直接看 [官方](https://source.android.com/docs/setup/download/downloading) 也可以参考 [MacOS 下载 Android 源码](/2018-09-03/androidSrcDownloadInMac)

## 主题

上文提到的关键 native 调用是 nativeCreate，我们就由此展开


### 先找到对应的 jni 调用

#### jni 注册流程

```c++
// 代码位于 runtime.cc
bool Runtime::Start() {
  ...
  Thread* self = Thread::Current();
  RegisterRuntimeNativeMethods(self->GetJniEnv());
  ...
}

// 代码位于 runtime.cc
void Runtime::RegisterRuntimeNativeMethods(JNIEnv* env) {
  ...
  register_java_lang_Thread(env);
  ...
}

//代码位于 java_long_thread.cc
void register_java_lang_Thread(JNIEnv* env) {
  REGISTER_NATIVE_METHODS("java/lang/Thread");
}


//代码位于 native_util.h
#define REGISTER_NATIVE_METHODS(jni_class_name) RegisterNativeMethodsInternal(env, (jni_class_name), gMethods, arraysize(gMethods))


//代码位于 native_util.h
ALWAYS_INLINE inline void RegisterNativeMethodsInternal(JNIEnv* env,
                                                        const char* jni_class_name,
                                                        const JNINativeMethod* methods,
                                                        jint method_count) {
  ScopedLocalRef<jclass> c(env, env->FindClass(jni_class_name));
  if (c.get() == nullptr) {
    LOG(FATAL) << "Couldn't find class: " << jni_class_name;
  }
  jint jni_result = env->RegisterNatives(c.get(), methods, method_count);
  CHECK_EQ(JNI_OK, jni_result);
}

```

#### 具体 jni 实现
其中的关键
```c++
namespace art {

static jobject Thread_currentThread(JNIEnv* env, jclass) {
  ScopedFastNativeObjectAccess soa(env);
  return soa.AddLocalReference<jobject>(soa.Self()->GetPeer());
}

static jboolean Thread_interrupted(JNIEnv* env, jclass) { ... }

static jboolean Thread_isInterrupted(JNIEnv* env, jobject java_thread) {
  ScopedFastNativeObjectAccess soa(env);
  MutexLock mu(soa.Self(), *Locks::thread_list_lock_);
  Thread* thread = Thread::FromManagedThread(soa, java_thread);
  return (thread != nullptr) ? thread->IsInterrupted() : JNI_FALSE;
}

static void Thread_nativeCreate(JNIEnv* env, jclass, jobject java_thread, jlong stack_size,
                                jboolean daemon) {
  Runtime* runtime = Runtime::Current();
  if (runtime->IsZygote() && runtime->IsZygoteNoThreadSection()) {
    jclass internal_error = env->FindClass("java/lang/InternalError");
    CHECK(internal_error != nullptr);
    env->ThrowNew(internal_error, "Cannot create threads in zygote");
    return;
  }

  Thread::CreateNativeThread(env, java_thread, stack_size, daemon == JNI_TRUE);
}

static jint Thread_nativeGetStatus(JNIEnv* env, jobject java_thread, jboolean has_been_started) { ... }

static jboolean Thread_holdsLock(JNIEnv* env, jclass, jobject java_object) { ... }

static void Thread_interrupt0(JNIEnv* env, jobject java_thread) { ... }

static void Thread_setNativeName(JNIEnv* env, jobject peer, jstring java_name) { ... }


static void Thread_setPriority0(JNIEnv* env, jobject java_thread, jint new_priority) {
  ScopedObjectAccess soa(env);
  MutexLock mu(soa.Self(), *Locks::thread_list_lock_);
  Thread* thread = Thread::FromManagedThread(soa, java_thread);
  if (thread != nullptr) {
    thread->SetNativePriority(new_priority);
  }
}

static void Thread_sleep(JNIEnv* env, jclass, jobject java_lock, jlong ms, jint ns) {
  ScopedFastNativeObjectAccess soa(env);
  ObjPtr<mirror::Object> lock = soa.Decode<mirror::Object>(java_lock);
  Monitor::Wait(Thread::Current(), lock.Ptr(), ms, ns, true, ThreadState::kSleeping);
}


static void Thread_yield(JNIEnv*, jobject) {
  sched_yield();
}

static JNINativeMethod gMethods[] = {
  FAST_NATIVE_METHOD(Thread, currentThread, "()Ljava/lang/Thread;"),
  FAST_NATIVE_METHOD(Thread, interrupted, "()Z"),
  FAST_NATIVE_METHOD(Thread, isInterrupted, "()Z"),
  NATIVE_METHOD(Thread, nativeCreate, "(Ljava/lang/Thread;JZ)V"),
  NATIVE_METHOD(Thread, nativeGetStatus, "(Z)I"),
  NATIVE_METHOD(Thread, holdsLock, "(Ljava/lang/Object;)Z"),
  FAST_NATIVE_METHOD(Thread, interrupt0, "()V"),
  NATIVE_METHOD(Thread, setNativeName, "(Ljava/lang/String;)V"),
  NATIVE_METHOD(Thread, setPriority0, "(I)V"),
  FAST_NATIVE_METHOD(Thread, sleep, "(Ljava/lang/Object;JI)V"),
  NATIVE_METHOD(Thread, yield, "()V"),
};

void register_java_lang_Thread(JNIEnv* env) {
  REGISTER_NATIVE_METHODS("java/lang/Thread");
}

}
```

### art 线程实现

由上可以看到 Thread_nativeCreate 调用的是 Thread::CreateNativeThread，而此 Thread 的代码位于 [thread.cc](https://cs.android.com/android/platform/superproject/+/master:art/runtime/thread.cc)。

```c++
void Thread::CreateNativeThread(JNIEnv* env, jobject java_peer, size_t stack_size, bool is_daemon) {
  ...
  Runtime* runtime = Runtime::Current();

  ...
  Thread* child_thread = new Thread(is_daemon);
  
  ...
  SetNativePeer(env, java_peer, child_thread);

  int pthread_create_result = 0;
  if (child_jni_env_ext.get() != nullptr) {
    pthread_t new_pthread;
    pthread_attr_t attr;

    ...
    pthread_create_result = pthread_create(&new_pthread, &attr, gUseUserfaultfd ? Thread::CreateCallbackWithUffdGc : Thread::CreateCallback,child_thread);
    ...
  }
  ...
}
```

由上可以看出， art 中的 thread.cc 角色跟 jdk 中的 Thread.java 其实差不多，都是包装层。
这么说或许也有些不准确，应该是 pthread 是  art thread 中的一个成员，具体线程相关操作由 pthread 去执行。
而具体线程资源的申请由 pthread_create 实现。

### pthread

```c++
__BIONIC_WEAK_FOR_NATIVE_BRIDGE
int pthread_create(pthread_t* thread_out, pthread_attr_t const* attr, void* (*start_routine)(void*), void* arg) {

  pthread_attr_t thread_attr;

  if (attr == nullptr) {
    pthread_attr_init(&thread_attr);
  } else {
    thread_attr = *attr;
    attr = nullptr;
  }

  bionic_tcb* tcb = nullptr;
  void* child_stack = nullptr;
  int result = __allocate_thread(&thread_attr, &tcb, &child_stack);
  if (result != 0) {
    return result;
  }

  pthread_internal_t* thread = tcb->thread();

  ...

    int rc = clone(__pthread_start, child_stack, flags, thread, &(thread->tid), tls, &(thread->tid));
  __rt_sigprocmask(SIG_SETMASK, &thread->start_mask, nullptr, sizeof(thread->start_mask));

  ...

  int init_errno = __init_thread(thread);
  ...
}
```

pthread_create 中的 __allocate_thread 函数去创建线程栈等
```c++
static int __allocate_thread(pthread_attr_t* attr, bionic_tcb** tcbp, void** child_stack) {
  ThreadMapping mapping;
  char* stack_top;
  bool stack_clean = false;

  if (attr->stack_base == nullptr) {
    // The caller didn't provide a stack, so allocate one.

    // Make sure the guard size is a multiple of page_size().
    const size_t unaligned_guard_size = attr->guard_size;
    attr->guard_size = __BIONIC_ALIGN(attr->guard_size, page_size());
    if (attr->guard_size < unaligned_guard_size) return EAGAIN;

    mapping = __allocate_thread_mapping(attr->stack_size, attr->guard_size);
    if (mapping.mmap_base == nullptr) return EAGAIN;

    stack_top = mapping.stack_top;
    attr->stack_base = mapping.stack_base;
    stack_clean = true;
  } else {
    mapping = __allocate_thread_mapping(0, PTHREAD_GUARD_SIZE);
    if (mapping.mmap_base == nullptr) return EAGAIN;

    stack_top = static_cast<char*>(attr->stack_base) + attr->stack_size;
  }

  // Carve out space from the stack for the thread's pthread_internal_t. This
  // memory isn't counted in pthread_attr_getstacksize.

  // To safely access the pthread_internal_t and thread stack, we need to find a 16-byte aligned boundary.
  stack_top = align_down(stack_top - sizeof(pthread_internal_t), 16);

  pthread_internal_t* thread = reinterpret_cast<pthread_internal_t*>(stack_top);
  if (!stack_clean) {
    // If thread was not allocated by mmap(), it may not have been cleared to zero.
    // So assume the worst and zero it.
    memset(thread, 0, sizeof(pthread_internal_t));
  }

  // Locate static TLS structures within the mapped region.
  const StaticTlsLayout& layout = __libc_shared_globals()->static_tls_layout;
  auto tcb = reinterpret_cast<bionic_tcb*>(mapping.static_tls + layout.offset_bionic_tcb());
  auto tls = reinterpret_cast<bionic_tls*>(mapping.static_tls + layout.offset_bionic_tls());

  // Initialize TLS memory.
  __init_static_tls(mapping.static_tls);
  __init_tcb(tcb, thread);
  __init_tcb_dtv(tcb);
  __init_tcb_stack_guard(tcb);
  __init_bionic_tls_ptrs(tcb, tls);

  attr->stack_size = stack_top - static_cast<char*>(attr->stack_base);
  thread->attr = *attr;
  thread->mmap_base = mapping.mmap_base;
  thread->mmap_size = mapping.mmap_size;
  thread->mmap_base_unguarded = mapping.mmap_base_unguarded;
  thread->mmap_size_unguarded = mapping.mmap_size_unguarded;
  thread->stack_top = reinterpret_cast<uintptr_t>(stack_top);

  *tcbp = tcb;
  *child_stack = stack_top;
  return 0;
}
```

pthread_create 中的 clone 去申请内核线程等资源，clone 的代码位于 [clone.cpp](https://cs.android.com/android/platform/superproject/+/refs/heads/master:bionic/libc/bionic/clone.cpp)
```c++
__BIONIC_WEAK_FOR_NATIVE_BRIDGE
int clone(int (*fn)(void*), void* child_stack, int flags, void* arg, ...) {
  int* parent_tid = nullptr;
  void* new_tls = nullptr;
  int* child_tid = nullptr;

  if (fn != nullptr && child_stack == nullptr) {
    errno = EINVAL;
    return -1;
  }

  // Extract any optional parameters required by the flags.
  va_list args;
  va_start(args, arg);
  if ((flags & (CLONE_PARENT_SETTID|CLONE_SETTLS|CLONE_CHILD_SETTID|CLONE_CHILD_CLEARTID)) != 0) {
    parent_tid = va_arg(args, int*);
  }
  if ((flags & (CLONE_SETTLS|CLONE_CHILD_SETTID|CLONE_CHILD_CLEARTID)) != 0) {
    new_tls = va_arg(args, void*);
  }
  if ((flags & (CLONE_CHILD_SETTID|CLONE_CHILD_CLEARTID)) != 0) {
    child_tid = va_arg(args, int*);
  }
  va_end(args);

  // Align 'child_stack' to 16 bytes.
  uintptr_t child_stack_addr = reinterpret_cast<uintptr_t>(child_stack);
  child_stack_addr &= ~0xf;
  child_stack = reinterpret_cast<void*>(child_stack_addr);

  // Remember the parent pid and invalidate the cached value while we clone.
  pthread_internal_t* self = __get_thread();
  pid_t parent_pid = self->invalidate_cached_pid();

  // Remmber the caller's tid so that it can be restored in the parent after clone.
  pid_t caller_tid = self->tid;
  // Invalidate the tid before the syscall. The value is lazily cached in gettid(),
  // and it will be updated by fork() and pthread_create(). We don't do this if
  // we are sharing address space with the child.
  if (!(flags & (CLONE_VM|CLONE_VFORK))) {
    self->tid = -1;
  }

  // Actually do the clone.
  int clone_result;
  if (fn != nullptr) {
    clone_result = __bionic_clone(flags, child_stack, parent_tid, new_tls, child_tid, fn, arg);
  } else {
#if defined(__x86_64__) // sys_clone's last two arguments are flipped on x86-64.
    clone_result = syscall(__NR_clone, flags, child_stack, parent_tid, child_tid, new_tls);
#else
    clone_result = syscall(__NR_clone, flags, child_stack, parent_tid, new_tls, child_tid);
#endif
  }

  if (clone_result != 0) {
    self->set_cached_pid(parent_pid);
    self->tid = caller_tid;
  } else if (self->tid == -1) {
    self->tid = syscall(__NR_gettid);
    self->set_cached_pid(self->tid);
  }

  return clone_result;
}

```

而这中的关键调用是 __bionic_clone，代码位于 [__bionic_clone.S](https://cs.android.com/android/platform/superproject/+/master:bionic/libc/arch-arm64/bionic/__bionic_clone.S)
```
ENTRY_PRIVATE(__bionic_clone)
    # Push 'fn' and 'arg' onto the child stack.
    stp     x5, x6, [x1, #-16]!

    # Make the system call.
    mov     x8, __NR_clone
    svc     #0

    # Are we the child?
    cbz     x0, .L_bc_child

    # Set errno if something went wrong.
    cmn     x0, #(MAX_ERRNO + 1)
    cneg    x0, x0, hi
    b.hi    __set_errno_internal

    ret

.L_bc_child:
    # We're in the child now. Set the end of the frame record chain.
    mov     x29, #0
    # Setting x30 to 0 will make the unwinder stop at __start_thread.
    mov     x30, #0
    # Call __start_thread with the 'fn' and 'arg' we stored on the child stack.
    ldp     x0, x1, [sp], #16
    b       __start_thread
END(__bionic_clone)
```

arm 64 下通过 svc 指令触发，然后就由用户态转到核态，具体看下一篇文章啦