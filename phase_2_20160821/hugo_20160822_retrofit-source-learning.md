title: 框架源码 -- Retrofit 简析学习

date: 2016-08-20 17:03:37
---

看过很多篇 Retrofit 的源码分析文章，但是别人一问起来总是讲不清楚到底 Retrofit 是怎么个流程，所以还是得自己亲自去看看源码，一步一步的分析。果然只有亲自动手实践，才有自己的收获。
告诫自己，__慢慢来，会很快。__

<!-- more -->

> - 文章来源：itsCoder 的 [WeeklyBolg](https://github.com/itsCoder/weeklyblog) 项目
>
>
> - itsCoder主页：[http://itscoder.com/](http://itscoder.com/)
> - 作者：[谢三弟](http://imxie.cc)
> - 审阅者：[Joe](http://extremej.itscoder.com/)

### 目录

- [目录](#目录)
- [Retrofit 简介](#Retrofit-简介)
- [Retrofit 分析](#Retrofit-分析)
  - [具体使用](#具体使用)
  - [工具箱：Retrofit.Builder()](#工具箱：Retrofit-Builder)
  - [外壳：Create()](#外壳：Create)
  - [结构：ServiceMethod](#结构：ServiceMethod)
  - [子弹：xxxFactory()](#子弹：xxxFactory)
  - [开枪打靶: Call.enqueue()](#开枪打靶-Call-enqueue)
- [参考](#参考)


### Retrofit 简介

Retrofit 源码开头的解释

```java
* Retrofit adapts a Java interface to HTTP calls by using annotations on the declared methods to
* define how requests are made. Create instances using {@linkplain Builder
* the builder} and pass your interface to {@link #create} to generate an implementation.
```

Retrofit 利用方法上的注解将接口转化成一个 HTTP 请求。

简单知道是什么了之后，我们对此提出疑问：

- 如何将接口转换为网络请求？
- 谁去进行网络请求？

接下来我们将从 Retrofit 的使用作为入口分析。

### Retrofit 分析

#### 具体使用

首先建立 API 接口类：

```java
interface GankApi {
    String host = "http://gank.io/api/data/";
    @GET("Android/10/{page}")
    Call<Android> getAndroid(@Path("page") int page);
}
```

```java

// 创建 Retrofit 实例
Retrofit retrofit = new Retrofit.Builder()
    .baseUrl(GankApi.host)
    .addConverterFactory(GsonConverterFactory.create())
    .build();

// 生成接口实现类
GankApi gankApi = retrofit.create(GankApi.class);

// 调用接口定义的请求方法，并且返回 Call 对象
Call<Android> call = gankApi.getAndroid(1);

// 调用 Call 对象的异步执行方法
call.enqueue(Callback callback)

```

简单的使用就是这样的流程。现在我们开始层层剖析。

#### 工具箱：Retrofit.Builder()

```java
private Platform platform;
private okhttp3.Call.Factory callFactory;
private HttpUrl baseUrl;
private List<Converter.Factory> converterFactories = new ArrayList<>();
private List<CallAdapter.Factory> adapterFactories = new ArrayList<>();
private Executor callbackExecutor;
private boolean validateEagerly;
```
创建 Retrofit 的实例，进行一些配置，这里我们不用多说。但是有一个参数必须得讲讲。

- __Platform__

在构建 Retrofit 的时候，会对当前使用平台进行判断，Java8，Android，IOS。

我们看看 Android 平台的代码：
```java
static class Android extends Platform {
  @Override public Executor defaultCallbackExecutor() {
    return new MainThreadExecutor();
  }

  @Override CallAdapter.Factory defaultCallAdapterFactory(Executor callbackExecutor) {
    return new ExecutorCallAdapterFactory(callbackExecutor);
  }

  static class MainThreadExecutor implements Executor {
    private final Handler handler = new Handler(Looper.getMainLooper());

    @Override public void execute(Runnable r) {
      handler.post(r);
    }
  }
}
```
从代码中我们得知两点：
1. 在 Android 里我们默认使用的 CallAdapter 是 `ExecutorCallAdapterFactory()` 它会返回的是 Call.class。关于 `ExecutorCallAdapterFactory()` 我们稍后再说，你先知道这是 Android 默认 CallAdapter 就好。
2. 默认的 Callback 是在主线程。


#### 外壳：Create()

```java
// 生成接口实现类
GankApi gankApi = retrofit.create(GankApi.class);
```

我在源码里写好了注释：

```java
public <T> T create(final Class<T> service) {
    // 检查传入的类是否为接口并且无继承
    Utils.validateServiceInterface(service);
    if (validateEagerly) {
      eagerlyValidateMethods(service);
    }
    // 重点是这里
    // 首先会返回一个利用代理实现的 GankApi 对象
    return (T) Proxy.newProxyInstance(service.getClassLoader(), new Class<?>[] { service },
        new InvocationHandler() {
          private final Platform platform = Platform.get();

          // 我们调用该对象的方法都会进入到这里
          @Override public Object invoke(Object proxy, Method method, Object... args)
              throws Throwable {
            // If the method is a method from Object then defer to normal invocation.
            if (method.getDeclaringClass() == Object.class) {
              return method.invoke(this, args);
            }
            if (platform.isDefaultMethod(method)) {
              return platform.invokeDefaultMethod(method, service, proxy, args);
            }
            // 解析方法 这里用到了注解（Runtime）这里我们标记下（A）稍后来看看里面具体实现
            ServiceMethod serviceMethod = loadServiceMethod(method);
            // 将刚刚解析完毕包装后的具体方法封装成 OkHttpCall ，你可以在该实现类找到 okhttp 请求所需要的参数
            // 所以它是用来跟 okhttp 对接的。
            OkHttpCall okHttpCall = new OkHttpCall<>(serviceMethod, args);
            // 将以上我们封装好的 call 返回给上层，这个时候我们就可以执行 call 的同步方法或者异步进行请求。
            return serviceMethod.callAdapter.adapt(okHttpCall);
          }
        });
  }
```

切合我们实际运用来看看顺序：

`GankApi gankApi = retrofit.create(GankApi.class);`—->
 `return (T) Proxy.newProxyInstance（...）{...}`—->
 `Call<Android> call = gankApi.getAndroid(1);` —->
  `public Object invoke(...){...}` 调用代理类的` invoke()`。

  直到这里我们已经宏观地了解 Retrofit 是怎样的一个流程。
  达成 __初窥门径__ 成就。

千万别骄傲，为了以后走的更远更稳，我们得好好筑基，上面我们用到的是动态代理，强烈建议认真阅读两篇文章。

- [Retrofit2源码分析[动态代理]](http://www.jianshu.com/p/a56c61da55dd)
- [Java静态代理和动态代理](http://blog.csdn.net/giserstone/article/details/17199755)



#### 结构：ServiceMethod

Retrofit 有一个双链表用来缓存方法
`private final Map<Method, ServiceMethod> serviceMethodCache = new LinkedHashMap<>();`
```java
  ServiceMethod loadServiceMethod(Method method) {
  ServiceMethod result;
  synchronized (serviceMethodCache) {
      // 从缓存中获取该方法
    result = serviceMethodCache.get(method);
    if (result == null) {
        // 没有就进行创建并且存入链表缓存
      result = new ServiceMethod.Builder(this, method).build();
      serviceMethodCache.put(method, result);
    }
  }
  return result;
}
```
我们发现主要的方法是 `new ServiceMethod.Builder(this, method).build();` ，所以接下来我们深入看看如何 __解析注解__ 以及 __构建请求方法__ 。

- 初始化一些参数

```java
public Builder(Retrofit retrofit, Method method) {
  this.retrofit = retrofit;
  this.method = method;
  this.methodAnnotations = method.getAnnotations();
  this.parameterTypes = method.getGenericParameterTypes();
  this.parameterAnnotationsArray = method.getParameterAnnotations();
}
```


- `build()`

这里的源码很长，做了很多异常处理，我截取重点来分析下。

```java
callAdapter = createCallAdapter();
responseConverter = createResponseConverter();
```

一个是用来发送请求的 client ，一个是结果的转换器（Gson，FastJson ...）之类，后面我们再讲这个。
上层配置就是当我们调用 Retrofit 的 `addConverterFactory()`和 `addCallAdapterFactory()`，内部会自动使用我们定义的组件。

```java
for (Annotation annotation : methodAnnotations) {
  parseMethodAnnotation(annotation);
}
```
在这里可以看到遍历我们使用方法的注解，并且解析他们。`parseMethodAnnotation()` 内部就是解析好 HTTP 的请求方式。

为了篇幅大小，可以在 [源码](https://github.com/square/retrofit/blob/master/retrofit/src/main/java/retrofit2/ServiceMethod.java) 里看看具体的操作。

同时也可以看看 http 包下注解用到的接口，你会发现 `@Retention(RUNTIME)` 所以，从这里我们就可以明白，Retrofit 是在在运行期通过反射访问到这些注解的。


- `return Call`

请求方法参数，请求客户端，返回值转换，我们都定义好了之后，便完成最后一步，构建好适合请求客户端的请求方法，Retrofit 默认的是 okhttpCall 。

```java
OkHttpCall okHttpCall = new OkHttpCall<>(serviceMethod, args);
return serviceMethod.callAdapter.adapt(okHttpCall);
```

最后将 call 返回给上层，用户调用方法进行请求。

- 总结

```java
/** Adapts an invocation of an interface method into an HTTP call. */
```
ServiceMethod 类开头注释已经很清楚的说明了作用，将接口方法改变成一个 HTTP call 。它对于 Retrofit 是很重要的存在，整个枪支内部都是由它来支撑起来。

#### 子弹：xxxFactory()

Retrofit 给我们最大的便利就是自身框架优雅的设计，只需要很小的改动，便可以优雅的适应不同的需求。所以很需要我们再补充点额外知识，了解什么是[适配器模式](http://blog.csdn.net/zhangjg_blog/article/details/18735243)，然后回到这里看看 Retrofit 是如何应用的。



在构建 `ServiceMethod` 对象的时候，有三个方法可以单独说说
1. `build()` 中 `createCallAdapter()` —-> `retrofit.callAdapter()`
2. 解析接口方法内注解时`parseParameterAnnotation()`调用到的`retrofit.requestBodyConverter()`
3. `build()` 中 `createResponseConverter()` —-> `retrofit.responseBodyConverter()`

> callAdapter()

最终会调用到 `nextCallAdapter()` 该方法主要是从 callAdapterFactories 中获取新的 CallAdapter，它会跳过 skipPast，以及 skipPast 之前的 Factory，然后找到与 returnType 和 annotations 都匹配的 CallAdapterFactory 。

> requestBodyConverter() & responseBodyConverter()

最终会调用到 `nextRequestBodyConverter()/nextResponseBodyConverter`利用 converterFactories 创建一个与 RequestBody/ResponseBody 对应的 Converter 对象。

所以在这里我们就可以装填我们需要的子弹类型了。

__进入实战，为我们的 Retrofit 添加 RxJava 和 Gson。__

- Rxjava:

![23:33:50.jpg](http://ww4.sinaimg.cn/large/006tNbRwgw1f71shdblvjj30wi08igms.jpg)

 __adapter-rxjava__ 我们重点看 RxJavaCallAdapterFactory 即可，它是实现了 CallAdapter.Factory 并在对应方法里将 Call 包装成 Observable.class 返回。
 然后给 Retrofit 对象加上 `.addCallAdapterFactory(RxJavaCallAdapterFactory.create())`，这样我们才可以优雅的使用 Retrofit + RxJava 。



- Gson:

![23:34:46.jpg](http://ww3.sinaimg.cn/large/006tNbRwgw1f71si9z928j30x607adh0.jpg)

我相信通过类名我们就可以知道每个类是用来做什么的，我在这里太过深入到具体实现反而一叶障目，
如果我们需要自定义数据转换格式，也是同样这样做。
继承 `Converter.Factory` 类作为适配类，同时创建两个实现 `Converter` 的类包装请求和响应的数据形式。

#### 开枪打靶: Call.enqueue()

<div class="tip">
注意：我这里只列举一个默认状态下的情况
</div>

还记得我工具箱里我们提到的 `ExecutorCallbackCall` 吗？
这里的 Call 是对应我们选择的 call ，而此时是默认的 `ExecutorCallbackCall` 。如果还要问我为什么，请去看看 [工具箱：Retrofit.Builder()](#工具箱：Retrofit-Builder) 里 Android 平台的源码。



```java
static final class ExecutorCallbackCall<T> implements Call<T> {
  final Executor callbackExecutor;
  final Call<T> delegate;

  ExecutorCallbackCall(Executor callbackExecutor, Call<T> delegate) {
    this.callbackExecutor = callbackExecutor;
    this.delegate = delegate;
  }

  @Override public void enqueue(final Callback<T> callback) {
    if (callback == null) throw new NullPointerException("callback == null");

    delegate.enqueue(new Callback<T>() {
      @Override public void onResponse(Call<T> call, final Response<T> response) {
        callbackExecutor.execute(new Runnable() {
          @Override public void run() {
            if (delegate.isCanceled()) {
              // Emulate OkHttp's behavior of throwing/delivering an IOException on cancellation.
              callback.onFailure(ExecutorCallbackCall.this, new IOException("Canceled"));
            } else {
              callback.onResponse(ExecutorCallbackCall.this, response);
            }
          }
        });
      }

      // 省略
}
```

这里的 delegate 对应的就是 okhttp 的 call ，不禁有疑问了，这里调用的是异步请求，但是我们的回调是怎么回到主线程的呢？

带着疑问我们来看看。
首先回调是在 `callbackExecutor.execute()` 我们从这里入手。
我们发现在 Retrofit 的 `build()` 方法里：

```java
Executor callbackExecutor = this.callbackExecutor;
if (callbackExecutor == null) {
  callbackExecutor = platform.defaultCallbackExecutor();
}
```

平台默认的回调调度器，连忙回到工具箱看看：

```java
static class Android extends Platform {
  @Override public Executor defaultCallbackExecutor() {
    return new MainThreadExecutor();
  }

  @Override CallAdapter.Factory defaultCallAdapterFactory(Executor callbackExecutor) {
    return new ExecutorCallAdapterFactory(callbackExecutor);
  }

  static class MainThreadExecutor implements Executor {
    private final Handler handler = new Handler(Looper.getMainLooper());

    @Override public void execute(Runnable r) {
      handler.post(r);
    }
  }
}
```
我们发现，Android 默认的调度器是主线程的 Handler ，`execute()`方法也只是 `mainHandler.post()` 。

所以这下就可以解决我们的疑问了。

```java
callbackExecutor.execute(new Runnable() {
  @Override public void run() {
    if (delegate.isCanceled()) {
      // Emulate OkHttp's behavior of throwing/delivering an IOException on cancellation.
      callback.onFailure(ExecutorCallbackCall.this, new IOException("Canceled"));
    } else {
      callback.onResponse(ExecutorCallbackCall.this, response);
    }
  }
});
```
这段代码我们就可以改写为：

```java
mainHandler.post(new Runnable() {
  @Override public void run() {
    if (delegate.isCanceled()) {
      // Emulate OkHttp's behavior of throwing/delivering an IOException on cancellation.
      callback.onFailure(ExecutorCallbackCall.this, new IOException("Canceled"));
    } else {
      callback.onResponse(ExecutorCallbackCall.this, response);
    }
  }
});
```

如果看到这，还不理解为什么那就得好好补补 handler 的知识啦！

我这里推荐 melo 写的这篇，风趣易懂 [带着这篇去通关所有Handler的提问](http://www.jianshu.com/p/fad4e2ae32f5) 。



__以上。〃´∀`)__


### 参考

- [Retrofit2源码分析[动态代理]](http://www.jianshu.com/p/a56c61da55dd)
- [Java静态代理和动态代理](http://blog.csdn.net/giserstone/article/details/17199755)
- [一个示例让你明白适配器模式](http://blog.csdn.net/zhangjg_blog/article/details/18735243)
- [Retrofit分析-漂亮的解耦套路](http://www.jianshu.com/p/45cb536be2f4)
- [我对Retrofit的认识](http://blog.fangjie.info/2016/07/14/%E6%88%91%E5%AF%B9Retrofit%E7%9A%84%E8%AE%A4%E8%AF%86/)
- [Retrofit 源码分析](https://github.com/android-cn/android-open-project-analysis/tree/master/tool-lib/network/retrofit)
- [Java 注解](http://wiki.jikexueyuan.com/project/java-reflection/java-at.html)
