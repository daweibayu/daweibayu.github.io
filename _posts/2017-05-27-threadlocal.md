---
layout: post
title:  "ThreadLocal"
author: "daweibayu"
tags: Android
excerpt_separator: <!--more-->
---

<!--more-->

这个东东大家可能平时会听说过，但是了解可能不是很多，当然网上也有了很多介绍文章，这里我只按照我的理解来说一下。知道这个东西应该挺多都是看 Looper 看到的。
我们先看看这个东西都是怎么用的：
```java
private final Handler uiThreadHandler = new Handler(Looper.getMainLooper());
ThreadLocal<Integer> threadLocal = new ThreadLocal<>();
int mainThreadNum = 1;int workerThreadNum = 2;
public void testThreadLocal() {
	threadLocal.set(mainThreadNum);
	new Thread(new Runnable() {
		@Override
		public void run() {
			threadLocal.set(workerThreadNum);
			uiThreadHandler.post(new Runnable() {
				@Override
				public void run() {
					Log.d("UIThread", "" + threadLocal.get());
				}
			});
			Log.d("WorkerThread", "" + threadLocal.get());
		}
	}).start();
	Log.d("UIThread", "" + threadLocal.get());
}
```
这里的输出
```
UIThread: 1
WorkerThread: 2
UIThread: 1
```

可见，ThreadLocal 的功能就是在不同 Thread set 进去的数据不会相互干扰并且可以直接获取到。

然后我们说一下这个东西是怎么实现的：

```java
public class ThreadLocal<T> {
	private final int threadLocalHashCode = nextHashCode();
	private static AtomicInteger nextHashCode = new AtomicInteger();
	public ThreadLocal() {  }
	...
	static class ThreadLocalMap {
		...
	}
}
```
不管别的，先看构造函数与成员变量。由上可知，可以随意构造实例，并且成员变量只有一个 threadLocalHashCode，所以很明显的可以知道这货就是个皮包公司，它并不存储任何实例，只是包含了一个这个实例的索引，也就是**threadLocalHashCode**。

那接下来我们在看看它的成员函数，这里只那 set 来举例：
```java
public void set(T value) {
	Thread t = Thread.currentThread();
	ThreadLocalMap map = getMap(t);
	if (map != null)
		map.set(this, value);
	else
		createMap(t, value);
}

ThreadLocalMap getMap(Thread t) {
	return t.threadLocals;
}

void createMap(Thread t, T firstValue) {
	t.threadLocals = new ThreadLocalMap(this, firstValue);
}
```
这里我们可能就比较清晰为什么 ThreadLocal 可以提供这种可以在不同线程存入/读取数据而互不干扰的功能了。
就是因为实际的变量并不是存入到了 ThreadLocal 中，而是存入到了这个操作对应的 Thread.threadLocals 中了（可以看上边的 getMap 函数）。而且这个 Thread.threadLocals 的实例化也是在 ThreadLocal 的代码中（createMap 函数）。

这个变量在 Thread 中的定义：
```java
public class Thread implements Runnable {
	...
	ThreadLocal.ThreadLocalMap threadLocals = null;
	...
}
```
就是一个引用而已，那现在就指向了 ThreadLocalMap 这个数据结构。

```java
static class ThreadLocalMap {
	static class Entry extends WeakReference<ThreadLocal> {
	Object value;
	Entry(ThreadLocal k, Object v) {
		super(k);
		value = v;
	}
}

private static final int INITIAL_CAPACITY = 16;
private Entry[] table;ThreadLocalMap(ThreadLocal firstKey, Object firstValue) {

	table = new Entry[INITIAL_CAPACITY];
	int i = firstKey.threadLocalHashCode & (INITIAL_CAPACITY - 1);
	table[i] = new Entry(firstKey, firstValue);
	...
}}
```
这里边最主要的其实就是这个数组 Entry[] table，这个 Entry 的定义也在上边了，其实就是一个包含了具体变量的 ThreadLocal 的弱引用。
而 threadLocalHashCode 这个变量就是计算具体变量在这个数组中的索引的。

ok，整个的东西已经串下来了。

当然，这里边疑问还有很多，比如 threadLocalHashCode 这个具体的计算方式，这个 table 数组是如何扩容等问题，大家可以自行看代码了。

