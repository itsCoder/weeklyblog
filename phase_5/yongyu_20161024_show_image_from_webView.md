---

 title: Android  WebView 实现点击界面图片滑动浏览和保存图片功能
 date:  2016/10/24
 categories:  Android WebView
 tags:  Android WebView
---



# 一、概要

最近在公司的项目中遇到**需求如下**：

1. 点击 WebView 页面的图片实现开启查看图片模式，即可以显示点击的图片，然后滑动显示下一张图片。

     ![showImage](https://github.com/yongyu0102/WeeklyBlogImages/blob/master/phase5/showImage.gif)

2. 长按 WebView 页面图片弹出对话框可以选择保存长按的图片到本地相册。

    ![showpop](https://github.com/yongyu0102/WeeklyBlogImages/blob/master/phase5/showpop.gif)

拿到这个需求笔者第一反应是没做过 WebView 相关的交互，甚至分不清这个需求是否需要服务端配合完成 Java 与 JavaScripe 的互相调用，一脸茫然。遇到这种情况笔者的解决思路一般**分两个方向**：

1. 找一个比较出名的客户端有类似功能的，然后 google 搜索，仿 XXXX，先粗略看一下有没有现成的 Demo 可以参考，比如我这个需要，先去搜索一下 ”Android 仿微信朋友圈浏览图片效果“ （这个搜索关键字很关键啊），可是笔者没找到符合该需要的 Demo。

2. 在第一个方案不好使的情况下，我们没有了参考，那么咱们就得自己思考这个大概实现思路，然后把这个需求进行拆分，逐一击破。所以思考大概如下：

   （1）要想展示图片那么就得先拿到图片，要拿到图片只有两种可能，第一种可能是 WebView 本身缓存了图片，我们去缓存中读取图片进行显示，可是想一下，咱们浏览微博看图的时候如果没有网，这时候去点击图片那么图片是加载不出来的，所以这种可能否定了；所以只有第二种可能就是点击图片的时候拿到该图片对应的 URL 网址，然后咱们自己去网络加载图片进行显示，所以这个点我们 get 到了。

   （2）要滑动图片进行显示下一张，那么就需要我们能拿到所有要显示的图片的 URL ，然后放到一个数组里面，每次滑动就进行加载一张图片，那么也就是我们一次性拿到所有 WebView 包含图片的 URL，这个就不是在点击图片的时候去获取，而是在 WebView 加载完成后获取到，这怎么能拿到？再想一下，WebView  进行加载显示的时候其实是加载html（比如 assets 目录中的文件）文本的字符串，然后进行渲染处理显示出来，所以 html 文本文件里面包含了我们想要的图片网址，大家看一下下面这张截图就是一个带图片的 WebView 对应加载的 html 文本文件部分截图，

   ![webViewSrc](image\webViewSrc.png)

   其中标签 src 对象的内容就是我们想要的图片 URL，所以到这里我们就有了思路，我们先拿到 WebView 加载的 HTML 内容，然后在从 HTML 里面提取我要想要的 URL。

   （3）现在我们能拿到所有图片对应的 URL，那么滑动图片显示下一张就简单了，我们直接用一个 ViewPaper 来实现滑动加载图片即可。

   **总结要实现这个需要我们需要做的工作有：**

   1. 拿到 WebView 加载的 HTML 文本。
   2. 从 HTML 文本中提取所有图片对应的 URL。
   3. 处理 WebView 中图片的点击和长按响应事件。
   4. 用 ViewPaper 来实现滑动加载下一张图片。

   下面我们就按照以上几个步骤来实现我们想要的功能。

# 二、主要内容

## 2.1 获取 WebView 页面所有图片对应地址

### 2.1.1 解析 WebView 页面加载的 Html 文本文件

   **定义供 JavaScript 调用的交互接口**

```java
   private class InJavaScriptLocalObj {
       /**
        * 获取 WebView 加载对应的 Html 文本
        * @param html WebView 加载对应的 Html 文本
        */
       @android.webkit.JavascriptInterface
       public void showSource(String html) {
         //从 Html 文件中提取页面所有图片对应的地址对象
           getAllImageUrlFromHtml(html);
       }
   }
```

   **WebView 开启 JavaScript 脚本执行，调用 JavaScript 代码**

```java
     mWebView.getSettings().setJavaScriptEnabled(true);
     mWebView.addJavascriptInterface(new InJavaScriptLocalObj(), "local_obj");
     mWebView.setWebViewClient(new WebViewClient() {
               // 网页跳转
               @Override
               public boolean shouldOverrideUrlLoading(WebView view, String url) {
                   view.loadUrl(url);
                   return true;
               }
               // 网页加载结束
               @Override
               public void onPageFinished(WebView view, String url) {
                   super.onPageFinished(view, url);
                   //解析 Html
                   parseHTML(view);
               }
       /**
        *  java 调取 js 代码,
        * @param view WebView
        */
   private void parseHTML(WebView view) {
     //这段 js 代码是解析获取到了 Html 文本文件，然后调用本地定义的 Java 代码返回
     //解析出来的 Html 文本文件
       view.loadUrl("javascript:window.local_obj.showSource('<head>'+"
               + "document.getElementsByTagName('html')[0].innerHTML+'</head>');");
   }
```

### 2.1.2 从获取到的 Html 文本文件中提取页面所有图片对应的地址对象

```java
    // 获取 img 标签正则
       private static final String IMAGE_URL_TAG = "<img.*src=(.*?)[^>]*?>";
     // 获取src路径的正则
       private static final String IMAGE_URL_CONTENT = "http:\"?(.*?)(\"|>|\\s+)";

   /***
    * 获取页面所有图片对应的地址对象，
    * 例如 <img src="http://sc1.hao123img.com/data/f44d0aab7bc35b8767de3c48706d429e" />
    * @param html WebView 加载的 html 文本
    * @return
    */
   private List<String> getAllImageUrlFromHtml(String html) {
       Matcher matcher = Pattern.compile(IMAGE_URL_TAG).matcher(html);
       List<String> listImgUrl = new ArrayList<String>();
       while (matcher.find()) {
           listImgUrl.add(matcher.group());
       }
       //从图片对应的地址对象中解析出 src 标签对应的内容
       getAllImageUrlFormSrcObject(listImgUrl);
       return listImgUrl;
   }

      /***
        * 从图片对应的地址对象中解析出 src 标签对应的内容，即 url
        * 例如 "http://sc1.hao123img.com/data/f44d0aab7bc35b8767de3c48706d429e"
        * @param listImageUrl 图片地址对象例如 ：
        *<img src="http://sc1.hao123img.com/data/f44daab" />
        */
       private List<String> getAllImageUrlFormSrcObject(List<String> listImageUrl) {
           for (String image : listImageUrl) {
               Matcher matcher = Pattern.compile(IMAGE_URL_CONTENT).matcher(image);
               while (matcher.find()) {
                   listImgSrc.add(matcher.group().substring(0, matcher.group().length() - 1));
               }
           }
           return listImgSrc;
       }
```

到这里我们获取到了 WebView 页面中所有图片对象对应的 URL 地址，下面就还差一步，就是在点击 WebView 界面的图片时候去响应点击事件，然后把相应的 URL 地址传递给 ViewPaper 进行显示就齐活了。

## 2.2 响应 WebView 界面图片的点击事件

### 2.2.1定义供 JavaScript 调用的交互接口

```java
// js 通信接口，定义供 JavaScript 调用的交互接口
private class MyJavascriptInterface {
    private Context context;
    public MyJavascriptInterface(Context context) {
        this.context = context;
    }
    /**
     * 点击图片启动新的 ShowImageFromWebActivity，并传入点击图片对应的 url
     * 和页面所有图片对应的 url
     * @param url 点击图片对应的 url
     */
    @android.webkit.JavascriptInterface
    public void openImage(String url) {
        Intent intent = new Intent();
        intent.putExtra("image", url);
      //listImgSrc 该参数为页面所有图片对应的 url
        intent.putStringArrayListExtra(URL_ALL, (ArrayList<String>) listImgSrc);
        intent.setClass(context, ShowImageFromWebActivity.class);
        context.startActivity(intent);
    }
}
```

### 2.2.2 WebView 开启 JavaScript 脚本执行，调用 JavaScript 代码

```java
 mWebView.getSettings().setJavaScriptEnabled(true);
//载入 js
mWebView.addJavascriptInterface(new MyJavascriptInterface(this), "imageListener");
mWebView.setWebViewClient(new WebViewClient() {
    // 网页跳转
    @Override
    public boolean shouldOverrideUrlLoading(WebView view, String url) {
        view.loadUrl(url);
        return true;
    }

    // 网页加载结束
    @Override
    public void onPageFinished(WebView view, String url) {
        super.onPageFinished(view, url);
        // web 页面加载完成，添加监听图片的点击 js 函数
        addImageClickListener();
    }
  
  /**
     *  注入 js 函数监听，这段 js 函数的功能就是，遍历所有的图片，并添加 onclick 函数，
     *  实现点击事件，
     *  函数的功能是在图片点击的时候调用本地 java 接口并传递点击图片对应的 url 过去
     */
    private void addImageClickListener() {
        mWebView.loadUrl("javascript:(function(){" +
                "var objs = document.getElementsByTagName(\"img\"); " +
                "for(var i=0;i<objs.length;i++)  " +
                "{"
                + "    objs[i].onclick=function()  " +
                "    {  "
                + "        window.imageListener.openImage(this.src);  " +
                "    }  " +
                "}" +
                "})()");
    }
```
到这里我们完成了前两步，拿去到 WebView 界面图片对应的所有 URL 地址和响应 WebView 界面图片的点击事件，下面的事情就简单了，用 ViewPaper 滑动显示每一张图片，再我们进行最后一步之前，我们再来实现一个功能就是长按 WebView 界面图片，弹出对话框来，然后可以选择保存图片功能，代码如下：

 **WebView  中图片长按点击事件处理**

```java
//长按点击事件
mWebView.setOnLongClickListener(new View.OnLongClickListener() {
    @Override
    public boolean onLongClick(View v) {
      //响应长按事件
        responseWebLongClick(v);
        return false;
    }
});

  /**
     * 响应 WebView 长按图片的点击事件
     * @param v
     */
    private void responseWebLongClick(View v) {
        if (v instanceof WebView) {
            WebView.HitTestResult result = ((WebView) v).getHitTestResult();
            if (result != null) {
                int type = result.getType();
                //判断点击类型如果是图片
                if (type == WebView.HitTestResult.IMAGE_TYPE || type == WebView.HitTestResult.SRC_IMAGE_ANCHOR_TYPE) {
                    longClickUrl = result.getExtra();
                    //弹出对话框
                   showDialog(longClickUrl);
                }
            }
        }
    }

  /**
     * 长按 WebView 中图片弹出对话框，可以选择保存图片
     * @param url 点击图片对应的 url
     */
    private void showDialog(final String url) {
        new ActionSheetDialog(this)
                .builder()
                .setCancelable(true)
                .setCanceledOnTouchOutside(true)
                .addSheetItem(
                        "保存到相册",
                        ActionSheetDialog.SheetItemColor.Blue,
                        new ActionSheetDialog.OnSheetItemClickListener() {
                            @Override
                            public void onClick(int which) {
                                //下载图片
                                downloadImage(url);
                            }
                        }).show();
    }

```

## 2.3 ViewPaper 滑动显示每一张图片，PhotoView  实现自由缩放功能

由于这部分代码比较简单，这里就直接贴出部分代码，如文章中所用的 Demo 代码最终会上传到 github 上，有兴趣可以去瞧一瞧完整的代码，这里简单介绍几个类，ShowImageFromWebActivity.java 这个类内部就包含一个 ViewPaper 和两个按钮， ViewPaper 用来滑动显示每一张图片，按钮用来显示滑动的页数和实现点击保存图片功能，代码如下：

```java
public class ShowImageFromWebActivity extends Activity implements View.OnClickListener {
    private ViewPager vpImageBrowser;
    private TextView tvImageIndex;//显示滑动页数
    private Button btnSave;//保存图片按钮

    private ImageBrowserAdapter adapter;
    private ArrayList<String> imgUrls;//WebView 页面所有图片 URL
    private String url;//WebView 页面所有图片中被点击图片对应 URL
    private int currentIndex;//标记被滑动图片在所有图片中的位置
    private Handler mHandler;//异步发送消息
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_show_image_from_web);
        initView();
        initListener();
        initData();
    }

    private void initView(){
        vpImageBrowser = (ViewPager) findViewById(R.id.vp_image_browser);
        tvImageIndex = (TextView) findViewById(R.id.tv_image_index);
        btnSave = (Button) findViewById(R.id.btn_save);
    }


    private void initData(){
        mHandler = new Handler();
        imgUrls=getIntent().getStringArrayListExtra(MainActivity.URL_ALL);
        url=getIntent().getStringExtra("image");
      //获取被点击图片在所有图片中的位置
        int position=imgUrls.indexOf(url);
        adapter=new ImageBrowserAdapter(this,imgUrls);
        vpImageBrowser.setAdapter(adapter);
        final int size=imgUrls.size();

        if(size > 1) {
            tvImageIndex.setVisibility(View.VISIBLE);
            tvImageIndex.setText((position+1) + "/" + size);
        } else {
            tvImageIndex.setVisibility(View.GONE);
        }
        vpImageBrowser.setOnPageChangeListener(new ViewPager.OnPageChangeListener() {
            @Override
            public void onPageSelected(int arg0) {
                currentIndex=arg0;
                int index = arg0 % size;
                tvImageIndex.setText((index+1) + "/" + size);
            }
            @Override
            public void onPageScrolled(int arg0, float arg1, int arg2) {
                // TODO Auto-generated method stub
            }
            @Override
            public void onPageScrollStateChanged(int arg0) {
                // TODO Auto-generated method stub
            }
        });
        vpImageBrowser.setCurrentItem(position);
    }

    private void initListener(){
        btnSave.setOnClickListener(this);
    }

    @Override
    public void onClick(View v) {
        switch (v.getId()){
            case R.id.btn_save :
                Toast.makeText(getApplicationContext(), "开始下载图片", Toast.LENGTH_SHORT).show();
                downloadImage();
                break;
        }

    /**
     * 开始下载图片
     */
    private void downloadImage() {
        downloadAsync(imgUrls.get(currentIndex), Environment.getExternalStorageDirectory().getAbsolutePath() + "/ImagesFromWebView");
    }
      
   }
```

```java
//ShowImageFromWebActivity.java 对应 xml 文件
<?xml version="1.0" encoding="utf-8"?>
<RelativeLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:background="@android:color/black"
    tools:context="activity.ShowImageFromWebActivity">
    <view.PhotoViewViewPager
        android:id="@+id/vp_image_browser"
        android:layout_width="match_parent"
        android:layout_height="match_parent" >
    </view.PhotoViewViewPager>

    <LinearLayout
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:layout_alignParentBottom="true"
        android:orientation="horizontal"
        android:padding="16dp" >
        <TextView
            android:textSize="18sp"
            android:id="@+id/tv_image_index"
            android:layout_width="56dp"
            android:layout_height="36dp"
            android:background="@drawable/shape_corner_rect_gray"
            android:gravity="center"
            android:paddingTop="3dp"
            android:paddingBottom="3dp"
            android:paddingRight="10dp"
            android:paddingLeft="10dp"
            android:text="1/9"
            android:textColor="@android:color/white" />
        <View
            android:layout_width="0dp"
            android:layout_height="1dp"
            android:layout_weight="1" />
        <Button
            android:id="@+id/btn_save"
            android:textSize="@dimen/_16sp"
            android:layout_width="56dp"
            android:layout_height="36dp"
            android:background="@drawable/shape_corner_rect_gray"
            android:gravity="center"
            android:padding="4dp"
            android:text="保存"
            android:textColor="@color/white" />
    </LinearLayout>
</RelativeLayout>

```

ImageBrowserAdapter.java 类：

```java
public class ImageBrowserAdapter extends PagerAdapter {
   private Activity context;
   private List<String> picUrls;

   public ImageBrowserAdapter(Activity context, ArrayList<String> picUrls) {
      this.context = context;
      this.picUrls = picUrls;
   }

   @Override
   public int getCount() {

      return picUrls.size();
   }
   
   @Override
   public boolean isViewFromObject(View view, Object object) {
      return view == object;
   }

   @Override
   public View instantiateItem(ViewGroup container, int position) {
      View view = View.inflate(context, R.layout.item_image_browser, null);
      ImageView iv_image_browser = (ImageView) view.findViewById(R.id.show_webimage_imageview);
      String picUrl = picUrls.get(position);
      final  PhotoViewAttacher photoViewAttacher=new PhotoViewAttacher(iv_image_browser);
      photoViewAttacher.setScaleType(ImageView.ScaleType.FIT_CENTER);
     //禁止缩放
      photoViewAttacher.setZoomable(false);
     //显示图片
      Glide.with(context).
            load(picUrl)
            .crossFade()
            .placeholder(R.drawable.avatar_default)
            .error(R.drawable.image_default_rect)
            .into(new GlideDrawableImageViewTarget(iv_image_browser){
               @Override
               public void onResourceReady(GlideDrawable resource, GlideAnimation<? super GlideDrawable> animation) {
                  super.onResourceReady(resource, animation);
                     photoViewAttacher.update();
                 //开启缩放
                     photoViewAttacher.setZoomable(true);
               }
            });

      container.addView(view);
      return view;
   }

   @Override
   public void destroyItem(ViewGroup container, int position, Object object) {
      container.removeView((View) object);
   }
```

上面代码也很简单，就是根据 URL 来加载显示图片，但是笔者在使用 PhtoView 时候遇到一个奇怪问题未能搞懂，这里说一下，希望大家看到能帮忙分析下，上面代码大家可能会发现，我在 instantiateItem 方法中开始用 Glide 加载显示图片之前调用了一遍  photoViewAttacher.setZoomable(false) 方法禁止了图片缩放，然后在用 Glide 加载完成图片之后又在 onResourceReady 内部又调用了一遍   photoViewAttacher.setZoomable(true) 方法开启图片缩放功能，之所以这里这么做是因为上面代码在注释掉这两行代码之后，在加载图片的时候，显示图片时候图片会被莫名的放大，显示不正常，笔者没找到原因，所以这里就采用这种笨方法手动控制下解决了这个问题，希望有人帮忙分析下，谢谢了。

```java
//ImageBrowserAdapter Item 布局文件
<?xml version="1.0" encoding="utf-8"?>
<RelativeLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:gravity="center">

        <RelativeLayout
            android:orientation="horizontal"
            android:layout_width="match_parent"
            android:layout_height="match_parent"
            android:gravity="center_vertical">
            <uk.co.senab.photoview.PhotoView
                android:layout_centerVertical="true"
                android:id="@+id/pv_show_image"
                android:layout_width="match_parent"
                android:layout_height="match_parent"
                android:layout_centerInParent="true"
                android:layout_gravity="center"
                android:adjustViewBounds="true"
                android:background="@android:color/white"
                android:scaleType="fitXY"
                android:src="@drawable/image_default_rect" />
        </RelativeLayout>
</RelativeLayout>

```

以上为本次学习内容，如有错误还望指正，谢谢！

文章中 Demo 已经上传在 github 上，地址为[**ShowImageFromWebView**](https://github.com/yongyu0102/ShowImageFromWebView)

参考文章：

 [JAVA抓取网页的图片,JAVA利用正则表达式抓取网站图片](http://blog.csdn.net/swingpyzf/article/details/16338903)

[android webview js交互， 响应webview中的图片点击事件](http://blog.csdn.net/wangtingshuai/article/details/8635787)

[从WebView中点击一张图片，转换成一个可缩放，可旋转的图片](http://www.jianshu.com/p/e24ee6d67f01)

