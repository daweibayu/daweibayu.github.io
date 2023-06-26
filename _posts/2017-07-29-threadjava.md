---
layout: post
title:  "线程的本质（jdk 层实现）"
author: "daweibayu"
tags: Android
excerpt_separator: <!--more-->
---

<!--more-->




关于上下文请看 [线程本质](/2017-04-26/threadintro)，下面我们直接进入正题。

## 使用
先看看我们平时使用的方式，使用线程常见的代码如下：
```java
Thread thread = new Thread(new Runnable() {
      @Override
      public void run() {
        while (true) {
          // 哒哒哒哒
        }
      }
    });
thread.start();
```
然后我们知道里边的代码不会阻塞当前的执行流，而是新开一个异步线程去执行 while 循环，但是，为什么呢？
为什么就不会阻塞 UI 线程？
以及它到底是如何调度系统资源的？
或者直接一点，这货到底是怎么实现的？

## 代码
带着上边的问题，我们先来看一下 java 端的代码[Thread.java](https://cs.android.com/android/platform/superproject/+/master:libcore/ojluni/src/main/java/java/lang/Thread.java?q=Thread.java)，Java 层的线程实现自然是最简单的，搞 Android 或者 java 的对这个应该也很熟悉。这里我们简要介绍一下：
为了方便，我们先把 Runnable 贴一下，因为 `Thread implements Runnable`。


```java
// 就是一个接口定义了一个 run 方法，这里不多说
public interface Runnable {
	public abstract void run();
}
```

接下来是 [Thread.java](https://cs.android.com/android/platform/superproject/+/android-13.0.0_r29:libcore/ojluni/src/main/java/java/lang/Thread.java) 的源码

```java
// 注意不要找到源码中其他的 Thread.java
package java.lang;

public class Thread implements Runnable {

    private final Object lock = new Object();

    private volatile long nativePeer;
    private volatile String name;
    private boolean     single_step;
    private boolean daemon = false;
    private boolean stillborn = false;
    private long eetop;
    private Runnable target;
    private ThreadGroup group;
    private ClassLoader contextClassLoader;
    private AccessControlContext inheritedAccessControlContext;
    private final long stackSize;
    private boolean unparkedBeforeStart;
    private boolean systemDaemon = false;
    boolean started = false;
    volatile Object parkBlocker;
    private volatile Interruptible blocker;
    private final Object blockerLock = new Object();

    // ThreadLocal，主要用来存储线程独有的变量，如果想了解的话可以参考 [ThreadLocal 框架](2017-05-27/threadlocal)
    ThreadLocal.ThreadLocalMap threadLocals = null;
    ThreadLocal.ThreadLocalMap inheritableThreadLocals = null;

    // 在 ThreadLocalRandom 中通过 Unsafe.objectFieldOffset 使用
    @jdk.internal.vm.annotation.Contended("tlr")
    long threadLocalRandomSeed;
    @jdk.internal.vm.annotation.Contended("tlr")
    int threadLocalRandomProbe;
    @jdk.internal.vm.annotation.Contended("tlr")
    int threadLocalRandomSecondarySeed;
  
    // 用于匿名的线程命名
    private static int threadInitNumber;
    private static synchronized int nextThreadNum() {
        return threadInitNumber++;
    }

    // 线程 id，通过 threadSeqNumber 的累加来赋值 tid（与 native 层的线程 id 没有一毛钱关系）
    private final long tid;
    private static long threadSeqNumber;
    public long getId() {
        return tid;
    }
    private static synchronized long nextThreadID() {
        return ++threadSeqNumber;
    }

    // 线程优先级，不多说，看代码就知道
    private int priority;
    public static final int MIN_PRIORITY = 1;
    public static final int NORM_PRIORITY = 5;
    public static final int MAX_PRIORITY = 10;

    public final int getPriority() {
        return priority;
    }

    public final void setPriority(int newPriority) {
      ...
      synchronized(this) {
          this.priority = newPriority;
          if (isAlive()) {
              // 调用 native 函数将优先级同步到 native
              setPriority0(newPriority);
          }
      }
    }


    // 构造函数
    private Thread(ThreadGroup g, Runnable target, String name,
                   long stackSize, AccessControlContext acc,
                   boolean inheritThreadLocals) {
      ...
      this.name = name;

      // 获取父线程，并给当前的 Thread copy 赋值
      Thread parent = currentThread();
      this.priority = parent.getPriority();
      ...
    }

    public synchronized void start() {
        ...
        try {
            // native 线程创建
            nativeCreate(this, stackSize, daemon);
            started = true;
        } finally {
            ...
        }
    }

    @Override
    public void run() {
        if (target != null) {
            target.run();
        }
    }

    private void exit() { ... }
    public void interrupt() { ... }
    public final void join(long millis, int nanos) throws InterruptedException { ... }
    public static void sleep(long millis, int nanos) throws InterruptedException { ... }
    public final boolean isAlive() {
        return nativePeer != 0;
    }

    public String toString() {
        ThreadGroup group = getThreadGroup();
        if (group != null) {
            return "Thread[" + getName() + "," + getPriority() + "," +
                           group.getName() + "]";
        } else {
            return "Thread[" + getName() + "," + getPriority() + "," +
                            "" + "]";
        }
    }


    // 线程栈输出相关
    private static final StackTraceElement[] EMPTY_STACK_TRACE = new StackTraceElement[0];
    public StackTraceElement[] getStackTrace() {
        StackTraceElement ste[] = VMStack.getThreadStackTrace(this);
        return ste != null ? ste : EmptyArray.STACK_TRACE_ELEMENT;
    }
    public static Map<Thread, StackTraceElement[]> getAllStackTraces() {

        int count = ThreadGroup.systemThreadGroup.activeCount();
        Thread[] threads = new Thread[count + count / 2];

        // Enumerate the threads.
        count = ThreadGroup.systemThreadGroup.enumerate(threads);

        // Collect the stacktraces
        Map<Thread, StackTraceElement[]> m = new HashMap<Thread, StackTraceElement[]>();
        for (int i = 0; i < count; i++) {
            StackTraceElement[] stackTrace = threads[i].getStackTrace();
            m.put(threads[i], stackTrace);
        }
        // END Android-changed: Use ThreadGroup and getStackTrace() instead of native methods.
        return m;
    }

    // 各种线程状态
    public enum State {
        NEW,
        RUNNABLE,
        BLOCKED,
        WAITING,
        TIMED_WAITING,
        TERMINATED;
    }


    public State getState() {
        return State.values()[nativeGetStatus(started)];
    }


    // 从这里往下，是错误处理相关
    @FunctionalInterface
    public interface UncaughtExceptionHandler {
        void uncaughtException(Thread t, Throwable e);
    }

    private volatile UncaughtExceptionHandler uncaughtExceptionHandler;

    private static volatile UncaughtExceptionHandler defaultUncaughtExceptionHandler;


    public static void setDefaultUncaughtExceptionHandler(UncaughtExceptionHandler eh) {
         defaultUncaughtExceptionHandler = eh;
     }

    public static UncaughtExceptionHandler getDefaultUncaughtExceptionHandler(){
        return defaultUncaughtExceptionHandler;
    }

    private static volatile UncaughtExceptionHandler uncaughtExceptionPreHandler;

    public static void setUncaughtExceptionPreHandler(UncaughtExceptionHandler eh) {
        uncaughtExceptionPreHandler = eh;
    }

    public static UncaughtExceptionHandler getUncaughtExceptionPreHandler() {
        return uncaughtExceptionPreHandler;
    }

    public UncaughtExceptionHandler getUncaughtExceptionHandler() {
        return uncaughtExceptionHandler != null ? uncaughtExceptionHandler : group;
    }

    public void setUncaughtExceptionHandler(UncaughtExceptionHandler eh) {
        checkAccess();
        uncaughtExceptionHandler = eh;
    }

    public final void dispatchUncaughtException(Throwable e) {
        Thread.UncaughtExceptionHandler initialUeh = Thread.getUncaughtExceptionPreHandler();
        if (initialUeh != null) {
            try {
                initialUeh.uncaughtException(this, e);
            } catch (RuntimeException | Error ignored) {
            }
        }
        getUncaughtExceptionHandler().uncaughtException(this, e);
    }


    // 为了方便梳理，所有 native 函数我都挪到了一起，对个别函数感兴趣的推荐去看源码中的注释
    @FastNative
    private static native void sleep(Object lock, long millis, int nanos) throws InterruptedException;
    private native static void nativeCreate(Thread t, long stackSize, boolean daemon);
    @FastNative
    public static native boolean interrupted();
    @FastNative
    public native boolean isInterrupted();
    public static native boolean holdsLock(Object obj);
    @HotSpotIntrinsicCandidate
    @FastNative
    public static native Thread currentThread();
    public static native void yield();
    private native void setPriority0(int newPriority);
    @FastNative
    private native void interrupt0();
    private native void setNativeName(String name);
    private native int nativeGetStatus(boolean hasBeenStarted);
}

```


## 总结

1. 其实 Java 层的线程就是一层皮，除 ThreadLocal 用于存储 java 层线程相关数据外，并没有太多实质性的逻辑
2. 线程 id 与 native 真实线程没有关系，只是累加所得
3. 线程 name 也与 native 无关，除了调用方参数传入，也是同 id 一样，是 java 层累加所得
4. 线程构造函数中有 currentThread native 调用，但因为是获取的“当前”线程，所以其实与本身线程资源的申请无关，具体实现下文再说
5. 核心逻辑其实是在 start 中的 nativeCreate

所以下文我们开启 native 层的线程逻辑
