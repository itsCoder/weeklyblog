---
title: 使用贝塞尔曲线实现“一键下班”功能
date: 2016-11-13 22:01:10
categories: Android
tags: [Android, Bezier, animation]
---

# 前言
什么是“一键下班”？

![QQbubble](http://ww2.sinaimg.cn/mw690/005X6W83jw1f9qtzfqtj2j30ge07x0sw.jpg)

就是这个 QQ 消息气泡清除效果, QQ 气泡拖拽会产生粘性效果，拖拽到一定长度之后，连线断裂，气泡随之消失。大家都知道这用到了贝塞尔曲线，用来画链接线部分，ok，来学习一下贝塞尔曲线吧。

# 贝塞尔曲线
如果想用计算机来画一条直线，很简单，只要确定两个点，起点和终点就可以了，但是如果想要让计算机画一条曲线，那计算机该如何做呢？咳咳，肯定是通过数学公式来呗，
1962年，法国数学家Pierre Bézier第一个研究了这种矢量绘制曲线的方法，并给出了详细的计算公式，因此按照这样的公式绘制出来的曲线就用他的姓氏来命名是为贝塞尔曲线。那么可以说贝塞尔曲线就是通过某种公式绘制出的一条光滑曲线

## 一介贝塞尔曲线（直线）
方程：
![bezier1](http://a.hiphotos.baidu.com/baike/s%3D325/sign=391820be267f9e2f74351b0a2a31e962/91529822720e0cf34f59dca30b46f21fbe09aa38.jpg)

曲线：
![bezier1_line](http://ww1.sinaimg.cn/mw690/005X6W83jw1f9rzdypx1vg306o02s74z.gif)

一介贝塞尔曲线就是一条直线，确定两个点，得到一条直线。Android 中对应的方法为`lineTo(float x, float y)`

## 二介贝塞尔曲线
方程：
![bezier2](http://c.hiphotos.baidu.com/baike/s%3D317/sign=9aefef4b08f79052eb1f413f3bf2d738/11385343fbf2b21129581916cb8065380cd78e70.jpg)

曲线：
![bezier2_line](http://ww3.sinaimg.cn/mw690/005X6W83jw1f9rzdz67fgg306o02sgnc.gif)

二介贝塞尔曲线需要一对起始点以及一个控制点，如图控制点 p1 控制着曲线的拉伸程度。对应于 Android 中方法`quadTo(float x1, float y1, float x2, float y2)`

## 三介贝塞尔曲线
方程：
![bezier3](http://e.hiphotos.baidu.com/baike/s%3D421/sign=9a6521eab8014a90853e47bf98763971/f603918fa0ec08fad54f8dff58ee3d6d55fbda1f.jpg)

曲线：
![bezier3_line](http://ww2.sinaimg.cn/mw690/005X6W83jw1f9rzdzqbqlg306o02s76z.gif)

三介贝塞尔曲线需要一对起始点以及两个额外的控制点，控制点 p1,p2 控制着曲线的弯曲程度以及弯曲方向。对应于 Android 中方法`cubicTo(float x1, float y1, float x2, float y2,float x3, float y3)`

## 任意介贝塞尔曲线
方程：
![bezierx](http://f.hiphotos.baidu.com/baike/s%3D801/sign=a9e1f30835a85edffe8cf323785509d8/f9dcd100baa1cd11675be878b812c8fcc2ce2dfc.jpg)

高阶实际用的不多，可以通过多次低阶来实现。

# “一键下班”设计
需求---分析---实现
详见[QQ手机版 5.0“一键下班”设计小结](http://isux.tencent.com/qq-mobile-off-duty.html)

# 计算连接线算法
算法产考了一篇博客，原文网址找不到了，找了一个内容一样的博客，应该也是原作者博客，在此先贴出[博客地址](http://blog.csdn.net/xieyupeng520/article/details/50374561)，根据博客介绍，总结出几个重要控制点的计算公式，如下图（凑合着看，呵呵）
![source](http://ww3.sinaimg.cn/mw690/005X6W83jw1f9u8et0kh4j30sg0lcgn2.jpg)

![no diao use](http://ww3.sinaimg.cn/mw690/005X6W83jw1f9s1bl264yj30eo0ag40d.jpg)

其中两条拉伸的曲线 p12、p34,和两个圆，分别是跟随手指一动的圆或椭圆(显示消息数目的 view)，和原始位置处的随着拖拽距离加大圆半径越来越小的圆。

# 实现思路
1. 开始显示消息数目的 view，可以是一个带背景的 TextView
2. 手指触摸该 view 时记录状态
3. 手指拖着 view 滑动时，计算触摸点和原始点距离，是否大于临界距离
    - 大于最大值：断裂链接曲线，并停止绘制
    - 不大于：计算2对顶点以及对应的控制点坐标
4. 使用计算好的坐标等，重绘 view，包括曲线和原始点的圆，其中原始点的圆半径根据拖拽距离进行动态计算
5. 重复3,4步

# 代码
```java
    @Override
    protected void onDraw(Canvas canvas) {
        //计算曲线路径，可能距离超过，曲线已经断裂
        calculatePath();
        if (isTouch && !isBroken) {
            //绘制曲线path
            canvas.drawPath(mPath, mPaint);
            canvas.drawColor(Color.TRANSPARENT, PorterDuff.Mode.OVERLAY);
            canvas.drawCircle(mStartX, mStartY, mDotRadius, mPaint);
//触摸点的圆            canvas.drawCircle(mTextView.getX() + mTextView.getWidth() / 2, mTextView.getY() + mTextView.getHeight() / 2, mDefaultDotRadius, mPaint);
        } else {
            //曲线断裂，清楚曲线
            canvas.drawCircle(mStartX, mStartY, 0, mPaint);
            canvas.drawCircle(mTouchX, mTouchY, 0, mPaint);
            canvas.drawLine(0, 0, 0, 0, mPaint);
            canvas.drawColor(Color.TRANSPARENT, PorterDuff.Mode.OVERLAY);
        }
        super.onDraw(canvas);
    }
    @Override
    public boolean onTouchEvent(MotionEvent event) {
        mTouchX = (int) event.getX();
        mTouchY = (int) event.getY();
        Log.d("mTouchX=", mTouchX + "");
        Log.d("mTouchY=", mTouchY + "");
        switch (event.getAction()) {
            case MotionEvent.ACTION_DOWN:
                isBroken = false;
                mDotRadius = mDefaultDotRadius;
                break;
            case MotionEvent.ACTION_UP:
            case MotionEvent.ACTION_CANCEL:
                //清楚消息动画或者返回远处反弹动画
                isTouch = false;
                if (isBroken) {
                    disappearAnim();
                } else {
                    returnBackAnim(mTextView.getX(), mTextView.getY());
                }
                break;
            case MotionEvent.ACTION_MOVE:
                if (Math.abs(mTouchX - mStartX) < TOUCH_SLOP && Math.abs(mTouchY - mStartY) < TOUCH_SLOP) {
                    isTouch = false;
                } else {
                    mTextView.setX(mTouchX - mTextView.getWidth() / 2);
                    mTextView.setY(mTouchY - mTextView.getHeight() / 2);
                    isTouch = true;
                }
                break;
        }
        invalidateView();
        return true;
    }
    //重绘视图
    public void invalidateView() {
        if (Looper.myLooper() == Looper.getMainLooper()) {
            invalidate();
        } else {
            postInvalidate();
        }
    }
    //计算绘制区域的 path 路径
    private void calculatePath() {
        int dragLen = (int) Math.sqrt(Math.pow(mTouchX - mStartX, 2) + Math.pow(mStartY - mTouchY, 2));
        if (dragLen > mMaxDragLen) {
            isBroken = true;
        } else {
            //未达到最大断裂距离，计算path: 四个顶点p1,p2,p3,p4 => 控制点 anchor1,anchor2
            //得到绘制贝塞尔曲线需要的四个点
            mDotRadius = Math.max(mDefaultDotRadius * (1.0f - dragLen / mMaxDragLen), 12);

            float r = (float) Math.asin((mDefaultDotRadius - mDotRadius) / dragLen);
            float a = (float) Math.atan((mStartY - mTouchY) / (mTouchX - mStartX));

            float offset1X = (float) Math.cos(Math.PI / 2 - r - a);
            float offset1Y = (float) Math.sin(Math.PI / 2 - r - a);

            float offset2X = (float) Math.cos(Math.PI / 2 + r - a);
            float offset2Y = (float) Math.sin(Math.PI / 2 + r - a);

            //第一条曲线
            float x1 = mStartX - offset1X * mDotRadius;
            float y1 = mStartY - offset1Y * mDotRadius;

            float x2 = mTouchX - offset1X * mDefaultDotRadius;
            float y2 = mTouchY - offset1Y * mDefaultDotRadius;

            //第二条曲线
            float x3 = mStartX + offset2X * mDotRadius;
            float y3 = mStartY + offset2Y * mDotRadius;

            float x4 = mTouchX + offset2X * mDefaultDotRadius;
            float y4 = mTouchY + offset2Y * mDefaultDotRadius;

            //控制点1,2
            float mAnchor1X = (x1 + x4) / 2;
            float mAnchor1Y = (y1 + y4) / 2;
            float mAnchor2X = (x2 + x3) / 2;
            float mAnchor2Y = (y2 + y3) / 2;


            mPath.reset();
            mPath.moveTo(x1, y1);
            mPath.quadTo(mAnchor1X, mAnchor1Y, x2, y2);
            mPath.lineTo(x4, y4);
            mPath.quadTo(mAnchor2X, mAnchor2Y, x3, y3);
            mPath.lineTo(x1, y1);
        }
    }
```

效果图：
![example](http://ww1.sinaimg.cn/mw690/005X6W83jw1f9u8phvgxyj307i0dc0sv.jpg)

# 总结
动画可以给交互带来很大的好处，提高用户体验，之前在使用 QQ 气泡功能时候也感到此功能很神奇，查阅一些质料后发现，实现出来也不是很难，主要是点坐标的计算问题，这也牵扯到数学知识，还有有学霸提供了公式，方便了功能的实现。通过这个实例，也不难发现学好数学是很有必要的。最后贴上项目地址[BezierAnimation](https://github.com/hymanme/BezierAnimation)。


