---
title: 讲讲 Java8 中的流
date: 2016-10-21 20:46:25
tags:
- Java8
- 流
categories:
- Java
- 基础知识
---

　　学习 Java 的过程中，我们已经接触到了各种各样的流：输入流、输出流、文件输入流、文件输出流等等，但是在 Java8 中又提出了一个流的概念，这篇文章就来讲讲 Java8 中的流，看看它与以前了解的流究竟有什么关系。

## 1. 什么是流

　　在 《Java8实战》 这本书中，流的定义是
>从支持数据处理操作的源生成的元素序列

 　　按照我的理解，流就是一种在 Java8 中新提出的数据集合。这个新的数据集合提供一系列的 API ，可以方便我们对集合进行数据的筛选、转换等操作。

## 2. 流之初体验

>阅读应该是人生活中就像阳光、空气和水一样自然而然的存在。

　　我就拿书作为例子来讲讲 Java8 中的流。首先，我们先来看看流到底是怎么样的，以及流的作用。定义一个 Book 的实体类：

``` java
public class Book {
	//书名
    private String name;
	//作者
    private String author;
	//价格
    private double price;
	//出版社
    private String publish;
	//出版年份
    private int publishYear;

    public Book(String name,String author,double price,String publish,int publishYear){
        this.name = name;
        this.author = author;
        this.price = price;
        this.publish = publish;
        this.publishYear = publishYear;
    }
	/*省略set和get方法*/
}
```

　　假使我们现在有很多本书，给我们的要求是找出这些书中所有在 2000 年以前出版的书的名字。这个任务看起来一点都不难啊，老司机一下子就能写好：

``` java
List<Book> books = Arrays.asList(
        new Book("书名一", "作者甲", 19.8, "一号出版社", 1995),
        new Book("书名二", "作者乙", 99.8, "一号出版社", 2016),
        new Book("书名三", "作者乙", 9.9, "一号出版社", 1994),
        new Book("书名四", "作者甲", 21.3, "一号出版社", 1998),
        new Book("书名五", "作者乙", 30.2, "一号出版社", 1999),
        new Book("书名六", "作者丙", 15.7, "二号出版社", 2000),
        new Book("书名七", "作者甲", 49.0, "二号出版社", 2007),
        new Book("书名八", "作者丁", 72.0, "二号出版社", 2012),
        new Book("书名九", "作者丙", 98.0, "二号出版社", 2015),
        new Book("书名十", "作者丁", 100.0, "二号出版社", 2014)
);
List<String> bookNames = new ArrayList<>();
for (Book book : books) {
    if (book.getPublishYear() < 2000) {
        bookNames.add(book.getName());
    }
}
```

　　但是使用流的写法可以更简便，一句话就能搞定：

``` java
List<String> bookNamesNew = books.stream()
                .filter(b -> b.getPublishYear() < 2000)
                .map(Book::getName)
                .collect(toList());
```

　　这里用到了 Lambda 写法，不会的可以参考之前的文章。这个任务比较简单，所以并不能很好说明流的好处，但是这只是为了简单的介绍一下流可以做什么，现在大家应该已经有一个初步的印象了。等了解完流其他的功能之后，更加能体会到流的好用之处。

## 3. 流之再体验

　　使用流来处理问题的时候，一般有三个步骤：


1. 获取流，像上面的 `books.stream()` 返回的是一个 `Stream<T>` 的流对象
2. 执行**中间操作**对数据进行处理，上面的 `fliter()` 和 `map()` 都是对流中的数据进行处理操作
3. 执行**终端操作**获得预期的返回结果，`collect(toList())` 根据我们的预期返回 `List` 类型的数据 	

　　Java8 中的流有一个很重要的特性就是**并行处理**，对于数据的处理不是一个步骤一个步骤地处理，而是像流水线一样处理，这个并行处理并不需要我们处理，Java 内部就会帮我们实现。只需要更改一下获取流对象的方法就行，将`stream()` 方法改为 `parallelStream()` 方法即可。上面代码中的 `filter()` 、 `map()` 和 `collect()` 这几个操作其实是并行处理的。这就省去了我们代码优化啊、实现多线程啊一系列工作。

### 3.1 中间操作

　　**中间操作**是对流对象进行处理的操作，中间操作会返回一个流对象，这样多个中间操作可以连起来，形成流水线。下面介绍一下主要的一些中间操作:

#### 3.1.1 筛选操作

**源码定义**：

``` java
Stream<T> filter(Predicate<? super T> predicate);
```

