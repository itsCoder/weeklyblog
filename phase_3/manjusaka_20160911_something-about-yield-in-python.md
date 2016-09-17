---
title: 聊聊 Python 中生成器和协程那点事儿
type: tags
date: 2016-09-11 10:27:15
tags: [Python,编程,协程,编程技巧]
categories: [编程,Python]
---
>- 文章来源：itsCoder 的 [WeeklyBolg](https://github.com/itsCoder/weeklyblog) 项目
>- itsCoder主页：[http://itscoder.com/](http://itscoder.com/)
>- 作者：[Manjusaka](https://github.com/Zheaoli)
>- 审阅者：[allenwu](https://github.com/wuchangfeng),[Brucezz](https://github.com/brucezz)

## 写在前面的话
本来想这周继续写写 **Flask** 那点破事儿的，但是想了想决定换换口味，来聊聊很不容易理解但是很重要的 **Python** 中的生成器和协程。
<!-- more -->

## Generators 科普
我猜大家对于生成器肯定并不陌生，但是为了能让我愉快的继续装逼，我们还是用点篇幅讲一下什么是生成器吧。
比如在 Python 里，我们想生成一个范围 (1,100000) 的一个 list，于是我们无脑写了如下的代码出来

~~~Python
def generateList(start,stop):
	tempList=[]
	for i in range(start,stop):
		tempList.append(i)
	return tempList
~~~
> 注1：这里有同学提出了为什么我们不直接返回 `range(start,stop)`，Nice question，这里涉及到一个基础问题，`range` 的机制究竟是怎样的。这就要分版本而论了，在 Python 2.x 的版本中，`range(start,stop)` 其实本质上是预先生成一个 `list` ,而 `list` 对象是一个 **Iterator** ，因此可以被 `for` 语句所使用。
![Python 2.x 中的 range](https://cloud.githubusercontent.com/assets/7054676/18607847/90092cf6-7d0b-11e6-9963-8f59f8ebbb28.png)
然后在 Python 2.x 中还有一个语句叫做 `xrange` ，其生成的是一个 **Generator** 对象。
![Python 2.x 中的 xrange](https://cloud.githubusercontent.com/assets/7054676/18607857/cf72561a-7d0b-11e6-9084-bc8a02404ed1.png)
在 Python 3 中事情发生了一点变化，可能社区觉得 `range` 和 `xrange` 分裂太过蛋疼，于是将其合并，于是现在在 Python 3 中，取消了 `xrange` 的语法糖，然后 `range` 的机制也变成生成一个 **Generator** 而不是 `list`
![Python 3 中的 range](https://cloud.githubusercontent.com/assets/7054676/18607878/6fc147f2-7d0c-11e6-9384-d1dd748ffb5d.png)


但是大家考虑过一个问题么，如果我们想生成数据量非常大，预先生成数据的行为无疑是很不明智的，这样会耗费大量的内存。于是 Python 给我们提供了一种新的姿势，**Generator** (生成器)

~~~Python
def generateList1(start,stop):
	for i in range(start,stop):
		yield i

if __name__=="__main__":
	c=generateList1(1,100000)
	for i in generateList1:
		print(i)
~~~
是的，**Generator** 其中一个特性就是不是一次性生成数据，而是生成一个可迭代的对象，在迭代时，根据我们所写的逻辑来控制其启动时机。

## Generator 深入

这里可能有一个问题，大家肯定想问 **Python** 开发者们不可能为了这一种使用场景而去单独创建一个 **Generator** 机制吧，那么我们 **Generator** 还有其余的使用场景么。当然，请看标题，对了嘛，**Generator** 另一个很大作用可以说就是当做协程使用。不过在这之前，我们要去深入的了解下 **Generator** 才能方便我们后面的讲解。

### 关于 Generator 中的内建方法

#### 关于 Python 中可迭代对象的一点背景知识
首先，我们来看看 Python 中的迭代过程。
在 Python 中迭代有两个概念，一个是 **Iterable** ，另一个是 **Iterator** 。让我们分别来看看
第N次首先，**Iterable** 近似的可以理解成为一个协议，判断一个 **Object** 是否是 **Iterable** 的方法就是看其实现了 **__iter__** 与否，如果实现了 **__iter__** ，那么这便可以认为是一个 **Iterable** 对象。空谈误国，实干兴邦，让我们直接来看一段代码理解下

~~~Python
class Counter:
    def __init__(self, low, high):
        self.current = low
        self.high = high

    def __iter__(self):
        return self

    def next(self):  # Python 3: def __next__(self)
        if self.current > self.high:
            raise StopIteration
        else:
            self.current += 1
            return self.current - 1

if __name__ == '__main__':
	a=Counter(3,8)
	for c in a:
		print(c)
~~~
好了，让我们来看看上面这段代码里发生了什么，首先 `for` 语句的引用首先去判断迭代的是 **Iterable** 对象还是 **Iterator** 对象，如果是实现了 `__iter__` 方法的对象，那么就是一个 **Iterable** 对象，`for` 循环首先调用对象的 `__iter__` 方法来获取一个 **Iterator** 对象。那么什么是 **Iterator** 对象呢，这里可以近似的理解为是实现了 **next()** 方法（注：在Python3中是 **__next__** 方法)。

OK，让我们继续回到刚刚说到的那里，在上面的代码中 `for` 语句首先判断是一个 **Iterable** 对象还是 **Iterator** 对象，如果是 **Iterable** 对象那么调用其 **__iter__** 方法来获取一个 **Iterator** 对象，接着 `for` 循环会调用 **Iterator** 对象中的 `next()` （注：Python3 里是 `__next__`)方法来进行迭代，直到迭代过程结束抛出 `StopIteration` 异常。

#### 好了，来聊聊 **Generator** 吧

让我们先看看前面那段代码吧：

~~~Python
def generateList1(start,stop):
	for i in range(start,stop):
		yield i

if __name__=="__main__":
	c=generateList1(1,100000)
	for i in generateList1:
		print(i)
~~~

首先我们要确定一点的是 **Generator** 其实也是一个 **Iterator** 对象。OK 让我们来看看上面这段代码，首先 `for` 确定 `generateList1` 是一个 **Iterator** 对象，然后开始调用 `next()` 方法进行进一步迭代。OK 此时你肯定想问这里面 `next()` 方法是怎样让 `generateList1` 进一步往下迭代的呢？答案在于 **Generator** 的内建 `send()` 方法。我们还是来看一段代码。

~~~Python
def generateList1(start,stop):
	for i in range(start,stop):
		yield i
if __name__=="__main__":
	a=generateList1(0,5)
	for i in range(0,5):
		print(a.send(None))
~~~
这里我们应该输出什么？答案就是 `0,1,2,3,4` ，结果上和我们用 `for` 循环进行运算的结果是不是一样。好了，我们现在可以得出一个结论就是

> **Generator** 迭代的本质就是通过内建的 `next()` 或 `__next__()` 方法来调用内建的 `send()` 方法。

### 继续吐槽 **Generator** 的内建方法

前面我们提到一个结论

> **Generator** 迭代的本质就是通过内建的 `next()` 或 `__next__()` 方法来调用内建的 `send()` 方法。

现在我们来看个例子：

~~~Python
def countdown(n):
    print "Counting down from", n
    while n >= 0:
        newvalue = (yield n)
        # If a new value got sent in, reset n with it
        if newvalue is not None:
            n = newvalue
        else:
            n -= 1

if __name__=='__main__':
	c = countdown(5)
	for x in c:
    	print x
    	if x == 5:
        	c.send(3)
~~~
好了这段代码的输出应该是什么？
答案是 `[5，2，1，0]` ，是不是很迷惑？别急，我们先来看看这段代码的运行流程

![代码运行流程](https://cloud.githubusercontent.com/assets/7054676/18417878/b17e6636-786e-11e6-9182-0e7a69b5611f.png)

简而言之就是，当我们调用 `send()` 函数的时候，我们 `send(x)` 的值会发送给 `newvalue` 向下继续执行直到遇到下一次 `yield` 的出现，然后返回值作为一个过程的结束。然后我们的 **Generator** 静静的沉睡在内存中，等待下一次的 `send` 来唤醒它。

> 注2：有同志问：“这里没想明白，c.send(3) 是 相当于 yield n 返回了个 3 给 newvalue ?”，好的，nice question，其实这个问题我们看前面之前的代码运行图就知道， `c.send(3)` 首先，将 `3` 赋值给 `newvalue` ，然后程序运行剩下的代码，直到遇到下一个 `yield` 为止，那么在这里，我们运行剩下完代码，在遇到 `yiled n` 之前，将 `n` 的值已经改变为 `3` ,接着，`yield n` 即约等于 `return 3`。接着 `countdown` 这个 **Generator** 将所有变量的状态冻结，然后静静的呆在内存中，等待下一次的 `next` 或 `__next__()` 方法或者是 `send()` 方法的唤醒。

> 小贴士：我们如果直接调用 `send()` 的话，第一次请务必 `send(None)` 只有这样一个 **Generator** 才算是真正被激活了。我们才能进行下一步操作。

## 说说关于协程

首先关于协程的定义，我们来看一段 wiki

> Coroutines are computer program components that generalize subroutines for nonpreemptive multitasking, by allowing multiple entry points for suspending and resuming execution at certain locations. Coroutines are well-suited for implementing more familiar program components such as cooperative tasks, exceptions, event loop, iterators, infinite lists and pipes.
According to Donald Knuth, the term coroutine was coined by Melvin Conway in 1958, after he applied it to construction of an assembly program.[1] The first published explanation of the coroutine appeared later, in 1963.

简而言之，协程是比线程更为轻量的一种模型，我们可以自行控制启动与停止的时机。在 **Python** 中其实没有专门针对协程的这个概念，社区一般而言直接将 **Generator** 作为一种特殊的协程看待，想想，我们可以用 `next` 或 `__next__()` 方法或者是 `send()` 方法唤醒我们的 **Generator** ，在运行完我们所规定的代码后， **Generator** 返回并将其所有状态冻结。这是不是很让我们 Excited 呢！！

## 关于 **Generator** 的一点课后作业

现在我们要后序遍历二叉树，我知道看这篇文章神犇们都能无脑写出来的，让我们看看代码先：

~~~Python
class Node(object):
    def __init__(self, val, left, right):
        self.val = val
        self.left = left
        self.right = right

def visit_post(node):
    if node.left:
        return visit_post(node.left)
    if node.right:
        return visit_post(node.right)
    return node.val

if __name__ == '__main__':
    node = Node(-1, None, None)
    for val in range(100):
        node = Node(val, None, node)
    print(list(visit_post(node)))
~~~
但是，我们知道递归深度太深的话，我们要么爆栈要么 py 交易失败，OK ，**Generator** 大法好，把你码农平安保，还是直接看代码：
~~~Python
def visit_post(node):
    if node.left:
        yield node.left
    if node.right:
        yield node.right
    yield node.val

def visit(node, visit_method):
    stack = [visit_method(node)]
    while stack:
        last = stack[-1]
        try:
            yielded = next(last)
        except StopIteration:
            stack.pop()
        else:
            if isinstance(yielded, Node):
                stack.append(visit_method(yielded))
            elif isinstance(yielded, int):
                yield yielded

if __name__ == '__main__':
    node = Node(-1, None, None)
    for val in range(100):
        node = Node(val, None, node)
    visit_generator = visit(node, visit_method=visit_post)
    print(list(visit_generator))
~~~

看起来很复杂是不是？没事当做课后作业，大家可以在评论里给我留言，我们一起进行一下 py 交易吧~

## 参考链接
1.[提高你的Python: 解释‘yield’和‘Generators（生成器）’](http://www.oschina.net/translate/improve-your-python-yield-and-generators-explained)
2.[yield大法好](http://www.jianshu.com/p/b37a92a5ada0)
3.[http://my.oschina.net/1123581321/blog/160560](http://my.oschina.net/1123581321/blog/160560)
4.[python的迭代器为什么一定要实现__iter__方法](https://www.zhihu.com/question/44015086)(关于迭代器那离，为了便于理解，我简化了一些东西，具体可以参看这个问题的高票答案)
