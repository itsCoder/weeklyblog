# RxLifecycle 使用与原理

RxJava 已经成为当前 Android 圈中很火的一门技术了，基于 Observable 的事件流配合上各种操作符黑魔法，让人爽到不能呼吸 😆。RxLifecycle 作为 RxJava 的扩展库，用起来也是很爽的。

> 本文基于 RxLifecycle v0.5 进行的分析，最新版本已经是 v0.7 了，项目结构略有变化，但是原理部分基本相同。

### 使用姿势
[RxLifecycle](https://github.com/trello/RxLifecycle) 的使用姿势比较简单，而且项目文档也很清晰，直接让自己的 Activity 基类继承自 RxActivity 就 ok了。

使用时就调用一下  `compose()` 轻松绑定！

```java
myObservable
    .compose(bindUntilEvent(ActivityEvent.DESTROY))
    .subscribe();
```

```java
myObservable
    .compose(bindToLifecycle())
    .subscribe();
```



### 实现原理

RxLifecycle 使用简单，原理也不复杂，只是设计得十分巧妙。

既然我们都要继承自 RxActivity，那就从这个类开始看吧。

![](http://ww1.sinaimg.cn/large/65e4f1e6jw1f7wpu70y6rj218a02yq3n.jpg)

RxActivity 在其内部维护了一个 BehaviorSubject 对象。Subject 对象相当于一个管道，一边可以接收数据，从另外一边发射出去。而 BehaviorSubject  继承自 Subject，并且还多了一项特性，在新的观察者订阅时，会首先发射**一个最近发射过的元素**([参考文档看这里](http://reactivex.io/RxJava/javadoc/rx/subjects/BehaviorSubject.html))。

![](http://ww2.sinaimg.cn/large/801b780ajw1f7wpv58ugmj21di0zg0yo.jpg)

其次，在 Activity 到达各个生命周期的时候，`lifecycleSubject` 会接收到对应的生命周期事件。

接下来看我们最常用的 `bindToLifecycle()` 方法。

![](http://ww2.sinaimg.cn/large/801b780ajw1f7wpvos4lkj21je0623zz.jpg)

跳转到 RxLifecycle 类中：

![](http://ww1.sinaimg.cn/large/801b780ajw1f7wpw1baa0j21gw06cq4l.jpg)

![](http://ww2.sinaimg.cn/large/801b780ajw1f7wpwc3qocj21hk0kk79m.jpg)

所谓的 `ACTIVITY_LIFECYCLE` 其实就是一个函数 Func1 而已，把 Activity 的生命周期事件进行一下变换，`CREATE` 对应 `DESTROY`，`RESUME` 对应 `PAUSE` 等等。看到这里是不是已经看出了一点端倪了，其含义就是 **把当前的 Activity 生命周期转换成对应的需要被销毁的时候的生命周期。**


最后都走到了这个方法 `RxLifecycle#bind(rx.Observable<R>, rx.functions.Func1<R,R>)`。

![](http://ww2.sinaimg.cn/large/801b780ajw1f7wpwkohmzj21c40pmgse.jpg)


这里是整个代码中最核心、最巧妙的部分。

首先第一个参数是 `lifecycle`，也就是 RxActivity 里的那个 BehaviorSubject 对象，这里作为 Observable 来使用。它会不断发射 **ActivityEvent**，也就是 Activity 的生命周期事件。

先介绍下里面用到的几个操作符：

- [share](http://reactivex.io/RxJava/javadoc/rx/Observable.html#share())：简单来说就是让 Observable 支持多订阅。
- [take](https://mcxiaoke.gitbooks.io/rxdocs/content/operators/Take.html)：只发射前面的 N 项数据。
- [skip](https://mcxiaoke.gitbooks.io/rxdocs/content/operators/Skip.html)：跳过 Observable 发射的前N项数据。
- [combineLatest](https://mcxiaoke.gitbooks.io/rxdocs/content/operators/CombineLatest.html)：当两个 Observables 中的任何一个发射了数据时，使用一个函数结合每个 Observable 发射的最近数据项，并且基于这个函数的结果发射数据。
- [takeUntil](https://mcxiaoke.gitbooks.io/rxdocs/content/operators/Conditional.html#takeuntil)：当第二个 Observable 发射了一项数据或者终止时，丢弃原始 Observable 发射的任何数据。
- [onErrorReturn](https://mcxiaoke.gitbooks.io/rxdocs/content/operators/Catch.html#onerrorreturn)：让 Observable 遇到错误时发射一个特殊的值并且正常终止。
- [takeFirst](https://mcxiaoke.gitbooks.io/rxdocs/content/operators/First.html#takefirst)：传入一个 Func1 给 takeFirst, 发射出能够使得这个 Func1 为 true 的元素。

>  这里再推荐一个网站 [http://rxmarbles.com/](http://rxmarbles.com/)，可视化交互式体验 RxJava 的操作符，能更好地理解各个操作符的含义。

先分析代码块的内层：

首先是 `combineLatest` 操作符需要接收两个 Observable 和一个函数 Func2。

- 第一个数据源由 `lifecycle` 只发射第一个元素，即当前所在的生命周期事件，然后进行一下 map 变换，得到应该被销毁的生命周期 (ActivityEvent)，并且这个 Observable 只有这一个元素；
- 第二个数据源由 `lifecycle` 跳过第一个元素构成，即后续生命周期事件所组成的数据流；
- 处理函数：当两个数据源的数据相等时，即 Activity 到达需要被销毁的生命周期时，返回 **true**，发射出一个元素。

然后执行 `onErrorReturn()`，这是对异常进行处理，使 Observable 不会因为错误而中断。

最后是 `takeFirst(Func1)`，返回第一个满足 `Func1` 的元素值。

这样一来，等到 Activity 到达需要被销毁的生命周期时，`combineLatest()` 所产生的 Observable 会发射出一个值。

然后分析外层代码：

`source.takeUntil(observable);`

source 就是我们在正常代码中使用到的 Observable，我们希望自动结束的 Observable。当内部 observable 发射出一个值时，source 丢弃掉后续数据，即终止掉这个 Observable。

通过以上的代码，就完成了 Observable 自动绑定到 Activity 的生命周期，并且自动结束。

另外 `RxLifecycle#bindUntilEvent()` 方法，用于将 Observable 绑定到特定的生命周期上，原理跟前面所介绍的基本一样，或者说更为简单。

![](http://ww2.sinaimg.cn/large/801b780ajw1f7wpwrm6oyj21c00iqgpn.jpg)

判断后续的生命周期等于传入的生命周期时，就停止接收数据。

对于 Fragmeng、View 等，原理和分析方法基本一致，不再详述了。


### 坑与提示

#### 1. BaseActivity 不方便继承 RxActivity

直接让 BaseActivity 实现 `ActivityLifecycleProvider` 接口，然后把 RxActivity 里的代码拷贝到 BaseActivity 就 ok 了。

#### 2. 在**界面切换**时使用 RxLifecycle 有个小坑

比如从 ActivityA 启动 ActivityB， 生命周期如下：
> A.onPause -> B.onCreate -> B.onResume -> A.onStop

如果 A 中启动 B 的同时，订阅了一个 Observable，同时调用了 `compose(bindToLifecycle())`，可能出现如下的 bug：
订阅时，A 还处于 `RESUME` 周期，`bindToLifecycle()` 会对应到 `PAUSE` 周期给销毁掉。这个时候启动了 B，A 会进入 'PAUSE' 生命周期，这是前面的 Observable 会被停掉。如果任务还没有执行完成的话，结果会达不到我们的预期。

所以这里应该使用 `bindUntilEvent(ActivityEvent.DESTROY)`。


### 参考
- [trello/RxLifecycle](https://github.com/trello/RxLifecycle)
- [ReactiveX/RxJava 文档中文版](http://mcxiaoke.gitbooks.io/rxdocs/content/)
- [RxMarbles](http://rxmarbles.com/)


