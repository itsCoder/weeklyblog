title: 沉浸式适配个人总结
date: 2016-11-08 16:02:31
---

### 目录

- [目录](#目录)
- [前言](#前言)
- [适配](#适配)
  - [标志](#标志)
  - [布局](#布局)
  - [修补](#修补)
  - [坑](#坑)
- [总结](#总结)
- [参考](#参考)


### 前言

> 本篇文章环境是
> - 主色调：**白色**
> - 右滑返回：**需要**


在做沉浸式之前，得知道下面几个问题：
1. 什么是沉浸式
2. Android 系统对沉浸式的支持

首先第一个问题，推荐阅读这篇 [Android 沉浸式 UI 实现及原理](http://www.jianshu.com/p/f3683e27fd94) ，作者对照哔哩哔哩进行分析「个人也认为 B 站在国内 Android 应用里算规范」。

第二个问题，也是个人在适配沉浸式过程中遇到的版本坑，大致总结如下：

- Android 4.4（19）以下
- Android 4.4（19）
- 「小米 MIUI V4 」和「魅族 Flyme 4.0」以上
- Android 5.0 （21）
- Android 6.0+（23+）

因为 Android 是从 4.4 开始引入 `android:windowTranslucentStatus` 标签，所以理论上 4.4 以上都可以实现沉浸式，而本方案也是基于 4.4 开始。

因为**即刻**的主色调是__纯白色__，在 Android 里纯白是 BUG 的存在，适配需要做更多对于__状态栏 icon 颜色__的处理。
收集资料可以参考：
- [Android 状态栏黑色字体](http://www.jianshu.com/p/2756d41e9697)
- [Android 系统更改状态栏字体颜色](http://blog.isming.me/2016/01/09/chang-android-statusbar-text-color/)
- [白底黑字！Android 浅色状态栏黑色字体模式](http://www.jianshu.com/p/7f5a9969be53)
- [Flyme 系统 API-沉浸式状态栏](http://open-wiki.flyme.cn/index.php?title=Flyme%E7%B3%BB%E7%BB%9FAPI)

按照刚才列出的版本，整理注意的细节如图：

![](http://ww3.sinaimg.cn/large/006y8lVajw1f9jzc1kxcwj31kw0e5n01.jpg)

关于小米 OS 和魅族 OS 可以参考维基百科：

- [Flyme OS \- 维基百科，自由的百科全书](https://zh.wikipedia.org/wiki/Flyme_OS)
- [MIUI \- 维基百科，自由的百科全书](https://zh.wikipedia.org/wiki/MIUI)


### 适配

#### 标志

适配中用到的 Flag 有：

```java
View.SYSTEM_UI_FLAG_LAYOUT_STABLE
View.SYSTEM_UI_FLAG_LAYOUT_FULLSCREEN
View.SYSTEM_UI_FLAG_LAYOUT_HIDE_NAVIGATION
```

在 4.4(API 19) 中还引入了 `WindowManager.LayoutParams.FLAG_TRANSUCENT_STATUS` 和`WindowManager.LayoutParams.FLAG_TRANSLUCENT_NAVIGATION` 用于控制 `System UI` 变透明，这两个 Flag 分别对应于 `windowTranslucentStatus` 和`windowTranslucentNavigation` 两个 attr，并同时提供了相应的 Theme（这些 Theme 都没有 ActionBar），当使用这两个 Flag 时，`SYSTEM_UI_FLAG_LAYOUT_STABLE、SYSTEM_UI_FLAG_LAYOUT_FULLSCREEN和SYSTEM_UI_FLAG_LAYOUT_HIDE_NAVIGATION` 会被自动添加。

#### 布局

从 4.4 开始，所以在 style-19 里所有父级主题增加：
```xml
<style name="JikeTheme.SystemUi">
	<item name="android:windowTranslucentStatus">true</item>
</style>
```

在业务基类 **BaseActivity**  `onCreate()` 中加入了对沉浸式的一些判断和处理：

```java
@Override
protected void onCreate(Bundle savedInstanceState) {
	super.onCreate(savedInstanceState);
	StatusBarUtil.setImmersiveStatusBar(this);
	// 适配状态栏字体颜色
	if (enableStatusIconsBlack()) {
		isOtherPhone = StatusBarUtil.setStatusBarDarkIcon(this);
	}
	if (isOtherPhone && StatusBarUtil.supportImmersiveStatusBar()) {
		if (enableAddStatusView()) {
			StatusBarUtil.addColorStatusView(this, R.color.very_dark_grayish_blue_26);
		} else {
			StatusBarUtil.addTranslucentView(this, 0);
		}
	}
}
```
这里有四个方法：

- StatusBarUtil.setImmersiveStatusBar(this);
- StatusBarUtil.setStatusBarDarkIcon(this);
- StatusBarUtil.addColorStatusView(this, R.color.very_dark_grayish_blue_26);
- StatusBarUtil.addTranslucentView(this, 0);

主要是这里的逻辑需要说明

 `if (isOtherPhone && StatusBarUtil.supportImmersiveStatusBar())`

前面已经说了 5.x 的非小米魅族手机无法更改状态栏 icon 颜色，所以我进行适配的方法是，isOtherPhone 来判断是不是非（魅族，小米） 的其他手机，然后加入一个与状态栏等高的带颜色的矩形 View ，也就是`StatusBarUtil.addColorStatusView(this, R.color.very_dark_grayish_blue_26);` 方法做的事情，同时通过 `enableAddStatusView() `标志是否需要添加。

这样说可能不好理解， 一个具体的场景是这样：

![](http://ww3.sinaimg.cn/large/006y8lVajw1f9k09xq8k9j30vf0sek07.jpg)



可以看到，在 5.x 状态栏是一个黑条，也就是我们手动加上去的矩形，这也就是 `enableAddStatusView` 标识的作用。



接下来适配 Toolbar ：

```java
protected void initToolbar(@NonNull Toolbar toolbar) {
  // other code
  StatusBarUtil.setImmersiveStatusBarToolbar(toolbar, this);
}
```

```java
    /**
     * 统一适配 toolbar
     * 设置 toolbar 高度为 ?attr/actionBarSize + statusBarHeight 并且设置 padding 布局还原
     */
    public static void setImmersiveStatusBarToolbar(Toolbar toolbar, Context context) {
        if (supportImmersiveStatusBar()) {
            ViewGroup.MarginLayoutParams toolLayoutParams = (ViewGroup.MarginLayoutParams) toolbar.getLayoutParams();
            toolLayoutParams.height = EnvUtil.getStatusBarHeight() + EnvUtil.getActionBarSize(context);
            toolbar.setLayoutParams(toolLayoutParams);
            setImmersiveStatusToolbarOnlyPadding(toolbar, 0, EnvUtil.getStatusBarHeight(), 0, 0);
        }
    }
```

主要做的事情就是，将 Toolbar 的高度增加一个状态栏高度，且设置 paddingTop。

这样在 BaseActivity 里就把状态栏和 Toolbar 都适配好了。

接下来就是给内容详情增加正确的 margin 值。

其实以上大部分都跟具体的界面编写有关，所以这里只提供一个思路。

#### 修补

适配之后整个页面看着是可行的，但是得额外注意一些具体场景：

1. 需要输入法

2. 与状态栏有关的遮挡交互动画

**第一个** 比较特殊性。为了适配沉浸式，我们使用了`View.SYSTEM_UI_FLAG_LAYOUT_FULLSCREEN` 标签，该标签下，adjustResize 会失效，adjustPan 效果很差。为了实现输入法弹出效果，写了一个 FullscreenInputModeUtil 工具类兼容实现，主要逻辑是动态计算可见高度来得到输入法高度，从而改变布局整个高度（输入栏框是始终在布局最底部）实现 adjustResize 效果。

具体可以参考  [code.google.com](https://code.google.com/p/android/issues/detail?id=5497)

**第二个** 发生在 layout 是 CoordinatorLayout Behavior 实现 Toolbar 上滑隐藏效果的 Bug ，所以现有的解决方案是在 layout 里写一个背景是白色的 View 并在代码里根据 sdk 修改高度：

```java
// 增加一个实体白色 View 在状态栏下，复原滑动效果
if (StatusBarUtil.supportImmersiveStatusBar()) {
  mTrickStatusBar.getLayoutParams().height = EnvUtil.getStatusBarHeight();
  mTrickStatusBar.requestLayout();
}
```

#### 坑

- ROM 坑

根据手机厂商判断 Rom 是不行的，因为会有小米手机但是刷了其他 Rom 的情况，所以我们得找到正确判断特殊 Rom 的方法。

```java
    private static String getRomProperty(String prop) {
        String line = "";
        BufferedReader reader = null;
        Process process = null;
        try {
            process = Runtime.getRuntime().exec("getprop " + prop);
            reader = new BufferedReader(new InputStreamReader(process.getInputStream()), 1024);
            line = reader.readLine();
        } catch (IOException e) {
            e.printStackTrace();
        } finally {
            IOUtil.close(reader);
            if (process != null) {
                process.destroy();
            }
        }
        return line;
    }
```
该方法主要是执行命令行，去获取 **build.prop** 文件的信息，里面记录了系统的设置和属性，相当于 Windows 系统注册表的文件。当然包括了该手机使用的 Rom 信息，上层我们只需要判断是否含有特殊适配的 Rom 字段即可。这样就可以准确适配不同手机不同系统。

- 虚拟导航栏是否存在

这里大部分的场景是需要输入栏的页面——因为大多输入栏的布局都会是`layout_alignParentBottom=true`。因为即刻中采取的沉浸式方案是全屏，有输入栏的页面，会设置一个虚拟导航栏的 padding 值，这样才能保证输入栏不会被虚拟导航栏遮挡。

**所以这里的关键就是判断该手机是否有虚拟导航栏。**

最开始的时候，我们是通过判断是否有硬件按钮（菜单键，返回键），来直接一刀决定是否有虚拟导航栏。但是在 Android 机型复杂的环境下，该方法并不能保证所有适配。所以换了一种方案：

用 Display 来帮助，该类中有个方法：

>Gets display metrics based on the real size of this display.
>The size is adjusted based on the current rotation of the display.
>The real size may be smaller than the physical size of the screen when the window manager is emulating a smaller display (using adb shell am display-size).


```java
public void More ...getRealMetrics(DisplayMetrics outMetrics) {
	synchronized (this) {
		updateDisplayInfoLocked();
		mDisplayInfo.getLogicalMetrics(outMetrics,
		CompatibilityInfo.DEFAULT_COMPATIBILITY_INFO,
		mDisplayAdjustments.getActivityToken());
	}
}
```

从上诉方法我们可以得到一个近似于手机物理屏幕的尺寸，这里我们认为是 realSize .

>Gets display metrics that describe the size and density of this display.
>The size is adjusted based on the current rotation of the display.
>The size returned by this method does not necessarily represent the actual raw size (native resolution) of the display. The returned size may be adjusted to exclude certain system decor elements that are always visible. It may also be scaled to provide compatibility with older applications that were originally designed for smaller displays.

```java
public void More ...getMetrics(DisplayMetrics outMetrics) {
	synchronized (this) {
		updateDisplayInfoLocked();
		mDisplayInfo.getAppMetrics(outMetrics, mDisplayAdjustments);
	}
}
```
同时通过该方法获取可见视图的一个尺寸。接下来的事情，就是分别比较长宽大小来判断是否有导航栏。

### 总结

本文主要是提供一个具体可行方案和开发中遇到的一些坑的解决方案提供出来。如果你的应用主色调不是纯白色，那么理论上可以完全适配到 4.4 (19) 。

对于沉浸式，众说纷纭，但是唯一的目的，就是提高用户体验。

如果还有什么遗漏的地方，欢迎补充。


### 参考

- [Android App 沉浸式状态栏解决方案](http://jaeger.itscoder.com/android/2016/02/15/status-bar-demo.html)
- [与 Status Bar 和 Navigation Bar 相关的一些东西](http://angeldevil.me/2014/09/02/About-Status-Bar-and-Navigation-Bar/)
- [Android How to adjust layout in Full Screen Mode when softkeyboard is visible](http://stackoverflow.com/questions/7417123/android-how-to-adjust-layout-in-full-screen-mode-when-softkeyboard-is-visible)
