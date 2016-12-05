---
title: Material Design 控件之 Toolbar
---

## 什么是 Toolbar？

**Toolbar 继承自 ViewGroup**，也就是说，Toolbar 也是一个官方定制的操作栏，这样可以做到向下兼容，在应用中使用 Toolbar，只需要引入 V7 appcompat 支持库。在 Material Design 中已统一被称为应用栏（App Bar）也称操作栏（action bar），而使用 Toolbar 作为操作栏可以兼容最广泛的设备。

对于 Toolbar，[官方的解释](https://developer.android.com/reference/android/support/v7/widget/Toolbar.html) 是（以下为笔者翻译自官方文档）：

> Toolbar 是一个用于应用程序内容中的标准工具栏。
>
> Toolbar 是用于应用程序布局中 ActionBar 的一个推广。虽然操作栏习惯性地作为一个由 framework 所控制的 Activity 不透明窗口装饰的一部分，但是 Toolbar 可以放置于视图层级中任意级别的嵌套之中。通过使用 setSupportActionBar() 方法，应用程序可以选择指定 Toolbar 作为 Activity 的操作栏。
>
> Toolbar 比 ActionBar 支持更多集中的特性。从头到尾，一个 Toolbar 可以包含以下可选元素的组合：
>
> - _导航按钮_。这可以是一个上级箭头，导航菜单切换，关闭，收起，完成或者是另一个应用程序的选择图像字符。这个按钮应始终被用于访问 Toolbar 中其它导航目的地及其所指的内容，否则离开当前被 Toolbar 所指定的上下文。导航按钮默认垂直排列对齐在 Toolbar 的最小高度。
>
>
> - _应用 Logo_。这个 Logo 图像高度可以延伸到 Toolbar 的高度，宽度任意。
>
>
> - _标题和副标题_。标题应起一个可标识 Toolbar 在当前导航层级中位置及此时所包含内容的作用。如果需要指示关于当前内容的任何扩展信息就可以设置显示副标题。而如果应用程序使用一个 logo 图像，你应该慎重考虑省略标题和副标题。
>
>
> - _一或多个自定义视图_。应用程序可以添加任意的子视图到 Toolbar 中。子视图出现于布局中的位置。如果子视图的 Toolbar.LayoutParams 指定其 Gravity 值为 CENTER_HORIZONTAL，则其将在 Toolbar 中除去其它已被测量的元素所占据的空间后所剩下的空间中居中。
>
>
> - _[活动菜单](https://developer.android.com/reference/android/support/v7/widget/ActionMenuView.html)_。被定于 Toolbar 尾部的活动菜单提供一些频繁，重要或者典型的行为连同一个可选的溢出菜单提供一些附加行为。这些动作按钮默认垂直排列在 Toolbar 的最小高度中。
>
> 现代 Android UI 开发者比起他们的应用图标应该更倾向于 Toolbar 在视觉上的直观配色方案。应用图标外加标题的标准布局在 API 21 及更新的设备上已不推荐使用。

总的来说就是：不要再用 ActionBar 啦！现在有更为好用而又标准的 Toolbar 啦！

## Toolbar 使用

1.**向项目中添加 v7 appcompat 支持库** 。

你可以在 build.gradle(Module:app) 的 dependencies 中直接添加,然后 Sync Now 就可以了：

``` gradle
compile 'com.android.support:appcompat-v7:25.0.1'
```

或者你可以更方便地使用 Ctrl + Shift + Alt + s 快捷键，调出 **Project Structure** 界面，选择 Modules 下 app 的 Dependencies 选项，点击 + 号，选择添加一个 Library dependency，搜索添加即可，如下所示：

![Add dependencies](http://7xtt0k.com1.z0.glb.clouddn.com/weeklyBlog/NO.7/add-dependencies.png)

2.确保每个使用 Toolbar 作为应用栏的 Activity 都可以扩展 AppCompatActivity：

``` java
public class MyActivity extends AppCompatActivity {
  // ...
}
```

3.在新建一个应用的时候，默认布局是有 ActionBar 的，所以我们现在 style 文件中将 ActionBar 去除。让 AppTheme 继承自一个没有 ActionBar 的主题：

![NoActionBar theme](http://7xtt0k.com1.z0.glb.clouddn.com/weeklyBlog/NO.7/no_actionbar.png)

将根布局改为 LinearLayout 并设置 orientation 为 vertical 垂直排列。并且 **将根布局的 padding 去除**：

``` xml
<RelativeLayout
    android:paddingBottom="@dimen/activity_vertical_margin"
    android:paddingLeft="@dimen/activity_horizontal_margin"
    android:paddingRight="@dimen/activity_horizontal_margin"
    android:paddingTop="@dimen/activity_vertical_margin">

</RelativeLayout>
```

现在的 activity_main 布局就是这个样子了：

![No ActionBar layout](http://7xtt0k.com1.z0.glb.clouddn.com/weeklyBlog/NO.7/NoActionBarLayout.png)

4.接下来就是在布局中引入 Toolbar，首先在根布局中 **添加 Toolbar 的自定义属性命名空间** `xmlns:toolbar="http://schemas.android.com/apk/res-auto"` 。添加自定义属性命名空间是为了能够引用 Toolbar 为我们提供的那些自定义属性。关于这一点可以参照 “[医生](http://blog.csdn.net/eclipsexys)” 《Android 群英传》第三章 “Android 控件架构与自定义控件详解”：

> **在需要使用的地方引用 UI 模板，在引用前，需要指定引用第三方控件的命名空间。** 在布局文件中，可以看到如下一行代码。

> `xmlns:android="http://schemas.android.com/apk/res/android"`

> 这行代码就是在指定引用的名字空间 `xmlns`，即 xml namespace。这里指定了名字空间为 "android"，因此在接下来使用系统属性的时候才可以使用 "android:" 来引用 Android 系统属性。同样地，如果要使用自定义的属性，那么就需要创建自己的名字空间，在 Android Studio 中，第三方的控件都使用如下代码来引入名字空间。

> `xmlns:custom="http://schemas.android.com/apk/res-auto"`

> 这里将引入第三方控件的名字空间取名为 custom，之后在 XML 文件中使用自定义属性时，就可以通过这个名字空间来引用。

添加 v7 widget 下的 Toolbar 控件，直接输入 `<Toolbar` 在提示中选择 `android.support.v7.widget.Toolbar` 即可，且根据 Material Design 规范设置其外观。

现 activity_main.xml 代码如下：

``` xml
<?xml version="1.0" encoding="utf-8"?>
<RelativeLayout
    xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools"
    xmlns:toolbar="http://schemas.android.com/apk/res-auto"
    android:id="@+id/activity_main"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    tools:context="cn.showzeng.toolbartest.MainActivity">

    <android.support.v7.widget.Toolbar
        android:id="@+id/my_toolbar"
        android:layout_width="match_parent"
        android:layout_height="?attr/actionBarSize"
        android:background="@color/colorPrimary"
        android:elevation="4dp"
        android:theme="@style/ThemeOverlay.AppCompat.ActionBar"
        toolbar:popupTheme="@style/ThemeOverlay.AppCompat.Light">

    </android.support.v7.widget.Toolbar>

    <TextView
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="Hello World!"/>
</RelativeLayout>
```

同时，在 Activity 的 onCreate() 方法中，调用 Activity 的 setSupportActionBar() 方法，然后传递 Activity 的工具栏。该方法会将工具栏设置为 Activity 的应用栏。

``` java
package cn.showzeng.toolbartest;

import android.support.v7.app.AppCompatActivity;
import android.os.Bundle;
import android.support.v7.widget.Toolbar;

public class MainActivity extends AppCompatActivity {

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        Toolbar toolbar = (Toolbar) findViewById(R.id.toolbar);
        setSupportActionBar(toolbar);
    }
}
```

预览效果：

![Add Toolbar widget](http://7xtt0k.com1.z0.glb.clouddn.com/weeklyBlog/NO.7/ToolbarLayout.png)

关于这里的高度值 `？attr/actionBarSize` ，按住 Ctrl 键跳进去看就可以知道是一个 values.xml 资源文件中指定的一个值，这里就是原本 ActionBar 控件的高度。对于 Toolbar 常用的属性，在文章开头说到有：导航按钮、Logo、标题和副标题、一个或多个自定义视图、活动菜单。下面我们来将其一一实现。

- 导航图标：在 xml 中为 `toolbar:navigationIcon=""` ，对应 Activity 中的 setNavigationIcon() 方法。对应的还有 setNavigationOnClickListener() 设定监听事件方法

- Logo：在 xml 中为 `toolbar:logo=""` ，对应 Activity 中的 setLogo() 方法。

- 标题和副标题：在 xml 中为 `toolbar:title=""` 和 `toolbar:subtitle=""`，对应 Activity 中的 setTitle() 和 setSubtitle() 方法。字体颜色和样式可以使用 `toolbar:titleMargin=""`、`toolbar:titleTextColor=""`、`toolbar:titleTextAppearance=""`、`toolbar:subtitleTextColor=""`、`toolbar:subtitleTextAppearance=""` 等方法，对应 Activity 中各 set 方法如 setTitleTextColor(int color) 。

- 一或多个自定义视图：直接在 Toolbar 控件中插入想要的子控件即可。

![Toolbar](http://7xtt0k.com1.z0.glb.clouddn.com/weeklyBlog/NO.7/toolbar.png)

从这里可以看出 NavigationIcon 和 Logo 的间距是不一样的，如果单独设置二者，可以对比出 NavigationIcon 离 title 和 subtitle 的间隔更大，比 Logo 显得优雅一些，当然你可以通过设置 title 的 margin 值来实现同样的效果。此状态下对应的 activity_main.xml 如下：

``` xml
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout
    xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools"
    xmlns:toolbar="http://schemas.android.com/apk/res-auto"
    android:id="@+id/activity_main"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:orientation="vertical"
    tools:context="cn.showzeng.toolbartest.MainActivity">

    <android.support.v7.widget.Toolbar
        android:id="@+id/toolbar"
        android:layout_width="match_parent"
        android:layout_height="?attr/actionBarSize"
        android:background="@color/colorPrimary"
        android:elevation="4dp"
        android:theme="@style/ThemeOverlay.AppCompat.ActionBar"
        toolbar:popupTheme="@style/ThemeOverlay.AppCompat.Light"
        toolbar:navigationIcon="@drawable/navigation"
        toolbar:logo="@drawable/logo"
        toolbar:title="@string/app_name"
        toolbar:titleTextColor="@color/white"
        toolbar:subtitle="@string/subtitle"
        toolbar:subtitleTextColor="@color/white">

        <TextView
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:text="@string/childrenview"
            android:textColor="@color/white"/>
    </android.support.v7.widget.Toolbar>
</LinearLayout>
```

- 活动菜单：在一些应用的应用栏右边经常可以看到一些按钮，例如常见的分享、编辑、扫码等，而这些正是 Toolbar 中的活动菜单。有趣的是，在此之前我一直以为最右边的展开菜单按钮，也是和旁边的其它按钮一样是自己设定的，让后设置点击事件弹出一个 PopupWindow。其实不然，由于 Toolbar 供给活动菜单使用空间限制的，当活动菜单里的活动按钮过多，这时就会将溢出的活动转移到溢出菜单中，而至于是谁将移入到溢出菜单中，则是看各活动显示优先级和其他一些属性。

而包括溢出菜单中所有的活动按钮都是在 menu 资源的 xml 文件中定义的，所以首先我们先新建 res/menu/toolbar_menu.xml:

``` xml
<?xml version="1.0" encoding="utf-8"?>
<menu xmlns:android="http://schemas.android.com/apk/res/android"
      xmlns:app="http://schemas.android.com/apk/res-auto"
      xmlns:tools="http://schemas.android.com/tools"
      tools:context=".MainActivity">
    <item
        android:id="@+id/menu_github"
        android:icon="@drawable/github"
        android:orderInCategory="1"
        android:title="@string/github"
        app:showAsAction="ifRoom" />
    <item
        android:id="@+id/menu_printer"
        android:icon="@drawable/printer"
        android:orderInCategory="2"
        android:title="@string/printer"
        app:showAsAction="ifRoom" />
    <item
        android:id="@+id/menu_setting"
        android:orderInCategory="3"
        android:title="@string/setting"
        app:showAsAction="never" />
    <item
        android:id="@+id/menu_about"
        android:orderInCategory="3"
        android:title="@string/about"
        app:showAsAction="never" />
</menu>
```

这里的 orderInCategory 类似布局中的权重，只是这里这个值越小，显示的优先级越高。而 showAsAction 属性有常用的三个值 ifRoom、never、always，其中 ifRoom 表示如果有足够容纳活动按钮的空间时就显示，否则移入溢出菜单，never 表示只显示在溢出菜单，对应的，always 表示显示在操作栏中。但是由于不同设备屏幕的差异，**一般建议将想要显示的活动按钮设为 ifRoom，可以为其设定优先级，不想显示的设为 never**。在 Android Studio 中你也会发现，如果你设了多个活动按钮为 always 的时候，就会发出警告。关于这些属性的使用，可以自行设置体会一下。想要了解更多关于菜单属性的详细介绍可以参照官方文档：[Menu Resource](https://developer.android.com/guide/topics/resources/menu-resource.html) 。

接着我们 **将菜单填充到我们的 Activity 中**，通常有以下两种方法。一是在 Activity 的 onCreate() 方法中使用 Toolbar 的 inflateMenu() 方法将菜单填充进去。

``` java
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        Toolbar toolbar = (Toolbar) findViewById(R.id.toolbar);
        toolbar.inflateMenu(R.menu.toolbar_menu);
        setSupportActionBar(toolbar);
    }
```

或者是重写 onCreateOptionsMenu() 方法将布局填充。

``` java
    @Override
    public boolean onCreateOptionsMenu(Menu menu) {
        getMenuInflater().inflate(R.menu.toolbar_menu, menu);
        return true;
    }
```

**为菜单活动设定响应事件**,当我们选中了这些活动按钮时，系统会调用 Activity 的 onOptionsItemSelected() 回调方法，通过传入一个 MenuItem 对象来指定被点击的选项。我们就可以通过重写 onOptionsItemSelected() 方法，调用 MenuItem.getItemId() 方法指定被点击的选项，以此来实现活动按钮的响应事件。

``` java
@Override
    public boolean onOptionsItemSelected(MenuItem item) {

        switch (item.getItemId()) {
            case R.id.menu_cardboard:
                //Action you want
                return true;
            case R.id.menu_cloud:
                //Action you want
                return true;
            case R.id.menu_setting:
                //Action you want
                return true;
            case R.id.menu_about:
                //Action you want
                return true;
            default:
                return super.onOptionsItemSelected(item);
        }
    }
```

当然，也可以使用 Toolbar 的 setOnMenuItemClickListener() 方法来设定响应事件，但与上面的方法相比，还是重写方法更优雅一点。

``` java
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        Toolbar toolbar = (Toolbar) findViewById(R.id.toolbar);
        toolbar.setOnMenuItemClickListener(new Toolbar.OnMenuItemClickListener() {
            @Override
            public boolean onMenuItemClick(MenuItem item) {
                switch (item.getItemId()) {
                    case R.id.menu_github:
                        //Action you want
                        return true;
                    case R.id.menu_printer:
                        //Action you want
                        return true;
                    case R.id.menu_setting:
                        //Action you want
                        return true;
                    case R.id.menu_about:
                        //Action you want
                        return true;
                }
                return true;
            }
        });

        setSupportActionBar(toolbar);
    }
```

此时的操作栏如下图所示：

![Full Toolbar](http://7xtt0k.com1.z0.glb.clouddn.com/weeklyBlog/NO.7/full_toolbar.jpg)

![Overflow Menu](http://7xtt0k.com1.z0.glb.clouddn.com/weeklyBlog/NO.7/overflow_menu.jpg)

我们会注意到溢出菜单默认显示黑色，明显不搭。关于这一点，可参考 [Android：改变 Toolbar 的文字和溢出图标颜色](http://blog.csdn.net/zhyh1986/article/details/51790570) 这篇文章。因为在 style 文件里给应用使用的是浅色主题，因此默认的标题和溢出菜单图标颜色都是黑色，Material Design 中也推荐这样的搭配使用，如果强行要改的话，就看前面给出的链接，这里就不费时间了。

## 参考文档

**[Adding the App Bar](https://developer.android.com/training/appbar/index.html)**

**[Toolbar](https://developer.android.com/reference/android/support/v7/widget/Toolbar.html#attr_android.support.v7.appcompat:collapseIcon)**

**[Android：Toolbar 详解](http://www.jcodecraeer.com/a/anzhuokaifa/androidkaifa/2014/1118/2006.html)**

**[Material Design 样式及 Toolbar 使用初探](https://segmentfault.com/a/1190000003695173)**
