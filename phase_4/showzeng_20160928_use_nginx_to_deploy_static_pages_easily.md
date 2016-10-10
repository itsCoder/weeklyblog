---
title: 使用 Nginx 部署静态页面
thumbnailImagePosition: left
metaAlignment: left
coverMeta: in
coverSize: full
date: 2016-09-28 23:13:41
tags:
- Nginx
category: Nginx
thumbnailImage:
coverImage:
coverCaption:
gallery:
---

你知道怎么在自己的服务器上部署一些简单的静态页面吗？使用 Nginx 可以让你爽到飞起！

<!-- more -->

#### **Nginx 介绍**

Nginx 是俄罗斯人编写的十分轻量级的 HTTP 服务器, Nginx，它的发音为 “ engine X ”，是一个高性能的 HTTP 和反向代理服务器，同时也是一个 IMAP/POP3/SMTP 代理服务器。Nginx 是由俄罗斯人 Igor Sysoev 为俄罗斯访问量第二的 Rambler.ru 站点开发的，它已经在该站点运行超过两年半了。Igor Sysoev 在建立的项目时,使用基于 BSD 许可。

英文主页：[http://nginx.net](http://nginx.net)。

Nginx 作为 HTTP 服务器，有以下几项基本特性：

- 处理静态文件，索引文件以及自动索引；打开文件描述符缓冲。
- 无缓存的反向代理加速，简单的负载均衡和容错。
-  FastCGI，简单的负载均衡和容错。
- 模块化的结构。包括 gzipping, byte ranges, chunked responses,以及 SSI-filter 等 filter。如果由 Fast CGI 或其它代理服务器处理单页中存在的多个 SSI，则这项处理可以并行运行，而不需要相互等待。
- 支持 SSL 和 TLSSNI。

即 Nginx 的优点：轻量、高性能、并发能力强。用来部署静态页面也是相当便捷。

这种高性能得益于 Nginx 的框架。在 Nginx 启动后，会有一个 master 进程和多个 worker 进程。master 进程主要用来管理 worker 进程，包含：接收来自外界的信号，向各 worker 进程发送信号，监控 worker 进程的运行状态，当 worker 进程退出后(异常情况下)，会自动重新启动新的 worker 进程。而基本的网络事件，则是放在 worker 进程中来处理
的。多个 worker 进程之间是对等的，他们同等竞争来自客户端的请求，各进程互相之间是独立的。一个请
求，只可能在一个 worker 进程中处理，一个 worker 进程，不可能处理其它进程的请求。worker 进程的个数是
可以设置的，一般我们会设置与机器 cpu 核数一致，这与 Nginx 的进程模型以及事件处理模型有关。

#### **为何选用 Nginx**

说到 Nginx，可能第一反应就是 `反向代理` 和 `负载均衡` 了。那么什么是 `反向代理`，什么又是 `负载均衡` 呢？

**反向代理**

首先了解一下什么是 `前向代理` 。代理 (Proxy) 也称网络代理，是一种特殊的网络服务，通俗来讲，就是在客户端和目标服务器之间的充当中间人，接收客户端的请求，再根据客户端请求向目标服务器发起相应的请求，从目标服务器获得指定资源后返回给客户端。且代理服务器可以对目标服务器的资源下载至本地缓存，如果客户端所要获取的资源在代理服务器的缓存之中，则代理服务器并不会再向目标服务器发起请求，而是直接返回缓存的资源。

其实代理服务器非常常见，就比如因为 `GWF` 而存在的一些科学上网的代理商，就是利用国外服务器作为代理服务器，从而正确解析域名来实现科学上网。代理服务器也可以实现隐藏真实 IP，比如著名的 Tor （洋葱路由器）就是通过多重代理和一些加密技术来实现匿名交流。

而 `反向代理` 则是在服务器端作为代理使用，而不是客户端。也就是说，`前向代理` 是代理内部网络用户访问 Internet 上服务器的连接请求，`反向代理` 是以代理服务器来接受 Internet 上的连接请求，然后将请求转发给内部网络上的服务器，并将从服务器上得到的结果返回给 Internet 上请求连接的客户端，此时的代理服务器对外就表现为一个服务器。

**负载均衡**

反向代理负载均衡技术是把将来自 Internet 上的连接请求以反向代理的方式动态地转发给内部网络上的多台服务器进行处理，从而达到负载均衡的目的。

**真巧， Nginx 都做到了**

Nginx 作为一个优秀的代理服务器，反向代理和负载均衡必然是都具备的。想要更详细得了解这方面的知识和使用方法，可参见文末给出的参考资料： Nginx 入门指南。

#### **Nginx 安装**

本人使用的是腾讯云的服务器，版本为：  Ubuntu Server 14.04.1 LTS 32 位。

``` bash
$ apt-get install nginx
```

Mac OS 系统参考这篇文章：[Installing Nginx in Mac OS X](http://learnaholic.me/2012/10/10/installing-nginx-in-mac-os-x-mountain-lion/)


#### **Nginx 配置**

简单地配置 Nginx 的配置文件，以便在启动 Nginx 时去启用这些配置。而本文的重点也是于此。

Nginx 的配置系统由一个主配置文件和其他一些辅助的配置文件构成。这些配置文件均是纯文本文件，一般地，我们只需要配置主配置文件就行了。例如在我的服务器上是在：`/etc/nginx/nginx.conf` 。

**指令上下文**

nginx.conf 中的配置信息，根据其逻辑上的意义，对它们进行了分类，也就是分成了多个作用域，或者称之为配置指令上下文。不同的作用域含有一个或者多个配置项。

其中每个配置项由配置指令和指令参数构成，形成一个键值对，`#` 后面地为注释，理解起来也非常容易。

一般配置文件的结构和通用配置如下：

``` nginx
user www-data;    # 运行 nginx 的所属组和所有者
worker_processes 1;    # 开启一个 nginx 工作进程,一般 CPU 几核就写几
pid /run/nginx.pid;    # pid 路径

events {
        worker_connections 768;    # 一个进程能同时处理 768 个请求
        # multi_accept on;
}

# 与提供 http 服务相关的配置参数，一般默认配置就可以，主要配置在于 http 上下文里的 server 上下文
http {
        ##
        # Basic Settings
        ##

        ... 此处省略通用默认配置

        ##
        # Logging Settings
        ##
        ... 此处省略通用默认配置

        ##
        # Gzip Settings
        ##

        ... 此处省略通用默认配置

        ##
        # nginx-naxsi config
        ##

        ... 此处省略通用默认配置

        ##
        # nginx-passenger config
        ##

        ... 此处省略通用默认配置

        ##
        # Virtual Host Configs
        ##

        ... 此处省略通用默认配置

        # 此时，在此添加 server 上下文，开始配置一个域名，一个 server 配置段一般对应一个域名
        server {
                listen 80;               # 监听本机所有 ip 上的 80 端口
                server_name _;           # 域名：www.example.com 这里 "_" 代表获取匹配所有
                root /home/filename/;    # 站点根目录

                location / {             # 可有多个 location 用于配置路由地址
                        try_files index.html =404;
                }
}

# 邮箱的配置，因为用不到，所以把这个 mail 上下文给注释掉
#mail {
#       # See sample authentication script at:
#       # http://wiki.nginx.org/ImapAuthenticateWithApachePhpScript
#       
#       # auth_http localhost/auth.php;
#       # pop3_capabilities "TOP" "USER";
#       # imap_capabilities "IMAP4rev1" "UIDPLUS";
#     
#       server {
#               listen      localhost:110;
#               protocol    pop3;
#               proxy       on;
#       }
#
#       server {
#               listen      localhost:143;
#               protocol    imap;
#               proxy       on;
#       }
#}
```

这里需要注意的是 http 上下文里的 server 上下文。

``` nginx
server {
        listen 80;               # 监听本机所有 ip 上的 80 端口
        server_name _;           # 域名：www.example.com 这里 "_" 代表获取匹配所有
        root /home/filename/;    # 站点根目录

        location / {             # 可有多个 location 用于配置路由地址
            try_files index.html =404;
        }
}
```

这里的 root 字段最好写在 location 字段的外边，防止出现无法加载 css、js 的情况。因为 css、js 的加载并不是自动的，nginx 无法执行，需要额外的配置来返回资源，所以，对于静态页面的部署，这样做是最为方便的。

这里对 root 作进一步解释，例如在服务器上有 /home/zhihu/ 目录，其下有 index.html 文件和 css/ 以及 img/ ， `root /home/zhihu/;` 就将指定服务器加载资源时是在 /home/zhihu/ 下查找。

其次， location 后的匹配分多种，其各类匹配方式优先级也各不相同。这里列举一精确匹配例子：

``` nginx
server {
        listen 80;               
        server_name _;           
        root /home/zhihu/;    

        location = /zhihu {
            rewrite ^/.* / break;
            try_files index.html =404;
        }
}
```

此时，访问 www.example.com/zhihu 就会加载 zhihu.html 出来了。由于 location 的精确匹配，只有访问  www.example.com/zhihu 这个路由时才会正确响应，而且此时要通过 rewrite 正则匹配，把 /zhihu 解析替换成原来的 / 。关于更多 location 字段用法，可以在文章最后给出的参考资料中查看。

**关于使用 nginx 部署静态页面，最简单便捷的配置方法**

上面说了挺多关于配置的说明，下面推荐一种个人认为最为便捷的配置方法。（特此感谢 [guyskk](https://github.com/guyskk) 学长的答疑解惑）

首先创建一个目录，例如： `/home/ubuntu/website` 然后在这个 website 文件夹下可以放置你需要部署的静态页面文件，例如 website 下我有 google、zhihu、fenghuang 这三个文件夹，其中 server 字段配置如下：

``` nginx
server {
        listen 80;
        server_name _;
        root /home/ubuntu/website;
        index index.html;
}
```

这里每个文件夹下面的静态页面文件名都是 index.html ，我以前有个很不好的习惯，比如 zhihu 页面就喜欢命名为 zhihu.html ，但就从前端来看，这也是不符合规范的。

这样配置的话，例如当你访问 [www.showzeng.cn/google/](http://www.showzeng.cn/google/) 时，nginx 就会去 website 目录下的 google 文件夹下寻找到 index.html 并把 google 页面返回，同理，访问 [www.showzeng.cn/zhihu/](http://www.showzeng.cn/zhihu/) 时，会寻找到 zhihu 文件夹下的 index.html 并把 zhihu 页面返回。

而在 zhihu、google 、fenghuang 文件夹的同级目录上，再添加你的域名首页 index.html 时，访问 www.example.com 时就会返回了。

这里唯一美中不足的是，访问域名中 www.showzeng.cn/zhihu 末尾会自动加上 `/` ，在浏览器中按 F12 调试会发现 www.showzeng.cn/zhihu 为 301 状态码，因为 index.html 是在 zhihu/ 文件夹下，所以在搜索过程中会重定向到 www.showzeng.cn/zhihu/ ，起初我是接受不了的，那一 `/` 看起来太难受了，但是只要一想到要一个一个 location 字段去匹配，我一下子就接受了。不知道你怎么看，反正我是接受了。

#### **Nginx 启动运行**

``` bash
$ sudo nginx -s reload
```

使用 reload 方法不用重启服务，直接重新加载配置文件，客户端感觉不到服务异常，实现平滑切换。当然你也可以重新启动 nginx 服务。

``` bash
$ sudo service nginx restart
```
#### **Nginx 停止运行**

``` bash
$ sudo nginx -s stop
```

#### **参考资料**

[Nginx 入门指南](http://wiki.jikexueyuan.com/project/nginx/)

[Nginx for Developers: An Introduction](http://carrot.is/coding/nginx_introduction) [（译文）](https://fraserxu.me/2013/06/22/Nginx-for-developers/)

[nginx配置location总结及rewrite规则写法](http://seanlook.com/2015/05/17/nginx-location-rewrite/)
