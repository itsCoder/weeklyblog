---
title: 关于 Lambda 表达式的一些事
date: 2016-09-24 20:46:25
tags:
- Java8
- Lambda
categories:
- Java
- 基础知识
---

　　从1998年 Java 发布以来，Java 版本从1.1到目前的 Java8，Java一直在升级，在增加新功能。这也就很好的解释了为什么 Java 在编程语言排行榜上一直占据前排位置。不断的创新才是保持优势的正确方式。Java8 在2014年被发布，提供了更多的编程工具和概念，让开发人员以更加简洁、易维护的方式去解决问题。Lambda 表达式作为 Java8 中一个重要的特性，是不可不学的一部分。

## 为什么需要 Lambda 表达式

### 怎么解决点外卖的问题

　　都说人生有三大难题：早饭吃什么？午饭吃什么？晚饭吃什么？每当到了
饭点，在决定点哪家外卖的时候，都像是在作出人生抉择。我们就从点外卖这件事上来讲一讲为什么需要引入 Lambda 表达式。

　　首先，我们定义一下我们供我们点外卖的店铺：

``` java
public class Store {
    private String name; //商店名称
    private double avePrice; //平均价格
    private int aveDeliveryTime; //平均送餐时间
    private String type; //商店类型

    public Store(){

    }

    public Store(String name,double avePrice,int aveDeliveryTime,String type){
        this.name = name;
        this.avePrice = avePrice;
        this.aveDeliveryTime = aveDeliveryTime;
        this.type = type;
    }

	@Override
    public String toString() {
        return String.format("店铺名称：%s  平均价格：%d  平均送餐时间：%d  店铺类型：%s",
                this.name,this.avePrice,this.aveDeliveryTime,this.type);
    }
	/*省略get和set方法*/
}
```

　　接着，我们开始定外卖，现在到月底了，又没钱了，点外卖也不挑什么了，就找最便宜的，省钱是硬道理。找最便宜的外卖店应该怎么找呢，很自然的就是对店铺的价格进行排序。

``` java
public static void main(String[] args){
        List<Store> stores = new ArrayList<Store>();
        stores.add(new Store("店铺一",20,10,"快餐"));
        stores.add(new Store("店铺二",30,20,"烧烤"));
        stores.add(new Store("店铺三",10,20,"披萨"));
        stores.add(new Store("店铺四",15,30,"炸鸡"));
        stores.add(new Store("店铺五",45,20,"甜品"));
        Collections.sort(stores, new Comparator<Store>() {
            public int compare(Store o1, Store o2) {
                return Double.compare(o1.getAvePrice(),o2.getAvePrice());
            }
        });
    }
```

　　可以看到这样的确解决了问题，但是当我们有新的需求的时候，比如说我想找出送餐时间最短的店铺，或者是我只想找出卖快餐的店铺时，这个代码就变得不适用了，我们需要更改代码来实现新的功能。这个还是简单的情况，真实开发中情况更加复杂，显然每次遇到新的需要就大改代码这样很不优雅，该怎么解决这个问题。熟悉设计模式的朋友很快就会想到，这很符合策略模式的情况，我们每次只是更改不同的选择店铺策略去选择店铺。

### 用策略模式来解决点外卖的问题

　　下面我们从新用策略模式来完成我们挑选外卖店铺的任务。我们将挑选店铺的策略抽象出来:

``` java
public interface Selector {
    public List<Store> select(List<Store> stroes);
}
```

　　然后实现这个选择:

``` java
//将店铺按照价格排序
class PriceSelector implements Selector{
    public List<Store> select(List<Store> stroes) {
        Collections.sort(stroes, new Comparator<Store>() {
            public int compare(Store o1, Store o2) {
                return Double.compare(o1.getAvePrice(),o2.getAvePrice());
            }
        });
        return stroes;
    }
}
//将店铺按照送餐时间排序
class TimeSelector implements Selector{
    public List<Store> select(List<Store> stroes) {
        Collections.sort(stroes, new Comparator<Store>() {
            public int compare(Store o1, Store o2) {
                return o1.getAveDeliveryTime() - o2.getAveDeliveryTime();
            }
        });
        return stroes;
    }
}
//选择所有的快餐店
class TypeSelector implements  Selector{
    public List<Store> select(List<Store> stroes) {
        List<Store> result = new ArrayList<Store>();
        for(Store store:stroes){
            if("快餐".equals(store.getType())){
                result.add(store);
            }
        }
        return result;
    }
}
```

　　之后我们就可以同传递不同的选择方式来选择外卖店:

``` java
	public static List<Store> selectStore(List<Store> stores,Selector s){
        return s.select(stores);
    }

    public static void main(String[] args){
        List<Store> stores = new ArrayList<Store>();
        stores.add(new Store("店铺一",20,10,"快餐"));
        stores.add(new Store("店铺二",30,20,"烧烤"));
        stores.add(new Store("店铺三",10,20,"披萨"));
        stores.add(new Store("店铺四",15,30,"炸鸡"));
        stores.add(new Store("店铺五",45,20,"甜品"));
        Selector s = new TypeSelector();
        stores = selectStore(stores,s);
        for (Store store : stores){
            System.out.println(store.toString());
        }
    }
```

