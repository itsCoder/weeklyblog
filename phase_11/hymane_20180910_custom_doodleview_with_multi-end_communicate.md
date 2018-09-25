---
layout: post
title: Android 实现实时多端白板通信
summary: 本篇主要介绍最近项目中实现的一个多端白板绘制的功能，适用于多终端实时绘制路径，并能互相同步数据，实现一个多端同步白板。主要涉及到自定义 View，数据传输的封装，异步数据处理，底层数据同步等。
date: 2018-09-10 22:39:08
categories: Android
tags: [Android, DoodleView, CustomView]
featured-img: bottle
---

![first_show](https://wx4.sinaimg.cn/mw690/005X6W83gy1fv4uapl5r8j31kw0vw42r.jpg)

本篇主要介绍最近项目中实现的一个多端白板绘制的功能，适用于多终端实时绘制路径，并能互相同步数据，实现一个多端同步白板。主要涉及到自定义 View，数据传输的封装，异步数据处理，底层数据同步等。

## 目标
首先让我们来理一下这个需求的主要功能点

* 实时绘制
* 修改画笔粗细，画笔颜色，以及橡皮擦，清屏
* 撤销与反撤销
* 多屏绘制以及记录
* 白板绘制区域的缩放移动
* 本地绘制以及操作实时同步到当前频道的其他白板
* 语言通话
* ......

## 分析
整体难点有 4 点，如下：
1. 白板绘制
2. 数据封装
3. 底层数据传输
4. 效率与性能

### 白板绘制
白板绘制即提供一个可绘制的画布，可以在画布画线条，实现橡皮擦等功能。此种功能可以通过自定义 View 来实现，也较为简单。除了自定义 View 来实现外，还有一个方案，那就是自定义 SurfaceView，其实也是自定义 View，只不过 SurfaceView 使用双缓冲机制，可以在异步线程进行 UI 绘制，可以使绘制更流畅。后面会详细介绍两种实现方式。

### 数据封装
考虑到需要进行多个端绘制与操作实时同步，那就避免不了进行对数据结构的确定与封装。保证不同的端在发送数据与接收数据使用的相同的数据协议，这样才能准确的发送和还原指令，这里的指令包含划线、缩放、移动、撤销与反撤销和清屏等。所以必须先确定好包含所有可使用指令的数据包要如何封装。有一下几个问题：

1. 用什么数据结构？
  

    自定义、xml、json、protobuf？
2. 需要哪些字段？
  

    指令类型、指令内容、指令时间？
3. 线条路径如何表示？
  

    一连串点的坐标？
4. 用什么库来编码与解码数据包？
  
  
    xml：DOM、SAX、PULL
    json：Gson、FastJson、Jackson
    protobuf：protobuf

### 底层数据传输
数据包已经封装好了，我们本地每产生一个数据包，我们就要发送这个数据包至当前频道的其他用户，以通知对方最新的消息。那么这个数据传输需要考虑到哪些问题呢？

1. 考虑到实时传输数据，可以使用 socket 长连接，通过服务器做一个中转。
2. 对于线条绘制可以使用 tcp 或 udp，因为绘制对于可靠性要求并不是特别高（产品要求）
3. 对于其他一些指令，则要求可靠性高，如清屏，移动，缩放，丢包影响很大
4. 对于可靠性要求严格的还需要再业务层上确保可靠性，如添加 ACK 机制
5. 处理多个频道（channel）消息互不干扰
6. 处理断线重连、消息发送状态通知等
7. 使用三方成熟解决方案？ok！

通过以上一番分析，发现可以通过一些成熟三方长连接 sdk 来完成这个最复杂的一个任务。可选的方案也很多，比如[网易云信](https://netease.im/)，[Agora 声网](https://www.agora.io/cn/)。目前我选的是声网。接下来我们只需要处理上层逻辑和业务即可，底层数据传输我们全部交给声网来处理。

### 效率与性能
因为我们是需要实时绘制和数据传输，数据传输之前还要进行装包，传输完了还需要解包，再根据接收到的指令进行处理对应的绘制或操作。所以此种需要注意的是在效率和性能上如何做到最优，不能在使用过程中出现明显的卡顿，发热等现象。

那么我们有哪些地方可以注意到的呢？

* 数据封装尽量最简，死磕基本数据类型，能用 byte 不用 int
* 数据结构使用最优，建议使用 protobuf，[数据交换对比](https://tech.meituan.com/serialization_vs_deserialization.html)
* 数据传输尽量进行压缩，如 Gzip，效果会很明显
* UI 绘制使用 SurfaceView 异步线程绘制，甚至只当有绘制数据修改时再去重绘
* 缓存本地数据包，达到一定量，或者按一定时间间隔（如有数据）再统一发送数据包，避免过于频繁发送数据包

## 开始

### 数据封装
1. 首先将指令类型确定下来，指令类型即整个模块可以传递的操作

```java
 public interface ActionStep {
    byte START = 1;//画笔路径开始
    byte MOVE = 2;//画笔路径移动
    byte END = 3;//单条路径绘制结束
    byte REVOKE = 4;//撤销
    byte REDO = 5;//反撤销
    byte CLEAR_SELF = 6;//清屏
    byte CLEAR_ACK = 7;//清屏指令确定
    byte TRANSLATE = 8;//画布平移
    byte SCALE = 9;//画布缩放
    byte PREVIOUS = 10;//上一页画布
    byte NEXT = 11;//下一页画布
}
```
各个指令的作用注释里面已经讲的很清楚了，不再赘述。

2. 封装最小数据包结构


```java
public class Transaction implements Serializable, Cloneable{
	private long timestamp = 0;//数据时间戳
    private byte step = ActionStep.START;
    private float x = 0.0f;
    private float y = 0.0f;
    private byte color = 0;//0,1,2:三种颜色
    private int size = 5;//画笔大小

    public Transaction() {
    }

    public Transaction(long timestamp, byte step, float x, float y) {
        this.timestamp = timestamp;
        this.step = step;
        this.x = x;
        this.y = y;
    }

    public Transaction(long timestamp, byte step, float x, float y, byte size, byte color) {
        this(timestamp, step, x, y);
        this.size = size;
        this.color = color;
    }
    
    private void make(byte step, float x, float y) {
        this.timestamp = System.currentTimeMillis();
        this.step = step;
        this.x = x;
        this.y = y;
    }

    public Transaction makeStartTransaction(float x, float y, int size, byte color) {
        make(ActionStep.START, x, y);
        this.size = size;
        this.color = color;
        return this;
    }

    public Transaction makeMoveTransaction(float x, float y, int size, byte color) {
        make(ActionStep.MOVE, x, y);
        this.size = size;
        this.color = color;
        return this;
    }

    public Transaction makeEndTransaction(float x, float y, int size, byte color) {
        make(ActionStep.END, x, y);
        this.size = size;
        this.color = color;
        return this;
    }

    public Transaction makeRevokeTransaction() {
        make(ActionStep.REVOKE, 0.0f, 0.0f);
        return this;
    }

    public Transaction makeRedoTransaction() {
        make(ActionStep.REDO, 0.0f, 0.0f);
        return this;
    }

    public Transaction makeClearSelfTransaction() {
        make(ActionStep.CLEAR_SELF, 0.0f, 0.0f);
        return this;
    }

    public Transaction makeClearAckTransaction() {
        make(ActionStep.CLEAR_ACK, 0.0f, 0.0f);
        return this;
    }

    public Transaction makeTranslateTransaction(float tx, float ty) {
        make(ActionStep.TRANSLATE, tx, ty);
        return this;
    }

    public Transaction makeScaleTransaction(float sx, float sy) {
        make(ActionStep.SCALE, sx, sy);
        return this;
    }

    public Transaction makePreviousTransaction(int index) {
        make(ActionStep.SCALE, 0.f, 0.f);
        return this;
    }

    public Transaction makeNextTransaction(int index) {
        make(ActionStep.SCALE, 0.f, 0.f);
        return this;
    }

    @JSONField(serialize = false)
    public boolean isPaint() {
        return !isRevoke() && !isClearSelf() && !isClearAck();
    }

    @JSONField(serialize = false)
    public boolean isRevoke() {
        return step == ActionStep.REVOKE;
    }

    @JSONField(serialize = false)
    public boolean isRedo() {
        return step == ActionStep.REDO;
    }

    @JSONField(serialize = false)
    public boolean isNext() {
        return step == ActionStep.NEXT;
    }

    @JSONField(serialize = false)
    public boolean isPrevious() {
        return step == ActionStep.PREVIOUS;
    }

    @JSONField(serialize = false)
    public boolean isClearSelf() {
        return step == ActionStep.CLEAR_SELF;
    }

    @JSONField(serialize = false)
    public boolean isClearAck() {
        return step == ActionStep.CLEAR_ACK;
    }
    
    @Override
    public Transaction clone() {
        Transaction transaction = null;
        try {
            transaction = (Transaction) super.clone();
        } catch (CloneNotSupportedException e) {
            e.printStackTrace();
            transaction = new Transaction();
        }
        return transaction;
    }
    
    public interface ActionStep {
        byte START = 1;
        byte MOVE = 2;
        byte END = 3;
        byte REVOKE = 4;
        byte REDO = 5;
        byte CLEAR_SELF = 6;
        byte CLEAR_ACK = 7;
        byte TRANSLATE = 8;
        byte SCALE = 9;
        byte PREVIOUS = 10;
        byte NEXT = 11;
    }
}
```
> PS：部分代码已省略

可以看到指令类型`ActionStep`作为数据包结构内部类的形式存在，也能体现出软件设计中的`单一职责`原则。需要注意的几个点：
* 时间戳`timestamp`主要用来记录时间发生的时间，暂时用不上，考虑后期可能需要存储整个白板演示的记录，那就需要继续各事件发生的时间点。
* `color`用的是`byte`存储，正常的颜色如白色`0xffffffff`是 32 位，`byte`只能存下 16 位，肯定存不下，故此处'约定大于传输'，先约定好颜色的对应关系，如 0 对应白色。
* `makeXXXTransaction`系列方法是封装单个`Transaction`对象，详情后面会介绍。因为每次创建该对象都要重新赋值各字段，所以这里重写了`clone`方法，避免了每次`new Transaction()`带来的性能损失。`new`和`clone`做了个简单对比，单位纳秒，如下：

```
new_cost:880220
Transaction{timestamp=11, step=1, x=3.0, y=4.0, color=0, size=5}
clone_cost2:33550
Transaction{timestamp=11, step=1, x=3.0, y=4.0, color=0, size=5}
```
* 目前使用的是`json`来包装数据包，使用`fastjson`来处理解析工作，为了避免`fastjson`将一些方法识别成字段，打包进`json`中去，在需要处理的方法上添加注解：`@JSONField(serialize = false)`告诉`fastjson`该方法不做解析。如若不加则在`json`文本中会出现`isPaint`等字段，这些是不需要传输给远端的。

### 白板绘制
白板绘制是一个主要功能，是直接呈现给用户看的，不管底层如何实现，要确保用户在使用过程中不会出现卡顿不流畅的问题。包括 UI 绘制，图形封装，逻辑处理等步骤。

#### DoodleView 白板
白板是一个自定义 View，承载着路径绘制，画布缩放与平移，最重要的一点是手势的识别与路径绘制。以下有两种方案，笔者采取的是第二种方案。

**1、自定义 View**

直接继承 View 类，然后重写`onDraw()`方法，处理各手势，再针对不同手势处理不同事件，为了降低开发难度可以使用系统提供的`GestureDetector`，`ScaleGestureDetector`来简化手势处理。

在需要路径重绘制时候，调用`invalidate()`或`postInvalidate()`来重新绘制界面。

方案总结，此种方案较为简单，不操作白板时不需要重新绘制界面，但是当快速操作白板，比如快速画线时，需要频繁重绘 UI，而且这些重绘工作发生在主线程，也就是 UI 线程，很容易导致不流畅的问题。

**2、自定义 SurfaceView**

直接继承 SurfaceView 类，或者 TextureView 类，重写`surfaceCreated`，`surfaceChanged`，`surfaceDestroyed`，`onTouchEvent`等方法，在`surfaceCreated`方法中开启一个子线程，每个时间间隔重绘白板，这样做的好处有2点：
1. 在子线程绘制，不影响主线程的用户交互，不易出现 ANR。
2. 定时重绘 UI，不需要关系在什么地方要显示调用`invalidate()`来更新 UI，so easy。

重写`onTouchEvent`方法，将时间分发给`GestureDetector`，`ScaleGestureDetector`来处理。然后再将当前指令（指令包括划线，撤销反撤销，橡皮擦，清屏等操作）保存下在一个通道（channel）中，等待下一个时间片到来时，会将通道内可见路径重绘制一次，即可呈现出刚在白板上画的线条了。此处通道是抽取出来绘制路径的封装，方便管理路径，尤其是在处理撤销与反撤销的指令时尤为重要。

最后将路径数据交由`TransactionManager`来处理，将数据打包，找到合适时机发送至频道。`TransactionManager`也是一个较为重要的模块，主要负责数据的装包和解包，以及寻找时机发送数据包。

#### 详细实现
1. 操作接口

    ```java
    /**
     * Author   :hymane
     * Email    :hymanme@163.com
     * Create at 2018/7/20
     * Description: 白板操作统一接口
     */
    public interface IDoodleCallback {
        void undo();
        boolean canUndo();
        void redo();
        boolean canRedo();
        void clear();
        void clean();
        void previous();
        boolean canPrevious();
        void go(int index);
        void next();
        boolean canNext();
        void translate(float dx, float dy);
        void scale(float sx, float sy, float focusX, float focusY);
        void setPaintType(ActionType type);
        void setPaintSize(int size);
        void setPaintColor(@ColorInt int color);
        void end();
    }
    ```

2. 白板 DoodleView

    ```java
    /**
     * Author   :hymane
     * Email    :hymanme@163.com
     * Create at 2018/7/20
     * Description:
     */
    public class DoodleView extends SurfaceView implements SurfaceHolder.Callback, IDoodleCallback, Runnable, TransactionObserver, LifecycleObserver {
        private static final String TAG = DoodleView.class.getSimpleName();
    
        /******************************常量*************************************/
        private static final float MAX_SCALE = 6f;
        private static final float MIN_SCALE = 0.1f;
        private static final int DEFAULT_BACKGROUND_COLOR = Color.WHITE;
    
        /******************************参数*************************************/
    
        private long mDrawDelayTime = 0;
        private int mCanvasBackgroundColor = DEFAULT_BACKGROUND_COLOR;
        
        private long mCurrentDrawDelayTime = mDrawDelayTime;
        protected Handler mPhotoHandler = new Handler();
        protected Easing mEasing = new Cubic();
    
        //SurfaceHolder
        private SurfaceHolder mSurfaceHolder;
        private IDoodleCallback mDoodleCallback;
        private DoodleChannel mDoodleSelfChannel;//本地绘制通道
        private DoodleChannel mDoodleRemoteChannel;//远程回放通道
    ```


        // 数据发送管理器
        private TransactionManager mTransactionManager;
    
        //画笔
        private Paint mClearPaint;
    
        //全局画板手势数据
        private float mTranslateX = 0.0f;//x位移
        private float mTranslateY = 0.0f;//y位移
        private float mCurrentScale = 1.0f;//当前scale
    
        //手势事件
        private ScaleGestureDetector mScaleDetector;
        private GestureDetector mGestureDetector;
        private GestureDetector.OnGestureListener mGestureListener;
        private ScaleGestureDetector.OnScaleGestureListener mScaleListener;


        //是否可滑动
        private boolean mScrollEnabled = true;
        //是否支持双击操作
        private boolean mDoubleTapEnabled = true;
        //是否支持缩放
        private boolean mScaleEnabled = true;
        private int mDoubleTapDirection;
        private boolean onMoving;
        private boolean shouldFling;


        private boolean mIsDrawing = false;
        private byte colorIndex = 0;


        public DoodleView(Context context) {
            this(context, null);
        }
    
        public DoodleView(Context context, AttributeSet attrs) {
            this(context, attrs, 0);
        }
    
        public DoodleView(Context context, AttributeSet attrs, int defStyleAttr) {
            super(context, attrs, defStyleAttr);
            init();
        }
    
        /***
         * 初始化view
         */
        private void init() {
            mSurfaceHolder = this.getHolder();
            mSurfaceHolder.addCallback(this);
            setZOrderOnTop(true);
            setZOrderMediaOverlay(true);
            //设置画布  背景透明
            mSurfaceHolder.setFormat(PixelFormat.TRANSLUCENT);
            //设置一些参数方便后面绘图
            this.setFocusable(true);
            this.setKeepScreenOn(true);
            this.setFocusableInTouchMode(true);
            //其他
            mClearPaint = new Paint();
            mClearPaint.setColor(Color.WHITE);
    
            //手势
            mGestureListener = new GestureListener();
            mScaleListener = new ScaleListener();
            mScaleDetector = new ScaleGestureDetector(getContext(), mScaleListener);
            mGestureDetector = new GestureDetector(getContext(), mGestureListener, null, true);
        }
    
        /***
         * 1. 初始化白板，数据包发送器
         */
        public void initDoodle(LifecycleOwner owner, String channelId) {
            owner.getLifecycle().addObserver(this);
            this.mTransactionManager = new TransactionManager(channelId, getContext());
            this.mTransactionManager.registerTransactionObserver(this);
        }
    
        /***
         * 2. 初始化channel，本地绘制通道
         */
        public void initChannel(DoodleChannel selfChannel, DoodleChannel remoteChannel) {
            if (selfChannel == null) {
                throw (new NullPointerException("doodleChannel can not be null."));
            }
            this.mDoodleSelfChannel = selfChannel;
            this.mTranslateX = selfChannel.getTranslateX();
            this.mTranslateY = selfChannel.getTranslateY();
            this.mCurrentScale = selfChannel.getCurrentScale();
            this.mDoodleRemoteChannel = remoteChannel == null ? new DoodleChannel() : remoteChannel;
        }
    
        @Override
        public void surfaceCreated(SurfaceHolder surfaceHolder) {
            mIsDrawing = true;
            new Thread(this).start();
        }
    
        @Override
        public void surfaceChanged(SurfaceHolder surfaceHolder, int format, int width, int height) {
        }
    
        @Override
        public void surfaceDestroyed(SurfaceHolder surfaceHolder) {
            synchronized (this) {
                mIsDrawing = false;
            }
        }
    
        @Override
        public boolean onTouchEvent(MotionEvent event) {
            if (!isEnabled()) {
                return false;
            }
            mScaleDetector.onTouchEvent(event);
    
            if (!mScaleDetector.isInProgress()) {
                mGestureDetector.onTouchEvent(event);
            }
    
            int action = event.getAction();
            switch (action & MotionEvent.ACTION_MASK) {
                case MotionEvent.ACTION_DOWN:
                    break;
                case MotionEvent.ACTION_UP:
                    return onUp(event);
                case MotionEvent.ACTION_POINTER_UP:
                    break;
                case MotionEvent.ACTION_POINTER_DOWN:
                    break;
    
            }
            return true;
        }
    
        /**
         * IDoodleCallback
         * 退出涂鸦板时调用
         * 清理数据
         */
        @Override
        public void end() {
            if (mTransactionManager != null) {
                mTransactionManager.end();
            }
        }
    
        private float getReviseX(float input) {
            return input / mCurrentScale - mTranslateX;
        }
    
        private float getReviseY(float input) {
            return (input / mCurrentScale - mTranslateY);
        }
    
        /********************************手势确定***********************************/
        protected boolean onDown(MotionEvent e) {
            //...
            return true;
        }
    
        protected boolean onUp(MotionEvent e) {
            //...
            return true;
        }
    
        protected boolean onScroll(MotionEvent e1, MotionEvent e2, float distanceX, float distanceY) {
            //...
            return true;
        }
    
        protected boolean onTranslate(MotionEvent e1, MotionEvent e2, float distanceX, float distanceY) {
            //...
            return true;
        }
    
        protected boolean onFling(MotionEvent e1, MotionEvent e2, float velocityX, float velocityY) {
            //...
            return false;
        }
    
        protected boolean onSingleTapUp(MotionEvent e) {
            log("onSingleTapUp");
            return true;
        }
    
        protected boolean onSingleTapConfirmed(MotionEvent e) {
            //...
            return true;
        }
    
        protected boolean onScale(float sx, float sy, float focusX, float focusY) {
            //...
            return true;
        }
    
        private void log(String msg) {
            Log.d(TAG, "log: " + msg);
        }
    
        /***
         * 子线程绘制
         */
        @Override
        public void run() {
            while (mIsDrawing) {
                synchronized (this) {
                    reDraw();
                }
                if (mCurrentDrawDelayTime != 0) {
                    try {
                        Thread.sleep(mCurrentDrawDelayTime);
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
            }
        }
    
        /***
         * 重回
         */
        protected void reDraw() {
            Canvas canvas = mSurfaceHolder.lockCanvas();
            if (canvas != null) {
                canvas.save();
                canvas.drawColor(mCanvasBackgroundColor);
                canvas.scale(mCurrentScale, mCurrentScale);
                canvas.translate(mTranslateX, mTranslateY);
                //绘制历史action
                for (int i = 0; i <= mDoodleSelfChannel.index(); i++) {
                    if (i >= mDoodleSelfChannel.actions.size()) {
                        break;
                    }
                    mDoodleSelfChannel.actions.get(i).onDraw(canvas);
                }
                //绘制当前action
                if (mDoodleSelfChannel.action != null) {
                    mDoodleSelfChannel.action.onDraw(canvas);
                }
    
                for (int i = 0; i <= mDoodleRemoteChannel.index(); i++) {
                    if (i >= mDoodleRemoteChannel.actions.size()) {
                        break;
                    }
                    mDoodleRemoteChannel.actions.get(i).onDraw(canvas);
                }
    
                if (mDoodleRemoteChannel.action != null) {
                    mDoodleRemoteChannel.action.onDraw(canvas);
                }
                canvas.restore();
                //绘制当前action
                mSurfaceHolder.unlockCanvasAndPost(canvas);
            }
        }
    
        /***
         * 数据包事务处理
         * 已将接收到的远程数据解压并整理成事务单元
         * 将事务单元转至可绘制action
         * @param transactions
         */
        @Override
        public synchronized void onTransaction(List<Transaction> transactions) {
            //...-->onMultiTransactionsDraw
        }
    
        private void onMultiTransactionsDraw(List<Transaction> transactions) {
            //...
        }
    
        /********************************生命周期***********************************/
    
        /********************* Lifecycle ***************************/
        @OnLifecycleEvent(Lifecycle.Event.ON_RESUME)
        public void onResume() {
            log("Lifecycle:onResume");
            mCurrentDrawDelayTime = mDrawDelayTime;
        }
    
        @OnLifecycleEvent(Lifecycle.Event.ON_PAUSE)
        public void onPause() {
            log("Lifecycle:onPause");
            mCurrentDrawDelayTime = Integer.MAX_VALUE;
        }
    
        @OnLifecycleEvent(Lifecycle.Event.ON_DESTROY)
        public void onDestroy() {
            log("Lifecycle:onDestroy");
            mIsDrawing = false;
            end();
        }
    
        /********************************手势处理***********************************/
        /***
         * GestureListener
         * 委托的手势监听
         */
        private class GestureListener extends GestureDetector.SimpleOnGestureListener {
            //...
        }
        /***
         * 缩放
         */
        private class ScaleListener extends ScaleGestureDetector.SimpleOnScaleGestureListener {
            //...
        }
    
        //移动
        public void scrollBy(float distanceX, float distanceY, final double durationMs) {
            //...
        }
    }
    ```

代码有点多，这里就没有全部贴出来，至贴了主要的代码。主要实现前面也已经介绍过，这里有一个`mDrawDelayTime`来控制刷新 UI 的时间间隔，可以调整绘制频率，建议 frame/30ms。

* DoodleChannel 为绘制通道保存绘制的记录，此处有两个通道，一个是远程绘制记录，一个是本地记录，区分开了，然后依次重回这两个通道，此处还有一个 bug，就是两个通道的覆盖绘制问题。
* 周期性的延迟绘制使用的是 Handler 来处理延迟消息
* 手势事件交给了`ScaleGestureDetector`，`GestureDetector`来处理，并重写`OnGestureListener`和`OnScaleGestureListener`监听来处理具体逻辑。
* SurfaceView 在处理背景是会出现一些坑，比如闪屏和黑屏等问题，在初始化的时候上以下设置，可以避免 SurfaceView 黑屏，已经挡住 UI 组件的问题。
```
setZOrderOnTop(true);
setZOrderMediaOverlay(true);
```
* TransactionManager 是数据包管理器，前面也有介绍过，用来处理数据包的装包解包以及发送的工作，后面会详细介绍。
* 橡皮擦用的是白色画笔简单代替，这样在配合两个通道进行重回时会出现之前的绘制路径覆盖当前绘制路径的问题。考虑优化，嘿嘿嘿。

3. 绘制通道

```java
/**
 * Author   :hymane
 * Email    :hymanme@163.com
 * Create at 2018/7/20
 * Description: 白板图形绘制通道，记录每一次绘制
 */
public class DoodleChannel {
    private static final String TAG = DoodleChannel.class.getSimpleName();
    //多线程原子操作
    private AtomicInteger index = new AtomicInteger(-1);
//    public volatile int index = -1;//当前绘制记录id,-1表示为空
    //当前画笔相关
    public int mPaintColor = Color.BLACK;
    //历史本地绘制记录
    public List<Action> actions = new ArrayList<>();
    private Bitmap mImage;
    private ActionType mPaintType = ActionType.Path;
    //当前画笔类型，即action类型

    private float mTranslateX = 0.0f;//x位移
    private float mTranslateY = 0.0f;//y位移
    private float mCurrentScale = 1.0f;//当前scale

    //当前使活动的动作
    public Action action;
    private int mPaintSize = 10;

    /***
     * 添加当前action至绘制记录
     */
    public void addCurrentAction() {
        if (action == null || !action.isStarted()) {
            return;
        }
        clearOut();
        index.incrementAndGet();
//        index++;
        actions.add(action);
        action.setAdded(true);
        Log.d(TAG, "READ_addCurrentAction: " + action.toString());
    }


    /***
     * 添加当前action至绘制记录
     */
    public void addCurrentAction(int id) {
        if (action == null || !action.isStarted()) {
            return;
        }
        clearOut();
        index.incrementAndGet();
//        index++;
        actions.add(id, action);
        Log.d(TAG, "READ_addCurrentAction: " + action.toString());
    }

    /***
     * 获取当前绘制id
     * @return
     */
    public int index() {
        return index.get();
    }

    /***
     * 添加当前action至绘制记录
     */
    public void addAction(Action a) {
        if (action == null) {
            return;
        }
        clearOut();
        index.incrementAndGet();
//        index++;
        actions.add(a);
    }

    /***
     * 添加当前action至绘制记录
     */
    public void addAction(Action a, int id) {
        if (action == null) {
            return;
        }
        clearOut();
        index.incrementAndGet();
//        index++;
        actions.add(id, a);
    }

    private void clearOut() {
        int count = actions.size() - 1;
        for (int i = count; i >= index.get() + 1; i--) {
            actions.remove(i);
        }
    }

    public boolean undo() {
        if (canUndo()) {
            index.decrementAndGet();
//            index--;
            return true;
        }
        return false;
    }

    public boolean redo() {
        if (canRedo()) {
            index.incrementAndGet();
//            index++;
            return true;
        }
        return false;
    }

    //检查index是否正常，可绘制
    public boolean checkIndex() {
        if (index.get() < 0) {
            return false;
        }
        if (index.get() >= actions.size()) {
            index.set(actions.size() - 1);
        }
        return true;
    }

    public boolean canUndo() {
        return index.get() >= 0 && !actions.isEmpty();
    }

    public boolean canRedo() {
        return index.get() < actions.size() - 1;
    }

    public void clear() {
        index.set(-1);
        actions.clear();
    }

    public boolean clear(int offset) {
        if (offset >= actions.size() || offset < -1) {
            return false;
        }
        index.set(offset);
        for (int i = offset; i < actions.size(); i++) {
            actions.remove(i);
        }
        return true;
    }
    //保存之前的画布缩放平移状态，保证在多画布切换时恢复现场
    public void save(float mTranslateX, float mTranslateY, float mCurrentScale) {
        this.mTranslateX = mTranslateX;
        this.mTranslateY = mTranslateY;
        this.mCurrentScale = mCurrentScale;
    }

    public void genAction(float x, float y) {
        action = genPaint(x, y);
    }

    public Action genPaint(float x, float y) {
        switch (mPaintType) {
            case Eraser:
                return new MyEraser(x, y, Color.WHITE, mPaintSize);
            case Image:
                return new MyImage(mImage, x, y, Color.WHITE, mPaintSize);
            case Path:
            default:
                return new MyPath(x, y, mPaintColor, mPaintSize);
        }
    }
}

```
我们已经知道了绘图通道用来保存各种绘制指令，所有绘制路径全部保存在`actions`数组中，并且有一个`index`指向当前最后一个可现实路径的坐标，这里主要用来处理撤销与反撤销工作。撤销与反撤销只需要在`actions`中移动这个`index`就可以了，当然需要严格判断这个`index`是否越界。

那么这个`Action`又是什么呢？是绘制的动作，或者说是绘制的类型。为了提高扩展性，我们日后可能会有更多绘制类型，比如绘制图片，绘制矩形，绘制直线甚至是绘制文本等。这里使用 Action进行派生出更多类型，而且我们将所有绘制工作全部交给绘制类型本身来处理，而不是在 DoodlView 中处理，先把职责分工好。

![](https://wx4.sinaimg.cn/small/005X6W83gy1fv5jmdpasmj309008e0t2.jpg)

这样一想上面的`actions`就很好理解了。

**Action 类**

```java
/**
 * Author   :hymane
 * Email    :hymanme@163.com
 * Create at 2018/7/20
 * Description:
 */
public abstract class Action {
    protected float startX;//图型开始x坐标
    protected float startY;//图型开始y坐标
    protected float endX;//图型结束x坐标
    protected float endY;//图型结束x坐标
    protected int color;//图型颜色
    protected Paint mPaint;//画笔
    protected int size;//画笔宽度
    protected boolean smooth;//是否是平滑的画笔，贝塞尔path
    protected boolean isAdded;//是否已添加

    public Action() {
    }

    public Action(float x, float y) {
        this.startX = x;
        this.startY = y;
    }

    public Action(float x, float y, int color, int size) {
        this.startX = x;
        this.startY = y;
        this.endX = x;
        this.endY = y;
        this.color = color;
        this.size = size;
    }

    public boolean isPoint() {
        return startX == endX && startY == endY;
    }

    public abstract void onStart(float sx, float sy);

    public abstract void onMove(float mx, float my);

    public abstract void onDraw(Canvas canvas);

    /***
     * 清理数据，action不再使用时，做清理工作
     * 尤其是类似bitmap占用内存高的内容
     */
    public void clear() {

    }

    @Override
    public String toString() {
        return "Action{" +
                "startX=" + startX +
                ", startY=" + startY +
                ", endX=" + endX +
                ", endY=" + endY +
                ", size=" + size +
                '}';
    }
}

```
这里可以看到我们有一个`onDraw`的抽象方法，为了就是让子类去根据自身类型来绘制自己。自己对自己才是最了解的，才知道自己要如何绘制。

4. ActionType

```java
/**
 * Author   :hymane
 * Email    :hymanme@163.com
 * Create at 2018/7/24
 * Description:
 */
public enum ActionType {
    UnKnow(0),
    Point(1),
    Path(2),
    SmoothPath(3),
    Image(4),
    Line(5),
    Rect(6),
    Circle(7),
    FilledRect(8),
    FilledCircle(9),

    Eraser(10);

    private int value;

    ActionType(int value) {
        this.value = value;
    }

    public static ActionType typeOfValue(int value) {
        for (ActionType e : values()) {
            if (e.getValue() == value) {
                return e;
            }
        }
        return UnKnow;
    }

    public int getValue() {
        return value;
    }
}
```
ActionType 是一个枚举类型，全局限定了支持的绘制类型。

#### 小小结
到这里白板绘制算告一段落，主要考虑的点是职责分工明确，而不是一味的丢给 DoodleView 来处理，虽然我们的白板就是 DoodleView。

### 数据包管理 -- TransactionManager
我们的绘制流程已经处理完毕，本地发生了绘制指令，需要将其同步到其他端，这里就需要数据包管理器来辅助处理。

**先贴代码**
```java
/**
 * Author   :hymane
 * Email    :hymanme@163.com
 * Create at 2018/7/27
 * Description: 事务数据包发送管理
 */
public class TransactionManager {
    private static final String TAG = "TransactionManager";
    //发送绘制消息的时间周期，每隔 {TIMER_TASK_PERIOD} 毫秒，
    //扫描一次缓存 cache，如果有未发生绘制路径，则发送。建议周期不易设置过小
    private final int TIMER_TASK_PERIOD = 30;
    private int mTimeTaskPeriod = TIMER_TASK_PERIOD;
    private Transaction transaction;
    private String channelId;//频道id
    private Handler handler;
    private List<Transaction> cache = new ArrayList<>(1000);

    private Runnable timerTask = new Runnable() {
        @Override
        public void run() {
//            handler.removeCallbacks(timerTask);
            {
                if (cache.size() > 0) {
                    sendCacheTransaction();
                }
            }
            handler.postDelayed(timerTask, TIMER_TASK_PERIOD);
        }
    };

    public TransactionManager(String channelId, Context context) {
        this.channelId = channelId;
        this.handler = new Handler(context.getMainLooper());
        this.handler.postDelayed(timerTask, TIMER_TASK_PERIOD); // 立即开启定时器
        this.transaction = transaction.clone();
    }

    public TransactionManager(String channelId, Context context, int timePeriod) {
        this.channelId = channelId;
        this.handler = new Handler(context.getMainLooper());
        this.mTimeTaskPeriod = timePeriod;
        this.handler.postDelayed(timerTask, mTimeTaskPeriod); // 立即开启定时器
        this.transaction = transaction.clone();
    }

    public void registerTransactionObserver(TransactionObserver o) {
        TransactionCenter.getInstance().registerObserver(channelId, o);
    }

    public void sendStartTransaction(float x, float y, int size, byte color) {
        pack(transaction.clone().makeStartTransaction(x, y, size, color));
    }

    public void sendMoveTransaction(float x, float y, int size, byte color) {
        pack(transaction.clone().makeMoveTransaction(x, y, size, color));
    }

    public void sendEndTransaction(float x, float y, int size, byte color) {
        pack(transaction.clone().makeEndTransaction(x, y, size, color));
    }

    public void sendTranslateTransaction(float tx, float ty) {
        pack(transaction.clone().makeTranslateTransaction(tx, ty));
    }

    public void sendScaleTransaction(float sx, float sy) {
        pack(transaction.clone().makeScaleTransaction(sx, sy));
    }

    public void sendRevokeTransaction() {
        pack(transaction.clone().makeRevokeTransaction());
    }

    public void sendRedoTransaction() {
        pack(transaction.clone().makeRevokeTransaction());
    }

    public void sendClearTransaction() {
        pack(transaction.clone().makeClearSelfTransaction());
    }

    private void pack(Transaction t) {
        Log.d(TAG, "pack: " + t.toString());
        cache.add(t);
    }

    //每隔一定时间将本地未发送的数据包发送到房间
    private void sendCacheTransaction() {
        TransactionCenter.getInstance().sendToRemote(channelId, this.cache);
        Log.d(TAG, "sendCacheTransaction: size=" + this.cache.size());
        cache.clear();
    }

    /***
     * 结束
     * 清理数据
     */
    public void end() {
        this.handler.removeCallbacks(timerTask);
        TransactionCenter.getInstance().end();
    }

    public int getTimeTaskPeriod() {
        return mTimeTaskPeriod;
    }

    public void setTimeTaskPeriod(int mTimeTaskPeriod) {
        this.mTimeTaskPeriod = mTimeTaskPeriod;
    }
}
```
这里需要注意的点有:
* 我们的数据包最小封装是 Transaction，之前也分析过
* 为了减少请求次数，我们添加了一缓存`cache`，是一个列表，存储尚未发生的数据包
* 数据管理器会每隔指定时间检查一下`cache`是否有待发送数据包，如果有则发，没有就等待。工作交由`timerTask`处理，时间间隔由`TIMER_TASK_PERIOD`确定
* `sendXXXTransaction`系列方法向外提供了向待发送缓存`cache`中添加数据包的入口。这里使用了`transaction.clone()`来优化创建对象的性能开销
* 装包操作，添加的数据包是 Java 对象，我们使用`pack()`方法将待发送的 Transaction 数据包添加到`cache`缓存中
* 待`timerTask`时间片到来时，通过`sendCacheTransaction`方法发送这些待发送数据包至远端。
* 真正发送数据的任务交给了 TransactionCenter（数据包发送中心）来处理，其实实际发送还不是数据包发送中心来处理的，而是`Agora`，后面再讲。
* 将`java`数据包装包成可传输的`json`字符串工作也是交给 TransactionCenter 来处理，由 TransactionManager 处理也行，但是为了更加高效，我们让 TransactionManager 处理单个数据包，TransactionCenter 处理一次发送任务数据包集合，如果每次都装包单个数据包成为`json`字符串，这未免也太低效了
* 如果需要监听远端的数据包，那就需要注册一个监听(观察者)`registerTransactionObserver()`，实际的解析`json`发送数据包任务也是由 TransactionCenter 来处理
* 不再使用时记得清理数据`end()`

### 数据包发送中心 -- TransactionCenter
**先上代码**

```java
/**
 * Author   :hymane
 * Email    :hymanme@163.com
 * Create at 2018/7/27
 * Description: 数据包底层收发处理中心
 */
public class TransactionCenter {
    private final String TAG = "TransactionCenter";

    // sessionId to TransactionObserver
    //可能包含多个session
    private Map<String, TransactionObserver> observers = new HashMap<>(2);

    public static TransactionCenter getInstance() {
        return TransactionCenter.TransactionCenterHolder.instance;
    }

    public void registerObserver(String sessionId, TransactionObserver o) {
        this.observers.put(sessionId, o);
    }

    /**
     * #1 数据发送
     */
    public void sendToRemote(String channelId, List<Transaction> transactions) {
        if (transactions == null || transactions.isEmpty()) {
            return;
        }

        //发送画笔轨迹消息
        String data = pack(transactions);
        Log.d(TAG, "sendToRemote: channelId:" + channelId + " data" + data);
        AgoraUtils.messageChannelSend(channelId, data);
    }

    /***
     * 压包，将本地数据压缩成网络传输数据
     * @param transactions
     * @return
     */
    private String pack(List<Transaction> transactions) {
        return JSON.toJSONString(transactions);
    }

    /***
     * 前：通信收到string字符数据
     * #2 单次数据包接收
     * @param sessionId
     * @param data 收到远端发送过来的 string 字符数据，
     *             包含画板同步数据，为 {@link Transaction} 封装类型
     */
    public void onReceive(String sessionId, String data) {
        if (observers.containsKey(sessionId)) {
            observers.get(sessionId).onTransaction(unpack(data));
        }
    }

    /***
     * 数据解包，将远端传过来的数据解压缩成数据单元
     * @param data 封装数据格式的字符串 see {@link Transaction}
     * @return 数据单元列表
     */
    private List<Transaction> unpack(String data) {
        if (TextUtils.isEmpty(data)) {
            return null;
        }
        return JSON.parseArray(data, Transaction.class);
    }

    /***
     * 清理缓存
     */
    public void end() {
        this.observers.clear();
    }

    /***
     * 单例
     */
    private static class TransactionCenterHolder {
        public static final TransactionCenter instance = new TransactionCenter();
    }

}

```
简单解释：
* `observers`为对通道到来的数据感兴趣的观察者，有数据 TransactionCenter 会通知这些观察者
* TransactionCenter 是一个单例的
* 此处也有一个`pack`方法，方法是将待发送的 cache 数据包**集合**一并装包成`json`字符串
* 最后通过`Agora`向该频道发送字符串数据

```java
//data 即数据集合的 json 字符串
AgoraUtils.messageChannelSend(channelId, data);
```
那么，接收到数据如何处理呢？

```java
    /***
     * 前：通信收到string字符数据
     * #2 单次数据包接收
     * @param sessionId
     * @param data 收到远端发送过来的 string 字符数据，
     *             包含画板同步数据，为 {@link Transaction} 封装类型
     */
    public void onReceive(String sessionId, String data) {
        if (observers.containsKey(sessionId)) {
            observers.get(sessionId).onTransaction(unpack(data));
        }
    }

    /***
     * 数据解包，将远端传过来的数据解压缩成数据单元
     * @param data 封装数据格式的字符串 see {@link Transaction}
     * @return 数据单元列表
     */
    private List<Transaction> unpack(String data) {
        if (TextUtils.isEmpty(data)) {
            return null;
        }
        return JSON.parseArray(data, Transaction.class);
    }
```
这样处理，就是一个反操作，解包的过程。

**观察者**

```java
/**
 * Author   :hymane
 * Email    :hymanme@163.com
 * Create at 2018/7/27
 * Description: 事务接收回调
 */
public interface TransactionObserver {
    void onTransaction(List<Transaction> transactions);
}
```

**json 与 protobuf 优化**

> 本例使用的是 json，前面效率与性能一节我们提到 json 效率和 protobuf 效率的对比，很明显 protobuf 比 json 效率要高不少，那么如果需要更换也很简单，修改本类中的`pack`和`unpack`方法即可，换成 protobuf 来解包与压，其他的地方几乎不需要做任何修改。

PS：其中`Agora`部分就不介绍了，官网有很多案例介绍。这里贴一下`Agroa`生命周期时间顺序吧

![](https://wx1.sinaimg.cn/mw690/005X6W83gy1fv5l78hvgsj30u01hcn30.jpg)
![](https://wx1.sinaimg.cn/mw690/005X6W83gy1fv5l7bk236j30u01hc43z.jpg)

### 小小结
本小小结主要是介绍了数据包的管理以及发送，还是遵循单一原则理念，将任务分发给特定的类来处理，而不是一股脑堆一起。从数据包处理，整合到最终的`json`数据的发送，一层一层向下推进。

## 如何使用

1. 新建一个 Activity 或者 Fragment
2. 布局中添加 DoodleView
3. 新建绘制通道列表，用来实现多页白板绘制任务

    ```java
    private List<DoodleChannel> mDoodleChannels;
    ```
4. 初始化白板

    ```java
     /**
         * 初始化 DoodleView
         */
        private void initDoodleView() {
            mAccount = Md5Utils.md5(Constant.User.getUserName());

            Log.d(TAG, "initDoodleView: " + mAccount);
            //初始化 Agora
            AgoraUtils.setMsgCallback(mMsgCallback);
            AgoraUtils.getAgoraAPI().channelJoin(mChannelId);
            //先initDoodle后initChannel
            //内部调用registerTransactionObserver实现监听
            mDoodleView.initDoodle(this, mChannelId);
            mDoodleView.setDrawDelayTime(30);
            initDoodleChannel();
        }

        /***
         * 加载白板通道
         * page：当前为白板第几页
         */
        private void initDoodleChannel() {
            mDoodleView.initChannel(mDoodleChannels.get(page), null);
            mDoodleView.setCanvasBackgroundColor(Color.WHITE);
        }
    ```
5. 接收到数据时调用

    ```java
    TransactionCenter.getInstance().onReceive(mChannelId, msg);
    ```

6. 结束白板时调用`mDoodleView.end();`

## 总结
从本次实践中，学到了不少东西，从需求分析到确定选型，再到方案的优化，以及最后的实现。在分析问题中学到了解决问题的方法，开发中也完全不是实现就够了，更重要的如何实现达到最优。虽然现在还有一些已知的问题没有解决，比如多通道绘制覆盖路径问题，画布缩放中心问题等。嘿嘿嘿，知道问题就不是问题了。
