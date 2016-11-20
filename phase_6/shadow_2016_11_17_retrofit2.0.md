---
layout: post
title:  Retrofit 2.0 应用场景概述
author: shaDowZwy
---

最近最火的网络库应该是`Retrofit`了，我也在项目中耍了起来，可以说是非常的有趣。
个人感觉是`Retrofit`网络库是简洁，实用和逻辑清晰，将网络接口和回调逻辑分层处理，在逻辑层你都可以不用知道网络接口的类型和参数。


>- 文章来源：itsCoder 的 [WeeklyBolg](https://github.com/itsCoder/weeklyblog) 项目
>- itsCoder主页：[http://itscoder.com/](http://itscoder.com/)
>- 作者：[Shadow]( https://github.com/shaDowZwy )
>- 审阅者：[ZetaoYang](https://github.com/ZetaoYang)

我就开始从自己的应用场景说起吧。


### **`Retrofit 2.0` 接入**
首先，我们应该是添加依赖，这应该大家都很熟悉吧，我就添加了我项目中需要用的依赖。

```java
    compile 'com.squareup.retrofit2:retrofit:2.1.0'
    compile 'com.squareup.retrofit2:converter-gson:2.1.0'
    compile 'com.squareup.retrofit2:adapter-rxjava:2.1.0'
```
添加了这三个依赖我们就可以正常的耍起来了，当然如果你要用`rxJava`别忘了添加相关依赖，这里我就不说了。

#### **1.先把`RetrofitClient`搞起来**
我们要先知道`Retrofit`其实一个对`okhttp`的一个封装，它是基于`okhttp`的，所以`okhttp`的有些特性也可以使用。不多说，我们先把`Retrofit`封装起来使用。


首先，我们需要定义一个网络请求的接口，它是负责写所有网络请求的，需要根据`Retrofit`的规范来写，待会下面会详细说。


```java
//Retrofit 网络请求接口
public interface RetrofitNetApi 
```

然后，我们还需要`OkHttpClient`，通过它可以设置一些属性，例如失败重连和网络拦截器，网络拦截可以设置`header`。

```java

private static final OkHttpClient mOkHttpClient = new OkHttpClient.Builder()
            .retryOnConnectionFailure(true)
            .addNetworkInterceptor(new StethoInterceptor())
            .addInterceptor(chain -> {
                //添加header
                Request original = chain.request();
                Request request = original.newBuilder()
                        .header("X-Token", token)
                        .addHeader("X-Device-ID", deviceID)
                        .addHeader("X-UID", userID + "")
                        .addHeader("X-App-Version", version + "")
                        .addHeader("user-agent", UA)
                        .build();

                return chain.proceed(request);
            })
            .build();

```
这种设置`header`的方式是`okhttp`的方式，通过拦截器设置`header`；其实，`Retrofit`也有设置`header`的方式，但是它是属于传参数的形式，需要每一个网络接口请求的时候进行设置，不利于封装，重用了代码。例如：

```java
@GET("user")
Call<ResponseBody> getUser(@Header("header1") String header)
```

最后，封装一个通用的`RetrofitClient`用于调用网络请求接口。

```java
    public static RetrofitNetApi getNetApi() {
        if (RetrofitApi == null) {
            Retrofit retrofit = new Retrofit.Builder()
                    .client(mOkHttpClient)
                    .baseUrl(getUrl())
                    .addConverterFactory(GsonConverterFactory.create())
                    .build();
            RetrofitApi = retrofit.create(RetrofitNetApi.class);
        }
        return RetrofitApi;
    }
```

`baseUrl()`是你的基础网络请求接口，例如是 "www.shadow.com/user" ，那个 "baseUrl" 就是 "www.shadow.com"。`addConverterFactory(GsonConverterFactory.create())`是吧`json`结果并解析成`DAO`，也可以添加其他的解析方式，但是必须换其他方式的依赖。


#### **2.定义`Retrofit`网络接口**

首先是一个简单的`Get`请求
网络接口是这样的”www.shadow.com/xxx/session“
那我们可以这样

```java
@GET("account/session")
Call<ResponseBody> getSession();
```

**还有很多**

例如`Get`请求 "www.shadow.com/xxxx/book?pageIndex=1&pageSize=20"


你可以这么拼接你的网络请求,使用操作符`@Query`。

```java
@GET("xxxx/book")
Call<ResponseBody> book(@Query("pageIndex") String pageIndex, @Query("pageSize") String pageSize);

```


还有`Get`请求 "www.shadow.com/xxxx/bookname" 通过替换`bookname`来请求接口,
可以使用操作符`@Path`。

```java
@GET("xxxx/{bookname}")
Call<ResponseBody> bookname(@Path("bookname") String bookname);
```




`Post`请求 "www.shadow.com/xxxx/booklist" 需要提交参数，可以使用操作符`@Field`，而且不用忘记加上`@FormUrlEncoded`.

```java
@FormUrlEncoded
@POST("xxxx/booklist")
Call<ResponseBody> booklist(@Field("booklist") String booklist);
```

而且`@Field`不限制请求方式，你用`Put`也可以，但是不能忘记`@FormUrlEncoded`。

```java
@FormUrlEncoded
@PUT("xxxx/booklist")
Call<ResponseBody> booklist(@Field("booklist") String booklist);
```

**其实`Retrofit`的请求方式很灵活，几乎能满足你想要的各种姿势。**

例如是这样的`Get`请求 
"www.shadow.com/xxxx/bookname/book?pageIndex=1&pageSize=20"

```java
@GET("xxxx/{bookname}/book")
Call<ResponseBody> bankLimit(@Path("bookname") String bookname, @Query("pageIndex") String pageIndex,@Query("pageSize") String pageSize);
```

或者是这样的`Post`请求
"www.shadow.com/xxxx/bookname/booklist"

```java

@FormUrlEncoded
@POST("xxxx/{bookname}/booklist")
Call<ResponseBody> bonusAutoInvest(@Path("bookname") int bookname, @Field("booklist") boolean booklist);

```

**就是这么的好玩**



### **封装通用网络库**

按照以上说的，我们已经把网络接口定义好了，那现在需要封装通用网络库。
上面的代码，可以发现我的网络接口返回的都是`Call<ResponseBody>`，而不是`Observable<T>`

`Call<T>`是`Retrofit`的一个`Callback`形式的参数回调,T是任意泛型，这个`Call<T>`有一个网络发送请求的方法，是同步的。

`ResponseBody`是`okhttp`的一个请求体，可以封装json。

这样使用，是因为我不想直接获取已经解析好的`bean`，而是拿到`json`，更加的灵活判断网络请求的各种状况。

以下代码是用`RxJava`封装的网络库

```java

private Observable<String> getResults(Call<ResponseBody> call, int delay) {

        return Observable.create(new Observable.OnSubscribe<String>() {
            @Override
            public void call(final Subscriber<? super String> subscriber) {
                try {
                    retrofit2.Response<ResponseBody> response = call.execute();
                    String result = response.body() != null ? response.body().string() : "";
                    String error = response.errorBody() != null ? response.errorBody().string() : "";
                    int code = response.code();
                   
                    if (response.isSuccessful()) {  //返回值在200至300之间表示返回成功
                        //根据某种约定的特殊处理 没有返回值
                        if (!result.startsWith("{") && (!result.startsWith("[")) || code == 204 || result.equals("{}"){ 
                            subscriber.onNext("ok");
                        } else {    //有返回值
                            subscriber.onNext(result);
                        }

                    } else {    //请求失败时的处理
                        if (TextUtils.isEmpty(error)) {    //没有返回标准的报错信息
                            subscriber.onError(new Exception("code:" + code));
                        } else {
                            subscriber.onError(new Exception(error));
                        }
                    }
                } catch (Exception e) {
                            subscriber.onError(e);
                }
            }

        }).delay(delay, TimeUnit.MILLISECONDS);
    }


```

如代码，我可以把网络接口`Call<ResponseBody>`传参进去，执行网络请求，然后根据返回的`json`，不同的情况做不同的处理，最后用`subscriber`回调正确或者错误的结果。正确的`json`可以解析成我们需要的`bean`，错误的`json`可以根据不同的状况作出不同的错误处理，这样一来比较灵活和解耦，能处理各种不同的网络请求的需求。

**最外层的就是普通的`Rxjava`调用了，这里就不描述了。**


最后，这就是我使用的`Retrofit 2.0`网络库的封装和各种应用场景，给我的感觉就是`Retrofit`非常的简洁和高效。





### **最后** 



这是一个简洁实用的网络封装，记录一下思路。

