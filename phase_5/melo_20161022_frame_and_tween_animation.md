>- 文章来源：itsCoder 的 [WeeklyBolg](https://github.com/itsCoder/weeklyblog) 项目
>- itsCoder主页：[http://itscoder.com/](http://itscoder.com/)
>- 作者：[Melo](https://itsmelo.github.io/)
>- 审阅者：[暂无](旋风)

**写在前面：**
为了使用户的交互更加流畅自然，动画也就成为了一个应用中必不可少的元素之一。在 Android 中常用的动画分类无外乎三种，最早的 **帧动画** 、**补间动画**，以及 3.0 之后加入的 **属性动画**，是它们组成了 Android 中各种炫酷亮眼的动画效果。

关于动画相关的博文说实话很多，但是为什么要写这篇文章呢？因为我发现很多博客都上来就“翻译”了一通 **API** ，这对很多没有建立起 Android 动画体系概念的新人来说，非常不友好。既没有说明各种动画的应用场景，也没有横向对比动画的优缺点。对于刚学习动画的同学来说，他们读起来心里就更没底了，面对稍微复杂的动画就无从下手，就好比那句歌词“懂得很多道理，却仍过不好这一生”。所以本文要有更多思考分析之外，也会教大家一些关于动画的小技巧和可能踩到的坑。本文我们就先来研究**帧动画**和**补间动画**，话不多说，现在开始我们的内容吧。

帧动画
--
我们由简到难，先来讲讲帧动画。帧动画就是 Frame 动画，它的原理十分“复古”，和我们小时候看的动画片原理一致（注意是我们小时候），就是把一张张准备好的，一系列图片，按照指定的时间播放出来，从而达到动画的效果。

如此简单而又看似过时的帧动画，是否就被淘汰了呢？答案的自然是否定的。帧动画依然在这个复杂而有机的 Android 系统中占有一席之地。先来告诉大家帧动画的使用场景吧。

 - 设备的开机动画

 - 及其“复杂”的效果，看似不可能完成的动画

设备的开机动画界面这个没什么好解释的，据我所知市面上99%的机器都是这么做的，因为这个时候系统或资源还没准备完全，所以就肯定会选择帧动画。

![开机动画](http://upload-images.jianshu.io/upload_images/1915184-135a4d08be849a81?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

我在我以前的魅族手机 /system/media 文件夹下，找到了一系列以上这种图片，组合到一起就是开机动画了。

你可能还对我上面所说的第二种使用场景表示怀疑，前几天我看到一个应用有一个非常酷炫的效果，3D特效旋转的画面，请脑补一下数码宝贝进化的样子，我刚开始还纳闷，这个用代码怎么实现啊，想了下我想通了，这个用帧动画其实最好实现了，难为的可能就是设计师了（ 那可就不关能怪我推责任了哦~）。

介绍完了应用场景，那现在就应该来介绍到底如何在代码中使用了。

准备一个帧动画的图片资源：

![帧动画的图片资源](http://upload-images.jianshu.io/upload_images/1915184-40afdfa768a4557f?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

可以通过代码或者 xml 方式来使用**帧动画**

**XML**
新建工程，然后在 drawable 目录下新建一个 xml 文件，名字是 bear_anim ，代码如下：

```
<?xml version="1.0" encoding="utf-8"?>
<animation-list xmlns:android="http://schemas.android.com/apk/res/android"
    android:oneshot="false">

    <item
        android:drawable="@drawable/sp1"
        android:duration="200" />
    <item
        android:drawable="@drawable/sp2"
        android:duration="200" />
    <item
        android:drawable="@drawable/sp3"
        android:duration="200" />
    <item
        android:drawable="@drawable/sp4"
        android:duration="200" />
    <item
        android:drawable="@drawable/sp5"
        android:duration="200" />
    <item
        android:drawable="@drawable/sp6"
        android:duration="200" />
    <item
        android:drawable="@drawable/sp7"
        android:duration="200" />
    <item
        android:drawable="@drawable/sp8"
        android:duration="200" />
    <item
        android:drawable="@drawable/sp9"
        android:duration="200" />
</animation-list>
```
需要注意的是，根节点必须为 `<animation-list>`，`oneshot`属性代表是否只播放一次，true 为一次，false 为循环播放。`duration`属性表示此张图片滞留的时间，然后注意从上到下依次引用图片即可。

接着给一个 ImageView 设置这个动画：

```
    <ImageView
        android:id="@+id/iv_frame"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:background="@drawable/bear_anim"
        android:layout_centerInParent="true" />
```
最后一步当然是让动画跑起来了，需要用到 AnimationDrawable 对象：
```
		AnimationDrawable animationDrawable = (AnimationDrawable) ivFrame.getBackground();
        animationDrawable.start();
```
这样一个帧动画就展现在我们面前了，想让它停下来也很简单：
```
        if(animationDrawable.isRunning()){
            animationDrawable.stop();
        }
```
在此补充下，bear_anim 同样可以设置给 src 属性，然后调用 `getDrawble().start()` 来播放动画，不过不推荐，具体原因自行查找下 `src` 和 `background` 属性的区别。
自然我们也可以用纯代码的方式实现，不过在此真的不推荐，显然 xml 的方式更省力，并且维护起来更方便。

补间动画
--
tween 动画也叫作补间动画，它可以在一定的时间内使 View 完成四种基本的动画，即**平移、缩放、透明度、旋转**，也可以将它们组合到一起播放出来。这里先提一下未来会研究的 **属性动画**，值得注意的是， 无论是**帧动画**还是**补间动画**，都是把动画效果作用到 View 上，如果一个不是 View 的元素想实现动画，那这两种就无能为力了，只能请 **属性动画** 帮忙了。

并且补间动画仅仅是给 View 增加了动画的“假象”，比如一个按钮从左侧跑到了右侧，你在右侧是无法点击它的，但是这不代表 补间动画就没有用武之地了，当你需要的动画效果无外乎上面那四种动画，并且仅仅是展示的时候，补间动画就再合适不过了。

同样，补间动画的实现依然可以有两种方式，xml 定义或者是纯代码的方式，这里依然是建议使用 xml 的方式。

**AlphaAnimation 透明度**

在 res 文件夹下新建文件夹 anim ，新建文件 alpha_anim：

```
<alpha xmlns:android="http://schemas.android.com/apk/res/android"
    android:duration="200"
    android:fillAfter="true"
    android:fromAlpha="0"
    android:interpolator="@android:anim/linear_interpolator"
    android:repeatCount="-1"
    android:repeatMode="reverse"
    android:shareInterpolator="false"
    android:toAlpha="1">

</alpha>
```
alpha 是透明度动画，分别介绍一下属性，共用属性下文不再重复介绍。

`duration`表示这一次动画持续的时间
`fillAfter`表示动画结束时，是否保持最后一帧的样子
`fillBefore`表示动画结束时，是否保持第一帧的样子
`repeatCount`表示动画循环的次数，默认为 0 次不循环，-1 为无限循环。
`repeatMode`表示是循环的模式，reverse 是从一次动画结束开始，restart 是从动画的开始处循环
`interpolator`是一个插值器资源，它可以控制动画的播放速度
`shareInterpolator`表示是否与 set 中其他动画共享插值器，false为各自使用各自的插值器

上面共有的属性讲完了，下面来说 AlphaAnimation 特有的属性
`fromAlpha`表示动画开始时的透明度
`toAlpha` 表示动画结束时候的透明度
取值为[0.0,1.0]，0代表完全透明，1代表不透明。

代码中借助 AnimationUtils 类来加载调用

```
        Animation alpha = AnimationUtils.loadAnimation(this, R.anim.alpha_anim);
        ivFrame.startAnimation(alpha);
```
以下的三种动画调用同理。

**RotateAnimation 旋转动画**

新建 xml 文件，rotate_anim 

```
<rotate xmlns:android="http://schemas.android.com/apk/res/android"
    android:duration="200"
    android:fromDegrees="0"
    android:pivotX="50%"
    android:pivotY="50%"
    android:toDegrees="360">

</rotate>
```
`fromDegrees` 起始角度 单位度 浮点值
`toDegrees` 结尾角度 单位度 浮点值
`pivotX`旋转中心点的 X 坐标，这个数值有三种表达方式
`pivotY`旋转中心丶的 Y 坐标，这个数值也有三种表达方式

 - 纯数字 例如 20 ，代表相对于自身左边缘或顶边缘 + 20 像素
 - num% 代表 相对于自身左边缘或顶边缘 + 自身宽 的百分之 num
 - num%p 代表相对于自身左边缘或顶边缘 + 父容器 的百分之 num

**ScaleAnimation 缩放动画**

```
<scale xmlns:android="http://schemas.android.com/apk/res/android"
    android:duration="2000"
    android:fromXScale="0.5"
    android:fromYScale="0.5"
    android:pivotX="50%"
    android:pivotY="50%"
    android:toXScale="2.0"
    android:toYScale="2.0">

</scale>
```
`fromXScale`
`fromYScale`
　代表缩放时，X/Y 坐标起始大小，浮点值，0.5代表自身的一半，2.0代表自身的两倍大小。
`toXScale`
`toYScale`
　代表缩放时，X/Y 缩放结束时候大小。
`pivotX`
`pivotY`
缩放的中心坐标，单位与上面 RotateAnimation 介绍的同理

**TranslateAnimation 平移动画**

新建 xml 文件 translate_anim 

```
<translate xmlns:android="http://schemas.android.com/apk/res/android"
    android:duration="200"
    android:fromXDelta="0"
    android:fromYDelta="0"
    android:toXDelta="100%"
    android:toYDelta="100%">

</translate>
```
`fromXDelta`
`fromYDelta`
起始时，X/Y 方向的位置
`toXDelta`
`toYDelta`
终止时，X/Y 方向的位置

这四个属性都支持同样的单位，依然是三种表达方式，浮点数、num% 和 num%p

 - 浮点数 位置为 View 的左边距/上边距 + 此数值 正数为右，负数为左
 - num% 位置为 View 的左边距/上边距 + View宽的百分之num 正数为右，负数为左
 - num%p 位置为 View 的左边距/上边距 + 父容器的百分之num 正数为右，负数为左

如果想将几个动画组合起来使用，可以选择跟布局节点为 set
```
<set xmlns:android="http://schemas.android.com/apk/res/android"
    android:duration="200">

    <translate
        android:fromXDelta="0"
        android:fromYDelta="0"
        android:toXDelta="100%"
        android:toYDelta="100%">
    </translate>

    <rotate
        android:fromDegrees="0"
        android:pivotX="50%"
        android:pivotY="50%"
        android:toDegrees="360">
    </rotate>

</set>
```
代码中组合动画的实现方式：

```
        Animation alpha = AnimationUtils.loadAnimation(this, R.anim.alpha_anim);
        Animation translate = AnimationUtils.loadAnimation(this, R.anim.translate_anim);
        AnimationSet set = new AnimationSet(false);
        set.addAnimation(alpha);
        set.addAnimation(translate);
        ivFrame.startAnimation(set);
```
到这里四种补间动画终于算是介绍完成了，下面在带来一点技巧

```
alpha.setAnimationListener(new Animation.AnimationListener() {
            @Override
            public void onAnimationStart(Animation animation) {
                
            }

            @Override
            public void onAnimationEnd(Animation animation) {

            }

            @Override
            public void onAnimationRepeat(Animation animation) {

            }
        });
```
给动画设置一个侦听，在一些回调中做你的操作。

经过朋友的提醒，事实上 `onAnimationEnd` 并不可靠，有时候动画结束时候并不会调用，详情请看 SO 上的一个提问：

http://stackoverflow.com/questions/5474923/onanimationend-is-not-getting-called-onanimationstart-works-fine

**停止一个补间动画的正确姿势：**

```
public void stopAnimation(View v) {
    v.clearAnimation();
    if (canCancelAnimation()) {
        v.animate().cancel();
    }
    animation.setAnimationListener(null);
	v.setAnimation(null);
}

/**
 * Returns true if the API level supports canceling existing animations via the
 * ViewPropertyAnimator, and false if it does not
 * @return true if the API level supports canceling existing animations via the
 * ViewPropertyAnimator, and false if it does not
 */
public static boolean canCancelAnimation() {
    return Build.VERSION.SDK_INT >= Build.VERSION_CODES.ICE_CREAM_SANDWICH;
}
```
问题详情看这里：

http://stackoverflow.com/questions/4112599/how-to-stop-an-animation-cancel-does-not-work

啊哈，终于完成本文了，希望大家遇到动画的需求不要怂，拿起键盘开写就对了，未来会给大家带来**属性动画**的教程。