　　这样一来看上去貌似是解决了我们的问题，当有新的选择方式的时候，我们只需要实现一个新的选择类，用户端基本上不用改变任何代码。但是仔细一考虑，当选择方式越来越多的时候，每次都需要重新实现新的类，而且仔细观察发现每个选择类中真正不同的只有 `select()` 方法中的代码而已。

### Lambda 在点外卖为题中的应用

　　有什么办法可以让我们不用每次都写那么繁琐的代码呢，这个时候就需要 Lambda 表达式登场了。等了这么久，终于写到文章的主题了（我的文章就是这么又臭又长）。为什么 Lambda 表达式可以简化我们的代码呢，因为 Lambda 提出的是一个**行为参数化**的概念，首先**行为**就是我们上面讲到的 `select()` 方法中的代码，这是选择的具体行为，**参数化**就是想方法中传递参数一样。**行为参数化**就是定义了一种新的编程方式，让我们可以将代码像参数一样传递到方法中使用。

　　看一下我们使用 Lambda 表达式完成店铺选择的功能是怎么样的：

``` java
public static void main(String[] args){
        List<Store> stores = new ArrayList<Store>();
        stores.add(new Store("店铺一",20,10,"快餐"));
        stores.add(new Store("店铺二",30,20,"烧烤"));
        stores.add(new Store("店铺三",10,20,"披萨"));
        stores.add(new Store("店铺四",15,30,"炸鸡"));
        stores.add(new Store("店铺五",45,20,"甜品"));
        stores.sort((Store o1,Store o2) -> Double.compare(o1.getAvePrice(),o2.getAvePrice()));
        for (Store store : stores){
            System.out.println(store.toString());
        }
    }
```

　　`stores.sort((Store o1,Store o2) -> Double.compare(o1.getAvePrice(),o2.getAvePrice()));` 就这么一行代码代替了我们之前那么多的代码，可以看到 Lambda 表达式可以使得我们的程序变得多么简洁吧。 `(Store o1,Store o2) -> Double.compare(o1.getAvePrice(),o2.getAvePrice())` 这行跟表达式一样的东西就这么传递给了 `sort()` 方法。这就是所谓的行为参数化。

## 什么是 Lambda 表达式

> Lambda 表达式可以理解为**简洁地表示可传递的匿名函数的一种方式**：他没有名称，但他有参数列表、函数主体、返回类型，还有一个可以抛出的异常列表。

　　Lambda 表达式由三个部分组成：参数列表、箭头、Lambda主体。具体的形式表现为 `(parameters) -> expression` 或 `(parameters) -> {statements;}`。

　　可以看到我们之前的 `(Store o1,Store o2) -> Double.compare(o1.getAvePrice(),o2.getAvePrice())` 是 `(parameters) -> expression` 的形式，`(Store o1,Store o2)` 是参数列表 ，`Double.compare(o1.getAvePrice(),o2.getAvePrice())` 是表达式。当 Lambda 主体中有多条语句的时候，就需要符合 `(parameters) -> {statements;}` 的形式了（**注意{}**）。

## 怎么使用 Lambda 表达式

　　在介绍 Lambda 表达式的使用之前，先提出一个概念：函数式接口。函数式接口定义：只定义一个抽象方法的接口。为什么需要这个概念呢，在 Java 中，"123" 是 `String` 类型； 12.3 是 `Double` 类型；true 是 `boolean` 类型...那 Lambda 这个实现行为参数化的新规范是什么类型呢，为了更符合我们的思维方式，Java8 给 Lambda 归为函数式接口类型。这样，我们能以我们更加理解的方式去学习 Lambda。

### 明确行为参数化

　　这其实就是一个去同存异的步骤，从上面的代码中我们可以发现，真正不同的地方就是选择店铺的策略上，所以我们需要做的就是将 `select()` 的行为参数化。

### 定义函数式接口来传递行为

　　知道我们需要参数化的行为之后，我们需要定义一个函数式接口，以便于我们之后传递 Lambda 表达式。

``` java
@FunctionalInterface
public interface StoreSelect {
    boolean select(Store store);
}
```

　　我们将对店铺的判断逻辑抽取出来，将其进行行为参数化。`@FunctionalInterface` 这个注解并不是必须的，但是为了将函数式接口与其他普通的接口区分开来，最好加上这个注解。

### 执行行为

　　定义完函数式接口之后，我们需要定义一个方法来接受 Lambda 表达式的函数，不然我们应该从哪边传入 Lambda 参数呢。

``` java
public static List<Store> selectStore(List<Store> stores,StoreSelect s){
        List<Store> result = new ArrayList<>();
        for(Store store:stores){
            if(s.select(store)){
                result.add(store);
            }
        }
        return result;
    }
```

　　可以看到我们调用了 `StoreSelect` 接口中的 `select()` 方法，但是我们并没有哪边实现过这个接口，这就需要我们在下一步中传入 Lambda 表达式来实现这个接口。

### 使用 Lambda

　　传入 Lambda 表达式，有种类似于用匿名方式实现接口：

