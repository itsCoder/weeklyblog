## GearMachine Canvas绘制漂亮的齿轮装置

#### 前言
无意中看到一加网站上有一个漂亮的齿轮装置动画 [www.oneplus.cn/3years](www.oneplus.cn/3years)，第一眼惊艳，于是想尝试一下在 Android 下利用 Canvas 的相关 API 以纯代码的方式绘制出来。

本文主要记录一下开发这个控件的过程，如果不了解 Canvas ， Paint ， Path 等相关概念的小伙伴，请先翻看官方文档进行学习。

临摹的效果来源于一加，关于设计部分的版权归一加所有。

#### 先看实现的效果

[Gif动图录制较卡顿，更清晰流畅的效果视频请移步 Youtube ](https://youtu.be/Bac1nlrCZyk)

<div align = center>
<img src="https://github.com/andyxialm/GearMachine/blob/master/art/gear.gif?raw=true" width = "60%" />
</div>



#### 效果分析
1. 首先确定四个关键点，即:
   左上角齿轮中心，右上角齿轮中心，和中线上的两个齿轮中心
2. 左上角和右上角的齿轮，需要三角形的锯齿效果。  
   下方的大齿轮和小齿轮需要梯形的锯齿效果。
   纵轴上的两个齿轮需要圆形的小孔
3. 传送带
4. 最下方齿轮中心的阴影效果

**分析完毕**


**首先获取几个关键点 计算的方式是按照位置占全图宽高的比值**

```java

    @Override
    protected void onSizeChanged(int w, int h, int oldw, int oldh) {
        super.onSizeChanged(w, h, oldw, oldh);

        int paddingLeft = getPaddingLeft();
        int paddingRight = getPaddingRight();
        int paddingTop = getPaddingTop();
        int paddingBottom = getPaddingBottom();

        // 横向有效宽度
        int validWidth = getWidth() - paddingLeft - paddingRight;
        // 纵向有效高度
        int validHeight = getHeight() - paddingTop - paddingBottom;

        // 左上齿轮中心
        mLeftTopCentre[X] = (float) (paddingLeft + validWidth * 0.237);
        mLeftTopCentre[Y] = (float) (paddingTop + validHeight * 0.102);

        // 右上齿轮中心
        mRightTopCentre[X] = (float) (paddingLeft + validWidth * 0.763);
        mRightTopCentre[Y] = (float) (paddingTop + validHeight * 0.102);

        // 中上齿轮中心
        mTopCentre[X] = (float) (paddingLeft + validWidth * 0.5);
        mTopCentre[Y] = (float) (paddingTop + validHeight * 0.311);

        // 中下齿轮中心
        mBottomCentre[X] = (float) (paddingLeft + validWidth * 0.5);
        mBottomCentre[Y] = (float) (paddingTop + validHeight * 0.597);
    }
```

**使用 Canvas 的相关 API 绘制左上角齿轮**
最上方两个齿轮效果除了动画转动的方向不同，其他的相同，所以我们现在以左上角的齿轮为例。

```java

    // 左上齿轮
    // 最底层(半径最大)的红色圆的背景
    canvas.drawCircle(mLeftTopCentre[X], mLeftTopCentre[Y], mTopCircleRadius, mRedPaint); 

    // 半径第二大的深红色圆背景
    canvas.drawCircle(mLeftTopCentre[X], mLeftTopCentre[Y], mTopCircleRadius / 4 * 3, mDarkRedPaint); 
    canvas.save();
    
    // 动画需要，根据mCurProgress的值来做画布旋转
    canvas.rotate(mCurProgress * 360, mLeftTopCentre[X], mLeftTopCentre[Y]);
    
    // 绘制24个三角齿轮
    for (int i = 0; i < 24; i ++) {
    
        // 齿轮的三角 path
        canvas.drawPath(mLeftTopTrianglePath, mRedPaint);
    
        // 旋转角为15度，绘制24个三角形
        canvas.rotate(15.0f, mLeftTopCentre[X], mLeftTopCentre[Y]);
    }
    canvas.restore();
    
    // 绘制其他圆形
    canvas.drawCircle(mLeftTopCentre[X], mLeftTopCentre[Y], mTopCircleRadius / 2, mRedPaint);
    canvas.drawCircle(mLeftTopCentre[X], mLeftTopCentre[Y], mTopCircleRadius / 5 * 2, mDarkRedPaint);
    canvas.drawCircle(mLeftTopCentre[X], mLeftTopCentre[Y], mTopCircleRadius / 4, mRedPaint);
    
```
**注意**
下方最大的齿轮相对特殊，原因是有一个 130 度的缺口，并且锯齿也需要转动的动画，所以此处我先绘制了一个大空心圆，并使用 drawArc() 绘制扇形以初始角度为 180 度，扇形圆心角 130 度再搭配使用 PorterDuffXfermode(PorterDuff.Mode.CLEAR) 来进行叠加，此时就可以绘制出 130 度缺口的大齿轮了。

**梯形**
至于梯形的绘制使用 Path 连线 4 个点后 并每次旋转画布 20 度，动态绘制 18 份就可以完成大齿轮的锯齿显示。

**传送带**
传送带的效果，利于 drawLine(float startX, float startY, float stopX, float stopY, Paint paint) 可绘制两个点的连线。


**阴影**
如果仔细观察，你会发现在下方大齿轮的中心，其实是存在阴影效果的。此时我选用的方案是利用 View 的 setLayerType(int layerType, Paint paint) 方法搭配使用 Paint 的 setShadowLayer(float radius, float dx, float dy, int shadowColor) 方法实现阴影效果。

**有了以上的分析与实现，剩下的部分可以通过举一反三的方式很轻松的绘制出来**
**...**

**绘制完成，怎么能让齿轮转动起来呢？**
这次实现的效果其实在动画方面可以说非常简单，只要让齿轮能转起来就好了嘛。齿轮转动无外乎从以圆心为准做 0 度到 360 度旋转就好了。咦，数值的变化，是不是脑海里第一想到的就是使用 ValueAnimator ？没错。

```java

	mValueAnimator = ValueAnimator.ofFloat(0f, 1f);
	mValueAnimator.setRepeatCount(ValueAnimator.INFINITE);
	mValueAnimator.addUpdateListener(new ValueAnimator.AnimatorUpdateListener() {
    	@Override
    	public void onAnimationUpdate(ValueAnimator valueAnimator) {
        	mCurProgress = (float) valueAnimator.getAnimatedValue();
        	postInvalidate();
    	}
	});
	mValueAnimator.setDuration(5000);
	mValueAnimator.setInterpolator(new LinearInterpolator());
	mValueAnimator.start();

```

动画时长 5 秒，设置重复模式为无限循环 (INFINITE) 。虽然此处的 AnimatedValue 的范围是 [0f,1f] ，不过只要再与 360 相乘即可获得画布旋转的角度。

[Talk is cheap, show me the code. 完整代码已开源至 Github](https://github.com/andyxialm/GearMachine)


####后记

关于这个漂亮的齿轮装置效果的实现就到此结束了。其实把一个复杂的效果拆分成小模块，问题就会变得非常容易解决。
笔者要继续去抢春运火车票啦。:grimacing:
