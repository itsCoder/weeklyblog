现在很多 APP 都有夜间模式，看到简书网站改版后也添加了夜间模式切换，所以就在自己的博客上实践。至于 Service Worker 是什么本文不再赘述，只是将实现过程记录下来，供各位看官参考。

### 如何实现夜间模式

其实，实现网页夜间模式的样式大致有两种方式：一种是直接在页面最上层添加一个半透明黑色遮罩，具体可参考『[使用 javascript 为网页增加夜间模式](http://www.jb51.net/article/46223.htm) 』另一种就是修改背景样式。网上有很多修改 CSS 样式太复杂，这里提供一个简单的示例：

```
<!DOCTYPE html>
<html lang="en">
<head >
    <meta charset="UTF-8">
    <title>night-mode</title>
</head>
<body style="background-color: rgb(255, 255, 255)">
    <p>switch button below!</p>
    <h1>Here</h1>
    <button onclick="darker()">Switch</button>
    <script type="text/javascript">
    function darker() {
        if (document.body.style.backgroundColor == 'rgb(255, 255, 255)') {

                document.body.style.backgroundColor = 'rgb(6, 23, 37)';
        }
        else {
                document.body.style.backgroundColor = 'rgb(255, 255, 255)';
        }
    }
    </script>
    
</body>
</html>
```
将上面的代码拷贝下来，保存到 HTML 文件中，看看效果。


### 客户端存储数据

仅仅按上面那样做是不够的，因为当你刷新标签页后，一切效果皆都消失，所以客户端(在这里也就是浏览器)还需要存储转换状态。

HTML5 提供了两种在客户端存储数据的方法：
localStorage —— 没有时间限制的数据存储，需手动清除
sessionStorage—— 针对 session 的数据存储，标签页关闭或者浏览器关闭后自动清除  
(以上两种方法最大能存储 5MB 数据)
其他具体相关细节可查阅 [MDN](https://developer.mozilla.org/)。

然后就只是修改上面的 Javascript 代码：
```
document.body.style.backgroundColor = sessionStorage.getItem('bg');
document.body.style.color = sessionStorage.getItem('md');

function darker() {
         if ( sessionStorage.getItem('bg') === 'rgb(255, 255, 255)') {
		sessionStorage.setItem('bg', 'rgb(6, 23, 37)');
        sessionStorage.setItem('md', 'night');
     }
    	else if (sessionStorage.getItem('bg') == null || undefined) {
        sessionStorage.setItem('bg', 'rgb(6, 23, 37)');
        sessionStorage.setItem('md', 'night');  
    }
    	else if( sessionStorage.getItem('bg') === 'rgb(6, 23, 37)') {
        sessionStorage.setItem('bg', 'rgb(255, 255, 255)');
        sessionStorage.setItem('md', 'day');
    }

document.body.style.backgroundColor = sessionStorage.getItem('bg');
document.body.style.color = sessionStorage.getItem('md');

}
```



另外还可以更精细地修改自己博客的 CSS 样式。[demo](https://zetaoyang.github.io/) 就是我的独立博客，奥秘就在左边的那个刷子。



### Service Worker 加速网页

首先，引入 `serviceworker.js`

```
if ('serviceWorker' in navigator) {
  navigator.serviceWorker.register('/serviceworker.js')
    .then(function(registration) {
    //Register succeed.
    console.log('[Service Worker] Registered，rage is', registration.scope);
    }).catch(function(err) {
    //Register failed.
      console.log('[Service Worker] Register failed:', err);
    });
}
```

`serviceworker.js ` 文件的内容(我的博客直接将 `serviceworker.js` 放得到了根目录，作用全局)，链接在此： https://zetaoyang.github.io/serviceworker.js 。

![serviceworker](https://cdn.rawgit.com/qanno/qanno.github.io/master/images/blog-serviceworker.png)  
注：serviceworker 生产环境只能是`https`,本地测试环境可以是`http://localhost`。

### CDN 加速网页

利用 CDN 服务 来加速你的网络。
#### 国内：

[BootCDN](http://www.bootcdn.cn/)
[又拍云 JS 库加速服务](http://jscdn.upai.com/)
[开放静态文件 CDN](https://staticfile.org/)
[百度静态资源公共库](http://cdn.code.baidu.com/)

#### 国外：

[Google Hosted Libraries](https://developers.google.com/speed/libraries/)
[cdnjs](https://cdnjs.com/) 
[jsDelivr](https://www.jsdelivr.com/) (有[国内节点](https://www.jsdelivr.com/features/cdn-in-asia-and-china))

注：
万一 CDN 服务挂了，还可以从自己的服务器上读取：
```
<script src="//cdn.bootcss.com/jquery/3.1.1/jquery.min.js"></script>
<script>window.jQuery || document.write('<script src="/jquery.min.js"><\/script>')</script>
```
可参考 [Stack Overflow](http://stackoverflow.com/questions/5257923/how-to-load-local-script-files-as-fallback-in-cases-where-cdn-are-blocked-unavai) 。

### 相关资料

[【翻译】Service Worker 入门](https://w3ctech.com/topic/866)  
[Service Workers 初体验](https://www.w3ctrain.com/2016/09/17/service-workers-note/)  
谷歌搜索的 serviceworker 文件：https://www.google.com/_/chrome/newtab-serviceworker.js  
[HTML5-Service Worker 实现离线页面访问](http://blog.csdn.net/qiqingjin/article/details/51629278)