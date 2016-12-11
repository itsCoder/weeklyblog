# 在 Android 中使用 UIAutomator 执行自动化任务

用代码来替代任何重复性的工作，一直是我的追求。这周，我又写了一段脚本，让我从无尽的重复工作中解脱了出来。

# 问题 & 需求

最近在修 bug 的时候，为了准备一个合适的测试环境，我需要一直要重复执行这些操作：

- 完全关闭 App
- 打开应用设置页清空 App 数据
- 在权限界面打开「存储空间」权限
- 启动 App，登录帐号

主要过程就是不断地点击屏幕，等待界面切换，输入内容。我就想让代码来帮忙点击这些固定的位置，然后输入预设的内容，自动化完成这些无聊的操作。

整理一下，需要实现的功能大概就是：

- 根据文本信息确定点击位置
- 执行点击操作
- 执行输入操作
- 启动一些界面

# 解决方案

要用命令来控制 Android 设备，那肯定是选用 adb 了。GitHub 上有个叫 [awesome-adb](https://github.com/mzlogin/awesome-adb) 的项目，列举了 `adb` 的各种用法，其中有提到 [调起 Activity](https://github.com/mzlogin/awesome-adb#%E8%B0%83%E8%B5%B7-activity) 和 [模拟按键输入](https://github.com/mzlogin/awesome-adb#%E6%A8%A1%E6%8B%9F%E6%8C%89%E9%94%AE%E8%BE%93%E5%85%A5) 的操作。

另外查阅[资料](https://testerhome.com/topics/1047)得知了一个命令—— `uiautomator`。

```
$ adb shell uiautomator -h

Usage: uiautomator <subcommand> [options]

Available subcommands:

    help: displays help message

    runtest: executes UI automation tests

    dump: creates an XML dump of current UI hierarchy

    events: prints out accessibility events until terminated
```

可以看到它可以通过子命令 `runtest` 进行 UI 自动化测试，还可以通过子命令 `dump` 将当前屏幕的 UI 层级信息输出到 XML 文件中去。后者是这里需要关注的功能，将屏幕信息输出到 XML 中之后，可以根据关键字等去提取到具体的控件节点，从而获取到它在屏幕上显示的位置，用于模拟点击。下面用 Python 来实现整个流程。

这里先提一下两个工具函数，方便后续的代码展示。一个是用来执行 `adb`的，另外一个是装饰器，在目标函数执行完之后休眠一会，等待 UI 的响应。

```python
def run(cmd):
    """执行 adb 命令"""
    # adb <CMD>
    return subprocess.check_output(('adb %s' % cmd).split(' '))


def sleep_later(duration=0.5):
    """装饰器：在函数执行完成之后休眠等待一段时间"""
    def wrapper(func):
        def do(*args, **kwargs):
            func(*args, **kwargs)
            if 'duration' in kwargs.keys():
                time.sleep(kwargs['duration'])
            else:
                time.sleep(duration)

        return do
    return wrapper
```

## 根据文本信息点击屏幕

需要先用 `uiautomator` 命令来获取屏幕信息。

```python
dump_file = '/sdcard/window_dump.xml'

def dump_layout():
    print 'Dump window layouts'
    # adb shell uiautomator dump <FILE>
    run('shell uiautomator dump %s' % dump_file)
```

得到的 XML 文件是由 `node` 节点组成的，其中的 `text` 和 `bounds` 属性是我们需要的。可以根据文本去匹配到相应的 `node` 节点，然后解析出控件的边界信息，后续只要在这个边界内点击就可以模拟真实的操作了。

```xml
<?xml version='1.0' encoding='UTF-8' standalone='yes' ?>
<hierarchy rotation="0">
    <node
        index="0"
        text=""
        resource-id=""
        class="android.widget.FrameLayout"
        package="com.teslacoilsw.launcher"
        content-desc=""
        checkable="false"
        checked="false"
        clickable="false"
        enabled="true"
        focusable="false"
        focused="false"
        scrollable="false"
        long-clickable="false"
        password="false"
        selected="false"
        bounds="[0,0][1080,1920]">

        <!-- many nodes -->

    </node>
</hierarchy>
```

这里先用了 `cat` 命令直接读出 XML 的内容，然后用 `lxml` 解析匹配目标节点。后面用正则表达式提取出边界的坐标点，然后直接计算出边界矩形的中心点。

```python
def parse_bounds(text):
    # adb shell cat /sdcard/window_dump.xml
    dumps = run('shell cat %s' % dump_file)
    nodes = etree.XML(dumps)
    return nodes.xpath(u'//node[@text="%s"]/@bounds' % (text))[0]

bounds_pattern = re.compile(r'\[(\d+),(\d+)\]\[(\d+),(\d+)\]')

def point_in_bounds(bounds):
    """
    '[42,1023][126,1080]'
    """
    points = bounds_pattern.match(bounds).groups()
    points = map(int, points)
    return (points[0] + points[2]) / 2, (points[1] + points[3]) / 2
```

再用 `input` 命令，结合上面的几个函数，可以完成这个需求了。

```python
@sleep_later()
def click_with_keyword(keyword, dump=True, **kwargs):
    # 有的屏幕需要多次点击时，dump 可以设置为 False，使用上一次的屏幕数据
    if dump:
        dump_layout()
    bounds = parse_bounds(keyword)
    point = point_in_bounds(bounds)

    print 'Click "%s" (%d, %d)' % (keyword, point[0], point[1])
    # adb shell input tap <x> <y>
    run('shell input tap %d %d' % point)
```

## 模拟输入

这个比较简单，直接使用 `input text` 命令。另外还实现了模拟按返回键。

```python
@sleep_later()
def keyboard_input(text):
    # adb shell input text <string>
    run('shell input text %s' % text)


@sleep_later()
def keyboard_back():
    # adb shell input keyevent 4
    run('shell input keyevent 4')
```

## 停止应用、清除数据、启动 Activity

这一些命令操作，按照 [awesome-adb](https://github.com/mzlogin/awesome-adb) 的文档执行就好。

```python
@sleep_later()
def force_stop(package):
    print 'Force stop %s' % package
    # adb shell am force-stop <package>
    run('shell am force-stop %s' % package)


@sleep_later(0.5)
def start_activity(activity):
    print 'Start activity %s' % activity
    # adb shell am start -n <activity>
    run('shell am start -n %s' % activity)


@sleep_later(0.5)
def clear_data(package):
    print 'Clear app data: %s' % package
    # adb shell pm clear <package>
    run('shell pm clear %s' % package)
```

另外，打开指定应用的设置界面，需要指定 ACTION 和 DATA。

```python
@sleep_later()
def open_app_detail(package):
    print 'Open application detail setting: %s' % package
    # adb shell am start -a ACTION -d DATA
    intent_action = 'android.settings.APPLICATION_DETAILS_SETTINGS'
    intent_data = 'package:%s' % package

    run('shell am start -a %s -d %s' % (intent_action, intent_data))
```

## 拼装整个流程

```python
target_package = 'com.mingdao'
launcher_activity = 'com.mingdao/.presentation.ui.login.WelcomeActivity'

def main():
    username, password = sys.argv[1:3]
    # 停止应用
    force_stop(target_package)
    # 清除数据
    clear_data(target_package)
    # 启动应用设置页
    open_app_detail(target_package)
    # 进入权限页
    click_with_keyword(u'权限')
    # 打开「存储控件权限」
    click_with_keyword(u'存储空间')
    # 按一下返回
    keyboard_back()
    # 启动 app
    start_activity(launcher_activity)
    # 欢迎页跳过
    click_with_keyword(u'跳过')
    # 选中「帐号」输入框
    click_with_keyword(u'手机或邮箱', duration=0)
    # 输入帐号
    keyboard_input(username)
    # 选中「密码」输入框
    click_with_keyword(u'密码', dump=False, duration=0)
    # 输入密码
    keyboard_input(password)
    # 点一下登录按钮
    click_with_keyword(u'登录', dump=False)
```

前面把各种操作写好，主流程就很清晰啦，照着手动操作的过程，一步一步调函数就好了。

## 效果图

![screenshot](http://ww2.sinaimg.cn/large/65e4f1e6jw1faf8l8gwbng20ry0i2e7c.gif)

# 后记

脚本实现后的第二天，我就用了它不下 20 次，感觉爽极了。不用做那么多重复的操作，趁着空闲喝点水，刷个知乎，太美好了😄~

> 源码地址：[ui_automator.py · brucezz/SomeScripts](https://github.com/brucezz/SomeScripts/blob/master/ui_automator.py)

# Reference

- [awesome-adb](https://github.com/mzlogin/awesome-adb)
- [通过 python 调用 adb 命令实现用元素名称、id、class 定位元素 · TesterHome](https://testerhome.com/topics/1047)

