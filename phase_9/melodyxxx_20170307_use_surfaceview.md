title: 使用 SurfaceView 实现一个下雨的天气效果
date: 2017-03-07 16:53:55
tags: [Android]
toc: true
---

介绍 SurfaceView 和 View 的区别，以及一些需要使用到 SurfaceView 的场景。

<!--more-->

>- 文章来源：itsCoder 的 [WeeklyBolg](https://github.com/itsCoder/weeklyblog) 项目
>- itsCoder主页：[http://itscoder.com/](http://itscoder.com/)
>- 作者：[Melodyxxx](http://melodyxxx.com/)
>- 审阅者：[暂无]()





文章之前，先上一张本文要最终实现的效果图:

<div align = center>
<img src="http://ww2.sinaimg.cn/large/0060lm7Tgy1fdee9hkoleg30a10he7mn.gif" width = "60%" />
</div>

先分析一下雨滴的实现：
- 每个雨滴其实就是一条线，通过 `canvas.drawLine()` 绘制
- 线（雨滴）的长度、宽度、下落速度、透明度以及位置都是在一定范围内随机生成
- 每 draw 一次然后改变雨滴的位置然后重绘即可实现雨滴的下落效果


分析完了，那么可以直接写一个类直接继承 View ，然后重写 `onDraw()` 吗？可以看到效果图中的雨滴的下落速度很快，那么意味着每一帧都要调用 `onDraw()` 一次使其重新绘制一次，假如你的 `onDraw()` 方法里面的渲染代码稍微有点费时，而 View 的 `onDraw()` 方法调用是在 UI 线程中，那么绘制出来的效果就不会那么流畅，甚至还会阻塞 UI 线程，所以为了更流畅的效果并且不阻塞 UI 线程，我们这里使用 SurfaceView 来实现。

## 初识 SurfaceView

SurfaceView 直接继承自 View，View 必须在 UI 线程中绘制，而 SurfaceView 不同于 View，它可以在非 UI 线程中绘制并显示在界面上，这意味着你可以自己新开一个线程，然后把绘制渲染的代码放在该线程中。

Surface 是 Z 轴排序的，SurfaceView 的 Z 轴位置小于它的宿主 Window，代表它总是在自己所在 Window 的后面，既然在后面，那么是怎么显示的呢？SurfaceView 在其 Window 中打出一个“孔”（其实就是在其宿主 Window 上设置了一块透明区域来使其能够显示），意味着他的兄弟节点的 View 会覆盖它，例如你可以在 SurfaceView 上方放置按钮，文本等控件。

要想访问下面的 Surface ，可以通过 Android 提供给我们的 SurfaceHolder 接口。可以调用 SurfaceView 的 getHolder() 来获取。

SurfaceView 是有生命周期的，我们必须在它生命周期期间进行执行绘制代码，所以我们需要监听 SurfaceView 的状态（例如创建以及销毁），这里 Android 为我们提供了 SurfaceHolder.Callback 这个接口来可以让我们方便的监听 SurfaceView 的状态。

那么下面看下 SurfaceHolder.Callback 接口
``` java
    public interface Callback {

		// SurfaceView 创建时调用(SurfaceView的窗口可见时)
        public void surfaceCreated(SurfaceHolder holder);

		// SurfaceView 改变时调用	
        public void surfaceChanged(SurfaceHolder holder, int format, int width,
                int height);

		// SurfaceView 销毁时调用(SurfaceView的窗口不可见时)
        public void surfaceDestroyed(SurfaceHolder holder);

    }
```

**我们的绘制代码需要在 surfaceCreated 和 surfaceDestroyed 之间执行，否则无效，SurfaceHolder.Callback的回调方法是执行在 UI 线程中的，绘制线程需要我们自己手动创建。**

具体可看官方文档：[https://developer.android.google.cn/reference/android/view/SurfaceView.html](https://developer.android.google.cn/reference/android/view/SurfaceView.html)

## View 和 SurfaceView 的使用场景
- View 适合那些与用户交互并且渲染时间不是很长的控件，因为 View 的绘制和用户交互都处在 UI 线程中。
- SurfaceView 适合迅速的更新界面或者渲染时间比较长以至于影响到用户体验的场景。

## 使用 SurfaceView（实现）

这里我们和自定义 View 类似，写一个类 DynamicWeatherView 继承自 SurfaceView，然后为了监听 SurfaceView 的状态，所以我们还需要实现 SurfaceHolder.Callback 接口来监听 SurfaceView 的状态，接口的回调具体时机上面也已经介绍过了。

``` java
public class DynamicWeatherView extends SurfaceView implements SurfaceHolder.Callback{

    public DynamicWeatherView(Context context) {
        this(context, null);
    }

    public DynamicWeatherView(Context context, AttributeSet attrs) {
        this(context, attrs, 0);
    }

    public DynamicWeatherView(Context context, AttributeSet attrs, int defStyleAttr) {
        super(context, attrs, defStyleAttr);
    }

    // SurfaceView 创建时调用(可见)
    @Override
    public void surfaceCreated(SurfaceHolder holder) {

    }

    // SurfaceView 销毁时调用(不可见)
    @Override
    public void surfaceDestroyed(SurfaceHolder holder) {

    }

    @Override
    public void surfaceChanged(SurfaceHolder holder, int format, int width, int height) {

    }

}
```

上面也提到了，控制 Surface 我们需要 SurfaceHolder 对象，调用 SurfaceView 的 getHolder() 即可获得，然后为这个 SurfaceHolder 添加一个 SurfaceHolder.Callback 回调，这里就是 DynamicWeatherView 当前对象

``` java
private SurfaceHolder mHolder;
```
``` java
mHolder = getHolder();
mHolder.addCallback(this);
mHolder.setFormat(PixelFormat.TRANSPARENT);
```

然后实现我们的绘制线程：

``` java
private class DrawThread extends Thread {

	// 用来停止线程的标记
    private boolean isRunning = false;

    public void setRunning(boolean running) {
        isRunning = running;
    }

    @Override
    public void run() {
        Canvas canvas;
		// 无限循环绘制
        while (isRunning) {
            if (mType != null && mViewWidth != 0 && mViewHeight != 0) {
                canvas = mHolder.lockCanvas();
                if (canvas != null) {
                    mType.onDraw(canvas);
                    if (isRunning) {
                        mHolder.unlockCanvasAndPost(canvas);
                    } else {
						// 停止线程
                        break;
                    }
                    SystemClock.sleep(1);
                }
            }
        }
    }
}
```


从上面的代码可以看出 SurfaceView 的更新流程具体为：
``` java
// 锁定画布并获得 canvas
canvas = mHolder.lockCanvas();
// 在 canvas 上进行绘制
mType.onDraw(canvas);
// 解除锁定并提交更改
mHolder.unlockCanvasAndPost(canvas);
```

绘制线程代码量不多，因为具体的绘制代码在 `mType.onDraw(canvas)`中，mType 是我们自己定义的一个接口，代表一种天气类型：

```
public interface WeatherType {
    void onDraw(Canvas canvas);

    void onSizeChanged(Context context, int w, int h);
}
```

这样要想实现不同的天气类型，只要实现这个接口重写 `onDraw` 和 `onSizeChanged` 方法即可，这里我们实现的是下雨的效果，所以实现了一个 RainTypeImpl 类：
```
public class RainTypeImpl extends BaseType {

    // 背景
    private Drawable mBackground;
    // 雨滴集合
    private ArrayList<RainHolder> mRains;
    // 画笔
    private Paint mPaint;

    public RainTypeImpl(Context context, DynamicWeatherView dynamicWeatherView) {
        super(context, dynamicWeatherView);
        init();
    }

    private void init() {
        mPaint = new Paint();
        mPaint.setAntiAlias(true);
        mPaint.setColor(Color.WHITE);
        // 这里雨滴的宽度统一为3
        mPaint.setStrokeWidth(3);
        mRains = new ArrayList<>();
    }

    @Override
    public void generate() {
        mBackground = getContext().getResources().getDrawable(R.drawable.rain_sky_night);
        mBackground.setBounds(0, 0, getWidth(), getHeight());
        for (int i = 0; i < 60; i++) {
            RainHolder rain = new RainHolder(
                    getRandom(1, getWidth()),
                    getRandom(1, getHeight()),
                    getRandom(dp2px(9), dp2px(15)),
                    getRandom(dp2px(5), dp2px(9)),
                    getRandom(20, 100)
            );
            mRains.add(rain);
        }
    }

    private RainHolder r;

    @Override
    public void onDraw(Canvas canvas) {
        clearCanvas(canvas);
        // 画背景
        mBackground.draw(canvas);
        // 画出集合中的雨点
        for (int i = 0; i < mRains.size(); i++) {
            r = mRains.get(i);
            mPaint.setAlpha(r.a);
            canvas.drawLine(r.x, r.y, r.x, r.y + r.l, mPaint);
        }
        // 将集合中的点按自己的速度偏移
        for (int i = 0; i < mRains.size(); i++) {
            r = mRains.get(i);
            r.y += r.s;
            if (r.y > getHeight()) {
                r.y = -r.l;
            }
        }
    }

    private class RainHolder {
        /**
         * 雨点 x 轴坐标
         */
        int x;
        /**
         * 雨点 y 轴坐标
         */
        int y;
        /**
         * 雨点长度
         */
        int l;
        /**
         * 雨点移动速度
         */
        int s;
        /**
         * 雨点透明度
         */
        int a;

        public RainHolder(int x, int y, int l, int s, int a) {
            this.x = x;
            this.y = y;
            this.l = l;
            this.s = s;
            this.a = a;
        }

    }

}
```

代码不难，基本都有注释，RainHolder 对象代表一个雨滴，每绘制一次然后改变雨滴的位置，然后准备下一次绘制，来实现雨滴的移动。

BaseType 类是我们的一个抽象基类，实现了 DynamicWeatherView.WeatherType 接口，内部有一些公共方法，具体可以看 Demo 中的代码。

最后我们的 Activity 代码：
``` java
public class MainActivity extends Activity {

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        DynamicWeatherView mDynamicWeatherView = (DynamicWeatherView) findViewById(R.id.dynamic_weather_view);
        mDynamicWeatherView.setType(new RainTypeImpl(this, mDynamicWeatherView));
    }
}
```

今后要想实现不同的天气类型，只需要继承 BaseType 类重写相关方法即可。

## Demo 下载
- **[https://github.com/melodyxxx/SurfaceViewDemo](https://github.com/melodyxxx/SurfaceViewDemo)**

## 最后
- 感谢 [Mixiaoxiao](https://github.com/Mixiaoxiao "Mixiaoxiao") 在实现过程中提供的帮助

> 本文参考资料
> - [https://developer.android.google.cn/reference/android/view/SurfaceView.html](https://developer.android.google.cn/reference/android/view/SurfaceView.html)
> - [http://blog.csdn.net/luoshengyang/article/details/8661317](http://blog.csdn.net/luoshengyang/article/details/8661317)
> - [http://www.cnblogs.com/xuling/archive/2011/06/06/android.html](http://www.cnblogs.com/xuling/archive/2011/06/06/android.html)




