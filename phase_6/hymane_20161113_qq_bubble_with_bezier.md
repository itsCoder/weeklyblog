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
1962年，法国数学家 Pierre Bézier 第一个研究了这种矢量绘制曲线的方法，并给出了详细的计算公式，因此按照这样的公式绘制出来的曲线就用他的姓氏来命名的，叫做贝塞尔曲线。那么可以说贝塞尔曲线就是通过某种公式绘制出的一条光滑曲线

## 一介贝塞尔曲线（直线）
方程：
![bezier1](http://ww4.sinaimg.cn/mw690/005X6W83jw1fa019aw3z7j309100iq2s.jpg)

曲线：
![bezier1_line](http://ww1.sinaimg.cn/mw690/005X6W83jw1f9rzdypx1vg306o02s74z.gif)

一介贝塞尔曲线就是一条直线，确定两个点，得到一条直线。Android 中对应的方法为`lineTo(float x, float y)`
x:终点 x 坐标，y:终点 y 坐标

## 二介贝塞尔曲线
方程：
![bezier2](http://ww1.sinaimg.cn/mw690/005X6W83jw1fa0199q31pj308t00l3yd.jpg)

曲线：
![bezier2_line](http://ww3.sinaimg.cn/mw690/005X6W83jw1f9rzdz67fgg306o02sgnc.gif)

二介贝塞尔曲线需要一对起点和终点以及一个控制点，如图控制点 p1 控制着曲线的拉伸程度。对应于 Android 中方法`quadTo(float x1, float y1, float x2, float y2)`
x1,y1:控制点 x,y 坐标，x2,y2:终点 x,y 坐标。

## 三介贝塞尔曲线
方程：
![bezier3](http://ww2.sinaimg.cn/mw690/005X6W83jw1fa019a5jfaj30bp00lq2u.jpg)

曲线：
![bezier3_line](http://ww2.sinaimg.cn/mw690/005X6W83jw1f9rzdzqbqlg306o02s76z.gif)

三介贝塞尔曲线需要一对起点和终点以及两个额外的控制点，控制点 p1,p2 控制着曲线的弯曲程度以及弯曲方向。对应于 Android 中方法`cubicTo(float x1, float y1, float x2, float y2,float x3, float y3)`
x1,y1:控制点1 x,y 坐标，x2,y2:控制点2 x,y 坐标，x3,y3:终点 x,y 坐标。

## 任意介贝塞尔曲线
方程：
![bezierx](http://ww2.sinaimg.cn/mw690/005X6W83jw1fa019aivwpj30m901cjri.jpg)

高阶实际用的不多，可以通过多次低阶来实现。

# “一键下班”设计
需求---分析---实现
详见[QQ手机版 5.0“一键下班”设计小结](http://isux.tencent.com/qq-mobile-off-duty.html)

# 计算连接线算法
算法参考了一篇博客，原文网址找不到了，找了一个内容一样的博客，应该也是原作者博客，在此先贴出[博客地址](http://blog.csdn.net/xieyupeng520/article/details/50374561)，根据博客介绍，总结出几个重要控制点的计算公式，如下图（凑合着看，呵呵）
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
                //清除消息动画或者返回远处反弹动画
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
    //每次手指按住气泡在屏幕上滑动都计算一下 path，来确定状态
    private void calculatePath() {
        //气泡拖拽的距离
        int dragLen = (int) Math.sqrt(Math.pow(mTouchX - mStartX, 2) + Math.pow(mStartY - mTouchY, 2));
        if (dragLen > mMaxDragLen) {
            //如果大于最大拖拽距离，则改变状态为 isBroken：断裂
            isBroken = true;
        } else {
            //未达到最大断裂距离，计算path: 四个顶点p1,p2,p3,p4 => 控制点 anchor1,anchor2
            //初始位置点圆半径，根据气泡拖拽偏离距离改变原始气泡的大小，即越网远处滑动，初始的圆就越小
            mDotRadius = Math.max(mDefaultDotRadius * (1.0f - dragLen / mMaxDragLen), 12);

            //伽马角和阿尔法角度
            float r = (float) Math.asin((mDefaultDotRadius - mDotRadius) / dragLen);
            float a = (float) Math.atan((mStartY - mTouchY) / (mTouchX - mStartX));

            //几个偏移量，中间数据用来确定4个顶点坐标
            float offset1X = (float) Math.cos(Math.PI / 2 - r - a);
            float offset1Y = (float) Math.sin(Math.PI / 2 - r - a);

            float offset2X = (float) Math.cos(Math.PI / 2 + r - a);
            float offset2Y = (float) Math.sin(Math.PI / 2 + r - a);


            //得到绘制贝塞尔曲线需要的四个点，p1,p2,p3,p4
            //第一条曲线:p1~~~p2
            float x1 = mStartX - offset1X * mDotRadius;
            float y1 = mStartY - offset1Y * mDotRadius;

            float x2 = mTouchX - offset1X * mDefaultDotRadius;
            float y2 = mTouchY - offset1Y * mDefaultDotRadius;

            //第二条曲线:p3~~~p4
            float x3 = mStartX + offset2X * mDotRadius;
            float y3 = mStartY + offset2Y * mDotRadius;

            float x4 = mTouchX + offset2X * mDefaultDotRadius;
            float y4 = mTouchY + offset2Y * mDefaultDotRadius;

            //两个控制点,anchor1：p1,p4点的重心,anchor2：p2,p3点的重心,
            float mAnchor1X = (x1 + x4) / 2;
            float mAnchor1Y = (y1 + y4) / 2;

            float mAnchor2X = (x2 + x3) / 2;
            float mAnchor2Y = (y2 + y3) / 2;

            //重置路径
            mPath.reset();
            //路径移动到起点，p1
            mPath.moveTo(x1, y1);
            //画曲线p1~~~p2,二介贝塞尔，参数一个控制点，一个重点
            mPath.quadTo(mAnchor1X, mAnchor1Y, x2, y2);
            //此时路径已经到p2点了，画直线p2---p4
            mPath.lineTo(x4, y4);
            //画曲线p4~~~p3
            mPath.quadTo(mAnchor2X, mAnchor2Y, x3, y3);
            //封闭曲线，画直线p3---p1，到此画了一个沙漏的形状，接下来只要画两头的圆就可以了
            mPath.lineTo(x1, y1);
        }
    }
```

效果图：
![example](http://ww1.sinaimg.cn/mw690/005X6W83jw1f9u8phvgxyj307i0dc0sv.jpg)

# 总结
动画可以给交互带来很大的好处，提高用户体验，之前在使用 QQ 气泡功能时候也感到此功能很神奇，查阅一些资料后发现，实现出来也不是很难，主要是点坐标的计算问题，这也牵扯到数学知识，还好有学霸提供了公式，方便了功能的实现。通过这个实例，也不难发现学好数学是很有必要的。最后贴上项目地址[BezierAnimation](https://github.com/hymanme/BezierAnimation)。


