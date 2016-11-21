---
title: Python 描述符入门指北
type: tags
date: 2016-10-12 22:48:30
tags: [Python,编程,协程,编程技巧]
categories: [编程,Python]
---
很久都没写 **Flask** 代码相关了，想想也真是惭愧，然并卵，这次还是不写 **Flask** 相关，不服你来打我啊（就这么贱，有本事咬我啊
这次我来写一下 Python 一个很重要的东西，即 Descriptor （描述符）
<!-- more -->

## 初识描述符
老规矩，**Talk is cheap,Show me the code.** 我们先来看看一段代码

~~~Python
class Person(object):
    """"""

    #----------------------------------------------------------------------
    def __init__(self, first_name, last_name):
        """Constructor"""
        self.first_name = first_name
        self.last_name = last_name

    #----------------------------------------------------------------------
    @property
    def full_name(self):
        """
        Return the full name
        """
        return "%s %s" % (self.first_name, self.last_name)

if __name__=="__main__":
    person = Person("Mike", "Driscoll")
    print(person.full_name)
    # 'Mike Driscoll'
    print(person.first_name)
    # 'Mike'
~~~
这段代大家肯定很熟悉，恩，`property` 嘛，谁不知道呢，但是 `property` 的实现机制大家清楚么？什么不清楚？那还学个毛的 Python 啊。。。开个玩笑，我们看下面一段代码

~~~Python
class Property(object):
    "Emulate PyProperty_Type() in Objects/descrobject.c"
    def __init__(self, fget=None, fset=None, fdel=None, doc=None):
        self.fget = fget
        self.fset = fset
        self.fdel = fdel
        if doc is None and fget is not None:
            doc = fget.__doc__
        self.__doc__ = doc

    def __get__(self, obj, objtype=None):
        if obj is None:
            return self
        if self.fget is None:
            raise AttributeError("unreadable attribute")
        return self.fget(obj)

    def __set__(self, obj, value):
        if self.fset is None:
            raise AttributeError("can't set attribute")
        self.fset(obj, value)

    def __delete__(self, obj):
        if self.fdel is None:
            raise AttributeError("can't delete attribute")
        self.fdel(obj)

    def getter(self, fget):
        return type(self)(fget, self.fset, self.fdel, self.__doc__)

    def setter(self, fset):
        return type(self)(self.fget, fset, self.fdel, self.__doc__)

    def deleter(self, fdel):
        return type(self)(self.fget, self.fset, fdel, self.__doc__)
~~~
看起来是不是很复杂，没事，我们来一步步的看。不过这里我们首先给出一个结论：**Descriptors 是一种特殊 的对象，这种对象实现了 `__get__` ，`__set__` ，`__delete__` 这三个特殊方法。**

## 详解描述符

### 说说 Property

在上文，我们给出了 Propery 实现代码，现在让我们来详细说说这个

~~~Python
class Person(object):
    """"""

    #----------------------------------------------------------------------
    def __init__(self, first_name, last_name):
        """Constructor"""
        self.first_name = first_name
        self.last_name = last_name

    #----------------------------------------------------------------------
    @Property
    def full_name(self):
        """
        Return the full name
        """
        return "%s %s" % (self.first_name, self.last_name)

if __name__=="__main__":
    person = Person("Mike", "Driscoll")
    print(person.full_name)
    # 'Mike Driscoll'
    print(person.first_name)
    # 'Mike'
~~~

