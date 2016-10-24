---
title: 常见内存泄漏场景以及解决办法
date: 2016-10-21 08:03:08
tags: 内存泄漏

---

阅读《Android开发艺术探索》中关于性能优化之**内存泄漏优化**所做读书笔记。结合网上资料，列举出一些常见的内存泄漏场景，以及相应解决策略。最后介绍下使用 LeakCanary 来检测 App 是否存在内存泄漏。

<!--more-->

关于什么是内存泄漏以及内存泄漏的危害，本文不做赘述，不了解的读者可以自行 Google 或者参考文章结尾部分给出的参考链接。

### Handler 导致的内存泄漏

稍微有点开发经验的开发者，都应该知道日常开发过程中，使用 Handler 不当会导致内存泄漏，如下代码就可能导致内存泄漏的发生：

``` java
public class SampleActivity extends AppCompatActivity {

    private final Handler mLeakyHandler = new Handler() {
        @Override
        public void handleMessage(Message msg) {
            // ...
        }
    };
  
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        // Post a message and delay its execution for 10 minutes.
        mLeakyHandler.postDelayed(new Runnable() {
            @Override
            public void run() { /* ... */ }
        },  1000 * 60 * 1);
        // Go back to the previous Activity.
        finish();
    }
}
```

运行实例程序，过一会 LeakCanary 就会弹出提示通知，说内存泄漏了，如下图：

