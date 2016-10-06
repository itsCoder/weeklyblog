---
title: 码农必知之上传开源库到 jcenter
date: 2016-09-11 09:12:19
categories: Android
tags: [jcenter ,gradle ,maven]
---

## 准备
跟着老司机学习一段时间 Android 后也想写一个自己的开源库，爬过无数坑之后终于完成了自己的第一个开源库。此时你肯定会想，我该怎么将它开源出去，让更多人来使用它呢？毕竟一个人的 coding 不是编程。吼吼，jcenter 可以帮到你。

> 开源让编程更加快乐

## 什么是 jcenter？
在 Android 开发过程中我们肯定会添加依赖库。其中依赖库添加有三种方式。

1. compile project(':tagflowlayout')
2. compile files('libs/nineoldandroids-2.4.0.jar')
3. compile 'com.hymane.expandtextview:library:1.0.1'

第一种方式是引用 module，将本地一个 module 导入到当前项目内；第二种是引用本地 jar 包，这种方式也很常见；最后一种是远程库依赖方式，添加依赖十分简单，而且依赖的后期更新也很方便，只要修改一下库的版本号就可以了。

比如 app 需要依赖下面这个远程库，
 
```java
	dependencies {
		compile 'com.hymane.expandtextview:library:1.0.1'
	}
```

这样定义了，这样我们就可以使用这个库了，那么我们是去哪里拿到库工程的代码和资源文件呢？肯定是要从某个源去获取。

jcenter 是一个声明仓库的源，之前版本则是 mavenCentral(), jcenter 可以理解成是一个新的中央远程仓库，兼容maven中心仓库，而且性能更优。只要我们将我们的库上传到 maven 库，然后再添加到 jcenter 上就可以直接添加依赖来使用了。

## 开始