首先，如果你对装饰器不了解的话，你可能要去看看这篇[文章](http://manjusaka.itscoder.com/2016/08/09/reading-the-fucking-flask-source-code-Part1/)，简而言之，在我们正式运行代码之前，我们的解释器就会对我们的代码进行一次扫描，对涉及装饰器的部分进行替换。类装饰器同理。在上文中，这段代码

~~~Python
    @Property
    def full_name(self):
        """
        Return the full name
        """
        return "%s %s" % (self.first_name, self.last_name)
~~~

会触发这样一个过程，即  `full_name=Property(full_name)` 。然后在我们后面所实例化对象之后我们调用 `person.full_name` 这样一个过程其实等价于 `person.full_name.__get__(person)` 然后进而触发`__get__()` 方法里所写的 `return self.fget(obj)` 即原本上我们所编写的 `def full_name` 内的执行代码。

这个时候，同志们可以去思考下 `getter()` ,`setter()` ,以及 `deleter()` 的具体运行机制了=。=如果还是有问题，欢迎在评论里进行讨论。

### 关于描述符

还记得之前我们所提到的一个定义么：**Descriptors 是一种特殊的对象，这种对象实现了 `__get__` ，`__set__` ，`__delete__` 这三个特殊方法**。然后在 Python 官方文档的说明中，为了体现描述符的重要性，有这样一段话：“They are the mechanism behind properties, methods, static methods, class methods, and super(). They are used throughout Python itself to implement the new style classes introduced in version 2.2. ” 简而言之就是 **先有描述符后有天，秒天秒地秒空气**。恩，在新式类中，属性，方法调用，静态方法，类方法等都是基于描述符的特定使用。

OK，你可能想问，为什么描述符是这么重要呢？别急，我们接着看

### 使用描述符
首先请看下一段代码

~~~Python
class A(object): #注：在 Python 3.x 版本中，对于 new class 的使用不需要显式的指定从 object 类进行继承，如果在 Python 2.X（x>2)的版本中则需要
    def a(self):
        pass
if __name__=="__main__":
    a=A()
    a.a()
~~~

大家都注意到了我们存在着这样一个语句 `a.a()` ，好的，现在请大家思考下，我们在调用这个方法的时候发生了什么？
OK？想出来了么？没有？好的我们继续
首先我们调用一个属性的时候，不管是成员还是方法，我们都会触发这样一个方法用于调用属性 `__getattribute__()` ,在我们的 `__getattribute__()` 方法中，如果我们尝试调用的属性实现了我们的描述符协议，那么会产生这样一个调用过程 `type(a).__dict__['a'].__get__(b,type(b))`。好的这里我们又要给出一个结论了：“在这样一个调用过程中，有这样一个优先级顺序，如果我们所尝试调用属性是一个 `data descriptors` ，那么不管这个属性是否存在我们的实例的 `__dict__` 字典中，优先调用我们描述符里的 `__get__` 方法，如果我们所尝试调用属性是一个 `non data descriptors`，那么我们优先调用我们实例里的 `__dict__` 里的存在的属性，如果不存在，则依照相应原则往上查找我们类，父类中的 `__dict__` 中所包含的属性，一旦属性存在，则调用 `__get__` 方法，如果不存在则调用 ` __getattr__()` 方法”。理解起来有点抽象？没事，我们马上会讲，不过在这里，我们先要解释下 `data descriptors` 与 `non data descriptors`，再来看一个例子。什么是 `data descriptors` 与 `non data descriptors` 呢？其实很简单，在描述符中同时实现了 `__get__` 与 `__set__` 协议的描述符是 `data descriptors` ，如果只实现了 `__get__` 协议的则是 `non data descriptors` 。好了我们现在来看个例子：

~~~Python
import math
class lazyproperty:
    def __init__(self, func):
        self.func = func

    def __get__(self, instance, owner):
        if instance is None:
            return self
        else:
            value = self.func(instance)
            setattr(instance, self.func.__name__, value)
            return value
class Circle:
    def __init__(self, radius):
        self.radius = radius
        pass

    @lazyproperty
    def area(self):
        print("Com")
        return math.pi * self.radius * 2

    def test(self):
        pass
if __name__=='__main__':
    c=Circle(4)
    print(c.area)
~~~

好的，让我们仔细来看看这段代码，首先类描述符 `@lazyproperty` 的替换过程，前面已经说了，我们不在重复。接着，在我们第一次调用 `c.area` 的时候，我们首先查询实例 `c` 的 `__dict__` 中是否存在着 `area` 描述符，然后发现在 `c` 中既不存在描述符，也不存在这样一个属性，接着我们向上查询 `Circle` 中的 `__dict__` ，然后查找到名为 `area` 的属性，同时这是一个 `non data descriptors` ，由于我们的实例字典内并不存在 `area` 属性，那么我们便调用类字典中的 `area` 的 `__get__` 方法，并在 `__get__` 方法中通过调用 `setattr` 方法为实例字典注册属性 `area` 。紧接着，我们在后续调用 `c.area` 的时候，我们能在实例字典中找到 `area` 属性的存在，且类字典中的 `area` 是一个 `non data descriptors`，于是我们不会触发代码里所实现的 `__get__` 方法，而是直接从实例的字典中直接获取属性值。

### 描述符的使用

描述符的使用面很广，不过其主要的目的在于让我们的调用过程变得可控。因此我们在一些需要对我们调用过程实行精细控制的时候，使用描述符，比如我们之前提到的这个例子

~~~Python
class lazyproperty:
    def __init__(self, func):
        self.func = func

    def __get__(self, instance, owner):
        if instance is None:
            return self
        else:
            value = self.func(instance)
            setattr(instance, self.func.__name__, value)
            return value

    def __set__(self, instance, value=0):
        pass


import math


class Circle:
    def __init__(self, radius):
        self.radius = radius
        pass

    @lazyproperty
    def area(self, value=0):
        print("Com")
        if value == 0 and self.radius == 0:
            raise TypeError("Something went wring")

        return math.pi * value * 2 if value != 0 else math.pi * self.radius * 2

    def test(self):
        pass
~~~

利用描述符的特性实现懒加载，再比如，我们可以控制属性赋值的值

~~~Python
class Property(object):
    "Emulate PyProperty_Type() in Objects/descrobject.c"
    def __init__(self, fget=None, fset=None, fdel=None, doc=None):
        self.fget = fget
        self.fset = fset
        self.fdel = fdel
        if doc is None and fget is not None:
            doc = fget.__doc__
        self.__doc__ = doc

    def __get__(self, obj, objtype=None):
        if obj is None:
            return self
        if self.fget is None:
            raise AttributeError("unreadable attribute")
        return self.fget(obj)

    def __set__(self, obj, value=None):
        if value is None:
            raise TypeError("You can`t to set value as None")
        if self.fset is None:
            raise AttributeError("can't set attribute")
        self.fset(obj, value)

    def __delete__(self, obj):
        if self.fdel is None:
            raise AttributeError("can't delete attribute")
        self.fdel(obj)

    def getter(self, fget):
        return type(self)(fget, self.fset, self.fdel, self.__doc__)

    def setter(self, fset):
        return type(self)(self.fget, fset, self.fdel, self.__doc__)

    def deleter(self, fdel):
        return type(self)(self.fget, self.fset, fdel, self.__doc__)

class test():
    def __init__(self, value):
        self.value = value

    @Property
    def Value(self):
        return self.value

    @Value.setter
        def test(self, x):
            self.value = x
~~~

如上面的例子所描述的一样，我们可以判断所传入的值是否有效等等。

## 总结
Python 中的描述符可以说是新式类调用链中的根基，所有的方法，成员，变量调用时都将会有描述符的介入。同时我们可以利用描述符的特性来将我们的调用过程变得更为可控。这一点，我们可以在很多著名框架中找到这样的例子。

## 参考
1.[《Python Cookbook》](http://item.jd.com/11681561.html) 8.10 章 P271
2.[《Descriptor HowTo Guid》](https://docs.python.org/3/howto/descriptor.html)
3.[《Python 黑魔法》](http://www.jianshu.com/p/250f0d305c35)
