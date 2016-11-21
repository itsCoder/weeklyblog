# Atom-Helper 小脚本

这几天在尝试用新的文本编辑器 Atom，上手第一件事就是装插件咯。Atom 和 Sublime 一样，社区提供了各种各样的插件，来提供酷炫的功能。 我在 Atom 网站上搜索很多插件然后看看介绍下载体验，配置了一个简单的 Python 开发环境。

这个时候就想写个小脚本玩玩了，大概功能就是根据关键字搜索 Atom 的插件或者主题，然后排序展示前 N 个项目，当然直接通过命令行改变参数就更好了。

Atom 插件地址：[Packages](https://atom.io/packages)

Atom 主题地址：[Themes](https://atom.io/themes)

# 分析问题

整个脚本流程：

1. 根据关键字发出第一个搜索请求
2. 解析出页面中的插件数据
3. 找到「下一页」的链接，请求下一页
4. 循环 2-3，直到最后一页，即「下一页」的链接不存在
5. 对获取到的插件进行排序，然后输出 topN

先手动搜索一个关键字，用 Chrome 开发者工具查看一下请求内容：

![Search](http://ww4.sinaimg.cn/large/006y8mN6gw1f9v6ufxhuij30m80csdgm.jpg)

这就是一个普通的 GET 请求，直接把关键字作为参数添加在 url 后面就好了。

接下来查看一下页面的 HTML 元素：

![HTML](http://ww3.sinaimg.cn/large/006y8mN6gw1f9vct78948j30go0f8gmt.jpg)

可以根据 `class="grid-cell"` 这一属性，把每一个插件作为整体提取出来，然后再详细地解析出具体的字段。

然后找一下 「下一页」的链接：

![Next](http://ww3.sinaimg.cn/large/006y8mN6gw1f9vcuu27iuj30m801s74c.jpg)

![Next empty](http://ww3.sinaimg.cn/large/006y8mN6gw1f9vcvvflcaj30m801kdfs.jpg)

这也验证了前面的想法，最后一页的时候，「下一页」的链接不存在，这样就能正确地结束循环请求了。

最后就是获取一下 topN 的数据，然后输出出来了。

# 获取数据

首先根据前面分析的结果，先写一个大概的类结构，有几个方法需要实现。

```python
class AtomHelper(object):
    def __init__(self, args):
        pass

    def extract_item(self, cell):
        """Extract item from cells."""
        pass

    def run(self):
        """Do search!"""
        pass

    def topN(self, N=10):
        """Get topN items."""
        pass
```

## 解析字段

这个步骤直接使用 lxml 的 xpath 进行解析就好了，熟悉 xpath 的话很简单，最后把字段组装成一个字典。

```python
def extract_item(self, cell):
    """Extract item from cells."""

    # Extract title, url, desc, download ...
    # ...

    star = self.first(cell.xpath('.//a[@class="social-count"]/text()'))

    return {
        'title': title,
        'url': url,
        'desc': desc,
        'download': self.number_format(download),
        'star': self.number_format(star),
    }
```

需要注意的是，`xpath` 方法返回的通常都是 `list` 对象，通常都只需要第一项。所以这里写了一个方法来从序列中提取第一个元素，提供默认值避免每次判空。

```python
def first(seq, default=''):
    """Extract first item in sequence, or default value."""
    return seq[0] if seq else default
```

另外网站上的数字是格式化后的，即 `2,333` 的形式，需要转换成正常的数字。

```python
def number_format(num_str):
    """Convert number string like '2,333' to integer."""
    return int(num_str.replace(',', '').strip()) if num_str else num_str
```

## 循环请求页面

这一部分就是一个循环，不断地请求「下一页」的地址，同时解析页面中的数据和下一个链接。

```python
def run(self):
    """Do search!"""

    next_url = self.search_url
    while next_url:
        html = etree.HTML(requests.get(next_url).text)

        # Extract cells, next_url
        # ...

        self.items.extend(map(self.extract_item, cells))
```

## 获取 topN

这个用一下内置函数 `sorted` 就好了。

```python
def topN(self, N=10, by='download'):
    return sorted(self.items, key=lambda item: item[by], reverse=True)[:N]
```

然后把具体实现的代码组装起来，添加一些必要的输出提示，主体功能就完成了。

# 添加命令行参数

脚本可以搜索「插件」或者「主题」，这个需要枚举参数来控制；输出多少个结果，也是需要一个参数；是按下载量排序还是星星数量排序，还需要一个枚举参数。

> `argparse` 模块使得编写用户友好的命令行接口非常容易。程序只需定义好它要求的参数，然后 `argparse` 将负责如何从 `sys.argv` 中解析出这些参数。`argparse` 模块还会自动生成帮助和使用信息并且当用户赋给程序非法的参数时产生错误信息。

`argparse` 这个库简单易用，文档写得很详细，还有各种示例。[戳这里](http://python.usyiyi.cn/python_278/library/argparse.html)

```python
def process_arg():
    parser = argparse.ArgumentParser(
        description='Atom helper to search packages or themes.',
        epilog='Have fun :)')

    parser.add_argument('keyword',
                        nargs='+', help='The keyword to search.')

    parser.add_argument('-n', type=int, default=5,
                        help='Show top N items.(default 5)')
    parser.add_argument('-t', '--type', choices=('packages', 'themes'),
                        default='packages',
                        help='The type to search.(default packages)')
    parser.add_argument('-s', '--sort', choices=('download', 'star'),
                        default='download',
                        help='Sort items by.(default download)')
    parser.add_argument('-v', '--verbose', action='store_true',
                        help='Show debug messages.')

    args = vars(parser.parse_args())
    args.update(keyword='+'.join(args['keyword']))
    return args
```

这里主要的几个参数：

![args](http://ww1.sinaimg.cn/large/006y8mN6gw1f9vcnvzmjrj30go09z3zg.jpg)

- `-t` 指定搜索类型
- `-s` 指定排序规则
- `-n` 指定输出多少个数据
- `-v` 显示 debug 信息
- `-g` 显示帮助信息

# 最终结果

![final](http://ww1.sinaimg.cn/large/006y8mN6gw1f9vcmj4wt9j30m80bv0tv.jpg)

花时间写完这个脚本，发现并没有什么卵用😂，还不如自己去网站上搜搜呢。

不过第一次写命令行参数，感觉还是挺好玩的，熟悉了下 `argparse` 模块，以后用 Python 写小工具更方便了。

完整脚本地址: [atom_helper.py](https://github.com/brucezz/SomeScripts/blob/master/atom_helper.py), 欢迎拍砖评论。

# Reference

- [argparse 官网文档](http://python.usyiyi.cn/python_278/library/argparse.html)
- [argparse - 命令行选项与参数解析（译）](http://blog.xiayf.cn/2013/03/30/argparse/)
