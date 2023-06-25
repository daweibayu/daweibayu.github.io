---
layout: post
title:  "线程的本质（jdk 层实现）"
author: "daweibayu"
tags: Android
excerpt_separator: <!--more-->
---

<!--more-->

[Thread.java](https://cs.android.com/android/platform/superproject/+/android-13.0.0_r29:libcore/ojluni/src/main/java/java/lang/Thread.java)


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
带着上边的问题，我们先来看一下 java 端的代码[Thread.java](http://hg.openjdk.java.net/jdk6/jdk6/jdk/file/5672a2be515a/src/share/classes/java/lang/Thread.java)，Java 层的线程实现自然是最简单的，搞 Android 或者 java 的对这个应该也很熟悉。这里我们简要介绍一下：
为了方便，我们先把 Runnable 贴一下，因为 `Thread implements Runnable`。

```java
// 就是一个接口定义了一个 run 方法，这里不多说
public interface Runnable {
	public abstract void run();
}
```
然后我们在把 Thread 的几个主要属性及函数列一下（不关主线的代码就都删掉了）：
```java
package java.lang;

public class Thread implements Runnable {

  // 先执行 native 函数来做一些注册相关的东西
  private static native void registerNatives();
  static {
    registerNatives();
  }

  private Runnable target;

  // 优先级
  public final static int MIN_PRIORITY = 1;
  public final static int NORM_PRIORITY = 5;
  public final static int MAX_PRIORITY = 10;
  private int         priority;

  public final void setPriority(int newPriority) {
    ...
    setPriority0(priority = newPriority);
  }

  public final int getPriority() {
    return priority;
  }



  // 以下几个变量会在 native 层 classloader 时被调用，这里不做过多解释（这里其实也是应该删掉的）
  private Thread      threadQ;
  private long        eetop;
  private boolean     single_step;
  private boolean     stillborn = false;
  private long nativeParkEventPointer;



  // ThreadGroup 相关
  private ThreadGroup group;

  public final ThreadGroup getThreadGroup() {
    return group;
  }

  public static int activeCount() {
    return currentThread().getThreadGroup().activeCount();
  }

  public static int enumerate(Thread tarray[]) {
    return currentThread().getThreadGroup().enumerate(tarray);
  }



  // name 相关（就是日志中打印出来的线程名字），当线程初始化时，会根据 threadInitNumber 来自动生成线程名字
  private char        name[];
  private static int threadInitNumber;

  private static synchronized int nextThreadNum() {
    return threadInitNumber++;
  }

  public final void setName(String name) {
    ...
    this.name = name.toCharArray();
  }

  public final String getName() {
    return String.valueOf(name);
  }



  // tid 相关（线程 id），通过 threadSeqNumber 的累加来赋值 tid（与 native 层的线程 id 没有一毛钱关系）
  private long tid;
  private static long threadSeqNumber;

  private static synchronized long nextThreadID() {
    return ++threadSeqNumber;
  }

  public long getId() {
    return tid;
  }



  // ThreadLocal，主要用来存储线程独有的变量，如果想了解的话可以参考 [ThreadLocal 框架](http://www.jianshu.com/p/e34ec28bf7a4)
  ThreadLocal.ThreadLocalMap threadLocals = null;
  ThreadLocal.ThreadLocalMap inheritableThreadLocals = null;



  //线程状态
  private int threadStatus = 0;

  public enum State {
    NEW,
    RUNNABLE,
    BLOCKED,
    WAITING,
    TIMED_WAITING,
    TERMINATED;
  }

  public State getState() {
    return sun.misc.VM.toThreadState(threadStatus);
  }


  // 线程休眠
  public static void sleep(long millis, int nanos) throws InterruptedException {
    。。。
    sleep(millis);
  }

  // 初始化
  private void init(ThreadGroup g, Runnable target, String name,
                    long stackSize) {
    Thread parent = currentThread();
    ...
    this.group = g;
    this.priority = parent.getPriority();
    this.name = name.toCharArray();
    this.target = target;
    setPriority(priority);
    ....
    tid = nextThreadID();
  }

  // 构造函数，有多个，为了方便展示框架，这里只贴了一个
  public Thread(Runnable target) {
    init(null, target, "Thread-" + nextThreadNum(), 0);
  }

  // 开始真正执行此线程
  public synchronized void start() {
    ...
    start0();
    ...
  }

  // 没什么好说的，直接执行了 Runnable 的 run 函数
  public void run() {
    if (target != null) {
      target.run();
    }
  }

  // 判断当前线程是否中断
  public void interrupt() {
    ...
    interrupt0();
  }
  
  public static boolean interrupted() {
    return currentThread().isInterrupted(true);
  }

  public boolean isInterrupted() {
    return isInterrupted(false);
  }

  // 阻塞调用此函数的线程，直到 join 归属的 Thread 实例执行完。核心就是执行了 wait，而 wait 这个函数其实也是 native 实现
  public final synchronized void join(long millis) throws InterruptedException {
    long base = System.currentTimeMillis();
    long now = 0;

    ...
    if (millis == 0) {
      while (isAlive()) {
        wait(0);
      }
    } else {
      while (isAlive()) {
        long delay = millis - now;
        if (delay <= 0) {
          break;
        }
        wait(delay);
        now = System.currentTimeMillis() - base;
      }
    }
  }


  // native 相关函数
  public static native boolean holdsLock(Object obj);
  private native static StackTraceElement[][] dumpThreads(Thread[] threads);
  private native static Thread[] getThreads();

  public static native Thread currentThread();
  public static native void yield();
  public static native void sleep(long millis) throws InterruptedException;

  private native boolean isInterrupted(boolean ClearInterrupted);
  public final native boolean isAlive();
  private native void start0();
  private native void setPriority0(int newPriority);
  private native void stop0(Object o);
  private native void suspend0();
  private native void resume0();
  private native void interrupt0();
}
```

## 总结
不多说，其实 Java 层的线程就是一层皮，没有什么实质性的逻辑，没有任何申请内存等相关的逻辑，就连 `tid` 都是根据 `++threadInitNumber` 叠加出来的，与 native 层的完全没有关系。
我们发现我们上边的疑惑仍然没有得到解答，那我们接下来就看一下 native 层的实现。

