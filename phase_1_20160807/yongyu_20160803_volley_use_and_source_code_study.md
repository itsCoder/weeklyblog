---

 title:  Volley学习笔记之简单使用及部分源码详解
 date:  2016-08-02 00:00：00
 categories:  Android源码学习

---

转载请附原文链接：[Volley学习笔记之简单使用及部分源码详解](http://yongyu.itscoder.com/2016/08/01/Volley%20Study/) 


## 一、使用背景简介

现在大多数手机App几乎都离不开网络技术，需要手机端与网络服务端进行数据交互，Android系统中主要提供了两种方式来进行HTTP通信，HttpURLConnection和HttpClient，在初学Android的时候，这两个类是我们最开始学着使用的，但是在使用过程中需要调取各种API，进行封，然后请求到的结果需要自己去解析，最后再将解析到的数据进行封装存到数据库，整个过程，相当复杂，而且重复性很高，于是针对这种情况，网络上就有大神封装了各种第三方使用库供我们使用，将这些流程封装组合包装优化，使得整个流程得以简化，只需简单配置，几行代码就完成整个过程。今天我们介绍的Volley 就是其中一个优秀第三方框架。笔者所在公司目前项目使用的就是Volley ，所以在使用的同时决定写一系列笔记分析，从学会简单使用，到最后的Volley源码分析一系列循序渐进的流程来理解Volley的实现原理。

## 二、Volley简介

Volley 是 Google在 Google I/O 2013 大会上 推出的 Android 异步网络请求和图片加载框架。

- 主要作用 ：实现异步网络请求、图片加载、本地数据缓存功能框架。

- 主要特点：

  (1).  Volley基于接口设计，扩展性强，我们可以根据我们的需求定制自己想要的请求方式方法。
  (2).  一定程度符合 Http 规范，包括返回 ResponseCode(2xx、3xx、4xx、5xx）的处理，请求头的处理，缓存机制的支持等。并支持重试及优先级定义。
  (3).  默认 Android2.3 及以上基于 HttpURLConnection，2.3 以下基于 HttpClient 实现。

  (4). 支持指定请求的优先级。

  (4). 高并发网络连接。

- 使用下载地址：https://android.googlesource.com/platform/frameworks/volley

  可以使用git 下载命令 git clone  https://android.googlesource.com/platform/frameworks/volley

  - 编译jar：
    `android update project -p . ant jar`


- 添加volley.jar到你的项目中

    **不过已经有人将volley的代码放到github上了：**
    [https://github.com/mcxiaoke/android-volley](https://github.com/mcxiaoke/android-volley)，你可以使用更加简单的方式来使用volley

## 三、简单使用

使用Volley框架实现网络数据请求主要有以下三个步骤：

1.  创建RequestQueue对象，定义网络请求队列，RequestQueue内部的设计就是非常合适高并发的，因此我们不必为每一次HTTP请求都创建一个RequestQueue对象，避免非常浪费资源的，一般全局使用一个就可以。
2.  创建XXXRequest对象(XXXRequest对象可以自己继承Request类进行封装定义，也可以使用Volley已经为我们提供的的StringReqeust、JsonArrayRequest、JsonObjectRequest)，这个类主要是功能是传入请求网址、解析返回数据、回调监听返回数据等功能的实现，也是我们经常继承包装的类，在这里可以实现我们想要的返回数据类型。
3.  把XXXRequest对象添加到RequestQueue中，开始执行网络请求。

怎么样，这样看来整个网络请求是否是变得便捷，不需要你去考虑，如何调取Http请求各种API，不需要考虑异步等问题，Volley已经帮助我们完成，下面简单以StringReqeust为例子发起一个Get请求，编写一个小的Demo用例，结合代码，加深理解：

网络请求队列一般都是整个APP内使用的全局性对象，因此最好写入Application类中，全局只使用一个，避免浪费资源：

```java
public class MyApplication extends Application {
// 建立请求队列
public static RequestQueue queue;

@Override
public void onCreate() {
    super.onCreate();
    queue = Volley.newRequestQueue(getApplicationContext());
}

public static RequestQueue getVolleyRequestQueue() {
    return queue;
 }
  
}
```
我们还需要修改AndroidManifest.xml文件，使APP的Application对象为我们刚定义的MyApplication，并添加INTERNET权限：

``` 
<uses-permission android:name="android.permission.INTERNET" />

<application

    android:name=".MyApplication"

    android:allowBackup="true"

    android:icon="@mipmap/ic_launcher"

    android:label="@string/app_name"

    android:supportsRtl="true"

    android:theme="@style/AppTheme" >

</application> 

```

创建StringReqeust对象，并将其添加到RequestQueue中：

```java
   /**
     * Request.Method.GET 指定请求方法，如果不输入，默认为Get方法
     *  new Response.Listener <String> 请求成功回调接口
     *   new Response.ErrorListener() 请求失败回到接口
     * @param url 要请求的网址
     */
    private void sendRequest(String url) {
        StringRequest stringRequest = new StringRequest(Request.Method.GET,url, new Response.Listener<String>() {
            @Override
            public void onResponse(String response) {
                //这里得到我们请求成功的结果
                Log.d("TAG", response);
            }
        }, new Response.ErrorListener() {
            @Override
            public void onErrorResponse(VolleyError error) {
                //这里得到我们请求请求失败的信息
                Log.e("TAG", error.getMessage());
            }
        });
        //将stringRequest添加到RequestQueue中
        MyApplication.getVolleyRequestQueue().add(stringRequest);
    }
```

好了，这样我们就完成完成了StringRequest请求操作，在onResponse方法内得到我们想要的请求结果String 类型数据。

## 四、使用思考
在上面使用StringRequest请求过程中，我们只需要简单三步就完成了整个请求，那么有没有想过，Volley内部是如何实现的呢，思考以下几个问题：

1. 如何实现高并发请求
2. 如何实现异步请求
3. 如何实现数据缓存
4. 内部是如何各种继承，组合封装最后几行代码就可以实现请求功能，但是使用拓展性又那么强。

相信不想只做代码搬运工的你在使用过程中也会有这些甚至更多疑问，笔者在使用过程中一直好奇这些，否则在项目中只是简单调用人家已经封装好的几行代码，总感觉自己是一个搬运工，不知其所以然，所以现在决定，静下心来，去尝试分析一下Volley源码，以此记录分享。

## 五、源码分析

首先我们来看看Volley官方给出的一张Volley工作流程图 

![“Volley工作原理如”](https://github.com/yongyu0102/yongyu0102.github.io/blob/master/images/operating%20principle.png?raw=true)

下面将这张图翻译如下：

![“Volley工作原理如”](https://github.com/yongyu0102/yongyu0102.github.io/blob/master/images/operating%20principle-cn.png?raw=true)

通过上面这张图我们可以对Volley工作流程有一个大概的印象，下面我们根据这张流程图以及Volley使用过程来结源码进行分析：

首先从我们使用的入口Volley.newRequestQueue(context)方法来来作为我们分析的切入点，代码如下：

```java
/**
 * 使用volley的入口，一般默认调用这个方法即可创建一个默认的网络请求列队，启动一个请求队列RequestQueue，
 * 只需要往这个RequestQueue不断 add Request 即可发起请求
 * @param context用于创建缓存文件夹
 * @return 返回 instance.
 */
public static RequestQueue newRequestQueue(Context context) {
    return newRequestQueue(context, null);
}
```

这个方法仅仅只有一行代码，只是调用了newRequestQueue()的方法重载，并给第二个参数传入null。那我们接着分析带有两个参数的newRequestQueue()方法中的代码，如下所示：

```java
/**
 * @param context A 用于创建缓存文件夹
 * @param stack HttpStack处理http网络请求包装，可以自己定义，如果传入null，那么就使用系统默认的HttpStack
 * @return A 返回 instance.
 */
public static RequestQueue newRequestQueue(Context context, HttpStack stack)
{
   return newRequestQueue(context, stack, -1);
}
```

这个方法也很简单，直接调用了含三个参数的构造方法，那么我们接着看看含三个参数的构造方法代码：

```java
 /**
     * @param context A 用于创建缓存文件夹
     * @param stack HttpStack处理http网络请求包装，内部就是我		*们使用过的HttpURLConnection或者HttpClient，进行包装处
     *理，可以自己定义，也可以传入null，那么就使用系统默认的		 *HttpStack
     * @param maxDiskCacheBytes 设置最大sd卡缓存,如果传入-1
     *就使用默认缓存
     * @return A 返回 instance.
     */
    public static RequestQueue newRequestQueue(Context context, HttpStack stack, int maxDiskCacheBytes) {
        //创建指定缓存文件
        File cacheDir = new File(context.getCacheDir(), DEFAULT_CACHE_DIR);
        /**
         * UA一般都用于统计与识别
         * User-Agent是Http协议中的一部分，属于头域的组成部
         *分，User Agent也简称UA。用较为普通的一点来说
         * ，是一种向访问网站提供你所使用的浏览器类型、操作系统
         *及版本、CPU 类型、浏览器渲染引擎、浏览器语言、浏览器插
         *件等信息的标识。
         * UA字符串在每次浏览器 HTTP 请求时发送到服务器
         */
        String userAgent = "volley/0";
        try {
            //获取包名
            String packageName = context.getPackageName();
            //获取包信息
            PackageInfo info = context.getPackageManager().getPackageInfo(packageName, 0);
            //获取userAgent
            userAgent = packageName + "/" + info.versionCode;
        } catch (NameNotFoundException e) {
        }
//如果传入stack参数为null，那么创建默认的stack
        if (stack == null) {
            //如果版本号大于9采用基于 HttpURLConnection 的 //HurlStack
            if (Build.VERSION.SDK_INT >= 9) {
                stack = new HurlStack();
            } else {
                //版本号小于9采用基于 HttpClient 的 //HttpClientStack
            stack = new HttpClientStack(AndroidHttpClient.newInstance(userAgent));
            }
        }
    //new一个网络请求包装对象，并传入stack
        Network network = new BasicNetwork(stack);
        RequestQueue queue;
        if (maxDiskCacheBytes <= -1)
        {
            //没有设置最大缓存maxDiskCacheBytes，创建默认大小缓存
           queue = new RequestQueue(new DiskBasedCache(cacheDir), network);
        }
        else
        {
            //设置最大缓存大小为maxDiskCacheBytes
           queue = new RequestQueue(new DiskBasedCache(cacheDir, maxDiskCacheBytes), network);
        }
//启动列队
        queue.start();
//返回列队instance
        return queue;
    }
    
```

这个方法代码稍微多一点，代码已经注释，这个构造方法主要是new 了一个 负责处理网络请求部分的stack对象，然后 用BasicNetwork将stack进行包装处理，又new了一个负责缓存部分的DiskBasedCache对象，并将这两个参数传入了列队queue中，并启动了列队；这里说明一下，在创建HttpStack对象的时候是比较了一下版本号，如果Build.VERSION.SDK_INT >= 9， stack = new HurlStack();这里是因为在Android 2.2版本之前，HttpURLConnection一直存在着一些令人厌烦的bug，在Android 2.3版本及以后，HttpURLConnection则是最佳的选择。它的API简单，体积较小，因而非常适用于Android项目。压缩和缓存机制可以有效地减少网络访问的流量。到这里我们就完成使用volley的第一步分析，即获取RequestQueue对象，那么接下来我们分析下RequestQueue构造方法的实现：

```java
/**
 * 创建请求列队，传入sd卡缓存包装类，和网络请求包装类
 * 会调用含三个参数的构造方法，并传入参数默认参数DEFAULT_NETWORK_THREAD_POOL_SIZE为4，即默认
 *线程数为4
 * @param cache sd卡缓存类
 * @param network 网络请求包装类
 */
public RequestQueue(Cache cache, Network network) {
    this(cache, network, DEFAULT_NETWORK_THREAD_POOL_SIZE);
}
```

下面分析含三个参数的构造方法代码：

```java
/**
 * 调用含四个参数的构造方法并传递默认 new ExecutorDelivery(new Handler(Looper.getMainLooper()))参数
 * @param cache sd卡缓存类
 * @param network 网络请求包装类
 * @param threadPoolSize 网络请求线程的数量，这里threadPoolSize为DEFAULT_NETWORK_THREAD_POOL_SIZE
 */
public RequestQueue(Cache cache, Network network, int threadPoolSize) {
    this(cache, network, threadPoolSize,
            new ExecutorDelivery(new Handler(Looper.getMainLooper())));
}
```

含四个参数的构造方法代码如下：

``` java
/**
 * @param cache sd卡缓存类 
 * @param network 网络请求包装类
 * @param threadPoolSize 网络请求线程的数量
 * @param delivery用来回调网络请求结果的事件分发类
 */
public RequestQueue(Cache cache, Network network, int threadPoolSize,
        ResponseDelivery delivery) {
    mCache = cache;
    mNetwork = network;
    //创建数量为DEFAULT_NETWORK_THREAD_POOL_SIZE 即数量为4的NetworkDispatcher数组
    mDispatchers = new NetworkDispatcher[threadPoolSize];
    //mDelivery为含有三个参数构造函数调用传入的new ExecutorDelivery(new 
    //Handler(Looper.getMainLooper()))对象
    mDelivery = delivery;
}
```

到这里我们来我们发现从调用含有两个参数的RequestQueue构造方法是逐渐调用三个、四个参数构造方法，那么最终实现了：mCachesd卡缓存类的实例化，network网络请求包装类 的实例化，创建了一个数量为4的NetworkDispatcher数组，mDelivery=new ExecutorDelivery(new Handler(Looper.getMainLooper())网络请求结果回调类的实例化。那么这里我们看一下回调网络请求结果类ExecutorDelivery构造方法代码：

``` java
/**
 创建一个回调网络请求结果的对象
 * @param handler 利用handler将runnable发送出去
 */
public ExecutorDelivery(final Handler handler) {
    //创建一个excuter包含了一个handler，然后利用传递进来的handler来将runnable发送出去
    mResponsePoster = new Executor() {
        @Override
        public void execute(Runnable command) {
            handler.post(command);
        }
    };
}
```

这里的handler为new Handler(Looper.getMainLooper())，即handler是创建在主线程中，所以当调用 handler.post(command)的时候，是将网络请求结果发送到主线程主去处理，这样就完成了异步任务，将子线程中网络请求的结果发送到主线程中去处理。以上就完成了构造RequestQueue对象过程中的代码分析。

下面还有一个疑问要分析就是我们在调用volley的时候执行的RequestQueue.add()方法，分析一下是如何将请求添加到缓存列队和网络请求列队，以及如何进行管理。

```java
/**
 * 该方法用于向列队中添加请求request
 * 其中包含了几种列队，起到不同的作用
 * mCurrentRequests列队，用于存储目前正在进行但是尚未完成的请求
 * mNetworkQueue网络请求列队，用于存储走网络的请求
 * mWaitingRequests 如果一个请求正在被处理并且可以被缓存，后续的相同 url 的请求，将进入此等待队列
 */
public <T> Request<T> add(Request<T> request) {
    // 将当前请求和处理请求的queue进行关联，这样当该请求结束的时候就会通知负责处理该请求的queue
    request.setRequestQueue(this);
    synchronized (mCurrentRequests) {
      //将请求添加到当前请求列队中
        mCurrentRequests.add(request);
    }
    //添加序列号
    request.setSequence(getSequenceNumber());
    //添加标记
    request.addMarker("add-to-queue");
    //如果请求不能缓存，直接添加到网络请求列队，结束方法
    if (!request.shouldCache()) {
        mNetworkQueue.add(request);
        return request;
    }
  //如果方法走到这一步说明，我们使用了缓存功能
    synchronized (mWaitingRequests) {
      //取出请求的缓存键
        String cacheKey = request.getCacheKey();
      
        if (mWaitingRequests.containsKey(cacheKey)) {
            Queue<Request<?>> stagedRequests = mWaitingRequests.get(cacheKey);
            if (stagedRequests == null) {
                stagedRequests = new LinkedList<Request<?>>();
            }
            //如果mWaitingRequests已经包含该请求，那么将该请求添加到mWaitingRequests中
            stagedRequests.add(request);
            mWaitingRequests.put(cacheKey, stagedRequests);
            if (VolleyLog.DEBUG) {
                VolleyLog.v("Request for cacheKey=%s is in flight, putting on hold.", cacheKey);
            }
        } else {
            //如果该请求不在正在执行等待的类列队mWaitingRequests中，直接添加到mCacheQueue中
            mWaitingRequests.put(cacheKey, null);
            mCacheQueue.add(request);
        }
      //返回当前请求
        return request;
    }
}
```

下面我们来捋一捋request是如何被添加到请求列队中的步骤如下：

1、当add一个请求request时候，首先该请求会被添加当当前请求列队mCurrentRequests中。

2、如果该请求不使用缓存那么直接被添加到网络请求列队mNetworkQueue结束方法。

3、如果该请求使用了缓存，那么先判断mWaitingRequests列队中是否有该请求，如果有那么添加到mWaitingRequests中，如果没有直接添加到缓存请求列队mCacheQueue中。

到这里我们理解了一个请求是如何被添加到列队中，但是同时会产生一个疑问就是mCurrentRequests和mWaitingRequests这两个列队的作用是什么，以及如何调用的，我们初略猜想一下，这两个列队中维护的请求应该在一个请求结束的时候，将该请求移除，那么应该是RequestQueue的finish方法中进行调用，那么下面我们来看一下代码：

```java
/**
 * 该方法会在Request的方法中进行调用finish(String)，表明当前请求已经结束
 * 需要从mCurrentRequests和mWaitingRequests中移除保存的请求Request
 */
<T> void finish(Request<T> request) {
    synchronized (mCurrentRequests) {
      //从当前请求列队mCurrentRequests中移除请求request
        mCurrentRequests.remove(request);
    }
    synchronized (mFinishedListeners) {
      for (RequestFinishedListener<T> listener : mFinishedListeners) {
        listener.onRequestFinished(request);
      }
    }

    if (request.shouldCache()) {
        synchronized (mWaitingRequests) {
            String cacheKey = request.getCacheKey();
            //如果请求可以使用缓存，将mWaitingRequests中存储的waitingRequests移除
            Queue<Request<?>> waitingRequests = mWaitingRequests.remove(cacheKey);
         
            if (waitingRequests != null) {
                if (VolleyLog.DEBUG) {
                    VolleyLog.v("Releasing %d waiting requests for cacheKey=%s.",
                            waitingRequests.size(), cacheKey);
                }
                 //如果移除当前结束的请求后，如果移除的waitingRequests！=null，
            // 那么将其添加到缓存列队mCacheQueue
                mCacheQueue.addAll(waitingRequests);
            }
        }
    }
}
```

上面代码只需要看注释部分即可，并不复杂，那么RequestQueue的finish应该是什么时候调用呢，我们猜想应该是在一个request方法请求结束的时候，我们来看一下在Request类的finish方法中代码是否如此：

```java
void finish(final String tag) {
  //调用处
    if (mRequestQueue != null) {
        mRequestQueue.finish(this);
    }
    if (MarkerLog.ENABLED) {
        final long threadId = Thread.currentThread().getId();
        if (Looper.myLooper() != Looper.getMainLooper()) {
            Handler mainThread = new Handler(Looper.getMainLooper());
            mainThread.post(new Runnable() {
                @Override
                public void run() {
                    mEventLog.add(tag, threadId);
                    mEventLog.finish(this.toString());
                }
            });
            return;
        }

        mEventLog.add(tag, threadId);
        mEventLog.finish(this.toString());
    }
}
```

上面代码我们在第三行看到了mRequestQueue.finish(this);那么request方法结束请求有三种可能分别是：手动调用request.cancel方法取消请求、request请求成功回调成功结果、request请求失败回调失败结果，这三种请求都是表面当前request请求结束，那么我们分别看一下这三处的代码是否进行了调用，我们以网络请求线程NetworkDispatcher为例（缓存线程调度原理相同）：

```java
//执行网络请求线程，耗时任务的执行
    @Override
    public void run() {
      ........

                if (request.isCanceled()) {
                    //如果请求取消，结束请求
                    request.finish("network-discard-cancelled");
                  //继续从列队拿请求
                    continue;
                }
          ........
               
                if (networkResponse.notModified && request.hasHadResponseDelivered()) {
                   //如果networkResponse状态为304，并且已经返回了响应结果，那么结束请求不进行第二次返回
                    request.finish("not-modified");
                  //继续请求列队拿请求
                    continue;
                }

               ..........
    }
```

然后看一下请求结果回调类ExecutorDelivery的run，执行了该方法表明将request请求结果进行了回调，所以request.finish()方法在此处也有调用：

```java
 public void run() {
        //调用处
        if (mRequest.isCanceled()) {
            mRequest.finish("canceled-at-delivery");
            return;
        }
       ........
        if (mResponse.intermediate) {
          //如果这个请求需要刷新那么这个请求又被添加到了列队中，所以还在列队中，不要finish
            mRequest.addMarker("intermediate-response");
        } else {
          //否则调用finish
            mRequest.finish("done");
        }

.......
}
```

这样mCurrentRequests和mWaitingRequests这两个列队的管理基本就捋清楚了。

下面我们就要思考获取到requestQueue对象，调用requestQueue.add（request）方法后，是如何启动网络请求的，那么我们看到在RequestQueue构造方法中有一行代码    queue.start()方法，那么接下来我们来分析一下这个方法具体内容：

``` java
/**
 * 开始执行在列队中的各个线程分发调度
 */
public void start() {
    //停止所有当前正在执行分发调度事件
    stop();  
    //创建一个执行缓存分发调度线程，并将mCacheQueue缓存列队, mNetworkQueue网络请求列队, mCache缓存对象, mDelivery事件分发对象传递进去
    mCacheDispatcher = new CacheDispatcher(mCacheQueue, mNetworkQueue, mCache, mDelivery);
    //启动缓存事件分发
    mCacheDispatcher.start();
    //当i < mDispatchers.length时循环创建networkDispatcher对象
    for (int i = 0; i < mDispatchers.length; i++) {
       //创建一个网络请求线程调度networkDispatcher线程，并将 mCacheQueue缓存列队, mNetworkQueue网络
       //请求列队, mCache负责处理缓存对象, mDelivery事件分发对象传递进去
        NetworkDispatcher networkDispatcher = new NetworkDispatcher(mNetworkQueue, mNetwork,
                mCache, mDelivery);
        mDispatchers[i] = networkDispatcher;
        //启动网络缓存线程
        networkDispatcher.start();
    }
}
```

上面代码主要实现功能是创建一个缓存事件调度线程并启动，然后循环创建了n个网络请求线程调度networkDispatcher对象，这里n= mDispatchers.length。mDispatchers数组的长度即是构造RequestQueue过程中传入的参数DEFAULT_NETWORK_THREAD_POOL_SIZE 即数量为4，也就是说创建了4个负责网络请求调度的线程。下面我们一起分析一下CacheDispatcher缓存调度线程代码，CacheDispatcher 继承自Thread，是一个线程，先看一下构造方法：

```java
/**
 * @param cacheQueue 缓存列队
 * @param networkQueue 网络请求列队
 * @param cache 缓存类
 * @param delivery 请求结果回调分发类
 */
public CacheDispatcher(
        BlockingQueue<Request<?>> cacheQueue, BlockingQueue<Request<?>> networkQueue,
        Cache cache, ResponseDelivery delivery) {
    mCacheQueue = cacheQueue;
    mNetworkQueue = networkQueue;
    mCache = cache;
    mDelivery = delivery;
}
```

然后主要看一下run方法：

```java
//复写Run方法，这里执行耗时操作，在子线程中执行
@Override
public void run() {
    if (DEBUG) VolleyLog.v("start new dispatcher");
    //设置线程优先级
 Process.setThreadPriority(Process.THREAD_PRIORITY_BACKGROUND);
    //初始化缓存类
    mCache.initialize();
  //循环从列队中读取请求并执行
    while (true) {
        try {
            //从列队的首部取出一个request
            final Request<?> request = mCacheQueue.take();
            //添加一个标记
            request.addMarker("cache-queue-take");
            //如果取出的请求取消，那么结束请求
            if (request.isCanceled()) {
                request.finish("cache-discard-canceled");
              //继续循环从缓存列队中拿请求
                continue;
            }

            //从缓存中读取缓存数据
            Cache.Entry entry = mCache.get(request.getCacheKey());
            if (entry == null) {
                //如果请求结果为null，那么添加标记
                request.addMarker("cache-miss");
                 //然后将请求发送到网络请求列队
                mNetworkQueue.put(request);
               //然后继续循环去缓存列队中拿取请求
                continue;
            }

            if (entry.isExpired()) {
                //如果缓存过期，添加标记
                request.addMarker("cache-hit-expired");
                request.setCacheEntry(entry);
              //将请求发送到网络请求列队
                mNetworkQueue.put(request);
              //然后继续去缓存列队中拿取请求
                continue;
            }

            //走到这一步，说明已经拿到缓存，添加标记命中
            request.addMarker("cache-hit");
            //解析网络缓存数据NetworkResponse得到Response<T> 结果
            Response<?> response = request.parseNetworkResponse(
                    new NetworkResponse(entry.data, entry.responseHeaders));
            //标记解析完成
            request.addMarker("cache-hit-parsed");
            if (!entry.refreshNeeded()) {
                //如果不需要刷新，那么直接将解析结果回调
                mDelivery.postResponse(request, response);
            } else {
                //如果需要刷新，那么添加标记
                request.addMarker("cache-hit-refresh-needed");
                request.setCacheEntry(entry);
                response.intermediate = true;
                //返回数据，并将将请求添加到网络请求列队，但是是发送出去到主线程中执行
                mDelivery.postResponse(request, response, new Runnable() {
                    @Override
                    public void run() {
                        try {
                            mNetworkQueue.put(request);
                        } catch (InterruptedException e) {
                        }
                    }
                });
            }

        } catch (InterruptedException e) {
            //调用mQuit方法结束循环
            if (mQuit) {
                return;
            }
          //否则继续缓存列队中取出request
            continue;
        }
    }
}
```

这个run方法主要作用是：启动后会不断从缓存请求队列中取请求处理，队列为空则等待，请求处理结束则将结果传递给`ResponseDelivery` 去执行，将请求结果发送到主线程中。当结果未缓存过、缓存失效或缓存需要刷新的情况下，该请求都需要重新进入`NetworkDispatcher`去调度处理。这里有一行代码mDelivery.postResponse(request, response);是将请求结果发送到主线程中，前面我们提到是利用handler.post()方法执行，那么我们看看ExecutorDelivery 类的postResponse的具体细节：

```java
@Override
public void postResponse(Request<?> request, Response<?> response) {
    //直接调用三个参数的post方法
    postResponse(request, response, null);
}
```

就一行代码，调用含三个参数的构造方法，那么我们继续追溯：

```java
@Override
public void postResponse(Request<?> request, Response<?> response, Runnable runnable) {
    request.markDelivered();
    request.addMarker("post-response");
    //通过ResponseDeliveryRunnable将request, response, runnable包装后发送出去到主线程handle中
    //
    mResponsePoster.execute(new ResponseDeliveryRunnable(request, response, runnable));
}
```

这里的mResponsePoster就是在构造ExecutorDelivery 时候生成的Executor对象，代码如下：

```java
public ExecutorDelivery(final Handler handler) {
    // Make an Executor that just wraps the handler.
    //创建一个excuter包含了一个handler，handler来将runnable发送出去
    mResponsePoster = new Executor() {
        @Override
        public void execute(Runnable command) {
          //将Runnable对象发送到主线程
            handler.post(command);
        }
    };
}
```

而ResponseDeliveryRunnable继承Runnable，所以当调用mResponsePoster.execut（new ResponseDeliveryRunnable(request, response, runnable)）方法时候就会将ResponseDeliveryRunnable对象发送到主线程中，那么我们看一下ResponseDeliveryRunnable类的run方法：

```java
//在主线程中运行
        @SuppressWarnings("unchecked")
        @Override
        public void run() {
            if (mRequest.isCanceled()) {
              //如果请求取消，那么结束方法
                mRequest.finish("canceled-at-delivery");
                return;
            }

            if (mResponse.isSuccess()) {
              //如果请求成功调用mRequest.deliverResponse，这个方法在我们自定义request时候需要重写的，每一条网络请求的响应都是回调到这个方法中，主线程中运行
            mRequest.deliverResponse(mResponse.result);
            } else {
              //请求失败回调
                mRequest.deliverError(mResponse.error);
            }
            if (mResponse.intermediate) {
                mRequest.addMarker("intermediate-response");
            } else {
                mRequest.finish("done");
            }
            if (mRunnable != null) {
              //如果传入Runnable不为空，那么执行Runnable
                mRunnable.run();
            }
       }
    }
```

这段代码我们看自定义Request时候需要复写的方法   mRequest.deliverResponse(mResponse.result);和mRequest.deliverError(mResponse.error)。到这里请求回调的疑问我们已经解决。

下面我们先分析一下NetworkDispatcher网络缓存线程，NetworkDispatcher 继承自Thread，也就是说NetworkDispatcher 是一个线程先看一下构造方法：

```java
/**
 * @param queue 网络请求列队
 * @param network 网络请求包装类
 * @param cache 缓存类
 * @param delivery 请求结果回调分发类
 */
public NetworkDispatcher(BlockingQueue<Request<?>> queue,
        Network network, Cache cache,
        ResponseDelivery delivery) {
    mQueue = queue;
    mNetwork = network;
    mCache = cache;
    mDelivery = delivery;
}
```

然后看一下复写的run方法：

```java
//执行网络请求线程，耗时任务的执行,在子线程中执行
    @Override
    public void run() {
      //设置线程优先级
 Process.setThreadPriority(Process.THREAD_PRIORITY_BACKGROUND);
        Request<?> request;
      //从网络请求列队中循环读取request并执行请求
        while (true) {
            long startTimeMs = SystemClock.elapsedRealtime();
            request = null;
            try {
                //从网络请求列队中拿到一个请求
                request = mQueue.take();
            } catch (InterruptedException e) {
                //如果调用了mQuit就直接结束该方法
                if (mQuit) {
                    return;
                }
              //否则继续从列队中拿请求
                continue;
            }

            try {
                //添加拿到标记
                request.addMarker("network-queue-take");
                if (request.isCanceled()) {
                    //如果请求取消，结束请求
                    request.finish("network-discard-cancelled");
                  //继续从列队拿请求
                    continue;
                }
                //设置线程状态
                addTrafficStatsTag(request);
                //执行网络请求，得到请求结果NetworkResponse
                NetworkResponse networkResponse = mNetwork.performRequest(request);
                //添加执行请求完成状态
                request.addMarker("network-http-complete");
               
                if (networkResponse.notModified && request.hasHadResponseDelivered()) {
                   //如果networkResponse状态为304，并且已经传递了响应结果，那么结束请求不进行第二次返回
                    request.finish("not-modified");
                  //继续请求列队拿请求
                    continue;
                }
                //将networkResponse解析成  Response<?> response，这个解析方法我们可以自己在
              //Request类进行复写，自己定义解析方法
                Response<?> response = request.parseNetworkResponse(networkResponse);
              //添加解析完成标记
                request.addMarker("network-parse-complete");
                if (request.shouldCache() && response.cacheEntry != null) {
                    //如果请求使用了缓存，并且响应结果不为空，那么将请求结果存入缓存
                    mCache.put(request.getCacheKey(), response.cacheEntry);
                  //添加完成缓存标记
                    request.addMarker("network-cache-written");
                }
                //添加请求结果回调标记
                request.markDelivered();
              //回调响应结果，即将请求响应结果，发送到主线程
                mDelivery.postResponse(request, response);
            } catch (VolleyError volleyError) {
                volleyError.setNetworkTimeMs(SystemClock.elapsedRealtime() - startTimeMs);
              //请求网络失败回调
                parseAndDeliverNetworkError(request, volleyError);
            } catch (Exception e) {
                VolleyLog.e(e, "Unhandled exception %s", e.toString());
                VolleyError volleyError = new VolleyError(e);
                volleyError.setNetworkTimeMs(SystemClock.elapsedRealtime() - startTimeMs);
              //请求失败回调
                mDelivery.postError(request, volleyError);
            }
        }
    }
```

run方法启动后会不断从网络请求队列中取请求处理，队列为空则等待，拿到请求后，则执行网络请求，请求处理结束则将结果传递给 ResponseDelivery 去执行后续处理，并判断结果是否要进行缓存。

下面我们来捋一捋volley如如何进行网络请求、缓存请求和请求结果的回调，步骤如下：

1、创建了一个缓存事件调度线程CacheDispatcher，负责从缓存列队中读取请求，然后从缓存中读取数据，进行解析返回，从缓存中读取数据为null，或者数据过期、需要刷新，那么将请求添加到网络请求列队。

2、创建了四个网络请求事件调度线程NetworkDispatcher，负责处理网络请求，并解析请求结果进行返回，并将结果缓存到本地。

3、创建了一个请求结果回调类ResponseDelivery，负责将请求结果返回到主线程，原理是利用handler将结果post到主线程。

到这里volley框架的主要流程我们基本梳理通顺了，下面我们在看一下细节方面，网络请求是如何执行的，上面代码中执行网络请求代码mNetwork.performRequest(request)，其中mNetwork就是我们在构造RequestQueue时候传入的BasicNetwork对象，那么看一下具体代码如下：

```java
/**
 * 通过HttpStack，即执行http网络请求请求
 * 并将HttpStack请求结果包装成NetworkResponse返回NetworkRespons
 */
@Override
public NetworkResponse performRequest(Request<?> request) throws VolleyError {
    long requestStart = SystemClock.elapsedRealtime();
    while (true) {
        //http请求头文件包装信息
        HttpResponse httpResponse = null;
        //请求返回内容body
        byte[] responseContents = null;
        //生成一个map对象
        Map<String, String> responseHeaders = Collections.emptyMap();
        try {
            Map<String, String> headers = new HashMap<String, String>();
            //添加请求头文件信息，entry即是定义request类时候复写setCacheEntry传入
            addCacheHeaders(headers, request.getCacheEntry());
            //执行mHttpStack操作返回请求结果httpResponse
            httpResponse = mHttpStack.performRequest(request, headers);
            //获取到请求状态
            StatusLine statusLine = httpResponse.getStatusLine();
            int statusCode = statusLine.getStatusCode();
            //取出httpResponse中header添加到responseHeaders
            responseHeaders = convertHeaders(httpResponse.getAllHeaders());
            //如果是304状态没有更新状态
            if (statusCode == HttpStatus.SC_NOT_MODIFIED) {
                //缓存中取出entry对象
                Entry entry = request.getCacheEntry();
                if (entry == null) {
                    //如果entry为null，返回一个entry为null的NetworkResponse
                    return new NetworkResponse(HttpStatus.SC_NOT_MODIFIED, null,
                            responseHeaders, true,
                            SystemClock.elapsedRealtime() - requestStart);
                }

                entry.responseHeaders.putAll(responseHeaders);
                //如果entry不为null，返回一个entry的NetworkResponse
                return new NetworkResponse(HttpStatus.SC_NOT_MODIFIED, entry.data,
                        entry.responseHeaders, true,
                        SystemClock.elapsedRealtime() - requestStart);
            }
            
            //301/302 Moved Permanently（重定向）请求的URL已移走。Response中应该包含一个Location URL, 说明资源现在所处的位置
            if (statusCode == HttpStatus.SC_MOVED_PERMANENTLY || statusCode == HttpStatus.SC_MOVED_TEMPORARILY) {
               String newUrl = responseHeaders.get("Location");
               request.setRedirectUrl(newUrl);
            }

            if (httpResponse.getEntity() != null) {
                //获取entry不为null,将entry转换为byte类型
              responseContents = entityToBytes(httpResponse.getEntity());
            } else {
                //否则new 一个responseContents对象
              responseContents = new byte[0];
            }

            long requestLifetime = SystemClock.elapsedRealtime() - requestStart;
            //打印请求生命周期，请求，响应内容，和状态
            logSlowRequests(requestLifetime, request, responseContents, statusLine);
            //根据状态码抛出异常
            if (statusCode < 200 || statusCode > 299) {
                throw new IOException();
            }
            //返回包含响应entry和header的NetworkResponse
            return new NetworkResponse(statusCode, responseContents, responseHeaders, false,
                    SystemClock.elapsedRealtime() - requestStart);
            //抛出各种异常
        } catch (SocketTimeoutException e) {
            attemptRetryOnException("socket", request, new TimeoutError());
        ...........
}
```

这段方法中大多都是一些网络请求细节方面的东西，部分已经添加注释，我们抓重点看即可，其中httpResponse = mHttpStack.performRequest(request, headers);这一行代码中的mHttpStack就是我们在构造RequestQueue的时候传入的HttpStack对象，负责处理http网络请求的，然后就是有几处代码是return new NetworkResponse(HttpStatus.SC_NOT_MODIFIED, null,        responseHeaders, true,        SystemClock.elapsedRealtime() - requestStart))，其中区别就是参数不同，所以这个方法其实只要实现功能就是调用`HttpStack`处理网络请求，并将结果进行包装转换为可被`ResponseDelivery`处理的`NetworkResponse`对象返回。

下面我们看一下mHttpStack.performRequest(request, headers)这个方法是如何处理网络请求的（以HurlStack类为例）：

```java
/**
 * 封装执行HttpURLConnection网络请求，返回HttpResponse对象
 * @param request the request to perform
 * @param additionalHeaders 附加的请求头文件信息
 */
@Override
public HttpResponse performRequest(Request<?> request, Map<String, String> additionalHeaders)
        throws IOException, AuthFailureError {
    //获取网络请求url
    String url = request.getUrl();
    //new 一个map对象
    HashMap<String, String> map = new HashMap<String, String>();
    //将请求header信息添加进去
    map.putAll(request.getHeaders());
    map.putAll(additionalHeaders);
    if (mUrlRewriter != null) {
        String rewritten = mUrlRewriter.rewriteUrl(url);
        if (rewritten == null) {
            throw new IOException("URL blocked by rewriter: " + url);
        }
        //获取到url
        url = rewritten;
    }
    //解析url
    URL parsedUrl = new URL(url);
    //创建HttpURLConnection网络请求连接
    HttpURLConnection connection = openConnection(parsedUrl, request);
    for (String headerName : map.keySet()) {
        //添加请求头信息
        connection.addRequestProperty(headerName, map.get(headerName));
    }
    //设置连接参数，主要设置各种请求类型
    setConnectionParametersForRequest(connection, request);
  ........
    //解析请求body请求体
    response.setEntity(entityFromConnection(connection));
    for (Entry<String, List<String>> header : connection.getHeaderFields().entrySet()) {
        if (header.getKey() != null) {
            //解析请求header添加请求头
            Header h = new BasicHeader(header.getKey(), header.getValue().get(0));
            response.addHeader(h);
        }
    }
  //请求结果返回
    return response;
}
```

这段代码中我们终于看见了我们熟悉的HttpURLConnection对象，并进行了一系列的参数设置，解析返回数据返回HttpResponse对象。到这里负责执行网络请求部分的内容我们也梳理结束。

至此，关于volley的学习笔记到此结束，如有理解错误地方，还请指教！！



