title: AsyncTask源码分析
date: 2016-09-08 22:12:13
tags: [Android]
toc: true
---

AsyncTask是Android提供给我们的一个轻量级处理异步任务的类，可以把执行的进度和执行的结果反馈给UI线程来更新UI。

<!--more-->

>- 文章来源：itsCoder 的 [WeeklyBolg](https://github.com/itsCoder/weeklyblog) 项目
>- itsCoder主页：[http://itscoder.com/](http://itscoder.com/)
>- 作者：[Melodyxxx](http://melodyxxx.com/)
>- 审阅者：[暂无]()

## AsyncTask的使用

### AsyncTask的基本使用

Android已经为我们将AsyncTask封装的已经很好了，所以AsyncTask使用起来也比较简单，下面是一般的使用方法：

``` java
new AsyncTask<String, Integer, Void>() {

            // 运行在UI线程，异步任务执行前调用
            @Override
            protected void onPreExecute() {

            }

            // 运行在子线程，用于处理耗时操作
            @Override
            protected Void doInBackground(String... strings) {
                // ...

                // 触发onProgressUpdate方法更新进度信息
                publishProgress(0);

                return null;
            }

            // 运行在UI线程，可用于更新进度条进度信息
            @Override
            protected void onProgressUpdate(Integer... values) {
                // update progress
            }

            // 运行在UI线程，异步任务执行完毕后调用
            @Override
            protected void onPostExecute(Void aVoid) {

            }

        }.execute("task_params");
```

### AsyncTask的取消

可以手动调用AsyncTask的cancle()方法来尝试取消一个任务，需要注意的是这里的取消只是改变任务的状态标记，并不会立马停止后台任务，可以通过isCancelled()方法获取当前的取消状态来做相应的处理，例如可以在doInBackground()方法里面像下面这样处理，不断的检测状态：
``` java
@Override
protected Void doInBackground(String... strings) {

	for (int i = 0; i <= 10; i++){
		// ...
		if (isCancelled()){
			break;
		}
	}

    return null;
}
```

## AsyncTask的工作原理分析

> 下面的分析的为Android 5.0的源码

先从AsyncTask从AsyncTask的执行入口方法execute()开始分析

``` java
@MainThread
public final AsyncTask<Params, Progress, Result> execute(Params... params) {
    return executeOnExecutor(sDefaultExecutor, params);
}
```

内部调用了AsyncTask的executeOnExecutor()方法，跟进去

``` java
@MainThread
public final AsyncTask<Params, Progress, Result> executeOnExecutor(Executor exec,
        Params... params) {

    // 首先是对当前的状态进行检测，如果当前任务正在进行或者已经完成，则会抛出异常
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

    // 将状态设置为"正在进行"
    mStatus = Status.RUNNING;

    // 回调我们自己可以复写的onPreExecute()方法
    onPreExecute();

    // 将我们传进来的参数赋给mWorker.mParams
    mWorker.mParams = params;
    exec.execute(mFuture);

    return this;
}
```
上面回调了onPreExecute()方法，而当前线程为AsyncTask创建并启动的线程，所以**AsyncTask启动必须在UI线程，不能在子线程中**，否则onPreExecute()方法会在子线程中被调用。

在execute方法中，将sDefaultExecutor传递给了executeOnExecutor的第一个参数，即上面的exec，这里的sDefaultExecutor是一个串行线程池，一个进程中的所有AsyncTask全部在这个串行线程池中排队执行：
``` java
private static volatile Executor sDefaultExecutor = SERIAL_EXECUTOR;
```
``` java
public static final Executor SERIAL_EXECUTOR = new SerialExecutor();
```
``` java
    private static class SerialExecutor implements Executor {
        final ArrayDeque<Runnable> mTasks = new ArrayDeque<Runnable>();
        Runnable mActive;

        public synchronized void execute(final Runnable r) {
            // 将任务排队处理
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
                // 当前没有正在执行的任务，取出下一个任务执行
                scheduleNext();
            }
        }

        protected synchronized void scheduleNext() {
            if ((mActive = mTasks.poll()) != null) {
                // 取出下一个任务放进THREAD_POOL_EXECUTOR线程池执行
                THREAD_POOL_EXECUTOR.execute(mActive);
            }
        }
    }
```
可以看到，这个线程池的execute方法会将任务进行排队，然后判断当前若没有正在执行的任务，则取出一个任务放进THREAD_POOL_EXECUTOR线程池去执行，而这个THREAD_POOL_EXECUTOR是一个并行线程池。所以又可以得出的结论是，**AsyncTaks内有两个线程池，一个串行线程池和一个并行线程池，串行线程池用于给任务排队使用，而THREAD_POOL_EXECUTOR线程池才是任务真正执行的地方。**

然后回到上面的executeOnExecutor()方法内的exec.execute(mFuture)这一行，由刚才分析可知这个mFuture就是我们的任务了，这个任务mFuture是在AsyncTask的构造器中初始化的：


``` java
public AsyncTask() {
    mWorker = new WorkerRunnable<Params, Result>() {
        public Result call() throws Exception {
            mTaskInvoked.set(true);

            Process.setThreadPriority(Process.THREAD_PRIORITY_BACKGROUND);
            //noinspection unchecked
            Result result = doInBackground(mParams);
            Binder.flushPendingCommands();
            return postResult(result);
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
```
可以看到mFuture初始化的时候用到了mWorker，先记住mWorker里面有个call()方法，再看mFuture的构造方法，可以看到，mFuture将mWorker赋值给了mFuture内的callable对象：
``` java
public FutureTask(Callable<V> callable) {
    if (callable == null)
        throw new NullPointerException();
    // mFuture将mWorker赋值给了mFuture内的callable对象
    this.callable = callable;
    this.state = NEW;       // ensure visibility of callable
}
```

前面说了，AsyncTask启动执行后，mFuture会被放到THREAD_POOL_EXECUTOR中去执行，执行会调用mFuture的run()方法，所以下面看mFuture对应的的FutureTask中的run方法()：


``` java
public void run() {
    if (state != NEW ||
        !U.compareAndSwapObject(this, RUNNER, null, Thread.currentThread()))
        return;
    try {
        // 这里的c为之前的mWorker对象
        Callable<V> c = callable;
        if (c != null && state == NEW) {
            V result;
            boolean ran;
            try {
                // 调用mWorker的call()方法
                result = c.call();
                ran = true;
            } catch (Throwable ex) {
                result = null;
                ran = false;
                setException(ex);
            }
            if (ran)
                set(result);
        }
    } finally {
        // runner must be non-null until state is settled to
        // prevent concurrent calls to run()
        runner = null;
        // state must be re-read after nulling runner to prevent
        // leaked interrupts
        int s = state;
        if (s >= INTERRUPTING)
            handlePossibleCancellationInterrupt(s);
    }
}
```

所以执行mFuture后会调用mWorker的call()方法，mWorker的call()方法实现如下：
``` java
mWorker = new WorkerRunnable<Params, Result>() {
    public Result call() throws Exception {
        mTaskInvoked.set(true);

        Process.setThreadPriority(Process.THREAD_PRIORITY_BACKGROUND);
        //noinspection unchecked
        Result result = doInBackground(mParams);
        Binder.flushPendingCommands();
        return postResult(result);
    }
};
```
call()方法内回调了熟悉的doInBackground()方法，并将之前保存起来的mWorker内的mParams传给了它。需要注意的是，当前线程已经切换到了线程池中，而非UI线程中，所以doInBackground()方法是在非UI线程中调用的。然后将doInBackground()执行完的返回值result传给了postResult()，跟进去看postResult()的实现：

``` java
private Result postResult(Result result) {
    @SuppressWarnings("unchecked")
    Message message = getHandler().obtainMessage(MESSAGE_POST_RESULT,
            new AsyncTaskResult<Result>(this, result));
    message.sendToTarget();
    return result;
}
```

这里使用Handler发送了一个MESSAGE_POST_RESULT的消息，这里getHandler()获取到的Handler是一个与UI主线程Looper相关联的sHandler，下面是sHandler的获取和实例化过程。

``` java
private static Handler getHandler() {
    synchronized (AsyncTask.class) {
        if (sHandler == null) {
            sHandler = new InternalHandler();
        }
        return sHandler;
    }
}
```

``` java
private static class InternalHandler extends Handler {
    public InternalHandler() {
        // 与UI主线程的Looper关联
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
```

通过使用sHandler发送MESSAGE_POST_RESULT消息，从线程池中切换到UI主线程中，然后执行AsyncTask的finish()方法：

``` java
private void finish(Result result) {
    if (isCancelled()) {
        onCancelled(result);
    } else {
        onPostExecute(result);
    }
    mStatus = Status.FINISHED;
}
```

在AsyncTask的finish()方法中首先会判断当前任务的状态是否已取消，如果已取消则会回调onCancelled()方法，否则直接回调onPostExecute()方法，最后将任务的状态设置为FINISHED完成状态。

## AsyncTask并行

在Android 1.6之前，AsyncTask是串行执行任务的，Android 1.6开始AsyncTask开始采用线程池处理并行任务，从Android 3.0开始，为了避免AsyncTask所带来的并发错误，AsyncTask又采用一个线程来串行执行任务。

根据上面Android 5.0的源码分析可以看到AsyncTask是串行执行任务的，当然我们也可以手动使其并行执行任务，只要执行时把`execute(params)`改成`executeOnExecutor(AsyncTask.THREAD_POOL_EXECUTOR,params)`即可，不过需要注意只能在Android 3.0+才能这样做。

## 总结

- AsyncTask使用起来比较方便，适合不是特别耗时的操作。
- AsyncTask的execute方法必须在主线程中调用。
- 再次启动一个已执行过的AsyncTask对象会抛异常。
- AsyncTask在不同的Android版本上会有不同的表现。


> 本文参考资料《Android开发艺术探索》