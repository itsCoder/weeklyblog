title: 使用 RenderScript 实现毛玻璃模糊效果
date: 2016-10-09 14:45:33
tags: [Android]
toc: true
---

利用 RenderScript 高效的实现毛玻璃模糊效果

<!--more-->

## RenderScript 介绍

在开始之前，先看下 RenderScript 的[官方介绍](https://developer.android.com/guide/topics/renderscript/compute.html)：

> RenderScript is a framework for running computationally intensive tasks at high performance on Android. RenderScript is primarily oriented for use with data-parallel computation, although serial workloads can benefit as well. The RenderScript runtime parallelizes work across processors available on a device, such as multi-core CPUs and GPUs. This allows you to focus on expressing algorithms rather than scheduling work. RenderScript is especially useful for applications performing image processing, computational photography, or computer vision.

大致意思就是说 RenderScript 是 Android 平台上为了计算密集型任务的一中高性能框架，并且RenderScript 尤其对图像的处理特别有用。另外，RenderScript 之所以效率高是因为其底层是 C 实现的。

## 使用 RenderScript Support Library

为了可以兼容到更早的版本，我们直接使用 android.support.v8.renderscript(支持API level 9+)包下的，而不使用android.renderscript(支持API level 11+)

以 Android Studio 为例，打开你的 app 的 build.gradle 文件，在 android 的 defaultConfig 结点添加两句：
```
renderscriptTargetApi 18
renderscriptSupportModeEnabled true
```
> 其中 renderscriptTargetApi 的值官方说的是从 11 到最新的 API Level 都可以

这样我们等下导包就可以导 v8 内的了。

## 模糊背景

### 局部模糊

先上一张我们要实现的效果图：

![](http://ww2.sinaimg.cn/large/df189f43gw1f8m3hvioi5j209c0gcwfm.jpg)

这里可以看到实现的是局部模糊，在图片的正中间有一个 TextView，TextView 的背景部分做了模糊处理。

先大致说下模糊的主要步骤(完全模糊步骤一样)：
- 首先取出 TextView 在 ImageView 正上方处的那一块背景
- 然后对取出的那一块背景做模糊处理
- 最后把模糊处理后的背景再设为 TextView 的背景

这样，就可以达到我们图片中的局部模糊效果，具体的过程在代码中有详细的注释。

下面先贴上布局文件：
``` xml
<?xml version="1.0" encoding="utf-8"?>
<RelativeLayout
    xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:padding="20dp">

    <FrameLayout
        android:layout_width="300dp"
        android:layout_height="300dp"
        android:layout_centerInParent="true">

        <ImageView
            android:id="@+id/image"
            android:layout_width="match_parent"
            android:layout_height="match_parent"
            android:scaleType="centerCrop"
            android:src="@drawable/img"/>

        <TextView
            android:id="@+id/text"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:layout_gravity="center"
            android:text="Melody"
            android:textColor="@android:color/white"
            android:textSize="45sp"/>

    </FrameLayout>

</RelativeLayout>
```

再贴上java代码：

``` java
public class MainActivity extends Activity implements Runnable {

    private static final String TAG = "MainActivity";

    private ImageView mImage;
    private TextView mText;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        mImage = (ImageView) findViewById(R.id.image);
        mText = (TextView) findViewById(R.id.text);
        // onCreate()内无法到ImageView的背景，所以需要 post 到消息队列内稍后执行
        mImage.post(this);
    }

    @Override
    public void run() {
        blur(getImageViewBitmap(mImage), mText);
    }

    /**
     * 取出一个imageView的bitmap背景
     */
    public Bitmap getImageViewBitmap(ImageView imageView) {
        imageView.setDrawingCacheEnabled(true);
        // 取出ImageView的Bitmap
        Bitmap bitmap = imageView.getDrawingCache();
        // 拷贝一份bitmap用作模糊
        Bitmap bitmapCopy = bitmap.copy(bitmap.getConfig(), true);
        imageView.setDrawingCacheEnabled(false);
        return bitmapCopy;
    }

    /**
     * 模糊的具体实现
     *
     * @param inputBitmap 要模糊的 bitmap
     * @param targetView  需要被模糊背景的 View
     */
    public void blur(Bitmap inputBitmap, View targetView) {
        // 创建一个和目标View(需要背景被模糊的View)宽高一样的控的 outputBitmap
        Bitmap outputBitmap = Bitmap.createBitmap((int) (targetView.getMeasuredWidth()),
                (int) (targetView.getMeasuredHeight()), Bitmap.Config.ARGB_8888);
        // 将 outputBitmap 关联在 canvas 上
        Canvas canvas = new Canvas(outputBitmap);
        // 画布移动到目标 View 在父布局中的位置
        canvas.translate(-targetView.getLeft(), -targetView.getTop());
        Paint paint = new Paint();
        paint.setFlags(Paint.FILTER_BITMAP_FLAG);
        // 将要模糊的 inputBitmap 绘制到 outputBitmap 上
        // 因为刚才做了 translate 操作，这样就可以裁剪到目标 View 在父布局内的那一块背景到 outputBitmap 上
        canvas.drawBitmap(inputBitmap, 0, 0, paint);

        // ----接下来做模糊 outputBitmap 处理操作----

        // 创建 RenderScript
        RenderScript rs = RenderScript.create(this);
        Allocation input = Allocation.createFromBitmap(rs, outputBitmap);
        Allocation output = Allocation.createTyped(rs, input.getType());
        // 使用 ScriptIntrinsicBlur 类来模糊图片
        ScriptIntrinsicBlur blur = ScriptIntrinsicBlur.create(
                rs, Element.U8_4(rs));
        // 设置模糊半径 ( 取值范围为( 0.0f , 25f ] )
        blur.setRadius(25f);
        blur.setInput(input);
        // 模糊计算
        blur.forEach(output);
        // 模糊 outputBitmap
        output.copyTo(outputBitmap);
        // 将模糊后的 outputBitmap 设为目标 View 的背景
        targetView.setBackground(new BitmapDrawable(getResources(), outputBitmap));
        rs.destroy()；
    }

}
```
导的是 v8 的包：
``` java
import android.support.v8.renderscript.Allocation;
import android.support.v8.renderscript.Element;
import android.support.v8.renderscript.RenderScript;
import android.support.v8.renderscript.ScriptIntrinsicBlur;
```

### 完全模糊

上面是局部模糊，然后我们改变一下 TextView 的宽高铺满父布局，使其和 ImageView 大小一样来实现完全模糊效果：
``` xml
...
<TextView
    android:id="@+id/text"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:gravity="center"
    android:text="Melody"
    android:textColor="@android:color/white"
    android:textSize="45sp"/>
...
```

java代码部分不需要改变，下面再看效果图：

![](http://ww3.sinaimg.cn/large/df189f43gw1f8m432a1d2j20970g50tf.jpg)

### 效率如何？

为了查看操作耗时，我使用 Log 在 blur() 方法的开头和结束地方分别计算了时间，然后查看时间差：
```
10-09 17:04:23.664 23665-23665/com.melodyxxx.blurdemo2 E/MainActivity: spend: 120ms
```
可以看到居然花了 120ms，显然效率不够高，有没有优化的方法？(测试机型为 魅族 PRO 6)


## 效率优化

上面的是直接将原 ImageView 的 bitmap 直接模糊处理，效率不够高，所以我们可以先将原图片进行压缩处理，然后在进行模糊，下面为关键代码，scaleFactor 为压缩比例大小，例如 scaleFactor 为 2，代表先将原图压缩为原来的 1/2，然后进行模糊，效率会高很多。
``` java
...
Bitmap outputBitmap = Bitmap.createBitmap((int) (mTargetView.getMeasuredWidth() / scaleFactor),
        (int) (mTargetView.getMeasuredHeight() / scaleFactor), Bitmap.Config.ARGB_8888);
Canvas canvas = new Canvas(outputBitmap);
canvas.translate(-mTargetView.getLeft() / scaleFactor, -mTargetView.getTop() / scaleFactor);
canvas.scale(1 / scaleFactor, 1 / scaleFactor);
Paint paint = new Paint();
paint.setFlags(Paint.FILTER_BITMAP_FLAG);
canvas.drawBitmap(mInputBitmap, 0, 0, paint);
...
```

具体可以看我的 Demo ：[BlurDemo](https://github.com/melodyxxx/BlurDemo)

下面是 Demo 效果图：

![](http://ww1.sinaimg.cn/large/df189f43gw1f8m5natyw0g209c0gb7wl.gif)

结果：
压缩比例为 2 时：耗时 19 ms
压缩比例为 8 时：耗时 2 ms

根据压缩比例配合不同的模糊半径可以达到不同模糊效果。

再来看 Demo 效果图中拖动 SeekBar 可以动态的实现模糊效果，首先想到的方法是每次拖动然后在里脊模糊一次？这样的效率肯定不行，还会造成卡顿，我这里的方法是先将图片最大化模糊一次设给上方 ImageView 的背景即可，然后 SeekBar 拖动时，只需要改变最上方或者下方图片的透明度就可以达到上面的效果。

Demo apk 下载：http://fir.im/snmb

## 总结

除了 RenderScript 以外还有一些其他的方法也可以实现高斯模糊，例如 FastBlur 等，只要在模糊的时候将不要将原图直接模糊处理，可采取先缩放然后再模糊，这样可以大大提高模糊的速度。

> 参考资料：
> https://developer.android.com/guide/topics/renderscript/compute.html
> http://mobile.51cto.com/aprogram-434699.htm

