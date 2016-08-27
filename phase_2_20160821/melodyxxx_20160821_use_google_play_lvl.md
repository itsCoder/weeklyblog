title: 从注册Google Play开发者到如何使用Google LVL验证服务
date: 2016-08-21 15:21:21
tags: [Android]
toc: true
---

如果你想上架付费应用到Google Play，License Verification Library (LVL)是你必不可少的。

<!--more-->

>- 文章来源：itsCoder 的 [WeeklyBolg](https://github.com/itsCoder/weeklyblog) 项目
>- itsCoder主页：[http://itscoder.com/](http://itscoder.com/)
>- 作者：[Melodyxxx](http://melodyxxx.com/)
>- 审阅者：[暂无]()

## 关于License Verification Library (LVL)

### 什么是LVL？
**License Verification Library (以下简称LVL)**是谷歌提供给开发者用于验证用户的合法身份的一个机制，比如你想上架一款付费应用(非付费应用也可以使用，但是这个一般没必要)到Google Play，为了防止APK的到处传播，你可以使用此证书验证机制，每次打开应用时，都会对当前用户的合法性进行授权检查。

### LVL的工作流程

![](http://i.imgur.com/rq5Zvpy.png)

上图来自Google的官方介绍。简单来说，验证时：
1. 应用通过Binder远程绑定到**Google Play Service**(所以用户的设备上必须装有Google Play)
2. 然后Google Play Service会将用户的信息和应用的数据去**Market License Server**(谷歌的证书服务器)去查询验证
3. 然后服务器返回给Google Play Service的证书状态(是否已经授权过的)
4. 然后Google Play Service再返回给我们的应用客户端，回调对应的方法，我们可以通过这些回调来处理对应的逻辑，例如未授权的就提示用户不允许继续使用此应用，然后自动退出应用等等
5. 用户付费购买时会在证书服务器有记录，所以未付费用户直接安装请求验证时，会返回未授权的状态

## 注册Google Play开发者

### 注册Google Play开发者条件
- 科学上网
- 一个Gmail邮箱
- 一张信用卡(如果没有也没关系，下面会说道)

### 注册步骤

1. 首先登陆：[Google Play 开发者控制台](https://play.google.com/apps/publish/?hl=zh),然后登陆你的Google账号，会自动跳转到开发者注册流程页面(前提你没有注册过开发者)
2. 然后填写相关信息，因为注册开发者账号，谷歌会收取25美元(约合人民币166元)作为注册费用，所以这里需要填写自己的信用卡信息，如果你是学生等原因没有信用卡，这里推荐一下万事达全球付，可以自己申请获取一个虚拟卡，获取成功会给你卡号，安全码等信息，然后可以使用此卡支持注册费用。具体可以自行网上搜索。
3. 注册完后会直接来答Google开发者页面，这里并不代表你已经成功注册开发者账号了，这个需要审核的，具体是否注册成功可以去Google Wallet查看自己的25美元订单信息，订单已完成说明审核通过了。

## 上传应用到Alpha/Beta渠道

注册完开发者后，你可以点击开发者界面首页的"添加应用"，创建你的应用
![](http://i.imgur.com/6hcQZYF.png)

然后来到下面的界面

![](http://i.imgur.com/EjqgJyJ.png)

需要注意的是这里创建的应用称为"草稿"，即未发布的(包括未发布在Alpha或者Beta渠道的)应用，由于Google的最新规定，草稿应用不能使用LVL测试，所以如果不把APP发布到Alpha或者Beta渠道(由于正式发布之前需要进行LVL测试，所以不能发布到正式版渠道，正式版渠道是所有用户都能在Google Play搜索到你的应用的)，所以你需要使用LVL测试之前，需要发布你的APP到Alpha或者Beta渠道，你可以点击右上角的"为什么不能发布？"，系统会罗列出你达到发布之前未达到的条件，你只需要按照他的条件一步一步去完成，然后发布到Alpha或者Beta渠道，提交发布后，等上几个小时应该就能审核完成(48小时之内)。然后下面就进行LVL测试

## 下载LVL并配置到项目中

### 下载安装LVL库

在进行LVL测试之前，你需要下载LVL库，这里从SDK Manager里面下载即可,找到Google Play License Library，然后Install安装
![](http://i.imgur.com/8B1sv7z.png)

### 导入到项目中

安装完成后在你的`\your sdk dir\extras\google`目录下会有一个`market_licensing`目录，里面包含了library和对应的sample文件，这里我们先把library导入到我们的项目中去，为了保证不修改原library，我这里拷贝了一份library目录到桌面。这里为了演示，已经提前创建好了一个LVLTest项目，然后再project内新建一个Module，如下所示：

点击File->New->Import Module

![](http://i.imgur.com/KZvFxU5.png)

目录我这里选的是刚刚拷贝到桌面的library目录，你也可以直接选sdk目录内的library，然后Module name这里自己填，我填的是lvlm

![](http://i.imgur.com/vZPIxdd.png)

然后Next，然后其他的别动，直接点击Finish，然后AS会自动Sync Gradle Files，一般这里肯定会报错的

![](http://i.imgur.com/G6cerZg.png)

很明显，我这里是Build Tools版本不对，这里需要修改下刚刚导入进来的Module内的build.gradle文件

![](http://i.imgur.com/87FVdLp.png)

```
buildToolsVersion "19.1.0"
```
将buildToolsVersion版本换成自己app内的build.gradle(不是project内的)对应的版本即可，这里我的是:
```
buildToolsVersion "24.0.0"
```
另外**把compileSdkVersion改成21即可**，不要改成23以上的，因为Android6.0+移除了httpclient，要不然等会lvl module还会报错。

然后在app的build.gradle的dependencies依赖内添加上module，这步不要少，也是官方文档未提到的地方:
```
compile project(':lvlm')
```

然后手动点击同步Gradle，然后不出意外就不会出任何错误了

至此，成功将LVL导入到项目中去，当然还有其他的方法导入，具体可以参考[官方文档](https://developer.android.com/google/play/licensing/setting-up.html)

## 项目中实现LVL验证

配置完后，下面开始正式在项目中实现LVL验证机制

### 配置权限

首先添加下面权限：
```
<uses-permission android:name="com.android.vending.CHECK_LICENSE"/>
```
另外需要说下，这里只需要这一个权限即可，有人会问为什么不需要INTERNET联网权限，验证不是要通过网络的吗？这里不要网络权限是因为验证时会远程绑定到Google Play的服务的，底层通过Binder的，具体验证操作是通过Google Play Service来完成的，自己是不需要进行网络请求操作的。

### 使用Sample参考代码

下面开始编码来实现，编码之前，可以参考刚刚下载的sdk内的sample目录内的演示项目的MainActivity.java内的代码。下面时我移除了一些sample中的代码，为了看得方便，代码中已经详细注释：

``` java
    // 这里填你的PUBLIC_KEY，具体的获取方法，见文章下面
    private static final String BASE64_PUBLIC_KEY ="YOUR PUBLIC KEY";
    // 盐(这里指参与加密的一组随机数)，这里应该是和验证的缓存有关，这里可以随便修改，也可以每次打开应用验证时随机成功
    private static final byte[] SALT = new byte[]{
            -43, 65, 36, -128, -103, -57, 74, -68, 51, 28, -26, -96, 37, -117, -33, -113, -44, 32, -64,
            89
    };

    // 证书检查器
    private LicenseChecker mChecker;
    // 证书检查回调
    private LicenseCheckerCallback mLicenseCheckerCallback;
    // 证书检查后用于更新UI的Handler，因为回调方法是运行在Binder线程池中的，不能直接更新UI，所以需要借助Handler
    private Handler mHandler;


    /**
     * 初始化证书，可在onCreate()方法内调用
     */
    private void init() {
        mHandler = new Handler();
        // 获取设备id
        String deviceId = Settings.Secure.getString(getContentResolver(), Settings.Secure.ANDROID_ID);
        // Library calls this when it's done.
        mLicenseCheckerCallback = new MyLicenseCheckerCallback();
        // Construct the LicenseChecker with a policy.
        mChecker = new LicenseChecker(
                this, new ServerManagedPolicy(this,
                new AESObfuscator(SALT, getPackageName(), deviceId)),
                BASE64_PUBLIC_KEY);
        // 如果顺便再初始化的时候立即验证授权(也可以手动调用doCheck()检查授权)
        doCheck();
    }

    // 开始检查
    private void doCheck() {
        // 检查
        mChecker.checkAccess(mLicenseCheckerCallback);
    }

    /**
     * 显示UI
     */
    protected void displayUI() {
        mHandler.post(new Runnable() {
            @Override
            public void run() {
                // 在这里可以显示对话框或其他信息，用于提示用户，是否已经授权通过
            }
        });
    }

    private class MyLicenseCheckerCallback implements LicenseCheckerCallback {

        // 授权通过会回调此方法
        public void allow(int policyReason) {
            if (isFinishing()) {
                // Don't update UI if Activity is finishing.
                return;
            }
            // do something...
        }

        // 授权不通过会回调此方法
        public void dontAllow(int policyReason) {
            if (isFinishing()) {
                // Don't update UI if Activity is finishing.
                return;
            }
            // do something...
        }

        // 一般是你的编码有问题导致错误时会回调此方法
        public void applicationError(int errorCode) {
            if (isFinishing()) {
                return;
            }
            // do something...
        }
    }
```

### 如何获取你的PUBLIC_KEY

首先登陆在自己的开发者控制台界面，进入应用，然后点击左侧的"服务和API"，即可看到：
![](http://i.imgur.com/ISa9idL.png)

然后复制到代码中替换`YOUR PUBLIC KEY`，**注意，千万不要复制出多余的空格，如果有，请删除**。

### Android5.0+需要注意的地方

如果你运行程序在Android5.0+的设备上，你会发现应用直接挂掉了，原因是，library中的远程绑定服务时使用的是隐式绑定，而Android5.0+已经不在支持隐式远程绑定了，所以我们需要修改一下library的源码：

首先搜索定位到**LicenseChecker.java**文件的**checkAccess()**方法，对应于源文件的**第153行**：
``` java
 Base64.decode("Y29tLmFuZHJvaWQudmVuZGluZy5saWNlbnNpbmcuSUxpY2Vuc2luZ1NlcnZpY2U="))),
```
为其指定Goolge Play的包名package name，修改如下所示：
``` java
Base64.decode("Y29tLmFuZHJvaWQudmVuZGluZy5saWNlbnNpbmcuSUxpY2Vuc2luZ1NlcnZpY2U="))).setPackage("com.android.vending"),
```

然后即可正常运行应用

### 测试以及对应的测试响应码说明

上面的代码还有具体的处理逻辑没有写，所以自己测试之前，请参考SDK内的sample的处理逻辑。

为了模拟各种授权的返回结果，Google为开发者提供了授权测试，可以模拟各种授权结果，以便开发者在应用发布之前处理好各种结果的逻辑，可以在开发者界面左侧的设置内的最下方即可找到，这里的Gmail账号不需要填，因为开发者自己的账号本身就在测试范围之内，如果你手机登录的其他Gmail账号，则需要将账号填进去即可，多个账号用逗号隔开

![](http://i.imgur.com/9LeluW8.png)

如图所示谷歌给我们提供了一些测试响应，下面列出几个常用的响应：

响应|具体说明|
----|------|
RESPOND_NORMALLY|模拟正常响应|
LICENSED|模拟已授权，对应于allow()方法回调|
NOT_LICENSED|模拟未经授权的，对应于dontAllow()方法回调|
ERROR_NOT_MARKET_MANAGED|此响应可模拟APP未发布到Google Play的情况，如果你之前没将APP发布到ALPHA或者BETA渠道，就会返回此响应，对应applicationError()方法回调|


## 最后一点微小的建议
Google的LVL验证并不是很安全的，其实是很脆弱的，例如可以使用幸运破解器(Lucky Patcher)轻松移除LVL验证，破解后无论怎样，你的allow()方法总是会被回调，所以如果你想发布付费应用到Google Play，这里推荐你几点可以**适当防止**盗版的传播：
- 首先使用LVL验证
- 代码内检查签名，若发现签名不一致被修改，立马关闭程序或者友情提醒，具体方法可以网上搜索(**这个方法最有效，可有效防止盗版APK的传播**)
- 代码内检查当前设备是否已安装幸运破解器，若已安装可以不让用户使用
- 代码内检查当前设备是否已安装Google Play，没有安装可以判定为盗版
- 使用网上各厂商的加固，可以防止被反编译

> 本文参考资料或者想了解具体内容请参阅: https://developer.android.com/google/play/licensing/index.html


