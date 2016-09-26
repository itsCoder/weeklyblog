---
layout: post
title:  Android设计模式 Builder模式的分析与实践
author: shaDowZwy
---

之前实现了一个`Demo`底部弹出框，是用`DialogFragment`实现的一个`Dialog`,虽然实现了链式调用，但是没有使用`Builder`模式，所以想试试`Builder`模式，写博客记录一下。


>- 文章来源：itsCoder 的 [WeeklyBolg](https://github.com/itsCoder/weeklyblog) 项目
>- itsCoder主页：[http://itscoder.com/](http://itscoder.com/)
>- 作者：[shadow]( https://github.com/shaDowZwy )
>- 审阅者：

这篇博客谈论的`Android`中的`Builder`模式没有`Java`传统的使用那么复杂，只是一种编程思路，使需要复杂的参数构建的对象变得简单，而且内部封装构建过程可以方便变换和复用，更是降低了耦合性。


### **分析 AlertDialog**
`AlertDialog`是谷歌原生的对话框，在V7包中提供了`Material Design`设计风格的`Dialog`，使用非常广泛。但是，在这篇博客里，我们只讨论分析它的编程实现模式，它使用的就是`Builder`模式。

#### 1.先来看看 AlertDialog 源码

![](https://github.com/shaDowZwy/shaDowZwy.github.io/blob/master/images/builder_blog%E7%B4%A0%E6%9D%90/img1.png?raw=true)


从`AlertDialog`源码中可以发现成员变量中有一个`AlertController`，从它的**命名**
可以了解它大概是个组织类，即是所谓的”控制层“；再从`AlertDialog`的构造方法中发现，它没做什么太多的操作，主要就是初始化了`AlertController`，那我们可以进一步了解到
`AlertController`可能是担任组织数据逻辑的作用。

**那我们就进一步的看看**


![](https://github.com/shaDowZwy/shaDowZwy.github.io/blob/master/images/builder_blog%E7%B4%A0%E6%9D%90/img2.png?raw=true)

往下阅读它的源码，`AlertDialog`在`onCreate`中与`AlertController`建立了联系，从它的方法**命名**`installContent()`可以了解它是初始化`AlertDialog`内容实体的，那我们可以确定刚刚的推测，`AlertController`是负责`AlertDialog`组织内容逻辑的，而`AlertDialog`只是简单的”UI层“，


**再往下就是`AlertDialog`中静态内部类，也是我们要说的重点`Builder`**

![](https://github.com/shaDowZwy/shaDowZwy.github.io/blob/master/images/builder_blog%E7%B4%A0%E6%9D%90/img3.png?raw=true)

从`Builder`类中发现了一个眼熟的东西——`AlertController.AlertParams`,从它的**命名**（命名果然好重要）可以发现它可能是逻辑类，也就是”模型层“（是不是有点像MVC = =），负责处理数据逻辑的。

**再看一看源码**

![](https://github.com/shaDowZwy/shaDowZwy.github.io/blob/master/images/builder_blog%E7%B4%A0%E6%9D%90/img4.png?raw=true)

跟推测的一样，直接把数据赋予给了`AlertController.AlertParams`，而且都是return Builder 对象，保证了链式调用；源码里面大部分都是类似的代码，这里只贴出部分。

**在源码的最后，发现了重点**

![](https://github.com/shaDowZwy/shaDowZwy.github.io/blob/master/images/builder_blog%E7%B4%A0%E6%9D%90/img5.png?raw=true)

在`create()`方法中的`apply()`是将`AlertDialog`中的`AlertController`与`AlertController.AlertParams`建立联系，其实就是控制层与逻辑层相通，最后会由
`AlertController`控制要显示的视图内容。

`AlertDialog`的源码基本上我们过了一遍，了解它的模式思路，那我们再从`apply()`进去，看看`AlertController`与`AlertController.AlertParams`是怎么建立联系的

#### 2.AlertController.AlertParams源码

![](https://github.com/shaDowZwy/shaDowZwy.github.io/blob/master/images/builder_blog%E7%B4%A0%E6%9D%90/img10.png?raw=true)

从代码中可以看见，`AlertController`获得了`AlertController.AlertParams`中保存的数据，其他代码不在详述，无非就是转换数据，最后还是要赋予`AlertController`。

#### 3.AlertController 源码

![](https://github.com/shaDowZwy/shaDowZwy.github.io/blob/master/images/builder_blog%E7%B4%A0%E6%9D%90/img7.png?raw=true)

在构造方法中获得相对应的视图。

![](https://github.com/shaDowZwy/shaDowZwy.github.io/blob/master/images/builder_blog%E7%B4%A0%E6%9D%90/img8.png?raw=true)

这是之前介绍过的初始化实体内容的方法，显然是负责构建视图的

![](https://github.com/shaDowZwy/shaDowZwy.github.io/blob/master/images/builder_blog%E7%B4%A0%E6%9D%90/img11.png?raw=true)

构建视图的过程，通过获得的数据构建相应的视图内容。
所以说`AlertController`是整个模式中负责组织建造的，这也是`Builder`模式的核心。


通过分析`AlertDialog`的源码，我们了解谷歌原生组件的`Builder`的模式，通过分层将逻辑简化，代码简洁。


### **Builder模式 实践**

根据以上，我使用Builder模式重构了之前的小项目`BottomPopUpDialog`。

[BottomPopUpDialog](https://github.com/shaDowZwy/BottomPopUpDialog)

```java

public class BottomPopUpDialog extends DialogFragment {


    private TextView mCancel;

    private LinearLayout mContentLayout;

    private Builder mBuilder;

    private static BottomPopUpDialog getInstance(Builder builder) {
        BottomPopUpDialog dialog = new BottomPopUpDialog();
        dialog.mBuilder = builder;
        return dialog;

    }

    @Override
    public void onViewCreated(View view, @Nullable Bundle savedInstanceState) {
        super.onViewCreated(view, savedInstanceState);
        //该方法需要放在onViewCreated比较合适, 若在 onStart 在部分机型(如:小米3)会出现闪烁的情况
        getDialog().getWindow().setBackgroundDrawableResource(mBuilder.mBackgroundShadowColor);
    }

    @Override
    public void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setStyle(DialogFragment.STYLE_NORMAL, android.R.style.Theme_Holo_Light_NoActionBar);
    }

    @Nullable
    @Override
    public View onCreateView(LayoutInflater inflater, @Nullable ViewGroup container, @Nullable Bundle savedInstanceState) {
        View view = inflater.inflate(R.layout.bottom_pop_up_dialog, null);
        initView(view);
        registerListener(view);
        setCancelable(true);
        return view;
    }


    private void initView(View view) {
        mContentLayout = (LinearLayout) view.findViewById(R.id.pop_dialog_content_layout);
        mCancel = (TextView) view.findViewById(R.id.cancel);
        initItemView();
    }


    private void registerListener(View view) {

        view.setOnTouchListener(new View.OnTouchListener() {
            @Override
            public boolean onTouch(View v, MotionEvent event) {
                if (event.getAction() == MotionEvent.ACTION_DOWN) {
                    dismiss();
                }
                return false;
            }
        });

        mCancel.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                dismiss();
            }
        });
    }


    @Override
    public void show(FragmentManager manager, String tag) {
        try {
            super.show(manager, tag);
        } catch (Exception e) {
            e.printStackTrace();
        }
    }


    private void initItemView() {
        //循环添加item
        for (int i = 0; i < mBuilder.mDataArray.length; i++) {
            final PopupDialogItem dialogItem = new PopupDialogItem(getContext());
            dialogItem.refreshData(mBuilder.mDataArray[i]);

            //最后一项隐藏分割线
            if (i == mBuilder.mDataArray.length - 1) {
                dialogItem.hideLine();
            }

            //设置字体颜色
            if (mBuilder.mColorArray.size() != 0 && mBuilder.mColorArray.get(i) != 0) {
                dialogItem.setTextColor(mBuilder.mColorArray.get(i));
            }

            if (mBuilder.mLineColor != 0) {
                dialogItem.setLineColor(mBuilder.mLineColor);
            }

            mContentLayout.addView(dialogItem);

            dialogItem.setOnClickListener(new View.OnClickListener() {
                @Override
                public void onClick(View v) {
                    mBuilder.mListener.onDialogClick(dialogItem.getItemContent());
                    if (mBuilder.mIsCallBackDismiss) dismiss();
                }
            });
        }
    }



    public static class Builder {

        private String[] mDataArray;

        private SparseIntArray mColorArray = new SparseIntArray();

        private BottomPopDialogOnClickListener mListener;

        private int mLineColor;

        private boolean mIsCallBackDismiss = false;

        private int mBackgroundShadowColor = R.color.transparent_70;


        /**
         * 设置item数据
         */
        public Builder setDialogData(String[] dataArray) {
            mDataArray = dataArray;
            return this;
        }

        /**
         * 设置监听item监听器
         */
        public Builder setItemOnListener(BottomPopDialogOnClickListener listener) {
            mListener = listener;
            return this;
        }


        /**
         * 设置字体颜色
         *
         * @param index item的索引
         * @param color res color
         */
        public Builder setItemTextColor(int index, int color) {
            mColorArray.put(index, color);
            return this;
        }

        /**
         * 设置item分隔线颜色
         */
        public Builder setItemLineColor(int color) {
            mLineColor = color;
            return this;
        }

        /**
         * 设置是否点击回调取消dialog
         */
        public Builder setCallBackDismiss(boolean dismiss) {
            mIsCallBackDismiss = dismiss;
            return this;
        }


        /**
         * 设置dialog背景阴影颜色
         */
        public Builder setBackgroundShadowColor(int color) {
            mBackgroundShadowColor = color;
            return this;
        }


        public BottomPopUpDialog create() {
            return BottomPopUpDialog.getInstance(this);
        }


        public BottomPopUpDialog show(FragmentManager manager, String tag) {
            BottomPopUpDialog dialog = create();
            dialog.show(manager, tag);
            return dialog;
        }


    }


    public interface BottomPopDialogOnClickListener {
        /**
         * item点击事件回调
         *
         * @param tag item字符串 用于识别item
         */
        void onDialogClick(String tag);
    }

}



```
这是我结合前面的`Builder`模式写的`BottomPopUpDialog`。`Buildr`模式是将一个对象的构建与显示分离，将不同的参数一个一个添加进去，也是对于外部隐藏实现细节，更是降低了耦合度，方便以后的自由扩展。还有，我的实现省去了`Controller`层代码，我把控制层和UI层放在一起，这样实现是为了简单，我觉得`Controller`层是在可以复用的场景下，使用起来更有价值，而小组件可以更简单的使用`Builder`模式。**编程是简单实用**。


重构之后的调用

```java

new BottomPopUpDialog.Builder()
      .setDialogData(getResources().getStringArray(R.array.popup_array))
      .setItemTextColor(2, R.color.colorAccent)
      .setItemTextColor(4, R.color.colorAccent)
      .setCallBackDismiss(true)
      .setItemLineColor(R.color.line_color)
      .setItemOnListener(new BottomPopUpDialog.BottomPopDialogOnClickListener() {
                                @Override
                                public void onDialogClick(String tag) {
                                    Snackbar.make(view, tag, Snackbar.LENGTH_LONG)
                                            .setAction("Action", null).show();
                                }
                            })
      .show(getSupportFragmentManager(), "tag");

```








### 最后

这是一个简单的编程模式，记录一下思路。

