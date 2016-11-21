## 自定义选择复制功能


### 写在前面

先来个段子：

> 刚工作时遇到一个特别难搞定的需求，当时没做出来，感到很羞耻。过了几年，再一次遇到这个需求，还是没做出来，只是不再感到羞耻了。

在我刚开始工作的时候，也有过一次这样的经历。当时项目中有个需求，让 TextView 中的文本可以选择复制，正常来讲，应该是很容易实现的，直接按照下面的设置就可以了：

```java
mTextView.setTextIsSelectable(true);
```
但是，这个简单的实现并不是完美的，主要有几个问题：

- **不同版本选择复制样式不统一**：在原生系统上 6.0 之前和之后的操作样式是不同的，这里不得不说，6.0 以下的这个选择复制操作交互很不合理，且对应用的界面侵入太多。

  ![](http://ac-QYgvX1CC.clouddn.com/d50f9abab0429d5c.png)

- **万恶的国产 ROM 问题**：当时公司测试同事提 bug 反馈，在 vivo 手机上这么设置，长按之后并没有效果。（再一次吐槽乱改系统的国产 ROM，这也是为什么 Android 开发比起 iOS 费事费力的原因之一）

- **可定制性不高**：如果仅仅是一个选择复制的功能，不考虑以上两个问题，还能凑合搞定，但是假如多个需求，选中文字之后直接进行某个操作，比如收藏、发送给好友，此时原生的选择复制功能可能就不足以胜任了。

以上说了这么多，问题的解决办法就是：自己写一个选择复制的功能，这样以上三个问题都能很好地解决了。

看起来很容易，但是对于当时刚刚入门的我来说，这是个完全没头绪的任务。

时隔一年之后，再遇到这个需求，这次通过 Google、GitHub ，以及参考 API 23 中 TextView 源码，基本上实现了自定义选择复制的功能，效果如下：

![](http://ac-QYgvX1CC.clouddn.com/378d52583767882d.png)

保证所有的平台上显示效果一致，弹出的操作菜单可以自己定制，并设置相应的操作。

### 实现要求和要点

在开始具体的实现之前，先确定下实现的要求：

- 尽可能保证和 Android 6.0 原生选择复制一样的交互和基础功能
- 尽可能不需要侵入太多，为了实现选择复制功能，重新自定义 TextView 的方式是不够优雅的，特别是考虑到项目中本来就已经使用了自定义的 TextView ，一旦需求变更，改动成本很大
- 可用的自定义配置

本文最终实现的使用方式如下所示，均满足以上的实现要求：

```java
mSelectableTextHelper = new SelectableTextHelper.Builder(mTvTest)
    .setSelectedColor(getResources().getColor(R.color.selected_blue))
    .setCursorHandleSizeInDp(20)
    .setCursorHandleColor(getResources().getColor(R.color.cursor_handle_color))
    .build();
```

整个自定义的选择复制功能视图上主要有三个部分：

![](http://ac-QYgvX1CC.clouddn.com/ea5c7296983d4e68.png)

- 选择游标
- 选中的文本
- 操作框

在具体实现中有以下要点：

- 自定义选择游标，可以拖动定位选中文本
- 文本的选中状态
- 操作框的显示，以及对应操作的处理
- 在可滑动布局中的特殊处理，例如在 ScrollView 中，当视图滚动时隐藏或者移动选择游标，隐藏操作框，停止滑动时重新显示选择游标和操作框
- 选中文本后，点击 TextView 取消选择


### 实现思路

在开始实践之前，查找资料是少不了的，首先找到了 [记划词模块重构感受\|开源实验室\-张涛](http://kymjs.com/code/2016/08/13/01) 这篇文章，但是这篇文章中更多是提供了一个改进某个开源项目的思路，并没有给出具体的代码，而且连那个开源项目也没给出地址。

后来通过搜索关键字，找到了那个开源项目：
[zhouray/SelectableTextView](https://github.com/zhouray/SelectableTextView)

如张涛吐槽的那样，这个项目的实现确实不够优雅，主要存在两个问题：

- 自定义 TextView 实现的，侵入太多
- 解决嵌套在滑动布局中的处理太简单粗暴，竟然自定义了一个 ScrollView 来处理，应用到实际场景中是存在问题的

如果你有时间可以看一下这个项目的代码，在本文后面的实现中，也部分参考了该项目。

参考上面提到的文章和开源项目，实现思路基本确定了：

- 选择游标使用 PopupWindow 实现，并重写 Touch 事件处理逻辑，实现拖动定位选择文本
- 选中文本使用 `BackgroundColorSpan` 来显示，比较简单
- 操作框同样使用 PopupWindow 实现，重点是处理好显示的位置

大致的思路确定，接下来就是具体的实现了。

### 具体实现过程

自定义的选择复制类取名为 `SelectableTextHelper`，其有一个字段 `mTextView`，持有需要设置选择复制的 `TextView` 对象。

#### 初步设置

由于 `TextView` 的文本的 `BufferType` 类型是 `SPANNABLE` 时才可以设置 Span ，实现选中的效果，因此在一开始先给 TextView 设置下：

```java
mTextView.setText(mTextView.getText(), TextView.BufferType.SPANNABLE);

```

接下来给 TextView 设置相关的点击、长按、Touch 事件：

```java
    mTextView.setOnLongClickListener(new View.OnLongClickListener() {
        @Override
        public boolean onLongClick(View v) {
            showSelectView(mTouchX, mTouchY);
            return true;
        }
    });
    
    mTextView.setOnTouchListener(new View.OnTouchListener() {
        @Override
        public boolean onTouch(View v, MotionEvent event) {
            mTouchX = (int) event.getX();
            mTouchY = (int) event.getY();
            return false;
        }
    });
    
    mTextView.setOnClickListener(new View.OnClickListener() {
        @Override
        public void onClick(View v) {
            resetSelectionInfo();
            hideSelectView();
        }
    });

```

- 其中 `onTouch()` 记录了触摸点坐标，用于后面的选择文本的位置定位以及选择游标的显示，即传递给 `showSelectView()` 方法。
- `onClick()` 中的处理比较简单，重置选中文本信息、隐藏选择相关的 View 。

直接看一下 `showSelectView()` 和 `hideSelectView()` 的实现：

#### 显示选择相关组件

```java
private void showSelectView(int x, int y) {
    hideSelectView();
    resetSelectionInfo();
    isHide = false;
    if (mStartHandle == null) mStartHandle = new CursorHandle(true);
    if (mEndHandle == null) mEndHandle = new CursorHandle(false);
    int startOffset = TextLayoutUtil.getPreciseOffset(mTextView, x, y);
    int endOffset = startOffset + DEFAULT_SELECTION_LENGTH;
    if (mTextView.getText() instanceof Spannable) {
        mSpannable = (Spannable) mTextView.getText();
    }
    if (mSpannable == null || startOffset >= mTextView.getText().length()) {
        return;
    }
    selectText(startOffset, endOffset);
    showCursorHandle(mStartHandle);
    showCursorHandle(mEndHandle);
    mOperateWindow.show();
}
```

- 在 show 方法开始，因为之前可能已经显示了选择相关的 View ，比如先长按 TextView 的 A 点，然后弹出选择游标、操作框，此时再长按 B 点，此时再次弹出选择游标和操作框时，就需要先隐藏之前的相关 View 了，这里就这样简单粗暴地处理了下。
- `int startOffset = TextLayoutUtil.getPreciseOffset(mTextView, x, y);` 是一个很有意思的地方，这里参考了前面提到的开源项目里面的实现，这个方法通过传入 TextView 中一个点的坐标，就可以计算出来对应的最接近的那个文字的索引，简单说明如下：

 ![](http://ac-QYgvX1CC.clouddn.com/e9bc785c43e4f2ba.png)

 通过传入『种』那个字附近的某个点的坐标 (x,y)，就可以得出『种』在 TextView 的文本中的索引是 9 (从 0 开始计数)。

 `TextLayoutUtil.getPreciseOffset()` 方法如下：

```java
public static int getPreciseOffset(TextView textView, int x, int y) {
    Layout layout = textView.getLayout();
    if (layout != null) {
        int topVisibleLine = layout.getLineForVertical(y);
        int offset = layout.getOffsetForHorizontal(topVisibleLine, x);
        int offsetX = (int) layout.getPrimaryHorizontal(offset);
        if (offsetX > x) {
            return layout.getOffsetToLeftOf(offset);
        } else {
            return offset;
        }
    } else {
        return -1;
    }
}
```

 这里涉及到 TextView 的文本布局类 Layout ，虽然看过这块的部分源码，但是这里的处理还是有点懵，本文就不多深入了，有兴趣的话可以自行了解下这块的源码。

- 文本的选中显示是在 `selectText()` 方法中处理的，重点是设置 Span 和记录选中的文本信息：

  ```java
  private void selectText(int startPos, int endPos) {
      if (startPos != -1) {
          mSelectionInfo.mStart = startPos;
      }
      if (endPos != -1) {
          mSelectionInfo.mEnd = endPos;
      }
      if (mSelectionInfo.mStart > mSelectionInfo.mEnd) {
          int temp = mSelectionInfo.mStart;
          mSelectionInfo.mStart = mSelectionInfo.mEnd;
          mSelectionInfo.mEnd = temp;
      }
      if (mSpannable != null) {
          if (mSpan == null) {
              mSpan = new BackgroundColorSpan(mSelectedColor);
          }
          mSelectionInfo.mSelectionContent = mSpannable.subSequence(mSelectionInfo.mStart, mSelectionInfo.mEnd).toString();
          mSpannable.setSpan(mSpan, mSelectionInfo.mStart, mSelectionInfo.mEnd, Spanned.SPAN_INCLUSIVE_EXCLUSIVE);
          if (mSelectListener != null) {
              mSelectListener.onTextSelected(mSelectionInfo.mSelectionContent);
          }
      }
  }
  ```

  其中处理了下可能存在的 endPos 小于 startPos 的情况，进行了一次交换，后面就是设置 `BackgroundColorSpan` 已经记录下选中文本的信息，已经设置了选中监听时的回调。

  其中 mSelectionInfo 是 `SelectionInfo` 类的一个简单实例，该类就三个字段，选中文字的开始位置、结束位置和选中的文本：

  ```java
  public class SelectionInfo {
      public int mStart;
      public int mEnd;
      public String mSelectionContent;
  }
  ```

- `showCursorHandle()` 方法顾名思义就是显示选择游标，因为是 PopupWindow 实现的，重点就是显示位置的确定，这里再次涉及到 Layout 相关的 API :

  ```java
  private void showCursorHandle(CursorHandle cursorHandle) {
      Layout layout = mTextView.getLayout();
      int offset = cursorHandle.isLeft ? mSelectionInfo.mStart : mSelectionInfo.mEnd;
      cursorHandle.show((int) layout.getPrimaryHorizontal(offset), layout.getLineBottom(layout.getLineForOffset(offset)));
  }
  ```

  这里和之前的是反的，通过文本中的文字索引，来获取到对应的点的坐标。然后显示 PopupWindow 即可。

- 最后是显示操作框，同样是一个  PopupWindow ，这里的细节后面再展开。



#### 隐藏选择相关组件

这里没啥好说的，就是判空下左右选择游标和操作框，如果非空，则调用对应的 `dismiss()` 方法

```java
private void hideSelectView() {
    isHide = true;
    if (mStartHandle != null) {
        mStartHandle.dismiss();
    }
    if (mEndHandle != null) {
        mEndHandle.dismiss();
    }
    if (mOperateWindow != null) {
        mOperateWindow.dismiss();
    }
}
```

这里基本的流程和相关的实现细节已大概讲述了下，接下来就是就是选择游标和操作框的实现。

#### 选择游标

由于游标的移动涉及到文字的选中，以及操作框的显隐、定位，就直接实现为 `SelectableTextHelper` 的内部类。直接上代码：

```java
private class CursorHandle extends View {
    private PopupWindow mPopupWindow;
    private Paint mPaint;
  
    private int mCircleRadius = mCursorHandleSize / 2;
    private int mWidth = mCircleRadius * 2;
    private int mHeight = mCircleRadius * 2;
    private int mPadding = 25;
    private boolean isLeft;
    public CursorHandle(boolean isLeft) {
        super(mContext);
        this.isLeft = isLeft;
        mPaint = new Paint(Paint.ANTI_ALIAS_FLAG);
        mPaint.setColor(mCursorHandleColor);
        mPopupWindow = new PopupWindow(this);
        mPopupWindow.setClippingEnabled(false);
        mPopupWindow.setWidth(mWidth + mPadding * 2);
        mPopupWindow.setHeight(mHeight + mPadding / 2);
    }
    @Override
    protected void onDraw(Canvas canvas) {
        canvas.drawCircle(mCircleRadius + mPadding, mCircleRadius, mCircleRadius, mPaint);
        if (isLeft) {
            canvas.drawRect(mCircleRadius + mPadding, 0, mCircleRadius * 2 + mPadding, mCircleRadius, mPaint);
        } else {
            canvas.drawRect(mPadding, 0, mCircleRadius + mPadding, mCircleRadius, mPaint);
        }
    }
  
  ......
    
}
```

直接继承 PopupWindow 的话，没有 onDraw 方法 ，这里直接继承 View ，然后在 CursorHandle 的构造函数中初始化了一个 PopupWindow ，并将 CursorHandle 实例作为 contentView 传递进去，然后在 `onDraw()` 方法中绘制了自定义的选择游标，仿照 6.0 的选择游标效果。

![](http://ac-qygvx1cc.clouddn.com/ba2a7ee85d2d0915c2bb.svg)

这个也是绘制起来也是很简单的，一个正方形和一个圆组合下即可，处理下是左边还是右边就可以了，具体参照上面的代码。

接下来就是设置相关的触摸事件，响应拖动游标时更新选中的文本。

```java
private int mAdjustX;
private int mAdjustY;
private int mBeforeDragStart;
private int mBeforeDragEnd;
@Override
public boolean onTouchEvent(MotionEvent event) {
    switch (event.getAction()) {
        case MotionEvent.ACTION_DOWN:
            mBeforeDragStart = mSelectionInfo.mStart;
            mBeforeDragEnd = mSelectionInfo.mEnd;
            mAdjustX = (int) event.getX();
            mAdjustY = (int) event.getY();
            break;
        case MotionEvent.ACTION_UP:
        case MotionEvent.ACTION_CANCEL:
            mOperateWindow.show();
            break;
        case MotionEvent.ACTION_MOVE:
            mOperateWindow.dismiss();
            int rawX = (int) event.getRawX();
            int rawY = (int) event.getRawY();
            update(rawX + mAdjustX - mWidth, rawY + mAdjustY - mHeight);
            break;
    }
    return true;
}
```

- 在游标移动时，隐藏操作框，停止移动时，再显示操作框。

- 在触摸发生移动时，即 `MotionEvent.ACTION_MOVE` 时，更新游标位置和选中的文本，`update()` 方法如下：

  ```java
  private int[] mTempCoors = new int[2];
  public void update(int x, int y) {
      mTextView.getLocationInWindow(mTempCoors);
      int oldOffset;
      if (isLeft) {
          oldOffset = mSelectionInfo.mStart;
      } else {
          oldOffset = mSelectionInfo.mEnd;
      }
      y -= mTempCoors[1];
      int offset = TextLayoutUtil.getHysteresisOffset(mTextView, x, y, oldOffset);
      if (offset != oldOffset) {
          resetSelectionInfo();
          if (isLeft) {
              if (offset > mBeforeDragEnd) {
                  CursorHandle handle = getCursorHandle(false);
                  changeDirection();
                  handle.changeDirection();
                  mBeforeDragStart = mBeforeDragEnd;
                  selectText(mBeforeDragEnd, offset);
                  handle.updateCursorHandle();
              } else {
                  selectText(offset, -1);
              }
              updateCursorHandle();
          } else {
              if (offset < mBeforeDragStart) {
                  CursorHandle handle = getCursorHandle(true);
                  handle.changeDirection();
                  changeDirection();
                  mBeforeDragEnd = mBeforeDragStart;
                  selectText(offset, mBeforeDragStart);
                  handle.updateCursorHandle();
              } else {
                  selectText(mBeforeDragStart, offset);
              }
              updateCursorHandle();
          }
      }
  }
  ```

  在一开始的实现中，`update()` 方法没这么复杂，但是考虑到左边的游标在移动到右边游标的右边时，如下面的动图所示：

  ![](http://ww2.sinaimg.cn/large/91e23208jw1f9s285jsn8g20900g0gop.gif)

  此时就需要多一点处理，左边的右边变右边，右边的游标变左边，同时选中的文本也需要重新变换起点位置，原来是 end ，现在则变成了 start 。

  具体的逻辑实现就是根据之前选中的文本的前后位置信息，进行前后位置的交换。同时调整游标的方向，更新视图，这个逻辑在 `changeDirection()` 方法中：

  ```java
  private void changeDirection() {
      isLeft = !isLeft;
      invalidate();
  }
  ```

- 更新选择游标位置：由于游标的位置处理成只和选中的文本有关，因而处理起来较为简单，在上面的反转变化中，只要选中的文本正确变化了，那么这里的游标位置更新就是正确的。

  ```java
  private void updateCursorHandle() {
      mTextView.getLocationInWindow(mTempCoors);
      Layout layout = mTextView.getLayout();
      if (isLeft) {
          mPopupWindow.update((int) layout.getPrimaryHorizontal(mSelectionInfo.mStart) - mWidth + getExtraX(),
              layout.getLineBottom(layout.getLineForOffset(mSelectionInfo.mStart)) + getExtraY(), -1, -1);
      } else {
          mPopupWindow.update((int) layout.getPrimaryHorizontal(mSelectionInfo.mEnd) + getExtraX(),
              layout.getLineBottom(layout.getLineForOffset(mSelectionInfo.mEnd)) + getExtraY(), -1, -1);
      }
  }
  ```



#### 操作框

操作框的实现则简单的多，就是自定义布局的 PopupWindow ，然后处理下内部的 View 的点击事件即可，直接贴代码：

```java
private class OperateWindow {
    private PopupWindow mWindow;
    private int[] mTempCoors = new int[2];
    private int mWidth;
    private int mHeight;
    public OperateWindow(final Context context) {
        View contentView = LayoutInflater.from(context).inflate(R.layout.layout_operate_windows, null);
        contentView.measure(View.MeasureSpec.makeMeasureSpec(0, View.MeasureSpec.UNSPECIFIED),
            View.MeasureSpec.makeMeasureSpec(0, View.MeasureSpec.UNSPECIFIED));
        mWidth = contentView.getMeasuredWidth();
        mHeight = contentView.getMeasuredHeight();
        mWindow =
            new PopupWindow(contentView, ViewGroup.LayoutParams.WRAP_CONTENT, ViewGroup.LayoutParams.WRAP_CONTENT, false);
        mWindow.setClippingEnabled(false);
        contentView.findViewById(R.id.tv_copy).setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                ClipboardManager clip = (ClipboardManager) mContext.getSystemService(Context.CLIPBOARD_SERVICE);
                clip.setPrimaryClip(
                    ClipData.newPlainText(mSelectionInfo.mSelectionContent, mSelectionInfo.mSelectionContent));
                if (mSelectListener != null) {
                    mSelectListener.onTextSelected(mSelectionInfo.mSelectionContent);
                }
                SelectableTextHelper.this.resetSelectionInfo();
                SelectableTextHelper.this.hideSelectView();
            }
        });
        contentView.findViewById(R.id.tv_select_all).setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                hideSelectView();
                selectText(0, mTextView.getText().length());
                isHide = false;
                showCursorHandle(mStartHandle);
                showCursorHandle(mEndHandle);
                mOperateWindow.show();
            }
        });
    }
  
    public void show() {
        mTextView.getLocationInWindow(mTempCoors);
        Layout layout = mTextView.getLayout();
        int posX = (int) layout.getPrimaryHorizontal(mSelectionInfo.mStart) + mTempCoors[0];
        int posY = layout.getLineTop(layout.getLineForOffset(mSelectionInfo.mStart)) + mTempCoors[1] - mHeight - 16;
        if (posX <= 0) posX = 16;
        if (posY < 0) posY = 16;
        if (posX + mWidth > TextLayoutUtil.getScreenWidth(mContext)) {
            posX = TextLayoutUtil.getScreenWidth(mContext) - mWidth - 16;
        }
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.LOLLIPOP) {
            mWindow.setElevation(8f);
        }
        mWindow.showAtLocation(mTextView, Gravity.NO_GRAVITY, posX, posY);
    }
  
    public void dismiss() {
        mWindow.dismiss();
    }
}
```

在显示的之后，判断了下是否会显示到屏幕外面，如果会超出屏幕，则做一下微调即可。

### 一些细节的处理

#### 嵌套在滚动视图中的处理

在一开始的实现要点中就提到，需要注意一下嵌套在滚动视图中的处理，在尝试了一些方法之后，最终直接设置 `OnScrollChangedListener` 来解决，具体代码如下：

```java
mOnScrollChangedListener = new ViewTreeObserver.OnScrollChangedListener() {
    @Override
    public void onScrollChanged() {
        if (!isHideWhenScroll && !isHide) {
            isHideWhenScroll = true;
            if (mOperateWindow != null) {
                mOperateWindow.dismiss();
            }
            if (mStartHandle != null) {
                mStartHandle.dismiss();
            }
            if (mEndHandle != null) {
                mEndHandle.dismiss();
            }
        }
    }
};
mTextView.getViewTreeObserver().addOnScrollChangedListener(mOnScrollChangedListener);
```

这倒是解决了滑动时可以隐藏相关的选择控件的问题，但是停止滚动之后呢，如何重新显示选择控件呢？

在经过一些尝试之后，发现了 `OnPreDrawListener` 这个接口，在 TextView 发生滚动时期间一直在被调用，因此在这个接口里处理重新显示选择控件的逻辑是合适的：

```java
mOnPreDrawListener = new ViewTreeObserver.OnPreDrawListener() {
    @Override
    public boolean onPreDraw() {
        if (isHideWhenScroll) {
            isHideWhenScroll = false;
            showSelectView();
        }
        return true;
    }
};
mTextView.getViewTreeObserver().addOnPreDrawListener(mOnPreDrawListener);
```

在这样的设置之后，确实能保证停止滚动时重新显示选择相关的控件，但是整个滚动过程变得异常卡顿。

原因其实很简单，前面也提到了，`onPreDraw`  方法在 TextView 发生滚动时期间一直在被调用，然后这里一直处理显示选择控件的逻辑，能不卡顿么？

最后的解决方法是在源码中找到的，将 `showSelectView()` 方法替换成 `postShowSelectView()` 方法，

```java
private void postShowSelectView(int duration) {
    mTextView.removeCallbacks(mShowSelectViewRunnable);
    if (duration <= 0) {
        mShowSelectViewRunnable.run();
    } else {
        mTextView.postDelayed(mShowSelectViewRunnable, duration);
    }
}

private final Runnable mShowSelectViewRunnable = new Runnable() {
    @Override
    public void run() {
        if (isHide) return;
        if (mOperateWindow != null) {
            mOperateWindow.show();
        }
        if (mStartHandle != null) {
            showCursorHandle(mStartHandle);
        }
        if (mEndHandle != null) {
            showCursorHandle(mEndHandle);
        }
    }
};
```

很巧妙的方法，通过延迟调用具体的逻辑，避免了一直调用显示选择控件的逻辑，又学习到了。

#### TextView 移除出 Window 时一些处理

在一开始没处理这个的时候，一直报如下的错误：

![](http://ww4.sinaimg.cn/large/91e23208jw1f9s28uh9y1j21kw0hbjzv.jpg)

这么明显的错误可不能不管，处理起来也很简单：

```java
mTextView.addOnAttachStateChangeListener(new View.OnAttachStateChangeListener() {
    @Override
    public void onViewAttachedToWindow(View v) {
    }
    @Override
    public void onViewDetachedFromWindow(View v) {
        destroy();
    }
});

public void destroy() {
    mTextView.getViewTreeObserver().removeOnScrollChangedListener(mOnScrollChangedListener);
    mTextView.getViewTreeObserver().removeOnPreDrawListener(mOnPreDrawListener);
    resetSelectionInfo();
    hideSelectView();
    mStartHandle = null;
    mEndHandle = null;
    mOperateWindow = null;
}
```

将上面添加 Listener 也移除，同时隐藏响应的视图并置空。

### 写在最后

至此，自定义的选择复制功能完成，效果如下，GitHub 地址：[laobie/SelectableTextHelper](https://github.com/laobie/SelectableTextHelper)

![](http://ww2.sinaimg.cn/large/91e23208jw1f9s29s2jf7g20900g0npd.gif)

在开发之初，通过简单的查阅资料，梳理了个大概的实现思路，并考虑到实现中需要注意到的点，保证在开发中保持足够的警惕，不给自己挖坑。在整个开发过程中，通过阅读他人的源码，以及直接看官方的源码，一点点解决所遇到的问题，以及一点点地尝试，都是一次不错的开发经历，也算是弥补了当初没做出来这个任务的缺憾。
