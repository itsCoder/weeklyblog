---
title: 菜鸟阅读 Flask 源码系列（1）：Flask的router初探
type: tags
date: 2016-08-09 15:54:20
tags: [Python,编程,Flask,源码阅读]
categories: [编程,Python]
---
>- 文章来源：itsCoder 的 [WeeklyBolg](https://github.com/itsCoder/weeklyblog) 项目
>- itsCoder主页：[http://itscoder.com/](http://itscoder.com/)
>- 作者：[写代码的香港记者](https://github.com/Zheaoli)
>- 审阅者：[Brucezz](https://github.com/brucezz)

## 前言

没有一个完整的开源项目的的阅读经验的程序猿是一个不合格的程序猿，虽然曾经阅读过部分诸如 **Redis** 等项目的源码，但是还没有过一个完整的开源项目的阅读经验，因此在经过某个前辈的不断安利后，我决定用 **Flask** 来作为阅读开源源码计划的开始。而这一个系列的文章，将作为我自己的阅读笔记，来巩固自己曾经所没有重视的 **Python** 的很多细节。

<!-- more -->

## 关于 **Flask**

关于 **Flask** 的背景知识，就不需要太多的描述了，网上已经有很多的资料了。在使用 **Flask** 的时候，我们经常用如下的方式来设置我们的自定义的路由：

~~~Python
##Flask官方Example中flaskr项目部分代码
app = Flask(__name__)

@app.route('/')
def show_entries():
    db = get_db()
    cur = db.execute('select title, text from entries order by id desc')
    entries = cur.fetchall()
    return render_template('show_entries.html', entries=entries)


@app.route('/add', methods=['POST'])
def add_entry():
    if not session.get('logged_in'):
        abort(401)
    db = get_db()
    db.execute('insert into entries (title, text) values (?, ?)',
                 [request.form['title'], request.form['text']])
    db.commit()
    flash('New entry was successfully posted')
    return redirect(url_for('show_entries'))


@app.route('/login', methods=['GET', 'POST'])
def login():
    error = None
    if request.method == 'POST':
        if request.form['username'] != app.config['USERNAME']:
            error = 'Invalid username'
        elif request.form['password'] != app.config['PASSWORD']:
            error = 'Invalid password'
        else:
            session['logged_in'] = True
            flash('You were logged in')
            return redirect(url_for('show_entries'))
    return render_template('login.html', error=error)

~~~
那么问题来了，上面的例子中，我们知道 `app.route('xxxx',methods=['xxx'])` 将会设置我们对应的方法与对应 url 的关联，那么这样一种做法是怎样生效的呢？

## **Flask** 源码阅读

### 让我们看看最开始的 router 是什么样子的
首先让我们从 [Flask](https://github.com/pallets/flask) 这里获取 flask 源码，然后我们将版本号切换至最初的 0.1 版（git tag为8605cc310d260c3b08160881b09da26c2cc95f8d）

> 小tips：阅读开源项目时，如果当前版本太过于复杂，可以切换至项目最初发布时的版本，然后根据每次项目版本发布的 Release Note 来进行跟进。

在 **flask.py** 文件里，我们能看到如下的的结构

![flask.py 文件目录](/images/flask1.png)

讲真这个时候我们就可以看到 **Flask** 里的 **route** 的核心代码了

~~~Python
def route(self, rule, **options):
    def decorator(f):
        self.add_url_rule(rule, f.__name__, **options)
        self.view_functions[f.__name__] = f
        return f
    return decorator

~~~
我们能很清楚的看到之前代码中的 `app.route('/')` 本质上是调用了一个装饰器来对我们对应的方法进行请求与方法之间进行关联。在 `route` 方法被触发后，进一步来调用 `add_url_rule` 来注册我们所设定的url。

~~~Python
def add_url_rule(self, rule, endpoint, **options):
    options['endpoint'] = endpoint
    options.setdefault('methods', ('GET',))
    self.url_map.add(Rule(rule, **options))
~~~

在最初版本的 **Flask** 中， **Router** 的实现就这么简单暴力

### 两个关于装饰器的知识点

第一个：很多人肯定想问，在之前的代码里，我们没有调用相关的方法，例如 `index()` ，那么装饰器为什么会被触发呢？

答：首先，请大声告诉我，装饰器的作用是什么？很明显嘛，在不修改原有代码的基础上，对函数进行一次封装，然后实现为原有方法增加一些功能的特殊实现。是不是感觉很抽象？来我们看个例子

~~~Python
def testDe1(func):
    def de(a, b, c):
        func(a, b, c)
        print('1')
    print('2')
    return de

@testDe1
def test2(a, b, c):
    print(a+b+c)
if __name__ == '__main__':
    test2(1,2,3)

~~~
来，告诉我，这段代码的输出应该是什么？答案是 `2,6,1`，看到这里，你是不是感觉似乎明白了些什么？是的没错，上面的例子其实等价于

~~~Python
def testDe1(func):
    def de(a, b, c):
        func(a, b, c)
        print('1')
    print('2')
    return de

def test2(a, b, c):
    print(a+b+c)

if __name__ == '__main__':
    testDe1(test2)(1,2,3)

~~~
那么我们换个例子

~~~Python
def testDe1(func):
    def de(a, b, c):
        func(a, b, c)
        print('1')
    print('2')
    return de

@testDe1
def test2(a, b, c):
    print(a+b+c)
if __name__ == '__main__':
    pass
~~~

这段代码的输出会是什么？是的没错，这段代码的输出是 `2` 。看到这里你是不是感觉更明白些什么？恩，在 **Python** 中，使用函数装饰器的时候，等于先行调用了装饰函数一次，具体来讲在使用装饰器后，装饰器会用装饰后的函数来进行一个替换，即在什么也不做的情况下，会产生这样一个调用 `test2=testDe1(test2)`，接着如果在 `__main__` 中添加一段代码 `test2(1,2,3)`,是不是就等价于 `testDe1(test2)(1,2,3)` 。看到这里是不是彻底明白了？
恩，来，我们再来复习下前面的例子
~~~Python

@app.route('/')
def show_entries():
    db = get_db()
    cur = db.execute('select title, text from entries order by id desc')
    entries = cur.fetchall()
    return render_template('show_entries.html', entries=entries)
~~~

上面这段代码里发生了什么？是不是有一个调用为 `show_entries=app.route('/')(show_entries)`？看到这里是不是很清楚了呢？

第二个，在第一个小 tip 的基础之上，我们来讲一个关于装饰器传参的问题
可能很多人不清楚装饰器传参的使用情景，首先如前面所说装饰器的最根本的作用在于

>在不修改原有代码的基础上，对函数进行一次封装，然后实现为原有方法增加一些功能的特殊实现

现在假设我们需要对函数的运行时间进行输出，这个时候我们该怎么办

~~~Python
def testTime(func):
    def dec(*args,**kwargs):
        flag=time.time()
        func(*args,**kwargs)
        print(time.time()-flag)
    return dec

def func():
    pass

if __name__=='__main__':
    func()
~~~
如前所述，前面这段代码等价于 `test(func)()`,那么这个时候我们想给我们时间输出以一定的单位进行格式化怎办，修改上面装饰器代码如下

~~~Python
def testtime(time=None):
    def dec1(func):
        def dec2(*args,**kwargs):
            flag=time.time()
            func(*args,**kwagrs)
            flag2=time.time()
            if time is not None:
                print((flag2-flag)/time)
            else:
                print(flag2-flag)
        return dec2
    return dec1

~~~
写到这里，大家是不是明白了带参数的装饰器的使用情景呢？

### 后记
**Flask** 的路由系统相对简单，其本质是利用带参数的装饰器来进行相应的路由记录，同时利用装饰器的包装特性，将我们的对应的处理函数进行包装，同时加入路由表中，一旦触发我们所注册的路由，便可调用我们所对应的处理函数。
