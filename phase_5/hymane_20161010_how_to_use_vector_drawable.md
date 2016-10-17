---
title: Android 如何使用矢量图片以及使用矢量动画
date: 2016-10-09 12:50:20
categories: Android
tags: [Android, vector-drawable, animated-vector-drawable, dpi]
---

# 引言
Android 开发中，屏幕适配一直是一个令人头疼的工作，就应用图标资源而言便需要准备 N 套以适配不同分辨率的屏幕。当然，随着手机硬件的飞速发展，曾经的 320*480、640*480 的手机现在已经基本被淘汰，可以说目前市场上手机几乎都是 1k+ 屏幕，并且向着 3k、4k 进军。这也意味着我们曾经的`drawable-hdpi`现在已经不适用了，细心的你肯定会发现现在新建的项目多了一个`drawable-xxxhdpi`文件夹，这是 Android4.2 引进的，是用来适配平板和电视的 4k 屏幕的。我们知道了这些，那么就会做图标适配了，但是不得不说，准备 N 套小图标确实很烦，还很浪费空间。幸运的是，我们可以使用 Vector Drawable 来代替那 N 套图标，还可以用它来做一些有趣的动画。

# 什么是 Vector Drawable
Vector Drawable 就是矢量图的意思，也就是 svg 图片，其实它并不是图片，而是一个文本文件，将图片的线条和土块用一种标记语言记录下来，然后就可以在支持矢量图的软件上使用它了。矢量图最突出的优点就是体积小、无限放大不失真等。当然 Android 也支持矢量图了，只不过目前只能在 [LOLLIPOP](https://www.android.com/versions/lollipop-5-0/) 及以上系统使用，使用 Vector Drawable 可以让你只使用一套体积极小的 xml 文件来代替那 N 套 PNG 图片，不仅仅减小了 app 体积，其显示效果要比 PNG 图片好很多。

# 不用 Vector 如何适配图标
如果我们考虑到 app 兼容低版本，暂不适用 Vector 图，只采用 PNG 图标，那么我们该如何做适配？以下是几条建议。

1.  让我们亲爱的 UI 准备 N 套不同分辨率的图标，包括`drawable-hdpi`,`drawable-mdpi`,`drawable-xhdpi`,`drawable-xxhdpi`,`drawable-xxxhdpi`。然后你将这些图整理一下，以此放入项目对应文件夹下，记得重命名。
2.  万一你很不幸，UI 给你的是`@1x`,`@2x`,`@3x`这3套图，呵呵，总不能自己 P 吧？这个时候如果你们的项目要求的精度不是很高的话，你可以考虑将`@3x`，`@2x`,`@1x`图标分别放入`drawable-xxhdpi`,`drawable-xhdpi`,`drawable-mdpi`,还有两个分辨率`drawable-hdpi`,`drawable-xxxhdpi`可以忽略，用不上。
3.  如果你们项目要求比较高，需要精确一点的图标显示，你可以考虑使用 Android Studio 自带的 Image Asset 工具，选取一张大图，比如`@3x`里面的图，让 Image Asset 帮你把图切好，会自动按比率生成对应 dpi 的图标，放入对应的文件夹里面，十分好用。不生成`drawable-xxxhdpi`分辨率。
4.  通过解压一些大公司的 apk 你会发现，他们的 app 普遍只用了2套图，那就是`drawable-xxhdpi`和`drawable-nodpi`，Android 系统在低分辨屏幕上会自动将图片进行压缩，以达到显示效果，毕竟压缩比放大产生的显示效果影响要小一点。所以我们可以在第3条建议的基础上再删除一些没必要的 drawable，或者只让 UI 准备一套`drawable-xxhdpi`图标，以达到节省体积的效果，对显示效果的影响并不大。

# 如何使用 Vector Drawable
1： 如果你的设备最低支持 Android5.0 及以上，那就可以直接在项目中使用 VectorDrawable，如同使用普通图片一样，比如 xml 文件中使用`android:src="@drawable/ic_action_find"`，其中 ic_action_find 就是一个 VectorDrawable,它不是一个 png 图片，而是一个 xml 文件。

![lizi](http://ww3.sinaimg.cn/mw690/005X6W83jw1f8vi4nn7udj3044048mx5.jpg)

```xml
<?xml version="1.0" encoding="utf-8"?>
<vector xmlns:android="http://schemas.android.com/apk/res/android"
    android:width="24dp"
    android:height="24dp"
    android:viewportHeight="24"
    android:viewportWidth="24">

    <path
        android:name="toe1"
        android:fillColor="#ffffff"
        android:pathData="M 4.5 7 C 5.88071187458 7 7 8.11928812542 7 9.5 C 7 10.8807118746 5.88071187458 12 4.5 12 C 3.11928812542 12 2 10.8807118746 2 9.5 C 2 8.11928812542 3.11928812542 7 4.5 7 Z" />
    <path
        android:name="toe2"
        android:fillColor="#ffffff"
        android:pathData="M 9 3 C 10.3807118746 3 11.5 4.11928812542 11.5 5.5 C 11.5 6.88071187458 10.3807118746 8 9 8 C 7.61928812542 8 6.5 6.88071187458 6.5 5.5 C 6.5 4.11928812542 7.61928812542 3 9 3 Z" />
    <path
        android:name="toe3"
        android:fillColor="#ffffff"
        android:pathData="M 15 3 C 16.3807118746 3 17.5 4.11928812542 17.5 5.5 C 17.5 6.88071187458 16.3807118746 8 15 8 C 13.6192881254 8 12.5 6.88071187458 12.5 5.5 C 12.5 4.11928812542 13.6192881254 3 15 3 Z" />
    <path
        android:name="toe4"
        android:fillColor="#ffffff"
        android:pathData="M 19.5 7 C 20.8807118746 7 22 8.11928812542 22 9.5 C 22 10.8807118746 20.8807118746 12 19.5 12 C 18.1192881254 12 17 10.8807118746 17 9.5 C 17 8.11928812542 18.1192881254 7 19.5 7 Z" />
    <path
        android:fillColor="#ffffff"
        android:pathData="M17.34 14.86c-.87-1.02-1.6-1.89-2.48-2.91-.46-.54-1.05-1.08-1.75-1.32-.11-.04-.22-.07-.33-.09-.25-.04-.52-.04-.78-.04s-.53 0-.79 .05 c-.11 .02 -.22 .05 -.33 .09 -.7 .24 -1.28 .78 -1.75 1.32-.87 1.02-1.6 1.89-2.48 2.91-1.31 1.31-2.92 2.76-2.62 4.79 .29 1.02 1.02 2.03 2.33 2.32 .73 .15 3.06-.44 5.54-.44h.18c2.48 0 4.81 .58 5.54 .44 1.31-.29 2.04-1.31 2.33-2.32 .31 -2.04-1.3-3.49-2.61-4.8z" />
</vector>
```

上面这个 xml 文件显示的图形如下

 ![vector pet](http://ww1.sinaimg.cn/mw690/005X6W83jw1f8vhn2wsv0j30ar0aqq2t.jpg)

最外层是一个 vector 标签，表示这是一个矢量图片，其中每一个 path 标签画出了图片中一个独立的“块”，比如上面一种有5个 path 标签，前4个画出了脚丫的四个脚趾，最后一个长的 path 则画出了脚掌心的区块，最终形成了脚丫的形状。`width`和`height`确定的是 drawable 的尺寸，`viewportHeight`和`viewportWidth`确定的是画布的大小。
我们再来看看 path 标签，`fillColor`属性很好理解，填充的颜色，每个 path 中最重要的是`pathData`属性，通过这个属性来确定这个区块到底是如何绘制。首先我们看看 SVG path 的基本命令：

* M： move to 移动绘制点
* L：line to 直线
* Z：close 闭合
* C：cubic bezier 三次贝塞尔曲线
* Q：quatratic bezier 二次贝塞尔曲线
* A：ellipse 圆弧

详细使用

* M (x y) 移动到 x,y
* L (x y) 直线连到 x,y，还有简化命令 H(x) 水平连接、V(y) 垂直连接
* Z，没有参数，连接起点和终点
* C(x1 y1 x2 y2 x y)，控制点 x1,y1 x2,y2，终点 x,y
* Q(x1 y1 x y)，控制点 x1,y1，终点 x,y
* A(rx ry x-axis-rotation large-arc-flag sweep-flag x y) rx ry 椭圆半径 x-axis-rotation x 轴旋转角度 large-arc-flag 为0时表示取小弧度，1时取大弧度 sweep-flag 0取逆时针方向，1取顺时针方向 

 ![no diao use](http://ww4.sinaimg.cn/mw690/005X6W83jw1f8vi5kdhrgj3046053jrd.jpg)

你知道这些命令也没什么用，实际上，几乎没人手写这个 path 路径，一般都是用软件生成，比如 AI,PS 等软件，了解一下就好。你可以在网上找一些现有的 svg 图片然后将其导入到 Android Studio 中来，或者直接使用 Android Studio 的 Vector Asset 工具选择你需要的 vector drawable。
> 右键项目任意目录->New->Vector Asset.

2： 抱歉，我想适配5.0以下系统。也可以，Android 给我们准备了向下兼容库 `Support Vector Drawables`，它可以帮助我们在5.0以下使用矢量图片。

* 如果你的 Gradle Plugin 版本为2.0+,那在你的 build.gradle 加入`vectorDrawables.useSupportLibrary = true`。

    ```xml
     // Gradle Plugin 2.0+  
     android {  
       defaultConfig {  
         vectorDrawables.useSupportLibrary = true  
        }  
     } 
    ```

> Gradle 2.0以下尼？

![you are drunk](http://ww4.sinaimg.cn/mw690/005X6W83jw1f8vi6z3joej30dw09a3yu.jpg)

*  然后在 xml 中将以往 `android:src="@drawable/ic_add"` 改成 `app:srcCompat="@drawable/ic_add"`来使用。
*  代码中使用和以往没区别，还是使用`setImageResource()`方法。
*  也许你会问，假如我想在 TextView 的`android:drawableLeft`中使用或者是 menu 文件的 icon 属性中使用 vector drawable 该如何使用？我们要知道 menu 中没有 src 属性，图片来自于 icon 属性，这里要明确的是，在
`Android Support Library 23.2.0` 中,可以在我们的 Vector Drawable 外面包裹一层 `StateListDrawable`, `InsetDrawable`, `LayerDrawable`, `LevelListDrawable`, or `RotateDrawable`等 Drawable 而不是直接去加载 vector，这样是可以在低版本上使用 vector 的。但是，从`Android Support Library 23.3.0`以后这种用法就失效了，只能通过`app:srcCompat` 或者 `setImageResource()`来使用。

3: 也许为了兼容以及能在 menu 中使用 vector 得不到很好的平衡。在 Android Studio 上新建带抽屉的模板 demo 时候，你会发现，它使用到了 vector，并且做了兼容处理，模板的做法是在5.0以上使用 vector 以得到更高效果的图片，而在5.0以下，默认使用 png 代替。将你所有可能需要在 menu 或者 drawableLeft 等地方使用 vector 的矢量图片全部放入 `drawable-v21` 中，然后在 `value`中创建一个`drawables.xml`文件。来做映射关系。

```xml
<resources>
    <item name="ic_action_find_xml" type="drawable">@android:drawable/ic_action_find_png</item>
</resources>
```

如上所示，如果该 vector 只在 src 属性中出现，也就意味着你可以使用兼容库来做适配，因此不需要做这个映射。

# 如何使用 Animated Vector Drawable

![pet](http://ww3.sinaimg.cn/mw690/005X6W83jw1f8vkft7yshg30dc0k01l0.gif)

![share](http://ww1.sinaimg.cn/mw690/005X6W83jw1f8vkfxczslg30dc0k0x6q.gif)

首先让我们看看上面这两个动画，一个是脚指头上下跳动动画，一个是分享按钮的路径跳动动画。通过 Android 属性动画或者帧动画都可以实现，帧动画需要提供图片资源，属性动画写起来应该很烦。如果考虑用`animated vector drawable`,那就很简单了。

1. 在`drawable-v21`新建`ic_action_pet`,即上面举得那个例子，脚掌的原始图片，即 VectorDrawable。
2. 在`drawable-v21`新建`ic_action_pet_anim.xml`，即 AnimatedVectorDrawable，带有动画效果的 drawable。
    ```xml
    <animated-vector xmlns:android="http://schemas.android.com/apk/res/android"
        android:drawable="@drawable/ic_action_pet">
        <target
            android:name="toe1"
            android:animation="@anim/anim_path_translate1"/>
        <target
            android:name="toe2"
            android:animation="@anim/anim_path_translate2"/>
        <target
            android:name="toe3"
            android:animation="@anim/anim_path_translate3"/>
        <target
            android:name="toe4"
            android:animation="@anim/anim_path_translate4"/>
    </animated-vector>
    ```
3. 在`anim`文件夹下新建`anim_path_translate1`,`anim_path_translate2`,`anim_path_translate3`,`anim_path_translate4`四个局部动画文件，用于对4个脚趾头进行动画。以下按顺序列出。
anim_path_translate1
    ```xml
        <?xml version="1.0" encoding="utf-8"?>
        <set xmlns:android="http://schemas.android.com/apk/res/android"
            android:ordering="sequentially">

            <objectAnimator
                android:duration="420"
                android:propertyName="pathData"
                android:repeatCount="-1"
                android:repeatMode="reverse"
                android:valueFrom="M 4.5 7 C 5.88071187458 7 7 8.11928812542 7 9.5 C 7 10.8807118746 5.88071187458 12 4.5 12 C 3.11928812542 12 2 10.8807118746 2 9.5 C 2 8.11928812542 3.11928812542 7 4.5 7 Z"
                android:valueTo="M 4.5 9 C 5.88071187458 9 7 10.1192881254 7 11.5 C 7 12.8807118746 5.88071187458 14 4.5 14 C 3.11928812542 14 2 12.8807118746 2 11.5 C 2 10.1192881254 3.11928812542 9 4.5 9 Z"
                android:valueType="pathType" />

            <objectAnimator
                android:duration="840"
                android:propertyName="pathData"
                android:repeatCount="-1"
                android:repeatMode="reverse"
                android:valueFrom="M 4.5 9 C 5.88071187458 9 7 10.1192881254 7 11.5 C 7 12.8807118746 5.88071187458 14 4.5 14 C 3.11928812542 14 2 12.8807118746 2 11.5 C 2 10.1192881254 3.11928812542 9 4.5 9 Z"
                android:valueTo="M 4.5 4 C 5.88071187458 4 7 5.11928812542 7 6.5 C 7 7.88071187458 5.88071187458 9 4.5 9 C 3.11928812542 9 2 7.88071187458 2 6.5 C 2 5.11928812542 3.11928812542 4 4.5 4 Z"
                android:valueType="pathType" />

            <objectAnimator
                android:duration="420"
                android:propertyName="pathData"
                android:repeatCount="-1"
                android:repeatMode="reverse"
                android:valueFrom="M 4.5 4 C 5.88071187458 4 7 5.11928812542 7 6.5 C 7 7.88071187458 5.88071187458 9 4.5 9 C 3.11928812542 9 2 7.88071187458 2 6.5 C 2 5.11928812542 3.11928812542 4 4.5 4 Z"
                android:valueTo="M 4.5 7 C 5.88071187458 7 7 8.11928812542 7 9.5 C 7 10.8807118746 5.88071187458 12 4.5 12 C 3.11928812542 12 2 10.8807118746 2 9.5 C 2 8.11928812542 3.11928812542 7 4.5 7 Z"
                android:valueType="pathType" />
        </set>
    ```
anim_path_translate2
    ```xml
        <?xml version="1.0" encoding="utf-8"?>
        <set xmlns:android="http://schemas.android.com/apk/res/android"
            android:ordering="sequentially"
            android:shareInterpolator="true">

            <objectAnimator
                android:duration="480"
                android:propertyName="pathData"
                android:repeatCount="-1"
                android:repeatMode="reverse"
                android:startOffset="100"
                android:valueFrom="M 9 3 C 10.3807118746 3 11.5 4.11928812542 11.5 5.5 C 11.5 6.88071187458 10.3807118746 8 9 8 C 7.61928812542 8 6.5 6.88071187458 6.5 5.5 C 6.5 4.11928812542 7.61928812542 3 9 3 Z"
                android:valueTo="M 9 5 C 10.3807118746 5 11.5 6.11928812542 11.5 7.5 C 11.5 8.88071187458 10.3807118746 10 9 10 C 7.61928812542 10 6.5 8.88071187458 6.5 7.5 C 6.5 6.11928812542 7.61928812542 5 9 5 Z"
                android:valueType="pathType" />

            <objectAnimator
                android:duration="960"
                android:propertyName="pathData"
                android:repeatCount="-1"
                android:repeatMode="reverse"
                android:valueFrom="M 9 5 C 10.3807118746 5 11.5 6.11928812542 11.5 7.5 C 11.5 8.88071187458 10.3807118746 10 9 10 C 7.61928812542 10 6.5 8.88071187458 6.5 7.5 C 6.5 6.11928812542 7.61928812542 5 9 5 Z"
                android:valueTo="M 9 0 C 10.3807118746 0 11.5 1.11928812542 11.5 2.5 C 11.5 3.88071187458 10.3807118746 5 9 5 C 7.61928812542 5 6.5 3.88071187458 6.5 2.5 C 6.5 1.11928812542 7.61928812542 0 9 0 Z"
                android:valueType="pathType" />

            <objectAnimator
                android:duration="480"
                android:propertyName="pathData"
                android:repeatCount="-1"
                android:repeatMode="reverse"
                android:valueFrom="M 9 0 C 10.3807118746 0 11.5 1.11928812542 11.5 2.5 C 11.5 3.88071187458 10.3807118746 5 9 5 C 7.61928812542 5 6.5 3.88071187458 6.5 2.5 C 6.5 1.11928812542 7.61928812542 0 9 0 Z"
                android:valueTo="M 9 3 C 10.3807118746 3 11.5 4.11928812542 11.5 5.5 C 11.5 6.88071187458 10.3807118746 8 9 8 C 7.61928812542 8 6.5 6.88071187458 6.5 5.5 C 6.5 4.11928812542 7.61928812542 3 9 3 Z"
                android:valueType="pathType" />
        </set>
    ```
anim_path_translate3
    ```xml
        <?xml version="1.0" encoding="utf-8"?>
        <set xmlns:android="http://schemas.android.com/apk/res/android"
            android:ordering="sequentially"
            android:shareInterpolator="true">

            <objectAnimator
                android:duration="420"
                android:propertyName="pathData"
                android:repeatCount="-1"
                android:repeatMode="reverse"
                android:startOffset="200"
                android:valueFrom="M 15 3 C 16.3807118746 3 17.5 4.11928812542 17.5 5.5 C 17.5 6.88071187458 16.3807118746 8 15 8 C 13.6192881254 8 12.5 6.88071187458 12.5 5.5 C 12.5 4.11928812542 13.6192881254 3 15 3 Z"
                android:valueTo="M 15 5 C 16.3807118746 5 17.5 6.11928812542 17.5 7.5 C 17.5 8.88071187458 16.3807118746 10 15 10 C 13.6192881254 10 12.5 8.88071187458 12.5 7.5 C 12.5 6.11928812542 13.6192881254 5 15 5 Z"
                android:valueType="pathType" />

            <objectAnimator
                android:duration="840"
                android:propertyName="pathData"
                android:repeatCount="-1"
                android:repeatMode="reverse"
                android:valueFrom="M 15 5 C 16.3807118746 5 17.5 6.11928812542 17.5 7.5 C 17.5 8.88071187458 16.3807118746 10 15 10 C 13.6192881254 10 12.5 8.88071187458 12.5 7.5 C 12.5 6.11928812542 13.6192881254 5 15 5 Z"
                android:valueTo="M 15 0 C 16.3807118746 0 17.5 1.11928812542 17.5 2.5 C 17.5 3.88071187458 16.3807118746 5 15 5 C 13.6192881254 5 12.5 3.88071187458 12.5 2.5 C 12.5 1.11928812542 13.6192881254 0 15 0 Z"
                android:valueType="pathType" />

            <objectAnimator
                android:duration="420"
                android:propertyName="pathData"
                android:repeatCount="-1"
                android:repeatMode="reverse"
                android:valueFrom="M 15 0 C 16.3807118746 0 17.5 1.11928812542 17.5 2.5 C 17.5 3.88071187458 16.3807118746 5 15 5 C 13.6192881254 5 12.5 3.88071187458 12.5 2.5 C 12.5 1.11928812542 13.6192881254 0 15 0 Z"
                android:valueTo="M 15 3 C 16.3807118746 3 17.5 4.11928812542 17.5 5.5 C 17.5 6.88071187458 16.3807118746 8 15 8 C 13.6192881254 8 12.5 6.88071187458 12.5 5.5 C 12.5 4.11928812542 13.6192881254 3 15 3 Z"
                android:valueType="pathType" />

        </set>
    ```
anim_path_translate4
    ```xml
        <?xml version="1.0" encoding="utf-8"?>
        <set xmlns:android="http://schemas.android.com/apk/res/android"
            android:ordering="sequentially"
            android:shareInterpolator="true">

            <objectAnimator
                android:duration="450"
                android:propertyName="pathData"
                android:repeatCount="-1"
                android:repeatMode="reverse"
                android:startOffset="300"
                android:valueFrom="M 19.5 7 C 20.8807118746 7 22 8.11928812542 22 9.5 C 22 10.8807118746 20.8807118746 12 19.5 12 C 18.1192881254 12 17 10.8807118746 17 9.5 C 17 8.11928812542 18.1192881254 7 19.5 7 Z"
                android:valueTo="M 19.5 9 C 20.8807118746 9 22 10.1192881254 22 11.5 C 22 12.8807118746 20.8807118746 14 19.5 14 C 18.1192881254 14 17 12.8807118746 17 11.5 C 17 10.1192881254 18.1192881254 9 19.5 9 Z"
                android:valueType="pathType" />

            <objectAnimator
                android:duration="900"
                android:propertyName="pathData"
                android:repeatCount="-1"
                android:repeatMode="reverse"
                android:valueFrom="M 19.5 7 C 20.8807118746 7 22 8.11928812542 22 9.5 C 22 10.8807118746 20.8807118746 12 19.5 12 C 18.1192881254 12 17 10.8807118746 17 9.5 C 17 8.11928812542 18.1192881254 7 19.5 7 Z"
                android:valueTo="M 19.5 4 C 20.8807118746 4 22 5.11928812542 22 6.5 C 22 7.88071187458 20.8807118746 9 19.5 9 C 18.1192881254 9 17 7.88071187458 17 6.5 C 17 5.11928812542 18.1192881254 4 19.5 4 Z"
                android:valueType="pathType" />

            <objectAnimator
                android:duration="450"
                android:propertyName="pathData"
                android:repeatCount="-1"
                android:repeatMode="reverse"
                android:valueFrom="M 19.5 4 C 20.8807118746 4 22 5.11928812542 22 6.5 C 22 7.88071187458 20.8807118746 9 19.5 9 C 18.1192881254 9 17 7.88071187458 17 6.5 C 17 5.11928812542 18.1192881254 4 19.5 4 Z"
                android:valueTo="M 19.5 7 C 20.8807118746 7 22 8.11928812542 22 9.5 C 22 10.8807118746 20.8807118746 12 19.5 12 C 18.1192881254 12 17 10.8807118746 17 9.5 C 17 8.11928812542 18.1192881254 7 19.5 7 Z"
                android:valueType="pathType" />

        </set>
    ```

这些值是如何确定的？我是用 AI 软件打开一个 svg（脚掌） 文件然后拖动脚趾头路径至一个合适的位置，然后记录下当前 path，反复找出 4 个脚趾所有的路径变化起始于终止时的 path 路径，然后填充到上面4个 xml 动画对应的 value 中去即可。呵呵。
# 总结
虽然矢量图片目前在 android 支持不全，但是矢量图的优点确是十分突出的，而且 android 一直再发展，完全兼容 vector drawable 指日可待。矢量图给我们带来的方便也不言而喻，大大缩减了 app 体积，简化了 android 图片适配工作，而且使得图片显示的质量也大大提高了。
我们在使用矢量图片动画时候也给我们带来了很大的方便，让我们很方便地就能实现一些很高端大气的动画效果。