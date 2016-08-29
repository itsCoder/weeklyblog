---
title: 豆瓣图书 Top250 爬虫
toc: true
categories: python
description: crawler
feature:
---

为了给 One App 添加一个书籍推荐的页面。写了一个简单的豆瓣图书 Top250 的 Python 爬虫。本篇文章详细介绍一下爬取思路以及过程。

<!--more-->

## 一 .  爬取目标

打开豆瓣图书 Top250 首页:

![douban1.png](http://7xrl8j.com1.z0.glb.clouddn.com/douban1.png)

点击标题进去看看:

![douban2.png](http://7xrl8j.com1.z0.glb.clouddn.com/douban2.png)

如上截图所示即为要爬取的内容,如下列出:

1. 书籍内容简 url
2. 书籍封面 url
3. 书籍标题
4. 书籍作者
5. 书籍评分以及评价人数
6. 书籍简介内容

## 二 .  思路分析

#### 2.1 元素提取分析

首先右键鼠标查看 HTML 源码,HTML 源码截图贴出如下:

![douban3.png](http://7xrl8j.com1.z0.glb.clouddn.com/douban3.png)

如上截图。我们查看页面源码时候发现每个书籍 Item 所对应的源码 Item 都如上所示。所以我们提取元素时候只需要研究如上一个 Item 即可。其他剩下的可以循环获取。

如上截图所标注数字即对应我们要提取的元素。而对于**书籍简介内容**,我们在获取标注 1 之后仍然要去请求一下。请求之后获取的 HTML 源码截图如下:

![douban4.png](http://7xrl8j.com1.z0.glb.clouddn.com/douban4.png)

#### 2.2 页面请求分析

打开豆瓣图书 Top250 , 浏览器地址栏 url 如下截图所示:

![douban12.png](http://7xrl8j.com1.z0.glb.clouddn.com/douban12.png)

可以认为其为页面 BaseUrl。然后我们拉到页面最低端**点击下一页**看看地址栏有没有变化,截图如下所示:

![douban13.png](http://7xrl8j.com1.z0.glb.clouddn.com/douban13.png) 

发现多了一个 ？start = 25 。而恰好每个页面的 Item 都是 25 个。我们猜想是不是只需要变化 25 这个数字所在位置的参数即可？我们将 25 改成 50 。发现确实会调转到下一个页面。**验证猜想。**

![douban11.png](http://7xrl8j.com1.z0.glb.clouddn.com/douban11.png)

注: 如果当时我们点击下一页的时候地址栏没有发生参数变化,那么我们就要 F12 了,从后台看看请求过程发生了什么。

以上我们分析了请求过程。那么我们真正爬取的时候参数从 0 (*25)- 9(*25) 即可。

## 三 .  爬取源码分析

```python
# 提取主要内容
def getContent(self,url):
    reqContent = requests.get(url, headers=self.headers)
    if reqContent.status_code == 200:
        i = 0
        soup = BeautifulSoup(reqContent.content, 'html.parser')
        # 循环获取一个页面的所有 Item 然后进行内部元素提取
        for link in soup.find_all('tr', 'item'):
            # 构建云端存储对象
            bookInfo = BookSave()
            print 'num',i-1,' data save success'
            try:
                # 用 bs 来提取相应元素并存储至云端
                bookInfo.set('bookIntr',  self.getBookIntro(link.a['href'])).save()
                bookInfo.set('imgUrl', link.find('a', 'nbg').contents[1]['src']).save()
                bookInfo.set('bookTitle', link.find('div', 'pl2').a['title']).save()
                bookInfo.set('bookAuth', link.find('p', 'pl').string).save()
                bookInfo.set('bookRate', link.find('span', 'rating_nums').string).save()
                bookInfo.set('rateNum', link.find('span', 'pl').string[1:-2].strip()).save()
            except leancloud.LeanCloudError as e:
                print e.message
            i = i + 1

# 提取书籍介绍内容 
def getBookIntro(self, url):
    # 请求书籍介绍内容页面
    intrContent = requests.get(url, headers=self.headers)
    if intrContent.status_code == 200:
        # 用 re 来提取书籍介绍内容
        pattern4 = re.compile('<div class="">.*?<style .*?>.*?</style>.*?<div .*?>(.*?)</div>', re.S)
        BookIntro = re.findall(pattern4, intrContent.content)
        if(len(BookIntro) > 0):
            bookInfro = BookIntro[0].strip().decode('utf-8').replace('<p>',' ').replace('</p>',' ')
        else:
            print 'data error'
        if(len(bookInfro) > 0):
            return bookInfro
        else:
            print 'no data'
    else:
        print 'error'
            
# 启动函数            
def start(self):
    for page in range(0,9):
        if(page == 0):
            baseUrl = 'https://book.douban.com/top250'
            self.getContent(baseUrl)
            print 'page 1 ok'
        else:
            baseUrl = 'https://book.douban.com/top250?start='
            pages = page * 25
            url = baseUrl + str(pages)
            self.getContent(url)
            time.sleep(3)
            print 'page', page, 'ok'
```

如上即为整个爬取代码的核心部分。

1. Requests 库来进行网络请求。其会返回请求状态码。我们可以根据该状态码是否为 200 来判断是否请求成功。
2. BeautifulSoup 库和 re 来进行页面元素的提取。之所以在用到 bs 的情况下又用到 re ，是因为在提取书籍介绍内容的过程中有很多相同的 Item ,这时候用 bs 提取会获取的一些你不想获取的数据。所以用正则来层层向外扩张过滤。
3. 加上一个 time.sleep(3)  来防止被豆瓣识别出来。也是极其简单的策略。
4. 爬取的数据大多都是不规整的。比如空格,扩号,换行 等等。这时候我们就需要 python 中非常基础的字符串处理函数和正则**来进一步过滤数据了**。
5. 对于有一些你爬取的数据,十分有必要检查一下它的大小。因为极有可能在该数据为空的情况下,你对他进行一些字符串的操作处理。这样会导致一些不必要的异常发生,从而会导致爬虫终止。

## 四 . 个人总结

整个爬取过程还是很简单的。没有涉及到登录以及其他复杂的验证过程。主要提取出来的数据是为了 One App 服务。爬到数据就好了。比如速度和量都没有刻意追求。加上延时的话也才一分多钟就爬好了。另外文中也说了这里请求的 url 很简单。直接在地址栏告诉你规律。如果有的网站地址栏变化不告诉你,你就要去后台看看**在你拖动页面往下或者点击下一页**的时候 network 中有没有什么参数发生变化。一般都是 [XHR](http://baike.baidu.com/item/XMLHTTPRequest) 这个参数会发生变化。点进去看看就好了。

以上,**感谢阅读**。
