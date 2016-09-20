---
title: Gson 解析那些事
thumbnailImagePosition: left
metaAlignment: left
coverMeta: in
coverSize: full
date: 2016-09-11 13:34:39
tags:
- Android
- Gson
- json
category: Android
thumbnailImage:
coverImage:
coverCaption:
gallery:
---

**网络请求常常伴随着数据的处理，而今 JSON 格式比较流行，就来看看 Android 中是如何去解析它的。**

<!-- more -->

### JSON 的那些事

#### 什么是 JSON？
维基百科对它的定义是：JSON（JavaScript Object Notation）是一种由道格拉斯·克罗克福特构想设计、轻量级的数据交换语言，以文字为基础，且易于让人阅读。尽管 JSON 是 Javascript 的一个子集，但 JSON 是独立于语言的文本格式，并且采用了类似于C语言家族的一些习惯。[戳我进入维基百科 JSON 词条](https://zh.wikipedia.org/wiki/JSON)

刘欣前辈在他的 [Javascript: 一个屌丝的逆袭](http://mp.weixin.qq.com/s?__biz=MzAxOTc0NzExNg==&mid=2665513059&idx=1&sn=a2eaf97d9e3000d15a33681d1b720463&scene=21#wechat_redirect) 一文中就有这么一段话：

> 后来有个好事之徒把上面的那种处理方式称为 AJAX 即 “Asynchronous Javascript And XML”（异步 JavaScript 和 XML）， 其实异步挺好， 但是 XML 就很不爽了。比如说服务器返回了下面这段 xml ：

``` xml
<book>
    <isbn>978-7-229-03093-3</isbn>
    <name>三体</name>
    <author>刘慈欣</author>
    <introduction>中国最牛的科幻书</introduction>
    <price>38.00</price>
</book>
```
> 真正的数据很少， 标签（像`<name>`这样的）反而占了大头， 把数据都给淹没了。

XML 的语义性强，也比较直观，与之相比，JSON 的主要优势在于它的体积更小，在网络上传输时就可以更省流量。可以看出 JSON 也是顺应趋势，为 JavaScript 而生，而今已在数据传输上占据一席之地。

而现在，RESTful API 也是一种流行趋势。就现在我所在的工作室，后台使用 Python 和 flask 框架，RESTful API。所以项目上正好碰上这样的 JSON 数据处理，便作下此篇。

### Gson 的那些事

#### 1. Gson 的来历
Gson（又称 Google Gson）是 Google 公司发布的一个开放源代码的 Java 库，主要用途为序列化 Java 对象为 JSON 字符串，或反序列化 JSON 字符串成 Java 对象。

#### 2. Gson 在 Android Studio 中的引入
首先是 Gson 的引入，在 Android Studio 界面按下 Ctrl + Shift + Alt + s 快捷键，调出 Project Structure 界面，选择 Modules 下 app 的 Dependencies 选项，点击 `+` 号，选择添加一个 Library dependency 。
![Project Structure](http://7xtt0k.com1.z0.glb.clouddn.com/weeklyBlog/NO.3/Project-Structure-screen-shot.png)

然后搜索所需要的第三方库即可，这里我们需要的是 gson，输入 com.google.code.gson:gson 就会看到有相应的结果显示出来了，我们直接添加导入即可。
![Choose Library Dependency](http://7xtt0k.com1.z0.glb.clouddn.com/weeklyBlog/NO.3/Choose-Library-Dependency-screen-shot.png)

当然，你也可以使用 build.gradle 或者下载并引入相应的 jar 包，这里就不再赘余了，而且也没有这方法来的快。

#### 3. Gson 的使用
Gson 的使用非常简单，只需要 new 一个 Gson() 对象即可：
``` java
Gson gson = new Gson();
Example example = gson.fromJson(exampleJsonString, Example.class);
```
详情参见 [Google 官方文档](https://github.com/google/gson/blob/master/UserGuide.md)

而反序列化 JSON 数据的重点还是在这里的 Example 实体类(或是常说的 JavaBeans )的书写。

### JavaBeans 的那些事

#### 1. 什么是 JavaBeans？
维基百科对它的定义是：JavaBeans 是 Java 中一种特殊的类，可以将多个对象封装到一个对象 ( bean ) 中。特点是可序列化，提供无参构造器，提供 getter 方法和 setter 方法访问对象的属性。名称中的 “ Bean ” 是用于 Java 的可重用软件组件的惯用叫法。[戳我进入维基百科 JavaBeans 词条](https://zh.wikipedia.org/wiki/JavaBeans)

如果看完维基百科的定义你还是觉得难以理解，这里推荐的是知乎上的一个[杨博前辈的高票解答](https://www.zhihu.com/question/19773379)，还有就是刘欣前辈写的 [Java 帝国之 Java Bean (上)]() 和 [Java 帝国之 Java Bean (下)]() 这两篇文章，不得不说刘欣前辈的文章都很风趣易懂，以小见大，有种《明朝那些事》的味道。

我对 JavaBean 的理解是，Java 为了做到所承诺的向后兼容，于是定义了这么一套规范，JavaBean 里的对象设为私有，将对象的 set 和 get 等方法暴露出来。也就是说这只是一种规范约束。而这种约束也体现了 Java 封装的思想，将对象设为私有变量，使得外部变量不能访问，收紧权限，同时又提供 set 和 get 方法来释放一些权限，使得外部可以间接的对属性值修改从而改变类内部变量的状态，这样的封装，隐藏了具体的实现细节，也降低了耦合性。

### 解析的那些事

**且看这解析三部曲：观、构、干**

#### 1. 解析第一步：观
要解析数据，第一步肯定就是观察数据啦！以下均以实际项目中的数据处理作为事例。在写图书馆功能模块时，调用后台所给的 API ，返回借阅信息的 json 数据如下：
``` json
{
    "data": [
        {
            "author": "郑斌编著",
            "barCode": "N1036673",
            "borrowDate": "2016-05-24",
            "check": "12F5319",
            "returnDate": "2016-09-18",
            "title": "黑客攻防入门与进阶"
        },
        {
            "author": "周颖等编著",
            "barCode": "A3421820",
            "borrowDate": "2016-05-24",
            "check": "757BBDA",
            "returnDate": "2016-09-18",
            "title": "程序员的数学思维修练:趣味解读"
        },
        {
            "author": "(美) Paul Graham著",
            "barCode": "N1401694",
            "borrowDate": "2016-05-24",
            "check": "63B1A99",
            "returnDate": "2016-09-18",
            "title": "黑客与画家:来自计算机时代的高见"
        },
        {
            "author": "(美) 萨旺特·辛格著",
            "barCode": "A1449697",
            "borrowDate": "2016-05-24",
            "check": "AFFA299",
            "returnDate": "2016-09-18",
            "title": "大未来:移动互联时代的十大趋势:implications for our future lives"
        },
        {
            "author": "陈根著",
            "barCode": "AN1456291",
            "borrowDate": "2016-05-24",
            "check": "7A3B7CE",
            "returnDate": "2016-09-18",
            "title": "可穿戴设备:移动互联网新浪潮"
        },
        {
            "author": "李维勇主编",
            "barCode": "AN145627",
            "borrowDate": "2016-05-24",
            "check": "E8909E9",
            "returnDate": "2016-09-18",
            "title": "Android UI设计"
        }
    ],
    "message": "done",
    "status": 1
}
```
在这里的 json 最外面是一个 {} ，里面有 ` { "data": "something", "message": "someting", "status": "something"}`，接着 data 里面是一个 [] ，里面包含着 `[ {"author": "something", "barCode": "something", "borrowDate": "something", "check": "something", "returnDate": "something", "title": "something"}, ***]`，而每一个 {} 里的键名和顺序都是一样的。

#### 2. 解析第二步：构
分析完数据，接下来就是要去构造实体类啦！

其中 {} 所对应的是一个类，[] 所对应的是 List 或者是数组，可以看到最外层是由 data 、message、status 组成，而 data 里面是一个 List 。由此看来，我们可以先写出最外层的实体类：

``` java
public class Root {
    private List<Data> data;
    private String message;
    private int status;

    public Root() {}

    public void setMessage(String message) {
        this.message = message;
    }
    public String getMessage() {
        return this.message;
    }
    public void setStatus(int status) {
        this.status = status;
    }
    public int getStatus() {
        return this.status;
    }
    public void setData(List<Data> data) {
        this.data = data;
    }
    public List<Data> getData() {
        return this.data;
    }
}
```

这就是一个标准的 JavaBean 了，提供默认的无参构造器，私有变量，以及暴露出各对象的 set 和 get 方法。

再来看 data 里面的数据，可以看出是每本书的信息所对应的键名是相同的，所以可以用同一个实体类表示，我们接着写这个 Data 类。

``` java
public class Data {
        private String author, barCode, borrowDate, check, returnDate, title;
        
        public void setAuthor(String author) {
            this.author = author;
        }
        public String getAuthor() {
            return this.author;
        }
        public void setBarCode(String barCode) {
            this.barCode = barCode;
        }
        public String getBarCode() {
            return this.barCode;
        }
        public void setBorrowDate(String borrowDate) {
            this.borrowDate = borrowDate;
        }
        public String getBorrowDate() {
            return this.borrowDate;
        }
        public void setCheck(String check) {
            this.check = check;
        }
        public String getCheck() {
            return this.check;
        }
        public void setReturnDate(String returnDate) {
            this.returnDate = returnDate;
        }
        public String getReturnDate() {
            return this.returnDate;
        }
        public void setTitle(String title) {
            this.title = title;
        }
        public String getTitle() {
            return this.title;
        }
    }
}
```

如果你要把这个 Data 类内嵌到刚才的 Root 类里面，必须声明是静态类型，不然是不能解析，示例如下。

``` java
public class Root {
    private List<Data> data;
    private String message;
    private int status;

    public Root() {}

    //这里省略 set 和 get 方法

    public static class Data {
        private String author, barCode, borrowDate, check, returnDate, title;

        //这里省略 set 和 get 方法
    }
}
```

还有一点就是，在返回的 JSON 数据中，如果有不需要的数据，在构造实体类时，是可以不需要写出来的。

#### 3. 解析第三步：干
实体类都写好了，还有什么好说的，就是干。 

``` java
public void getLibraryInfo() {
    //此方法为部分代码,且这里省略网络请求部分
    
    Response response = client.newCall(request).execute();
    String responseBody = response.body().string();
        
    Gson gson = new Gson();
    Root root = gson.fromJson(responseBody, Root.class);

    //这里是示例，取出一些数据
    status = root.getStatus();
    message = root.getMessage();
    
    List<Data> data = root.getData();
    title = data.get(1).getTitle();
    author = data.get(1).getAuthor();
    barCode = data.get(1).getBarCode();
    borrowDate = data.get(1).getBorrowDate();
    numberOfBooks = data.size();
}
```

这里有个点，是在我不懂之前一直疑惑且写了很多无用赘余的代码，就是在 `Root root = gson.fromJson(responseBody, Root.class);` 这步执行完后，相应的数据其实就已经填充到对应的实体类的变量中，所以直接 get 就可以取到数据啦！那可能有人要问了，这里的 set 方法有什么用，根本就没用上啊。只能说只是你没有碰上那个需求，所以没用上，如果说你的程序里需要在请求回来的数据里设定一些特定的值，此时你就需要用到 set 方法。

这里取出各个数据（ List 可以遍历取出），之后你需要什么要的格式和组合就凭君调遣啦！我的习惯是将这个图书馆数据查询获取模块写成一个 Util ，应用中需要用到什么样的数据，在 Util 里写一个对应的方法返回即可。
