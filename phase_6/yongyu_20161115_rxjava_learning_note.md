---

 title:  RxJava 学习笔记（部分示例代码及源码）
 date:  2016/11/15
 categories:  Android RxJava
 tags:  Android RxJava

---

-文章来源：itsCoder 的 [WeeklyBolg](https://github.com/itsCoder/weeklyblog) 项目

- itsCoder主页：[http://itscoder.com/](http://itscoder.com/)
- 作者：[yongyu0102](https://github.com/yongyu0102)
- 审阅者：[]()

# 一、初识 RxJava 

**RxJava 是什么** ：它就是一个实现异步操作的库，使你的程序逻辑简介清晰实现链式调用，避免代码的迷之嵌套以及各种接口回调。

**扩展的观察者模式**：RxJava 的异步实现，是通过一种扩展的观察者模式来实现的，观察者模式面向的需求是：A 对象（观察者）对 B 对象（被观察者）的某种变化高度敏感，需要在 B 变化的一瞬间做出反应。Android 开发中一个比较典型的例子是点击监听器 `OnClickListener` 。对设置`OnClickListener` 来说， View 是被观察者， `OnClickListener` 是观察者，二者通过 `setOnClickListener()`方法达成订阅关系。订阅之后用户点击按钮的瞬间，Android Framework 就会将点击事件发送给已经注册的`OnClickListener` 。

**RxJava 中重要概念**
**Observable**：被观察者，这个类提供一系列方法用于被 Observers 去订阅，即在 RxJava 中 一个 Observer  观察者去 subscribe 订阅一个 Observable 被观察者，Observable 决定事件触发的时候将有怎样的行为，即事件的产生者。


![observable_observer](image\observable_observer.png)

**Observer**： 观察者身份，用于观察 Observable，接受被观察者发送的事件，下面这段原文说的很形象：

```
After an Observer calls an  Observable's  subscribe method, the
Observable calls the Observer's onNext method to provide notifications. A well-behaved
 Observable will call an Observer's onCompleted method exactly once or the Observer's
onError method exactly once.
```

大概意思是：在一个观察者 Observer 调用 （calls ）一个被观察者 Observable  的 subscribe 方法之后，这个被观察者就会调用（calls ）观察者的 `onNext()` 方法来发送消息。

 **subscribe**：动词订阅，执行订阅，用于 Observer 去订阅 Observable，使二者之间建立联系。

**最后三者之间的关系**：Observable 和 Observer 通过 `subscribe()` 方法实现订阅关系，从而 Observable 可以在需要的时候发出事件来通知  Observer。

**订阅之后结果**：在 Observer  观察者 subscribe 订阅了被观察者 Observaber 之后会产生 onCompleted（表示事件完成）、onNext（接受事件产生的结果）、onError（表示事件产生错误）。

**onCompleted()**: 表示事件队列完结。RxJava 不仅把每个事件单独处理，还会把它们看做一个队列，RxJava 规定，当不会再有新的 `onNext()` 发出时，需要触发 `onCompleted()` 方法作为结束标志。

**onNext()：** 接受发送的事件，即接受数据。

**onError()**: 事件队列异常。在事件处理过程中出异常时，`onError()` 会被触发，同时队列自动终止，不允许再有事件发出。在一个正确运行的事件序列中，`onCompleted()` 和 `onError()` 有且只有一个，并且是事件序列中的最后一个。需要注意的是 `onCompleted()` 和 `onError()` 二者是互斥的，即在队列中调用了其中一个，就不再调用另一个。

RxJava 的观察者模式大致如下图：

![observer_detail](image\observer_detail.png)

**Subscriber**：Subscriber 对 Observer 接口进行了一些扩展，

```java
public abstract class Subscriber<T> implements Observer<T>, Subscription
```

也是观察者（订阅者），他的基本使用方式与 Observer 是完全一样的，在订阅者即 Subscriber 调用了被观察者 Observabler 的方法 subscribe 之后，被观察者 Observable 将会调用 Subscriber's  的方法 onNext 发送事件，而且在事件发送完毕会调用 Subscriber 的 onCompleted 方法或者在发送事件过程中出现错误就会调用 Subscriber  的 onError 方法。

```
 After a Subscriber calls an  Observable's  subscribe method, the
Observable calls the Subscriber's onNext method to emit items. A well-behaved
Observable will call a Subscriber's onCompleted method exactly once or the Subscriber's
onError  method exactly once.
```

而且这是一个抽象类，使用的时候必须实现其抽象方法，不可以直接 new ，可以使用匿名内部类的方式进行 new ，如下：

```java
 Subscriber<String> subscriber = new Subscriber<String>() {
@Override

    public void onNext(String s) {

        Log.d(tag, "Item: " + s);
    }

    @Override

    public void onCompleted() {
        Log.d(tag, "Completed!");
    }

    @Override

    public void onError(Throwable e) {
        Log.d(tag, "Error!");
    }

};
```

# 二、Observable 创建的几种方式及源码

## 2.1 Observable.create(new Observable.OnSubscribe<T>() 

```java
/**
 * 传入一个 OnSubscribe 对象，用于产生被被观察者行为
 */
Observable<String> observable=Observable.create(new Observable.OnSubscribe<String>() {
    //该方法在形成订阅关系的时候就会调用，在这里被观察者执行要执行的逻辑，发送对象
    //观察者就会接收到
  //而这个 call 方法持有观察者 Observer ，即 call(Subscriber<? super String> subscriber
  //中的 subscriber 就是传递就去的观察者 Observer，这里怎么把观察者传递进来的后面进行分析
    @Override
    public void call(Subscriber<? super String> subscriber) {
        //被观察者产生行为执行逻辑
        subscriber.onNext("Hello");
        subscriber.onNext("Tome");
        //注意这里强制抛出一个异常错误，那么 Observer 会接受到这个错误
        //然后 OnComplete()方法就不会调用， Observer 就接受不到
        subscriber.onError(new AndroidException("onError"));
        subscriber.onCompleted();
    }
});
```

看一下 OnSubscribe 这个类的源码：

```java
/**
 *是一个实现了 Action1 类的接口，当 Observable 的 subscribe 方法被调用的时候会被调用。
 * Invoked when Observable.subscribe is called.
 * @param <T> the output value type
 */
public interface OnSubscribe<T> extends Action1<Subscriber<? super T>> {
    // cover for generics insanity
}

/**
 * Action1 就是对有参数且没有返回值的一类方法的处理
 * A one-argument action.
 * @param <T> the first argument type
 */
public interface Action1<T> extends Action {
    void call(T t);
}
```

看一下 `Observable.create` 源码

```java
public static <T> Observable<T> create(OnSubscribe<T> f) {
    return new Observable<T>(hook.onCreate(f));
}
```

直接调用 了Observable 构造方法，只是将 OnSubscribe 参数进行了一层的包装，下面看一下如何包装的，这里的 hook 对象为 RxJavaObservableExecutionHook  类，是 RxJavaPlugins 中的一个类，用于插入一些你所需要的代码，记录，测试等，在默认的情况下，没有做任何对代码逻辑功能有影响的事情，以下是官方文档给出的解释：

> This plugin allows you to register functions that RxJava will call upon certain regular RxJava activities, for instance for logging or metrics-collection purposes.

`hook.onCreate(f)` 源码如下：

```java
public <T> OnSubscribe<T> onCreate(OnSubscribe<T> f) {
    return f;
}
```

大家看一下，这里直接返回了传入的参数，所以说这个类没做对业务逻辑有影响的事情，其他调用也类似，只是做了个包装，所以我们在分析源码思路的时候可以忽略其作用。那么接着看 Observable 构造函数干了什么：

```java
protected Observable(OnSubscribe<T> f) {
    this.onSubscribe = f;
}
```

很简单，直接保存全局持有创建的 onSubscribe 对象。这里被观察者创建源码就这么简单，分析完毕。下面看一下，我们实例化观察者 Observable 对象做了什么：

```java
public interface Observer<T> {
    void onCompleted();
    void onError(Throwable e);
    void onNext(T t);
}
```

一看就这么简单，就是一个接口，里面是我们实例化时候需要重写的几个方法，大家都很熟悉。

下面看一下订阅 `Observable.subscribe(observer)` 方法干了什么？

```java
public final Subscription subscribe(final Observer<? super T> observer) {
    if (observer instanceof Subscriber) {
      //判断 observer 是 Subscriber 类型，直接将 observer 强转为 Subscriber 类型
      //然后调用 obsaverble 的 subscribe(Subscriber<? super T> subscriber) 方法
        return subscribe((Subscriber<? super T>)observer);
    }
  //如果不是 Subscriber 类型 将 observer 包装成 Subscriber 类型，具体代码如下
    return subscribe(new ObserverSubscriber<T>(observer));
}
```

将 Observer 包装成 Subscriber代码：

```java
public final class ObserverSubscriber<T> extends Subscriber<T> {
    final Observer<? super T> observer;

    public ObserverSubscriber(Observer<? super T> observer) {
        this.observer = observer;
    }
  
    @Override
    public void onNext(T t) {
        observer.onNext(t);
    }
    
    @Override
    public void onError(Throwable e) {
        observer.onError(e);
    }
    
    @Override
    public void onCompleted() {
        observer.onCompleted();
    }
}
```

这里也很简单，没什么好说的，接着看 subscribe 方法：

```java
public final Subscription subscribe(Subscriber<? super T> subscriber) {
    return Observable.subscribe(subscriber, this);
}
```

该方法直接调用了两个参数的 subscribe 方法，而传递进去的参数一个是我们创建的观察者 subscriber (Observer) ，一个是被观察者自己本身 Observable 即参数 this ，接着看：

```java
static <T> Subscription subscribe(Subscriber<? super T> subscriber, Observable<T> observable) {
........
  //以上为非 null 判断
    
  // new Subscriber so onStart it
  //调用观察者的 onStart 方法 
    subscriber.onStart();
   
    // if not already wrapped
    if (!(subscriber instanceof SafeSubscriber)) {
      //对 subscriber 进行安全包装，是为了使 Subscriber 
      //遵守 Observable 的某种规则而进行的一次封装，
      //保证 onComplete 和 onError 互斥，onNext 在 onComplete 执行后，不再发送数据，
      //对异常做了一些操作等等。
      // assign to `observer` so we return the protected version
        subscriber = new SafeSubscriber<T>(subscriber);
    }

    // The code below is exactly the same an unsafeSubscribe but not used because it would 
    // add a significant depth to already huge call stacks.
    try {
        // allow the hook to intercept and/or decorate
      //真正实现观察者 Subscriber 和被观察者 Observable 两者关系的核心代码
      //  hook.onSubscribeStart(observable, observable.onSubscribe) 这段代码就只是返回了
      // onSubscribe 对象，而这个对象就是我们创建 Observable 时候创建的，
      //接着调用了 onSubscribe.call(subscriber),即调用 onSubscribe 的call 方法，
      //而传入的参数就是我们创建的观察者 这样实现了观察者的回调，完成了二者的订阅关系
      //这个 call 方法相当于点击事件的 click 方法
      //传入观察者（接口）对象，然后当实现订阅 Subscribe 的时候，观察者实现的接口就会接受数据，
      //相当于点击事件执行 setClickLisner 方法传入 listner 对象，
      //然后 listner 对象实现的接口就可以接受数据一样。
        hook.onSubscribeStart(observable, observable.onSubscribe).call(subscriber);
      //返回被观察者 subscriber 本身,因为 subscriber 也实现了Subscription 所以
      //返回该对象可以用于订阅取消的管理
        return hook.onSubscribeReturn(subscriber);
    } catch (Throwable e) {
        // special handling for certain Throwable/Error/Exception types
        Exceptions.throwIfFatal(e);
        // in case the subscriber can't listen to exceptions anymore
        if (subscriber.isUnsubscribed()) {
            RxJavaPluginUtils.handleException(hook.onSubscribeError(e));
        } else {
            // if an unhandled error occurs executing the onSubscribe we will propagate it
            try {
                subscriber.onError(hook.onSubscribeError(e));
            } catch (Throwable e2) {
                Exceptions.throwIfFatal(e2);
                hook.onSubscribeError(r);
                // TODO why aren't we throwing the hook's return value.
                throw r;
            }
        }
        return Subscriptions.unsubscribed();
    }
}
```

## 2.2 Observable.just(......)

用法如下：

```java
/**
 * just 函数的用法
 * 将传入的参数依次全部发送出来
 * @param t1 依次传入的参数，这些参数是不固定个数的可以是 n 个
 * @param t2
 * @param t3
 * @param <T>
 */
public <T> void  justTest(T t1,T t2,T t3 ){
    Observable.just(t1,t2,t3)
            //形成订阅关系
            .subscribe(new Action1<T>() {
                //接受发送结果
                @Override
                public void call(T t) {
                    Log.d(TAG,t.toString());
                }
            });
}
```

源码：

```java
public static <T> Observable<T> just(T t1, T t2) {
    return from((T[])new Object[] { t1, t2 });
}
```

源码可以看出， 在 `just()`方法内部直接将传入的不固定个数的参数直接转换为一个数组，然后传递给 `from()` 方法，那么我们看一下 `from()`  方法的用法：

## 2.3 Observable.from(T[] array)

```java
/**
 * 实现打印数组功能
 * 将传入的数组或 Iterable 拆分成具体对象后，依次发送出来。
 */
public void FromTest(){
    String [] names={"Tome","LiLei","XiaoMing"};
    Observable.from(names)
            //形成订阅关系
            //并只接受发送类型为String 类型对象
            .subscribe(new Action1<String>() {
                //接受发送的结果
                @Override
                public void call(String s) {
                    Log.d(TAG,s);
                }
            });
}
```

用法很简单，看源码：

```java
  /**
     * Converts an Array into an Observable that emits the items in the Array.
     * @param array the source Array
     * @param <T> the type of items in the Array and the type of items to be
     * emitted by the resulting Observable
     * @return an Observable that emits each item in the source Array
     */
public static <T> Observable<T> from(T[] array) {
    int n = array.length;
    if (n == 0) {
      //数组为 0
        return empty();
    } else
    if (n == 1) {
      //数组个数为 1
        return just(array[0]);
    }
  //数组个数大于 1
    return create(new OnSubscribeFromArray<T>(array));
}
```

从源码可以看出，这个方法的作用就是将一个数组转变为一个能够发送数组元素的 Observable 对象。

根据传入的数组长度分为三种情况进行调用，我们一起分析下:

**第一情况，数组长度为 0：**

这种情况调用了 `empty()` 方法，即数据为空，这种情况最终会调用  EmptyObservableHolder 类的 `call()` 方法，而 EmptyObservableHolder  继承自 OnSubscribe ，重写了 `call()` 方法：

```java
public enum EmptyObservableHolder implements OnSubscribe<Object> {
    @Override
    public void call(Subscriber<? super Object> child) {
      //直接调用 Subscriber 的 onCompleted 
        child.onCompleted();
    }
}
```

很明显，如果数组内元素个数为 0，那么直接调用了 Subscriber 的 `onCompleted() ` 方法完成数据发送。

**第二情况，数组长度为1，源码如下：**

```java
 /**
     * Returns an Observable that emits a single item and then completes.
     *返回一个发送单一数据的 Observable 对象
  **/
public static <T> Observable<T> just(final T value) {
    return ScalarSynchronousObservable.create(value);
}
```

接着看 ` ScalarSynchronousObservable.create()` 源码：

```java
/**
 * Constructs a ScalarSynchronousObservable with the given constant value.
 * @return the new Observable
 */
public static <T> ScalarSynchronousObservable<T> create(T t) {
  //直接根据传入的数据 new 了一个 ScalarSynchronousObservable 对象返回
    return new ScalarSynchronousObservable<T>(t);
}
```

看 ScalarSynchronousObservable 含一个参数的构造方法源码：

```java
protected ScalarSynchronousObservable(final T t) {
  //直接调用父类构造方法并传入 JustOnSubscribe 对象
    super(hook.onCreate(new JustOnSubscribe<T>(t)));
  //将传入参数保存为成员变量
    this.t = t;
}
```

这个方法很关键，ScalarSynchronousObservable 这个类继承自 Observable 类，所以 

`super(hook.onCreate(new JustOnSubscribe<T>(t)))` 就是调用了 Observable  含有一个参数的构造方法，然后看一下传入的参数即 `hook.onCreate(new JustOnSubscribe<T>(t)) `这个方法 返回的对象，这里的 `hook` 就是我们前面说过的 RxJavaObservableExecutionHook 类，是RxJavaPlugins中的一个类，用于插入一些你所需要的代码，记录，测试等，最终直接返回了传入的参数，没做对业务逻辑有作用的事情，所以 ` super(hook.onCreate(new JustOnSubscribe<T>(t)));` 方法我们就可以简化为 `new Observer( new JustOnSubscribe<T>(t))`，即直接 new 一个 Observable 对象，传入一个 OnSubscribe 参数，这个结果和我们前面分析的直接创建 Observable 对象的方法` Observable.create(OnSubscribe<T> f) `执行结果是一样的，即这个方法最终其实还是调用了我们前面直接使用的方法，豁然开朗。

```java
protected Observable(OnSubscribe<T> f) {
    this.onSubscribe = f;
}
```

那么再看一下这个 JustOnSubscribe 类：

```java
static final class JustOnSubscribe<T> implements OnSubscribe<T> {
    final T value;

    JustOnSubscribe(T value) {
        this.value = value;
    }

    @Override
    public void call(Subscriber<? super T> s) {
        s.setProducer(createProducer(s, value));
    }
}
```

这个类继承自 OnSubscribe 类，并重写了 `call()` 方法，这里先看一下`    s.setProducer(createProducer(s, value))` 这个方法：

```java
static <T> Producer createProducer(Subscriber<? super T> s, T v) {
    if (STRONG_MODE) {
        return new SingleProducer<T>(s, v);
    }
    return new WeakSingleProducer<T>(s, v);
}
```

这个方法的作用就是根据 STRONG_MODE 参数和传入的 Subscriber 参数和 泛型参数 T 创建一个数量发生器（Producer，是一个接口，它只有一个 `request()`方法，用来在 Observable 和 Subscriber 直接创建一个请求信道，允许 Subscriber 向 Observable 请求确定个数的事件，这个确定的数量将会影响调用 `Observer.onNext(Object)`方法，这样可以限制请求，一般实现该接口的类，都会包含一个 Subscriber 对象和一个待处理的数据，`createProducer(s, t)` 方法中，s 是一个 Subscriber 对象，t 是一个待处理的参数，可以在Producer 中先对 t 进行相应的处理随后，再将数据传送给 Subscriber ，STRONG_MODE  为引用模式，默认为 false，那么就会执行 `new WeakSingleProducer<T>(s, v)`  ，看一下这个方法：

```java
/**
 * This is the weak version of SingleProducer that uses plain fields
 * to avoid reentrancy and as such is not threadsafe for concurrent
 * request() calls.
 * @param <T> the value type
 */
static final class WeakSingleProducer<T> implements Producer {
    final Subscriber<? super T> actual;
    final T value;
  //标记 request 是否已经调用过一次
    boolean once;
    
    public WeakSingleProducer(Subscriber<? super T> actual, T value) {
        this.actual = actual;
        this.value = value;
    }
    
    @Override
    public void request(long n) {
      //这个方法如果调用一次，直接结束方法
        if (once) {
            return;
        }
      //如果发生数据个数小于 0，不合法
        if (n < 0L) {
            throw new IllegalStateException("n >= required but it was " + n);
        }
      //发送 0 个数据，直接结束方法
        if (n == 0L) {
            return;
        }
      //如果这个方法调用走到这里，标记该方法已经调用一次
        once = true;
        Subscriber<? super T> a = actual;
        if (a.isUnsubscribed()) {
            return;
        }
        T v = value;
        try {
          //调用 Subscriber 的 onNext 方法发送数据
            a.onNext(v);
        } catch (Throwable e) {
            Exceptions.throwOrReport(e, a, v);
            return;
        }

        if (a.isUnsubscribed()) {
            return;
        }
      // 调用 Subscriber 的 onCompleted 方法结束数据发送
        a.onCompleted();
    }
}
```

这个类里面主要方法就是 `request(long n)` 方法，而该方法的作用就是只执行一遍 Subscriber 的 `onNext()` 和 `onCompleted() `方法，来发送一次数据并结束订阅过程。

再看一下 Subscriber 的 `setProducer(Producer p)` 方法：

```java
public void setProducer(Producer p) {
    long toRequest;//请求事件限制个数
    boolean passToSubscriber = false;
    synchronized (this) {
        toRequest = requested;
      //将 producer 进行赋值
        producer = p;
        if (subscriber != null) {//一般情况下该结果为假 
            // middle operator ... we pass through unless a request has been made
            if (toRequest == NOT_SET) {
                // we pass through to the next producer as nothing has been requested
              //如果 subscriber != null 且 toRequest == NOT_SET 
              //将 passToSubscriber 设置为 true
                passToSubscriber = true;
            }
        }
    }
    // do after releasing lock
    if (passToSubscriber) {//一般情况该行结果为假  
      //如果 passToSubscriber为 true ，进行递归调用，
      //调用设定的那个 subscriber 的 setProducer 方法
        subscriber.setProducer(producer);
    } else {
        // we execute the request with whatever has been requested (or Long.MAX_VALUE)
        if (toRequest == NOT_SET) {
          //toRequest == NOT_SET 请求事件个数限制失效
            producer.request(Long.MAX_VALUE);
        } else {
          //设定请求个数限制，调用 producer 的 request 方法
            producer.request(toRequest);
        }
    }
}
```

通过源码可以看出 `setProducer(Producer p)` 方法主要完成的任务有：给 Subscriber 对象的 Producer 赋值，调用 `producer.request()` 方法，这样就完成了数据的发送。而上面那些个 if 语句判断情况，其实方法注释已经写的很清楚，我这里简单翻译下：如果设定了其他的 subscriber （通过调用构造函数） ，那么这个方法将会执行 `subscriber.setProducer(producer)` 方法，注意这里是调用你设定那个其他 subscriber 的  `setProducer(producer)` 方法  ；如果没有设定其他的 subscriber  并且 现在这个 subscriber  没有设定限定请求个数（toRequest == NOT_SET） ，那么 `producer.request(Long.MAX_VALUE)` 方法将会调用；如果设定了其他 subscriber   并且限制了请求事件个数（toRequest ！= NOT_SET），那么 `producer.request(toRequest)` 方法将得到执行。

**第三情况，数组长度大于 1：**

调用代码为 `create(new OnSubscribeFromArray<T>(array))`  ，直接调用 Observable 类的 `create(OnSubscribe<T> f)` 方法，这个构造方法前面我们分析过，所以直接看 `new OnSubscribeFromArray<T>(array)` 方法，OnSubscribeFromArray 这个类实现了 OnSubscribe 类，我们先看这个类的构造方法源码：

```java
/**
*@param array 传入的要发送的数组对象
**/
public OnSubscribeFromArray(T[] array) {
    this.array = array;
}
```

这个构造方法很简单，就是将传递进来的参数保存为成员变量，既然 OnSubscribeFromArray 这个类实现了 OnSubscribe 类，我们肯定要去看一下重写的 `call(Subscriber<? super T> child) ` 方法：

```java
@Override
public void call(Subscriber<? super T> child) {
    child.setProducer(new FromArrayProducer<T>(child, array));
}
```

这里 调用的 Subscriber 的 `setProducer(Producer p)` 方法前面我们分析过，所以直接看 `new FromArrayProducer<T>(child, array)` 方法，FromArrayProducer 这个类继承自 Producer 类，先看一下构造方法：

```java
public FromArrayProducer(Subscriber<? super T> child, T[] array) {
    this.child = child;
    this.array = array;
}
```

构造方法就是将传递进来的 Subscriber 对象和数组 array 保存为成员变量。再看一下重写的 `request(long n)` 方法：

```java
@Override
public void request(long n) {
  //请求数量为 0 ，抛出异常
    if (n < 0) {
        throw new IllegalArgumentException("n >= 0 required but it was " + n);
    }
  //请求数量没有限制调用
    if (n == Long.MAX_VALUE) 
        if (BackpressureUtils.getAndAddRequest(this, n) == 0) {
            fastPath();
        }
    } else
      //请求数量做了限制调用
    if (n != 0) {
        if (BackpressureUtils.getAndAddRequest(this, n) == 0) {
            slowPath(n);
        }
    }
}
```

这个方法内部调用分了三种情况：第一种当 请求数量 n 小于 0 的时候直接抛出一个异常；第二种当 请求数量 n == Long.MAX_VALUE 的时候，首先进行了 `BackpressureUtils.getAndAddRequest(this, n) == 0` 判断，这行代码的作用是采用 CAS 操作模式将数量 n 赋值给 request 如果操作成功则返回原始值，这个原始值是 0，即返回值为 0，代表操作成功了，其中 CAS 操作模式主要应用在 Java 并发编程，大家可以 Google 了解下，然后看一下调用的 `fastPath()`  方法代码：

```java
void fastPath() {
    final Subscriber<? super T> child = this.child;
    
    for (T t : array) {
        if (child.isUnsubscribed()) {
            return;
        }
        
        child.onNext(t);
    }
    
    if (child.isUnsubscribed()) {
        return;
    }
    child.onCompleted();
}
```

这个方法也很简单，就是直接遍历数组 array 并调用 Subscriber 的 `onNext(t)` 发送数据，最后调用 `onCompleted()` 方法结束发送；第三种情况调用 `slowPath(n)` 方法源码如下：

```java
void slowPath(long r) {
    final Subscriber<? super T> child = this.child;
    final T[] array = this.array;
    final int n = array.length;
    
    long e = 0L;
    int i = index;

    for (;;) {
        
        while (r != 0L && i != n) {
            if (child.isUnsubscribed()) {
                return;
            }
            //如果循环第 i 次时，没超过请求限制个数和数组长度并且没有取消订阅
          //那么调用 Subscriber 的 onNext 方法发送数据
            child.onNext(array[i]);
            
            i++;
            
            if (i == n) {
                if (!child.isUnsubscribed()) {
                    child.onCompleted();
                }
                return;
            }
            
            r--;
            e--;
        }
        
        r = get() + e;
        
        if (r == 0L) {
            index = i;
            r = addAndGet(e);
            if (r == 0L) {
                return;
            }
            e = 0L;
        }
    }
```

这个方法主要作用其实还是遍历数组  array 并调用 Subscriber 的 `onNext(t)` 发送数据，最后调用 `onCompleted()` 方法结束发送，只不过是添加了请求限制个数限制条件的各种判断。

## 2.4  Observable.map(Func1<? super T, ? extends R> func)

`map()` 函数使用如下：

```java
/**
 * map 函数的用方法：一对一的进行转换
 * 根据传入的一个泛型类类型进行转换为另一个需要的泛型类型，
 * 比如被观察者泛型对象为 Integer 类型，而订阅者或者观察者需要的是 Drawable 类型
 * 对象，此时使用 map 函数进行转换
 * @param integer 输入类型
 */
public void mapTest(Integer integer, final ImageView imageView){
    //指定被观察者对象类型为 Integer
    Observable.just(integer)
            //使用 map 函数指定将 Integer 类型对象换换为 Drawable 类型
            .map(new Func1<Integer, Drawable>() {
                //进行类型转换 并将最终转换成的类型对象返回
                @Override
                public Drawable call(Integer integer) {
                    return imageView.getContext().getResources()
                      .getDrawable(R.drawable.icon_one);
                }
                //形成订阅关系，并指定 观察者接受的对象类型为 Drawable 类型
            })
      .subscribe(new Action1<Drawable>() {
        //观察者接受最终经过 map 函数转换成的对象类型为 Drawable 类型对象
        @Override
        public void call(Drawable drawable) {
            //观察者接受结果进行处理
            imageView.setImageDrawable(drawable);
        }
    });
}
```

下面看一下源码：

```java
/**
 * Returns an Observable that applies a specified function to each item emitted by the 
 * source Observable and emits the results of these function applications.
 * @param func a function to apply to each item emitted by the Observable
 * @return an Observable that emits the items from the source Observable, transformed by the specified function
 */
public final <R> Observable<R> map(Func1<? super T, ? extends R> func) {
    return lift(new OperatorMap<T, R>(func));
}
```

这个方法主要作用就是得到一个新的 Observable 对象， 将原始的 Observable  发送的对象添加一个功能处理一下再将处理后的对象发送出去；看一下方法传递的参数 Func1 类源码：

```java
/**
 * Represents a function with one argument.
 * @param <T> the first argument type
 * @param <R> the result type
 */
public interface Func1<T, R> extends Function {
    R call(T t);
}
/**
 * All Func and Action interfaces extend from this.
 */
public interface Function {

}
```

在 RxJava 中除了有 Func1 还有 Func2 等，其实 FuncX 就是对有参数且有返回值的一类方法的包装而已，将T类型的数据转换为R类型数据。OperatorMap 类源码：

```java
public final class OperatorMap<T, R> implements Operator<R, T> {

    final Func1<? super T, ? extends R> transformer;
//这里的 transformer 就是我们 map(new Func1<Integer, Drawable>() 传入对象
  //实现将泛型 T 转换为 泛型 R 
    public OperatorMap(Func1<? super T, ? extends R> transformer) {
        this.transformer = transformer;
    }
//重写 call 方法，利用 Subscriber<R>将 转换由泛型 <T> 转换得到的泛型 <R>发送出去
  //这里的  Subscriber<? super R> o 就是我们订阅时候传入的
  //subscribe(new Observable <Drawable>() )
  //这里调用 OperatorMap 的 call 方法得到 MapSubscriber<T, R>  parent 对象
  //然后调用 parent 的 onNext 方法完成数据转换和发送
    @Override
    public Subscriber<? super T> call(final Subscriber<? super R> o) {
      //在这个 MapSubscriber 对象中重写 onNext 方法内部完成泛型 T 转换为泛型 R ，并
      //利用传递进来的  Subscriber<? super R> o 方法将转换结果 R 类型数据发送出去
        MapSubscriber<T, R> parent = new MapSubscriber<T, R>(o, transformer);
        o.add(parent);
      // MapSubscriber<T, R>  类继承 Subscriber<? super T> ，这里返回 parent 
      //而函数要求的返回类型是  Subscriber<? super T> ，这里是一个多态
        return parent;
    }
```

OperatorMap 类实现了 Operator 类，而 Operator 类实现了 Func1 类，OperatorMap  内部主要是重写了 call 方法，注意这里看着像是将 Subscriber<R>  转换成一个 Subscriber<T>，并返回Subscriber<? super T>  对象，我记得之前看其他 RxJava 源码分析有是这么写的，但是我感觉这样写容易误解的，其实这里是通过 OperatorMap 构造函数传递进来的 Func1<? super T, ? extends R>  transformer 将输入类型 T 转换为 R类型，然后利用 OperatorMap 重写的 call 方法传递进来的 Subscriber<? super R>  o 的 onNext(R)   方法将转换结果 R发送出去，然后返回  MapSubscriber<T, R> 对象，这个MapSubscriber<T, R>  实现了Subscriber<T> 类并重写了 ` onNext(T t)` 方法，这些转换都是在这个重写的  ` onNext(T t)`  方法中进行 ，所以当我们调用 `OperatorMap. call(final Subscriber<? super R> o) `得到 返回的 MapSubscriber<T, R> 对象 parent （Subscriber 类型），那么再去调用`parent.onNext()` 方法 就会完成将输入类型 T 转换为 R类型并发送出去的效果，一起看一下  MapSubscriber 这个类：

```java
static final class MapSubscriber<T, R> extends Subscriber<T> {
    
    final Subscriber<? super R> actual;
    
    final Func1<? super T, ? extends R> mapper;

    boolean done;
    
    public MapSubscriber(Subscriber<? super R> actual, Func1<? super T, ? extends R> mapper) {
        this.actual = actual;
        this.mapper = mapper;
    }
    //在这个方法内部完成 T转换为 R 并发送出去
    @Override
    public void onNext(T t) {
        R result;
        
        try {
          //通过 Func1 的 call 方法将输入泛型 T 转换为 R 
          //这个方法就是我们在使用 map 函数需要传入一个 Func1 对象，并重写
          //的 call 方法，就是在这里得到的调用进行转换
            result = mapper.call(t);
        } catch (Throwable ex) {
            Exceptions.throwIfFatal(ex);
            unsubscribe();
            onError(OnErrorThrowable.addValueAsLastCause(ex, t));
            return;
        }
        //通过 Subscriber 的 onNext 方法将转换得到的结果 T 发送出去
        //这里的  actual 就是我们订阅时候传入的
  //subscribe(new Observable <Drawable>() )
        actual.onNext(result);
    }
```

这个类内部也很简单，主要是重写了 Subscriber 的 `onNext(T t)` 方法，  就是通过构造方法传递进来的 Func1<? super T, ? extends R> 对象将输入类型 T 转换为 R类型，然后在通过传递进来的 Subscriber<? super R> 的 `onNext(R)`  将转换后得到的结果 R 发送出去。

最后看一下 ` lift(final Operator<? extends R, ? super T> operator)` 方法源码：

```java
/**
 * Lifts a function to the current Observable and returns a new Observable that when subscribed
 * to will pass the values of the current Observable through the Operator function.
 * In other words, this allows chaining Observers together on an Observable for
 * acting on the values within  the Observable
 * @param operator the Operator that implements the Observable-operating 
 * function to be applied to the source  Observable       
 * @return an Observable that is the result of applying 
 * the lifted Operator to the source Observable
 */
public final <R> Observable<R> lift(final Operator<? extends R, ? super T> operator) {
  //onSubscribe 为原始数据，即创建原始 Observable 时候传递进来的 onSubscribe 对象
    return new Observable<R>(new OnSubscribeLift<T, R>(onSubscribe, operator));
}
```

这个方法的主要作用就是对当前的 Observable 对象进行一个功能变化，并返回一个新的 Observable 对象，当这个新的 Observable 对象被订阅之后，就可以通过这个 Operator 对象的功能变换来发送当前 Observable  对象的数据。换句话说。这个方法通过在一个特定的 Observable 内部使得观察者 Observers 和 被观察者 Observable 来接发数据形成关联。lift 方法内部直接调用了 Observable 的构造函数创建一个 Observable 并返回，而这里传入的参数 OnSubscribe 和 Operator 我们前面已经分析过，下面只要看 OnSubscribeLift 这个类：

```java

public final class OnSubscribeLift<T, R> implements OnSubscribe<R> {
    
    static final RxJavaObservableExecutionHook hook = RxJavaPlugins.getInstance().getObservableExecutionHook();

    final OnSubscribe<T> parent;

    final Operator<? extends R, ? super T> operator;

    public OnSubscribeLift(OnSubscribe<T> parent, Operator<? extends R, ? super T> operator) {
        this.parent = parent;
        this.operator = operator;
    }
//重写 call 方法，传入我们订阅 Obdervable 时候创建的 Subscriber<? super R> 对象
  //这里的 call 方法在我们最后 subcribe 订阅的时候进行调用
    @Override
    public void call(Subscriber<? super R> o) {
        try {
          //调用 Operator 的 call 方法 返回 一个新的 Subscriber(MapSubscriber) 对象
          //并将我们订阅 Obdervable 时候创建的 Subscriber<? super R> 对象传递给 
          //(operator).call(o)方法
            Subscriber<? super T> st = hook.onLift(operator).call(o);
            try {
                // new Subscriber created and being subscribed with so 'onStart' it
                st.onStart();
              //调用最开始创建 Obsaverble 时候创建的那个的 OnSubscribe 的 call 方法
              //传入新的 operator 的 MapSubscriber 对象
              //在这个新的 Subscriber（MapSubscriber) 的 onNext(T t) 方法内部我们完成了
              //将输入类型 T 转换为 R ，然后
              //调用了Subscriber<? super R> o.onNext(R) 将转换结果发送出去
                parent.call(st);
            } catch (Throwable e) {
               
                Exceptions.throwIfFatal(e);
                st.onError(e);
            }
        } catch (Throwable e) 
            o.onError(e);
        }
    }
}
```

这个类继承自 OnSubscribe 重写了 call 方法，在 call 方法内部通过调用 operator.call(o) 方法得到一个新的 Subscriber，最后将这个 Subscriber 传递给原始 OnSubscribe 的 call 方法，到这里就完成了整个转换操作，剩下的就是我们在 2.1 章节分析过的 Observable.subscribe 方法部分了，即在执行 map 函数转换之后得到 Observable<R> 对象然后进行 subscribe 订阅的时候 OnSubscribeLift 的 call 方法就会执行调用，然后在这个重写的 call 方法内部会调用 operator.call(o) 得到 Subscriber(MapSubscriber) 对象，接着就会调用第一次创建 Observable<T> 时候创建的 OnSubscribe 的call 方法即  parent.call(st)，其中传递进去的参数 st  即为新的 Subscriber(MapSubscriber) 对象，在其内部完成转换操作 。

**对象变化**

这里面最后总结一下，其实 RxJava 这个 lift （包括其他 map 、flatMap) 操作符就是完成各种对象的变换，而变换主要涉及到的就是 Subscriber、Observable、OnSubscribe  这三个对象的各种转换。

**Observable的变化**

1. 每个操作符都会新建一个 Observable 和一个新建的 OnSubscribe   (下游Observable和下游 OnSubscribe  )；

2. 下游 OnSubscribe 中持有上游的 OnSubscribe；

3. 下游的 OnSubscribe 先调用 Operator 拿到针对上游的 Subscriber，然后就可以调用上游OnSubscribe.call() 方法了。
   **流程图（代码分解）：**

   ![map](image\map.png)

   **当 Subscribe 订阅时代码执行流程：**

    ![map2](image\map2.png)

## 2.5 Observable.flatMap(Func1<? super T, ? extends Observable<? extends R>> func)

```java
/**
 * flatMap 函数使用，实现一对多转换，其实 flatmap 是返回一个 observable 对象，所以可以继续操作
 * flatMap 最终返回的是一个 Observable 对象
 * 传入一组学生List<Student> 对象，然后取出每一个学生，
 * 打印每一个学生的多个成绩
 * @param students
 */
public void flatMapTest(final List<Student> students){
    //直接订阅最终结果 Course ，即订阅类型为 Course 类型
    Subscriber <Course> subscriber=new Subscriber<Course>() {
        @Override
        public void onCompleted() {
            Log.d(TAG,"onCompleted");
        }

        @Override
        public void onError(Throwable e) {

        }
//取出 Course 对象进行打印
        @Override
        public void onNext(Course course) {
        Log.d(TAG,course.getName());
        }
    };
//传入一组List<Student> 对象
    Observable.from(students)
            //将一个 Observable<Student>> 对象
            //转换为多个 Observable<Course>> 对象
            // 并激活进行发送 Course 对象，统一发送到subscribe 订阅的
            // observable 对象内
            //其实flatMap 函数是创建一个新的 Observable 对象，
            //这个新的 Observable 对象就像一个代理
            //接受原始 Observable 对象发送的事件并进行处理然后发送给 subscriber 这个订阅者
            .flatMap(new Func1<Student, Observable<Course>>() {
                @Override
                public Observable<Course> call(Student student) {
                    //flatMap 最终返回的是 Observable 对象
                    //1、使用传入的事件对象创建一个 Observable 对象
                    //2、并不发送这个 Observable, 而是将它激活，于是它开始发送事件
                   //3、每一个创建出来的 Observable 发送的事件，都被汇入同一个Observable 中
                    // 而这个 Observable 负责将这些事件统一交给 订阅者 Subscriber 
                    return  Observable.from(student.getCourses());
                }
            })
            .subscribe(subscriber);
}
```

## 2.6 Compose 操作符

在说 **compose**之前要先介绍下[Transformer](http://reactivex.io/RxJava/javadoc/rx/Observable.Transformer.html)
Transformer 实际上就是一个`Func1, Observable>`，换言之就是：可以通过它将一种类型的 Observable 转换成另一种类型的 Observable，比如通过 Transformer 将`Observable<T>`转换成了`Observable<R>` ，`compose()`和 `lift()` 的区别在于，**lift() 是针对事件项和事件序列的，而 compose() 是针对 Observable 自身进行变换。**这个功能其实 flatmap 函数也能实现，但是 compose 操作符实现了代码重用，只需要写一个 transfrmer 就可以利用 compose 操作符实现反复利用，不用每次都写 flatmap 写重复代码，源码：

```java
public interface Transformer<T, R> extends Func1<Observable<T>, Observable<R>> {
    // cover for generics insanity
}
```

以下代码可以将 Obsaverble<Integer> 变换为 Obsaverble<String>：

```java
public class LiftAllTransformer implements Observable.Transformer<Integer, String> {
  @Override
  public Observable<String> call(Observable<Integer> observable) 
  {return observable.lift1().lift2().lift3().lift4();}}
.....................
Transformer liftAll = new LiftAllTransformer();
//obsaverble1 为 Obsaverble<Integer> 类型，subscriber1 为 Subscriber<String> 类型
//通过 compose 操作符完成了转变
observable1.compose(liftAll).subscribe(subscriber1);
observable2.compose(liftAll).subscribe(subscriber2);
observable3.compose(liftAll).subscribe(subscriber3);
observable4.compose(liftAll).subscribe(subscriber4);
```

使用示例：

```java
/**
 * compose 函数实现对整体 Observable 对象进行变化，产生新的 Observable 对象返回
 * @param i
 */
public void composeTest(final Integer i){
    LiftAllTransformer liftAllTransformer=new LiftAllTransformer();
    Observable<Integer> observable=Observable.create(new Observable.OnSubscribe<Integer>() {
        @Override
        public void call(Subscriber<? super Integer> subscriber) {
            subscriber.onNext(i);
        }
    });

    observable.compose(liftAllTransformer).subscribe(new Action1<String>() {
        @Override
        public void call(String s) {
            Log.d(TAG,s);
        }
    });
}

/**
 * Transformer 实现对 Observable 对象整体变化，并返回新的 Observable对象
 * Created by zhangpeng on 2016/8/28.
 */
public class LiftAllTransformer implements Observable.Transformer<Integer,String>{
    //对原始 Observable 对象进行变化并返回新的 Observable对象
    @Override
    public Observable<String> call(Observable<Integer> integerObservable) {
        return integerObservable.lift(new Observable.Operator<String, Integer>() {
            @Override
            public Subscriber<? super Integer> call(final Subscriber<? super String> subscriber) {
                return new Subscriber<Integer>() {
                    @Override
                    public void onCompleted() {

                    }

                    @Override
                    public void onError(Throwable e) {

                    }

                    @Override
                    public void onNext(Integer integer) {
                        subscriber.onNext(integer+"第一次变化==");
                    }
                };
            }
        }).lift(new Observable.Operator<String, String>() {
            @Override
            public Subscriber<? super String> call(final Subscriber<? super String> subscriber) {
                return new Subscriber<String>() {
                    @Override
                    public void onCompleted() {

                    }

                    @Override
                    public void onError(Throwable e) {

                    }

                    @Override
                    public void onNext(String s) {
                    subscriber.onNext(s+"第二次变换==");
                    }
                };
            }
        });
               
    }
}

```

## 2.7 变换的原理：lift()

这些变换虽然功能各有不同，但实质上都是**针对事件序列的处理和再发送**。而在 RxJava 的内部，它们是基于同一个基础的变换方法： `lift(Operator)`

```java
public final <R> Observable<R> lift(final Operator<? extends R, ? super T> operator) {
    return new Observable<R>(new OnSubscribeLift<T, R>(onSubscribe, operator));
}
```

这个变换原理我们在分析 map 函数的时候已经分析过了，总结起来就是通过新建一个 Obsaverble 和 OnSubscribe 来发送数据，而发送的数据是通过 Operator<? extends R, ? super T> operator 来实现变换的。主要分为以下几步：

  1.在执行`lift()`后会创建一个新的  Observable 我们标记为 Observable2，加上之前的原始  Observable 我们标记为 Observable1，现在有两个  Observable ； 
  2.在创建新 Observable2 的时候会创建一个新的 OnSubscribe2 标记为 OnSubscribe2 ， 加上之前的原始 Observable1 中的原始  OnSubscribe1，也就有了两个 OnSubscribe； 
  3.当用户调用 `lift()` 后 再去调用 `subscribe()` 的时候，其实是使用的 `lift()` 所返回的新的 Observable2 ，于是它所触发的 `onSubscribe2.call(subscriber1)`，即在 `lift()` 中生成的那个 OnSubscribe2； 
  4.而这个新  OnSubscribe2 的 `call()` 方法中会持有原始的  onSubscribe1 ，就是指的原始  Observable1 中的原始  OnSubscribe1 ，在这个 `call()`方法里，用 `operator.call(subscriber1)` 生成了一个新的  Subscriber2，Operator  就是在这里，通过自己的 `call()` 方法将新  Subscriber2  和原始  Subscriber1 进行关联，并插入自己的『变换』代码以实现变换，然后利用这个新  Subscriber2 向原始  Observable1 进行订阅。  这样就实现了 `lift()` 过程，有点像一种代理机制，通过事件拦截和处理实现事件序列的变换。RxJava 都不建议开发者自定义 Operator 来直接使用 lift()，而是建议尽量使用已有的 `lift()` 包装方法（如 `map()` `flatMap()` 等）进行组合来实现需求，因为直接使用 `lift()` 非常容易发生一些难以发现的错误。

下面这是一个将事件中的 `Integer` 对象转换成 `String` 的例子:

```java
/**
* 利用 lift 函数实现对对象变换操作
* 这里实现对被观察者 Integer 类型转换为 String 类型
* @param i
*/
public void liftTest(final Integer i){
   //被观察者指定泛型为 Integer
   Observable<Integer> observable= Observable.create(new Observable.OnSubscribe<Integer>() {
       @Override
       public void call(Subscriber<? super Integer> subscriber) {
           subscriber.onNext(i);
       }
   });

   //实现lift变化，将 Integer 类型转换为 String 类型
   observable.lift(new Observable.Operator<String, Integer>() {

       @Override
       public Subscriber<? super Integer> call(final Subscriber<? super String> subscriber) {
           //返回一个新的 subscriber 对象
           return new Subscriber<Integer>() {
               @Override
               public void onCompleted() {
                   Log.d(TAG,"onCompleted");
               }

               @Override
               public void onError(Throwable e) {

               }
       //实现类型转换
               @Override
               public void onNext(Integer integer) {
                       subscriber.onNext(integer+"");
               }
           };
       }
   })
           //对被观察者进行订阅，指定订阅对象返回 String 类型
           .subscribe(new Action1<String>() {
               //对订阅结果进行处理
               @Override
               public void call(String s) {
                   Log.d(TAG,s);
               }
           });
}
```



以上是本次学习笔记内容，完结！

参考文章：

[给 Android 开发者的 RxJava 详解](https://gank.io/post/560e15be2dca930e00da1083)

