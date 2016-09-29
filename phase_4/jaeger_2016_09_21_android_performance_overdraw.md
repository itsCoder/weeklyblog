---
title: Android 过度绘制优化
category: Android 
excerpt: Android 从一诞生到现在已经发布的 7.0 版本，卡顿和不流畅问题却一直被人们所诟病。从开发角度来说，每个开发者都应该关注下性能优化，在平时的开发工作中注意一些细节，尽可能地去优化应用。本文作为性能优化系列的开篇，先从过度绘制优化讲起。
--- 
>- 文章来源：itsCoder 的 [WeeklyBolg](https://github.com/itsCoder/weeklyblog) 项目
>- itsCoder主页：[http://itscoder.com/](http://itscoder.com/)
>- 作者：[Jaeger]( https://github.com/laobie)
>- 审阅者：[yongyu0102 \(用语\)](https://github.com/yongyu0102)

Android 从一诞生到现在已经发布的 7.0 版本，卡顿和不流畅问题却一直被人们所诟病。客观地来讲，Android 的流畅性确实一直不给力，哪怕是某些大厂的 App ，也都不同程度地存在卡顿问题。从开发角度来说，每个开发者都应该关注下性能优化，在平时的开发工作中注意一些细节，尽可能地去优化应用。本文作为性能优化系列的开篇，先从过度绘制优化讲起。

### 过度绘制（Overdraw）的概念

> 过度绘制（Overdraw）描述的是屏幕上的某个像素在同一帧的时间内被绘制了多次。在多层次重叠的 UI 结构里面，如果不可见的 UI 也在做绘制的操作，会导致某些像素区域被绘制了多次，同时也会浪费大量的 CPU 以及 GPU 资源。

在 Android 手机的开发者选项中，有一个『调试 GPU 过度绘制』的选项，该选项开启之后，手机显示如下，显示出来的蓝色、绿色的色块就是过度绘制信息。

![](http://ac-QYgvX1CC.clouddn.com/c7de9ce128cd8921.png)

比如上面界面中的『调试 GPU 过度绘制 』的那个文本显示为蓝色，表示其过度绘制了一次，因为背景是白色的，然后文字是黑色的，导致文字所在的区域就会被绘制两次：一次是背景，一次是文字，所以就产生了过度重绘。

在官网的 [Debug GPU Overdraw Walkthrough](https://developer.android.com/studio/profile/dev-options-overdraw.html) 说明中对过度重绘做了简单的介绍，其中屏幕上显示不同色块的具体含义如下所示：

![](http://ac-QYgvX1CC.clouddn.com/46397b26da912658.png)

每个颜色的说明如下：

- 原色：没有过度绘制
- 蓝色：1 次过度绘制
- 绿色：2 次过度绘制
- 粉色：3 次过度绘制
- 红色：4 次及以上过度绘制

过度绘制的存在会导致界面显示时浪费不必要的资源去渲染看不见的背景，或者对某些像素区域多次绘制，就会导致界面加载或者滑动时的不流畅、掉帧，对于用户体验来说就是 App 特别的卡顿。为了提升用户体验，提升应用的流畅性，优化过度绘制的工作还是很有必要做的。

### 优化原则

- 一些过度绘制是无法避免的，比如之前说的文字和背景导致的过度绘制，这种是无法避免的。
- 应用界面中，应该尽可能地将过度绘制控制为 2 次（绿色）及其以下，原色和蓝色是最理想的。
- 粉色和红色应该尽可能避免，在实际项目中避免不了时，应该尽可能减少粉色和红色区域。
- 不允许存在面积超过屏幕 1/4 区域的 3 次（淡红色区域）及其以上过度绘制。

### 优化方法

以下部分是根据我在公司项目的实践来整理出来的一些实际的优化步骤和方法，避免像看完大部分性能优化的文章，然后发现『懂得太多道理还是写不好一个 App』的尴尬局面。

1. 移除默认的 Window 背景

   一般应用默认继承的主题都会有一个默认的 `windowBackground` ，比如默认的 Light 主题：

   ```xml
   <style name="Theme.Light">
       <item name="isLightTheme">true</item>
       <item name="windowBackground">@drawable/screen_background_selector_light</item>
       ...
   </style>
   ```

   但是一般界面都会自己设置界面的背景颜色或者列表页则由 item 的背景来决定，所以默认的 Window 背景基本用不上，如果不移除就会导致所有界面都多 1 次绘制。

   可以在应用的主题中添加如下的一行属性来移除默认的 Window 背景：

   ```xml
   <item name="android:windowBackground">@android:color/transparent</item>
   <!-- 或者 -->
   <item name="android:windowBackground">@null</item>
   ```

   或者在 `BaseActivity` 的 `onCreate()` 方法中使用下面的代码移除：

   ```java
   getWindow().setBackgroundDrawable(null);
   // 或者
   getWindow().setBackgroundDrawableResource(android.R.color.transparent);
   ```

   移除默认的 Window 背景的工作在项目初期做最好，因为有可能有的界面未设置背景色，这就会导致该界面显示成黑色的背景，如下所示，如果是后期移除的，就需要检查移除默认 Window 背景之后的界面是否显示正常。

   ![](http://ac-QYgvX1CC.clouddn.com/8bb76d317ff0d5ff.png)

2. 移除不必要的背景

   还是上面的那个界面，因为移除了默认的 Window 背景，所以在布局中设置背景为白色：

   ```xml
   <?xml version="1.0" encoding="utf-8"?>
   <LinearLayout
       xmlns:android="http://schemas.android.com/apk/res/android"
       android:layout_width="match_parent"
       android:layout_height="match_parent"
       android:background="@color/white"
       android:orientation="vertical">
       
       <android.support.v7.widget.RecyclerView
           android:id="@+id/rv_apps"
           android:layout_width="match_parent"
           android:layout_height="match_parent"
           android:visibility="visible"/>
       
   </LinearLayout>
   ```

   然后在列表的 item 的布局如下所示：

   ```xml
   <?xml version="1.0" encoding="utf-8"?>
   <LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
       xmlns:tools="http://schemas.android.com/tools"
       android:layout_width="match_parent"
       android:layout_height="wrap_content"
       android:background="@color/white"
       android:orientation="horizontal"
       android:padding="@dimen/mid_dp">

       <ImageView
           android:id="@+id/iv_app_icon"
           android:layout_width="40dp"
           android:layout_height="40dp"
           tools:src="@mipmap/ic_launcher"/>

       <TextView
           android:id="@+id/tv_app_label"
           android:layout_width="wrap_content"
           android:layout_height="wrap_content"
           android:layout_gravity="center_vertical"
           android:layout_marginLeft="@dimen/mid_dp"
           android:textColor="@color/text_gray_main"
           android:textSize="@dimen/mid_sp"
           tools:text="test"/>
   </LinearLayout>
   ```

   看起来是没问题的，但是因为我界面的背景和 item 布局的背景都是白色，所以 item 布局中的背景是不必要的，可以移除。优化前后的过度绘制结果如下：

   ![](http://ac-QYgvX1CC.clouddn.com/eeffd1ea58fd9598.png)

   很明显优化后过度绘制比之前均少了一次，但是这种场景还是比较特殊的，因为界面背景和 item 的背景色一样，假如不一样的话，就无法避免多 1 次过度绘制了。

   还有一个比较常见的可优化场景：ViewPager 加多个 Fragment 组成的首页界面，如果你的每个 Fragment 都设置有背景色的话， 你就可以不用给 Activity 的根布局设置背景，如果你还给 ViewPager 还设置了背景，那个这个背景是没必要的，同样可以移除。

   如果你不知道存在哪些无用的背景，你可以借助 Hierarchy View 来查看，具体的这块可以参照 [Android 性能优化之过渡绘制（二）](http://androidperformance.com/2015/01/13/android-performance-optimization-overdraw-2.html) 这篇文章来操作。

3. 写合理且高效的布局

   由于 Android 的布局是通过编写 xml 来实现，相对比较简单，这也就导致很多开发者在写布局时很随意，而不会考虑性能、过度重绘等问题。

   比如上面列表布局中的分割线，可以按照如下编写布局来实现：

   ```xml
   <?xml version="1.0" encoding="utf-8"?>
   <LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
       xmlns:tools="http://schemas.android.com/tools"
       android:layout_width="match_parent"
       android:layout_height="wrap_content"
       android:paddingBottom="8dp"
       android:background="@color/divider_gray">

       <LinearLayout
           android:padding="@dimen/mid_dp"
           android:layout_width="match_parent"
           android:layout_height="wrap_content"
           android:orientation="horizontal"
           android:background="@color/white">

           <ImageView
               android:id="@+id/iv_app_icon"
               android:layout_width="40dp"
               android:layout_height="40dp"
               tools:src="@mipmap/ic_launcher"/>

           <TextView
               android:id="@+id/tv_app_label"
               android:layout_width="wrap_content"
               android:layout_height="wrap_content"
               android:layout_gravity="center_vertical"
               android:layout_marginLeft="@dimen/mid_dp"
               android:textColor="@color/text_gray_main"
               android:textSize="@dimen/mid_sp"
               tools:text="test"/>
       </LinearLayout>

   </LinearLayout>
   ```

   这种改变布局实现分割线的方式虽然很快捷方便，但是存在不少问题的：

   - 加深了布局层级，和之前的布局相比多了一级

     ![](http://ac-QYgvX1CC.clouddn.com/2aceb1e5a933352a.jpg)

   - 多了 2 次过度绘制

   解决方式有两种：

   1. 一种是使用 `RelativeLayout` 将分割线添加在 item 的布局中，但是这样会导致布局复杂度增加，同时因为 `RelativeLayout` 布局的两次测量，也会延长 View 测量的时间，在解决这种需求时并不是一个好的方式。
   2. 另一种是使用 `RecyclerView` 的 `addItemDecoration(ItemDecoration decor)` 方法添加分割线，这种方式在你自定义好一个分割线 `ItemDecoration` 时是很方便的，网上有很多关于这方面的例子（如果你使用 ListView 的话，则使用 `setDivider(Drawable divider)` 方法）。

   我们采用第二种解决方法，优化前后的对比如下：

   ![](http://ac-QYgvX1CC.clouddn.com/fad0b600790d3986.png)

   优化后的布局 ImageView 和 item 背景区域均比优化前少了 2 次过度重绘，布局层级也没增加，需求也实现了。

   > 注：很多开发者在开发中一般很少注意这种小细节，一般以完成需求为目的，可能还认为这么点细节优化不优化其实也没什么，但是积少成多，小的细节优化多了，整体性能和体验可能就上升了，相反，这个细节不注意那个细节无所谓，最终就导致应用卡顿，体验糟糕。注重细节的开发者运气一般都不会太差。: )

4. 自定义控件使用 `clipRect()` 和 `quickReject()` 优化 

   当某些控件不可见时，如果还继续绘制更新该控件，就会导致过度绘制。但是通过 Canvas  `clipRect()` 方法可以设置需要绘制的区域，当某个控件或者 View 的部分区域不可见时，就可以减少过度绘制。

   先看一下 `clipRect()` 方法的说明：

   > Intersect the current clip with the specified rectangle, which is expressed in local coordinates.

   顾名思义就是给 Canvas 设置一个裁剪区，只有在这个裁剪矩形区域内的才会被绘制，区域之外的都不绘制。 `DrawerLayout` 就是一个很不错的例子，先来看一下使用 DrawerLayout 布局的过度绘制结果：

   ![](http://ac-QYgvX1CC.clouddn.com/3ac552385fa37312.png)

   按道理左边的抽屉布局出来时，应该是和主界面的布局叠加起来的，但是为什么抽屉的背景过度绘制只有一次呢？如果是叠加的话，那最少是主界面过度绘制次数 +1，但是结果并不是这样。直接看源码：

   ```java
   @Override
   protected boolean drawChild(Canvas canvas, View child, long drawingTim
       final int height = getHeight();
       final boolean drawingContent = isContentView(child);
       int clipLeft = 0, clipRight = getWidth();
       final int restoreCount = canvas.save();
       if (drawingContent) {
           final int childCount = getChildCount();
           for (int i = 0; i < childCount; i++) {
               final View v = getChildAt(i);
               if (v == child || v.getVisibility() != VISIBLE
                       || !hasOpaqueBackground(v) || !isDrawerView(v)
                       || v.getHeight() < height) {
                   continue;
               }
               if (checkDrawerViewAbsoluteGravity(v, Gravity.LEFT)) {
                   final int vright = v.getRight();
                   if (vright > clipLeft) clipLeft = vright;
               } else {
                   final int vleft = v.getLeft();
                   if (vleft < clipRight) clipRight = vleft;
               }
           }
           canvas.clipRect(clipLeft, 0, clipRight, getHeight());
       }
       ......                       
   }
   ```

   在 DrawerLayout 的 `drawChild()` 方法一开始会判断是是否是 DrawerLayout 的 ContentView，即非抽屉布局，如果是的话，则遍历 DrawerLayout 的 child view，拿到抽屉布局，如果是左边抽屉，则取抽屉布局的右边边界作为裁剪区的左边界，得到的裁剪矩形就是下图中的红色框部分，然后设置裁剪区域。右边抽屉同理。

   ![](http://ac-QYgvX1CC.clouddn.com/f2bd8c92d4f03a9b.jpg)

   这样一来，只有裁剪矩形内的界面需要绘制，自然就减少了抽屉布局的过度绘制。自定义控件时可以参照这个来优化过度绘制问题。

   除了 `clipRect()` 以为，还可以使用 `canvas.quickreject()` 来判断和某个矩形相交，如果相交的话，则可以跳过相交的区域减少过度绘制。

### 优化实践

前面其实已经讲了很多了，但是实际去优化过度绘制时，可能还是会比较懵，看着屏幕上的大片大片的红色，不知道从何下手。接下来就以实际项目中的过度绘制优化经历来谈谈，如何进行优化？

先上图，前面是未开启 『调试 GPU 过度绘制』 的界面图，中间的是优化前的过度绘制结果，后面的是优化后的过度绘制结果，不难看出来，中间那张图过度绘制是很严重的，一眼看过去一片红，很显示不符合优化原则。

![](http://ac-QYgvX1CC.clouddn.com/1bed5940cdfa0701.png)

优化步骤如下：

1. 先分析每个地方最少可以绘制几次，不合理的地方就可以优化。

   例如：中间那张图显示的每个 item 的背景是绿色的，也就是 2 次过度绘制，这肯定是不合理的。因为整个界面大背景是灰色的，item 背景是白色的，按道理应该就 1 次过度绘制。检查下来发现没去掉默认的 Window 背景，移除之后 item 背景就变成了蓝色了，也就是 1 次过度绘制。

2. 叠加的布局，过度绘制次数是否合理递增

   还是看中间那张图，item 的背景过度绘制是 2 次，按道理九宫格图片每张图应该是过度绘制 3 次，但是却显示成红色的，显然没有合理递增而出现了跳跃。

   先猜测是不是因为给九宫格图片控件设置了白色背景？但是想一下就排除了，因为图片间隙的过度绘制次数和 item 背景是相同的。

   那就是每个 ImageView 有问题了，后来发现之前设置占位图的时候，给每个 ImageView 设置了一个灰色的背景色：

   ```java
   imageView.setBackgroundColor(Color.parseColor("#eeeeee"));
   ```

   这也就导致了每个 ImageView 的过度绘制直接多了 1 次。

   这两步优化后，再看最后一张图中的优化结果，基本是可以的了。

3. 在 **优化方法** 中讲到的 ViewPager 布局加 Fragment 实现的首页布局，一个不注意很容易出现过度绘制严重的问题，在移除 ViewPager 和 Activity 根布局的白色背景后，以及默认的 Window 背景，原来红成一片的首页现在基本上是大部分蓝色和小部分绿色了。

   ![](http://ac-QYgvX1CC.clouddn.com/5e3d906c721cc9f3.png)



### 小插曲

最后来个小插曲，因为开启 『调试 GPU 过度绘制』比较麻烦，我就想找个比较方便快捷的方式，一开始想着写个桌面插件应用，一键切换。

- 查文档发现没有相关的设置的 API 

- 直接翻源码，发现相关的 API 是隐藏的，集中在 `SystemProperties` 类中，可以通过如下代码设置：

  ```java
  SystemProperties.set(HardwareRenderer.DEBUG_OVERDRAW_PROPERTY, "show");
  ```

- 直接编译源码拿到了没隐藏的 jar 包，暂时能调用到该类，但是运行之后发现需要系统权限才能设置

- 通过一些方式企图让这个 App 获取到系统权限，但是均失败了 : (

如果你对相关的知识有所了解，请联系我和我探讨下，谢谢。

不过最后也算是找到了一个比较方便的方法，省去了去设置里面一步步点。直接运行 adb 指令：

开启『调试 GPU 过度绘制』：

```shell
adb shell setprop debug.hwui.overdraw show
```

关闭『调试 GPU 过度绘制』：

```shell
adb shell setprop debug.hwui.overdraw false
```

再取个指令别名，使用起来还是很方便的。

### 参考资料

- [Android 性能优化之渲染篇 \- 胡凯](http://hukai.me/android-performance-render/)
- [Android 性能优化之过渡绘制\(一\) \| Performance](http://androidperformance.com/2014/10/20/android-performance-optimization-overdraw-1.html)
- [Android 性能分析案例 \- 云在千峰](http://blog.chengyunfeng.com/?p=458#)
- [Android UI 性能优化详解](http://mrpeak.cn/android/2016/01/11/android-performance-ui)
- [Speed up your app](http://blog.udinic.com/2015/09/15/speed-up-your-app)



