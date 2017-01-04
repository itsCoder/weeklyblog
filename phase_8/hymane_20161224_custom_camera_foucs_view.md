---
title: 仿 google 相机点击聚焦效果
date: 2016-12-24 14:50:11
categories: Android
tags: [custom camera ,animation]
---

# 引言
最近需要做一个自定义相机的功能，然后就模仿了一下谷歌相机，加上自己的需求来进行自定义，由于暂时还未完工，所以就拿其中的触摸聚焦来交下作业。

# 简介
在 Android 中使用相机有两种，一种是使用 Intent 调用相机，还有一种是自定义相机。不管哪一种都有一个很常用且重要的功能，那就是聚焦，虽然本文并非讲解聚焦的实现，但是也算是其一个辅助功能吧。自定义一个 View，用于在用户触摸屏幕进行聚焦时给用户一个反馈，以提高用户体验，一般我们会看到一些自定义相机用一个方框来做聚焦反馈元件。如下图。

![focusview](http://ww2.sinaimg.cn/mw690/005X6W83jw1fb3cz92kaoj305203n3yh.jpg)

# 效果展示
谷歌相机聚焦效果

![googlefocusview](http://ww3.sinaimg.cn/mw690/005X6W83jw1fb3ci3davwg308w0fsqv5.gif)

自定义相机聚焦效果

![customfocusview](http://ww1.sinaimg.cn/mw690/005X6W83jw1fb3ci0ugf0g308w0fs7wi.gif)

# 分析
首先想到的是自定义一个 FocusView，通过监听 SurfaceView 的触摸事件，当用户点击屏幕对应的点时候，动态设置这个 FocusView 的位置，最后开启动画。接下来将这个动画分解成若干步。

1. 点击屏幕时，外圆环半径从最大位置缩小到最小位置，同时半透明内圆半径从0扩大到与外圆环最小半径相等为止，两个动画时间是同步的。
2. 开始动画结束后，内圆将逐渐消失，即透明度从原始透明度逐渐变为全透明。
3. 当内圆不见了（变透明）的时候，外圆环和内圆一样开始逐渐消失。
4. 
ps：当用户点击屏幕时，重复以上动画。动画开始时需要重置动画各个状态，不然会出现动画错乱。

# 实现
### 初始化变量、状态以及画笔
```java
  private void init() {
        this.mPaint = new Paint();
         this.mPaint.setAntiAlias(true); //消除锯齿
        //当前外圆环半径
        mOutCircleRadius = DensityUtils.dp2px(mContext, OUT_CIRCULAR_START_RADIUS);
        //当前内圆半径，一开始内圆不显示
        mInnerCircleRadius = 0;
        //当前外圆环透明度，一开始外圆不显示，只有点击屏幕时候才显示
        mOutCircleAlpha = 0;
        //当前内圆透明度
        mInnerCircleAlpha = INNER_CIRCULAR_START_ALPHA;
    }
```

### 重写 onMeasure() 方法
不重写该方法，我们的 view 宽高则为屏幕宽高，在计算以及设置视图位置时会不方便，所以让 view 宽高保持一致，呈现一个正方形区域。
```java
    @Override
    protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
        super.onMeasure(widthMeasureSpec, heightMeasureSpec);
        setMeasuredDimension(ScreenUtils.getScreenWidth(mContext), ScreenUtils.getScreenWidth(mContext));
    }
```

### 重写 OnDraw() 方法
根据当前状态绘制 view。
```java
    @Override
    protected void onDraw(Canvas canvas) {
        //view 中心点
        int center = getWidth() / 2;
        mPaint.reset();
        //绘制内圆
        mPaint.setARGB(mInnerCircleAlpha, 255, 255, 255);//设置内圆画笔颜色，半透明
        //画圆
        canvas.drawCircle(center, center, mInnerCircleRadius, mPaint);

        //绘制外圆环
        mPaint.setARGB(mOutCircleAlpha, 255, 255, 255);
        this.mPaint.setStyle(Paint.Style.STROKE); //绘制空心圆
        mPaint.setStrokeWidth(ringWidth);//设置圆环宽度
        //画圆环
        canvas.drawCircle(center, center, mOutCircleRadius + ringWidth / 2, mPaint);
        super.onDraw(canvas);
    }
```

### 暴露一个共有方法供外界调用
```java
    /***
     * 开始动画
     *
     * @param x 动画发生x坐标
     * @param y 动画发生y坐标
     */
    public void startFocus(float x, float y) {
        //设置 focusView 位置
        setX(x - getWidth() / 2);
        setY(y - getHeight() / 2);
        startAnim();//开启动画
    }
```

### 开启动画
```java
    /**
     * 开启动画
     */
    private void startAnim() {
        final int maxInnerCircleRadius = DensityUtils.dp2px(mContext, INNER_CIRCULAR_END_RADIUS);
        final int maxOutCircleRadius = DensityUtils.dp2px(mContext, OUT_CIRCULAR_START_RADIUS);
        final int minOutCircleRadius = DensityUtils.dp2px(mContext, OUT_CIRCULAR_END_RADIUS);
        //外圆环逐渐透明，动画3
        if (alphaAnimator2 == null) {
            //外圆环透明动画有一个延时效果
            alphaAnimator2 = ValueAnimator.ofFloat(1.0f, 0.9f, 0.0f);
            alphaAnimator2.setDuration(OUT_CIRCULAR_ALPHA_DURATION);
            alphaAnimator2.setInterpolator(new DecelerateInterpolator());
            alphaAnimator2.addUpdateListener(new ValueAnimator.AnimatorUpdateListener() {
                @Override
                public void onAnimationUpdate(ValueAnimator animation) {
                    final float value = (float) animation.getAnimatedValue();
                    //计算外圆环透明度
                    mOutCircleAlpha = (int) (value * OUT_CIRCULAR_START_ALPHA);
                    invalidate();
                }
            });
        }
        //内圆逐渐透明，动画2
        if (alphaAnimator1 == null) {
            alphaAnimator1 = ValueAnimator.ofFloat(1.0f, 0.0f);
            alphaAnimator1.setDuration(INNER_CIRCULAR_ALPHA_DURATION);
            alphaAnimator1.setInterpolator(new LinearInterpolator());
            alphaAnimator1.addUpdateListener(new ValueAnimator.AnimatorUpdateListener() {
                @Override
                public void onAnimationUpdate(ValueAnimator animation) {
                    final float value = (float) animation.getAnimatedValue();
                    //计算内圆透明度
                    mInnerCircleAlpha = (int) (value * INNER_CIRCULAR_START_ALPHA);
                    invalidate();
                }
            });

            alphaAnimator1.addListener(new Animator.AnimatorListener() {
                @Override
                public void onAnimationStart(Animator animation) {

                }

                @Override
                public void onAnimationEnd(Animator animation) {
                    if (alphaAnimator2 != null) {
                        alphaAnimator2.start();
                    }
                }

                @Override
                public void onAnimationCancel(Animator animation) {

                }

                @Override
                public void onAnimationRepeat(Animator animation) {

                }
            });
        }
        //开始缩放动画，动画1
        if (scaleAnimator == null) {
            scaleAnimator = ValueAnimator.ofFloat(0.0f, 1.0f);
            scaleAnimator.setDuration(START_CIRCULAR_ANIM_DURATION);
            scaleAnimator.setInterpolator(new DecelerateInterpolator());
            scaleAnimator.addUpdateListener(new ValueAnimator.AnimatorUpdateListener() {
                @Override
                public void onAnimationUpdate(ValueAnimator animation) {
                    final float value = (float) animation.getAnimatedValue();
                    //分别计算内外圆当前半径，内圆半径是逐渐增大
                    //外圆环半径是逐渐缩小
                    mOutCircleRadius = maxOutCircleRadius - (maxOutCircleRadius - minOutCircleRadius) * value;
                    mInnerCircleRadius = maxInnerCircleRadius * value;
                    invalidate();
                }
            });
            scaleAnimator.addListener(new Animator.AnimatorListener() {
                @Override
                public void onAnimationStart(Animator animation) {
                    //动画开始时初始化状态
                    mInnerCircleAlpha = INNER_CIRCULAR_START_ALPHA;
                    mOutCircleAlpha = OUT_CIRCULAR_START_ALPHA;
                }

                @Override
                public void onAnimationEnd(Animator animation) {
                    if (alphaAnimator1 != null) {
                        alphaAnimator1.start();
                    }
                }

                @Override
                public void onAnimationCancel(Animator animation) {

                }

                @Override
                public void onAnimationRepeat(Animator animation) {

                }
            });
        }
        if (alphaAnimator1.isRunning() || alphaAnimator2.isRunning()) {
            alphaAnimator1.cancel();
            alphaAnimator2.cancel();
        }
        //开始动画
        scaleAnimator.start();
    }
```

# 总结
这个动画其实也不难，但对于从来不搞动画的我来说，弄起来还是花了点时间的，关键点是在动画的分解，然后再一步一步实现。感觉动画真的是开始于模仿啊。