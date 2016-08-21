## BottomPopUpDialog底部弹出框的实现
这是一个之前实现的简单效果，类似于ios底部弹出框，所以想写一篇博客记录一下。


效果如图：![](https://github.com/shaDowZwy/shaDowZwy.github.io/blob/master/images/bottomdialog_1471771884.gif?raw=true)



### 使用 DialogFragment 
当我要实现这个效果的时候，首先想到的是 DialogFragment ，因为可以使用其生命周期来管理各种事件的处理，以及通过 onCreateView 或者  onCreateDIalog 来自定义视图，方便需求的变化和功能的扩展。


如果不了解 DialogFragment 的话，可以去看看相关资料，这里不做详述哈。

##### 我们来看看底部弹出框的布局
```java
<RelativeLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="wrap_content"
    android:background="@color/transparent"
    android:gravity="bottom">


    <com.shadow.bottompopupdialog.MaxHeightScrollView
        android:id="@+id/sl_root"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:layout_marginLeft="10dp"
        android:layout_marginRight="10dp"
        android:background="@drawable/round_rect_white"
        android:padding="2dp">

        <LinearLayout
            android:id="@+id/pop_dialog_content_layout"
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:background="@drawable/round_rect_white"
            android:orientation="vertical">
        </LinearLayout>
    </com.shadow.bottompopupdialog.MaxHeightScrollView>

    <TextView
        android:id="@+id/cancel"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:layout_below="@+id/sl_root"
        android:layout_marginBottom="5dp"
        android:layout_marginLeft="10dp"
        android:layout_marginRight="10dp"
        android:layout_marginTop="5dp"
        android:background="@drawable/round_rect_white"
        android:gravity="center"
        android:minHeight="55dp"
        android:padding="10dp"
        android:text="@string/cancel"
        android:textColor="@color/text_color"
        android:textSize="18sp" />


</RelativeLayout>

```
如代码所示，主要布局是 ScrollView 里面嵌套一个 LinearLayout 布局，用来在代码里添加item。

##### 别忘了这个自定义 MaxHeightScrollView

上面的布局中我使用的是 MaxHeightScrollView ，自定义 ScrollView 是想要显示最高的高度为屏幕的三分之二。

```java

public class MaxHeightScrollView extends ScrollView {


    public MaxHeightScrollView(Context context) {
        super(context);

    }

    public MaxHeightScrollView(Context context, AttributeSet attrs) {
        super(context, attrs);
    }

    @Override
    protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
        Context context = getContext();
        if (null != context) {
            int screenHeight = getScreenHeight(context);
            heightMeasureSpec = MeasureSpec.makeMeasureSpec(screenHeight * 2 / 3, MeasureSpec.AT_MOST);
        }
        super.onMeasure(widthMeasureSpec, heightMeasureSpec);
    }

    /**
     * 获取屏幕高度
     */
    private int getScreenHeight(Context context) {
        DisplayMetrics metrics = context.getResources().getDisplayMetrics();
        return metrics.heightPixels;
    }


}


```


### BottomPopUpDialog 详情

####先来看看一些view的细节

```java
    @Override
    public void onViewCreated(View view, @Nullable Bundle savedInstanceState) {
        super.onViewCreated(view, savedInstanceState);
        //该方法需要放在onViewCreated比较合适, 若在 onStart 在部分机型会出现闪烁的情况
        getDialog().getWindow().setBackgroundDrawableResource(mBackgroundShadowColor);
    }

    @Override
    public void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        //去掉title
        setStyle(DialogFragment.STYLE_NORMAL, android.R.style.Theme_Holo_Light_NoActionBar);
    }

```

如代码所示，在 onViewCreated 里设置弹窗的背景阴影颜色，默认是黑色透明的颜色。

 ```java
 mBackgroundShadowColor = R.color.transparent_70
 ```
 
 而且可以在代码中设置它的颜色。
 
 ```java
 
    /**
     * 设置dialog背景阴影颜色
     */
    public BottomPopUpDialog setBackgroundShadowColor(int color) {
        mBackgroundShadowColor = color;
        return this;
    }
 
 
 ```
 
 setStyle是为了去掉title，当然也可以使用
 
 ```java
getDialog().requestWindowFeature(Window.FEATURE_NO_TITLE)
 ```
 
 
####可调用的公共方法
```java

    /**
     * 设置item数据
     */
    public BottomPopUpDialog setDialogData(String[] dataArray) {
        mDataArray = dataArray;
        return this;
    }

    /**
     * 设置监听item监听器
     */
    public BottomPopUpDialog setItemOnListener(BottomPopDialogOnClickListener listener) {
        mListener = listener;
        return this;
    }


    /**
     * 设置字体颜色
     *
     * @param index item的索引
     * @param color res color
     */
    public BottomPopUpDialog setItemTextColor(int index, int color) {
        mColorArray.put(index, color);
        return this;
    }

    /**
     * 设置item分隔线颜色
     */
    public BottomPopUpDialog setItemLineColor(int color) {
        mLineColor = color;
        return this;
    }

    /**
     * 设置是否点击回调取消dialog
     */
    public BottomPopUpDialog setCallBackDismiss(boolean dismiss) {
        mIsCallBackDismiss = dismiss;
        return this;
    }


```

设置相应的数据后会在 onCreateView 里初始化数据，这些公共方法是必须要在dialogfragment.show() 之前调用，因为只有在show方法调用之后，dialogfragment 才会初始化，开始相应的生命周期。

####初始化数据

```java
 private void initItemView() {
        //循环添加item
        for (int i = 0; i < mDataArray.length; i++) {
            final PopupDialogItem dialogItem = new PopupDialogItem(getContext());
            dialogItem.refreshData(mDataArray[i]);

            //最后一项隐藏分割线
            if (i == mDataArray.length - 1) {
                dialogItem.hideLine();
            }

            //设置字体颜色
            if (mColorArray.size() != 0 && mColorArray.get(i) != 0) {
                dialogItem.setTextColor(mColorArray.get(i));
            }

            if (mLineColor != 0) {
                dialogItem.setLineColor(mLineColor);
            }

            mContentLayout.addView(dialogItem);

            dialogItem.setOnClickListener(new View.OnClickListener() {
                @Override
                public void onClick(View v) {
                    mListener.onDialogClick(dialogItem.getItemContent());
                    if (mIsCallBackDismiss) dismiss();
                }
            });
        }
    }

```
可以从代码中看到 LinearLayout 里循环添加的 item 是 PopupDialogItem ，它其实主体就是一个 TextView ，负责显示
String数组里的内容。

### BottomPopUpDialog 使用
```java
  BottomPopUpDialog dialog = new BottomPopUpDialog()
                            .setDialogData(getResources().getStringArray(R.array.popup_array))
                            .setItemTextColor(2, R.color.colorAccent)
                            .setItemTextColor(4, R.color.colorAccent)
                            .setCallBackDismiss(true)
                            .setItemOnListener(new BottomPopUpDialog.BottomPopDialogOnClickListener() {
                                @Override
                                public void onDialogClick(String tag) {
                                    Snackbar.make(view, tag, Snackbar.LENGTH_LONG)
                                            .setAction("Action", null).show();
                                }
                            });
                    dialog.show(getSupportFragmentManager(), "tag");

```

这些方法之前介绍过，setItemTextColor 可以重复设置颜色。

###最后
这是一个简单的小组件，记录一下编程的思路。

点击这里有源码 [GitHub地址](https://github.com/shaDowZwy/BottomPopUpDialog)

