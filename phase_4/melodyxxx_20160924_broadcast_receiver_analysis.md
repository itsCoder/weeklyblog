title: BroadcastReceiver 的工作过程分析
date: 2016-09-24 16:16:27
tags: [Android]
toc: true
---

了解 BroadcastReceiver 的注册过程以及广播的发送与接收的过程。

<!--more-->

>- 文章来源：itsCoder 的 [WeeklyBolg](https://github.com/itsCoder/weeklyblog) 项目
>- itsCoder主页：[http://itscoder.com/](http://itscoder.com/)
>- 作者：[Melodyxxx](http://melodyxxx.com/)
>- 审阅者：[暂无]()


在 Android 中，广播的注册分为静态注册和动态注册，静态注册是指在 AndroidManifest 文件中注册，动态注册是指在 Java 代码中通过 `registerReceiver` 方法注册的。静态注册的广播在应用安装时由系统自动完成注册，具体来说是由 PMS (PackageManagerService)来完成注册的。本文只分析广播的动态注册过程。

> 本文为 《 Android开发艺术探索 》 笔记
> 源码为 Android 6.0

## 广播的注册过程

首先，分析过程当然从注册的入口说起，我们平常在 Activity 内写 registerReceiver 来注册动态广播调用的是父类 ContextWrapper 内的 registerReceiver 方法，代码如下：

``` java
@Override
public Intent registerReceiver(
    BroadcastReceiver receiver, IntentFilter filter) {
    return mBase.registerReceiver(receiver, filter);
}
```
内部使用的是典型的桥接模式，转而调用 mBase 的 registerReceiver 方法，这里的 mBase 是一个 Context 类型对象，其实现类是 ContextImpl (在 Activity 的启动过程和创建过程中可以分析得出其实现类是 ContextImpl )，所以下面直接来看 ContextImpl 的 `registerReceiver` 方法，如下：

``` java
@Override
public Intent registerReceiver(BroadcastReceiver receiver, IntentFilter filter) {
    return registerReceiver(receiver, filter, null, null);
}
```
继续跟下去：

``` java
@Override
public Intent registerReceiver(BroadcastReceiver receiver, IntentFilter filter,
        String broadcastPermission, Handler scheduler) {
    return registerReceiverInternal(receiver, getUserId(),
            filter, broadcastPermission, scheduler, getOuterContext());
}
```
内部调用了 registerReceiverInternal ，再继续跟下去：

``` java
private Intent registerReceiverInternal(BroadcastReceiver receiver, int userId,
        IntentFilter filter, String broadcastPermission,
        Handler scheduler, Context context) {
    IIntentReceiver rd = null;
    if (receiver != null) {
        if (mPackageInfo != null && context != null) {
            if (scheduler == null) {
                scheduler = mMainThread.getHandler();
            }
            rd = mPackageInfo.getReceiverDispatcher(
                receiver, context, scheduler,
                mMainThread.getInstrumentation(), true);
        } else {
            if (scheduler == null) {
                scheduler = mMainThread.getHandler();
            }
            rd = new LoadedApk.ReceiverDispatcher(
                    receiver, context, scheduler, null, true).getIIntentReceiver();
        }
    }
    try {
        return ActivityManagerNative.getDefault().registerReceiver(
                mMainThread.getApplicationThread(), mBasePackageName,
                rd, filter, broadcastPermission, userId);
    } catch (RemoteException e) {
        return null;
    }
}
```
上面一段代码不长，也很好理解，看代码的最后面一段：
``` java
...
try {
    return ActivityManagerNative.getDefault().registerReceiver(
            mMainThread.getApplicationThread(), mBasePackageName,
            rd, filter, broadcastPermission, userId);
} catch (RemoteException e) {
    return null;
}
...

```
先看这个 ActivityManagerNative 的 `getDefault()` 方法：
``` java
static public IActivityManager getDefault() {
    return gDefault.get();
}
```
返回类型是 IActivity Manager ,是一个 Binder 接口，实际上就是 ActivityManagerService (后面简称 AMS )，因为 ActivityManagerNative 继承自 Binder 且实现了 IActivityManager 接口，而 AMS 继承自 ActivityManagerNative 。

所以到这里可以得到的结论是，注册广播时，最终会跨进程调用交给 AMS 去完成注册。

因为需要 AMS 来注册，所以需要把广播传给 AMS ，但是这里 ActivityManagerService 的 `registerReceiver` 方法的第三个参数传的是 IIntentReceiver 类型的 rd ，为 IIntentReceiver 类型对象，不像类似启动 Service 或者 Activity 直接传递 Intent 给 AMS 那样来直接传递广播。之所以采用 IIntentReceiver 而不是直接采用 BroadcastReceiver ，这是因为上述注册过程是一个进程间通信的过程，而 BroadcastReceiver 作为 Android 的一个组件是不能直接跨进程传递的，所以需要 IIntentReceiver 来中转一下，毫无疑问， IIntentReceiver 必须是一个 Binder 接口。

所以看第三个参数 IIntentReceiver 的实例化过程，其中可以看到：
``` java
...
rd = mPackageInfo.getReceiverDispatcher(
    receiver, context, scheduler,
    mMainThread.getInstrumentation(), true);
...
```
 mPackageInfo 为 LoadApk 类型对象，看 `getReceiverDispatcher` 方法实现：
``` java
public IIntentReceiver getReceiverDispatcher(BroadcastReceiver r,
        Context context, Handler handler,
        Instrumentation instrumentation, boolean registered) {
    synchronized (mReceivers) {
        LoadedApk.ReceiverDispatcher rd = null;
        ArrayMap<BroadcastReceiver, LoadedApk.ReceiverDispatcher> map = null;
        if (registered) {
            map = mReceivers.get(context);
            if (map != null) {
                rd = map.get(r);
            }
        }
        if (rd == null) {
            rd = new ReceiverDispatcher(r, context, handler,
                    instrumentation, registered);
            if (registered) {
                if (map == null) {
                    map = new ArrayMap<BroadcastReceiver, LoadedApk.ReceiverDispatcher>();
                    mReceivers.put(context, map);
                }
                map.put(r, rd);
            }
        } else {
            rd.validate(context, handler);
        }
        rd.mForgotten = false;
        return rd.getIIntentReceiver();
    }
}
```
最后一行`return rd.getIIntentReceiver();`，这里的 rd 为 LoadedApk 的内部类 ReceiverDispatcher ， ReceiverDispatcher 的内部封装了 BroadcastReceiver 和 InnerReceiver ( InnerReceiver 为 ReceiverDispatcher 的内部类)对象。这个 InnerReceiver 继承自 IIntentReceiver.Stub ，很明显是 AIDL 的形式，没错，就是 IIntentReceiver 的具体实现，其也含有 ReceiverDispatcher 的弱引用。 ReceiverDispatcher的 `getIIntentReceiver()` 方法返回的就是这个 InnerReceiver 对象，将其传递给了 AMS 。 ReceiverDispatcher 构造器实现：

``` java
ReceiverDispatcher(BroadcastReceiver receiver, Context context,
        Handler activityThread, Instrumentation instrumentation,
        boolean registered) {
    if (activityThread == null) {
        throw new NullPointerException("Handler must not be null");
    }

    mIIntentReceiver = new InnerReceiver(this, !registered);
    mReceiver = receiver;
    mContext = context;
    mActivityThread = activityThread;
    mInstrumentation = instrumentation;
    mRegistered = registered;
    mLocation = new IntentReceiverLeaked(null);
    mLocation.fillInStackTrace();
}
```

再来梳理一遍，跨进程交由 AMS 去注册广播的时候，将 InnerReceiver 对象传递给了 AMS ， InnerReceiver 是 ReceiverDispatcher 的内部类， ReceiverDispatcher 内保存了要注册的 BroadcastReceiver 和继承了 IIntentReceiver.Stub 的 InnerReceiver ， InnerReceiver 也有其外部类的 ReceiverDispatcher 的引用，后面可以方便的调用 ReceiverDispatcher 内部的 BroadcastReceiver 的 `onReceive()` 方法。

下面再看 AMS 中的 registerReceiver 的方法，这个方法很长，只看关键的部分：
``` java
public Intent registerReceiver(IApplicationThread caller, String callerPackage,
        IIntentReceiver receiver, IntentFilter filter, String permission, int userId) {
	....
	// 将远程receiver(IIntentReceiver)对象保存起来
    mRegisteredReceivers.put(receiver.asBinder(), rl);
	....
    BroadcastFilter bf = new BroadcastFilter(filter/*远程IntentFilter*/, rl, callerPackage,
            permission, callingUid, userId);
    rl.add(bf);
    if (!bf.debugCheck()) {
        Slog.w(TAG, "==> For Dynamic broadcast");
    }
	// 保存远程IntentFilter
    mReceiverResolver.addFilter(bf);
	....
```
最终在 AMS 内，将远程 receiver ( IIntentReceiver ) 对象和远程 IntentFilter 保存起来，完成动态广播的注册。

## 广播的发送与接收过程

下面开始分析发送广播和接收广播的过程，也是先从 ContextImpl 的 `sendBroadcast` 方法出发：

``` java
@Override
public void sendBroadcast(Intent intent) {
    warnIfCallingFromSystemProcess();
    String resolvedType = intent.resolveTypeIfNeeded(getContentResolver());
    try {
        intent.prepareToLeaveProcess();
        ActivityManagerNative.getDefault().broadcastIntent(
                mMainThread.getApplicationThread(), intent, resolvedType, null,
                Activity.RESULT_OK, null, null, null, AppOpsManager.OP_NONE, null, false, false,
                getUserId());
    } catch (RemoteException e) {
        throw new RuntimeException("Failure from system", e);
    }
}
```
内部没干什么事情，由上面分析可知，直接跨进程交由AMS处理，调用了 AMS 的 `broadcastIntent` 方法，在 AMS 的 `broadcastIntent` 方法内调用了 `broadcastIntentLocked` 方法，这个方法很长，先看该方法的最前面有下面这句话：

``` java
// By default broadcasts do not go to stopped apps.
intent.addFlags(Intent.FLAG_EXCLUDE_STOPPED_PACKAGES);
```
从注释可以看出，添加这个标记的目的是默认广播不会传递给那些已经停止的应用，一个应用处于停止状态有两种情形：
- 应用安装后未运行
- 应用被手动或者其他应用强行停止了

从Android 3.1开始，为Intent添加了两个标记位：
- **`FLAG_EXCLUDE_STOPPED_PACKAGES :** 表示不包含已经停止的应用，这个时候广播不会发送给已经停止的应用
- **FLAG_INCLUDE_STOPPED_PACKAGES :** 表示包含已经停止的应用，这个时候广播会发送给已经停止的应用

从 Android 3.1 开始，系统为所有的广播默认添加了 `FLAG_EXCLUDE_STOPPED_PACKAGES` 标志，这样做是为了防止广播无意间或者在不必要的时候调起已经停止运行的应用。如果的确需要调起已经停止的应用，那么只需要为广播的Intent添加 `FLAG_INCLUDE_STOPPED_PACKAGES` 标记即可。当两个标记共存时，以 `FLAG_INCLUDE_STOPPED_PACKAGES` 为准。

下面继续分析发送流程，在 `broadcastIntentLocked` 内部，会根据 intent-filter 查找出匹配的广播接收者并经过一系列的过滤，最终会将满足条件的广播接收者添加到 BroadcastQueue 中，接着 BroadcastQueue 就会将广播发送给相应的广播接收者，过程如下：

``` java
if ((receivers != null && receivers.size() > 0)
        || resultTo != null) {
    BroadcastQueue queue = broadcastQueueForIntent(intent);
    BroadcastRecord r = new BroadcastRecord(queue, intent, callerApp,
            callerPackage, callingPid, callingUid, resolvedType,
            requiredPermissions, appOp, brOptions, receivers, resultTo, resultCode,
            resultData, resultExtras, ordered, sticky, false, userId);

    if (DEBUG_BROADCAST) Slog.v(TAG_BROADCAST, "Enqueueing ordered broadcast " + r
            + ": prev had " + queue.mOrderedBroadcasts.size());
    if (DEBUG_BROADCAST) Slog.i(TAG_BROADCAST,
            "Enqueueing broadcast " + r.intent.getAction());

    boolean replaced = replacePending && queue.replaceOrderedBroadcastLocked(r);
    if (!replaced) {
        queue.enqueueOrderedBroadcastLocked(r);
        queue.scheduleBroadcastsLocked();
    }
}
```
在最后的 `queue.scheduleBroadcastsLocked();` 内看广播的发送过程，下面是 BroadcastQueue 的 `scheduleBroadcastsLocked` 方法：
``` java
public void scheduleBroadcastsLocked() {
    if (DEBUG_BROADCAST) Slog.v(TAG_BROADCAST, "Schedule broadcasts ["
            + mQueueName + "]: current="
            + mBroadcastsScheduled);

    if (mBroadcastsScheduled) {
        return;
    }
    mHandler.sendMessage(mHandler.obtainMessage(BROADCAST_INTENT_MSG, this));
    mBroadcastsScheduled = true;
}
```
`scheduleBroadcastsLocked` 里面并没有实际的处理广播，而是发送了一个 `BROADCAST_INTENT_MSG` 消息：
``` java
...
case BROADCAST_INTENT_MSG: {
    if (DEBUG_BROADCAST) Slog.v(
            TAG_BROADCAST, "Received BROADCAST_INTENT_MSG");
    processNextBroadcast(true);
} break;
...
```
收到消息后，调用了 `processNextBroadcast` 方法，在这个方法中有一段：
``` java
// First, deliver any non-serialized broadcasts right away.
while (mParallelBroadcasts.size() > 0) {
    r = mParallelBroadcasts.remove(0);
    r.dispatchTime = SystemClock.uptimeMillis();
    r.dispatchClockTime = System.currentTimeMillis();
    final int N = r.receivers.size();
    if (DEBUG_BROADCAST_LIGHT) Slog.v(TAG_BROADCAST, "Processing parallel broadcast ["
            + mQueueName + "] " + r);
    for (int i=0; i<N; i++) {
        Object target = r.receivers.get(i);
        if (DEBUG_BROADCAST)  Slog.v(TAG_BROADCAST,
                "Delivering non-ordered on [" + mQueueName + "] to registered "
                + target + ": " + r);
        deliverToRegisteredReceiverLocked(r, (BroadcastFilter)target, false);
    }
    addBroadcastToHistoryLocked(r);
    if (DEBUG_BROADCAST_LIGHT) Slog.v(TAG_BROADCAST, "Done with parallel broadcast ["
            + mQueueName + "] " + r);
}
```
通过注释可以看到， mParallelBroadcasts 中保存的都是无序广播，系统会遍历这个集合，然后将广播发送给它们的所有接收者，具体过程是通过 `deliverToRegisteredReceiverLocked` 方法实现的，在 `deliverToRegisteredReceiverLocked` 方法内部，调用了 `performReceiveLocked` 方法，如下：
``` java
private static void performReceiveLocked(ProcessRecord app, IIntentReceiver receiver,
        Intent intent, int resultCode, String data, Bundle extras,
        boolean ordered, boolean sticky, int sendingUser) throws RemoteException {
    // Send the intent to the receiver asynchronously using one-way binder calls.
    if (app != null) {
        if (app.thread != null) {
            // If we have an app thread, do the call through that so it is
            // correctly ordered with other one-way calls.
            app.thread.scheduleRegisteredReceiver(receiver, intent, resultCode,
                    data, extras, ordered, sticky, sendingUser, app.repProcState);
        } else {
            // Application has died. Receiver doesn't exist.
            throw new RemoteException("app.thread must not be null");
        }
    } else {
        receiver.performReceive(intent, resultCode, data, extras, ordered,
                sticky, sendingUser);
    }
}
```
上面的 `app.thread` 就是我们最开始在 ContextImpl 中的注册广播的时候 `registerReceiverInternal` 内传给AMS的 `mMainThread.getApplicationThread()` ：
``` java
...
return ActivityManagerNative.getDefault().registerReceiver(
        mMainThread.getApplicationThread(), mBasePackageName,
        rd, filter, broadcastPermission, userId);
...
```
所以 `app.therad` 就是 ApplicationThread 类型对象，下面看 ApplicationThread 的 `scheduleRegisteredReceiver` 方法实现：
``` java
public void scheduleRegisteredReceiver(IIntentReceiver receiver, Intent intent,
        int resultCode, String dataStr, Bundle extras, boolean ordered,
        boolean sticky, int sendingUser, int processState) throws RemoteException {
    updateProcessState(processState, false);
    receiver.performReceive(intent, resultCode, dataStr, extras, ordered,
            sticky, sendingUser);
}
```
直接调用了 IIntentReceiver 的 `performReceive` 方法，这个 IIntentReceiver 就是我们上面传递给 AMS 的那个，其实现类是 LoadApk 的内部类 InnerReceiver ，而 InnerReceiver 的 `performReceive` 方法内转而调用了其内部之前保存着的 ReceiverDispatcher 的 `performReceive` 方法，所以下面直接看 ReceiverDispatcher 的 `performReceive` 方法：
``` java
public void performReceive(Intent intent, int resultCode, String data,
        Bundle extras, boolean ordered, boolean sticky, int sendingUser) {
    if (ActivityThread.DEBUG_BROADCAST) {
        int seq = intent.getIntExtra("seq", -1);
        Slog.i(ActivityThread.TAG, "Enqueueing broadcast " + intent.getAction() + " seq=" + seq
                + " to " + mReceiver);
    }
    Args args = new Args(intent, resultCode, data, extras, ordered,
            sticky, sendingUser);
    if (!mActivityThread.post(args)) {
        if (mRegistered && ordered) {
            IActivityManager mgr = ActivityManagerNative.getDefault();
            if (ActivityThread.DEBUG_BROADCAST) Slog.i(ActivityThread.TAG,
                    "Finishing sync broadcast to " + mReceiver);
            args.sendFinished(mgr);
        }
    }
}
```
看 `mActivityThread.post(args) `，这个 mActivityThread 是一个 Handler ，其实就是 ActivityThread 中的那个 `mH` ，是一个与 UI 主线程相关联的 Handler ， Args 类实现了 Runnable 接口，所以其 `run` 方法是在主线程执行的，所以看 Args 的 `run` 方法：
``` java
public void run() {
    final BroadcastReceiver receiver = mReceiver;
	...
    try {
        ClassLoader cl =  mReceiver.getClass().getClassLoader();
        intent.setExtrasClassLoader(cl);
        setExtrasClassLoader(cl);
        receiver.setPendingResult(this);
        receiver.onReceive(mContext, intent);
    } catch (Exception e) {
       ...
    }
	...
}
```
很明显，在这里终于回调了 mReceiver 的 `onReceive` 方法，也是就我们自己继承 BroadcastReceiver 时所必须实现的那个 `onReceive` 方法，而且是在主线程中执行的。

## 总结

- 广播的注册、发送、接收都会经由 AMS 处理
- 广播的发送和接收其实是一个过程的两个阶段
- Android 3.1 之后系统默认那些已经停止的应用不会接收到广播


> 参考资料: 《 Android 开发艺术探索 》