``` java
    public static void main(String[] args){
        List<Store> stores = new ArrayList<Store>();
        stores.add(new Store("店铺一",20,10,"快餐"));
        stores.add(new Store("店铺二",30,20,"烧烤"));
        stores.add(new Store("店铺三",10,20,"披萨"));
        stores.add(new Store("店铺四",15,30,"炸鸡"));
        stores.add(new Store("店铺五",45,20,"甜品"));
        stores = selectStore(stores,(Store store) -> store.getAvePrice() <= 20.0);
        for (Store store : stores){
            System.out.println(store.toString());
        }
    }
```

　　使用了一个 `(Store store) -> store.getAvePrice() <= 20.0` Lambda 表达式，可以看到在传入 Lambda表达式的时候，并不能乱写，我们 Lambda 表达式中的参数列表必须与函数式接口的方法的参数列表一致，Lambda 主体中的返回结果必须与函数式接口中的方法的返回类型一致。Java 程序在编译的过程中会检查我们的代码是否满足这些要求，不然编译不会通过。

### 使用 Java8 为我们准备的函数式接口

　　如果每次都使用 Lambda 表达式，都需要完成上面的这些步骤，会觉得很麻烦，人都是有惰性的。所以 Java 已经为我们准备好了许多内置的函数式接口，当我们需要函数式接口的时候，只需要寻找一下有没有我们需要的，如果找不到再去自己定义。

　　[Java内置函数式接口](http://www.runoob.com/java/java8-functional-interfaces.html)

## 使用复合 Lambda 表达式

　　上面的过程中，我们每次使用 Lambda 表达式的时候传递进的只是一个逻辑，当需要的逻辑更加复杂，简单的一句话已经不能描述清楚的时候，应该怎么办，多定义几个函数式接口？并不需要，我们可以对 Lambda 表达式进行复合使用，就像 if 语句中出传入的条件一样，可以传入多个。

　　在介绍如何复合使用 Lambda 表达式之前，我们先来升华一下**函数式接口**这个定义，之前的定义是函数式接口中只有一个抽象方法。Java 中引入一个叫**默认方法**的机制，让我们可以在接口中实现方法。只需要在方法前加上 `default` 关键字就能为方法提供实现了。为什么要讲这个呢，因为默认方法为我们复合使用 Lambda 表达式提供了可能。

　　Java 中内置的函数式接口中的 `Comparator`、`Function`、 `Predicate` 都提供了允许进行表达式复合的方法，这些方法都是默认方法，已经为我们提供了实现。

>Function<T,R>    T -> R
>Predicate<T>     T -> boolean

### 比较器复合

　　我们可以通过使用静态方法 `Comparator.comparing`，根据提取用于比较的 Function 来返回一个 Comparator。`Comparator.comparing((Store store) -> store.getAvePrice())` 这样就可以获得一个根据店铺平均价格进行排序的 Comparator。如果我们想逆序排序,Compator 函数式接口中定义了一个 `reversed()` 函数，这可以返回一个逆序排序的 Comparator，类似于:

``` java
Comparator.comparing((Store store) -> store.getAvePrice()).reversed();
```

　　当两家店铺的价格一样的时候，我们希望根据送餐时间进行排序又该怎么办呢，Comparator 很贴心的又准备了一个 `thenComparing()` 方法，用法如下：

``` java
Comparator.comparing((Store store) -> store.getAvePrice())
.reversed()
.thenComparing((Store store) -> store.getAveDeliveryTime());
```

### 谓词复合

　　谓词接口 Predicate 提供了三个方法:`and`、`or`、`negate`，分别代表与、或、非。

　　首先，定义一个 Predicate 接口,选择出所有的快餐店：

``` java
Predicate<Store> typeStores = (Store store) -> "快餐".equals(store.getType());
```

　　那要找出所有的非快餐店呢：

``` java
Predicate<Store> notTypeStores = typeStores.negate();
```

　　找出非快餐店中,价格低于20的呢：

``` java
Predicate<Store> priceAndTypeStores = typeStores.negate()
.and((Store store) -> store.getAvePrice() < 20);
```

　　找出非快餐店中，价格低于20或者送餐时间低于30的呢：

``` java
Predicate<Store> priceAndTypeStores1 = typeStores.negate()
.and((Store store) -> store.getAvePrice() < 20)
.or((Store store) -> store.getAveDeliveryTime() < 30);
```


### 函数复合

　　函数复合和数学中的函数复合基本上概念是一致的，提供了 `andThen` 和 `compose` 两个方法:

``` java
Function<String,String> f1 = (String s) -> s.trim();
Function<String,String> f2 = (String s) -> s.substring(4);
```

　　`f1.andThen(f2)` 表示 `f2(f1(s))`，`f1.compose(f2)` 表示 `f1(f2(s))`。 

## 小结

　　啰啰嗦嗦的讲 Lambda 的基本概念讲完了，最后只想表明一个观点：实践出真知。光看懂 Lambda 表达式的概念，不在实际中使用，这一切都是空谈，Lambda 的出现就是为了简化我们的代码，将它用到平时的代码中去才能说是真正掌握了知识。