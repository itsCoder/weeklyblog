### 前言

笔者是一名 Android 开发者，之前没有任何前端开发经验，也是从0开始学习一些前端知识。有兴趣的朋友可以和我一起学习，后续也会更新一些前端的学习笔记和自己的心得体会。

### React Native 环境搭建

#### 安装

**1. Homebrew**, Mac 系统的包管理器，用于安装 NodeJS 和一些其他必需的工具软件。

```c
/usr/bin/ruby -e "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install)"
```

**2. Node** ,使用 Homebrew 来安装 [Node.js](https://nodejs.org/).

```c
brew install node
```

**3. React Native 的命令行工具**

```c
sudo npm install -g react-native-cli
```

**4. Git**，如果你已经安装过 [Xcode](https://developer.apple.com/xcode/)，则 Git 也已经一并安装了。如若没有，则使用下列命令安装：

```c
brew install git
```

**5. 获取 React Native 的源代码和依赖包**

```c
react-native init AwesomeProject
```

这边执行这个命令的话会被墙，可以按下面的命令使用淘宝的镜像去执行。笔者亲测，大概2分钟创建项目完毕。

```c
npm config set registry=http://registry.npm.taobao.org/
react-native init AwesomeProject
```

#### 推荐安装

**1. Watchman**，是由 Facebook 提供的监视文件系统变更的工具

```c
brew install watchman
```

**2. Flow**,是一个静态的 JS 类型检查工具

```c
brew install flow
```

#### 测试安装

```c
cd AwesomeProject
react-native run-android
```

这边跑项目会遇到各式各样的问题，我列举一些我遇到的问题和解决方案

**1.**问题：提示 SDK location not found. Define location with sdk.dir in the local.properties file or with an ANDROID_HOME environment variable.

解决方案：在创建的 AwesomeProject 下的 android 目录下常见一个 `local.properties` 文件，文件内容加入

```c
sdk.dir = /Users/USERNAME/Library/Android/sdk
```

其中 USERNAME 为你的用户名

**2.**问题：提示 failed to find Build Tools revision 23.0.1

解决方案：打开 Android SDK，安装 Build Tools revision 23.0.1

**3.**问题：手机运行成功屏幕显示红色，提示没有连接服务器 js Server

解决方案：摇一摇手机，点击Dev Settings -> Debug server host for device ，填入你开发电脑的 IP 地址。

**4.**问题：手机运行成功屏幕显示红色，提示 Could not get BatchedBridge, make sure your bundle is packaged properly” on start of app

解决方案：我用了很多办法都没有成功，后来用系统6.0以上的手机就ok了

**5.**问题：手机运行成功且出现 Welcome to React Native！

解决方案：休息一会，恭喜你环境搭建成功了！！

### 第一个 React Native 程序

打开目录下的 index.android.js ，对代码进行相应的修改

```jsx
import React, { Component } from 'react';
import {
  AppRegistry,
  StyleSheet,
  Text,
  View
} from 'react-native';

export default class AwesomeProject extends Component {
  render() {
    return (
      <View style={styles.container}>
        <Text style={styles.welcome}>
          Welcome to React Native!
        </Text>
        <Text style={styles.instructions}>
          JangGwa
        </Text>
        <Text style={styles.instructions}>
          一起从0开始学习 React Native
        </Text>
      </View>
    );
  }
}

const styles = StyleSheet.create({
  container: {
    flex: 1,
    justifyContent: 'center',
    alignItems: 'center',
    backgroundColor: '#F5FCFF',
  },
  welcome: {
    fontSize: 20,
    textAlign: 'center',
    margin: 10,
  },
  instructions: {
    textAlign: 'center',
    color: '#333333',
    marginBottom: 5,
  },
});

AppRegistry.registerComponent('AwesomeProject', () => AwesomeProject);
```

最后说一下如何调试，当我们修改完代码，不需要重新部署，如果是在真机上调试，可以摇一摇弹出菜单然后点击 Reload；如果在虚拟机上可以按两下 R 键进行 Reload，也可以通过输入命令`adb shell input keyevent 82`弹出菜单。