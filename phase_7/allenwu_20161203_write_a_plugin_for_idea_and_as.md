尝试一下 IDEA 插件的开发。

之前具有试过开发一个翻译插件，没错，就是那个 AS 翻译插件很火的那段时间，但是开发过程中很多 Bug 解决不了，而且国内这方面资料相对较少，也很难查出个为什么。所以这个小项目就一直摆在那里。最近刚好都是零碎时间，又调试了几次，没想到竟然成功运行了。下面就从一个开发者的角度来讲讲如何开发一个自己的翻译插件。

### 流程步骤规划

网上也有几款类似的翻译插件，大部分的思路都是如下所示，我们就不另辟他径了，直接按照这种流程来吧：

* 熟悉基本的 IDEA 插件开发过程；
* 能够获取选中的单词；
* 调用某家 API，获得返回 JSON 数据并解析；
* 展示出解析的数据；
* 优化显示界面。

### 基本开发流程

下载 Intellij IDEA 编辑器，新建工程如下所示：

![](http://ww1.sinaimg.cn/large/b10d1ea5jw1fabhh2prwrj20mp0m5gnn.jpg)

在工程中的 src 处右键 New 选择 Action，如下图红色箭头所示：

![](http://ww2.sinaimg.cn/large/b10d1ea5jw1fabhhc9xxuj20wm0k1n5c.jpg)

紧接着自动弹出如下 New Action 配置框：

![](http://ww3.sinaimg.cn/large/b10d1ea5jw1fabhjuf615j20p80figpb.jpg)

填写好插件的基本属性之后，要选择启动方式，也就是上图蓝色选中处，我们选中的是 EditMenu 也就是 AS 或者 IDEA 中的 toolbar 中的 EDIT 选项。当你成功完成插件的编写和安装之后，选中单词然后点击 EDIT 选项，就能显示出对应的内容了。当然 JetBrains 作为如此优秀的 IDE 设计者，肯定是支持快捷方式的，即上图中的 `Keyboard Shortcuts` 我们可以输入启动该插件功能的快捷方式，如：Alt + A，当然如果的键盘快捷键设置的比较多，也可以设置第二快捷键，是很方便的：Alt + X.

完成上述 Action 的基本配置之后，在如下图所示的 plugin.xml 文件中就有对应的项添加进来了：

![](http://ww2.sinaimg.cn/large/b10d1ea5jw1fabhk1ew2ej20wm0k1dn1.jpg)

接下来看看 TestAction 类中代码吧：

``` java
public class TestAction extends AnAction {

    @Override
    public void actionPerformed(AnActionEvent e) {
        // TODO: insert action logic here
    }
}
```

基本的格式就如同上面的样子了，继承自 AnAction 类。英文注释说的很明显，在 actionPerFormed() 方法中进行你的逻辑代码编写，感觉有点像 Main() 方法的感觉啊。

### 获取选中数据

这一块大家所采用的方法比较一致，其实也就是 IDEA 开发过程中的 API 的调用而已，获取选中字符。代码如下所示：

``` java
// 获得 editor 对象
mEditor = e.getData(PlatformDataKeys.EDITOR);
   if (mEditor != null) {
      // 获得编辑模式
      SelectionModel model = mEditor.getSelectionModel();
      // 获取选中的文本
      String selectedText = model.getSelectedText();
      if (selectedText != null) {
      // 翻译并显示
      getTranslation(selectedText);
      }
}
```

### 接口和网络请求

在接口请求方面，我们选择扇贝的 API，除了简单清晰之外，其查询单词 API 直接返回单词的读音，这方便了后续的功能扩展。

其 API 为： https://api.shanbay.com/bdc/search/?word={word} ：查询 hello 返回的 API 接口数据如下：

``` json
{
    "status_code": 0,
    "msg": "SUCCESS",
    "data": { 
        "en_definitions": {
            n: [
                "an expression of greeting"
            ]
        },
        "cn_definition": {
            "pos": "",
            "defn": "int.（见面打招呼或打电话用语）喂,哈罗"
        },
        "id": 3130,
        "retention": 4,
        "definition": " int.（见面打招呼或打电话用语）喂,哈罗",
        "target_retention": 5,
        "en_definition": {
          "pos": "n",
          "defn": "an expression of greeting"
        },
        "learning_id": 2915819690,
        "content": "hello",
        "pronunciation": "hә'lәu",
        "audio": "http://media.shanbay.com/audio/us/hello.mp3",
        "uk_audio": "http://media.shanbay.com/audio/uk/hello.mp3",
        "us_audio": "http://media.shanbay.com/audio/us/hello.mp3"
    },
}
```

可见如上数据基本上能满足我们当前的需要和日后的扩展了。接下来我们选择网络请求框架，看了两款翻译插件的源码都是基本的 httpclient 来作为请求数据的框架的，比较接近最底层的 http 请求，这种其实挺好的，学习一下最基础的请求过程。但是底层的话随之带来的就是自己要去配置很多参数，配置请求头，同时也要去处理异步和回调之类的，就有点略显麻烦。作为一个追求效率的程序员，我们力求用最少的代码，完成同样的效果。所以我们想到了 `Retrofit`这个目前为止最火的网络请求框架：

![](http://ww2.sinaimg.cn/large/b10d1ea5jw1fabfjqjthej20l403vq37.jpg)

嘿嘿，其官网主页介绍 label，完美支持 Java 平台。所以我们就选择该网络请求框架啦。配合 Gson 解析 Json 数据。

为了确保 `跨平台`所带来的问题不会影响到插件的开发过程，先在 Android 平台写了一个 demo App 测试了一下，发现整个流程是没有问题的。接下来我们就在 IDEA 中编写核心代码啦。不过在此之前我们要导入两个 jar 格式的数据包，没错，你应该很熟悉，在 AS 中我们都是以 Gradle 方式导入进去的，这里好像就只能新建文件目录 libs，然后 Add 进去了：

![](http://ww3.sinaimg.cn/large/b10d1ea5jw1fabhnjk9ydj20qs0ik7c6.jpg)

开发 IDEA 插件的或者阅读本篇博客的应该都是 Android 司机啦，一定能够明白下面写法的用意了，没错，就跟 AS 中使用 Retrofit 作为网络请求框架的过程一样的，先定义一个 `ShanBeiApi`接口，里面写上相关代码，指定了实体类为 `Translation`,然后其他的看看 Retrofit 的使用文档就明白了:

``` java
public interface ShanBeiApi {
    // https://api.shanbay.com/bdc/search/?word={}
    @GET("bdc/search")
    Call<Translation> listInfos(@Query("word") String query);
}
```

如下就是本插件的核心之处了，我们将选中的单词传入该方法中:

``` java
private void getTranslation(String selectedText) {
        Retrofit retrofit = new Retrofit.Builder()
                .baseUrl(BASE_URL)
                .addConverterFactory(GsonConverterFactory.create())
                .build();
        ShanBeiApi shanbei= retrofit.create(ShanBeiApi.class);
        Call<Translation> call = shanbei.listInfos(selectedText);
  		// 事件回调
        call.enqueue(new Callback<Translation>() {
            @Override
            public void onResponse(Call<Translation> call, Response<Translation> response) {
       			// 获取在实体类中定义好的字符串
                String msg = response.body().toString();
              	// 显示返回的数据
                showPopupBalloon(msg);
            }

            @Override
            public void onFailure(Call<Translation> call, Throwable throwable) {
                //showPopupBalloon("error msg:"+ throwable.getMessage());
            }
        });
}
```

如上代码看起来应该没什么问题，因为在 App 中是能够行得通的，但是写好插件后调试偏偏又不显示窗口，更别说显示数据是否能正常返回了。解决这个有点小麻烦的 Bug，我的大概思路是这样的：先传入固定字符到调用的函数中，看看窗口是否能正常显示。测试一下之后发现窗口的显示是没有问题的。接着由于刚开始开发插件，还不太懂怎么去调式，把 Log 打出到控制台上。想着能不能在窗口显示的基础之上，把 Log 给显示出来，于是我就在 onFailure() 方法中打印了 error 的 msg，显示如下信息：

![](http://ww1.sinaimg.cn/large/b10d1ea5jw1fabhpmh6iyj20hu01l74f.jpg)

错误信息给出来了就好办了，赶紧复制黏贴一下 Google, 成功的找到了解决方案，导入一个 Gson.jar 即可。恩，虽然，成功解决了这个 Bug，但是具体原因我还真不知道为什么，我导入的相关的 gson 包中应该包含了所需的模块啊？

### 展示数据内容

获得了相关的数据之后，我们考虑的就是如何展示这些返回的数据了，是以 Dialog 的形式还是以 Popupwindow 的形式，默认的 Dialog 样式感觉有点难看，Popupwindow 有一种气泡模式，还挺好看的，也就是下面这段代码带来的效果：

``` java
// Refrence:https://github.com/Skykai521/ECTranslation/blob/master/src/RequestRunnable.java#L77 
private void showPopupBalloon(final String result) {
        ApplicationManager.getApplication().invokeLater(new Runnable() {
            public void run() {
                JBPopupFactory factory = JBPopupFactory.getInstance();
                factory.createHtmlTextBalloonBuilder(result, null, new JBColor(new Color(186, 238, 186), new Color(73, 117, 73)), null)
                  		// 显示时长
                        .setFadeoutTime(5000)
                        .createBalloon()
                        .show(factory.guessBestPopupLocation(mEditor), Balloon.Position.below);
            }
        });
 }
```

目前入门阶段就这么显示一下了，其实接口返回的数据还有音频的链接，下一步的如果要优化和丰富该翻译插件，就需要自定义显示的样式了，毕竟目前的显示样式还是一个气泡样式的，并且过了指定时间就会消失的，根本来不及响应事件的触发。具体显示的效果就是下面截图所示啦：

![](http://ww3.sinaimg.cn/large/b10d1ea5jw1facdyu5lw8j20s10hbte3.jpg)

如下所示即可生成 zip 或者 jar 格式的插件包，让别人的 IDEA 或者 Android Studio 导入了：

![](http://ww1.sinaimg.cn/large/b10d1ea5jw1facddgmsx4j20s10hbn4s.jpg)

如下图所示，在 File -> Setting 底下即可见到，我们选择红色箭头所示，从磁盘中导入，当然也可以从插件中心下载，不过本篇文章只做实例讲解，就不搞这些了，大家有兴趣的话，可以试试其他两款优秀的翻译插件：

![](http://ww2.sinaimg.cn/large/b10d1ea5jw1face04td6uj20s80ha41w.jpg)

还有一点就是如上截图你可以看到显示的音标还有点乱码，这个原因我看另一款翻译插件的 Issues 上也有提到，大概是系统字体的原因。但是这个理由在我的电脑上却得到并未解决，后续再看看吧。

### 扩展开发

* 通常为了防止错误输入我们可以，用一个正则来检测一个选取的单词，如果前后有空格就说明是一个完整的单词，如果选取的单词前面或者后面还有字符或者字母，就可以一定程度上断言这不是一个完整的单词了。


* 另外之前也提到过，扇贝 API 接口返回了单词读音数据，我们就要考虑如何能将读音播放出来，此时，只有气泡显示界面是不可以的了，我们得考虑能够触发事件的 Dialog 界面啦，其实也不是很麻烦的，如下所示截图，我们还是右键 New -> Dialog：

![](http://ww3.sinaimg.cn/mw690/b10d1ea5jw1facqcdc07vj20qh0hejwm.jpg)

​	    点击 TestDialog.form 就能见到如下所示界面了，这个其实跟 Java Swing 编程很类似，可以拖动具体的

​             控件到容器布局中，然后在 TestDialog 中编写相应代码，进行事件响应：

![](http://ww1.sinaimg.cn/mw690/b10d1ea5jw1facqbno3duj20s30hen3i.jpg)

* 恩，如果你用心阅读了本篇文章，会发现其实这个插件的开发一点挑战性都没有，建议的话，还是多看一些 star 数目比较高的开源项目，深入学习一下。

后续看看吧，有时间就继续优化一下，完善一下。不过也不一定，毕竟我这人喜欢跳票(逃...

最后本篇文章示例代码的 Demo 在[这里](https://github.com/wuchangfeng/idea-plugins/tree/master/TranslationPlugin),有需要的自取一下就好了。

文章中单词显示的悬浮窗效果代码参考[这里。](https://github.com/Skykai521/ECTranslation/blob/master/src/RequestRunnable.java#L77)如果读者有兴趣后续自己去优化的话，可以参考[这里](https://github.com/YiiGuxing/TranslationPlugin)，这个插件的样子大概做出来我想要的效果,Orz.

以上，谢谢阅读！！！

