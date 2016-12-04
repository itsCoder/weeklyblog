---
title: JitPack 指南
date: 2016-12-04 16:02:31
---
>- 文章来源：itsCoder 的 [WeeklyBolg](https://github.com/itsCoder/weeklyblog) 项目
>- itsCoder主页：[http://itscoder.com/](http://itscoder.com/)
>- 作者：[谢三弟](https://github.com/xcc3641)
>- 审阅者：[]()

### 目录

### 前言

最初发布一些 lib 是用的 jCenter ，但是每次上传都跟玄学一样，就算我全局或者指定代理还是凭运气，后来问了老司机，结识了 [JitPack](https://jitpack.io/) ，体验更加方便和易懂。

### 步骤

#### 账号

1. 首先进入网页我们看到是：

![](http://ww2.sinaimg.cn/large/006y8lVagw1faer45z2g0j31kw12542t.jpg)

然后我们进行账号登陆，默认是跟 Github 绑定的。

2. 成功之后网站会读取你的 repo 显示在左侧：

![](http://ww4.sinaimg.cn/large/006y8lVagw1faer6xve2gj31kw10itbq.jpg)


#### Gradle 配置

1. 首先在你的 root build.gradle 添加：


```gradle
buildscript {
    repositories {
        jcenter()
    }
    dependencies {
        classpath 'com.android.tools.build:gradle:2.2.2'
        classpath 'com.github.dcendents:android-maven-gradle-plugin:1.5' // 这一栏
    }
}
```

2. 接下来在你的 Lib build.gradle 添加：

```gralde
group = 'com.github.<username>' // 最好是你的 github 名字

// 指定编码
tasks.withType(JavaCompile) {
    options.encoding = "UTF-8"
}

// 打包源码
task sourcesJar(type: Jar) {
    from android.sourceSets.main.java.srcDirs
    classifier = 'sources'
}

artifacts {
    archives sourcesJar
}
```
3. 配置完成后，将你的项目 push 到 github
4. 点击 Releases 编写好之后发布，回到 JitPack

#### 发布

1. 根据你的 group/Lib 搜索你的裤子。

![](http://ww3.sinaimg.cn/large/006y8lVagw1faerh75xvtj30yu0o8ta9.jpg)

2. 点击 **get it**

![](http://ww2.sinaimg.cn/large/006y8lVagw1faerhpbkemj318u0v0gpe.jpg)

别人就可以用到你写的裤子了 (๑•̀ㅂ•́) ✧

### 参考

全文以 [Watcher: Help to watch the fps and used memory of your app](https://github.com/xcc3641/Watcher) 为实践，如果有疑问可以参考该项目模仿配置。

- [JitPack](https://jitpack.io)