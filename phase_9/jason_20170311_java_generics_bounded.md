---
title: 你了解泛型通配符与上下界吗？
date: 2017-03-11 10:30:16
categories: Java
tags: [Java] #文章标签，可空，多标签请用格式，注意:后面有个空格
description: 你了解泛型通配符与上下界吗？
---

# 你了解泛型通配符与上下界吗？
在进入主题之前， 我们先简单说一下 Java 的泛型（generics）。它是JDK 5中引入的一个新特性，允许在定义类和接口的时候使用类型参数（type parameter）。声明的类型参数在使用时用具体的类型来替换。泛型最主要的应用是在JDK 5中的新集合类框架中。

今天我们主要说如下类型：
+ 泛型的背景
+ 通配符以及上下界
+ 泛型及通配符的使用场景

## 为什么使用泛型及背后的问题？
我们来看一下官方的说法：
>+ Stronger type checks at compile time. A Java compiler applies strong type checking to generic code and issues errors if the code violates type safety. Fixing compile-time errors is easier than fixing runtime errors, which can be difficult to find.
>+  Elimination of casts.
>+ Enabling programmers to implement generic algorithms. By using generics, programmers can implement generic algorithms that work on collections of different types, can be customized, and are type safe and easier to read.

是的， 终止目的就是想把程序员解放出来，关注他们更应该关注的事情上面去。当我第一次学习 Java 的泛型时，总感觉它类似于 C++ 中的模板。但随着慢慢的深入了解发现它们之间有本质的区别。

Java 中的泛型基本上完全在编译器中实现，由编译器执行类型检查和类型推断，然后生成普通的非泛型的字节码。这种实现技术称为 擦除（erasure）（编译器使用泛型类型信息保证类型安全，然后在生成字节码之前将其清除），这项技术有一些奇怪，并且有时会带来一些令人迷惑的后果。

对于泛型概念的引入，开发社区的观点是褒贬不一。从好的方面来说，上面已经说了，主要是在编译时刻就能发现很多明显的错误。而从不好的地方来说，主要是为了保证与旧有版本的兼容性，Java 泛型的实现上存在着一些不够优雅的地方。

下面我们来看一下，泛型类型的一个定义，后面我们要在这个的基础上进行改造：
```
public class Box<T> {
	// T stands for "Type"
	private T t;

	public Box(T t) ￼ ￼{ this.t = t;￼ }￼
	public void set(T t) { this.t = t; }
	public T get() { return t; }
}
```

接下来下面我们来聊聊 Java 泛型的通配符， 记得刚开始看到通配符（？）时我是惊喜的，因为既然有通配符那么就可以这样定义：
```
public void doSometing(List<?> list) {
	list.add(1); //illegal
}
``` 

可是我们如上写法，总是出现编译错误，然后从惊喜变成惊吓，心想有什么卵用了。最后发现原因是在于通配符的表示的类型是未知的。那在这种情况下，我们可以使用上下界来限制未知类型的范围。好吧，写了那么多， 终于等到今天的主角登场了，容易吗？

还记得我们上面定义的 Box 吗， 现在我们再定义 Fruit 类以及它的子类 Orange 类。
```
class Fruit { }
class Orange extends Fruit {}
```

现在我们想它里面能装水果，那么我可以这么写。
`Box<Fruit> box = Box<Orange>(new Orange) //illegal`

不幸的是编译器会报错，这就尴尬了，why？why？ why？实际上，编译器认为的容器之间没有继承关系。所以我们不能这样做。

为了解决这样的问题， 大神们想出来了<? extens T> 和 <? super T> 的办法，来让它们之间发生关系。
##  上界通配符（Upper Bounded Wildcards）
现在我们把上面的 Box 定义改成：
`Box<? extends Fruit>`

这就是上界通配符， 这样 Box<Fruit> 及它的子类如 Box<Orange> 就可以赋值了。
`Box<? extends Fruit> box = new Box<Orange>(new Orange)`

当我们扩展一下上面的类， 食物分成为水果和蔬菜类， 水果有苹果和橘子。
在上面的结构中， Box<? extends Fruit> 涵盖下面的蓝色的区域。

![](http://7xnilf.com1.z0.glb.clouddn.com/upper.png)

### 上界只能外围取，不能往里放
我们先看一下下面的例子：
```
Box<? extends Fruit> box = new Box<Orange>(new Orange);

//不能存入任何元素
box.set(new Fruit);  //illegal
box.set(new Orange);//illegal

//取出来的东西只能存放在Fruit或它的基类里
Fruit fruit = box.get();
Object fruit1 = box.get();
Orange fruit2 = box.get(); //illegal
```

上面的注释已经很清楚了， 往 Box 里放东西的 set() 方法失效， 但是 get() 方法有效。

原因是 Java 编译器只知道容器内是 Fruit 或者它的派生类， 但是不知道是什么类型。可能是 Fruit、 可能是 Orange、可能是Apple？当编译器在看到 box 用 Box<Orange> 赋值后， 它就把容器里表上占位符 “AAA” 而不是 “水果”等，当在插入时编译器不能匹配到这个占位符，所有就会出错。

## 下界通配符（Lower Bounded Wildcards）
和上界相对的就是下界 ，语法表示为：
`<? super T>`

表达的相反的概率：一个能放水果及一切水果基类的 Box。 对应上界的那种图， 下图 Box<? super Fruit> 覆盖黄色区域。

![](http://7xnilf.com1.z0.glb.clouddn.com/lower.png)

### 下界不影响往里存，但往外取只能放在Object 对象里
同上界的规则相反，**下界不影响往里存，但往外取只能放在Object 对象里**。

因为下界规定元素的最小的粒度，实际上是容器的元素的类型控制。所以放比 Fruit 粒度小的如 Orange、Apple 都行， 但往外取时， 只有所有类的基类Object对象才能装下。但是这样的话，元素的类型信息就全部消失了。

# 写在最后
## 使用场景  
在使用泛型的时候可以遵循一些基本的原则，从而避免一些常见的问题。
+ 在代码中避免泛型类和原始类型的混用。比如List<String>和List不应该共同使用。这样会产生一些编译器警告和潜在的运行时异常。当需要利用JDK 5之前开发的遗留代码，而不得不这么做时，也尽可能的隔离相关的代码。
+ 在使用带通配符的泛型类的时候，需要明确通配符所代表的一组类型的概念。由于具体的类型是未知的，很多操作是不允许的。
+ 泛型类最好不要同数组一块使用。你只能创建new List<?>[10]这样的数组，无法创建new List<String>[10]这样的。这限制了数组的使用能力，而且会带来很多费解的问题。因此，当需要类似数组的功能时候，使用集合类即可。
+ 不要忽视编译器给出的警告信息。

### PECS 原则
如果要从集合中读取类型T的数据， 并且不能写入，可以使用 上界通配符（<？extends>）—Producer Extends。

如果要从集合中写入类型T 的数据， 并且不需要读取，可以使用下界通配符（<? super>）—Consumer Super。

如果既要存又要取， 那么就要使用任何通配符。