通过传入一个 `Predicate` 类型的 Lambda 表达式，对流对象中的数据进行筛选。

**写法示例**：
``` java
List<Book> books1 = books.stream()
				.filter(b -> b.getPublishYear() < 2000)
				.collect(toList());
```

#### 3.1.2 去重操作

**源码定义**：

``` java
Stream<T> distinct();
```

对流对象中的数据进行去重，其判断对象是否一样调用的是 `equals()` 方法。

**写法示例**:

``` java
List<Book> books2 = books.stream()
        .filter(b -> b.getPublishYear() < 2000)
        .distinct()
        .collect(toList());
```

#### 3.1.3 限制返回个数

**源码定义**：

``` java
Stream<T> limit(long maxSize);
```

对流操作返回的结果中的个数进行限制，按顺序返回前 `maxSize` 个元素，若元素个数不足 `maxSize` 则返回所有元素。

**写法示例**:

``` java
List<Book> books3 = books.stream()
        .filter(b -> b.getPublishYear() < 2000)
        .limit(3)
        .collect(toList());
```

#### 3.1.4 跳过操作

**源码定义**:

``` java
Stream<T> skip(long n);
```

将流对象中前 n 个元素去除，并返回一个流对象，如果元素个数不足 n ，则返回一个数据为空的一个流对象。

**写法示例**:

``` java
List<Book> books4 = books.stream()
        .filter(b -> b.getPublishYear() < 2000)
        .skip(3)
        .collect(toList());
```

#### 3.1.5 映射操作

**源码定义**:

``` java
<R> Stream<R> map(Function<? super T, ? extends R> mapper);
```

通过传入一个 `Function` 类型的函数式接口，完成流对象中元素的类型转换功能。

**写法示例**:

``` java
List<String> books5 = books.stream()
        .map(Book::getName)
        .collect(toList());
```

#### 3.1.6 排序操作

**源码定义**：

``` java
Stream<T> sorted();
Stream<T> sorted(Comparator<? super T> comparator);
```

有默认排序和自定义排序，自定义排序的话，传入一个 `Comparator` 类型的函数式接口，按照自己预期的情况进行排序，返回排序之后的流对象。

**写法示例**:

``` java
List<Book> books6 = books.stream()
                .sorted()
                .collect(toList());
```

### 3.2 终端操作

　　**终端操作**将流对象转换成我们所期望的结果，终端操作是流水线上的最后一个操作，一般终端操作都不会像中间操作一样返回一个流对象。因为流对象像迭代器一样只能遍历一次，所以当流对象遇到终端操作之后，整个流水线就结束了。

#### 3.2.1 返回集合

**源码定义**：

``` java
<R> R collect(Supplier<R> supplier,
                  BiConsumer<R, ? super T> accumulator,
                  BiConsumer<R, R> combiner);
```

通过传入参数来确定返回的结果类型，像之前一直用的 `collect(toList())` 就是说将对象流转换成 `List` 类型的集合，`toList()` 是在 `java.util.stream.Collectors` 中定义好的， Java8 已经为我们准备好了一些常用的转换规则，这可以在 `java.util.stream.Collectors` 中找到。

**写法示例**：

``` java
List<Book> books1 = books.stream()
				.filter(b -> b.getPublishYear() < 2000)
				.collect(toList());
```

#### 3.2.2 查找是否至少存在一个

**源码定义**:

``` java
boolean anyMatch(Predicate<? super T> predicate);
```

判断流对象中是否存在满足 `Predicate` 类型的元素，如果存在则返回 `true`，否则返回 `false`。

**写法示例**:

``` java
boolean books7 = books.stream()
                .anyMatch(b -> b.getPublishYear() < 2000);
```

#### 3.2.3 查找第一个元素

**源码定义**：

``` java
Optional<T> findFirst();
``` 

返回流对象中第一个元素。

**写法示例**：

``` java
Book books8 = books.stream()
                .filter(b -> b.getPublishYear() < 2000)
                .findFirst().get();
```

#### 3.2.4 返回元素个数

**源码定义**：

``` java
long count();
```

返回流对象中元素的个数。

**写法示例**：

``` java
long books9 = books.stream()
                .filter(b -> b.getPublishYear() < 2000)
                .count();
```

## 4. 小结

　　其实这篇文章就只是简单的介绍了一下 Java8 中流的内容，本来想把 Java8 中定义的关于流的 API都讲一下的，但是发现这个量有点大，所以就挑了一些比较有代表性的**中间操作**和**终端操作**讲一下，更多的内容还是得靠自己发掘了。