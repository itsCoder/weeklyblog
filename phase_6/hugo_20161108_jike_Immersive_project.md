title: 沉浸式适配个人总结
date: 2016-11-08 16:02:31
---
>- 文章来源：itsCoder 的 [WeeklyBolg](https://github.com/itsCoder/weeklyblog) 项目
>- itsCoder主页：[http://itscoder.com/](http://itscoder.com/)
>- 作者：[谢三弟](https://github.com/xcc3641)
>- 审阅者：[Jaeger](https://github.com/laobie)


### 目录

- [目录](#目录)
- [前言](#前言)
- [适配](#适配)
  - [标志](#标志)
  - [布局](#布局)
  - [修补](#修补)
  - [坑](#坑)
- [源码](#源码)
  - [状态栏工具部分核心代码](#状态栏工具部分核心代码)
  - [环境工具类部分核心代码](#环境工具类部分核心代码)
  - [修复全屏输入框工具类](#修复全屏输入框工具类)
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
isSuccessStatusIcons = true; // 默认认为可以设置状态栏 Icon 颜色
@Override
protected void onCreate(Bundle savedInstanceState) {
  super.onCreate(savedInstanceState);
  StatusBarUtil.setImmersiveStatusBar(this);
  // 适配状态栏字体颜色
  if (needStatusIconsBlack()) {
    isSuccessStatusIcons = StatusBarUtil.setStatusBarDarkIcon(this);
  }
  if (!isSuccessStatusIcons && ImmersiveUtil.supportImmersiveStatusBar()) {
    if (needAddColorStatusView()) {
      StatusBarUtil.addColorStatusView(this, R.color.black);
    } else {
      StatusBarUtil.addTranslucentView(this, 0);
    }
  }
}
```
这里有四个方法：

- StatusBarUtil.setImmersiveStatusBar(this);  // 添加全屏/透明状态栏的 Flag
- StatusBarUtil.setStatusBarDarkIcon(this); // 设置状态栏 Icon 为黑色
- StatusBarUtil.addColorStatusView(this, R.color.very_dark_grayish_blue_26); // 添加一个带颜色的矩形块
- StatusBarUtil.addTranslucentView(this, 0); // 添加一个透明状态栏

主要是这里的逻辑需要说明

 > `if (!isSuccessStatusIcons && StatusBarUtil.supportImmersiveStatusBar())`

前面已经说了 5.x 的非小米魅族 Rom 无法更改状态栏 icon 颜色，所以我进行适配的方法是，isSuccessStatusIcons 来标识是不是非（魅族，小米） 的其他 Rom，然后加入一个与状态栏等高的带颜色的矩形 View ，也就是`StatusBarUtil.addColorStatusView(this, R.color.very_dark_grayish_blue_26);` 方法做的事情，同时通过 `needAddColorStatusView() `标志是否需要添加。

这样说可能不好理解， 一个具体的场景是这样：

![](http://ww3.sinaimg.cn/large/006y8lVajw1f9k09xq8k9j30vf0sek07.jpg)



可以看到，在 5.x 首页状态栏是一个黑条，也就是我们手动加上去的矩形，但在有图片的 activity 是透明，这也就是 `needAddColorStatusView` 标识的作用。



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
    if (ImmersiveUtil.supportImmersiveStatusBar()) {
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

**第二个** 是这样的：

![](http://ww3.sinaimg.cn/large/006tNc79gw1f9xpphob6wg30di0o11kx.gif)

可以看到 BUG 是原本上滑应该隐藏在状态栏后的 Title 并没有消失，原本是 CoordinatorLayout Behavior 实现 Toolbar 上滑隐藏效果。
所以现有的解决方案是在 layout 里写一个背景是白色的 View 并在代码里根据 sdk 修改高度：

```java
// 增加一个实体白色 View 在状态栏下，复原滑动效果
if (ImmersiveUtil.supportImmersiveStatusBar()) {
  mTrickStatusBar.getLayoutParams().height = EnvUtil.getStatusBarHeight();
  mTrickStatusBar.requestLayout();
}
```

#### 坑

- ROM 坑

根据手机厂商判断 Rom 是不行的，因为会有小米手机但是刷了其他 Rom 的情况，所以我们得找到正确判断 Rom 的方法。

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
同时通过该方法获取可见视图的尺寸。接下来的事情，就是分别比较长宽大小来判断是否有导航栏。


### 源码

#### 状态栏工具部分核心代码
```java
public class StatusBarUtil {

    /**
     * 透明状态栏 让布局延伸到状态栏
     */
    public static void setImmersiveStatusBar(@NonNull Activity activity) {
        if (ImmersiveUtil.supportImmersiveStatusBar()) {
            if (SdkUtil.sdkVersionGe21()) {
                activity.getWindow().setStatusBarColor(Color.TRANSPARENT);
            }
            if (SdkUtil.sdkVersionEq(19)) {
                activity.getWindow().setFlags(WindowManager.LayoutParams.FLAG_TRANSLUCENT_STATUS,
                        WindowManager.LayoutParams.FLAG_TRANSLUCENT_STATUS);
            }
            activity.getWindow()
                    .getDecorView()
                    .setSystemUiVisibility(View.SYSTEM_UI_FLAG_LAYOUT_STABLE | View.SYSTEM_UI_FLAG_LAYOUT_FULLSCREEN);
        }
    }

    /**
     * 统一适配 toolbar
     * 设置 toolbar 高度为 ?attr/actionBarSize + statusBarHeight 并且设置 padding 布局还原
     */
    public static void setImmersiveStatusBarToolbar(@NonNull Toolbar toolbar, Context context) {
        if (ImmersiveUtil.supportImmersiveStatusBar()) {
            ViewGroup.MarginLayoutParams toolLayoutParams = (ViewGroup.MarginLayoutParams) toolbar.getLayoutParams();
            toolLayoutParams.height = EnvUtil.getStatusBarHeight() + EnvUtil.getActionBarSize(context);
            toolbar.setLayoutParams(toolLayoutParams);
            setImmersiveStatusToolbarOnlyPadding(toolbar, 0, EnvUtil.getStatusBarHeight(), 0, 0);
        }
    }

    /**
     * 为沉浸式抽离设置 toolbar padding 值的方法
     */
    public static void setImmersiveStatusToolbarOnlyPadding(@NonNull Toolbar toolbar, int left, int top, int right, int bottom) {
        if (SdkUtil.sdkVersionGe21()) {
            toolbar.setPadding(left, top, right, bottom);
        } else if (SdkUtil.sdkVersionGe19()) {
            toolbar.setPadding(left, top - DensityUtil.dimenPixelSize(R.dimen.shadow_size), right, bottom);
        } else {
            toolbar.setPadding(left, 0, right, bottom);
        }
        toolbar.requestLayout();
    }

    /**
     * 给内容页设置正确的 margin 值
     *
     * @param includeActionBar true： Height = StatusBar + ActionBar false： Height = StatusBar
     */

    public static void setImmersiveStatusBarContent(@NonNull View view, boolean includeActionBar) {

        ViewGroup.MarginLayoutParams params = (ViewGroup.MarginLayoutParams) view.getLayoutParams();
        if (ImmersiveUtil.supportImmersiveStatusBar()) {
            if (includeActionBar) {
                params.topMargin = EnvUtil.getStatusBarHeight() + EnvUtil.getActionBarSize(view.getContext());
            } else {
                params.topMargin = EnvUtil.getStatusBarHeight();
            }
        } else {
            if (includeActionBar) {
                params.topMargin = EnvUtil.getActionBarSize(view.getContext()) + DensityUtil.dimenPixelSize(R.dimen.shadow_size) * 2;
            }
        }
        view.setLayoutParams(params);
        view.requestLayout();

    }

    /**
     * 给需要弹出输入法的页面设置弹出效果和正确的 padding 值
     */
    public static void setImmersiveNeedInputView(@NonNull Activity activity, @NonNull ViewGroup viewGroup) {
        if (ImmersiveUtil.supportImmersiveStatusBar()) {
            FullscreenInputModeUtil.attachActivity(activity, viewGroup);
            viewGroup.setPadding(0, 0, 0, EnvUtil.getNavigationBarHeight());
            viewGroup.requestLayout();
        }
    }

    /**
     * 添加颜色矩形条
     */
    public static void addColorStatusView(@NonNull Activity activity, @ColorRes int color) {
        ViewGroup contentView =
                (ViewGroup) activity.findViewById(Window.ID_ANDROID_CONTENT);
        if (contentView.getChildCount() > 1) {
            contentView.getChildAt(1).setBackgroundColor(ContextCompat.getColor(activity, color));
        } else {
            contentView.addView(createColorStatusBarView(activity, color));
        }
    }

    /**
     * 添加半透明矩形条
     */
    public static void addTranslucentView(@NonNull Activity activity, int statusBarAlpha) {
        ViewGroup contentView = (ViewGroup) activity.findViewById(Window.ID_ANDROID_CONTENT);
        if (contentView.getChildCount() > 1) {
            contentView.getChildAt(1).setBackgroundColor(Color.argb(statusBarAlpha, 0, 0, 0));
        } else {
            contentView.addView(createTranslucentStatusBarView(activity, statusBarAlpha));
        }
    }

    /**
     * 创建半透明矩形 View
     */
    private static View createTranslucentStatusBarView(@NonNull Activity activity, int alpha) {
        // 绘制一个和状态栏一样高的矩形
        View statusBarView = new View(activity);
        LinearLayout.LayoutParams params =
                new LinearLayout.LayoutParams(ViewGroup.LayoutParams.MATCH_PARENT, EnvUtil.getStatusBarHeight());
        statusBarView.setLayoutParams(params);
        statusBarView.setBackgroundColor(Color.argb(alpha, 0, 0, 0));
        return statusBarView;
    }

    /**
     * 创建一个带颜色矩形 View
     */
    private static View createColorStatusBarView(@NonNull Activity activity, @ColorRes int color) {
        // 绘制一个和状态栏一样高的矩形
        View statusBarView = new View(activity);
        LinearLayout.LayoutParams params =
                new LinearLayout.LayoutParams(ViewGroup.LayoutParams.MATCH_PARENT, EnvUtil.getStatusBarHeight());
        statusBarView.setLayoutParams(params);
        statusBarView.setBackgroundColor(ContextCompat.getColor(activity, color));
        return statusBarView;
    }

    /**
     * @return 是否设置颜色成功
     */
    public static boolean setStatusBarDarkIcon(@NonNull Activity activity) {
        if (!ImmersiveUtil.supportImmersiveStatusBar()) {
            return false;
        } else {
            if (EnvUtil.isMeizu()) {
                StatusBarUtil.setMeizuStatusBarDarkIcon(activity, true);
                return true;
            } else if (EnvUtil.isXiaomi()) {
                StatusBarUtil.setMiuiStatusBarDarkIcon(activity, true);
                return true;
            } else if (EnvUtil.isZuk()) {
                // Zuk 的 rom 无法修改状态栏 icon 颜色
                return false;
            } else if (SdkUtil.sdkVersionGe23()) {
                activity.getWindow().getDecorView().setSystemUiVisibility(View.SYSTEM_UI_FLAG_LAYOUT_STABLE | View.SYSTEM_UI_FLAG_LAYOUT_FULLSCREEN | View.SYSTEM_UI_FLAG_LIGHT_STATUS_BAR);
                return true;
            } else {
                return false;
            }
        }
    }

    /**
     * 修改魅族状态栏字体颜色 Flyme 4.0
     */
    private static void setMeizuStatusBarDarkIcon(@NonNull Activity activity, boolean dark) {
        if (ImmersiveUtil.supportImmersiveStatusBar()) {
            try {
                WindowManager.LayoutParams lp = activity.getWindow().getAttributes();
                Field darkFlag = WindowManager.LayoutParams.class
                        .getDeclaredField("MEIZU_FLAG_DARK_STATUS_BAR_ICON");
                Field meizuFlags = WindowManager.LayoutParams.class
                        .getDeclaredField("meizuFlags");
                darkFlag.setAccessible(true);
                meizuFlags.setAccessible(true);
                int bit = darkFlag.getInt(null);
                int value = meizuFlags.getInt(lp);
                if (dark) {
                    value |= bit;
                } else {
                    value &= ~bit;
                }
                meizuFlags.setInt(lp, value);
                activity.getWindow().setAttributes(lp);
            } catch (Exception e) {
                JLog.e(e, e.toString());
            }
        }
    }

    /**
     * 修改 MIUI V6  以上状态栏颜色
     */
    private static void setMiuiStatusBarDarkIcon(@NonNull Activity activity, boolean dark) {
        if (ImmersiveUtil.supportImmersiveStatusBar()) {
            Class<? extends Window> clazz = activity.getWindow().getClass();
            try {
                Class<?> layoutParams = Class.forName("android.view.MiuiWindowManager$LayoutParams");
                Field field = layoutParams.getField("EXTRA_FLAG_STATUS_BAR_DARK_MODE");
                int darkModeFlag = field.getInt(layoutParams);
                Method extraFlagField = clazz.getMethod("setExtraFlags", int.class, int.class);
                extraFlagField.invoke(activity.getWindow(), dark ? darkModeFlag : 0, darkModeFlag);
            } catch (Exception e) {
                JLog.e(e, e.toString());
            }
        }
    }
}
```

#### 环境工具类部分核心代码
```java
public class EnvUtil {
    public static void checkDeviceHasNavigationBar(Activity activity) {

        if (sHasNavigationBar != null) {
            return;
        }

        WindowManager windowManager = activity.getWindowManager();
        Display display = windowManager.getDefaultDisplay();
        DisplayMetrics realDisplayMetrics = new DisplayMetrics();

        if (SdkUtil.sdkVersionGe(17)) {
            display.getRealMetrics(realDisplayMetrics);
        }

        int realHeight = realDisplayMetrics.heightPixels;
        int realWidth = realDisplayMetrics.widthPixels;

        DisplayMetrics displayMetrics = new DisplayMetrics();
        display.getMetrics(displayMetrics);

        int displayHeight = displayMetrics.heightPixels;
        int displayWidth = displayMetrics.widthPixels;

        sHasNavigationBar = (realWidth - displayWidth) > 0 || (realHeight - displayHeight) > 0;
    }

    public static boolean isXiaomi() {
        return MIUI.equalsIgnoreCase(getRomInfo());
    }

    public static boolean isMeizu() {
        return FLYME.equalsIgnoreCase(getRomInfo());
    }

    public static boolean isZuk() {
        return ZUK.equalsIgnoreCase(getRomInfo());
    }

    private static String romInfo = "";

    private static final String MIUI = "miui";
    private static final String FLYME = "flyme";
    private static final String ZUK = "zuk";
    private static final String UNKNOWN = "unknown";
    private static final String RUNTIME_MIUI = "ro.miui.ui.version.name";
    private static final String RUNTIME_DISPLAY = "ro.build.display.id";
    private static final String RUNTIME_ZUK = "ro.com.zui.version";

    public static String getRomInfo() {

        if (!TextUtils.isEmpty(romInfo)) {
            return romInfo;
        }

        if (!TextUtils.isEmpty(getRomProperty(RUNTIME_MIUI))) {
            romInfo = MIUI;
        } else if (!TextUtils.isEmpty(getRomProperty(RUNTIME_ZUK))) {
            romInfo = ZUK;
        } else if (getRomProperty(RUNTIME_DISPLAY).toLowerCase().contains(FLYME)) {
            romInfo = FLYME;
        } else {
            romInfo = UNKNOWN;
        }
        return romInfo;
    }

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
}
```

#### 修复全屏输入框工具类

```java
public class FullscreenInputModeUtil {

    public static void attachActivity(Activity activity, ViewGroup group) {
        new FullscreenInputModeUtil(activity, group);
    }


    private View mChildOfContent;
    private int usableHeightPrevious;
    private FrameLayout.LayoutParams mChildLayoutParams;
    private ViewGroup mInputParentView;

    private FullscreenInputModeUtil(Activity activity, ViewGroup viewGroup) {
        FrameLayout content = (FrameLayout) activity.findViewById(Window.ID_ANDROID_CONTENT);
        mInputParentView = viewGroup;
        mInputParentView.setBackgroundColor(Color.WHITE);
        mChildOfContent = content.getChildAt(0);
        mChildOfContent.getViewTreeObserver().addOnGlobalLayoutListener(() -> {
            // 当在一个视图树中全局布局发生改变或者视图树中的某个视图的可视状态发生改变时，所要调用的回调函数的接口类
            resizeChildOfContent();
        });
        mChildLayoutParams = (FrameLayout.LayoutParams) mChildOfContent.getLayoutParams();
    }

    /**
     * 重置 content 布局，通过计算可见范围的变化值，确定输入框的高度（兼容一个状态栏高度）
     */
    private void resizeChildOfContent() {
        int visibleHeightNow = ViewUtil.getVisibleHeightInLayout(mChildOfContent);
        if (visibleHeightNow != usableHeightPrevious) {
            int visibleHeightSansKeyboard = mChildOfContent.getRootView().getHeight();
            int heightDifference = visibleHeightSansKeyboard - visibleHeightNow;
            if (heightDifference > (visibleHeightSansKeyboard / 4)) {
                // 输入法出现
                mChildLayoutParams.height = visibleHeightSansKeyboard - heightDifference + EnvUtil.getStatusBarHeight();
                mInputParentView.setPadding(0, 0, 0, 0);
            } else {
                // 输入法消失
                mChildLayoutParams.height = visibleHeightSansKeyboard;
                mInputParentView.setPadding(0, 0, 0, EnvUtil.getNavigationBarHeight());
            }
            mChildOfContent.requestLayout();
            usableHeightPrevious = visibleHeightNow;
        }
    }

}
```

### 总结

本文主要是提供一个具体可行方案和开发中遇到的一些坑的解决方案提供出来。如果你的应用主色调不是纯白色，那么理论上可以完全适配到 4.4 (19) 。

对于沉浸式，众说纷纭，但是唯一的目的，就是提高用户体验。

如果还有什么遗漏的地方，欢迎补充。


### 参考

- [Android App 沉浸式状态栏解决方案](http://jaeger.itscoder.com/android/2016/02/15/status-bar-demo.html)
- [与 Status Bar 和 Navigation Bar 相关的一些东西](http://angeldevil.me/2014/09/02/About-Status-Bar-and-Navigation-Bar/)
- [Android How to adjust layout in Full Screen Mode when softkeyboard is visible](http://stackoverflow.com/questions/7417123/android-how-to-adjust-layout-in-full-screen-mode-when-softkeyboard-is-visible)
