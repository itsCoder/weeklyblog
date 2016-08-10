---
title: rxjava retrofit2配合MVP实现豆瓣图书展示
date: 2016-08-05 20:14:19
categories: Android
tags: [Android ,MVP ,Retrofit2, Rxjava]
---
刚接触rxjava和retrofit，加上自己对MVP架构的一点理解，就结合起来实践一下。豆瓣图书api个人版是免费试用的，不过有访问限制，你可以去[这里](https://developers.douban.com/wiki/?title=api_v2)了解更多。本文将展示的是一个小项目的开始工作，而不仅仅是一个小demo(什么意思？)，这意味着图书列表的UI不会仅用一个TextView来示意一下(⊙o⊙)…，那样就太demo了。

![book item](http://ww3.sinaimg.cn/mw690/005X6W83gw1f6p032la2tj30dw04vq4g.jpg)

最终效果图：

![demo](http://ww4.sinaimg.cn/mw690/005X6W83gw1f6p0339jkbj30c00lcn3f.jpg)

## 一 .  准备工作
第一步不急着写代码，先把项目目录整理一下，当然对于rxjava和retrofit的实践可以把所有代码全写在一个文件里面，但是这样就很不清真了。项目目录如下：
```
    ---app
    ----api
    -----common
    -----model
    -----presenter
    -----view
    ----bean
    ----common
    ----ui
    -----acitvity
    -----adapter
    -----fragment
    -----widget
    ----utils
    ----BaseApplication
```
然后就是项目依赖库：
```
    //Butter Knife
    compile 'com.jakewharton:butterknife:8.0.1'
    apt 'com.jakewharton:butterknife-compiler:8.0.1'
    //RxJava & RxAndroid
    compile 'io.reactivex:rxandroid:1.2.1'
    compile 'io.reactivex:rxjava:1.1.6'
    //okhttp3 & okio
    compile 'com.squareup.okhttp3:okhttp:3.4.1'
    compile 'com.squareup.okio:okio:1.9.0'
    //retrofit2
    compile 'com.squareup.retrofit2:retrofit:2.1.0'
    compile 'com.squareup.retrofit2:converter-gson:2.1.0'
    compile 'com.squareup.retrofit2:adapter-rxjava:2.1.0'

    //Glide
    compile 'com.github.bumptech.glide:glide:3.7.0'
    compile 'com.github.bumptech.glide:okhttp3-integration:1.4.0@aar'
    //gson
    compile 'com.google.code.gson:gson:2.7'
```
！整个目录结构如下图：

![project tree](http://ww2.sinaimg.cn/mw690/005X6W83gw1f6p033lyt6j30990i5q3e.jpg)

！你可以直接去github主页[**查看源码**](https://github.com/Hymanme/MaterialHome)
##二.  分析豆瓣图书接口
通过豆瓣图书开发者文档可以得到图书列表接口
Url：`https://api.douban.com/v2/book/search?tag=热门&start=0&count=20&fields=`
我们写一个公共类URL.java存放这些url常量。
```java 
    public static final String HOST_URL_DOUBAN = "https://api.douban.com/v2/";
```

##三.  获取图书列表的service
通过以上已经获得了restful url，接下来用retrofit库的注解方法写一个获取图书列表的service
```java
    public interface IBookService {
        @GET("book/search")
        Observable<Response<BookListResponse>> getBookList(
            @Query("q") String q, 
            @Query("tag") String tag, 
            @Query("start") int start, 
            @Query("count") int count, 
            @Query("fields") String fields);
    }
```

##四.  使用Observable完成一次网络请求
首先需要初始化Retrofit对象，然后用Retrofit对象使用反向代理去创建service。Retrofit默认是使用okhttp请求网络，如果我们需要做一些配置需要构建一个自己的okhttpClient，然后设置给Retrofit，如果你想对网络请求进行log输出或者是设置自己的缓存机制，好在okhttp有拦截器，你可以使用拦截器对网络请求进行监控。
>构建Retrofit

```java
public class ServiceFactory {
    private volatile static OkHttpClient okHttpClient;
    private static final int DEFAULT_CACHE_SIZE = 1024 * 1024 * 20;
    private static final int DEFAULT_MAX_AGE = 60 * 60;
    private static final int DEFAULT_MAX_STALE = DEFAULT_MAX_AGE * 24 * 7;
    //构建okhttpClient对象
    public static OkHttpClient getOkHttpClient() {
        if (okHttpClient == null) {
            synchronized (OkHttpClient.class) {
                if (okHttpClient == null) {
                    File cacheFile = new File(comtext.getCacheDir(), "responses");
                    Cache cache = new Cache(cacheFile, DEFAULT_CACHE_SIZE);
                    okHttpClient = new OkHttpClient.Builder()
                            .cache(cache)
                            .addInterceptor(CACHED_INTERCEPTOR)
                            .addInterceptor(LoggingInterceptor)
                            .build();
                }
            }
        }
        return okHttpClient;
    }
    //缓存拦截器
    private static final Interceptor CACHED_INTERCEPTOR = chain -> {
        Request request = chain.request();
        if (!NetworkUtils.isConnected(BaseApplication.getApplication())) {
            request = request.newBuilder()
                    .cacheControl(CacheControl.FORCE_CACHE)
                    .build();
        }
        Response response = chain.proceed(request);
        if (NetworkUtils.isConnected(BaseApplication.getApplication())) {
            int maxAge = DEFAULT_MAX_AGE;
            // 有网络时 设置缓存超时时间1个小时
            response.newBuilder()
                    .removeHeader("Cache-Control")
                    .removeHeader("Expires")//(注1)
                    .removeHeader("Pragma")// 清除头信息，因为服务器如果不支持，会返回一些干扰信息，不清除下面无法生效
                    .header("Cache-Control", "public, max-age=" + maxAge)
                    .build();
        } else {
            // 无网络时，设置超时为1周
            int maxStale = DEFAULT_MAX_STALE;
            response.newBuilder()
                    .removeHeader("Cache-Control")
                    .removeHeader("Pragma")
                    .removeHeader("Expires")
                    .header("Cache-Control", "public, only-if-cached, max-stale=" + maxStale)
                    .build();
        }
        return response;
    };
    //log拦截器
    private static final Interceptor LoggingInterceptor = chain -> {
        Request request = chain.request();
        long t1 = System.nanoTime();
        Log.i("okhttp:", String.format("Sending request %s on %s%n%s", request.url(), chain.connection(), request.headers()));
        Response response = chain.proceed(request);
        long t2 = System.nanoTime();
        Log.i("okhttp:", String.format("Received response for %s in %.1fms%n%s", response.request().url(), (t2 - t1) / 1e6d, response.headers()));
        return response;
    };
    public static <T> T createService(String baseUrl, Class<T> serviceClazz) {
        Retrofit retrofit = new Retrofit.Builder()
                .client(getOkHttpClient())
                .baseUrl(baseUrl)
                .addConverterFactory(GsonConverterFactory.create())
                .addCallAdapterFactory(RxJavaCallAdapterFactory.create())
                .build();
        return retrofit.create(serviceClazz);
    }
}
```

>创建service执行网络请求

```java
IBookService iBookService = ServiceFactory.createService(URL.HOST_URL_DOUBAN, IBookService.class);
    iBookService.getBookList(q, tag, start, count, fields)
            .subscribeOn(Schedulers.newThread())    //请求在新的线程中执行
            .observeOn(AndroidSchedulers.mainThread()) //在主线程中执行
            .subscribe(new Subscriber<Response<BookListResponse>>() {
                @Override
                public void onCompleted() {

                }

                @Override
                public void onError(Throwable e) {
                    //listener.onFailed(new BaseResponse(404, e.getMessage()));
                    //请求出错
                }

                @Override
                public void onNext(Response<BookListResponse> bookListResponse) {
                    /*if (bookListResponse.isSuccessful()) {
                        listener.onComplected(bookListResponse.body());
                    } else {
                        listener.onFailed(new BaseResponse(bookListResponse.code(), bookListResponse.message()));
                    } */
                    //请求成功

                }
            });
```

(注1):在正常web请求时用户可以指定自己的缓存策略，可以在请求头里面加入`Cache-control : cache` 其中`cache`有`no-cache`,`no-store`,`max-age = time`,`max-stale = time`,`only-if-cached`等等。服务器响应头也有类似的`Cache-control`,当然这些策略需要服务器支持才行，比如你请求缓存某个请求的数据，然后服务器返回的确实`no-cache`,这就是服务器有意为之不让客户端缓存，比如豆瓣接口 api,就不允许缓存，如下豆瓣响应头信息：

![douban_cache](http://ww4.sinaimg.cn/mw690/005X6W83gw1f6p032730gj309n06wt90.jpg)

一般的网络请求框架都考虑到这个特性，比如 okhttp，他会根据响应头还做缓存机制，但是对于豆瓣这种不让缓存该怎么办尼？我们会想到人为伪造响应头，将不缓存策略改成缓存策略，这就需要使用**拦截器**拦截响应头，然后修改之。

如上我们根据有无网络动态修改了响应头`"Cache-Control", "public, max-age=" + maxAge`。maxAge为一个缓存时间，考虑到在`HTTP/1.0`协议里面`Cache-Control`支持的不全，但是另一个头信息`Pragma: no-cache`却有效，可以看到豆瓣响应头，能设置不缓存的地方都给设置了，所以我们要删掉这些头信息，防止有所干扰。
`Expires `:表示存在时间，允许客户端在这个时间之前不去检查（发请求），等同max-age的
效果。但是如果同时存在，则被Cache-Control的max-age覆盖。
格式：
Expires = "Expires" ":" HTTP-date

##五.  配合MVP实现逻辑清晰易维护的代码
MVP模型分3块 Model，View，Presenter，相互配合分工明确。

1.  Model主要处理数据部分，**比如执行网络请求，获取网络数据**；<br>
2.  View主要用来更新视图，**比如数据请求完成了页面数据需要刷新**，就是通过view来控制的；
3.  Presenter主要用来处理逻辑的，协调配合Model层和View层，**比如让model去请求数据，让view更新视图，处理网络异常等逻辑**。

####1. View层 由activity或者fragment继承，来更新视图

```java
    public interface IBookListView {
        void showMessage(String msg);

        void showProgress();

        void hideProgress();

        void refreshData(Object result);

        void addData(Object result);
    }
```

####2. Presenter层 先定义好presenter的接口然后创建实现类，负责逻辑的处理工作

```java
    public interface IBookListPresenter {
        void loadBooks(String q, String tag, int start, int count, String fields);

        void cancelLoading();
    }
```

```java
public class BookListPresenterImpl implements IBookListPresenter, ApiCompleteListener {
    private IBookListView mBookListView;
    private IBookListModel mBookListModel;

    public BookListPresenterImpl(IBookListView view) {
        mBookListView = view;
        mBookListModel = new BookListModelImpl();
    }

    /**
     * 加载数据
     */
    @Override
    public void loadBooks(String q, String tag, int start, int count, String fields) {
        if (!NetworkUtils.isConnected(BaseApplication.getApplication())) {
            mBookListView.showMessage(BaseApplication.getApplication().getString(R.string.poor_network));
            mBookListView.hideProgress();
//            return;
        }
        mBookListView.showProgress();
        mBookListModel.loadBookList(q, tag, start, count, fields, this);
    }

    @Override
    public void cancelLoading() {
        mBookListModel.cancelLoading();
    }

    /**
     * 访问接口成功
     *
     * @param result 返回结果
     */
    @Override
    public void onComplected(Object result) {
        if (result instanceof BookListResponse) {
            int index = ((BookListResponse) result).getStart();
            if (index == 0) {
                mBookListView.refreshData(result);
            } else {
                mBookListView.addData(result);
            }
            mBookListView.hideProgress();
        }
    }

    /**
     * 请求失败
     *
     * @param msg 错误信息
     */
    @Override
    public void onFailed(BaseResponse msg) {
        mBookListView.hideProgress();
        mBookListView.showMessage(msg.getMsg());
    }
}
```

####3. Model层 现在到了具体数据访问的地方了，model接口以及其实现类

```java
    public interface IBookListModel {
    /**
     * 获取图书接口
     */
    void loadBookList(String q, String tag, int start, int count, String fields, ApiCompleteListener listener);

    /**
     * 取消加载数据
     */
    void cancelLoading();
}
```

在实现类中进行网络访问，这里配合retrofit和rxjava进行数据访问，同创建service执行网络请求
```java
public class BookListModelImpl implements IBookListModel {

/**
 + 获取图书列表
 */
@Override
public void loadBookList(String q, final String tag, int start, int count, String fields, final ApiCompleteListener listener) {
    IBookService iBookService = ServiceFactory.createService(URL.HOST_URL_DOUBAN, IBookService.class);
    iBookService.getBookList(q, tag, start, count, fields)
            .subscribeOn(Schedulers.newThread())    //请求在新的线程中执行
            .observeOn(AndroidSchedulers.mainThread())//最后在主线程中执行
            .subscribe(new Subscriber<Response<BookListResponse>>() {
                @Override
                public void onCompleted() {

                }

                @Override
                public void onError(Throwable e) {
                    listener.onFailed(new BaseResponse(404, e.getMessage()));
                }

                @Override
                public void onNext(Response<BookListResponse> bookListResponse) {
                    if (bookListResponse.isSuccessful()) {
                        listener.onComplected(bookListResponse.body());
                    } else {
                        listener.onFailed(new BaseResponse(bookListResponse.code(), bookListResponse.message()));
                    }

                }
            });
}

@Override
public void cancelLoading() {

}
}
```

##总结
到此为止已经基本完成，但是在使用缓存的时候还遇到一点问题，豆瓣接口响应头默认不允许缓存`Pragma: no-cache` `Cache-Control: must-revalidate, no-cache, private`,在缓存拦截器里面修改这些值达到强制缓存，结果还是无效。有知道解决方法的还望不吝赐教。
**谢谢阅读**
>来自github项目[weeklyblog](https://github.com/itsCoder/weeklyblog)。