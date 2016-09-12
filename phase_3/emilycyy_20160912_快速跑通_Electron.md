> Electron 是一款可以利用 Web 技术 开发跨平台桌面应用的框架，最初是 Github 发布的 Atom 编辑器衍生出的 Atom Shell，后更名为 Electron 。Electron 提供了一个能通过 JavaScript 和 HTML 创建桌面应用的平台，同时集成 Node 来授予网页访问底层系统的权限。目前常见的有 [NW](http://nwjs.io/)、[heX](http://hex.youdao.com/zh-cn/index.html)、[Electron](http://electron.atom.io/)，可以打造桌面应用。

![Electron](http://upload-images.jianshu.io/upload_images/1896270-a05736e88054b1e4.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


#一、环境搭建
它的方式是使用 nodeJs API 调用系统资源，所以第一步就是安装 node.js 环境哟。
##1.1、安装 node.js
如果你的机器上还没有 **[Node.js® 和 npm](https://nodejs.org/en/download/)** ，请安装它们。
##1.2、全局安装 electron
```
npm install -g electron-prebuilt
```
mac 系统需要在管理员权限下安装哟，输好密码就可以开始等他安装了。
```
sudo npm install -g electron-prebuilt
```
全局安装后就可以在命令行使用 electron 工具
全局安装之后，就可以通过 electron .  启动应用，当然也可以选择局部安装。
##1.3、安装打包工具 electron-packager
```
npm install -g electron-packager
```
同样，mac 系统需要在管理员权限下安装哟
```
sudo npm install -g electron-packager
```
打包需要注意的点会在后面讲解
#二、简单开发
那么，我们 Electron 程序到底是怎么跑起来的呢？先看下一个 Electron 项目的基本框架组成吧。
##2.1、项目框架
参看官方的 demo ，一个 Electron 应用的目录结构大体如下：
```
app/
├── package.json
├── main.js
└── index.html
```

- **package.json**
  可以理解为 android 里面的 mainfest 文件，里面声明了程序的名称、简介、版本等信息；设置 Electron 主进程运行的脚本（main.js），即设置程序的入口；设置快捷键，在你的 CLI（命令行）中可以用 electron . 方便地启动应用。可以看下面例子：
```
{
    "name": "channel",
    "version": "1.0.0",
    "description": "",
    "main": "main.js",
    "author": "Young",
    "scripts": {
        "start": "electron ."
    }
}
```

- **main.js**
  这个文件是程序的入口，Electron 的主进程将用它来启动并创建桌面应用。
```
const {app, BrowserWindow} = require('electron')
let win
function createWindow(){
    win = new BrowserWindow({width:800, height:600})
    win.loadURL(`file://${__dirname}/index.html`)
    win.webContents.openDevTools()//开启调试工具
    win.on('close', () => {
        win = null
    })
    win.on('resize', () => {
        win.reload()
    })
}
app.on('ready', createWindow)
app.on('window-all-cloased', () => {
    if(process.platform !== 'drawin' ){
        app.quit()
    }
})
app.on('activate', () => {
    if(win === null){
        createWindow()
    }
})
```
app 模块：会控制应用的生命周期（例如， 对应用的 ready 状态做出反应），有点像 Application ，对应有各个生命周期会有不同的状态。

 BrowserWindow 模块：为你创建窗口。

 win 对象：是你应用的主窗口，被声明成 null ，否则当 JavaScript 垃圾回收掉这个对象时，窗口会被关闭。

 当 app 捕获 ready 事件，BrowserWindow 创建一个800*600大小的窗口。浏览器窗口的渲染进程会渲染 index.html 文件。

 当 app 捕获 resize 事件，BrowserWindow 会重新加载，以此类推。

- **index.html**
  这个文件就是我们要呈现出来的网页了。这就需要你自己发挥想象写咯~

##2.2、初次开发踩坑记录
- **Electron 加载带 jquery 的项目报错**
  solution： [详细解答可以查看这里](https://github.com/electron/electron/issues/254#issuecomment-183483641)
```
<!-- Insert this line above script imports -->
<script>if (typeof module === 'object') {window.module = module; module = undefined;}</script>
<!-- normal script imports etc -->
<script src="scripts/jquery.min.js"></script> 
<script src="scripts/vendor.js"></script> 
<!-- Insert this line after script imports -->
<script>if (window.module) module = window.module;</script>
```
**Benefits**
Works for both browser and electron with the same code
Fixes issues for ALL 3rd-party libraries (not just jQuery) without having to specify each one
Script Build / Pack Friendly (i.e. Grunt / Gulp all scripts into vendor.js)
Does NOT require node-integration
 to be false

- **Electrons 使用网络请求**
  solution：原来也有很多库提供给我们使用的，使用方式方法也很简单。
  **superagent**
  [superagent](https://github.com/visionmedia/superagent) 是一个极其简单的 AJAX 库。看过下面例子，相信机智的你已经知道怎么使用了吧。
```
var request = require( 'superagent' );
request 
.get( 'http://xxx.com' ) 
.end( function ( err, res ) { 
// Do something
});
```
**bluebird**
[bluebird](https://github.com/petkaantonov/bluebird/) 是一个 Promise 库。
>凡是类似 IO 的操作，必定需要异步。经典的解决方法是回调，然而是时候用 Promise 了！
>bluebird 声称拥有无与伦比的速度。其实更实用的功能是它支持能够将一些本身是不支持 Promise 的库转化为支持 Promise 的库。
>然而，要配合之前的 superagent，则需要另外一个库 [superagent-bluebird-promise](https://github.com/KyleAMathews/superagent-bluebird-promise)。superagent 本身不支持 Promise，从上面的代码来看就是使用回调的方法，这个库就是将 superagent 和 bluebird 融合在一起的“融合卡”。使用的时候只需要：
```
var Promise = require( 'bluebird' );
var request = require( 'superagent-bluebird-promise' );
request 
.get( 'http://xxxx.com' ) 
.then( function ( res ) { 
// do something when resolved 
}, function ( err ) {
 // do something when rejected 
});
```
立刻就可以使用上 then了，方便吧。


#三、打包
开发搞定后最后就是打包啦！！这里给大家介绍的是在 mac os 环境下用 electron-packager 打包的流程（原谅我刚入门 electron ，目前也只学会了这种方式）。mac 下打包 mas 程序当然是可行的，但是打包 win32 的程序的话，就需要花一点时间配置 Wine 环境了，后面会为大家介绍。
##3.1 mac os 环境下打包 mas 安装包
###3.1.1 在 package.json 中添加安装包依赖
```
"devDependencies": {
        "electron-prebuilt": "^1.3.5",
        "electron-packager": "latest"
    }
```

###3.1.2 命令行打包
在你项目工程的文件下，输入以下命令进行打包（为了好说明，我把空格换成换行符了）。
```
electron-packager 
./          //location of project
cyyDemo   //name of project
--platform=mas 
--arch=x64 
--version=1.3.5 //electron version
--out=dist/ 
--overwrite 
--ignore=node_modules/electron-* 
--ignore=node_modules/.bin 
--ignore=.git 
--ignore=dist --prune
```
- **location of project** 是你项目文件夹的位置，
- **name of project** 定义你的项目名，
- **platform** 决定要构建的平台（包括 Windows，Mac 和 Linux，all 表示所有平台 ）
- **architecture** 决定构建哪个构架下（ x86 或 x64 ，all表示两者），
- **electron version** 让你选择要用的 Electron 版本

命令的选项理解起来都比较简单。为了获得精美的图标，你首先要找一款可以把 png 文件转换到这些格式的工具，把它转换成 .icns 格式（ Mac 用）或者 .ico 格式（ Window 用）。如果在非 Windows 系统给 Windows 平台的应用打包，你需要安装 wine（ Mac 用户用 homebrew，Linux 用户用 apt-get ）。另外，第一次打包用时比较久，因为要下载平台的二进制文件，随后的打包将会快的多。
##3.2 mac os 环境下打包 win32 安装包
这个问题比较麻烦的要安装 Wine 环境。假设你设备上拥有 Wine 环境，打包流程可参考打包 mas 安装包的流程，下面主要就介绍下 mac 中如何安装配置 Wine 环境。
###3.2.1 安装 homebrew
brew 的安装很简单，使用一条 ruby 命令即可，Mac 系统上已经默认安装了 ruby。
```
/usr/bin/ruby -e "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install)"
```
安装完成后，就可以使用 brew 命令了，可以输入命令自检：
```
brew doctor
```
也可以参看安装版本：
```
brew -v
```
###3.2.1 使用 brew 安装 Wine
- 打开终端，输入以下命令行：
```
brew install wine
```
出现以下错误，提示我们安装缺失组件，下一步就是安装缺失组件了。
![提示安装缺失组件](http://upload-images.jianshu.io/upload_images/1896270-8681bc07d92513d3.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


- 安装缺失组件 Xquartz
  浏览器打开https://xquartz.macosforge.org/landing 下载并安装即可

- 重试安装 Wine
```
brew install wine
```
接下来的事就是等待了，我等了一个上午吧~~ that's a really long time

- 使用命令行打包 Win32 安装包
```
electron-packager ./ cyyDemo --platform=win32 --arch=x64 —version=1.3.5 --out=dist/ --overwrite --ignore=node_modules/electron-* --ignore=node_modules/.bin --ignore=.git --ignore=dist --prune
```