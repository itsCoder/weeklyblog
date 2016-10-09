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

# 使用 Vector 如何适配图标

# 如何使用 Animated Vector Drawable

# 总结