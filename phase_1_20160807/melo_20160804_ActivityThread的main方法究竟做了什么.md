---
title: ActivityThread的main方法究竟做了什么？
categories: Android
tags: [Android,ActivityThread,源码] 
---
ActivityThread的main方法究竟做了什么？
==
**写在前面：**
在暴雨天能去上课的都是好学生，能去上班的都是游泳运动员~
<!--more-->
![轻松一下](http://upload-images.jianshu.io/upload_images/1915184-0dd4fc7c8f275bd3?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


**问大家一个问题：**

Android中一个应用程序的**真正入口**是什么？

**无论你知道不知道，别着急回答，再问大家一个问题：**

Android不能像java一样直接跑在**main方法**的原因是什么？

Android应用程序的载体是APK文件，它本质上，是一个**资源和组件的容器**，APK文件和我们常见的**可执行文件**的区别在何处？

每个可执行文件运行在一个进程中，但是APK文件可能运行在一个单独的进程，也可以和其他APK运行在同一进程中，结合上面，我想表达的是：

**Android系统的设计理念就是弱化进程，取而代之是组件的概念。**

但是我们都知道，Android系统基于**Linux**系统之上，而Linux系统的运行环境恰恰就是由**进程**组成。所有的Android应用进程都是有Zygote进程fork出来的，因此构成进程的地层系统、虚拟机、动态库等，都是相同的。

当然Android除了继承从Zygote中得到的某些基础的“家当”之外，Android还需要在应用的Java层建立一套框架来管理运行的组件。由于每个应用的配置都不相同，因此不能再Zygote中完全建立好再继承，只能在应用启动时创建。

**这套框架就构成了Android应用的基础。**

而这套框架有很多核心类，比如：

**ActivityThread**、**ApplicationThread**、**Context**、**ActivityManagerService**等等。这里我先给自己挖一个坑，将来慢慢填上，争取清晰简洁的给大家讲明白**Android的组件管理**。

而今天，我们现在聊聊**ActivityThread**的**main**方法

ActivityThread
--

好像忘了点什么。。。

对对，回头看看，我们还有两个问题没解答呢，整个Android应用进程的体系非常复杂，而ActivityThread是真正的核心类，它的**main方法**，是**整个应用进程的入口**。

所以当有人问你应用进程的真正入口是什么，你回答“Activity 的 onCreate 方法”显然就没理解这个问题的意思。

而第二个问题，相信你心里肯定知道大概怎么回答，我们的一个Android应用程序可以是理解为是**四大组件和各种资源的集合**，它需要各种各样的环境资源，当然不能像Java直接跑在main方法里面。

而今天我们就来看看ActivityThread的main方法究竟做了些什么。

在此之前，安利一个看源码的网站，非常不错

[http://grepcode.com/](http://grepcode.com/)

点击进去类名就可以查看源码了

![成员变量](http://upload-images.jianshu.io/upload_images/1915184-a143e41ffdd1b71a?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

ActivityThread的源码有5000多行，显然我没能力弄懂它每一行代码的意思，不过我们只要知道它大体上负责着什么功能和职责，就可以了。

看看上图中的成员变量，在给大家上一个图，就能理解**ActivityThread管理着**什么。

![ActivityThread类的对象的关联图](http://upload-images.jianshu.io/upload_images/1915184-b76be1a1e19381df?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

从图中可以知道，**mActivities**、**mServices**和**mProviderMap** 这三个变量都被保存在**ArrayMap**之中，他们分别保存了应用中所有的**Activity**对象、**Services**对象、和**ContentProvider**对象。 咦？同为四大组件的BroadcastReceive去哪里了？注意，BroadcastReceiver对象没有必要用任何数据结构来保存，因为BroadcastReceiver对象的生命周期很短暂，属于我调用它时，再创建运行，因此不需要保存**BroadcastReceiver**的对象。

我们都知道应用中Applicaiton对象是唯一的，而**mInitialApplication**变量是恰恰是Application对象。当你的应用自定义一个派生Applicaiton类，则它就是mInitialApplication了。

**ApplicationThread**类型变量mAppThread是一个**Binder**实体对象，**ActivityManagerService**作为Client端调用ApplicationThread的接口，目的是用来调度管理Activity，这个我们未来会细说。

变量**mResourcesManager**管理着应用中的资源。

一口气说了这么多，怎么样，ActivityThread是不是相当于一个CEO，管理调度着几乎所有的**Android应用进程的资源和四大组件**

上面非常多的问题我未来会给大家慢慢解答，因为篇幅太长反而会影响阅读和知识的吸收，话不多说，**来看看入口方法main都做了些什么**？

ActivityThread的main方法
--

感兴趣的同学去刚才给出的网站上搜搜ActivityThread的类，大致浏览一下，这里先贴出**main方法**的代码：

```
public static void More ...main(String[] args) {
5220        SamplingProfilerIntegration.start();
5221
5222        // CloseGuard defaults to true and can be quite spammy.  We
5223        // disable it here, but selectively enable it later (via
5224        // StrictMode) on debug builds, but using DropBox, not logs.
5225        CloseGuard.setEnabled(false);
5226		// 初始化应用中需要使用的系统路径
5227        Environment.initForCurrentUser();
5228
5229        // Set the reporter for event logging in libcore
5230        EventLogger.setReporter(new EventLoggingReporter());
5231		//增加一个保存key的provider
5232        Security.addProvider(new AndroidKeyStoreProvider());
5233
5234        // Make sure TrustedCertificateStore looks in the right place for CA certificates
			//为应用设置当前用户的CA证书保存的位置
5235        final File configDir = Environment.getUserConfigDirectory(UserHandle.myUserId());
5236        TrustedCertificateStore.setDefaultUserDirectory(configDir);
5237		//设置进程的名称
5238        Process.setArgV0("<pre-initialized>");
5239
5240        Looper.prepareMainLooper();
5241		//创建ActivityThread 对象
5242        ActivityThread thread = new ActivityThread();
5243        thread.attach(false);
5244
5245        if (sMainThreadHandler == null) {
5246            sMainThreadHandler = thread.getHandler();
5247        }
5248
5249        if (false) {
5250            Looper.myLooper().setMessageLogging(new
5251                    LogPrinter(Log.DEBUG, "ActivityThread"));
5252        }
5253
5254        Looper.loop();
5255
5256        throw new RuntimeException("Main thread loop unexpectedly exited");
5257    }
```
代码并不多，但是条条关键，这些操作我都为大家写了注释，看一下就知道程序在做什么。

```
			Looper.prepareMainLooper();
			//创建ActivityThread 对象
	        ActivityThread thread = new ActivityThread();
	        thread.attach(false);
	        
	        if (sMainThreadHandler == null) {
		            sMainThreadHandler =thread.getHandler();
	        }

	        if (false) {
	            Looper.myLooper().setMessageLogging(new
	                    LogPrinter(Log.DEBUG, "ActivityThread"));
	        }
	
	        Looper.loop();
	
	        throw new RuntimeException("Main thread loop unexpectedly exited");
	    }
```
这几行代码拿出来单独讲解一下，首先Looper.prepareMainLooper();是为主线程创建了Looper，然后thread.getHandler();是保存了主线程的Handler，最后Looper.loop();进入消息循环。

如果不了解Android的消息机制，大家可以来看看以前我写的文章来了解一下：

[Android消息机制详解](http://www.jianshu.com/p/fad4e2ae32f5)

马上就要大功告成了，最后还剩下一行代码还没解释：

thread.attach(false);

继续跟进attach方法，一探究竟：

```
			if (!system) {
5080            ViewRootImpl.addFirstDrawHandler(new Runnable() {
5081                @Override
5082                public void More ...run() {
5083                    ensureJitEnabled();
5084                }
5085            });
5086            android.ddm.DdmHandleAppName.setAppName("<pre-initialized>",
5087                                                    UserHandle.myUserId());
//将mAppThread放到RuntimeInit类中的静态变量
5088            RuntimeInit.setApplicationObject(mAppThread.asBinder());
5089            final IActivityManager mgr = ActivityManagerNative.getDefault();
5090            try {
					//将mAppThread传入ActivityThreadManager中
5091                mgr.attachApplication(mAppThread);
5092            } catch (RemoteException ex) {
5093                // Ignore
5094            }
5095            // Watch for getting close to heap limit.
5096            BinderInternal.addGcWatcher(new Runnable() {
5097                @Override public void More ...run() {
5098                    if (!mSomeActivitiesChanged) {
5099                        return;
5100                    }
5101                    Runtime runtime = Runtime.getRuntime();
5102                    long dalvikMax = runtime.maxMemory();
5103                    long dalvikUsed = runtime.totalMemory() - runtime.freeMemory();
5104                    if (dalvikUsed > ((3*dalvikMax)/4)) {
5105                        if (DEBUG_MEMORY_TRIM) Slog.d(TAG, "Dalvik max=" + (dalvikMax/1024)
5106                                + " total=" + (runtime.totalMemory()/1024)
5107                                + " used=" + (dalvikUsed/1024));
5108                        mSomeActivitiesChanged = false;
5109                        try {
5110                            mgr.releaseSomeActivities(mAppThread);
5111                        } catch (RemoteException e) {
5112                        }
5113                    }
5114                }
5115            });
5116        }
```

当传入的参数为false时，就走到了如上面贴出的代码中：

此时主要完成两件事

1.调用 RuntimeInit.setApplicationObject() 方法，把对象mAppThread（Binder）放到了RuntimeInit类中的静态变量mApplicationObject中。

```
	public static final void More ...setApplicationObject(IBinder app) {
360        mApplicationObject = app;
361    }
```

mAppThread的类型是**ApplicationThread**，它是ActivityThread的成员变量，定义和初始化如下：

```
final ApplicationThread mAppThread = new ApplicationThread();
```

第二件事比较关键了，就是调用**ActivityManagerService**的attachApplication()方法，将mAppThread 作为参数传入ActivityManagerService，这样ActivityManagerService就可以调用**ApplicaitonThread**的接口了。这与我们刚才说的，ActivityManagerService作为Client端调用ApplicaitonThread的接口管理Activity，就不谋而合了。

**写在后面：**
本文我们明白了ActiivtyThread作为进程的核心类它都管理着哪些对象，并且解释了程序真正入口ActivityThread的main方法都完成了哪些重要的操作，之后会继续带大家了解相关共同组成Android应用进程的核心类，如果有问题和疑问可以多交流，毕竟我也是边学习变整理总结嘛~

如果有需要，推荐你了解一下Context，对你会很有帮助哦~

[你足够了解Context吗？](http://www.jianshu.com/p/46c35c5079b4)

最后PS：
**注意保护电脑不要被水淹！**