### 申请 Bintray 账号
Bintray 的基本功能类似于 Maven Central，一样的我们需要一个账号，[Bintray传送门](https://bintray.com)，自行准备梯子，注册完成后第一步算完成了，建议直接使用 [Github](https://github.com) 账号登录。

### 本地项目的 jcenter 配置
这里默认你的开源项目已经完成。我们接下来需要上传我们的 JavaDoc 和 source JARs。

#### 1：先修改项目的 build.gradle 文件，添加两个插件

```java
	buildscript {
	    repositories {
	        jcenter()
	    }
	    dependencies {
	        classpath 'com.android.tools.build:gradle:2.1.3'
	        classpath 'com.jfrog.bintray.gradle:gradle-bintray-plugin:1.7'
	        classpath 'com.github.dcendents:android-maven-gradle-plugin:1.5'
	        // NOTE: Do not place your application dependencies here; they belong
	        // in the individual module build.gradle files
	    }
	}
	
	allprojects {
	    repositories {
	        jcenter()
	    }
	}
	
	task clean(type: Delete) {
	    delete rootProject.buildDir
	}
```

最好使用最新版本的插件，因为版本不对有时候会出现上传不了项目等问题。想知道最新版本的插件版本号请去 [gradle-bintray-plugin](https://github.com/bintray/gradle-bintray-plugin) 和 [android-maven-gradle-plugin](https://github.com/dcendents/android-maven-gradle-plugin)，

#### 2： 然后在你需要发布的那个 module（即需要发布的 library）的 build.gradle 里配置如下内容：

```java
	apply plugin: 'com.android.library'
	apply plugin: 'com.github.dcendents.android-maven'
	apply plugin: 'com.jfrog.bintray'
	
	version = "0.2.0"
	android {
	    compileSdkVersion 24
	    buildToolsVersion "24.0.2"
	
	    defaultConfig {
	        minSdkVersion 15
	        targetSdkVersion 24
	        versionCode 2
	        versionName version
	    }
	    buildTypes {
	        release {
	            minifyEnabled false
	            proguardFiles getDefaultProguardFile('proguard-android.txt'), 'proguard-rules.pro'
	        }
	    }
	}
	
	dependencies {
	    compile fileTree(dir: 'libs', include: ['*.jar'])
	    testCompile 'junit:junit:4.12'
	    compile 'com.android.support:appcompat-v7:24.2.0'
	}
	
	def siteUrl = 'https://github.com/Hymanme/TagFlowLayout' // 项目的主页
	def gitUrl = 'https://github.com/Hymanme/TagFlowLayout.git' // Git 仓库的 url
	// Maven Group ID for the artifact，填你唯一的groupid，
	// 强烈建议填写`com.github.yourusername`,其中 yourusername 为你 github 用户名
	group = "com.github.hymanme.tagflowlayout" 
	install {
	    repositories.mavenInstaller {
	        // This generates POM.xml with proper parameters
	        pom {
	            project {
	                packaging 'aar'
	                // Add your description here
	                name 'Android TagFlowLayout Widget' //项目描述
	                url siteUrl
	                // Set your license
	                licenses {
	                    license {
	                        name 'The Apache Software License, Version 2.0'
	                        url 'http://www.apache.org/licenses/LICENSE-2.0.txt'
	                    }
	                }
	                developers {
	                    developer {
	                        id 'hymanme'    //填写的一些基本信息
	                        name 'hymane'
	                        email 'hymanme@163.com'
	                    }
	                }
	                scm {
	                    connection gitUrl
	                    developerConnection gitUrl
	                    url siteUrl
	                }
	            }
	        }
	    }
	}
	task sourcesJar(type: Jar) {
	    from android.sourceSets.main.java.srcDirs
	    classifier = 'sources'
	}
	task javadoc(type: Javadoc) {
	    options.encoding = "utf-8"
	    source = android.sourceSets.main.java.srcDirs
	    classpath += project.files(android.getBootClasspath().join(File.pathSeparator))
	}
	task javadocJar(type: Jar, dependsOn: javadoc) {
	    classifier = 'javadoc'
	    from javadoc.destinationDir
	}
	artifacts {
	    archives javadocJar
	    archives sourcesJar
	}
	Properties properties = new Properties()
	properties.load(project.rootProject.file('local.properties').newDataInputStream())
	bintray {
	    user = properties.getProperty("bintray.user")
	    key = properties.getProperty("bintray.apikey")
	    configurations = ['archives']
	    pkg {
	        repo = "maven"
	        name = "TagFlowLayout"    //发布到JCenter上的项目名字
	        websiteUrl = siteUrl
	        vcsUrl = gitUrl
	        licenses = ["Apache-2.0"]
	        publish = true
	    }
	}
	apply plugin: 'maven'
```

### 3： 配置 apikey
配置好后需要在你的项目的根目录的 local.properties 文件里（一般这文件需添加到 gitignore，防止泄露账户信息）配置你的 bintray 账号信息，username 为你的用户名，apikey 为你 Bintray 账户的 apikey ，可以点击进入你的账户信息里再点击 Edit 即可在左边查看 API Key 的选项，把他复制下来。

![jcenter1](http://ww4.sinaimg.cn/mw690/005X6W83gw1f7pexb9glnj30l806bmyg.jpg)

![jcenter2](http://ww1.sinaimg.cn/mw690/005X6W83gw1f7pexawt20j305f0b5wes.jpg)

在 local.properties 中添加

```java
	bintray.user=username
	bintray.apikey=apikey
```

### 4: 编译并上传到 Bintray

Rebuild 一下，然后执行如下命令( Windows 中)完成上传：

```java
	./gradlew install
	./gradlew bintrayUpload
```

第一条命令是下载必需库，只需要执行一次就可以，他会下载一些文件下来，以后每次更新只需要更新代码然后修改一下 version ，再`./gradlew bintrayUpload`一下就可以了。

### 5： 添加到jcenter
以上如果上传成功之后，代码会存在于 maven 库内，还差最后一步，我们需要向 bintray 提出添加到 jcenter 的申请。 进入[这个页面](https://bintray.com/bintray/jcenter)点击 include My package,输入你的库的名字比如我这里填的是 `TagFlowLayout` 然后下面会列出结果点。点击匹配到的项目，写一个 Comments(写不写无所谓) 再 send，然后就坐等管理员审核了，一般等40分钟左右（美国工作时间）会有一个通过的消息发给你，这就可以了。

![jcenter3](http://ww3.sinaimg.cn/mw690/005X6W83gw1f7pf9yj2cnj30oc07hjrx.jpg)

### 5： 大功告成
现在就可以添加你的依赖到 build.gradle 试试了

```java
	dependencies {
        compile 'com.github.hymanme.tagflowlayout:tagflowlayout:0.2.0'
    }
```
## 总结
Android 开发中添加 jcenter 依赖库可以说是一件十分常见且方便的事，自己亲自创建一个开源库到 jcenter 也是一件很有意思的事。