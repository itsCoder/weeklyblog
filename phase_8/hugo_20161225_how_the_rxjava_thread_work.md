---
title: RxJava 线程切换源码的一些体会和思考
date: 2016-12-25 16:02:31
---
>- 文章来源：itsCoder 的 [WeeklyBolg](https://github.com/itsCoder/weeklyblog) 项目
>- itsCoder主页：[http://itscoder.com/](http://itscoder.com/)
>- 作者：[谢三弟](https://github.com/xcc3641)
>- 审阅者：
>      - [用语](https://github.com/yongyu0102)
>      - [JasonThink](https://github.com/jasonim)


### 目录

- [目录](#目录)
- [前言](#前言)
- [切换](#切换)
  + [SubscribeOn](#SubscribeOn)
  + [ObserveOn](#ObserveOn)
  + [共用时各自的作用域](#共用时各自的作用域)
- [思考](#思考)
- [参考](#参考)

### 前言

RxJava 是在今年年初的时候上的车，接触也快要满一年了。从最初只知道几个操作符，写写 Demo ，或者跟着别人的项目和经验依葫芦画瓢，到目前终于有点初窥门径的地步。

RxJava 对于 Android 来说，最直观地便利就在于线程切换。所以本篇内容就是学习 **RxJava 是如何实现切换线程**。 

**希望读者阅读此篇文章，是有用过 RxJava 的童鞋。**

<!-- more -->

> 本章内容基于源码版本
>
> **RxJava: 1.2.4**

### 准备

**答案我会放在文章末尾**

先来一道开胃菜：

指出下列程序操作符所运行的线程。

```java
Observable.just() //1
          .subscribeOn(Schedulers.newThread())
          .map() //2
          .subscribeOn(Schedulers.io())
          .map() //3
          .observeOn(Schedulers.computation())
          .map() //4
          .observeOn(Schedulers.newThread())
          .subscribe() //5
```

------

开胃菜就到上面结束，如果你能够清楚明白每个操作运行的线程，说明对于 RxJava 的线程切换的理解很正确。

再具体分析 RxJava 是如何线程切换的，希望能清楚以下几个 RxJava 中名词的意思。

- Create()
- OnSubscribe
- Operator

*如果你特别明白这几个 RxJava 类/方法的作用，可以直接跳过看[切换](#切换)这部分。*

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

      方法注释上说明，当订阅者订阅之后，该函数会返回将会执行具体功能的流。[操作符](https://mcxiaoke.gitbooks.io/rxdocs/content/Operators.html)进入源码会发现他们最终都会调用到 ```create()``` 函数。

2.    OnSubscribe

      ```java
      /**
       * Invoked when Observable.subscribe is called.
       * @param <T> the output value type
       */
      public interface OnSubscribe<T> extends Action1<Subscriber<? super T>> {}
      ```
      首先我们知道这是一个继承 ```Action1``` 的接口，并且是在 ```Observable.subscribe``` 流进行订阅操作后回调。而且回顾刚刚 ```create()``` 源码中也发现参数就是这个 ```OnSubscribe``` 。 ```Action``` 的作用就是执行其中的 ```call()``` 方法。

      **Observable.OnSubscribe** 有点像 Todo List ，里面都是一个一个待处理的事务，并且这个 List 是有序的（这个很关键）。

3.    Operator

      ```java
      public interface Operator<R, T> extends Func1<Subscriber<? super R>, Subscriber<? super T>> {
        // cover for generics insanity
      }
      ```

      简单来说它的职责就是将一个 ```Subscriber``` 变成另外一个 ```Subscriber```。



### 切换


上面知识点是一些小铺垫，因为后面的内容的核心其实就是上面几个类的作用。

#### SubscribeOn

追踪这个方法，核心是在这个类：

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
      	// 根据传入的调度器，创建一个 Worker 对象 inner
        final Worker inner = scheduler.createWorker();
        subscriber.add(inner);
        
      	// 在 Worker 对象 inner 中执行（意思就是，在我们指定的调度器创建的线程中运行）
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
              	// 首先 source 是之前传入的流（也就是当前流），在 Worker 内部进行了订阅操作，所以该流所有操作都执行在其中
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

> `subscribeOn` 其实是改变了调用前序列所运行的线程。

#### ObserveOn

同样的方法来分析，最终的回调会到：

```java
public final Observable<T> observeOn(Scheduler scheduler, boolean delayError, int bufferSize) {
  if (this instanceof ScalarSynchronousObservable) {
    return ((ScalarSynchronousObservable<T>)this).scalarScheduleOn(scheduler);
  }
  return lift(new OperatorObserveOn<T>(scheduler, delayError, bufferSize));
}

```

其实看到关键字 lift 和 operator 就大约可以猜到是做什么的了。

接下来我们进入到 ```OperatorObserveOn``` 类中：

```java
public final class OperatorObserveOn<T> implements Operator<T, T> {

    private final Scheduler scheduler;
  	// 省略不必要的代码
      
    @Override
    public Subscriber<? super T> call(Subscriber<? super T> child) {
        	// 省略 ···
            ObserveOnSubscriber<T> parent = new ObserveOnSubscriber<T>(scheduler, child, delayError, bufferSize);
            parent.init();
            return parent;
        }
    }
}
```

我们首先会注意到它是一个 ```Operator``` ，并且没有对上层 Observale 做任何修改和包装。那么它的作用就是将一个 ```Subscriber``` 变成另外一个 ```Subscriber```。所以接下来我们的首要任务就是看转换后的 ```Subscriber``` 做了什么改变。

关键代码在

```java
ObserveOnSubscriber<T> parent = new ObserveOnSubscriber<T>(scheduler, child, delayError, bufferSize);
parent.init();
```

**child** 是改变前的 ```Subscriber``` ，最后返回了 **parent** 。

我们发现 ```ObserveOnSubscriber``` 同样也是一个 ```Subscriber``` 类，所以肯定含有 ```onNext/onError/onComplete``` 这三个标准方法，重要的肯定是 ```onNext``` ，所以我只贴上了该类三个有关函数。

```java
void init() {
    Subscriber<? super T> localChild = child;
    
    localChild.setProducer(new Producer() {

        @Override
        public void request(long n) {
            if (n > 0L) {
                BackpressureUtils.getAndAddRequest(requested, n);
              	// 执行
                schedule();
            }
        }

    });
    // recursiveScheduler 这个是构造函数时传入调度器创建的 worker
    localChild.add(recursiveScheduler);
    localChild.add(this);
}

@Override
public void onNext(final T t) {
  if (isUnsubscribed() || finished) {
    return;
  }
  // 条件判断里先将之前流的结果缓存进队列
  if (!queue.offer(on.next(t))) {
    onError(new MissingBackpressureException());
    return;
  }
  // 执行
  schedule();
}


protected void schedule() {
	if (counter.getAndIncrement() == 0) {
      	// 在当前 worker 上执行该类的 call 方法
		recursiveScheduler.schedule(this);
	}
}
```


```call()``` 方法有点冗长，做的事情其实很简单，就是取出我们缓存之前流的所有值，然后在 Worker 工作线程中传下去。



总结：

> 1. ObserveOn 不会关心之前的流的线程
> 2. ObserveOn 会先将之前的流的值缓存起来，然后再在指定的线程上，将缓存推送给后面的 ```Subscriber```


#### 共用时各自的作用域


```java
 Observable.just() //1
            .subscribeOn(Schedulers.newThread())
            .map() //2
            .map() //3
            .observeOn(Schedulers.computation())
            .map() //4
            .observeOn(Schedulers.newThread())
            .subscribe() //5
```
如果分析这个流各个操作符的执行线程，我们先把第一个 ```subscribeOn()``` 之前和第一个 ```observeOn()``` 之前的 Todo Items 找出来然后求并集：

得到的结果就是 ```subscribeOn()``` 的作用域。

![](http://ww1.sinaimg.cn/large/006y8lVagw1fb80zs48yrj30yc0fcwg4.jpg)


之后的线程切换简单了，遇到 ```observeOn()``` 就切换一次。

### 思考



#### 为什么``` subscribeOn ``` 只有第一次调用生效？

我的理解如下：

```subscribeOn``` 的作用域就是调用前序列中所有的 **Todo List 任务清单**（Observable.OnSubscribe），当我们执行 ```subscribe()``` 时，这些任务清单就会执行在 ```subscribeOn```  指定的工作线程，而第二个 ```subscribeOn``` 早就没有任务可做了，所以无法生效。



------



*知乎里这段说的比我专业：*

> 正像 StackOverflow 上那段描述的，整个 Observable 数据流工作起来是分为两个阶段（或者说是两个 lifecycle）：upstream 的 subscription-time 和 downstream 的 runtime。
>
> subscription-time 的阶段，是为了发起和驱动数据流的启动，在内部实现上体现为 OnSubscribe 向上游的逐级调用（控制流向上游传递）。支持 backpressure 的 producer request 也属于这个阶段。除了 producer request 的情况之外，subscription-time 阶段一般就是从下游到上游调用一次就结束了，最终到达生产者（以最上游的那个 OnSubscribe 来体现）。接下来数据流就开始向下游流动了。

[Rxjava 中， subscribeOn 及 observeOn 方法切换线程发生的位置为什么设计为不同的？ \- 知乎](https://www.zhihu.com/question/41779170)



#### doOnSubscribe 的例外

我们再改动下开胃菜的代码：

```java
Observable.just() //1
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

只添加了一行```.doOnSubscribe() //6``` ，也是探讨这个操作符执行的线程。


```java
public class OperatorDoOnSubscribe<T> implements Operator<T, T> {
    private final Action0 subscribe;

    public OperatorDoOnSubscribe(Action0 subscribe) {
        this.subscribe = subscribe;
    }

    @Override
    public Subscriber<? super T> call(final Subscriber<? super T> child) {
        // 执行我们的 Action
        subscribe.call();
        // Wrap 里面是包装成一个新的 Subscriber 返回，不对这个流做任何改变
        return Subscribers.wrap(child);
    }
}
```

doOnSubscribe 执行的线程其实就是 ```subscribe.call();``` 所在的线程。这里触发的时机就是，当我们进行 ```Observable.subscribe()``` 时，如果我们没有在紧接之后```SubscribeOn``` 指定线程，那么它就会运行在默认线程，然后返回一个新的流。


------

**关于 ```doOnSubscribe()``` 留一个问题**

```java
Observable.just()
          .doOnSubscribe() // 1
          .doOnSubscribe() // 2
          .subscribe()
```
> 问题是，对于 1 和 2 的执行顺序？


在开发中，我们肯定不会像问题那样写代码，只是自己在看 doOnSubscribe 源码的时候，在问自己为什么它在其他操作符之前，拓展到了 RxJava 流的一个执行顺序，也是自己想要明白的地方。所以下次准备探讨学习。

> 对了，老司机说 RxJava 很像洋葱，一层一层。


### 参考

[Thomas Nield: RxJava\- Understanding observeOn\(\) and subscribeOn\(\)](http://tomstechnicalblog.blogspot.jp/2016/02/rxjava-understanding-observeon-and.html)

[SubscribeOn 和 ObserveOn |Piasy Blog](http://blog.piasy.com/AdvancedRxJava/2016/09/16/subscribeon-and-observeon/)


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
