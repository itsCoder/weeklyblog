---

 title: View 的事件分发机制（Android 开发艺术探索读书笔记）
 date:  2016-08-02 00:00：00
 categories:  Android View
 tags:  Android View
---

>-文章来源：itsCoder 的 [WeeklyBolg](https://github.com/itsCoder/weeklyblog) 项目
>- itsCoder主页：[http://itscoder.com/](http://itscoder.com/)
>- 作者：[yongyu0102](https://github.com/yongyu0102)
>- 审阅者：[HanJie](https://github.com/melodyxxx)

### 前言

在写这篇笔记的时候想了好久，也拖了好长时间，关于事件分发的博客看了很多，有的写的思路很清晰，画了事件分发的整体流程图，但是没有源码，看过之后只能知道事件是怎么分发的，但完全是记住的，而不是通过源码分析出来的，试想，如果以后再遇到其他知识点还是这样，那么我们就完全成了不能靠自己去分析问题，只能去食他人知识，没有自我学习分析能力，所以笔者试着结合艺术探索的讲解，尝试在源码的基础上加以理解，本文的写作逻辑是先从文字描述上尽量让大家先大概了解，事件分发的概况，先有个感性认识，再结合源码进行分析，如果错误的地方，还请指出。

### 1.1 点击事件的传递规则

在我们进行分析事件分发机制之前，先思考一下我们要研究哪些问题：

**1、事件分发的对象是什么？**

当用户触摸屏幕时，将创建一个 MotionEvent 对象即点击事件。MotionEvent 包含关于发生触摸的位置、时间、历史记录、手势动作等细节信息， Touch 事件相关细节被封装成了 MotionEvent 对象。理解了这一个知识点后，其实我们就很容易理解所谓点击事件的分发，其实就是对 MotionEvent 事件分发的过程，即当一个 MotionEvent 产生后，系统需要把这个事件传递给一个具体的 View 去处理, 而这个传递的过程就是分发过程。

**2、事件是在哪些对象之间传递？**

Android 与用户交互的界面就是由一系列 Activity（Fragment)、ViewGroup、View 组成如图：

​                                                 ![Activity_ViewGroup_View](https://github.com/yongyu0102/yongyu0102.github.io/blob/master/images/Activity_ViewGroup_View.png?raw=true)

当然一个界面可能由多个ViewGroup或者多个View 组成，所以事件就是在这三者之间进行传递，我们要分析的就是要捋清楚事件是由哪个对象发出，经过哪些对象，最终达到哪个对象，在某些条件改变的时候下他们之间的关系又是怎样的，理解了这些之后，当我们在遇到点击或者滑动事件冲突的情况，相信一切问题就迎刃而解了。

**3、这个分发过程由哪些对象协作完成？**

其实点击事件分发过程主要由三个重要方法共同完成：dispatchTouchEvent(MotionEvent event) 、onInterceptTouchEvent(MotionEvent event)和onTouchEvent(MotionEvent event)，下面先介绍一下这三个方法的主要作用，这样有利于后面我们对源码的分析：

1. public boolean dispatchTouchEvent(MotionEvent event) ：用来进行事件的分发。如果事件能够传递给当前view，那么此方法一定会被调用，返回结果受当前 view 的 onTouchEvent 和下级 view 的 dispatchTouchEvent 方法的影响，表示是否消耗当前事件。返回值 true，表示触摸事件被消费，已经分发出去，后续事件会继续分发到该 View；返回值 false，则表示触摸事件没有被消费，即事件没有分发出去，那么后续事件就不会继续向该 View 分发，该方法在 View 和 ViewGroup 中都有。

2. public boolean onInterceptTouchEvent(MotionEvent event)
   在 dispatchTouchEvent 方法内部调用，用来判断是否拦截某个事件，如果当前 view 拦截了某个事件，那么在同一个事件序列当中，此方法不会再被调用，返回结果表示是否拦截当前事件。只有 ViewGroup 中才有该方法 ，返回值 true，表示ViewGroup拦截了该触摸事件，该事件就不会分发给它的子 View 或者子 ViewGroup，事件会由自己的 onTouchEvent() 方法处理。返回值 false，表示 ViewGroup 没有拦截该事件，该事件就可以分发给它的子 View 和子 ViewGroup ，事件传递到子 view 的 dispatchTouchEvent() 方法中去处理。

3. public boolean onTouchEvent(MotionEvent event)
   在 dispatchTouchEvent 方法内部调用，用来处理点击事件，返回结果表示是否消耗当前事件，如果 ACTION_DOWN不消耗，则在同一个事件序列中，当前 view 无法再次接收到事件。返回值为 True ，事件由自己处理，后续事件序列让其处理；返回值为 False ，自己不消耗事件，向上返回让其他的父容器的onTouchEvent接受处理。

   这三个方法都是通过 dispatchTouchEvent() 联系在一起，他们之间的关系可以用如下伪代码表示：
``` java
public boolean dispatchTouchEvent(MotionEvent ev) {
  //表示事件是否被分发出去（消耗），默认为false，即默认不拦截
    boolean consume = false;
    if (onInterceptTouchEvent(ev)) {
      //如果拦截了，那么事件交给当前view的onTouchEvent(ev)
      //方法进行处理，返回值即onTouchEvent(ev)的结果
        consume = onTouchEvent(ev);
    } else {
     //如果不拦截，那么事件交给子view的dispatchTouchEvent(ev)
      //方法进行处理，返回值即是dispatchTouchEvent(ev)事件
      //处理结果
        consume = child.dispatchTouchEvent(ev);
    }
    return consume;
}
```

结论：对于一个根 ViewGroup，点击事件产生后，首先会传递给它，这时 ViewGroup 的 dispatchTouchEvent 会调用，而在 dispatchTouchEvent 方法中会调用 onInterceptTouchEvent() 方法，如果它的 onInterceptTouchEvent 返回 true 表示要拦截当前事件，接下来事件会交给这个 ViewGroup 处理，此时 ViewGroup 的 onTouchEvent 方法就会被调用，如果这个ViewGroup 的 onInterceptTouchEvent 返回 false，则事件会继续传递给子元素，子元素的 dispatchTouchEvent 会调用，如此反复直到事件被处理。

当一个View需要处理事件时，如果设置了 OnTouchListener ，那么 OnTouchListener 的 onTouch方法会回调，如果 onTouch 返回 false，则当前 View 的 onTouchEvent 方法会被调用；如果返回 true，那么 onTouchEvent 方法将不会调用。由此可见，OnTouchListener 优先级高于 onTouchEvent。OnClickListener 优先级处在事件传递的尾端。

**4、事件的传递顺序是什么？**

这里就直接给出答案:一个点击事件产生后，传递顺序：Activity -> Window -> ViewGroup  -> View；如果一个 View 的 onTouchEvent 返回 false 即 View 没有处理事件，那么它的父容器的onTouchEvent 会被调用，以此类推，所有元素都不处理该事件，最终将传递给 Activity 处理，即 Activity 的 onTouchEvent 会被调用，事件顺序为：View ->  ViewGroup -> Window ->  Activity。

5、关于事件传递的其他一些结论这里先给出，以便我们更好的理解时间传递机制，结论如下：

1. 同一个事件序列是指从手指触摸屏幕那一刻开始，中间包含数量不定的 move 事件到手指离开屏幕那一刻（down->move...move->up)。
2. 正常情况下一个事件序列只能被一个 View 拦截且消耗，每个 View  一旦决定拦截，同一个事件序列所有事件都会直接交给它处理，并且它的 onInterceptTouchEvent 不会再被调用。
3. 某个 View 一旦开始处理事件，如果它不消耗 ACTION_DOWN（ onTouchEvent  返回了 false ），那么同一事件序列中其他事件都不会再交给它来处理，事件将重新交给他的父元素处理，即父元素的 onTouchEvent 会被调用。
4. 如果某个 View 不消耗除  ACTION_DOWN 以外的其他事件，那么这个点击事件会消失，此时父元素的 onTouchEvent 并不会被调用，并且当前 View 可以收到后续事件，最终这些消失的点击事件会传递给 Activity 处理。
5. ViewGroup 默认不拦截任何事件，ViewGroup 的 onInterceptTouchEvent 方法默认返回 false。
6. View 没有 onInterceptTouchEvent 方法，一旦有事件传递给它，那么它的 onTouchEvent 方法就会被调用。
7. View 的 onTouchEvent 方法默认消耗事件（返回 true ），除非他是不可点击的（ clickable 和 longClickable 同时为 false ）。View 的 longClickable 属性默认都为 false ，clickable 属性分情况，Button 默认为 true，TextView 默认为 false。
8. onClick 发生的前提是View可点击，并且它收到了down 和 up 事件。
9. 事件传递过程是由外而内，事件总是先传递给父元素，然后在由父元素分发给子 View，通过requestDisallowInterceptTouchEvent 方法可以在子元素干预父元素的事件分发过程，但 ACTION_DOWN 事件除外。                

### 1.2 事件分发的源码解析

这里要分别分析 Activity  、ViewGroup 和 View 对事件的分发。

####  1. Activity 对点击事件的分发过程

   由于平时我们对 Activity 事件分发接触不是很多（笔者是这样），使用应该不是很多，所以这里只做简单介绍。

   (1) Activity 中与触摸事件相关API主要是 dispatchTouchEvent() 和 onTouchEvent()。dispatchTouchEvent() 是传递触摸事件的API，而 onTouchEvent() 则是 Activity 处理触摸事件的API。

   (2) Activity 中的 dispatchTouchEven 会将触摸事件传递给Activity 所包含的视图。具体的实现方式在通过调用到 Activity 所属 Window 的  superDispatchTouchEvent，进而调用到 Window 的 DecorView 的 superDispatchTouchEvent，进一步又调用到 ViewGroup 的 dispatchTouchEvent() ，这样事件就从 Activity 传递到了 ViewGroup 。
   如果 Activity 所包含的视图拦截或者消费了该触摸事件的话，就不会再执行 Activity 的 onTouchEvent() ；
   如果 Activity 所包含的视图没有拦截或者消费该触摸事件的话，则会执行 Activity 的 onTouchEvent() 。
   (3) Activity 中的 onTouchEvent 是 Activity 自身对触摸事件的处理。如果该 Activity 的 android:windowCloseOnTouchOutside 属性为 true，并且当前触摸事件是 ACTION_DOWN ，而且该触摸事件的坐标在 Activity 之外，同时 Activity 还包含了视图的话就会导致 Activity 被结束。

#### 2. View 对点击事件的处理过程

   这里的 View 不包含 ViewGroup ，这里说下为什么先分析 View 而不是像书上那样先分析 ViewGroup ，因为在 ViewGroup 的事件分发过程中会调用到 View 对事件的处理，而且 View 的事件处理相比 ViewGroup 而言简单些，所以这里先分析 View 。

   我们来看一下 View 中 dispatchTouchEvent 方法的源码：

```java
    public boolean dispatchTouchEvent (MotionEvent event){
           .........
           //标记处理结果是否成功
           boolean result = false;
           ...........
           if (onFilterTouchEventForSecurity(event)) {
               ListenerInfo li = mListenerInfo;
               // 如果 mListenerInfo 不为 null，且点击的控件是否是 enable 的
               //那么就会调用 mOnTouchListener.onTouch(this, event) 方法
               if (li != null && li.mOnTouchListener != null
                       && (mViewFlags & ENABLED_MASK) == ENABLED
                       && li.mOnTouchListener.onTouch(this, event)) {
                   //如果 mOnTouchListener.onTouch(this, event)
                   //方法返回值为 Ture
                   //那么 设置 result = true;即 dispatchTouchEvent 
                   //方法返回值为 true
                   result = true;
           }
           //如果 result 为 false ，则调用 onTouchEvent(event)方法
           //也就是如果 mOnTouchListener.onTouch(this, event)方法返回 true
           //那么 onTouchEvent(event)方法就不会调用
           if (!result && onTouchEvent(event)) {
               //如果 onTouchEvent(event) 返回值为 true ，
               //那么 dispatchTouchEvent 方法返回 true
               result = true;
           }
       }
   .............
       //返回处理结果
       return result;
   }
```

   从上面源码的10行代码可以看出，首先会判断 mOnTouchListener 是否为空，而mOnTouchListener 是在 setOnTouchListener 方法里赋值的，也就是说只要我们给控件注册了touch 事件，mOnTouchListener 就一定不为空，而(mViewFlags & ENABLED_MASK) == ENABLED 是判断当前点击的控件是否是 enable 的，按钮默认都是 enable 的，因此这个条件恒定为 true。mOnTouchListener.onTouch(this, event)，其实也就是去回调控件注册 touch 事件时的 onTouch 方法。也就是说如果我们在 onTouch 方法里返回 true，就会让这三个条件全部成立，从而整个方法返回 true。在结合22行代码 if (!result && onTouchEvent(event))，可以得出结论，如果我们在 onTouch 方法里返回 false，就会再去执行 onTouchEvent(event)方法，如果 onTouch 方法里返回 true ，onTouchEvent(event) 方法将不会执行, 可见 OnTouchListener 优先级高于 onTouchEvent(event) 方法。
   接下来我们分析一下 onTouchEvent(event) 方法的源码：

```java
                  public boolean onTouchEvent(MotionEvent event) {   
   			       final float x = event.getX();
                   final float y = event.getY();
                   final int viewFlags = mViewFlags;
                   final int action = event.getAction();
               // 如果是不可用状态下
                   if ((viewFlags & ENABLED_MASK) == DISABLED) {
                       if (action == MotionEvent.ACTION_UP && (mPrivateFlags & PFLAG_PRESSED) != 0) {
                           setPressed(false);
                       }
                       // A disabled view that is clickable
                       //still consumes the touch
                       // events, it just doesn't respond to them.
                       //只要 view 是 clickable 那么仍然会消费事件，
                       //返回值为 true，
                       //只不过不响应事件
                       return (((viewFlags & CLICKABLE) == CLICKABLE
                               || (viewFlags & LONG_CLICKABLE) == LONG_CLICKABLE)
                               || (viewFlags & CONTEXT_CLICKABLE) == CONTEXT_CLICKABLE);
                   }

                   if (mTouchDelegate != null) {
                       if (mTouchDelegate.onTouchEvent(event)) {
                           return true;
                       }
                   }
                   // 如果 view 是 CLICKABLE 或者 LONG_CLICKABLE，
                   //即 view 是 clickable 状态
                   // 就会进入以下方法的 switch 语句 ACTION_MOVE、ACTION_UP、
                   // ACTION_DOWN、ACTION_CANCEL、ACTION_CANCEL
                   //各种状态的处理
                   //无论处于那种状态，最终返回结果都是 true
                   if (((viewFlags & CLICKABLE) == CLICKABLE ||
                           (viewFlags & LONG_CLICKABLE) == LONG_CLICKABLE) ||(viewFlags & CONTEXT_CLICKABLE) == CONTEXT_CLICKABLE) {
                       switch (action) {
                           case MotionEvent.ACTION_UP:
                               boolean prepressed = (mPrivateFlags & PFLAG_PREPRESSED) != 0;
                               if ((mPrivateFlags & PFLAG_PRESSED) != 0 || prepresse
                                   boolean focusTaken = false;
                                   if (isFocusable() && isFocusableInTouchMode() && !isFocused()) {
                                       focusTaken = requestFocus();
                                   }

                                   if (prepressed) {
                                       setPressed(true, x, y);
                                   }

                                   if (!mHasPerformedLongPress && !mIgnoreNextUpEvent) {
                                       removeLongPressCallback();
                                       if (!focusTaken) {
                                           if (mPerformClick == null) {
                                               mPerformClick = new PerformClick();
                                           }
                                           if (!post(mPerformClick)) {
                                               performClick();
                                           }
                                       }
                                   }

                                   if (mUnsetPressedState == null) {
                                       mUnsetPressedState = new UnsetPressedState();
                                   }

                                   if (prepressed) {
                                       postDelayed(mUnsetPressedState,
                                               ViewConfiguration.getPressedStateDuration());
                                   } else if (!post(mUnsetPressedState)) {
                                       mUnsetPressedState.run();
                                   }

                                   removeTapCallback();
                               }
                               mIgnoreNextUpEvent = false;
                               break;

                           case MotionEvent.ACTION_DOWN:
                               mHasPerformedLongPress = false;

                               if (performButtonActionOnTouchDown(event)) {
                                   break;
                               }
                               boolean isInScrollingContainer = isInScrollingContainer();

                               if (isInScrollingContainer) {
                                   mPrivateFlags |= PFLAG_PREPRESSED;
                                   if (mPendingCheckForTap == null) {
                                       mPendingCheckForTap = new CheckForTap();
                                   }
                                   mPendingCheckForTap.x = event.getX();
                                   mPendingCheckForTap.y = event.getY();
                                   postDelayed(mPendingCheckForTap, ViewConfiguration.getTapTimeout());
                               } else {
                                   setPressed(true, x, y);
                                   checkForLongClick(0);
                               }
                               break;

                           case MotionEvent.ACTION_CANCEL:
                               setPressed(false);
                               removeTapCallback();
                               removeLongPressCallback();
                               mInContextButtonPress = false;
                               mHasPerformedLongPress = false;
                               mIgnoreNextUpEvent = false;
                               break;

                           case MotionEvent.ACTION_MOVE:
                               drawableHotspotChanged(x, y);
                               if (!pointInView(x, y, mTouchSlop)) {
                                   // Outside button
                                   removeTapCallback();
                                   if ((mPrivateFlags & PFLAG_PRESSED) != 0) {
                                       removeLongPressCallback();

                                       setPressed(false);
                                   }
                               } 
                               break;
                       }
                   //如果该控件是可以点击的就会进入 switch 判断中去，
                   //最终返回值为 true ，即 onTouchEvent
                       //返回值为 true 。
                       return true;
                   }
               //如果该控件是不可点击的返回 false ，如 TextView
                   return false;
               }
```

   先看第7行代码 if ((viewFlags & ENABLED_MASK) == DISABLED) 这个条件是判断 view 是不可用状态，而在该条件下我们看第17行代码 如下：

```java
                             return (((viewFlags & CLICKABLE) == CLICKABLE || (viewFlags & LONG_CLICKABLE) == LONG_CLICKABLE)
                              || (viewFlags & CONTEXT_CLICKABLE) == CONTEXT_CLICKABLE);
```

   这个返回结果是判断只要 view 是可点击状态 那么返回值就问 true ，即如果  view 是不可用状态，但只要 view 是可点击状态，就会消耗点击事件，只不过不响应结果。
   再看第33行会判断只要 view 是可点击的，那么就会进入到下面的 switch 语句，最终在第123行返回 true，即只要 view 是可点击的，那么 ontouchEvent（Event v) 方法就会返回 true 消费点击事件。
   然后在 ACTION_UP 事件触发的时候，在55行代码会执行 performClick() 方法，我们看一下这个方法的实现：

```java
          public boolean performClick() {
                    final boolean result;
                    final ListenerInfo li = mListenerInfo;
                    //如果 mOnClickListener 不为null，就会调用
                    // mOnClickListener.onClick(this) 方法
                    if (li != null && li.mOnClickListener != null) {
                        playSoundEffect(SoundEffectConstants.CLICK);
                        //这个方法就是 回调
                        //我们给 view 设置的 view.setOnClickListener()方法
                        li.mOnClickListener.onClick(this);
                        result = true;
                    } else {
                        result = false;
                    }
             sendAccessibilityEvent(AccessibilityEvent.TYPE_VIEW_CLICKED);
               return result;
               }
```

   上面第6行代码 if (li != null && li.mOnClickListener != null) 判断如果给当前 view 设置了点击事件，那么就会执行回调设置的 onClick() 方法即 mOnClickListener.onClick(this)，这样 view 的 dispatchTouchEvent(MotionEvent event) 这里就分析结束了！

#### 3 ViewGroup 对点击事件的处理过程

   同样我们来看一下 ViewGroup 中 dispatchTouchEvent 方法的源码：

```java
    public boolean dispatchTouchEvent(MotionEvent ev) {
      ............
    //标记是否分发成功
    boolean handled = false;
    // 第1步：检测是否要分发该触摸事件
    // 如果该 View 不是位于顶部，并且有设置属性使该 View 不在顶部时不响应触摸事件
    if (onFilterTouchEventForSecurity(ev)) {
        final int action = ev.getAction();
        final int actionMasked = action & MotionEvent.ACTION_MASK;

        if (actionMasked == MotionEvent.ACTION_DOWN) {
            // 如果是 ACTION_DOWN(即按下事件)，则清空之前的触摸事件处理目标和状态。
            cancelAndClearTouchTargets(ev);
            resetTouchState();
        }

        // 标记 ViewGroup 是否要拦截事件
        final boolean intercepted;
      //第2步，判断是否拦截事件
        // 如果是 ACTION_DOWN  或者 mFirstTouchTarget不为null , 
      //记住 mFirstTouchTarget 在事件分发到
        // 子 view 的时候会被负值， 即一旦 事件传递到子 view 后 mFirstTouchTarget 不为 null ,而
        // 一旦在 ACTION_DOWN 时候拦截了事件，那么 mFirstTouchTarget=null ;此时 intercepted=true
        //进行拦截，这就是前面说的，一旦在 ACTION_DOWN 时候拦截了事件，那么后续事件将由 ViewGroup
        //处理，不分发到子 view 。
        if (actionMasked == MotionEvent.ACTION_DOWN
                || mFirstTouchTarget != null) {
            // 检查禁止拦截标记：FLAG_DISALLOW_INTERCEPT
            // 如果调用了子 view 调用了 requestDisallowInterceptTouchEvent()标记的话，
          //则FLAG_DISALLOW_INTERCEPT会为true。
            //意思是禁止父类进行拦截事件
            final boolean disallowIntercept = (mGroupFlags & FLAG_DISALLOW_INTERCEPT) != 0;
            if (!disallowIntercept) {
                // 如果禁止拦截标示为 false ,那么调用 ViewGroup 的onInterceptTouchEvent(ev) 方法
                //获取是否拦截标记 ，onInterceptTouchEvent(ev) 返回值默认为 false
                intercepted = onInterceptTouchEvent(ev);
                ev.setAction(action); // restore action in case it was changed
            } else {
                //如果子 view 调用了requestDisallowInterceptTouchEvent() ，
              //那么 !disallowIntercept 为false ，此时 intercepted=false，
              //ViewGroup 的 onInterceptTouchEvent(ev) 将失效
                intercepted = false;
            }
        } else {
            //这就是上面提到的 一旦在 ACTION_DOWN 时候拦截了事件，那么 mFirstTouchTarget=null
            //此时  intercepted = true; 继续拦截
            intercepted = true;
        }

        ..............

        TouchTarget newTouchTarget = null;
        //  标记已经将事件是否发生到目标 view
        boolean alreadyDispatchedToNewTouchTarget = false;
        if (!canceled && !intercepted) {
            View childWithAccessibilityFocus = ev.isTargetAccessibilityFocus()
                    ? findChildWithAccessibilityFocus() : null;
             //第3步，如果不拦截，不取消，进行事件分发
            // 在 ACTION_DOWN、ACTION_POINTER_DOWN 或者 ACTION_HOVER_MOVE 动作的时候执行
            // 在MotionEvent.ACTION_MOVE 、MotionEvent.ACTION_UP 动作的时候不会执行以下代码
            //会从下面 第7步开始执行
          //if (mFirstTouchTarget == null) 行代码执行，而如果从ACTION_DOWN开始没有事件传
          //递到子 view ，那么if (mFirstTouchTarget == null) 条件就会成立，
          // 否则会执行 while (target != null) ， 
          //直接遍历mFirstTouchTarget链表，查找之前接受ACTION_DOWN的子view，
            // 并将触摸事件分配给这些子 view 。也就是前面说的说，
          //如果ViewGroup的某个孩子没有接受ACTION_DOWN事件；
            // 那么，ACTION_MOVE和ACTION_UP等事件也一定不会分发给这个孩子！
            if (actionMasked == MotionEvent.ACTION_DOWN
                    || (split && actionMasked == MotionEvent.ACTION_POINTER_DOWN)
                    || actionMasked == MotionEvent.ACTION_HOVER_MOVE) {
                final int actionIndex = ev.getActionIndex(); // always 0 for down
                final int idBitsToAssign = split ? 1 << ev.getPointerId(actionIndex)
                        : TouchTarget.ALL_POINTER_IDS;

                // Clean up earlier touch targets for this pointer id in case they
                // have become out of sync.
                removePointersFromTouchTargets(idBitsToAssign);

                final int childrenCount = mChildrenCount;
                if (newTouchTarget == null && childrenCount != 0) {
                    final float x = ev.getX(actionIndex);
                    final float y = ev.getY(actionIndex);
                    // Find a child that can receive the event.
                    // Scan children from front to back.
                    final ArrayList<View> preorderedList = buildOrderedChildList();
                    final boolean customOrder = preorderedList == null
                            && isChildrenDrawingOrderEnabled();
                       // 获取该 ViewGroup 包含的子 View 和 ViewGroup 
                    final View[] children = mChildren;
                    // 第4步，遍历 ViewGroup的 孩子，对触摸事件进行分发。
                    for (int i = childrenCount - 1; i >= 0; i--) {
                        final int childIndex = customOrder
                                ? getChildDrawingOrder(childrenCount, i) : i;
                        final View child = (preorderedList == null)
                                ? children[childIndex] : preorderedList.get(childIndex);
                        if (childWithAccessibilityFocus != null) {
                            if (childWithAccessibilityFocus != child) {
                                continue;
                            }
                            childWithAccessibilityFocus = null;
                            i = childrenCount - 1;
                        }
                        //判断子 View 是否可以接受事件
                        if (!canViewReceivePointerEvents(child)
                                || !isTransformedTouchPointInView(x, y, child, null)) {
                            ev.setTargetAccessibilityFocus(false);
                            continue;
                        }
                        // getTouchTarget() 的作用是查找 child 是否存在于
                      //mFirstTouchTarget 的单链表中。
                        // 是的话，返回对应的 TouchTarge t对象；跳出循环
                        newTouchTarget = getTouchTarget(child);
                        if (newTouchTarget != null) {
                            // Child is already receiving touch within its bounds.
                            // Give it the new pointer in addition to the ones it is handling.
                            newTouchTarget.pointerIdBits |= idBitsToAssign;
                            break;
                        }

                        resetCancelNextUpFlag(child);
                        // 第5步，真正开始执行事件分发
                      //调用 dispatchTransformedTouchEvent() 执行触摸事件分发给 child。
                        if (dispatchTransformedTouchEvent(ev, false, child, idBitsToAssign)) {
                            // Child wants to receive touch within its bounds.
                            mLastTouchDownTime = ev.getDownTime();
                            if (preorderedList != null) {
                                // childIndex points into presorted list, find original index
                                for (int j = 0; j < childrenCount; j++) {
                                    if (children[childIndex] == mChildren[j]) {
                                        mLastTouchDownIndex = j;
                                        break;
                                    }
                                }
                            } else {
                                mLastTouchDownIndex = childIndex;
                            }
                            mLastTouchDownX = ev.getX();
                            mLastTouchDownY = ev.getY();
                               
                            //第6步，如果事件分发成功  调用 addTouchTarget()
                          //将 child 添加到 mFirstTouchTarget 链表中，
                            // 并返回表头对应的 TouchTarge ,mFirstTouchTarget 将被负值
                            newTouchTarget = addTouchTarget(child, idBitsToAssign);
                            // 分发成功 设置 alreadyDispatchedToNewTouchTarget 
                          //标记为 true 跳出循环
                            alreadyDispatchedToNewTouchTarget = true;
                            break;
                        }

                        // The accessibility focus didn't handle the event, so clear
                        // the flag and do a normal dispatch to all children.
                        ev.setTargetAccessibilityFocus(false);
                    }
                    if (preorderedList != null) preorderedList.clear();
                }

                if (newTouchTarget == null && mFirstTouchTarget != null) {
                    // Did not find a child to receive the event.
                    // Assign the pointer to the least recently added target.
                    newTouchTarget = mFirstTouchTarget;
                    while (newTouchTarget.next != null) {
                        newTouchTarget = newTouchTarget.next;
                    }
                    newTouchTarget.pointerIdBits |= idBitsToAssign;
                }
            }
        }

        // 第7步，进一步的对触摸事件进行分发，如果mFirstTouchTarget为null，
        // 意味着还没有任何View来接受该触摸事件；
        if (mFirstTouchTarget == null) {
            // 调用 ViewGroup 的父类，即 view 的dispatchTouchEvent(MotionEvent ev)方法处理
          //处理方法就是我们上面分析的 view 对点击事件的分发流程
            // 并将分发结果的返回值赋值给 handler 进行返回
            handled = dispatchTransformedTouchEvent(ev, canceled, null,
                    TouchTarget.ALL_POINTER_IDS);
        } else {
            // Dispatch to touch targets, excluding the new touch target if we already
            // dispatched to it.  Cancel touch targets if necessary.
            TouchTarget predecessor = null;
            TouchTarget target = mFirstTouchTarget;
            // 如果mFirstTouchTarget ！= null，则循环从链表 mFirstTouchTarget 中取出 target
            while (target != null) {
                final TouchTarget next = target.next;
                // 如果 alreadyDispatchedToNewTouchTarget 为 true
                // 并且target == newTouchTarget 即事件(ACTION_DOWN)已经分发成功设置 handled=true
                // 返回事件分发成功
                if (alreadyDispatchedToNewTouchTarget && target == newTouchTarget) {
                    handled = true;
                } else {
                    final boolean cancelChild = resetCancelNextUpFlag(target.child)
                            || intercepted;
                    // 如果 alreadyDispatchedToNewTouchTarget && target == newTouchTarget 条件
                    //不成立，说明该事件还没有进行分发(ACTION_MOVE、ACTION_UP）
                  //那么将此事件分发给之前已经成功接收事件的子
                    // view ，即 target.child ，并将执行 handled = true 表示事件分发结果成功
                    if (dispatchTransformedTouchEvent(ev, cancelChild,
                            target.child, target.pointerIdBits)) {
                        handled = true;
                    }
                    ..........
                    //返回事件处理结果
                    return handled;
                }
```

事件分发主要分为以上几个步骤，代码中已经标出，下面加以总结：

在上面执行第2步的时候，ViewGroup 在两种情况下会判断是否拦截事件即 事件类型为 ACTION_DOWN 和 mFirstTouchTarget ！= null 条件下， ACTION_DOWN  好理解，那么 mFirstTouchTarget ！= null  是什么意思呢，我们看一下在执行第5步，事件由 ViewGroup 子 View 处理成功的时候有一行代码是：newTouchTarget = addTouchTarget(child, idBitsToAssign);这行代码实现方法如下：

```java
     private TouchTarget addTouchTarget(View child, int pointerIdBits) {
    TouchTarget target = TouchTarget.obtain(child, pointerIdBits);
    target.next = mFirstTouchTarget;
    mFirstTouchTarget = target;
    return target;
}
```
通过上面代码可以看出，事件由 ViewGroup 子 View 处理成功的时候 mFirstTouchTarget 将会被赋值，即事件分发到子 view 的时候mFirstTouchTarget ！= null，反过来如果事件由 ViewGroup 进行拦截，那么mFirstTouchTarget ！= null 就不成立，那么当 ACTION_MOVE 和 ACIONT_UP 事件到来的时候  if (actionMasked == MotionEvent.ACTION_DOWN    || mFirstTouchTarget != null) 条件为 false，将导致 ViewGroup 的 onInterceptTouchEvent(ev) 方法不再调用，同一系列事件中的其他事件将由 ViewGroup 进行处理。当然有一种特殊情况，就是 FLAG_DISALLOW_INTERCEPT 这个标记位，这个标记位是子 view 调用了 requestDisallowInterceptTouchEvent() 来进行设置，ViewGroup 将无法进行拦截除了 ACTION_DOWN 以外的其他事件，这是因为在进行 ACTION_DOWN 的时候，会重置 FLAG_DISALLOW_INTERCEPT  这个标记位，这样导致子 view 设置的 requestDisallowInterceptTouchEvent() 失效。

在看一下第5步，是如何调用 dispatchTransformedTouchEvent(ev, false, child, idBitsToAssign)  方法进行事件分发的，这行代码实现方法如下：

```java
private boolean dispatchTransformedTouchEvent(MotionEvent event, boolean cancel,
        View child, int desiredPointerIdBits) {
    final boolean handled;

    // Canceling motions is a special case.  We don't need to perform any transformations
    // or filtering.  The important part is the action, not the contents.
    final int oldAction = event.getAction();
    if (cancel || oldAction == MotionEvent.ACTION_CANCEL) {
        event.setAction(MotionEvent.ACTION_CANCEL);
        if (child == null) {
            handled = super.dispatchTouchEvent(event);
        } else {
            handled = child.dispatchTouchEvent(event);
        }
        event.setAction(oldAction);
        return handled;
    }
    .......
    }
```

我们只看与我们要分析目标相关的代码，上面代码中我们看到，如果 child!=null ，那么执行 handled = child.dispatchTouchEvent(event) 之后的逻辑就是前面分析的 view 的事件分发，这样就完成了事件分发到子 view 的流程，我们看 child.dispatchTouchEvent(event) 是由返回值的，即如果返回 true ，事件由子 view 消费，事件分发成功，跳出第4步 for 循环，停止遍历子 view。

我们再看一下第7步，如果 mFirstTouchTarget == null ，说明没有任何子 view 接受触摸事件，那么调用 handled = dispatchTransformedTouchEvent(ev, canceled, null, TouchTarget.ALL_POINTER_IDS) 注意这里第三个参数传入 null，就是我们上面分析的另一种情况 ，如果 child == null ，执行 handled = super.dispatchTouchEvent(event) ，而 ViewGroup 的父类是 view ，所以之后的处理逻辑又和前面说的 view 一样了，即 ViewGroup 的 onTouch 方法会得到执行 ，而如果 mFirstTouchTarget ！= null ，表明已经有事件分发成功了，那么会执行  while (target != null) 循环从从链表 mFirstTouchTarget 中取出 target，条件 if (alreadyDispatchedToNewTouchTarget && target == newTouchTarget)  如果成立，那么说明这个事件是是已经分发过成功的事件，那么直接执行 handled = true ，即让 dispatchTouchEvent 方法返回 true， 表明事件分发成功，而如果 if (alreadyDispatchedToNewTouchTarget && target == newTouchTarget)  不成立，表明该事件是新到来的事件还没进行分发，那么执行 dispatchTransformedTouchEvent(ev, cancelChild,  target.child, target.pointerIdBits)) ，进一步对事件进行分发，并将 分发处理结果进行返回，这这情况下就是前面 第3步所说，不执行第三步，直接执行第7步对事件进一步就行分发。

最后说了这么多，大家可以结合下面这张流程图来对整理分发流程进行梳理，图片引自：

[图解 Android 事件分发机制](http://www.jianshu.com/p/e99b5e8bd67b)  

![TouchEvent_disaptch](https://github.com/yongyu0102/yongyu0102.github.io/blob/master/images/TouchEvent_disaptch.png?raw=true)

其中 super 表示调用关系，true 表示消费事件， false 表示没有消费事件。

到这里关于 View 的事件分发分析就结束了！

如果感觉笔者写的不好，可以参考以下博客：

[郭神的 Android事件分发机制完全解析，带你从源码的角度彻底理解(下)](http://blog.csdn.net/guolin_blog/article/details/9153747)

[skywangkw 的Android 触摸事件机制(四) ViewGroup中触摸事件详解](http://wangkuiwu.github.io/2015/01/04/TouchEvent-ViewGroup/)



