Activity / Fragment 的生命周期是每个 Android 开发者最最基础的知识点。所以特别有必要自己整理一番。总看别人博客和书上的死知识，还不如自己动手实践，然后输出要印象深刻，理解透彻。

<!-- more -->

<!-- toc -->

- [Activity 生命周期](#activity-生命周期)
  - [正常情况下的生命周期分析](#正常情况下的生命周期分析)
  - [异常状态下的生命周期](#异常状态下的生命周期)
- [Fragment](#fragment)
  - [普通的 Fragment](#普通的-fragment)
  - [ViewPager 中的 Fragment](#viewpager-中的-fragment)
- [启动模式](#启动模式)
  - [Activity 的四种启动模式](#activity-的四种启动模式)
  - [具体实践](#具体实践)
- [参考文档](#参考文档)

<!-- tocstop -->

### Activity 生命周期


#### 正常情况下的生命周期分析

![22:39:52.jpg](http://ww2.sinaimg.cn/large/006tKfTcgw1f6i3dzl70yj30fj0j2q41.jpg)

1. 针对一个特定的 Activity ，第一次启动，回调如下：`onCreate` —-> `onStart` —-> `onResume`
>Log 日志
>D/KLog: (MainActivity.java:19) onCreate
>D/KLog: (MainActivity.java:44) onStart
>D/KLog: (MainActivity.java:62) onResume

1. 切换回到桌面的时候，回调如下：`onPause` —-> `onStop`
> Log 日志
> D/KLog: (MainActivity.java:50) onPause
> D/KLog: (MainActivity.java:68) onStop

1. Back 键退出的话，最后会 `onDestroy`

2. 启动一个新的 Activity , 我们看看两个 Activity 的生命周期：
> Log 日志
> D/KLog: (MainActivity.java:50) onPause
> D/KLog: (OtherActivity.java:25) onCreate
> D/KLog: (OtherActivity.java:31) onStart
> D/KLog: (OtherActivity.java:49) onResume
> D/KLog: (MainActivity.java:68) onStop
> 可以得到顺序是：`onPause(A)` —-> `onCreate(B)` —-> `onStart(B)` —->  `onResume(B)` —-> `onStop(A)`

1. 这个时候我们 Back 回到第一个 Activity 时发生的回调：
> Log 日志
> D/KLog: (OtherActivity.java:37) onPause
> D/KLog: (MainActivity.java:56) onRestart
> D/KLog: (MainActivity.java:44) onStart
> D/KLog: (MainActivity.java:62) onResume
> D/KLog: (OtherActivity.java:55) onStop
> D/KLog: (OtherActivity.java:61) onDestroy
> 可以得到顺序是： `onPause(B)` —-> `onRestart(A)` —-> `onStart(A)` —-> `onResume(A)` —-> `onStop(B)` —-> `onDestroy(B)`

1. 如果我在 B Activity 中的 `onCreate` 回调中直接 `finish()`：
> Log 日志
> D/KLog: (MainActivity.java:50) onPause
> D/KLog: (OtherActivity.java:25) onCreate
> D/KLog: (MainActivity.java:62) onResume
> D/KLog: (OtherActivity.java:62) onDestroy
> 我们发现 B Activity 只会执行 `onCreate` 和 `onDestroy`。

1. 接下来我们启动一个特殊的 Activity （半透明或者对话框样式）到关闭它：
> Log 日志
> D/MainActivity: onPause
> D/DialogActivity: onCreate
> D/DialogActivity: onStart
> D/DialogActivity: onResume
> D/DialogActivity: onPause
> D/MainActivity: onResume
> D/DialogActivity: onStop
> D/DialogActivity: onDestroy



在正常使用应用的过程中，前台 Activity 有时会被其他导致 Activity 暂停的可视组件阻挡。 例如，当半透明 Activity 打开时（比如对话框样式中的 Activity ），上一个 Activity 会暂停。 只要 Activity 仍然部分可见但目前又未处于焦点之中，它会一直暂停。





<div class="tip">
问题：如果是启动一个普通的 Dialog ，Activity 的会执行 onPause 吗？
</div>

答案是不会的，我们可以这样理解，普通的 dialog 是依附在本 Activity 的，相当于是一个整体，所以它本身的焦点也是属于 Activity 的。故不会调用 onPause 。

#### 异常状态下的生命周期

![22:41:53.jpg](http://ww1.sinaimg.cn/large/006tKfTcgw1f6i3g0h83uj30d1067jrs.jpg)


`onSaveInstanceState` 方法只会出现在 `Activity` 被异常终止的情况下，它的调用时机是在 `onStop` 之前，它和 `onPause` 方法没有既定的时序关系，可能在它之前，也可能在它之后。当 `Activity` 被重新创建的时候， `onRestoreInstanceState` 会被回调，它的调用时机是 `onStart` 之后。
系统只会在 `Activity` 即将被销毁并且有机会重新显示的情况下才会去调用 `onSaveInstanceState` 方法。
当 `Activity` 在异常情况下需要重新创建时，系统会默认为我们保存当前 `Activity` 的视图结构，并且在 `Activity` 重启后为我们恢复这些数据，比如文本框中用户输入的数据、`listview` 滚动的位置等，这些 `view` 相关的状态系统都会默认为我们恢复。具体针对某一个 `view` 系统能为我们恢复哪些数据可以查看 `view` 的源码中的 `onSaveInstanceState` 和 `onRestoreInstanceState` 方法。

Demo 代码：
```java
@Override
protected void onSaveInstanceState(Bundle outState) {
    super.onSaveInstanceState(outState);
    KLog.d(getClass().getSimpleName(),"onSaveInstanceState");
    outState.putString(STATE, "test");
}

@Override
protected void onRestoreInstanceState(Bundle savedInstanceState) {
    super.onRestoreInstanceState(savedInstanceState);
    KLog.d(getClass().getSimpleName(),"[onRestoreInstanceState]: " + savedInstanceState.getString(STATE));
}
```

为了方便我们旋转下屏幕来异常终止 Activity :
> Log 日志
> D/MainActivity: onPause
> D/MainActivity: onSaveInstanceState
> D/MainActivity: onStop
> D/MainActivity: onDestroy
> D/MainActivity: onCreate
> D/MainActivity: onStart
> D/MainActivity: [onRestoreInstanceState]: test
> D/MainActivity: onResume


摘自 [Android 开发者艺术探索](https://book.douban.com/subject/26599538/) 一书：

> 关于保存和恢复 View 的层次结构，系统工作流程是： Activity 异常终止, Activity 调用 onSaveInstanceState 去保存数据，然后  Activity 会委托 Windows 去保存数据，接着 Window 再委托它上面的顶层容器去保存数据。顶层容器是一个 ViewGroup ，一般来说它很可能是 DectorView ，最后顶层容器再去通知它的子元素保存数据。（这是一种委托思想，上层委托下层，父容器委托子元素去处理事情，如 View 的绘制过程，事件分发都是采用类似的思想）

### Fragment

#### 普通的 Fragment

![22:42:29.jpg](http://ww3.sinaimg.cn/large/006tKfTcgw1f6i3gmvtroj308t0njab8.jpg)

![22:43:01.jpg](http://ww3.sinaimg.cn/large/006tKfTcgw1f6i3h6vqznj309g0irq3t.jpg)


从图可以看出，Fragment 生命周期大部分的状态与 Activity 相似，特殊的是

- `onAttach()` —— 当 Fragment 被加入到 Activity 时调用(在这个方法中可以获得所在的 Activity ).
- `onCreateView()` —— 当 Activity 要得到 Fragment 的 Layout 时，调用此方法，Fragment 在其中创建自己的 Layout (界面)。
- `onActivityCreated()` —— 当 Activity 的 onCreated() 方法返回后调用此方法
- `onDestroyView()` —— 当 Fragment 中的视图被移除的时候，调用这个方法。
- `onDetach()` —— 当 Fragment 和 Activity 分离的时候，调用这个方法。

#### ViewPager 中的 Fragment

我们开发中经常会用到 ViewPager + Fragment 组合的形式来完成特定的需求。本身 Fragment 生命周期就比 Activity 要复杂很多，当它在 ViewPager 中又是怎么回调呢？

我先给 ViewPager 加入三个 Fragment:

```java
viewPager = (ViewPager) findViewById(R.id.viewpager);
fragmentList.add(new OneTextFragment());
fragmentList.add(new TwoTextFragment());
fragmentList.add(new ThreeTextFragment());
viewPager.setAdapter(new FtAdapter(getSupportFragmentManager(), fragmentList));
```

启动这个 Activity 的日志如下：

```Log
D/ViewPagerHostActivity: onCreate
D/ViewPagerHostActivity: onStart
D/ViewPagerHostActivity: onResume
D/OneTextFragment: onAttach
D/OneTextFragment: onCreate
D/TwoTextFragment: onAttach
D/TwoTextFragment: onCreate
D/TwoTextFragment: onActivityCreated
D/OneTextFragment: onActivityCreated
D/OneTextFragment: onStart
D/OneTextFragment: onResume
D/TwoTextFragment: onStart
D/TwoTextFragment: onResume
```

我们发现启动后，有两个 Fragment(one,two) 被创建，为什么会创建两个？这个问题我们留着后面说。

当 Activity 进入后台：

```Log
D/ViewPagerHostActivity: onPause
D/ViewPagerHostActivity: onSaveInstanceState
D/TwoTextFragment: onStop
D/OneTextFragment: onStop
D/ViewPagerHostActivity: onStop
```

当 Activity 返回前台：
```Log
D/ViewPagerHostActivity: onRestart
D/TwoTextFragment: onStart
D/OneTextFragment: onStart
D/ViewPagerHostActivity: onStart
D/ViewPagerHostActivity: onResume
D/TwoTextFragment: onResume
D/OneTextFragment: onResume
```

当 Activity 销毁：
```Log
D/ViewPagerHostActivity: onPause
D/TwoTextFragment: onStop
D/OneTextFragment: onStop
D/ViewPagerHostActivity: onStop
D/TwoTextFragment: onDestroyView
D/TwoTextFragment: onDestroy
D/TwoTextFragment: onDetach
D/OneTextFragment: onDestroyView
D/OneTextFragment: onDestroy
D/OneTextFragment: onDetach
D/ViewPagerHostActivity: onDestroy
```

滑动一页：
```Log
D/ThreeTextFragment: onAttach
D/ThreeTextFragment: onCreate
D/ThreeTextFragment: onActivityCreated
D/ThreeTextFragment: onStart
D/ThreeTextFragment: onResume
```
当前显示的页面是 TwoTextFragment 然后 ThreeTextFragment 也已经创建好了。

再滑动一页：
```Log
D/OneTextFragment: onStop
D/OneTextFragment: onDestroyView
```
当前显示的页面是 ThreeTextFragment ，我们发现 OneTextFragment 已经销毁。

我们可以得到默认状态下的 ViewPager 会缓存 1 个 Fragment，相当于有两个 Fragment 是创建状态。
当我们增加一行代码：
```java
viewPager.setOffscreenPageLimit(2);
```
当 Activity 创建时的生命周期：
```Log
D/ViewPagerHostActivity: onCreate
D/ViewPagerHostActivity: onStart
D/ViewPagerHostActivity: onResume
D/OneTextFragment: onAttach
D/OneTextFragment: onCreate
D/TwoTextFragment: onAttach
D/TwoTextFragment: onCreate
D/ThreeTextFragment: onAttach
D/ThreeTextFragment: onCreate
D/TwoTextFragment: onActivityCreated
D/OneTextFragment: onActivityCreated
D/OneTextFragment: onStart
D/OneTextFragment: onResume
D/TwoTextFragment: onStart
D/TwoTextFragment: onResume
D/ThreeTextFragment: onStart
D/ThreeTextFragment: onResume
```
三个 Fragment 都创建好了，并且左右切换不会走任何生命周期（虽然是废话）。

setOffscreenPageLimit 源码：

```java
public void setOffscreenPageLimit(int limit) {
	if (limit < DEFAULT_OFFSCREEN_PAGES) {
		Log.w(TAG, "Requested offscreen page limit " + limit + " too small; defaulting to " +
				DEFAULT_OFFSCREEN_PAGES);
		limit = DEFAULT_OFFSCREEN_PAGES;
	}
	if (limit != mOffscreenPageLimit) {
		mOffscreenPageLimit = limit;
		populate();
	}
}
```

通过源码我们可以知道，ViewPager 的缓存的默认值和最小值是 __1__。

### 启动模式
#### Activity 的四种启动模式

- Standard：标准模式，一调用 startActivity() 方法就会产生一个新的实例。

- SingleTop: 来了 intent, 每次都创建新的实例，仅一个例外：当栈顶的activity 恰恰就是该activity的实例（即需要创建的实例)时，不再创建新实例。这解决了栈顶复用问题

- SingleTask: 来了 intent 后，检查栈中是否存在该 activity的实例，如果存在就把 intent 发送给它，否则就创建一个新的该activity的实例，放入一个新的 task 栈的栈底。肯定位于一个task的栈底，而且栈中只能有它一个该 activity 实例，但允许其他 activity 加入该栈。解决了在一个 task 中共享一个 activity。

- SingleInstance: 这个跟 SingleTask 基本上是一样，只有一个区别：在这个模式下的Activity实例所处的task中，只能有这个activity实例，不能有其他的实例。一旦该模式的activity的实例已经存在于某个栈中，任何应用在激活该activity时都会重用该栈中的实例，解决了多个task共享一个 activity。

这些启动模式可以在功能清单文件 AndroidManifest.xml 中进行设置，中的 launchMode 属性。

#### 具体实践

- __SingleTop__ 栈顶复用模式

```xml
<activity android:name=".dLaunchChapter.OneActivity"
	android:launchMode="singleTop"/>
```
我们在清单里先给 OneActivity 启动模式设置为 singleTop ，然后代码启动活动的顺序为 `One --> One`，反复点击多次，然后我们看看栈内情况。

adb 命令 ：`dumpsys activity | grep -i run`
```adb
root@vbox86p:/ # dumpsys activity | grep -i run
    Running activities (most recent first):
        Run #1: ActivityRecord{23e3b5b u0 com.hugo.demo.activitydemo/.dLaunchChapter.OneActivity t595}
        Run #0: ActivityRecord{1a2c6f3 u0 com.hugo.demo.activitydemo/.LaunchActivity t595}
```
该启动模式下并且 OneActivity 在栈顶所以不会创建新的实例，其生命周期调用 `onPause —-> onResume`



- __SingleTask__ 栈内复用模式

修改 OneActivity 的启动模式为 SingleTask ，然后我们代码启动的顺序为 `One —-> Two —-> One`，接了下看看栈内情况：

`One —-> Two` 我们记录下当前的 Activity 栈：

```adb
Running activities (most recent first):
        Run #2: ActivityRecord{1e8701b7 u0 com.hugo.demo.activitydemo/.dLaunchChapter.TwoActivity t632}
        Run #1: ActivityRecord{39e11719 u0 com.hugo.demo.activitydemo/.dLaunchChapter.OneActivity t632}
```

接下来我们执行 `Two —-> One` ，当前 Activity 栈信息：

```adb
Running activities (most recent first):
	   Run #1: ActivityRecord{39e11719 u0 com.hugo.demo.activitydemo/.dLaunchChapter.OneActivity t632}
```

当 TwoActivity 启动 OneActivity（SingleTask） 的时候，堆栈信息里只剩下了 OneActivity 并且和第一次内存信息 __39e11719__ 相同，所以确实是复用了没有新建实例，接下来我们看看 Log 日志，再验证下我们的猜想，看看具体走了哪些生命周期：

```Log
D/TwoActivity: onPause
D/OneActivity: onNewIntent
D/OneActivity: onRestart
D/OneActivity: onStart
D/OneActivity: onResume
D/TwoActivity: onStop
D/TwoActivity: onDestroy
```

果然此时 OneActivity 没有重新创建，并且系统把它切换到了栈顶并调用 onNewIntent 方法，同时我们发现， SingleTask 默认具有 clearTop 效果，导致 TwoActivity 出栈。

<div class="tip">
我们代码指定 OneActivity 的栈，效果还是一样的吗？
</div>

带着问题我们修改下代码，增加一行：
```xml
<activity
	android:name=".dLaunchChapter.OneActivity"
	android:launchMode="singleTask"
	android:taskAffinity="com.hugo.demo.singleTask"/>
```

然后我们按照刚刚顺序 `One —-> Two —-> One` 启动 Activity ，现在的栈内信息还会更上次一样吗？
```adb
Running activities (most recent first):
        Run #2: ActivityRecord{1bc18519 u0 com.hugo.demo.activitydemo/.dLaunchChapter.OneActivity t636}
        Run #1: ActivityRecord{36e5e368 u0 com.hugo.demo.activitydemo/.dLaunchChapter.TwoActivity t635}
```

我们发现，虽然是复用了 OneActivity 而且移到了栈顶，但是并没有销毁 TwoActivity 。

原因在于 singleTask 模式受 [taskAffinity](https://developer.android.com/guide/topics/manifest/activity-element.html?hl=zh-cn#aff) 影响，TwoActivity 和 OneActivity 所在的 Activity 栈不同。

总结，启动一个 lauchMode 为 singleTas k的 Activity 时有两种情况:

> 1. 若系统中存在相同 taskAffinity 值的任务栈 (tacks1 )时,会把 task1 从后台调到前台，若实例存在则干掉其上面的所有 Activity 并调用 onNewInstance 方法重用，没有该实例则新建一个。
> 2. 否则，新建一个任务栈，并以此 Activity 作为 root 。



- SingleInstance 单实例模式

这是一种加强的 singleTask 模式，它除了具有 singleTask 模式的所有特性以外，还加强了一点，就是具有此模式的 Activity 只能单独地位于任务栈。




<div class="tip">
好了，关于生命周期和启动模式实践+知识点整理已经完成啦，
非常推荐大家下载源码自己运行看看 Log 日志，查看源码：[Github](https://github.com/xcc3641/ActivityLifeDemo)
，这样可以对这篇文章知识更加深刻。
</div>

### 参考文档

- Android 开发艺术探索 第一章
- [管理Activity生命周期](https://developer.android.com/training/basics/activity-lifecycle/index.html)
- [Print the current back stack in the log](http://stackoverflow.com/questions/11549366/print-the-current-back-stack-in-the-log/11549400#11549400)
