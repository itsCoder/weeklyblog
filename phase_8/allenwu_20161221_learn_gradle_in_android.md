---
layout: post
title: 利用 Gradle 来构建编译应用程序
date: 2016-12-21 11:42:46 +0800
categories: Android
---

系统学习一下 Android Studio 中的 Gradle 相关知识，主要是跟随 《神兵利器》这本书总结的一些知识点进行实战。本篇博客系统的学习一下与 Gradle 相关的知识点,体会一下其带来的好处和方便。文章主要分为以下知识点：

* 构建 buildTypes
* 多渠道打包
* 配置签名
* 混淆 APK
* 批量命名 APK
* 为不同版本添加不同的代码
* 解决最大方法书的限制

### 开始

新建一个 GradleDemo，以 Android 方式展开 Project，如下所示，红色部分就是主要与 Gradle 相关的配置和文件区域啦：

![](http://ww3.sinaimg.cn/large/b10d1ea5jw1f9zzvyyf4rj20t50i2dkc.jpg)

下面分别讲讲一下以上每一个与 gradle 相关文件的说明和作用：

- build.gradle(Project) : 项目全局的 gradle 文件，在文件内部中的 buildScript 中 gradle 指定了 jcenter 代码仓库，同时声明了依赖的 Android gradle 的插件版本;
- build.gradle(Module): 
  - apply plugin 领域描述了 gradle 所引入的插件
  - android 领域描述了该 Android studio 构建过程中所用到的参数。默认情况下IDE 自动创建了 complieSdkVersion、buildToolsVersion 这两个参数，分别对应 sdk 和 android build tools 版本。
  - dependencies 领域描述了该 android module 构建过程中所依赖的所有库，当然其也可以以 jar 形式或者 aar 形式依赖。
- local.properties 该文件配置了 android gradle 插件所需要使用的 android sdk 的路径。

### 构建 buildTypes

构建类型基础，主要配置好 gradle 环境，在 IDE 自带的命令行环境执行 gradle build 命令，进行基本的编译打包工作，如下所示生成 debug 和 release 两个版本：

![](http://ww3.sinaimg.cn/large/b10d1ea5jw1f9zyc3ivstj20sh0dvdll.jpg)

gradle 同样支持自定义创建新的构建类型，如下所示，在脚本中增加一个 vino 类型，生成的 apk 如下图，左边所示：

![](http://ww4.sinaimg.cn/large/b10d1ea5jw1f9zyh4q17kj20st0fp0yh.jpg)

我们理解下 上图中的 applicationIdSuffix 参数的作用。android 中系统是通过包名来区分应用的，如果包名相同，就以为这是同一个应用，因此在构建类型的时候，可以指定 applicationIdSuffix 参数为默认的包名增加一个后缀，例如上述例子，加上了 vino 后缀。

### 多渠道打包

创建渠道占位符，在 Androidmainifest 文件下的 Application 节点下面，创建如下所示的 meta-data 节点。

```xml
 <meta-data android:name="UMENG_CHANNEL" android:value="${UMENG_CHANNEL}"></meta-data>
```

接下来配置 gradle 脚本，脚本位置与 gradle 脚本的 android 领域当中，使用 manifestPlaceholders 指定要替换的渠道占位符的值：

```shell
// 多渠道打包
    productFlavors {
        GooglePlay {
            manifestPlaceholders = [UMENG_CHANNEL: "GooglePlay"]
        }
        Baidu {
            manifestPlaceholders = [UMENG_CHANNEL: "Baidu"]
        }
        Wandoujia {
            manifestPlaceholders = [UMENG_CHANNEL: "Wandoujia"]
        }
        Xiaomi {
            manifestPlaceholders = [UMENG_CHANNEL: "Xiaomi"]
        }
    }
```

在终端执行 gradle build 即可开始构建，发现每个渠道包都有三种(系统自带的两种和一种自定义的)。

![](http://ww3.sinaimg.cn/large/b10d1ea5jw1f9zzdupocuj20s20j17ba.jpg)

### 构建 SigningConfigs

使用 Apk 签名来保证 app 的合法性。按照如下所示生成签名，将生成的签名文件保存在主 module 的根目录下，这样可以较方便的引用起来：

![](http://ww2.sinaimg.cn/large/b10d1ea5jw1f9zz0drdr7j20en0dyabs.jpg)

生成了签名文件之后，就可以在 build.gradle 脚本的 android 领域当中配置签名相关的参数了，即如下箭头所示的 signingConfig 括号内的配置：

![](http://ww1.sinaimg.cn/large/b10d1ea5jw1fatlkln0ohj20sg0j6q9w.jpg)

如上所示脚本设置了 vino 构建类型的签名，没有给 release 构建类型设置签名，因此在 gradle build 命令之后，生成的签名apk 如下所示：

![](http://ww1.sinaimg.cn/large/b10d1ea5jw1f9zyzxlcaqj20st0jun3e.jpg)

### 构建 Proguard

Proguard 配置在 Android 中起到的是混淆 apk 的作用，但是其作用不仅仅只是混淆 apk 代码，其可以精简代码，资源，优化代码结构。在 android studio 中启用默认的混淆机制很简单，只需要在 minifyEnabled 后面启用 true 即可。

```shell
buildTypes {
        release {
            minifyEnabled true
            proguardFiles getDefaultProguardFile('proguard-android.txt'), 'proguard-rules.pro'
        }

        vino {
            signingConfig signingConfigs.vino
            applicationIdSuffix ".vino"
        }
    }
```

如上配置在构建 release 版本时候，系统就会自动进行混淆，混淆文件的地址，可以通过 getDefaultProguardFile 来获取。默认的混淆文件配置模板在 sdk 的 tools/proguard 目录下找到。

### 生成重命名包

在生成渠道包之后，生成的包名通常是默认的命名：app-渠道名-buildType.apk。但是还有些需求就是对它们进行重新命名，满足市场部的要求，此时就可以用 gradle 进行快速重命名：

![](http://ww2.sinaimg.cn/large/b10d1ea5jw1f9zzqrgl21j20s20j17bn.jpg)

将以下代码放置于 android 领域即可，当执行 gradle build 指令时候，该 task 也会执行。这段脚本提取出所有生成的 apk 包，判断其是否为 apk，是否是 release 版本。如果是则将其重命名为 VinoApp-渠道名-ver 版本号.apk 。

```groovy
 applicationVariants.all { variant ->
                variant.outputs.each { output ->
                    def outputFile = output.outputFile
                    if (outputFile != null && outputFile.name.endsWith('.apk')) {
                    	// 输出指定的命名格式 apk 文件
                      def fileName = "VinoApp-${variant.flavorName}-ver${variant.versionNmae}.apk"
                        output.outputFile = new File(outputFile.parent, fileName)
                    }
          }
}
```

生成的 apk 如下所示。这里注意一下，以上代码将 _ 改成了 -，markdown 语法会对 _ 做解析，所以用 - 替代。

![](http://ww3.sinaimg.cn/large/b10d1ea5jw1fatm270bqdj20sg0j6dm3.jpg)

### 为不同版本添加不同的代码

在开发过程中，不同的版本有不同的功能。利用常用的 log 开关，在 debug 版本中会打开开发日志，但是在发行版本中就会关闭。

```groovy
buildTypes {
        debug {
            // 显示Log
            buildConfigField "boolean", "LOG_DEBUG", "true"
            versionNameSuffix "-debug"
          	// 开启混淆
            minifyEnabled false
            zipAlignEnabled false
            // 移除无用资源
            shrinkResources false
            signingConfig signingConfigs.debug
        }

        release {
            // 不显示Log
            buildConfigField "boolean", "testFlag", "false"
			// 开启混淆
            minifyEnabled true
            zipAlignEnabled true
            // 移除无用的resource文件
            shrinkResources true
            // 混淆文件路径
            proguardFiles getDefaultProguardFile('proguard-android.txt'), 'proguard-rules.pro'
            signingConfig signingConfigs.release
            }
}
```

通过指定 buildConfigFiled 的三个参数--类型、名称、值，就可以将一个变量设置到不同的 buildType 中去。打开系统的 BuildType 类，可以看到不同的 buildType 下面对应的 testFlag 的数值

```java
package com.allenwu.gradledemo2;

public final class BuildConfig {
  public static final boolean DEBUG = false;
  public static final String APPLICATION_ID = "com.allenwu.gradledemo2";
  public static final String BUILD_TYPE = "release";
  public static final String FLAVOR = "GooglePlay";
  public static final int VERSION_CODE = 1;
  public static final String VERSION_NAME = "1.0";
  // Fields from build type: release
  public static final boolean testFlag = true;
}
```

BuildType 为 vino 的 BuildConfig 的类：

```java
package com.allenwu.gradledemo2;

public final class BuildConfig {
  public static final boolean DEBUG = false;
  public static final String APPLICATION_ID = "com.allenwu.gradledemo2.vino";
  public static final String BUILD_TYPE = "vino";
  public static final String FLAVOR = "GooglePlay";
  public static final int VERSION_CODE = 1;
  public static final String VERSION_NAME = "1.0";
  // Fields from build type: vino
  public static final boolean testFlag = false;
}
```

### 解决方法数超过 65535 的限制

在 Android 域下的 defaultConfig 下添加如下代码：

``` java
 // dex 突破 65535 的限制
multiDexEnabled true
```

引入依赖库：

``` java 
//引入解决方法数超过 65535 限制的库
compile 'com.android.support:multidex:1.0.0'
```

接着在 Application() 下的 onCreate() 下添加如下代码：

``` java
@Override
public void onCreate() {
    //MultiDex 支持 65535 方法数量限制
    MultiDex.install(getApplicationContext()); 
    super.onCreate(); 
}
```

这个我还没测试过，期待有测试过的朋友，能够反馈一下。

好了，基本上都是使用 gradle 来进行编译打包相关知识点，后续还可以继续学习如下知识点，大都可以自行 google 到：

1. gradle 加速
2. 增加编译内存
3. 使用 gradle 精简资源
4. 清除 gradle 缓存
5. 使用 gradle 本地缓存
6. gradle 资源冲突

另外 gradle 最大的特点就是可以自定义开发插件，貌似 IDEA 对 gradle 插件开发更加友好，以后需要时候再研究吧！

以上，谢谢阅读！！！

**参考文章：《Android群英传-神兵利器-第四章》**