---
layout: post
title:  "AsyncTask"
author: "daweibayu"
tags: Android
excerpt_separator: <!--more-->
---

<!--more-->

面试的时候在问线程通讯的时候很多人都会说到 AsyncTask，但是问到具体实现原理的时候发现很多人都不清楚，甚至有很多都是工作好几年的，感觉也是挺有意思。这里大概介绍一下吧（怎么感觉这篇文章完全是在凑字数？）。

## 组成 & 目的

### 官网资料
先列一下官方资料：Android doc [AsyncTask](https://developer.android.com/reference/android/os/AsyncTask.html)，源代码位于 [AsyncTask.java](https://github.com/android/platform_frameworks_base/blob/master/core/java/android/os/AsyncTask.java)，七百行多一点的代码（排除注释后只有不到三百行，我写的其实大多都是废话，看源码最清晰了）。

### 组成
AsyncTask 主要包含两部分，**静态线程池 + handler**。
静态线程池：因为耗时操作肯定要在子线程中处理（不然就阻塞 UI 线程啦），静态线程池就是用来干这个的。
Handler：负责子线程与 UI 线程的通讯。

### 目的
这个类的设计目的就是可以让开发者只关心业务，因为 Android 暂时跳不出线程这个框框，还没有让开发者完全不关心线程问题的能力（简而言之就是还需要区分主线程与子线程），但是在开发者使用时又会产生关于线程处理的多余代码，所以 google 搞了一个 AsyncTask，主要的目的就是让用户只**聚焦自己业务**，线程的事情交由 AsyncTask 内部处理（只不过我是使用不惯这货）。

## 使用方式
基本使用方式如下（实例化 AsyncTask，然后 execute）：
```java
AsyncTask<String, Integer, Exception> asyncTask = new AsyncTask<String, Integer, Exception>() {

  @Override
  protected void onPreExecute() {
    super.onPreExecute();
  }

  @Override
  protected Exception doInBackground(String... params) {
    return null;
  }

  @Override
  protected void onProgressUpdate(Integer... values) {
  }

  @Override
  protected void onPostExecute(Exception e) {
  }

  @Override
  protected void onCancelled() {
  }
};
asyncTask.execute("");
```

这是 google 想让开发者关心的地方，总结就是**三个参数，五个回调**。

### 三个参数
参见上边的代码，三个参数主要是指<Params, Progress, Result>，对应上边的代码就是<String, Integer, Exception>(方便区分，专门用了三个不同的类型)，
Params 是传入的参数，即 doInBackground 中的 params。
Progress 传出参数，表示进度类型，即 onProgressUpdate 中的 values。
Exception 传出参数，表示返回值类型，即 onPostExecute 中的 e。

### 五个回调
五个回调也挨个说一下：
onPreExecute（主线程）：开始真正的任务（doInBackground）前会调用此回调，让开发者做一些准备工作。
doInBackground（子线程）：这就是真正的耗时任务了。
onProgressUpdate（主线程）：更新进度，当用户主动调用 publishProgress 后，AsyncTask 会通过 handler 通知到主线程，然后主线程调用 onProgressUpdate 来通知开发者。
onPostExecute（主线程）：耗时任务执行完以后，执行此回调。
onCancelled（主线程）：如果耗时任务被 cancel 的话，则调用此回调。

（发现写起来比预想的更难以解释，其实看源代码比看我解释清楚多了，源代码只有两百多行，为啥就这么多人不看呢）

好吧，我们还是来直接看代码吧（其实我是真不想这么干）。

## 源码（android-25）
```java
public abstract class AsyncTask<Params, Progress, Result> {
    
    // 顾名思义，CPU 的数量，主要是用来计算线程池默认包含线程的数量（最小值）
    private static final int CPU_COUNT = Runtime.getRuntime().availableProcessors();

    // 线程池包含线程的最小数量（当有新任务时，如果线程池中线程小于此值，则会创建新线程，即使其他已有线程处于闲置状态）
    private static final int CORE_POOL_SIZE = Math.max(2, Math.min(CPU_COUNT - 1, 4));

    // 线程池的最大数量（超了就只能等着现有的某个线程执行完了）
    private static final int MAXIMUM_POOL_SIZE = CPU_COUNT * 2 + 1;

    // 已经执行任务结束的线程的续命时间（过了就被回收了）
    private static final int KEEP_ALIVE_SECONDS = 30;

    // 顾名思义，线程工厂，就是用来生成新线程的
    private static final ThreadFactory sThreadFactory = new ThreadFactory() {
        private final AtomicInteger mCount = new AtomicInteger(1);

        public Thread newThread(Runnable r) {
            return new Thread(r, "AsyncTask #" + mCount.getAndIncrement());
        }
    };

    // 任务队列
    private static final BlockingQueue<Runnable> sPoolWorkQueue = new LinkedBlockingQueue<Runnable>(128);

    // ThreadPoolExecutor 的引用（关于 ThreadPoolExecutor，就是线程池，这里不做过多介绍）
    public static final Executor THREAD_POOL_EXECUTOR;

    // 线程池初始化
    static {
        ThreadPoolExecutor threadPoolExecutor = new ThreadPoolExecutor(
                CORE_POOL_SIZE, MAXIMUM_POOL_SIZE, KEEP_ALIVE_SECONDS, TimeUnit.SECONDS,
                sPoolWorkQueue, sThreadFactory);
        threadPoolExecutor.allowCoreThreadTimeOut(true);
        THREAD_POOL_EXECUTOR = threadPoolExecutor;
    }

    // 顾名思义，串行的执行器，至于其细节，请看 SerialExecutor 的注释
    public static final Executor SERIAL_EXECUTOR = new SerialExecutor();

    // 线程间通讯的标记，这个是用来标记传递的 Message 其中的内容为结果
    private static final int MESSAGE_POST_RESULT = 0x1;

    // 线程间通讯的标记，这个是用来标记传递的 Message 其中的内容为进度
    private static final int MESSAGE_POST_PROGRESS = 0x2;

    // 默认的执行器，默认值为串行，可通过 setDefaultExecutor 函数设置
    private static volatile Executor sDefaultExecutor = SERIAL_EXECUTOR;

    // AsyncTask 中全局的、单例的、主线程的 handler，这个 handler 主要用来子线程往主线程通讯使用
    private static InternalHandler sHandler;

    // 一个 WorkerRunnable 的实例（成员变量，具体 WorkerRunnable 是干嘛的，可以看 WorkerRunnable 的注释）
    private final WorkerRunnable<Params, Result> mWorker;

    // 一个 FutureTask 的实例（成员变量，同样，具体 FutureTask 是干嘛的请看 FutureTask 的注释）
    private final FutureTask<Result> mFuture;

    // 当前 AsyncTask 的状态
    private volatile Status mStatus = Status.PENDING;
    
    // 标记是否已经被取消了
    private final AtomicBoolean mCancelled = new AtomicBoolean();
    private final AtomicBoolean mTaskInvoked = new AtomicBoolean();

    // 这也是比较坑的东西，串行控制器，所以现在 AsyncTask 没有做其他设置的话，默认是串行的
    // 但是这货却只负责将具体的 Runnable 包装了一下，然后再扔给 THREAD_POOL_EXECUTOR，并且 THREAD_POOL_EXECUTOR 里边还有多个线程
    // 我是没悟透这其中的逻辑，所以我感觉这么写就是吃饱了撑的
    private static class SerialExecutor implements Executor {
        final ArrayDeque<Runnable> mTasks = new ArrayDeque<Runnable>();
        Runnable mActive;

        public synchronized void execute(final Runnable r) {
            mTasks.offer(new Runnable() {
                public void run() {
                    try {
                        r.run();
                    } finally {
                        scheduleNext();
                    }
                }
            });
            if (mActive == null) {
                scheduleNext();
            }
        }

        protected synchronized void scheduleNext() {
            if ((mActive = mTasks.poll()) != null) {
                THREAD_POOL_EXECUTOR.execute(mActive);
            }
        }
    }

    // 三种状态
    public enum Status {
        PENDING,
        RUNNING,
        FINISHED,
    }

    // 单例获取 Handler（主线程的 Handler）
    private static Handler getHandler() {
        synchronized (AsyncTask.class) {
            if (sHandler == null) {
                sHandler = new InternalHandler();
            }
            return sHandler;
        }
    }

    // 可以让开发者主动调用来设置执行器
    public static void setDefaultExecutor(Executor exec) {
        sDefaultExecutor = exec;
    }

    // 构造函数，而且注释明确写着，这个函数必须要在主线程中调用
    // 构造函数就干了两件事，初始化 mWorker 与 mFuture
    public AsyncTask() {
        mWorker = new WorkerRunnable<Params, Result>() {
            public Result call() throws Exception {
                mTaskInvoked.set(true);
                Result result = null;
                try {
                    Process.setThreadPriority(Process.THREAD_PRIORITY_BACKGROUND);
                    //noinspection unchecked
                    result = doInBackground(mParams);
                    Binder.flushPendingCommands();
                } catch (Throwable tr) {
                    mCancelled.set(true);
                    throw tr;
                } finally {
                    postResult(result);
                }
                return result;
            }
        };

        mFuture = new FutureTask<Result>(mWorker) {
            @Override
            protected void done() {
                try {
                    postResultIfNotInvoked(get());
                } catch (InterruptedException e) {
                    android.util.Log.w(LOG_TAG, e);
                } catch (ExecutionException e) {
                    throw new RuntimeException("An error occurred while executing doInBackground()",
                            e.getCause());
                } catch (CancellationException e) {
                    postResultIfNotInvoked(null);
                }
            }
        };
    }

    // 如果 mWorker call 没有被执行的话，这通过此函数讲结果返回给开发者
    private void postResultIfNotInvoked(Result result) {
        final boolean wasTaskInvoked = mTaskInvoked.get();
        if (!wasTaskInvoked) {
            postResult(result);
        }
    }

    // 当任务执行完成后，通过调用此函数将结果发送给主线程
    private Result postResult(Result result) {
        @SuppressWarnings("unchecked")
        Message message = getHandler().obtainMessage(MESSAGE_POST_RESULT,
                new AsyncTaskResult<Result>(this, result));
        message.sendToTarget();
        return result;
    }

    // 获取当前 AsyncTask 状态
    public final Status getStatus() {
        return mStatus;
    }

    // 需要开发者必须实现的任务执行函数，此函数在工作线程执行
    @WorkerThread
    protected abstract Result doInBackground(Params... params);

    // 在执行 doInBackground 前，给用户一个回调，让开发者做一些准备工作，此函数在主线程执行
    @MainThread
    protected void onPreExecute() {
    }

    // 回调接口，可以将 doInBackground 返回的结果通过此回调返回给开发者（其实主要还是一个跨线程的问题）
    @SuppressWarnings({"UnusedDeclaration"})
    @MainThread
    protected void onPostExecute(Result result) {
    }

    // 回调接口，将进度通过此回调返回给开发者，当开发者调用 publishProgress 才会执行此回调（主要也还是因为跨线程的问题，让 sHandler 来将运行在子线程的数据传送到主线程）
    @SuppressWarnings({"UnusedDeclaration"})
    @MainThread
    protected void onProgressUpdate(Progress... values) {
    }

    // 回调接口，当开发者主动调用 cancel 时会通过此回调通知开发者
    @SuppressWarnings({"UnusedParameters"})
    @MainThread
    protected void onCancelled(Result result) {
        onCancelled();
    }    
    
    // 回调接口，当开发者主动调用 cancel 时会通过此回调通知开发者
    @MainThread
    protected void onCancelled() {
    }

    // 获取当前 AsyncTask 是否被 cancel
    public final boolean isCancelled() {
        return mCancelled.get();
    }

    // 主动 cancel 当前执行的任务
    public final boolean cancel(boolean mayInterruptIfRunning) {
        mCancelled.set(true);
        return mFuture.cancel(mayInterruptIfRunning);
    }

    // 获取执行结果，注意这是一个同步函数，只有运算结束后，此函数才会继续执行
    public final Result get() throws InterruptedException, ExecutionException {
        return mFuture.get();
    }

    // 获取执行结果，这也是同步函数，当设置的时间后如果执行仍未结束，则通过 TimeoutException 来告诉开发者
    public final Result get(long timeout, TimeUnit unit) throws InterruptedException,
            ExecutionException, TimeoutException {
        return mFuture.get(timeout, unit);
    }

	// 开发者调用的执行函数
    @MainThread
    public final AsyncTask<Params, Progress, Result> execute(Params... params) {
        return executeOnExecutor(sDefaultExecutor, params);
    }

	// 开发者调用的执行函数
    @MainThread
    public final AsyncTask<Params, Progress, Result> executeOnExecutor(Executor exec,
            Params... params) {
        if (mStatus != Status.PENDING) {
            switch (mStatus) {
                case RUNNING:
                    throw new IllegalStateException("Cannot execute task:"
                            + " the task is already running.");
                case FINISHED:
                    throw new IllegalStateException("Cannot execute task:"
                            + " the task has already been executed "
                            + "(a task can be executed only once)");
            }
        }

        mStatus = Status.RUNNING;

        onPreExecute();

        mWorker.mParams = params;
        exec.execute(mFuture);

        return this;
    }

    // 开发者调用的执行函数，将 runnable 传给 sDefaultExecutor
    @MainThread
    public static void execute(Runnable runnable) {
        sDefaultExecutor.execute(runnable);
    }

    // 更新进度函数，会通过 InternalHandler 把进度传递给主线程
    @WorkerThread
    protected final void publishProgress(Progress... values) {
        if (!isCancelled()) {
            getHandler().obtainMessage(MESSAGE_POST_PROGRESS,
                    new AsyncTaskResult<Progress>(this, values)).sendToTarget();
        }
    }

    // 结束的时候调用此函数，然后调用回调通知开发者
    private void finish(Result result) {
        if (isCancelled()) {
            onCancelled(result);
        } else {
            onPostExecute(result);
        }
        mStatus = Status.FINISHED;
    }

    // 主线程的 Handler，用于工作线程与主线程之间的通讯
    private static class InternalHandler extends Handler {
        public InternalHandler() {
            super(Looper.getMainLooper());
        }

        @SuppressWarnings({"unchecked", "RawUseOfParameterizedType"})
        @Override
        public void handleMessage(Message msg) {
            AsyncTaskResult<?> result = (AsyncTaskResult<?>) msg.obj;
            switch (msg.what) {
                case MESSAGE_POST_RESULT:
                    // There is only one result
                    result.mTask.finish(result.mData[0]);
                    break;
                case MESSAGE_POST_PROGRESS:
                    result.mTask.onProgressUpdate(result.mData);
                    break;
            }
        }
    }

    // 虽然这货不是一个 Runnable，但是其实就是把一些逻辑封装到一起，可以让其他线程直接调用一个函数就可以执行了
    // 具体肯以看 AsyncTask 构造函数中 WorkerRunnable 的实例化代码，主要就是把 doInBackground 等函数包装了一下
    private static abstract class WorkerRunnable<Params, Result> implements Callable<Result> {
        Params[] mParams;
    }

    @SuppressWarnings({"RawUseOfParameterizedType"})
    private static class AsyncTaskResult<Data> {
        final AsyncTask mTask;
        final Data[] mData;

        AsyncTaskResult(AsyncTask task, Data... data) {
            mTask = task;
            mData = data;
        }
    }
}
```

## 注意

需要注意的是 AsyncTask 不同版本的实现是不同的，这也是比较坑的地方。