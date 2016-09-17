
自己很少做自定义 View ，只有最开始的时候跟着郭神写了一个小 Demo ，后来随着见识的越来越多，特别是在开源社区看到很多优秀的漂亮的控件，都是羡慕的要死，但是拉下来的代码还是看不明白，而且当时因为时间因素，没有深入学习和研究控件和动画方面的知识，而是把更多时间花在了 Android 的异步通信和网络框架这一块。
因为想起暑假实习的时候有个小需求，当时因为忙着主要的业务，一直搁浅没有做，回到学校发现其实不难。索性从这个人生第一个上架的小控件慢慢深入一点，顺带复习 View 的绘制原理。

<!-- more -->

>- 文章来源：itsCoder 的 [WeeklyBolg](https://github.com/itsCoder/weeklyblog) 项目
>- itsCoder主页：[http://itscoder.com/](http://itscoder.com/)
>- 作者：[谢三弟](https://github.com/xcc3641)
>- 审阅者：[Brucezz](https://github.com/brucezz)


### 目录


- [目录](#目录)
- [目标效果](#目标效果)
- [继承 ImageView 开始](#继承-ImageView-开始)
- [工作流程](#工作流程)
  - [onMeasure()](#onMeasure)
  - [onLayout()](#onLayout)
  - [onDraw()](#onDraw)
- [使用](#使用)
- [额外阅读](#额外阅读)


### 目标效果

> 需求：实习公司一个产品，因为很多是临时用户，需要为这些没有自觉设置头像的用户，给予随机头像。生成的规则是根据用户用户名的第一个字符随机匹配颜色集。

从需求中我们可以知道：
- 该控件需要展示图片
- 该控件需要按照规则生成图像
- 一般头像都是圆形

![](http://ww3.sinaimg.cn/large/801b780agw1f7ugm27t4oj205m05agll.jpg)

大致上可以知道是这样的。
开搞！

###  继承 ImageView 开始

我们都知道 Android 自带了很多控件，我们自定义控件的出发点只是官方提供的控件无法满足业务需求的时候。
从我们的需求来看，该控件是图片展示类的，所以我们很自然想到了只需要在系统 ImageView 上进行功能拓展即可，这样就可以满足新的需求又不会失去 ImageView 自带的功能。

```Java
public class CharAvatarView extends ImageView {
    private static final String TAG = CharAvatarView.class.getSimpleName();
    // 颜色画板集
    private static final int[] colors = {
        0xFF1abc9c, 0xFF16a085, 0xFFf1c40f, 0xFFf39c12, 0xFF2ecc71,
        0xFF27ae60, 0xFFe67e22, 0xFFd35400, 0xFF3498db, 0xFF2980b9,
        0xFFe74c3c, 0xFFc0392b, 0xFF9b59b6, 0xFF8e44ad, 0xFFbdc3c7,
        0xFF34495e, 0xFF2c3e50, 0xFF95a5a6, 0xFF7f8c8d, 0xFFec87bf,
        0xFFd870ad, 0xFFf69785, 0xFF9ba37e, 0xFFb49255, 0xFFb49255, 0xFFa94136
    };

    private Paint mPaintBackground;
    private Paint mPaintText;
    private Rect mRect;

    private String text;

    private int charHash;

    public CharAvatarView(Context context) {
        this(context, null);
    }

    public CharAvatarView(Context context, AttributeSet attrs) {
        this(context, attrs, 0);
    }

    public CharAvatarView(Context context, AttributeSet attrs, int defStyleAttr) {
        super(context, attrs, defStyleAttr);
        init();
    }

    private void init() {
        mPaintBackground = new Paint(Paint.ANTI_ALIAS_FLAG);
        mPaintText = new Paint(Paint.ANTI_ALIAS_FLAG);
        mRect = new Rect();
    }
}
```
在这里我做了一些初始化工作，并且在其中的一个构造函数中实例化了 `Paint` 和 `Rect` 。

关于 View 的构造函数的区别：

```Java
public CharAvatarView(Context context) {
    super(context);
}
public CharAvatarView(Context context, AttributeSet attrs) {
    super(context, attrs);
}
public CharAvatarView(Context context, AttributeSet attrs, int defStyleAttr) {
    super(context, attrs, defStyleAttr);
}
```
- 第一种属于程序内实例化时采用，之传入 Context 即可
```Java
CharAvatarView avatarView = new CharAvatarView(this);
```
这样我们的 View 就新建出来了，根据需求添加到布局即可。

- 第二种用于 layout 文件实例化，会把 XML 内的参数通过 AttributeSet 带入到 View 内。

- 第三个主题的 style 信息，也会从 XML 里带入

为了自定义的 View 兼容 Java 和 Xml 两种代码的使用方式，一般推荐这样写构造方法：
```java
  public CharAvatarView(Context context) {
        this(context, null);
    }

    public CharAvatarView(Context context, AttributeSet attrs) {
        this(context, attrs, 0);
    }

    public CharAvatarView(Context context, AttributeSet attrs, int defStyleAttr) {
        super(context, attrs, defStyleAttr);
        init();
    }

    private void init() {
        mPaintBackground = new Paint(Paint.ANTI_ALIAS_FLAG);
        mPaintText = new Paint(Paint.ANTI_ALIAS_FLAG);
        mRect = new Rect();
    }
```

### 工作流程
我们的 View 系统是如何将它绘制到屏幕上的呢？

> View 的绘制流程是从 ViewRoot 的 `performTraversals` 方法开始，它经过 measure 、 layout 和 draw 三个过程才能最终将一个 View 绘制出来，其中 measure 用来测量 View 的宽和高，layout 用来确定 View 在父容器中的放置位置，而 draw 则负责将 View 绘制在屏幕上。针对 performTraversals 的大致流程如图：

![](http://ww4.sinaimg.cn/large/801b780agw1f7um4igvkcj20up0imta0.jpg)

> Measure 过程决定了 View 的宽/高， Measure 完成以后，可以通过 `getMeasuredWidth` 和 `getMeasuredHeight` 方法来获取到 View 测量后的宽/高，在几乎所有的情况下它都等同于 View 最终的宽/高，但是特殊情况除外。
> Layout 过程 决定了 View 的四个顶点的坐标和实际的 View 的宽/高，完成以后，可以通过 `getTop`、`getBottom`、`getLeft`、`getRight` 来拿到 View 的四个顶点的位置，并可以通过 `getWidth` 和 `getHeight` 方法拿到 View 最终的宽/高。
> Draw 过程则决定了 View 的显示，只有 draw 方法完成以后 View 的内容才能呈现在屏幕上。

关于 View 工作流程的深入我们在以后另外开篇进行研究。目前我们已经从宏观了解到了 View 会经历三个过程绘制出来，而且清楚了其中不同方法中的用途。接下来我们看看 CharAvatarView 在这三个流程中分别做了什么。

#### onMeasure()

```Java
@Override
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
    super.onMeasure(widthMeasureSpec, widthMeasureSpec); // 宽高相同
}
```
让宽高相同，我在这里是只直接传入宽度进行测量。
这样会得到一个正方形的 View。



#### onLayout()
```Java
@Override
protected void onLayout(boolean changed, int left, int top, int right, int bottom) {
    super.onLayout(changed, left, top, right, bottom);
}
```
我在这里什么也没有做，因为需求里对 View 的位置没有什么需要特殊的处理。

#### onDraw()

大部分自定义控件，最核心的代码就是在 `onDraw()` 里了。

```Java
    protected void onDraw(Canvas canvas) {
        super.onDraw(canvas);
        if (null != text) {
            int color = colors[charHash % colors.length];
            // 画圆
            mPaintBackground.setColor(color);
            canvas.drawCircle(getWidth() / 2, getWidth() / 2, getWidth() / 2, mPaintBackground);
            // 写字
            mPaintText.setColor(Color.WHITE);
            mPaintText.setTextSize(getWidth() / 2);
            mPaintText.setStrokeWidth(3);
            mPaintText.getTextBounds(text, 0, 1, mRect);
            // 垂直居中
            Paint.FontMetricsInt fontMetrics = mPaintText.getFontMetricsInt();
            int baseline = (getMeasuredHeight() - fontMetrics.bottom - fontMetrics.top) / 2;
            // 左右居中
            mPaintText.setTextAlign(Paint.Align.CENTER);
            canvas.drawText(text, getWidth() / 2, baseline, mPaintText);
        }
    }
```
1. 首先从颜色数组里根据 hash 取余得到背景颜色
2. 然后画出背景圆
3. 接下来就是写字
4. 最后是对字居中的处理


```Java
    /**
     * @param content 传入字符内容
     * 只会取内容的第一个字符,如果是字母转换成大写
     */
    public void setText(String content) {
        if (content == null) {
            throw new NullPointerException("字符串内容不能为空");
        }
        this.text = String.valueOf(content.toCharArray()[0]);
        this.text = text.toUpperCase();
        charHash = this.text.hashCode();
        // 重绘
        invalidate();
    }
```
这是暴露给外部的方法，我们也是在这里得到要画的字符。

### 使用

在 gradle 依赖里添加:

```java
compile 'com.github.xcc3641:charavatarview:0.1'
```
```XML
<com.hugo.charavatarview.CharAvatarView
    android:layout_width="50dp"
    android:layout_height="50dp"
    android:id="@+id/avatar"/>
```
```Java
CharAvatarView mAvatarView;
mAvatarView = (CharAvatarView) findViewById(R.id.avatar);
mAvatarView.setText("谢三弟");
```

运行：
![](http://ww4.sinaimg.cn/large/801b780agw1f7upxsczbsj20p815egnp.jpg)

人生第一个自定义 View 就完成了。

上传到可以参考司机的这篇文章[码农必知之上传开源库到 jcenter]()，配置好各种参数。以后更新版本就执行一行代码就行啦。

```gradle
./gradlew install // 只需要第一次执行
./gradlew bintrayUpload
```

开源地址：[GitHub 地址](https://github.com/xcc3641/CharAvatarView)


### 额外阅读
- [讲解 Canvas 中的一些重要方法](http://www.cnblogs.com/tianzhijiexian/p/4300988.html)
- [教你步步为营掌握自定义 View](http://www.jianshu.com/p/d507e3514b65)
