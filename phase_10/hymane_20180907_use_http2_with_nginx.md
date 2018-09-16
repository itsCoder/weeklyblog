---
title: CentOS 搭建 Nginx 服务器并开启 HTTP2 协议
date: 2018-09-07 15:10:19
categories: web
tags: [http2, nginx, linux]
---
# 
### 准备工作
> * 要开启 HTTP/2 需要 nginx 版本在 1.10.0 以上且需要 openssl 版本在1.0.2以上编译。
> * HTTP2.0 支持开启了 HTTPS 的网站（h2协议本身是支持 HTTP 的，但是目前主流浏览器只支持基于 TLS 部署的HTTP2.0协议）

### 现在开始
1. 添加 Nginx 的 YUM 源

    ```
    sudo rpm -Uvh http://nginx.org/packages/centos/7/noarch/RPMS/nginx-release-centos-7-0.el7.ngx.noarch.rpm
    ```
    保证使用 YUN 安装的 nginx 版本为新的版本

2. 安装 Nginx
    使用 yum 获取 nginx 软件包，进行安装

    ```
    sudo yum install -y nginx
    ```

3. 启动 Nginx 

    至此已经安装完毕，来试试是否成功
    
    ```
    sudo service nginx start
    ```
    如果浏览器访问看到以下结果表示安装成功了
    
    ![nginx](https://wx4.sinaimg.cn/mw690/005X6W83gy1fucrtw4m7dj30vm0bs76t.jpg)

4. 配置 nginx
    复制个人配置文件
    
    ```
    # cd /etc/nginx/conf.d/
    # cp default.conf myName.conf
    ```

5. 修改 myName.conf 如下

    ```
    server {
         listen 80;
         server_name hymane.com www.hymane.com;
         access_log  /home/hymane/www/logs/www.hymane.com.log;
         return 301 HTTPs://www.hymane.com$request_uri;
         #所有 HTTP 请求转给 HTTPs 来处理
    }
    
    server {
        listen 443 ssl http2;
        server_name www.hymane.com hymane.com;
        access_log  /home/hymane/www/logs/www.hymane.com.log;
        # root /home/wwwroot;
        ssl on;
        ssl_certificate /etc/nginx/certs/www.hymane.com/ssl.crt;
        ssl_certificate_key /etc/nginx/certs/www.hymane.com/ssl.key;
    
        location / {
            proxy_pass HTTP://localhost:9000;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_redirect off;
            #root   /usr/share/nginx/html;
            #index  index.html index.htm;
        }
    }
    ```
    因为 HTTP/2 需要 https, 所以这里需要配置 ssl 证书，设置在 443 端口，在 `listen` 后添加 `http2` 即可。是不是很简单。
    注意以下几行
    
    ```
    listen 443 ssl http2;#监听 443 端口并开启 h2 协议
    ssl on;
    ssl_certificate /etc/nginx/certs/www.hymane.com/ssl.crt;
    ssl_certificate_key /etc/nginx/certs/www.hymane.com/ssl.key;
    ```
6. 重启 nginx 服务器
  
    ```
    # service nginx stop
    # service nginx start
    ```
    验证 nginx 配置项是否有误
    
    ```
    # nginx -t
    nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
    nginx: configuration file /etc/nginx/nginx.conf test is successful
    ```
    
    以上表示配置没问题，如果有问题针对修改就可以，每次修改配置，重新生效需要`reload` nginx 一下.
    
    ```
    # nginx -s reload
    ```
    查看 openssl 版本
    
    ```
    openssl version
    OpenSSL 1.0.2k-fips  26 Jan 2017
    ```
    > 1.0.2k版本 ok 哦，😊
    
    **注意：**
    > * 配置完 h2 协议之后必须重启 nginx 服务，仅仅 `reload` 配置是不生效的。
    > * openssl 版本过低会导致无法开启 h2，请升级版本后再试。

### 验证 h2 是否生效

* 方法1：谷歌浏览器安装一个插件 `HTTP/2 and SPDY indicator`， 然后浏览待测试网址，有蓝色小闪电⚡️图标表示 h2 生效，灰色表示未开启 h2，

![h2](https://wx3.sinaimg.cn/mw690/005X6W83gy1fucsjlfo9zj309202iglk.jpg)
* 方法2：打开开发者工具，打开 network,刷新一下页面，打开 protocol 选项卡查看请求是否是 h2 协议访问。
![protocol](https://wx2.sinaimg.cn/mw690/005X6W83gy1fucskylw55j314q0l6wjb.jpg)

### 总结
HTTP/2.0 是 HTTP 协议自 1999 年 HTTP/1.1 发布后的首个更新，HTTP/1.1 目前是主流的 HTTP 协议版本，虽然在 2015年5月就已经正式发表，但到目前支持的站点还是较少。

HTTP 2.0 相比 HTTP 1.x 大幅度的提升了 web 性能。在完全兼容 1.x 版本外，进一步减少了网络延迟。h2 速度更快，延迟更小了。

![](http://wx2.sinaimg.cn/mw690/005X6W83gy1fv3avml6zsj30k00ce7ah.jpg)
> 图片来源 [知乎回答](https://www.zhihu.com/question/34074946)

[点这里直观对比](https://imagekit.io/demo/http2-vs-http1)


