---
layout: post
title: "Thinking in Java 学习笔记之复用类与多态"
date: 2017-02-25 00:00:00 +0800
category: Java
excerpt: 组合与继承是 Java 中重要的两种代码重用机制，与之引申出多态，这个在面向对象的程序设计语言中，继数据抽象和继承之后的第三种基本特征。
---

> 导语：《Thinking in Java》系列笔记，因为在此前的学习过程中，一些比较细的知识点没有梳理和记忆，容易忘记，所以在看完《疯狂 Java 讲义》后，决定在读这本书的过程中，将自己觉得重要或者是自己平时并不关注的细节给记录下来。同时，也可以查看我读书所做的[思维导图](http://naotu.baidu.com/file/c2d3c32533ee65a57ea46aecf4dce3cc?token=974de6a59ff1a15c)（读完之前持续更新）。当你读到这篇文章时，我的进度是这样的：

![Thinking in Java MindMap](http://7xtt0k.com1.z0.glb.clouddn.com/weeklyBlog/NO.9/ThinkingInJavaMindMap.png)

我们知道，类是一种高度抽象的类型结构体，用以表示除去基本类型无法表述的那些复杂类型。在 C++ 中，类的数据成员甚至不能进行初始化赋值，因为类是一种抽象数据类型，并不占存储空间，所以所赋的值并不能被存储。

而代码重用，一直被视为是一个高级语言所必须具备的组成部分，代码的复用，即类的复用，其中最重要的就是组合与继承两种机制。

#### 组合

**什么是组合？**

顾名思义，组合就是像搭积木一样分块地拼接起来。在 Java 中，那一块块积木便是类。构造新类以形成一种新的数据类型，将其实例对象（就是常说的 new 一个对象）作为一个类的内部成员变量，进行 “拼接”，这就是组合。

在读书过程中学习到一个之前不曾注意的方法： **toString()** 。

每一个非基本类型对象都有一个 **toString()** 方法，而且当编译器需要一个 **String** 而你却只有一个对象时，该方法便会被调用。Talk is cheap, show me the code!

```java
// Test.java

class Person {
    public string toString() {
        return "I am a good guy!";
    }
}

public class Test {
    public static void main(String[] args) {
        Person p = new Person();
        System.out.println(p);
    }
}

//~ output: I am a good guy!
```

#### 继承

**什么是继承？**

继承是所有面向对象语言中不可缺少的组成部分，**当你创建一个类时，你总是在继承** 。可别忘了 Java 的单根继承结构，在除了 C++ 以外的所有面向对象语言中，所有的类最终都继承自单一的基类，在 Java 中就是 Object，而你之所以看不到，是因为这是语言设计之初的一种框架结构，给隐藏起来了。那么到底什么是继承？我总结为：由通用或高度抽象的类作为基类，根据需求，导出更为实用具体的子类，而子类就继承自基类，获得基类的域和所有方法。

因为继承，子类可以自动获得基类的所有域和方法，这就是代码的复用，你也可以根据具体的需求，在子类中添加新的方法，或者覆盖重写继承所得的方法。

为了继承，一般的规则是将所有的数据成员都指定为 **private** ，将所有的方法指定为 **public** 。

> 向上转型

由继承，引出了向上转型这一概念。可以想象，由继承带来的，是一种像树根一样的框架结构，自上而下，其中任一个类的基类，都是比它更为抽象的数据类型，子类和基类的关系可用 “子类是基类的一种类型” 这句话来概括。如下：

```java
// ShowZeng.java

class People {
    public void kiss() {
        System.out.println("Give you a kiss!");
    }
    static void act(People p) {
        p.kiss();
    }
}

public class ShowZeng extends People {
    public static void main(String[] args) {
        ShowZeng zeng = new ShowZeng();
        People.act(zeng);
    }
}

//~ output:Give you a kiss!
```

可以看到 **People** 类中的 **act(People p)** 方法传入的参数应该是 **People** 类型的实例对象，然而这里传入的是 **People** 的子类而没有出错，这种将子类引用转换为基类引用的动作，称之为 **向上转型** 。

#### 再论组合与继承

虽然面向对象语言多次强调继承，但并不意味着要尽量使用它，相反，应当慎用这一技术。那么，什么情况下该使用继承，什么情况下使用组合？一个最清晰的判断方法就是：首先考虑是否需要从新类向基类进行向上转型，如果需要向上转型，则继承是必须的。反之则仔细思考是否有必要使用继承。

#### 多态

什么是多态？我对多态的理解是：由继承引出的 **向上转型** 和 **动态绑定** 来实现某些功能。例如：可用基类类型作为参数，而可传入任意子类，从而实现参数的多态化，其中涉及向上转型和动态绑定。

> 绑定

将一个方法调用和一个方法主体关联起来叫做 **绑定** 。

若在程序执行前进行绑定（如果有的话，由编译器和连接程序实现），叫做 **前期绑定** 。这是面向过程语言默认的绑定方式，例如 C 语言中只有一种方法调用，那就是前期绑定。

而在运行时根据对象的类型进行绑定，就称为 **后期绑定** ，又叫做 **运行时绑定** 或 **动态绑定** 。

用一个例子来说明多态（以下是用来说明的文字表述，不代表 Java 语法），如下：

```java
人 {
    打(手) {}
}

小王 继承 人 {
    打(手) {
        return "小王动手打人了！";
    }
}

小李 继承 人 {
    打(手) {
        return "小李动手打人了！";
    }
}

主类 {
    main方法() {
        人 someone = new random(小王(), 小李());

        打人了(someone) {
            someone.打(手);
        }
    }
}
```

这里，**someone** 变量是一个人的类型，其所赋予的实例引用只能在运行时获得，且是随机的（所以称之为动态绑定），所有其输出结果可能是 “小王打人了！”，也可能是 “小李打人了！”。由人这个基类类型作为方法参数，而可以传入其任一子类（子类引用传入时向上转型为基类引用），得到不同的输出结果，这样就具有了多态性，使得程序更加地灵活。

#### 多态的缺陷

> 覆盖私有方法

对于基类的 **private** 方法，导出类尝试 “覆盖” 其方法，但在调用此方法时，其结果是调用基类的方法。

即：非 private 方法才能被覆盖，否则在导出类中只是一个全新的方法。所以在导出类中，对于基类中的 private 方法，最好采用不同的名字。

> 域与静态方法

域：如果基类和子类中有一同名公开变量 **field** 并初始化以不同的值,那么在实例化 `BaseClass bc = new SubClass();` 之后，你输出 `bc.field` 的时候会发现是 **基类的变量值** 。而在实例化 `SubClass sb = new SubClass();` 之后， `sb.field` 就自然是子类的变量值，而此时若需得到基类变量值，就需显式地指定 `super.field` （当然，一般情况下我们并不会这么做，通常基类变量都指定为私有，而子类也不会取同名变量，这里只是说明这种特殊情况，仍然值得我们注意）。因为在运行前，编译器已经为 **BaseClass.field** 和 **SubClass.field** 分配了不同的存储空间，因此，任何域访问操作都将由编译器解析，而不具多态性。

静态方法：同样的，如果某个方法是静态的，它的行为不具多态性。在基类有一静态方法 **baseStaticMethod()**，在子类中试图去覆盖它。实例化 `BaseClass bc = new SubClass();`，调用 `bc.baseStaticMethod()；` 时，运行的是基类的方法体，而此静态方法就不具多态性了。

#### 构造器与多态

> 构造器的调用顺序

按继承层次逐层向上链接。

1）调用基类构造器。会反复递归，由根类向下调用，直至最底层的导出类。

2）按声明顺序调用成员的初始化方法。

3）调用导出类的构造器。

看看下面这个例子,测测你是否真的懂了初始化顺序：

```java
class Meal {
    Meal() { System.out.println("Meal()"); }
}

class Bread {
    Bread() { System.out.println("Bread()"); }
}

class Cheese {
    Cheese() { System.out.println("Cheese()"); }
}

class Lettuce {
    Lettuce() { System.out.println("Lettuce()"); }
}

class Lunch extends Meal {
    private Bread b = new Bread();

    private void myMethod() {
        Meal m = new Meal();
    }

    Lunch() { System.out.println("Lunch()"); }
}

class PortableLunch extends Lunch {
    PortableLunch() { System.out.println("PortableLunch()"); }
}

public class Sandwich extends PortableLunch {
    private Bread b = new Bread();
    public Cheese c = new Cheese();


    public Sandwich() { System.out.println("Sandwich()"); }

    private Lettuce l = new Lettuce();

    public static void main(String[] args) {
        new Sandwich();
    }
}
```

运行结果如下：

![initialization](http://7xtt0k.com1.z0.glb.clouddn.com/weeklyBlog/NO.9/initialization.png)

> 初始化的实际过程

1） 在其他任何事物发生之前，将分配给对象的存储空间初始化为二进制的零。

2） 如上所述调用构造器的顺序。

3） 按照声明的顺序调用成员的初始化方法。

4） 调用导出类的构造器主体。

**编写构造器的有效准则：尽可能用简单的方法使对象进入正常状态，如果可以，避免调用其他方法。**

同样的，给出一个例子，来试试吧：

```java
class Glyph {
    void draw() { System.out.println("Glyph.draw()"); }

    Glyph() {
        System.out.println("Glyph() before draw()");

        draw();

        System.out.println("Glyph() after draw()");
    }
}

class RoundGlyph extends Glyph {
    private int radius = 1;

    RoundGlyph(int r) {
        radius = r;
        System.out.println("RoundGlyph.RoundGlyph(), radius = " + radius);
    }

    void draw() {
        System.out.println("RoundGlyph.draw(), radius = " + radius);
    }
}

public class PolyConstructors {
    public static void main(String[] args) {
        new RoundGlyph(5);
    }
}
```

![PolyConstructors](http://7xtt0k.com1.z0.glb.clouddn.com/weeklyBlog/NO.9/PolyConstructors.png)

这里是因为子类覆盖了基类的 **draw()** 方法，如果去掉子类的 **draw()** 方法，那么 **Glyph** 构造器里调用的就是基类自身的 **draw()** 方法了。

#### 协变返回类型

协变返回类型：导出类中被覆盖的方法，可以返回基类方法的返回类型的某种导出类。举个栗子：

```java
// 这里Wheat 类继承自 Grain 类

class Mill {
    Grain process() { return new Grain(); }
}

class WheatMill extends Mill {
    Wheat process() { return new Wheat(); }
}
```

即在早期 Java 版本中，将强制 **process()** 方法的覆盖版本必须返回 **Grain** ，不能返回 **Wheat** ，尽管 **Wheat** 继承自 **Grain** ，而后来添加了协变返回类型就允许返回更为具体的 **Wheat** 类型。

#### 用继承进行设计

> 优先考虑组合。

一个通用准则：用继承表达行为间的差异，并用字段表达状态上的变化。

为了在运行期间获得动态灵活性，可以用一个新的类以组合的形式，将各种导出类组合起来，达到动态绑定的目的，从而获得由于状态的不同使得方法产生动态变化的效果。

> 纯继承与扩展

**纯继承**：导出类与基类的接口一致，称之为 **is-a** 关系。

**扩展**：导出类中，除去与基类有着相同的基本接口，还增加了自己的方法，称之为 **is-like-a** 关系。
