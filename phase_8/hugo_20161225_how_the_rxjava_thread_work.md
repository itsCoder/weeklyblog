---
title: RxJava 线程切换源码
date: 2016-12-25 16:02:31
---
>- 文章来源：itsCoder 的 [WeeklyBolg](https://github.com/itsCoder/weeklyblog) 项目
>- itsCoder主页：[http://itscoder.com/](http://itscoder.com/)
>- 作者：[谢三弟](https://github.com/xcc3641)
>- 审阅者：[]()

### 目录

### 前言

RxJava 是在今年年初的时候上的车，接触也快要满一年了。从最初只知道几个操作符，写写 Demo ，或者跟着别人的项目和经验依葫芦画瓢，到目前终于有点初窥门径的地步。

RxJava 对于 Android 来说，最直观地便利就在于线程切换。所以本篇内容就是学习 **RxJava 是如何实现切换线程**。

<--! more -->

> 本章内容基于源码版本
>
> **RxJava: 1.1.5**

### 准备

**答案我会放在文章末尾**

先来一道开胃菜：

```java
 Observable.just(1) //1
   			.subscribeOn(Schedulers.newThread())
            .map() //2
            .subscribeOn(Schedulers.io())
            .map() //3
            .observeOn(Schedulers.computation())
            .map() //4
            .observeOn(Schedulers.newThread())
            .subscribe() //5
```

我们再改动下：

```java
 Observable.just(1) //1
   			.subscribeOn(Schedulers.newThread())
            .map() //2
            .subscribeOn(Schedulers.io())
            .map() //3
            .observeOn(Schedulers.computation())
            .map() //4
   			.doOnSubscribe() //6
            .observeOn(Schedulers.newThread())
            .subscribe() //5
```

只添加了一行```.doOnSubscribe() //6``` 。



------

开胃菜就到上面结束，如果你能够清楚明白每个操作运行的线程，说明对于 RxJava 的线程切换的理解很正确。

再具体分析 RxJava 是如何线程切换的，希望能清楚以下几个 RxJava 中名词的意思。

- Create()
- OnSubscribe
- Operator

*如果你特别明白这几个 RxJava 类/方法的作用，可以直接跳过看 **[切换](#切换)** 这部分。*

1.    Create()

      ```java
      /**
      * Returns an Observable that will execute the specified function when a {@link Subscriber} subscribes to
      * it.
      */

      public static <T> Observable<T> create(OnSubscribe<T> f) {
         return new Observable<T>(RxJavaHooks.onCreate(f));
      }
      ```

      方法注释上说明，当订阅者订阅之后，该函数会返回将会执行具体功能的流。像新手向[操作符](https://mcxiaoke.gitbooks.io/rxdocs/content/Operators.html)      ```just/map/...``` 进入源码会发现他们最终都会调用到 ```create()``` 函数。

2.    OnSubscribe
         接着讲
      ```java
       /**
      * Invoked when Observable.subscribe is called.
      * @param <T> the output value type
      */
      public interface OnSubscribe<T> extends Action1<Subscriber<? super T>> {}
      ```
      首先我们知道这是一个继承 ```Action1``` 的接口，并且是在 ```Observable.subscribe``` 流进行订阅操作后回调。而且回顾刚刚 ```create()``` 源码中也发现参数就是这个 ```OnSubscribe``` 。 ```Action``` 的作用就是执行其中的 ```call()``` 方法。

      **Observable.OnSubscribe** 有点像 Todo list ，里面都是一个一个待处理的事务。

3.    Operator

         简单理解就是对这个流进行改变，然后返回一个新的流，通常是跟 ```lift()``` 操作符一起。


      > Lifts a function to the current Observable and returns a new Observable





### 切换



#### SubscribeOn()

这个方法最后会到这个类：

```java
public final class OperatorSubscribeOn<T> implements OnSubscribe<T> {

    final Scheduler scheduler;
    final Observable<T> source;

    public OperatorSubscribeOn(Observable<T> source, Scheduler scheduler) {
        this.scheduler = scheduler;
        this.source = source;
    }
}
```

我先贴出这个类的，构造方法和成员变量，因为很重要，我们先把**前因**弄清楚。

首先我们发现这个类是实现了 ```OnSubscribe``` 接口，之前复习到这个的作用就是在该流被订阅之后执行 ```call()``` 方法，这里面就是**后果**，待会我们来看。

前因其实很简单，就是传入两个参数：

1. 一个是 ```Scheduler``` ，调度器，它的具体实现在 ```Schedulers``` 里。

2. ```Observable<T> source``` 这个其实就是当前这个流。

   ```java
   public final Observable<T> subscribeOn(Scheduler scheduler) {
     if (this instanceof ScalarSynchronousObservable) {
       return ((ScalarSynchronousObservable<T>)this).scalarScheduleOn(scheduler);
     }
     return create(new OperatorSubscribeOn<T>(this, scheduler));
   }
   ```



接下来看看 ```call()``` 核心代码里做的事情：

```java
    
	// 因为是 OnSubscribe 类，这里 call() 中传入的参数是 Observable.subscribe(s) 中的 s 
	@Override
    public void call(final Subscriber<? super T> subscriber) {
      	// 根据传入的调度器，创建一个 Worker
        final Worker inner = scheduler.createWorker();
        subscriber.add(inner);
        
      	// 在 Worker 内部执行
        inner.schedule(new Action0() {
            @Override
            public void call() {
                final Thread t = Thread.currentThread();
                
              	// 对订阅者包装
                Subscriber<T> s = new Subscriber<T>(subscriber) {
                    @Override
                    public void onNext(T t) {
                        subscriber.onNext(t);
                    }
                    ······
                };
                
              	// 这一句位置很关键
              	// 首先 source 是之前传入的当前流，在 Worker 内部进行了订阅操作，所以执行的线程就在 Worker 里
                source.unsafeSubscribe(s);
            }
        });
    }

```

通过我们指定的调度器，创建好 Worker ，之前传入的流在 Worker 内部，对重新包裹的 subscriber 进行订阅操作。

所以 ```SubscribeOn()```最关键的地方其实是因为这行代码在调度器创建的 Worker 的 ```call()``` 中

```java
source.unsafeSubscribe(s);
```

总结：

> `subscribeOn` 的调用，改变了调用前序列所运行的线程。



#### ObserveOn









### 参考

[Thomas Nield: RxJava\- Understanding observeOn\(\) and subscribeOn\(\)](http://tomstechnicalblog.blogspot.jp/2016/02/rxjava-understanding-observeon-and.html)

[Rxjava 中， subscribeOn 及 oberveOn 方法切换线程发生的位置为什么设计为不同的？ \- 知乎](https://www.zhihu.com/question/41779170)



1. ​

> 答案：
>
> 1 newThread
>
> 2 newThread
>
> 3 newThread
>
> 4 computation
>
> 5 newThread

2. ​