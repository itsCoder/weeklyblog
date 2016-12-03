# Android 项目架构- MVP基础篇
##写在前面
今天我们聊一聊传说中 Android 框架 MVP， 想必大家早就听过了， 最早接触这个名词时在今年1月份左右，那时候在 medium 上看到的一篇关于它的介绍， 看的也不是很明白， 不过被一些设计思想吸引。然后参阅一些相关的资料， 发现 Google 在 Github 有一个实现的代码， 然后模仿它例子也在 Github 上创建第一个 Android Demo。

现在真后悔当时没有写一篇相关的文章， 总给自己理解不深找借口， 越拖发现类似的文章越多， 现在都已经烂大街了， 本来不想凑这个热闹的，好吧， 管它了，就当来补债的 。当时还看了 MVP、MVVM、Facebook Flux 比较的文章，现在 Android 主流框架是： Retrofit + OkHttp + RxJava + MVP + Dagger 2。

言归正传，今天我们主要说明一下情况：
+ 什么是 MVP， 有什么优缺点。
+ 如何实现。
+ 实际项目中如何改造。

## MVP
全称 Model-View-Presenter，其中 Presenter 解耦了 Model 与 View，使得每个模块的职责更加单一，Model 负责获取数据，View 只关心视图的绘制，Presenter 关联 Model 和 View 处理业务逻辑。

需要注意的是，「Model」 这个词并不正确。严格意义上来说，它指的应该是检索或控制一个 Model 的业务逻辑层。举个例子，比如你的数据库里面包含了 User，而你的 View 想要显示一个 User 列表，那么 Presenter 会引用数据库中的业务逻辑层，查询一个 User 列表，如下图：

![](http://7xnilf.com1.z0.glb.clouddn.com/mvp.png)

有兴趣的大家可以参考下面的文章， 概念写的比较详细，里面讲到了， 它的的发展历程、MVX  解析。
> Android MVP 详解（上）

>*http://www.jianshu.com/p/9a6845b26856*

我们把 MVP 的优缺点单独说一下， 「知己知彼，访客百战不殆」，不然想想下面场景：
>女朋友：你喜欢我什么？

>我：(⊙v⊙)嗯，。。。

>女朋友：连喜欢我什么都不知道， 走跟我回去跪方便面。

>我：。。。。。

**优点**：
+ 降低耦合度，实现了Model和View真正的完全分离，可以修改View而不影响Model
+ 方便进行单元测试， 这个很重要。
+ Presenter可以复用，一个Presenter可以用于多个View。
+ View 可以进行组件化。在 MVP 当中，View 不依赖 Model。这样就可以让View 从特定的业务场景中脱离出来，可以说 View 可以做到对业务完全无知。它只需要提供一系列接口提供给上层操作。这样就可以做到高度可复用的View组件。

**缺点**：
+ Presenter 中除了应用逻辑以外，还有大量的 View->Model，Model->View 的手动同步逻辑，造成 Presenter 比较笨重，维护起来会比较困难。
+ 如果 Presenter 过多地渲染了视图，往往会使得它与特定的视图的联系过于紧密。一旦视图需要变更，那么 Presenter 也需要变更了。
+ View 实现接口比较多，显得代码可读性增高。

## DEMO 展示
我最开始看的 Demo 就是 Google 官网提供的，给我很多启发，项目主要展示类似便签的应用。
>项目地址:

> *https://github.com/googlesamples/android-architecture*

下面是它框图：
![](http://7xnilf.com1.z0.glb.clouddn.com/google-arch.jpg)

本来想拿官方源码进行剖析的，发现以及有人已经写了， 而且还写比我要好， 你说气人不， 下面是它的分析。
> Android官方MVP架构示例项目解析

>[*http://www.infoq.com/cn/articles/android-official-mvp-architecture-sample-project-analysis*]

## 实践项目使用
我们来思考一个问题， 我们通过观察上一章 Google Demo 我们会发现一个问题， 事实上每个功能块的代码都是类似的，只是细节上会有所不同。重构的原则告诉我们，这些地方是可以进行重构的。在这个时候，一般会首先想到把一些相同的功能块抽象成一个基类。例如网络错误处理、服务器拒绝请求返回的错误处理等。但是随着项目的进行，很快就会发现，类文件量、代码量仍然会增加得很快，随之带来的问题是项目的管理会变得越来越复杂。

不满现状的我们又进行方案的重组和选择， 最后想到可以通过泛型和抽象，进一步简化 MVP 框架。所有view的基类是 IView（activity或fragment 也是这 view）。
**IView**:
```java
public interface IView {
}
```
**IPresenter**:
```java
public interface IPresenter<V extends IView> {
    void attachView(V view);
    void detachView(boolean retainInstance);
}
```
上面我们提到，我们把 Activity 和 Fragment 看成 View。所以我们提供了 MVPActivity 和 MVPFragment 作为他们的基类。

这里我们仅仅看 **MVPActivity**， MVPFragment类似。
```java
public abstract class MVPActivity<V extends IView, P extends IPresenter<V>>
        extends BaseActivity implements IView, IMvpBase<V>{
    protected P presenter;

    @Override
    public void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        presenter = createPresenter();
        presenter.attachView(getMvpView());
    }

    @Override
    protected void onDestroy() {
        super.onDestroy();
        presenter.detachView(false);
    }

    public abstract P createPresenter();

    @Override
    public V getMvpView() {
        return (V)this;
    }
}
```

主要思想是 IView 会关联一个 IPresenter, 并且管理 IPresenter 的生命周期。大家从上面的代码片段可以看到， 通常 presenter 是绑定在该生命周期上。所有的初始化或者清理工作都放在 `presenter.onAttach()` 和 `presenter.detach()` 上进行。想大家注意到 IPresenter 是一个接口。 我们还提供一个 **BasePresenter**,  它只持有 View 的弱引用， 从而避免内存泄漏。所有，当 presenter 想要调用 view 的方法是， 我们需要判断 `isViewPresenter()` 并使用 `getView()`来获取引用，以坚持view是否连接当了 presenter。

![](http://7xnilf.com1.z0.glb.clouddn.com/mvparch.png)

看一下上图我项目结构， 其中 BaseLib 下有两个库，分别是 baseapp、mvplib。
+ baseapp：主要包含BaseActiviy、AppMain 类
+ mvplib： 主要是时 MVP 封装框架，更高效开发
+ app: 主要是我们的演示程序

>项目GitHub：
>*https:/github.com/jasonim/mvparchitecture*

这样我们上面基本上解决上面我们提出的问题， 当然我们经常可能遇到屏幕旋转的问题， 这样一般处理数据持久化问题， 一般的做法是在 `onSaveInstanceState()`处理， 简单做法记录数据。当然我们可以通过状态来修复view的状态。就不在这里说了， 感兴趣的可以参考 **mosby** 的做法。

>mosby:
>*https://github.com/sockeqwe/mosby*

## 写在最后
「一万个心中有一万个哈姆雷特」，没有任何事情是绝对不变的， 框架也是一样，对 MVP 的争论不断：
>MVP，太多的接口，如果项目大的话，还可以，如果不大，就建太多的类。感觉有点笨重。。。

把握以下两点就行：
+ 我们引用项目框架目的是什么？是不是为了业务扩展，如果没有扩展需求， 什么项目框架也不需要。
+ 每个人的理解不一样，实践中找到一个适合自己的就好, 如果感觉 MVVM 的双向绑定很好，那就改造一下， 没有最好，只有更好。实践、再实践。 

下一篇我们来聊一聊， 怎样在 MVP 中加入 dagger、rxjava；怎么通过组件化优化项目结构。
