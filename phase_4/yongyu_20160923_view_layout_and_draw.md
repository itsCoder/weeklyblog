---

 title:  View 的工作原理下 View 的 layout 和 draw 过程 （Android 开发艺术探索读书笔记）
 date:  2016/9/3
 categories:  Android View
 tags:  Android View

---

>-文章来源：itsCoder 的 [WeeklyBolg](https://github.com/itsCoder/weeklyblog) 项目
>- itsCoder主页：[http://itscoder.com/](http://itscoder.com/)
>- 作者：[yongyu0102](https://github.com/yongyu0102)
>- 审阅者：

# 一、概要

本次介绍的主要内容是 View 的工作原理下 View 的 layout 和 draw 过程，同时介绍自定义 View 的注意事项并结合一个小的 Demo 进行说明，其中涉及到的 onMeasre  测量部分知识可以看上一篇文章[View 的工作原理上 View 绘制流程梳理及 Measure 过程详解](http://yongyu.itscoder.com/2016/09/11/view_measure/) ，以下开始正文：

# 二、 layout 过程详解

 layout 的作用是 ViewGroup 来确定子元素的位置，当 ViewGroup 的位置被确定后，在  layout 中会调用 onLayout ，在 onLayout 中会遍历所有的子元素并调用子元素的 layout 方法，在子元素的 layout 方法中 onLayout 方法又会被调用，layout 方法是确定 View 本身在屏幕上显示的具体位置，即在代码中设置其成员变量 mLeft，mTop，mRight，mBottom 的值，这几个值是在屏幕上构成矩形区域的四个坐标点，就是该 View 显示的位置，不过这里的具体位置都是相对与父视图的位置而言，而 onLayout 方法则会确定所有子元素位置，ViewGroup 在 onLayout 函数中通过调用其 children 的 layout 函数来设置子视图相对与父视图中的位置，具体位置由函数 layout 的参数决定。下面我们先看 View 的layout 方法如下：   

```java
  /* 
  *@param l view 左边缘相对于父布局左边缘距离
  *@param t view 上边缘相对于父布局上边缘位置
  *@param r view 右边缘相对于父布局左边缘距离
  *@param b view 下边缘相对于父布局上边缘距离
  */
  public void layout(int l, int t, int r, int b) {
  if ((mPrivateFlags3 & PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT) != 0) {
       onMeasure(mOldWidthMeasureSpec, mOldHeightMeasureSpec);
       mPrivateFlags3 &= ~PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT;
   }
   //记录 view 原始位置
   int oldL = mLeft;
   int oldT = mTop;
   int oldB = mBottom;
   int oldR = mRight;
 //第1步，调用 setFrame 方法 设置新的 mLeft、mTop、mBottom、mRight 值，
 //设置 View 本身四个顶点位置
 //并返回 changed 用于判断 view 布局是否改变
   boolean changed = isLayoutModeOptical(mParent) ?
           setOpticalFrame(l, t, r, b) : setFrame(l, t, r, b);
              //第二步，如果 view 位置改变那么调用 onLayout 方法设置子 view 位置
              if (changed || (mPrivateFlags & PFLAG_LAYOUT_REQUIRED) == PFLAG_LAYOUT_REQUIRED) {
     //调用 onLayout
       onLayout(changed, l, t, r, b);
       mPrivateFlags &= ~PFLAG_LAYOUT_REQUIRED;
       ListenerInfo li = mListenerInfo;
       if (li != null && li.mOnLayoutChangeListeners != null) {
           ArrayList<OnLayoutChangeListener> listenersCopy =
                   (ArrayList<OnLayoutChangeListener>)li.mOnLayoutChangeListeners.clone();
           int numListeners = listenersCopy.size();
           for (int i = 0; i < numListeners; ++i) {
               listenersCopy.get(i).onLayoutChange(this, l, t, r, b, oldL, oldT, oldR, oldB);
           }
       }
   }
```

 layout 方法大致流程：先通过上面代码第一步调用  setFrame 设置 view 本身四个顶点位置，其中 setOpticalFrame 内部也是调用 setFrame  方法来完成设置的，即为 View 的4个成员变量（mLeft，mTop，mRight，mBottom）赋值，View 的四个顶点一旦确定，那么 View 在父容器中的位置就确定了，接着进行第二步，调用 onLayout 方法，这个方法用途是父容器确定子 View 位置，和 onMeasure 方法类似， onLayout 方法的具体实现同样和具体布局有关，所以 View 和 ViewGroup 中都没有真正实现 onLayout 方法，都是一个空方法。

   先看一下 View 的 onLayout 方法：

 ``` protected void onLayout(boolean changed, int left, int top, int right, int bottom) {}```

   那么对于我们自定义的 View 是继承自 View 的情况下，我们一般不需要重写 onLayout 方法，因为 这个方法用途是父容器确定子 View 位置，对于 View 来说是没有子 View 的，所以一般不需要重写。

   再看一下 ViewGroup 的 onLayout 方法：

```java
protected abstract void onLayout(boolean changed,
           int l, int t, int r, int b);
```

   相对于 view 来说，ViewGroup 中 onLayout 多了关键字abstract的修饰，也就说对于继承自 ViewGroup 的自定义 View 必须要重写 onLayout 方法，而重载 onLayout 的目的就是安排其子元素在父视图中的具体位置，为了更好的理解，接下来我们看一下 LinearLayout  的 onLayout 方法：

```java
   @Override
   protected void onLayout(boolean changed, int l, int t, int r, int b) {
       if (mOrientation == VERTICAL) {
           layoutVertical(l, t, r, b);
       } else {
           layoutHorizontal(l, t, r, b);
       }
   }
```

   和 onMeasure 类似，这里也是分为竖直方向和水平方向的布局安排，二者原理一样，我们选择竖直方向的 layoutVertical 来进行分析，这里给出主要代码如下：

```java
   void layoutVertical(int left, int top, int right, int bottom) {
       final int paddingLeft = mPaddingLeft;
     //记录子 View 上边缘相对于父容器上边缘距离
       int childTop;
       //记录子 View 左边缘相对于父容器左边缘距离
       int childLeft;
     //第1步，主要是根据不同的 gravity 属性来确定子元素的 child 的位置
       switch (majorGravity) {
              case Gravity.BOTTOM:
                  // mTotalLength contains the padding already
                  childTop = mPaddingTop + bottom - top - mTotalLength;
                  break;
                  // mTotalLength contains the padding already
              case Gravity.CENTER_VERTICAL:
                  childTop = mPaddingTop + (bottom - top - mTotalLength) / 2;
                  break;

              case Gravity.TOP:
              default:
                  childTop = mPaddingTop;
                  break;
           }
      ...............................
   //第2步，循环遍历子 view 
       for (int i = 0; i < count; i++) {
         //获取指定位置 view 
           final View child = getVirtualChildAt(i);
           if (child == null) {
               childTop += measureNullChild(i);
           } else if (child.getVisibility() != GONE) {
             //第2.1步，如果 view 可见，获取 view  的测量宽/高
               final int childWidth = child.getMeasuredWidth();
               final int childHeight = child.getMeasuredHeight();
               //获取 view  的 LayoutParams 参数
               final LinearLayout.LayoutParams lp =
                       (LinearLayout.LayoutParams) child.getLayoutParams();
           .............
               if (hasDividerBeforeChildAt(i)) {
                   childTop += mDividerHeight;
               }
   			
               childTop += lp.topMargin;
             //第3步，设置子 view 位置
               setChildFrame(child, childLeft, childTop + getLocationOffset(child),
                       childWidth, childHeight);
             //第4步，重新计算子 view 的 顶部 top 位置，也就是每增加一个子 view 
             //下一个子 view 的 top 顶部位置就会相应的增加
               childTop += childHeight + lp.bottomMargin + getNextLocationOffset(child);
               i += getChildrenSkipCount(child, i);
           }
       }
   }
```

  简单梳理下整个流程，此方法会遍历所有子 view ，并调用 setChildFrame 方法来设定子元素位置，然后重新计算 childTop ，childTop  随着子元素的遍历而逐渐增大，这就意味着后面的子元素会被放置在当前子元素的下方，这正是我们平时使用竖直方向 LinearLayout 的特性。这里我们看一下第三步执行的 setChildFrame 方法类设置子元素位置方法代码：

```java
   private void setChildFrame(View child, int left, int top, int width, int height) {        
       child.layout(left, top, left + width, top + height);
   }
```

可以发现，这个方法只是调用子元素的 layout 方法而已，这样父元素在自己的 layout 方法中完成自己的定位之后，通过 onLayout 方法去调用了子元素的 layout 方法，子元素又会通过自己的 layout 方法完成自己的位置设定，这样一层一层的传递下去就完成了整个 view 数的 layout 过程。

这里我们注意到在第三步调用 setChildFrame 方法中的 传入的参数 childWidth 和 childHeight 是上面第2.1步获取的子元素的测量宽/高，而在 layout 过程中会通过 setFrame 方法设置子元素四个顶点位置，这样子元素的位置就确定了，在 setFrame 中有如下赋值语句：

```java
mLeft = left;
mTop = top;
mRight = right;
mBottom = bottom;
```

也就是说在 LinearLayout 中其子视图显示的宽和高由 measure 过程来决定的，因此 measure 过程的意义就是为 layout 过程提供视图显示范围的参考值。为什么说是提供参考值呢？因为 layout 过程中的4个参数  left,  top, iwidth, height 完全可以由视图设计者任意指定，而最终视图的布局位置和大小完全由这4个参数决定，measure 过程得到的mMeasuredWidth 和 mMeasuredHeight 提供了视图大小的测量值，只是提供一个参考一般情况下我们使用这个参考值，但我们完全可以不使用这两个值，而自己在 layout 过程中去设定一个值，可见 measure 过程并不是必须的。

说到这里就不得说一下 getWidth() 、getHeight() 和 getMeasuredWidth()、getMeasuredHeight() 这两对函数之间的区别，即 View 的测量宽/高和最终显示宽/高之间的区别。首先我们看一下 getWith() 和 getHeight() 方法的具体实现：

```java
public final int getWidth() {
    return mRight - mLeft;
}

 public final int getHeight() {
        return mBottom - mTop;
    }
```

通过 getWith() 和 getHeight() 源码和上面 setChildFrame(View child, int left, int top, int width, int height) 方法设置子元素四个顶点位置的四个变量 mLeft、mTop、mRight、mBottom 的赋值过程来看，默认情况下 getWidth() 、getHeight() 方法返回的值正好就是 view 的测量宽/高，只不过 view 的测量宽/高形成于 view 的measure 过程，而最终宽/高形成于 view 的 layout 方法中，但是对于特殊情况，两者的值是不相等的，就是我们在 layout 过程中不按默认常规套路出牌，即不使用 measure 过程得到的 mMeasuredWidth 和 mMeasuredHeight ，而是人为的去自己根据需要设定的一个值的情况，例如以下代码，重写 view 的 layout 方法：

```java
public void layout(int l, int t, int r, int b) {
  //在得到的测量值基础上加100
    super.layout(int l, int t, int r+100, int b+100);
    }
```

上面代码会导致在任何情况下 view 的最终宽/高总会比测量宽高大100px。

# 三、 draw 过程详解

draw 的作用是将 view 绘制到屏幕上，view 的绘制过程准守以下几个步骤：

1. 绘制背景：`background.draw(canvas)`；
2. 绘制自己：`onDraw()`；
3. 绘制 children：`dispatchDraw`；
4. 绘制装饰：`onDrawScrollBars`。

 通过源码可以看出来，部分源码如下：

```java
public void draw(Canvas canvas) {
    final int privateFlags = mPrivateFlags;
    final boolean dirtyOpaque = (privateFlags & PFLAG_DIRTY_MASK) == PFLAG_DIRTY_OPAQUE &&
            (mAttachInfo == null || !mAttachInfo.mIgnoreDirtyState);
    mPrivateFlags = (privateFlags & ~PFLAG_DIRTY_MASK) | PFLAG_DRAWN;

    /*
     * Draw traversal performs several drawing steps which must be executed
     * in the appropriate order:
     *
     *      1. Draw the background
     *      2. If necessary, save the canvas' layers to prepare for fading
     *      3. Draw view's content
     *      4. Draw children
     *      5. If necessary, draw the fading edges and restore layers
     *      6. Draw decorations (scrollbars for instance)
     */

    // Step 1, draw the background, if needed
  //绘制背景
    int saveCount;

    if (!dirtyOpaque) {
        drawBackground(canvas);
    }

    // skip step 2 & 5 if possible (common case)
    final int viewFlags = mViewFlags;
    boolean horizontalEdges = (viewFlags & FADING_EDGE_HORIZONTAL) != 0;
    boolean verticalEdges = (viewFlags & FADING_EDGE_VERTICAL) != 0;
    if (!verticalEdges && !horizontalEdges) {
        // Step 3, draw the content
      //调用 onDraw 方法，绘制自己本身内容，这个方法是个空方法，没有具体实现，
      //因为每个视图的内容部分肯定都是各不相同的，这部分的功能交给子类来去实现，
      //如果要自定义 view ，需要重载该方法完成绘制工作
        if (!dirtyOpaque) onDraw(canvas);

        // Step 4, draw the children
      //绘制子视图
      //View 中的 dispatchDraw()方法也是一个空方法，因为 view 本身没有子视图，所以不需要，
      //而 ViewGroup 的 dispatchDraw() 方法中就会有具体的绘制代码，来实现子视图的绘制工作
        dispatchDraw(canvas);

        // Overlay is part of the content and draws beneath Foreground
        if (mOverlay != null && !mOverlay.isEmpty()) {
            mOverlay.getOverlayView().dispatchDraw(canvas);
        }

        // Step 6, draw decorations (foreground, scrollbars)
      //绘制装饰
      //对视图的滚动条进行绘制，其实任何一个视图都是有滚动条的，只是一般情况下都没有让它显示出来，
      //而例如像 ListView 等控件是进行了显示而已。
        onDrawForeground(canvas);

        // we're done...
        return;
    }
```

通过上面代码可以发现，View 绘制过程的传递是通过 dispatchDraw() 方法完成，这个方法会遍历调用所有子视图的 draw ()方法，这样事件就一层一层的传递下去了。其中 View 中有一个特殊方法 setWillNotDraw ，源码如下：

```java
/**
 * If this view doesn't do any drawing on its own, set this flag to
 * allow further optimizations. By default, this flag is not set on
 * View, but could be set on some View subclasses such as ViewGroup.
 *
 * Typically, if you override {@link #onDraw(android.graphics.Canvas)}
 * you should clear this flag.
 *
 * @param willNotDraw whether or not this View draw on its own
 */
public void setWillNotDraw(boolean willNotDraw) {
    setFlags(willNotDraw ? WILL_NOT_DRAW : 0, DRAW_MASK);
}
```

看注释部分大概意思是，如果一个 View 不需要绘制任何内容，那么设置这个标记位为 true 后，系统会进行相应的优化，默认情况下 View 没有启动这个默认标记位，但 viewGroup 默认启用这个标记位，这个标记位对实际开发的意义是：当我们的自定义的控件继承自 viewGroup 并且本身不具备绘制功能的时候，就可以开启这个标记位，从而便于系统进行后续的优化工作，当我们明确知道 viewGrop 需要通过 onDraw 来绘制本身内容时，需要我们去关闭 WILL_NOT_DRAW 这个标记位。

# 四、自定义 View

## 4.1 自定义 View 分类

1. **继承自 View 重写 ondraw 方法** 

   这种方法主要用于实现一些不规则的效果，即想要达到的 View 效果无法使用已有的 View 通过布局组合的方式来实现，所以需要我们自己去绘制去画一个出来，即重写 onDraw 方法，采用这种方式需要注意处理自定义的 View 支持 wrap_content ，并且 padding 也需要自己处理。

2. **继承自 ViewGroup 实现特殊的 Layout 容器**

   主要实现除了 LinearLayout 、 RelativeLayout  等系统已有的 View 容器之外的特殊 View 容器，需要处理 ViewGroup 的测量 onMeasure 和布局 onLayout 这两个方法，并同时处理子元素的测量和布局。

3. **继承自 Android 系统本身已有的特定 View （如 TextView）**

   这种方法是要是为拓展某个已有 View 的功能，在已有的 View 的基础上添加一些功能，方便我们重复使用，这种方法不需要我们进行特殊的处理。

4. **继承自 Android 系统本身已有的特定的 ViewGroup （如 LinearLayout)**

   这种方法主要是为了实现将几个 View 组合在一起形成一个特定的组合模块，来方便我们后续进行使用，例如我们想要一个特定的 TitleBar ，我们可以可以将几个 TextView 和 Button 放在一个 LinearLayout 布局中组合成一个自定义的控件，采用这种方式不需要进行特殊的处理。

## 4.2 自定义 View 须知

自定义 View 过程中需要注意一些事项，如果这些问题处理不好，可能会影响 View 的正常使用和性能。

1. **让 View 支持 wrap_content** 

   在自定义 View 时，如果是直接继承自 View 或者 View Group ，并且不在 onMeasure 中对 wrap_content 做特殊处理，那么在我们使用这个自定义的 View  的 wrap_content 属性时，就无法达到预期效果，而是和使用 match_parent 属性效果一样。

2. **如果有必要，让自定义的 View 支持 padding 属性**

   在自定义 View 时，如果是直接继承自 View  ，不在 onDraw 方法中处理 padding ，那么该自定义的 View padding属性将失效；如果是直接继承自 ViewGrop 需要在 onMeasure 和 onLayout 中考虑 padding 和 margin 对其造成的影响，否则将导致自定义的控件 padding 和子元素的 margin 属性失效。

3. **尽量不要在 View 中使用 Handler ，没必要**

   View 本身内部提供了一些列的 post 方法，完全可以替代 Handler 作用。

4. **View 中如果有线程或者动画需要在特定生命周期进行停止**

   当包含此 View 的 Activity 退出或者当前 View 被 remove 掉时，View 的 onDetachedFromWindow() 方法会被调用，所以如果有需要停止的线程或者动画可以在这个方法中执行，和此方法相对应的是 onAttachedToWindow() 方法，当包含该 View 的 Activity 启动的时候，该方法就会被调用。同时当 View 变得不可见时，我们需要及时停止线程和动画，否则可能造成内存泄露。

5. **View 带有滑动嵌套情形时，需要处理好滑动冲突**

   如果有滑动冲突需要合适的进行处理。如果要处理好滑动处理可以看一下[View 事件的分发机制](http://yongyu.itscoder.com/2016/08/28/view_touchEvent_dispatch/)

## 4.3 自定义 View 示例

### 4.3.1 继承自 View 重写 onDraw 方法

这种方法一般为了实现一些不规则的效果，需要我们自己去绘制去画一个出来 View 出来，即重写 onDraw 方法，采用这种方式需要考虑 View 四周的空白即处理 padding 值，而 margin 值是受父容器控制所以不需要进行处理，并且需要注意处理自定义的 View 支持 wrap_content ，即重写 onMeasure 方法，如果不进行处理那么当在 xml 文件中使用 wrap_content 属性的时候，就相当于 match_parent 属性，这里为了更详细的说明问题，我们一起来实现一个简单的自定义 View ，只简单的画一个圆出来，这里给出关键地方的代码，先看看 onMeasure 部分代码如下：

```java
/**
 * 这里进行重写 onMeasure 方法，让自定义的 View 支持 Wrap_content 模式
 * 当使用该自定义的 View 时候，如果使用了 Wrap_content 属性后
 * 该 View  的宽和高都为 200dp ，这个尺寸在实际应用中需要根据具体需要和情况进行计算
 * 这里只是为了解释这个原理，任意给定了一个值恰巧是 200dp 而已
 * @param widthMeasureSpec 父容器给定的宽度约束条件
 * @param heightMeasureSpec 父容器给定的高度约束条件
 */
@Override
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
    super.onMeasure(widthMeasureSpec, heightMeasureSpec);
    // 在 xml 文件中使用 wrap_content 属性时，该 View 的默认宽/高值为 200dp
    int width=200;
    int height=200;
  //获取测量值和模式
    int widthMode=MeasureSpec.getMode(widthMeasureSpec);
    int heightMode=MeasureSpec.getMode(heightMeasureSpec);
    int withSize=MeasureSpec.getSize(widthMeasureSpec);
    int heightSize=MeasureSpec.getSize(heightMeasureSpec);
    if(widthMode==MeasureSpec.AT_MOST&&heightMode==MeasureSpec.AT_MOST){
        //宽和高都为 wap_content 模式，进行设定默认值
        setMeasuredDimension(width,height);
    }else if(widthMode==MeasureSpec.AT_MOST){
        //如果只有宽为 wrap_content 模式
        setMeasuredDimension(width,heightSize);
    }else if(heightMode==MeasureSpec.AT_MOST){
        //如果只有高为 wrap_content 模式
        setMeasuredDimension(withSize,height);
    }
}
```

以上代码就解决了让 自定义的 View 支持 wrap_content 的问题。下面在看看在 onDraw 方法中进行绘制的时候处理  padding 值的问题，代码如下：

```java
/**
 * 这里进行绘制 View 的内容，这里要注意需要处理 padding 值，
 * 让自定义的 View 支持 padding 属性，如果不处理，
 *那么在 xml 文件中使用该自定义的 View 的 padding
 * 属性时候，将会失效
 * @param canvas 画布
 */
@Override
protected void onDraw(Canvas canvas) {
    super.onDraw(canvas);
    //获取 view 最终宽和高
    int width=getWidth();
    int height=getHeight();
    //获取 padding 值
    int paddingLeft=getPaddingLeft();
    int paddingRight=getPaddingRight();
    int paddingTop=getPaddingTop();
    int paddingBottom=getPaddingBottom();
    //计算去掉 padding 的宽和高
    int withFinal=width-paddingLeft-paddingRight;
    int heightFinal=height-paddingTop-paddingBottom;
    //计算半径
    int radius=Math.min(withFinal/2,heightFinal/2);
    //绘制视图内容
    //确定x轴和y轴圆中心点位置，主要受 paddingLeft 和 withFinal/2 影响
    //即受左上方侧偏移量和圆半径有关，与 RightPadding 无关
    canvas.drawCircle(paddingLeft+withFinal/2,paddingTop+heightFinal/2,radius,paint);
}
```

以上代码解决了让自定义的 View 支持 padding 属性。

### 4.3.2 继承自 ViewGroup  实现特殊的 Layout 容器

这种自定义的 ViewGroup 需要处理 onMeasure 测量和 onLayout 布局两个过程，同时需要处理子元素的测量和布局过程。采用这种方法实现一个规范的自定义 View 是相当复杂的，通过前面分析的 LinearLayout 代码就可以发现，因为要考虑如何摆放子视图以及各种细节的处理，Android 开发艺术探索书中给出了一个相对规范（不完全规范）的自定义的 HorizontalScrollViewEx 视图容器，实现了一个类似 ViewPaper 的控件，内部子视图可以水平方向滑动，并且子视图的内部子元素可以实现竖直方向滑动，很显然这个控件解决了水平方向和竖直方向滑动冲突的问题，该部分知识可以看一下[View 的事件分发机制（Android 开发艺术探索读书笔记）](http://yongyu.itscoder.com/2016/08/28/view_touchEvent_dispatch/) 这篇文章从源码角度分析了事件分发机制，理解了就可以解决滑动冲突的问题。下面一起看一下关键部分代码，先看 onMeasure 部分代码如下：

```java
/**
 * 重写 onMeasure 方法，处理自定义 View 支持 wrap_content 模式
 * @param widthMeasureSpec 父容器给定宽度约束条件
 * @param heightMeasureSpec 父容器给定高度约束条件
 */
@Override
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
    super.onMeasure(widthMeasureSpec, heightMeasureSpec);
    int measuredWidth = 0;
    int measuredHeight = 0;
    //获取子视图个数
    final int childCount = getChildCount();
    //测量子视图
    measureChildren(widthMeasureSpec, heightMeasureSpec);
    //获取父容器给定测量模式和测量值
    int widthSpaceSize = MeasureSpec.getSize(widthMeasureSpec);
    int widthSpecMode = MeasureSpec.getMode(widthMeasureSpec);
    int heightSpaceSize = MeasureSpec.getSize(heightMeasureSpec);
    int heightSpecMode = MeasureSpec.getMode(heightMeasureSpec);
    if (childCount == 0) {
        //如果没有子视图直接设定 View 的宽/高为0
        setMeasuredDimension(0, 0);
    } else if (widthSpecMode == MeasureSpec.AT_MOST && heightSpecMode == MeasureSpec.AT_MOST) {
        //如果视图宽/高都采用 wrap_content 模式
        final View childView = getChildAt(0);
        //宽度为第一个视图宽度乘以所有子视图个数
        measuredWidth = childView.getMeasuredWidth() * childCount;
       // 高度为第一个视图宽度
        measuredHeight = childView.getMeasuredHeight();
        //设置 自定义视图宽/高值
        setMeasuredDimension(measuredWidth, measuredHeight);
    } else if (heightSpecMode == MeasureSpec.AT_MOST) {
        //如果只有视图高采用 wrap_content 模式
        final View childView = getChildAt(0);
        //设置视图高度为第一个视图高度
        measuredHeight = childView.getMeasuredHeight();
        setMeasuredDimension(widthSpaceSize, childView.getMeasuredHeight());
    } else if (widthSpecMode == MeasureSpec.AT_MOST) {
        //如果视图宽度使用 wrap_content 模式，设置宽度为第一个视图宽度乘以所有子视图个数
        final View childView = getChildAt(0);
        measuredWidth = childView.getMeasuredWidth() * childCount;
        setMeasuredDimension(measuredWidth, heightSpaceSize);
    }
}
```

以上代码实现了让自定义的 View 支持  wrap_content 属性。

**说明，这里为了方便处理有几处不规范的地方如下：**

1. 假设了所有子视图的高度和宽度都相等，而实际应用中这是不可能的，所以计算起来会更复杂。
2. 没有子元素的时候不应该直接设置宽/高为 0，而是应该根据 LayoutParams 的宽/高来做相应的处理，因为当使用 padding 属性的时候，虽然没有子视图，但 padding 值也会占据一定空间，你可以设置 LinearLayout 子视图个数为 0，然后给定一个 padding 值去试试。
3. 在测量 HorizontalScrollViewEx  的高/宽的时候没有考虑它的 padding 值和子视图的 margin 值，因为自己的 padding 值和子视图的 margin 值都是占据空间的。

下面再看一下 onLayout 代码如下：

```java
/**
 * 重写 onLayout 方法，实现摆放子视图功能
 */
@Override
protected void onLayout(boolean changed, int l, int t, int r, int b) {
    //记录子视图左边距位置
    int childLeft = 0;
    //获取子视图个数
    final int childCount = getChildCount();
    //记录子视图个数
    mChildrenSize = childCount;
    //遍历子视图
    for (int i = 0; i < childCount; i++) {
        //获取子视图
        final View childView = getChildAt(i);
        if (childView.getVisibility() != View.GONE) {
            //如果子视图可见，获取子视图测量宽度
            final int childWidth = childView.getMeasuredWidth();
            //记录子视图宽度
            mChildWidth = childWidth;
            //设置摆放子视图位置，每次子视图放置在上一个子视图右边依次排放
            childView.layout(childLeft, 0, childLeft + childWidth,
                    childView.getMeasuredHeight());
            childLeft += childWidth;
        }
    }
}
```

以上代码实现了摆放子视图的功能，从代码可以看出放置子视图是从左至右依次摆放。

**说明，以上代码不规范之处：**

在摆放子视图的过程中，没有考虑自身的 padding 和子视图的 margin 值。

### 4.3.3 自定义 View 的总结

到这里，关于 View 的基础知识基本学习完毕，笔者写到这里也完全不能写出了一个牛逼的自定义控件（一个基友曾经这样问我：你学完了这些知识，还不徒手撸出一个牛逼的自定义 View 啊--------[阿风](http://extremej.itscoder.com/) ，我的回答当然不能。因为自定义 View 是一个综合的知识体系，需要灵活的运用各种知识和经验，这里我们只是学习了一下基础理论知识，知其原理，懂其思路，如果我们想自定义 View，首先要掌握基本功，比如 [View 的弹性滑动](http://yongyu.itscoder.com/2016/08/14/view_to_scroll/)，[滑动冲突](http://yongyu.itscoder.com/2016/08/28/view_touchEvent_dispatch/)，绘制原理等，这些都是自定义 View 所必须知识点，再复杂的自定义 View 也是离不开这些知识点，尤其是那些看起来很炫酷的自定义 View，往往对这些技术点要求更高，只有熟悉掌握这些基础知识点以后，在面对新的自定义 View 时，才能够根据需求情况选择合适的实现思路，实现大体方法就是 4.1 节中介绍的四种分类，另外还需要学习一下 Canvas 这个类的用法才能画出想要的 View 。

最后，文章中的 Demo 会上传在 github，[链接地址](https://github.com/yongyu0102/ViewDrawDemo) ，这里只是一个简单的 Demo ，分析一下原理，（这些 Demo 源码出自 Android 开发艺术探索）如果想撸出更炫酷的 View 还需要掌握 Canvas 这个类的用法，后面需要续学习。

以上就是本次的笔记内容，如有错误，希望指出，谢谢！！！