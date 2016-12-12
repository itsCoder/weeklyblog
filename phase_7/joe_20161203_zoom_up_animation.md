- 文章来源：itsCoder 的 [WeeklyBolg](https://github.com/itsCoder/weeklyblog) 项目

- itsCoder主页：[http://itscoder.com/](http://itscoder.com/)
- 作者：[Joe](http://extremej.itscoder.com/)
- 审阅者：[Brucezz](http://brucezz.itscoder.com/)

### 前言

上次碰到个需求，要从 Feed 页以 item 放大的动画打开第二个页面，先来看一个效果：

![效果图](http://7xtakx.com1.z0.glb.clouddn.com/zoom_up.gif)

就是这样一个转场动画～有了思路后要实现起来还是不难

### 思路

#### 1.API

一般我在需要实现某个功能的时候，会首先寻找 Android 是否已经提供了相关的 Api ，这样可以避免盲目的造轮子，尤其对于项目来说，时间成本还是很重要的，而且 Android 提供的 Api 使用起来一般也会更简单易于维护。

事实上在 Android 5.0 确实提供了这样的 Api — `ActivityOptions` 以及兼容包 `ActivityOptionsCompat`。

```java
ActivityOptionsCompat.makeCustomAnimation(Context context, int enterResId, int exitResId)
ActivityOptionsCompat.makeScaleUpAnimation(View source,int startX, int startY, int startWidth, int startHeight)
ActivityOptionsCompat.makeThumbnailScaleUpAnimation(View source,Bitmap thumbnail, int startX, int startY)
ActivityOptionsCompat.makeSceneTransitionAnimation(Activity activity, View sharedElement, String sharedElementName)
ActivityOptionsCompat.makeSceneTransitionAnimation(Activity activity,Pair<View, String>… sharedElements)

```

这个类使用非常的简单，总共提供了五种方法来实现不同的动画效果，其中就包括我们需要的这种效果。所以一开始我是选择了用 Api 来实现，但是很快发现了有一些问题：

- `makeScaleUpAnimation`可以实现我们需要的效果，但是在5.0及以下的手机上效果并不好，会闪屏。（看了下快手貌似是用这个方法实现的，测试发现有同样的问题）

- 后面两个效果较好的共享元素的方法在 5.0 以下的手机上无效果

- 虽然提供了兼容包，但是兼容包的作用仅仅是保证在低版本的系统上不会崩溃，但是不会有效果

  由于我希望兼容到更低的版本上去，因此不得不放弃这个方案，如果你不需要兼容的话可以参考这篇文章来实现炫酷的转场动画[你所不知道的Activity转场动画——ActivityOptions](https://www.kancloud.cn/qibin0506/android-md/117682)。

#### 2.实现思路

通过使用 ActivityOptions 可以得出一个思路

- 共享元素 — 两个 `Activity` 间有相同内容的 `View`
- TargetView — 第二个页面的 `View` 来执行动画
- 执行动画的过程中，第二个 `Activity` 背景透明

明确了这三个关键点之后，实现起来就清晰很多，事实上这个思路不仅可以实现 `Activity` 间的动画跳转，一样可以适用于同一个 `Activity` 中的其它放大动画，例如点击头像出现大图，没有必要再开一个新的 `Activity`。

所以我们要做的是封装一个工具类，更加通用的执行动画。

### 封装动画工具类

我们首先不去考虑 `Activity` 具体要怎么执行动画，先来封装一个动画的工具类。 

#### 1.获取共享元素的初始信息

我们封装一个工具类 — `ZoomAnimationUtils`，在这个类中定义一个 `ZoomInfo` model：

```java
public static class ZoomInfo implements Parcelable {
    int screenX;
    int screenY;
    int width;
    int height;
  ... 省略其它内容 ...
｝
```

这个类用于获取动画开始时 `View` 在屏幕上的位置，大小。实现序列化是为了在 `Activity` 之间能通过 `Intent` 传递。然后在 `ZoomAnimationUtils`  中封装一个获取 `ZoomInfo`  方法

```java
public static ZoomAnimationUtils.ZoomInfo getZoomInfo(@NonNull View view) {
    int[] screenLocation = new int[2];
    view.getLocationOnScreen(screenLocation);
    return new ZoomInfo(screenLocation[0],
                        screenLocation[1],
                        view.getWidth(),
                        view.getHeight());
}
```

#### 2.封装动画执行接口

在这个需求中我们至少需要提供两个方法，一个放大，一个缩小。先来写放大的动画，想一想我们需要哪些参数？

- 第一个共享元素的信息 `ZoomInfo` (也就是效果图中列表里小图的信息)
- 第二个共享元素 `TargetView`  ，在这个 `View`  上来执行动画
- Listener ，给调用者提供监听动画执行的回调 

```java
public static void startZoomUpAnim(final ZoomInfo preViewInfo,
                                   final View targetView,
                                   final Animator.AnimatorListener listener) {
    
}
```

首先我们要将 `targetView` 的初始属性设置为 `preViewInfo` ，跟小图一样，因此要在 `onDraw` 之前就把这件事情先做完，系统提供了一个方法 `onPreDraw()` 很好的让我们来做这件事情，在上面的方法中添加下面的代码

```java
targetView.getViewTreeObserver().addOnPreDrawListener(new ViewTreeObserver.OnPreDrawListener() {
    @Override
    public boolean onPreDraw() {
      // 务必记得 remove 掉 
        targetView.getViewTreeObserver().removeOnPreDrawListener(this);
    }
});
```

开始计算开始的状态值，体力活～把逻辑先理清楚再计算，否则会掉入一直调试的怪圈中。

```java
int startWidth = preViewInfo.getWidth();
int startHeight = preViewInfo.getHeight();
int endWidth = targetView.getWidth();
int endHeight = targetView.getHeight();
int[] screenLocation = new int[2];
targetView.getLocationOnScreen(screenLocation);
int endX = screenLocation[0];
int endY = screenLocation[1];
float startScaleX = (float) endWidth / startWidth;
float startScaleY = (float) endHeight / startHeight;
int translationX = preViewInfo.getScreenX() - endX;
int translationY = preViewInfo.getScreenY() - endY;

targetView.setPivotX(0);
targetView.setPivotY(0);
targetView.setTranslationX(translationX);
targetView.setTranslationY(translationY);
targetView.setScaleX(1 / startScaleX);
targetView.setScaleY(1 / startScaleY);
```

上面这步做完后，就得到了一个在屏幕上与小图一样的一个 `View`  啦，现在来执行动画：

```java
targetView.setVisibility(View.VISIBLE);
ViewPropertyAnimator animator = targetView.animate();
animator.setDuration(ANIMATION_DURATION)
        .scaleX(1f)
        .scaleY(1f)
        .translationX(0)
        .translationY(0);
if (listener != null) {
    animator.setListener(listener);
}
animator.start();
```

放大的动画就写完了～接下来我们写缩小的动画，思路与放大一样，但是有个小区别，我们缩小的动画实际也是大的那个 `View`  来执行，这个 `View` 当前的状态就是动画开始的初始状态，因此不需要做初始化：

```java
public static void startZoomDownAnim(ZoomInfo preViewInfo, final View targetView, final Animator.AnimatorListener listener){
    int endWidth = preViewInfo.getWidth();
    int endHeight = preViewInfo.getHeight();
    int startWidth = targetView.getWidth();
    int startHeight = targetView.getHeight();
    int endX = preViewInfo.getScreenX();
    int endY = preViewInfo.getScreenY();
    float endScaleX = (float) endWidth / startWidth;
    float endScaleY = (float) endHeight / startHeight;
    int[] screenLocation = new int[2];
    targetView.getLocationOnScreen(screenLocation);
    int startX = screenLocation[0];
    int startY = screenLocation[1];
    int translationX = endX - startX;
    int translationY = endY - startY;

    targetView.setPivotX(0);
    targetView.setPivotY(0);
    targetView.setVisibility(View.VISIBLE);
    ViewPropertyAnimator animator = targetView.animate();
    animator.setDuration(ANIMATION_DURATION)
            .scaleX(endScaleX)
            .scaleY(endScaleY)
            .translationX(translationX)
            .translationY(translationY);
    if (listener != null) {
        animator.setListener(listener);
    }
  	animator.start();
}
```

### 实现 Activity 转场动画

#### 1.进入动画

上一步我们已经把动画封装好了，现在我们需要实现 `Activity`  的转场动画，还记得最开始的那个三个关键点中有一点，将 `Activity` 背景透明，注意这里的背景是指 `Window` 的背景，因此要在 style.xml 中定义一个透明主题：

```xml
<style name="Translucent" parent="AppTheme">
    <item name="windowNoTitle">true</item>
    <item name="android:windowBackground">@android:color/transparent</item>
    <item name="android:colorBackgroundCacheHint">@null</item>
    <item name="android:windowIsTranslucent">true</item>
    <item name="android:windowAnimationStyle">@null</item>
</style>
```

**着重说下`windowIsTranslucent` 必须为 true，否则即使设置了 background 为透明，也是黑色**

第二件事情要将 `Activity` 本身的转场动画去掉 `overridePendingTransition(0,0)`，这个方法要在调用 `startActivity` 后立即调用，我在 `DetailActivity`  中封装了一个静态方法

```java
public static void startActivity(Activity from, View sharedView, FeedItem item){
    if (item == null) return;
    Intent intent = new Intent(from, DetailActivity.class);
    intent.putExtra(EXTRA_FEED_ITEM, item);
  // 获取到前一个 view 的基本信息
    intent.putExtra(EXTRA_ZOOM_INFO, ZoomAnimationUtils.getZoomInfo(sharedView));
    from.startActivity(intent);
  // 去掉自带的转场动画
    from.overridePendingTransition(0,0);
}
```

这里我们的共享 `View` 是一个 `ImageView` ，使用的图片加载框架是 `Picasso`，当然用什么框架都可以，但是要保证一点：**动画开始执行时，这个共享元素的内容应该加载完成了**

我们在 `onCreate` 中先加载好图片

```java
Picasso.with(this)
        .load(mFeedItem.getImageUrl())
        .into(new Target() {
            @Override
            public void onBitmapLoaded(Bitmap bitmap, Picasso.LoadedFrom from) {
              // 加载图片
                setBitmap(bitmap);
              // 执行进入的动画
                tryEnterAnimation();
            }

            @Override
            public void onBitmapFailed(Drawable errorDrawable) {}

            @Override
            public void onPrepareLoad(Drawable placeHolderDrawable) {}
        });
```

`tryEnterAniamtion()` 中的代码就简单了

```java
private void tryEnterAnimation() {
    ZoomAnimationUtils.startZoomUpAnim(mZoomInfo, mImageView, null);
}
```

到这里，转场动画的进入部分就写完了，但是我们发现个问题～，那就是因为 `Activity` 是透明的，图片没有盖住的部分是能看到上一个 `Activity` 的，为了让动画效果更好，我们再加一个背景渐变的动画。

在 `ZoomAnimationUtils` 中添加一个方法

```java
public static void startBackgroundAlphaAnim(final View targetView,
                                            final ColorDrawable color,
                                            int...values) {
    if (targetView == null) return;
    if (values == null || values.length == 0) {
        values = new int[]{0, 255};
    }
    ObjectAnimator bgAnim = ObjectAnimator
            .ofInt(color, "alpha", values);
    bgAnim.setDuration(ANIMATION_DURATION);
    bgAnim.addUpdateListener(new ValueAnimator.AnimatorUpdateListener() {
        @Override
        public void onAnimationUpdate(ValueAnimator animation) {
            targetView.setBackgroundDrawable(color);
        }
    });
    bgAnim.start();
}
```

然后在 `DetailActivity` 中的 `tryEnterAnimation()`  中给背景加上动画：

```java
private void tryEnterAnimation() {
    ZoomAnimationUtils.startZoomUpAnim(mZoomInfo, mImageView, null);
    ZoomAnimationUtils.startBackgroundAlphaAnim(mBackgroundView,
            new ColorDrawable(getResources().getColor(android.R.color.black)), 0, 255);
}
```

#### 2.退出动画

重写两个方法

```java
@Override
public void onBackPressed() {
    tryExitAnimation();
}

@Override
public void finish() {
    super.finish();
  // 去掉自带的转场动画
    overridePendingTransition(0, 0);
}
```

退出动画

```java
private void tryExitAnimation() {
    ZoomAnimationUtils.startBackgroundAlphaAnim(mBackgroundView,
                                                new ColorDrawable(getResources().getColor(android.R.color.black)),
                                                255,
                                                0);
    ZoomAnimationUtils.startZoomDownAnim(mZoomInfo,
                                         mImageView,
                                         new AnimatorListenerAdapter() {
        @Override
        public void onAnimationEnd(Animator animation) {
            finish();
        }
    });
}
```

### 坑

我们好像顺利的写完了动画，但是 QA 后来给我提了一个 bug，这个 bug 极其的蛋疼，由这个 bug 又引发了其它的坑，而这些坑基本来自于 `Window` 的透明。

- 坑一：透明主题的 `Activity` 如果弹出键盘，并且是 `adjustResize`  模式，在键盘弹出的一瞬间可以看到前一个 `Activity` 
  - 原因：键盘弹起时，`Activity` 重新计算了高度缩短了，而键盘弹起有一个动画，在动画没有执行完毕之前，键盘所占用的空间上没有别的布局只有 `Window` ，而  `Window` 是透明的因此就看到了上一个 `Activity`
  - 尝试的解决方案：在动画执行完毕后，将 `Window` 的背景设置为不透明，但是失败了，见坑二
- 坑二：如果当前的 `Window` 背景带有透明度，在动态改变背景的时候会闪一下屏，如果 `Window` 的初始背景颜色没有透明度，动态改变背景很完美
  - 原因：不详，目测是 Android 的 bug
  - 解决方案：无。。。
- 坑三：也是由解决坑二造成的，我试图通过监听键盘弹起事件来手动调节布局，这样 `Activity` 不需要 resize 也就没有那个 bug，但是发现监听键盘弹起的方法基本都是监听 `Activity` 重新布局后对比高度来判断的，因此在 `adjustNothing` 状态下无效，**值得一提的是，跟三弟交流发现他们钻了一个空子，当 `Activity` 为全屏状态时 `adjustResize` 是不生效的，但是可以监听到键盘弹出，所以相当于是`adjustNothing`的效果**，但这种情况没法在我这个项目使用。

最后的解决方案：最后为了解决这个 bug ，我用了一个很蠢的方法，在这个透明的 `Activity` 的前面那个 `Activity` 上面加上一个前景～（不要跟我说清真。。。逼急了什么都能干得出来）

### 相关资料

- [动画Demo](https://github.com/JoeSteven/ZoomAnimDemo)
- [微次元](https://github.com/qii/weiciyuan) — 实现时参考了这个开源项目
- [你所不知道的Activity转场动画——ActivityOptions](https://www.kancloud.cn/qibin0506/android-md/117682)
- [Android - 监听软键盘状态以及获取软键盘的高度](http://cashow.github.io/android-get-keyboard-height.html)