![](http://7xrl8j.com1.z0.glb.clouddn.com/Screenshot_20161018-130237.png?imageView2/1/w/350/h/500)

 **原理浅析**

当 Android 应用程序启动时，framework 会为该应用程序的主线程创建一个 Looper 对象。Looper 对象包含一个简单的消息队列 Message Queue，并且能够循环的处理队列中的消息。这些消息包括大多数应用程序 framework 事件，例如 Activity 生命周期方法调用、button 点击等，这些消息都会被添加到消息队列中并被逐个处理。主线程的 Looper 对象会伴随该**应用程序的整个生命周期**。

当我们在主线程中实例化一个 Handler 对象后，会自动与主线程 **Looper 的消息队列关联起来**。所有发送到消息队列的消息 Message 都会拥有一个对 Handler 的引用，而此时当前 Activity 如果已经结束/销毁，而 Handler 由于是非静态内部类就会持有外部类的对象，抓住当前 Activity 对象不放，此时就极有可能导致内存泄漏。

> 静态内部类不会持有外部类的引用，其跟外部类的关系，可以看成平级。

解决办法就是使用静态内部类加 WeakRefrence，如下所示：

``` java
 private static class MyHandler extends Handler {
        private final WeakReference<Sample2Activity> mActivity;

        public MyHandler(Sample2Activity activity) {
            mActivity = new WeakReference<Sample2Activity>(activity);
        }

        @Override
        public void handleMessage   (Message msg) {
            Sample2Activity activity = mActivity.get();
            if (activity != null) {
                // ...
            }
        }
    }
```

> WeakRefrence 的相关概念：弱引用对象的存在不会阻止它所指向的对象变被垃圾回收器回收。弱引用最常见的用途是实现规范映射(canonicalizing mappings，比如哈希表）。假设垃圾收集器在某个时间点决定一个对象是**弱可达的(weakly reachable)**（也就是说当前指向它的全都是弱引用），这时垃圾收集器会清除所有指向该对象的弱引用，然后垃圾收集器会把这个弱可达对象标记为可终结(finalizable)的，这样它们随后就会被回收。与此同时或稍后，垃圾收集器会把那些刚清除的弱引用放入创建弱引用对象时所登记到的**引用队列(Reference Queue)**中。

### 静态变量导致内存泄漏

``` java
public class Sample3Activity extends AppCompatActivity{
    private static Context sContext;
  
    @Override protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_sample3);
        sContext = this;
        //finish();
        Button button = (Button)findViewById(R.id.finish);
        button.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View view) {
                finish();
            }
        });
    }
}
```

如上也有可能导致内存泄漏，导致内存泄漏的原因是：静态变量持有当前 Activity。导致当前 Activity 结束时候，静态变量仍然持有它的引用。深层次探究就要清楚静态变量在 Android 中的生命周期了。可以参见[[Android静态变量的生命周期](http://blog.csdn.net/ctcwri/article/details/8858414)].这篇文章讲述的非常清楚，**大力推荐**。

### 单利模式导致内存泄漏

``` java
public class AppManager {
    private static AppManager instance;
    private Context context;
    private AppManager(Context context) {
        this.context = context;
    }
    public static AppManager getInstance(Context context) {
        if (instance != null) {
            instance = new AppManager(context);
        }
        return instance;
    }
}
```

当创建上述单例的时候，由于需要传入一个Context，所以这个 Contex t的生命周期的长短至关重要： 
**1.如果是 Application 的 Context：**OK，这样是可以的，因为单例的生命周期和 Application 的一样长 。
**2.如果是 Activity 的 Context：**当这个 Context 所对应的 Activity 退出时，它的内存并不会被回收，因为单例对象持有该 Activity 的引用。

解决办法如下所示：

``` java
public class AppManager {
    private static AppManager instance;
    private Context context;
    private AppManager(Context context) {
        this.context = context.getApplicationContext();
    }
    public static AppManager getInstance(Context context) {
        if (instance != null) {
            instance = new AppManager(context);
        }
        return instance;
    }
}
```

这样不管传入什么 Context 最终将使用 Application 的 Context，而单例的生命周期和应用的一样长，这样就防止了内存泄漏

### 非静态内部类持有外部类的实例

```java
public class Sample4Activity extends AppCompatActivity {
    private static LeakSample mLeakSample = null;
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        if(mLeakSample == null){
            mLeakSample = new LeakSample();
        }
        //...
    }
    class LeakSample {
        //...
    }
}
```

上述代码在 Activity 内部创建了一个非静态内部类的单例，每次启动 Activity 时都会使用该单例的数据(避免了资源的重复创建),**这种写法却会造成内存泄漏**，同样因为非静态内部类持有外部类对象的原因。正确的做法为： 将该内部类设为静态内部类或将该内部类抽取出来封装成一个单例，如果需要使用Context，请使用ApplicationContext。

### 线程造成的内存泄漏

 Runnable 是一个匿名内部类( AsyncTask 存在匿名内部类的情况)，对当前 Activity 都有一个隐式引用。如果在当前 Activity 在销毁之前，任务还未完成，那么将导致 Activity 的内存资源无法回收，导致内存泄漏。实例代码如下：

``` java
public class Sample4Activity extends Activity {

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_sample3);
        leakSample();
        finish();
    }

    private void leakSample() {
        new MyThread().start();
    }
	
    private class MyThread extends Thread {
        @Override
        public void run() {
            while (true) {
                SystemClock.sleep(1000);
            }
        }
    }
}
```

如上在程序开启之后，App 配置好 LeakCanary 的手机界面上很快就会出现内存泄漏的通知。正确做法用静态内部类即可，如下所示：

``` java
public class Sample4Activity extends Activity {
    /**...**/
    private class MyThread extends Thread {
        @Override
        public void run() {
            while (true) {
                SystemClock.sleep(1000);
            }
        }
    }
}
```

### 属性动画导致内存泄漏

属性动画中有一类无线循环的动画，如果在当前 Activity 中播放此类动画，并且没有在结束的时候(onDestory)去停止该动画，那么动画会一直播放下去，尽管在界面上无法看见动画的运转，**但是在此时 Activity 的 View 会被动画所持有**，而 View 又持有当前 Activity，最终导致 Activity 无法被释放。动画的特征代码如下：

```java
animator.setRepeatCount(ValueAnimator.INFINITE);
```

解决办法自然很简单，在 OnDestory() 中去取消动画即可。

### 使用 LeakCanary 去检测内存泄漏

LeakCanary 应该可以说现在检查内存泄漏所必备的一个工具库了，使用也很简单：

在 build.gradle 中：

``` java
dependencies {
   debugCompile 'com.squareup.leakcanary:leakcanary-android:1.5'
   releaseCompile 'com.squareup.leakcanary:leakcanary-android-no-op:1.5'
   testCompile 'com.squareup.leakcanary:leakcanary-android-no-op:1.5'
 }
```

在 Application 中：

``` java
public class ExampleApplication extends Application {

  @Override public void onCreate() {
    super.onCreate();
    if (LeakCanary.isInAnalyzerProcess(this)) {
      // This process is dedicated to LeakCanary for heap analysis.
      // You should not init your app in this process.
      return;
    }
    LeakCanary.install(this);
    // Normal app init code...
  }
}
```

最后不要忘了去注册 Application 哦！

当你调试 App 时候，一旦发现内存吃紧。LeakCanary 会自动以通知的方式提醒你。Easy to go.当然啦，LeakCanary 的用法还有很多，更详细的就请大家移步 LeakCanary 主页查看啦。

**最后个人推荐一篇内存泄漏较好的文章[内存管理(3)-Android内存泄露分析](http://www.jianshu.com/p/c59c199ca9fa)。**

### 参考文章

* http://blog.nimbledroid.com/2016/05/23/memory-leaks-zh.html
* https://drakeet.me/android-leaks
* 《Android 开发艺术探索》