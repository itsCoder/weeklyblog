---
 title： Android View 动画、帧动画和属性动画学习笔记 
 date:  2016-12-25 00:00：00
 categories:  Android 动画 Android开发艺术探索
 tags:  Android 动画 Android开发艺术探索
---

>-文章来源：itsCoder 的  [WeeklyBolg](https://github.com/itsCoder/weeklyblog)  项目
>- itsCoder主页：[http://itscoder.com/](http://itscoder.com/)
>- 作者：[yongyu0102](https://github.com/yongyu0102)
>- 审阅者：[Melo](https://github.com/itsMelo)

# 一、概要

以下内容来自 Android 开发艺术探索第七章 Android 动画深入分析学习笔记。

Android 的动画可以分为三种 ： View 动画、帧动画和属性动画。

View 动画：通过对场景里的对象不断做图像变化（平移、缩放、旋转、透明度）从而产生动画效果，它是一种渐近式动画，并且 View 动画支持自定义。

帧动画： 通过顺序播放一系列图像从而产生动画效果，可以简单理解为图片切换动画，如果图片过多过大就会导致 OOM ，其实帧动画也属于 View 动画的一种，只不过它和平移、旋转等常见的 View 动画在表现形式上略有不同而已。

属性动画：通过动态地改变对象的属性从而达到动画效果，属性动画为 API 11 的新特性，在低版本中无法直接使用属性动画，但是可以使用动画兼容库 nineoldandroids 来使用它。

# 二、View 动画

## 2.1 View 动画的种类

View 动画四种变换效果对应着 Animation 的四个子类：TranslateAnimation 、ScaleAnimation 、RotateAnimation 和 AlphaAnimation ，这四种动画既可以通过 XML 来定义，也可以通过代码来动态创建，建议采用 XML 来定义 View 动画，这样可读性更好。

![view_animation](https://github.com/yongyu0102/WeeklyBlogImages/blob/master/phase8/view_animation.png?raw=true)

要使用 View 动画，首先要创建动画的 XML 文件，这个文件的路径为： res/anim/filename.xml 。 View  的 XML 动画描述文件是有固定语法的，如下所示：

```xml
<?xml version="1.0" encoding="utf-8"?>
<set xmlns:android="http://schemas.android.com/apk/res/android"
android:duration="300"
  //动画插值器，影响动画的播放速度
android:interpolator="@android:anim/accelerate_interpolator"
  //表示集合中的动画是否和集合共享一个插值器
android:shareInterpolator="true" >
//透明度动画，对应 AlphaAnimation 类，可以改变 View 的透明度
  <alpha
        android:duration="3000"
        android:fromAlpha="0.0"
        android:toAlpha="1.0" />
          //旋转动画，对应着 RotateAnimation ,它可以使 View 具有旋转的动画效果
    <rotate
        android:duration="2000"
        android:fromDegrees="0"
        android:interpolator="@android:anim/accelerate_decelerate_interpolator"
        android:pivotX="50%"
        android:pivotY="50%"
        android:startOffset="3000"
        android:toDegrees="180" />
       <!--通过设置第一个alpha动画播放3s后启动rotate动画实现组合动画，如果不设置startOffset则同时播放
    pivotX:表示旋转时候的相对轴的坐标点，即围绕哪一点进行旋转，默认情况下轴点是 View 中心
    -->
      //平移动画，对应 TranslateAnimation 类，可以使 View 完成垂直或者水平方向的移动效果。
    <translate
        android:fromXDelta="500"
        android:toXDelta="0" />
          //缩放动画，对应 ScaleAnimation 类，可以使 View 具有放大和缩小的动画效果。
    <scale
        android:duration="1000"
        android:fromXScale="0.0"
        android:fromYScale="0.0"
        android:interpolator="@android:anim/accelerate_decelerate_interpolator"
        android:pivotX="50"
        android:pivotY="50"
        android:toXScale="2"
        android:toYScale="2" />
    </set>
```

从上面代码可以看出 View 动画既可以是单个动画，也可以是一些列动画组成。`<set>` 标签表示动画集合，对应 Animationset 类，它可以表示若干个动画，并且它的内部也可以嵌套其他动画集合，它的两个属性含义如下：

**android ：interpopator**

表示动画集合所采用的插值器，插值器影响动画的播放速度，比如实现非匀速动画效果，这个属性可以不指定，默认为 `android:interpolator="@android:anim/accelerate_decelerate_interpolator"`，即加速减速插值器。

**android:shareInterpolator**

表示集合中的动画是否和集合共享一个插值器，如果集合不指定插值器，那么子动画就需要单独指定所需要的插值器或者使用默认值。
**android:fillAfter**

表示动画结束以后， View 是否停留在结束动画的位置，如果为 false ， View 会回到动画开始的位置。
fillAfter是指动画结束时画面停留在最后一帧，这个参数不能在 `</alpha>,</scale>,</translate>,</rotate>` 中设置，这是没有作用的，必须通过以下方式设置：在动画 XML 文件的 `</set>` 节点中设置：在程序 Java 代码中进行设置：`setFillAfter(true) `。

以上为 View 动画的 XML  格式，下面介绍在 Java 代码中去如何使用：

```java
//透明渐变动画
Animation animation = AnimationUtils.loadAnimation(this, R.anim.alpha);
view.startAnimation(animation);
```

除了通过 XML 文件来实现动画效果外，可以直接在 Java 代码中去实现：

```java
//创建一个透明度渐变动画，在 2000ms 内，将 VIew 的透明度由 0.1f 变为 1.0f 
AlphaAnimation alphaAnimation=new AlphaAnimation(0.1f,1.0f);
alphaAnimation.setDuration(2000);
alphaAnimation.setRepeatCount(4);
alphaAnimation.setRepeatMode(Animation.REVERSE);
view.startAnimation(alphaAnimation);
```

另外可以通过 setAnimationListener 给 View 动画添加过程监听，接口如下：

```java
public static interface AnimationListener {
   //监听动画开始
    void onAnimationStart(Animation animation);

    /**
     * <p>Notifies the end of the animation. This callback is not invoked
     * for animations with repeat count set to INFINITE.</p>
     *
     * @param animation The animation which reached its end.
     */
    void onAnimationEnd(Animation animation);

    /**
     * <p>Notifies the repetition of the animation.</p>
     *
     * @param animation The animation which was repeated.
     */
    void onAnimationRepeat(Animation animation);
}
```

本文中所使用 Demo 代码托管在 GitHub 上，地址为  [AnimationDemo](https://github.com/yongyu0102/AnimationDemo) 。

## 2.2 自定义 View 动画

除了系统提供的四种动画外，我们可以根据需求自定义动画，自定义一个新的动画只需要继承 Animation 这个抽象类，然后重写它的 inatialize 和 applyTransformation 这两个方法，在 initialize 方法中做一些初始化工作，在 Transformation 方法中进行矩阵变换即可，很多时候才有 Camera 来简化矩阵的变换过程，其实自定义动画的主要过程就是矩阵变换的过程，矩阵变换是数学上的概念，需要掌握该方面知识方能轻松实现自定义动画，例子可以参考 Android 的 APIDemos 中的一个自定义动画 Rotate3dAnimation ，这是一个可以围绕 Y 轴旋转并同时沿着 Z 轴平移从而实现类似一种 3D 效果的动画。

## 2.3 帧动画

帧动画是顺序播放一组预先定义好的图片，类似于电影播放。不同于 View 动画，系统提供了另一个类 AnimationDrawble 来使用帧动画，使用的时候，需要通过 XML 定义一个 AnimationDrawble ，如下：

```xml
//\res\drawable\frame_animation_list.xml
<?xml version="1.0" encoding="utf-8"?>
<!--
    根标签为 animation-list，其中 oneshot 代表着是否只展示一遍，设置为 false 会不停的循环播放动画
    根标签下，通过 item 标签对动画中的每一个图片进行声明
    android:duration 表示展示所用的该图片的时间长度
 -->
<animation-list xmlns:android="http://schemas.android.com/apk/res/android"
    android:oneshot="false">
    <item
        android:drawable="@drawable/one"
        android:duration="2000"/>
    <item
        android:drawable="@drawable/two"
        android:duration="2000"/>
    <item
        android:drawable="@drawable/three"
        android:duration="2000"/>
    <!--<item-->
        <!--android:drawable="@drawable/four"-->
        <!--android:duration="500"/>-->
    <!--<item-->
        <!--android:drawable="@drawable/five"-->
        <!--android:duration="500"/>-->
    <!--<item-->
        <!--android:drawable="@drawable/six"-->
        <!--android:duration="500"/>-->

</animation-list
```

然后将上述 Drawble 作为 View 的背景并通过 Drawble 来进行播放动画：

```java
  				//将图片设置成背景图片
                 view.setBackgroundResource(R.drawable.frame_animation_list);
                 AnimationDrawable frameAnimation = (AnimationDrawable) image.getBackground();
                frameAnimation.start();
```

帧动画使用比较简单，但是容易引起 OOM ，所以尽量避免使用过多过大的图片。

# 三、 View 动画的特殊使用场景

View 动画除了可以实现的四种基本的动画效果外，还可以在一些特殊的场景下使用，比如在 ViewGroup 中可以控制子元素的出场效果，在 Activity 中可以实现不同 Activity 之间的切换效果。

## 3.1 LayoutAnimation

LayoutAnimation 作用于 ViewGroup ，为 ViewGroup 指定一个动画，这样当它的子元素出场时候就有动画效果了。这种效果常常被用在 ListView 上，这样 ListView 每个 Item  都会以一种动画效果形式出现，LayoutAnimation 也是一个 View 动画，给 ViewGroup 子元素添加动画效果遵守以下几个步骤：

（1） 定义 LayoutAnimation ，如下所示：

```xml
//res/anim/layout_animation.xml
<?xml version="1.0" encoding="utf-8"?>
<layoutAnimation xmlns:android="http://schemas.android.com/apk/res/android"
    android:delay="0.5"
    android:animationOrder="normal"
    android:animation="@anim/zoom_in">
</layoutAnimation>
```
其中几个属性解释如下：

**android:dela**y 表示子元素开始动画的延时时间，取值为子元素入场动画时间 duration 的倍数，比如子元素入场动画时间周期为 300ms ，那么 0.5 表示每个子元素都需要延迟 150ms 才能播放入场动画，即第一个子元素延迟 150ms 开始播放入场动画，第二个子元素延迟 300ms 开始播放入场动画，依次类推进行。

**android:animationOrder**  表示子元素动画的开场顺序，normal(正序)、reverse(倒序)、random(随机)。

**android:animation**  表示为子元素指定具体的动画效果，例如上面代码中 "@anim/zoom_in" 是我们自己定义的一个 XML 动画。

（2）为子元素指定具体的动画效果：

```xml
//\res\anim\zoom_in.xml
<set xmlns:android="http://schemas.android.com/apk/res/android">
    <alpha
        android:duration="1000"
        android:fromAlpha="0.1"
        android:toAlpha="1.0"/>
    <scale
        android:pivotY="50%"
        android:pivotX="50%"
        android:fromYScale="0.1"
        android:toYScale="1.0"
        android:duration="1000"
        android:fromXScale="0.1"
        android:toXScale="1.0"/>

</set>
```

（3） 为 ViewGroup 指定 android:layoutAnimation 属性 ：`android:layoutAnimation="@anim/layout_animation"`对于 ListView 来说，这样它的所有 Item 就具有入场动画效果了，这种方式适合于所有的 ViewGroup 。

```xml
<ListView
    android:layoutAnimation="@anim/layout_animation"
    android:id="@+id/lv_content"
    android:layout_width="wrap_content"
    android:layout_height="wrap_content">
</ListView>
```

除了在 XML 文件中指定 layout Ainmation 属性外，还可以通过 LayoutAnimationController 来实现，具体代码如下：

```java
//用于控制子 view 动画效果
LayoutAnimationController layoutAnimationController= new LayoutAnimationController(AnimationUtils.loadAnimation(this,R.anim.zoom_in));
layoutAnimationController.setOrder(LayoutAnimationController.ORDER_NORMAL);
listView.setLayoutAnimation(layoutAnimationController);
listView.startLayoutAnimation();
```

## 3.2 Activity 的切换效果

Activity 有默认的切换效果，但是这个效果也可以我们自己定义，主要用到的方法是 `overridePenddingTransation（int enterAnim，int exitAnim）`，该方法必须要在 `startActivity(intent)` 和 `finish()`  方法之后调用才会有效，参数含义：

enterAnim: Activity 打开时所需动画的资源 id 。

exitAnim: Acitivty  退出时所需动画的资源 id 。

当启动一个 Activity 的时候，可以按照如下方式添加自定义的切换效果：

```java
//启动Activity带动画
Intent intent=new Intent(MainActivity.this,Main2Activity.class);
startActivity(intent);
overridePendingTransition(R.anim.zoom_in,R.anim.zoom_out);
```

当 Activity 退出时，指定切换效果：

```java
@Override
public void finish() {
    super.finish();
    overridePendingTransition(R.anim.zoom_in,R.anim.zoom_out);
}
```

需要注意的是 `overridePendingTransition(R.anim.zoom_in,R.anim.zoom_out)`这个方法必须位于 startActivity(intent) 或者 finish() 方法后面，否则将不起作用。

Fragment 也可以添加切换动画，通过 FragmentTransation 中的 setCustomAnimations() 方法来实现切换动画，这个动画需要的是 View 动画，不能使用属性动画，因为属性动画也是 API11 才引入的，不兼容。

# 四、属性动画

属性动画是 API 11 引入的新特性，属性动画可以对任何对象做动画，甚至还可以没有对象。

## 4.1 使用属性动画

属性动画可以对任意对象的属性做动画而不仅仅是 View 对象，动画默认时间间隔是 300ms ，默认帧率是 10ms/帧。其可以达到的效果是：在一个时间间隔内完成对象从一个属性值到另一个属性值的改变，因此属性动画几乎无所不能，只要对象有这个属性，它都能实现动画效果，但是属性动画是从 API 11 才开始有，可以使用动画兼容库 nineoldandroids 动画库来兼容以前版本，在 API 11 以前其内部是通过代理 View 动画来实现的，因此在低版本上，它本质上还是 View 动画。nineoldandroids 动画库和原生属性动画 API 使用功能完全一样。比较常见的几个动画类：ValueAnimator、ObjectAnimator 和 AnimatorSet ，其中 ObjectAnimator 继承自 ValueAnimator ，AnimatorSet 是动画集合，可以定义一组动画，使用示例如下：

（1） 改变一个对象 TranslationY 属性，让其沿着 Y 轴平移一段距离

```java
/**
 * 将 View 沿着垂直方向移动 View 高度的距离
 * @param targetView 被移动的 View
 */
private void translateViewByObjectAnimator(View targetView){
    //TranslationY 目标 View 要改变的属性
    //ivShow.getHeight() 要移动的距离
    ObjectAnimator objectAnimator=ObjectAnimator.ofFloat(targetView,"TranslationY",ivShow.getHeight());
    objectAnimator.start();
}
```

（2）改变一个对象的背景色属性，以下代码实现改变一个 View 的背景色：

```java
/**
 * 改变 View 对象的背景色由红色变为蓝色
 * @param targetView
 */
private void changeViewBackGroundColor(View targetView){
    ValueAnimator valueAnimator=ObjectAnimator.ofInt(targetView,"backgroundColor", Color.RED,Color.BLUE);
    valueAnimator.setDuration(3000);
    //设置估值器，该处插入颜色估值器
    valueAnimator.setEvaluator(new ArgbEvaluator());
    //无限循环
    valueAnimator.setRepeatCount(ValueAnimator.INFINITE);
    //翻转模式
    valueAnimator.setRepeatMode(ValueAnimator.REVERSE);
    valueAnimator.start();
}
```

（3）动画集合，5 秒内对 View 旋转、平移、缩放和透明度进行了改变

```java
/**
 * 启动一个动画集合
 * @param targetView
 */
private void startAnimationSet(View targetView){
    AnimatorSet animatorSet=new AnimatorSet();
    animatorSet.playTogether(ObjectAnimator.ofFloat(targetView,"rotationX",0,360),
             //旋转
            ObjectAnimator.ofFloat(targetView,"rotationY",0,360),
            ObjectAnimator.ofFloat(targetView,"rotation",0,-90),
             //平移
            ObjectAnimator.ofFloat(targetView,"translationX",0,90),
            ObjectAnimator.ofFloat(targetView,"translationY",0,90),
             //缩放
            ObjectAnimator.ofFloat(targetView,"scaleX",1,1.5f),
            ObjectAnimator.ofFloat(targetView,"scaleY",1,1.5f),
             //透明度
            ObjectAnimator.ofFloat(targetView,"alpha",1,0.25f,1));
    	    animatorSet.setDuration(3000).start();
}
```

属性动画除了通过代码实现以外，还可以通过 XML 来实现，属性动画需要定义在 res/animator/目录下，示例如下：

```xml
\res\animator\value_animator.xml
<?xml version="1.0" encoding="utf-8"?><!--set 标签对应着 AnimatorSet-->
<set xmlns:android="http://schemas.android.com/apk/res/android"
    android:ordering="together">
    <!--对应着 ObjectAnimator-->
    <objectAnimator
        android:propertyName="x"
        android:repeatCount="infinite"
        android:repeatMode="reverse"
        android:startOffset="10"
        android:valueTo="300"
        android:valueType="floatType" />
    <!--其中propertyName 属性设置为translationX ，valueType 设置为floatType 可以正常启动
       如果 valueType 设置为 intType 将报错,即属性类型必须为 floatType 类型，并且android:propertyName="translationX" 表示移动 300 ，而  android:propertyName="x"表示移动到300 ，是两个不同属性-->
    <!--startOffset 指定延迟多久开始播放动画-->
    <!--valueType 表示指定的 android:propertyName 所指定属性的类型，intType 表示指定属性是整型的，
     如果指定属性为颜色，那么不需要指定 valueType 属性，系统会自动处理
     repeatCount 表示动画循环次数，默认值为0，-1 表示无限循环-->
    <objectAnimator
        android:propertyName="y"
        android:repeatCount="infinite"
        android:repeatMode="reverse"
        android:startOffset="10"
        android:valueTo="300"
        android:valueType="floatType" />
    <!--对应着 ValueAnimator-->
    <!--<animator-->
    <!--android:propertyName="rotation"-->
    <!--android:duration="300"-->
    <!--android:valueFrom="0"-->
    <!--android:valueTo="360"-->
    <!--android:startOffset="10"-->
    <!--android:repeatCount="infinite"-->
    <!--android:repeatMode="reverse"-->
    <!--android:valueType="intType"/>-->
</set>
```

实际开发中建议使用代码实现属性动画。

## 4.2 理解插值器和估值器

**Timeinterpolator** ：时间插值器，是一个接口类，它的作用是根据时间流逝的百分比来计算当前属性改变的百分比，系统自带有 LinearInterpolator (线性时间插值器，匀速动画)、AccelerateDecelerateInterpolater(加速减速插值器，动画两头慢，中间快)和DecelerateInterpolater (减速插值器，动画越来越慢)等，均实现了该接口。

**TypeEvaluator**: 类型估值算法，也叫估值器，是一个接口类，它的作用是根据当前属性改变的百分比来计算改变后的属性值，系统自带有 IntEvaluator(针对整型属性)、FloatAvaluator(浮点型属性)和 ArgbEvaluator(针对 Color 属性)等，均实现了该接口。

如图所示，表示一个匀速动画，采用了线性插值器和整型估值算法，在 40ms 内，View 的 X 属性实现了从 0 到 40 的变化。

 ![aninatorTimeLine](https://github.com/yongyu0102/WeeklyBlogImages/blob/master/phase8/aninatorTimeLine.png?raw=true)

由于动画的默认刷新率为 10ms/帧，所以该动画将分为 5 帧执行，对于第三帧（x=20，t=20ms），当时间 t=20ms 的时候，时间流逝的百分比是 0.5（20/40=0.5)，表示时间流逝一半，对应的 x 值就是由插值器和估值算法来确定。对于线性插值器来说，当时间流逝一半时，那么 x 的变化值也应该是一半，即 x  的改变是 0.5，因为他是线性插值器，是实现匀速动画的。下面看一下源码：

```java
/**
 * An interpolator where the rate of change is constant
 */
@HasNativeInterpolator
public class LinearInterpolator extends BaseInterpolator implements NativeInterpolatorFactory {

    public LinearInterpolator() {
    }

    public LinearInterpolator(Context context, AttributeSet attrs) {
    }
//返回值
    public float getInterpolation(float input) {
        return input;
    }

    /** @hide */
    @Override
    public long createNativeInterpolator() {
        return NativeInterpolatorFactoryHelper.createLinearInterpolator();
    }
}
```

从上面代码可以看出，线性插值器的返回值和输入值一样，因此插值器返回值是 0.5，这意味着 x 的改变值是 0.5，这个时候插值器的工作就完成了。具体 x  变成了什么值，这个需要估值算法来确定，看一下整型估值算法的源码：

```java
public class IntEvaluator implements TypeEvaluator<Integer> {

    /**
     * This function returns the result of linearly interpolating the start and end values
     * @param fraction   变化的比例系数
     * @param startValue 起始值
     * @param endValue   结束值
     */
    public Integer evaluate(float fraction, Integer startValue, Integer endValue) {
        int startInt = startValue;
        return (int)(startInt + fraction * (endValue - startInt));
    }
}
```

上面源码算法中的 evaluate 三个参数分别表示估值系数、起始值和结束值，对应于上面例子中的  0.5，0，40。根据上述算法整型估值器返回的结果是 0+40*0.5=20，这就是（x=20，t=20ms）的由来。

属性动画要求对象的该属性有 set 和 get（可选） 方法，插值器和估值算法除了系统提供的外，我们还可以自己定义，插值器或者估值算法都是一个接口，且内部只有一个方法，我们只需要派生一个类实现该接口即可，然后就可以做出千变万化的动画效果了。具体而言是：自定义插值器需要实现 Interpolator 或者 TimeInterpolator ，自定义估值算法需要实现 TypeEvaluator 。

## 4.3 属性动画的监听器

属性动画提供了监听器用于监听动画的播放过程，主要有如下两个接口：AnimatorUpdateListener 和 AnimatorListener 。AnimatorListener 源码如下：

```JAVA
public static interface AnimatorListener {
    //动画开始
    void onAnimationStart(Animator animation);

  //动画结束
    void onAnimationEnd(Animator animation);
//动画取消
    void onAnimationCancel(Animator animation);

    //动画重播
    void onAnimationRepeat(Animator animation);
}
```

AnimatorListenerAdapter 实现了 AnimatorListener  接口，这样我们可以有选择的实现上面四个方法，不需要每次都去重写。AnimatorUpdateListener 接口源码：

```java
public static interface AnimatorUpdateListener {
   //监听整个动画过程，动画是由许多帧组成，动画每播放一帧，该方法就会调用一次
    void onAnimationUpdate(ValueAnimator animation);

}
```

## 4.4 对任意属性做动画

当我们想给一个 Button 加一个动画，让这个 Button 宽度从当前宽度增加至 500px，直接用 View 动画是无法达到效果的，因为 View 动画只支持四种效果：平移、缩放、旋转、透明度，当我们强行用 X 方向进缩放可以让 Button 宽度放大，实际上只是 Button 宽度被放大了而已，由于只是 X 方向被放大，这个时候 Button 的背景及上面的文字都会被拉伸变形，效果很差，所以不是真正的对 Button 宽度做动画，所以我们用属性动画方法去实现，代码如下：

```java
private void performAnimation(View button){
    ObjectAnimator.ofInt(button,"width",500)
            .setDuration(200)
            .start();
}
```

上面代码运行后发现无效，其实无效就对了，因为这么随便的给定一个属性值去执行属性动画，轻则无效果，重则程序直接炸掉，可怕。

**下面分析属性动画原理：**属性动画要求动画作用的对象提供 get 方法和 set  方法，属性动画根据外界传递该属性的初始值和最终值以动画的效果去多次调用 set 方法，每次传递给 set 方法的值都不一样，确切的来说是随着时间的推移，所传递的值越来越接近最终值。总结一下，我们对 object 对象属性 abc 做动画，如果想要动画生效，要同时满足两个条件：

（1）object 必须要提供 setAbc() 方法，如果动画的时候没有传递初始值，那么还要提供 getAbc() 方法，因为系统要去取 abc 属性的初始值（如果这条不满足，程序直接炸）。

（2）object 的 setAbc() 对属性 abc 所做的改变必须能够通过某种方法反应出来（即最终体现了 UI  的变化），比如会带来 UI 的改变之类（如果这条不满足，动画无效果，但是程序不会炸）。

以上条件缺一不可，我们给 Button 的 width 属性做动画无效果但是没有炸的原因就是 Button 内部提供了 setWidth 和 getWidth 方法，但是这个 setWidth 方法并不是改变 UI 大小的，而是用来设置最大宽度和最小宽度的。对于上面属性动画的两个条件来说，这个例子只满足了条件 1 而未满足条件 2。

**针对上面问题，官方文档给出了 3 种解决方法：**

（1）给你想要执行属性动画的对象加上 get 和 set 方法，如果你有权限的话。

（2）用一个类来包装你的原始对象，间接的为其提供 get 和 set  方法。

（3）采用 ValueAnimator 监听动画整个过程，自己实现属性的改变。

针对以上三种解决方案具体介绍如下：

**（1）给你想要执行属性动画的对象加上 get 和 set 方法，如果你有权限的话。**

这种方案就是如果你有权限的情况下很简单，只需要给想要执行属性动画的对象直接加上 get 和 set 方法就行了，但是很多情况下我们是没有权限这么做的，比如上面例子中给 Button 添加属性动画，我们就没有办法给 Button 添加一个合乎要求的 setWidth 方法，因为这是 Android 内部 SDK 的实现，我们是没有权限的，所以这种方法实用性不是很强。

**（2）用一个类来包装你的原始对象，间接的为其提供 get 和 set  方法。**

这是一个很有用的解决方案，使用起来也很方便，下面通过一个具体的例子来进行介绍：

```java
/**
 * 将 Button 沿着 X 轴方向放大
 * @param button
 */
private void performAnimationByWrapper(View button){
    ViewWrapper viewWrapper=new ViewWrapper(button);
    ObjectAnimator.ofInt(viewWrapper,"width",800)
            .setDuration(5000)
            .start();
}

 private class ViewWrapper {

        private View targetView;

        public ViewWrapper(View targetView) {
            this.targetView = targetView;
        }

        public int getWidth() {
            //注意调用此函数能得到 View 的宽度的前提是， View 的宽度是精准测量模式，即不可以是 wrap_content
            //否则得不到正确的测量值
            return targetView.getLayoutParams().width;
        }

        public void setWidth(int width) {
          //重写设置目标 view 的布局参数，使其改变大小
            targetView.getLayoutParams().width = width;
          //view 大小改变需要调用重新布局
            targetView.requestLayout();
        }
    }
```

以上代码让 Button 在 5s 内宽度变为 500px，为了达到这个效果我们专门提供了 ViewWrapper 类来包装 View

，然后对 ViewWrapper 的属性 width 做属性动画。

**（3）采用 ValueAnimator 监听动画整个过程，自己实现属性的改变。**

ValueAnimator 本身不作用于任何对象，也就是说直接使用它没有任何动画效果（所以系统提供了它的子类 ObjectAnimator 供我们直接使用，作用于对象直接执行动画效果，而 ValueAnimator 只是提供改变一个值的过程，并能监听到整个值的改变过程，我们基于这个过程可以自己去实现动画效果，在这个过程中做想要达到的效果，自己去实现）。它可以对一个值做动画，然后我们可以监听其动画过程，在动画过程中修改我们对象的属性，这样我们自己就实现了对对象做了动画，示例说明如下：

```java
//new 一个整型估值器，用于下面比例值计算使用（可以自己去计算，这里直接使用系统的）
private IntEvaluator intEvaluator = new IntEvaluator();
private void performAnimatorByValue(final View targetView, final int start, final int end) {
    ValueAnimator valueAnimator = ValueAnimator.ofInt(1, 100);
    valueAnimator.addUpdateListener(new ValueAnimator.AnimatorUpdateListener() {
        @Override
        public void onAnimationUpdate(ValueAnimator animation) {
            //获取当前动画进度值
            int currentValue = (int) animation.getAnimatedValue();
            //获取当前进度占整个动画比例
            int fraction = (int) animation.getAnimatedFraction();
            //直接通过估值器根据当前比例计算当前 View 的宽度,然后设置给 View
            targetView.getLayoutParams().width = intEvaluator.evaluate(fraction, start, end);
            targetView.requestLayout();
        }
    });
    valueAnimator.setDuration(5000)
            .start();
}
```

上面代码在 5s 内将一个数从 1 变到 100，然后动画的每一帧会调用 onAnimationUpdate 方法，在这个方法里面我们手动去设置 View 的属性值，去执行动画效果。比如时间过了一半，那么当前值应该是 50 ，比例值是 0.5，假设 Button 起始宽度为 100 ，最终宽度变为 500，那么 Button 当前值应该为 100+（500-100）*0.5=300。

## 4.5 属性动画的工作原理

属性动画要求作用的对象提供该属性的 set 方法，属性动画根据你传递的该属性的初始值和最终值以动画的效果去多次调用 set 方法。每次传递给 set  方法的值都不一样，确切的说是随着时间的推移，所传递的值越来越接近最终值。如果动画的时候没有传递初始值，那么还要提供 get  方法，因为系统要去获取该属性的初始值。对于属性动画来说，其动画过程中所做的就这么多，下面分析一下源码，首先找一个合适的入口，就从我们使用动画的过程作为切入点， `ObjectAnimator.ofInt(viewWrapper, "width", 800).setDuration(5000).start();`

```java
public void start() {
    // See if any of the current active/pending animators need to be canceled
    AnimationHandler handler = sAnimationHandler.get();
    if (handler != null) {
        int numAnims = handler.mAnimations.size();
        for (int i = numAnims - 1; i >= 0; i--) {
            if (handler.mAnimations.get(i) instanceof ObjectAnimator) {
                ObjectAnimator anim = (ObjectAnimator) handler.mAnimations.get(i);
                if (anim.mAutoCancel && hasSameTargetAndProperties(anim)) {
                    anim.cancel();
                }
            }
        }
        numAnims = handler.mPendingAnimations.size();
        for (int i = numAnims - 1; i >= 0; i--) {
            if (handler.mPendingAnimations.get(i) instanceof ObjectAnimator) {
                ObjectAnimator anim = (ObjectAnimator) handler.mPendingAnimations.get(i);
                if (anim.mAutoCancel && hasSameTargetAndProperties(anim)) {
                    anim.cancel();
                }
            }
        }
        numAnims = handler.mDelayedAnims.size();
        for (int i = numAnims - 1; i >= 0; i--) {
            if (handler.mDelayedAnims.get(i) instanceof ObjectAnimator) {
                ObjectAnimator anim = (ObjectAnimator) handler.mDelayedAnims.get(i);
                if (anim.mAutoCancel && hasSameTargetAndProperties(anim)) {
                    anim.cancel();
                }
            }
        }
    }
  //以上代码用于判断如果当前动画、等待动画（Pendding)、延迟动画（Delay)中有和当前动画相同的动画，就给取消掉
 ........
   //调用父类 ValueAnimator 方法
    super.start();
}
```

下面继续查询 ValueAnimator 的 start 方法源码：

```java
private void start(boolean playBackwards) {
  //必须运行在 Looper 线程
    if (Looper.myLooper() == null) {
        throw new AndroidRuntimeException("Animators may only be run on Looper threads");
    }
    mReversing = playBackwards;
    mPlayingBackwards = playBackwards;
    if (playBackwards && mSeekFraction != -1) {
        if (mSeekFraction == 0 && mCurrentIteration == 0) {
            // special case: reversing from seek-to-0 should act as if not seeked at all
            mSeekFraction = 0;
        } else if (mRepeatCount == INFINITE) {
            mSeekFraction = 1 - (mSeekFraction % 1);
        } else {
            mSeekFraction = 1 + mRepeatCount - (mCurrentIteration + mSeekFraction);
        }
        mCurrentIteration = (int) mSeekFraction;
        mSeekFraction = mSeekFraction % 1;
    }
    if (mCurrentIteration > 0 && mRepeatMode == REVERSE &&
            (mCurrentIteration < (mRepeatCount + 1) || mRepeatCount == INFINITE)) {
        // if we were seeked to some other iteration in a reversing animator,
        // figure out the correct direction to start playing based on the iteration
        if (playBackwards) {
            mPlayingBackwards = (mCurrentIteration % 2) == 0;
        } else {
            mPlayingBackwards = (mCurrentIteration % 2) != 0;
        }
    }
    int prevPlayingState = mPlayingState;
    mPlayingState = STOPPED;
    mStarted = true;
    mStartedDelay = false;
    mPaused = false;
    updateScaledDuration(); // in case the scale factor has changed since creation time
    AnimationHandler animationHandler = getOrCreateAnimationHandler();
    animationHandler.mPendingAnimations.add(this);
    if (mStartDelay == 0) {
        // This sets the initial value of the animation, prior to actually starting it running
        if (prevPlayingState != SEEKED) {
            setCurrentPlayTime(0);
        }
        mPlayingState = STOPPED;
        mRunning = true;
        notifyStartListeners();
    }
  //关键调用处
    animationHandler.start();
}
```

上面代码我们不要在意细节，抓主要与我们相关的看，也就是代码最终调用的 `animationHandler.start()`方法，这个 AnimationHandler 并不是 Handler 而是一个 Runnable ，看一下它的源码会发现很快调用了 JNI 层代码，这部分代码我们通过 AS 是看不见的，不过 JNI 层最终还是会调用回来，它的 Run 方法最终会被调用，这部分我们忽略，直接看 ValueAnimator 的 doAnimationFrame 方法：

```java
final boolean doAnimationFrame(long frameTime) {
    if (mPlayingState == STOPPED) {
        mPlayingState = RUNNING;
        if (mSeekFraction < 0) {
            mStartTime = frameTime;
        } else {
            long seekTime = (long) (mDuration * mSeekFraction);
            mStartTime = frameTime - seekTime;
            mSeekFraction = -1;
        }
        mStartTimeCommitted = false; // allow start time to be compensated for jank
    }
    if (mPaused) {
        if (mPauseTime < 0) {
            mPauseTime = frameTime;
        }
        return false;
    } else if (mResumed) {
        mResumed = false;
        if (mPauseTime > 0) {
            // Offset by the duration that the animation was paused
            mStartTime += (frameTime - mPauseTime);
            mStartTimeCommitted = false; // allow start time to be compensated for jank
        }
    }
    // The frame time might be before the start time during the first frame of
    // an animation.  The "current time" must always be on or after the start
    // time to avoid animating frames at negative time intervals.  In practice, this
    // is very rare and only happens when seeking backwards.
    final long currentTime = Math.max(frameTime, mStartTime);
  //关键代码处
    return animationFrame(currentTime);
}
```

上面代码末尾处调用了 animationFrame 方法，animationFrame  方法内部调用了 animateValue 方法，接着看 animateValue 源码：

```java
void animateValue(float fraction) {
    fraction = mInterpolator.getInterpolation(fraction);
    mCurrentFraction = fraction;
    int numValues = mValues.length;
    for (int i = 0; i < numValues; ++i) {
      //计算每帧动画对应的属性值，这里找到我们在执行动画过程中
      //给对象属性设定值的数据来源是在这里计算获取的
        mValues[i].calculateValue(fraction);
    }
    if (mUpdateListeners != null) {
        int numListeners = mUpdateListeners.size();
        for (int i = 0; i < numListeners; ++i) {
            mUpdateListeners.get(i).onAnimationUpdate(this);
        }
    }
}
```

上面代码中 calculateValue 计算出每帧动画对应的属性值，那么哪里调用属性的 set 和 get 方法去设定和读取属性值呢，这个是我们要寻找的关键。

属性动画在初始化的时候，如果初始值没有提供，那么会调用 get 方法去读取初始值，一起看 PropertyValuesHolder 的 setupValue 方法：

```java
private void setupValue(Object target, Keyframe kf) {
    if (mProperty != null) {
        Object value = convertBack(mProperty.get(target));
        kf.setValue(value);
    }
    try {
        if (mGetter == null) {
          //反射方法调用 get 方法
            Class targetClass = target.getClass();
            setupGetter(targetClass);
            if (mGetter == null) {
                // Already logged the error - just return to avoid NPE
                return;
            }
        }
        Object value = convertBack(mGetter.invoke(target));
        kf.setValue(value);
    } catch (InvocationTargetException e) {
        Log.e("PropertyValuesHolder", e.toString());
    } catch (IllegalAccessException e) {
        Log.e("PropertyValuesHolder", e.toString());
    }
} 
```

通过上代码会发现 get 方法是通过反射机制调用，**这里我们找到了 get 方法调用处。**

当动画下一帧到来时候，PropertyValuesHolder  的 setAnimatedValue 方法会将新的属性值设置给对象，调用其 set 方法，其中 set 方法也是通过反射来调用：

```java
void setAnimatedValue(Object target) {
    if (mProperty != null) {
        mProperty.set(target, getAnimatedValue());
    }
    if (mSetter != null) {
        try {
            mTmpValueArray[0] = getAnimatedValue();
          //反射调用 set 方法
            mSetter.invoke(target, mTmpValueArray);
        } catch (InvocationTargetException e) {
            Log.e("PropertyValuesHolder", e.toString());
        } catch (IllegalAccessException e) {
            Log.e("PropertyValuesHolder", e.toString());
        }
    }
}
```

## 4.6 使用属性动画的注意事项

主要分为以下几类：

**（1）OOM 问题**

在帧动画中如果图片使用过多或者过大，容易产生该问题，应该尽量避免使用帧动画。

**（2）内存泄露问题**

属性动画有一个类是无限播放模式，因此在退出 Activity 时候，应该停止动画播放，否则导致 Activity 无法释放，导致内存泄露，通过验证发现 View  动画并不存在此问题。

**（3）兼容问题**

动画在 3.0 系统以下有兼容性问题，有些场合可能导致无法工作，因此要做好适配。

（4）View 动画问题

View 动画是对 View 的影像做动画，并不是正在改变 View 状态，因此有的时候会出现动画完成以后 View 无法隐藏的现象，即 View.setVisibility(GONE) 失效，这个时候只需要调用 View.clearAnimation 清除动画即可解决该问题。

**（5）不要使用 px**

动画过程中不要使用 px ，应尽量使用 dp，避免不同设备适配问题。

**（6）动画元素的交互**

在 3.0 以前的系统上，不管是 View 动画还是属性动画，新位置无法触发点击事件，而老位置点击事件仍然有效，尽管 View 视觉上已经不在老位置上。

**（7）硬件加速**

在动画过程中建议开启硬件加速，这样增加动画的流畅性。

以上为本次笔记内容，完结！！