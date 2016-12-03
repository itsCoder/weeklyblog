---
title: Java注解(Annotation)简单介绍
date: 2016-09-06 18:46:55
tags:
- 注解
categories:
- Java
- 基础知识
---

　　在进行 Java 开发的过程中，我们一定会看到像 `@Override` 或 `@SuppressWarnings` 这种既不像是代码又不像是注释的东西。老师在上课过程中也没提到过(如果没记错。但是这东西从第一次看到它，我就接受了它是合理存在的设定，从来没想过为什么要出现，有什么作用。后来才知道这叫注解，作用也不只是看上去的装饰作用。

## 注解介绍
>注解（也被称为元数据）为我们在代码中添加信息提供了一种形式化的方法，使我们可以在稍后某个时刻非常方便地使用这些数据。

　　Java 内置了三种注解供我们使用：

- @Override，表示当前的方法定义将覆盖超类中的方法。
- @Deprecated，表示被标记的这个方法或类不再建议使用，如果程序中仍然使用了这个方法或者类，编译器将会发出警告信息。
- @SuppressWarnings，表示忽略编译器发出的警告。

## 自定义一个注解

　　除了使用 Java 自带的三种内置注解，我们也可以根据自己的实际需要来自定义注解，怎么定义一个注解呢。定义一个注解和定义一个接口，定义一个类是一样的。先上一个例子：

``` java
	import java.lang.annotation.ElementType;
	import java.lang.annotation.Retention;
	import java.lang.annotation.RetentionPolicy;
	import java.lang.annotation.Target;
	
	@Target(ElementType.METHOD)
	@Retention(RetentionPolicy.RUNTIME)
	public @interface Example {
	    public int id();
	    public String name() default "there is something";
	}
```

　　这样子就定义了一个最简单的注解，通过 `@Example` 来使用这个注解。接下来，我们一点一点的看定义一个注解需要哪些部分。

### 命名注解

　　首先，`public @interface Example` 这是定义一个注解，注解的名称为 `Example` ，`@interface` 是定义注解的关键字，和定义一个类或者一个接口的时候使用的 `class`, `interface` 关键字是一样的。

### 注解元素

　　接着看 `public int id();` 和 `public String name()` 这是在定义注解中的元素，这和定义类的时候定义类中的属性是一个道理，用于定义这个注解中拥有哪些元素。定义注解的时候可以使用的元素类型包括：

- 所有基本类型 （int,float,boolean 等）
- String
- Class
- enum
- Annotation

　　所以注解中使用的元素类型基本上没有限定，都是可以使用的，最后一个 Annotation 说明注解也可以作为元素的类型，注解之间可以进行嵌套。

　　读者一定注意到例子中的注解元素一个 `id` 后面没有跟 `default` 内容，一个注解元素 `name` 后面跟了 `default` 内容。`default` 用于指定这个元素的默认值，如果没有指定默认值，表示这个元素在使用的时候是一定需要被赋值的，否则是错误的。在设置默认值上这一点注解和类和接口有一点不同，类和接口中通过 = 赋值来设定默认值，对于没有默认值的，在使用过程中会自动给予一个默认值。

### 元注解

　　接下来是看 `@Target(ElementType.METHOD)` , `@Target` 注解是 Java 提供的四种用于定义注解的元注解之一。`@Target` 表示该注解可以使用在什么地方，`ElementType` 是一个 `enum` 类型的数据，其中包括：

- CONSTRUCTOR：说明这个注解使用于构造器声明
- FIELD: 说明这个注解用于域声明（包括 enum ）
- LOCAL_VARIABLE： 说明这个注解用于局部变量声明
- METHOD: 说明这个注解用于方法声明
- PACKAGE: 说明这个注解用于包声明
- PARAMETER: 说明这个注解用于参数声明
- TYPE: 说明这个注解用于类、接口或 enum 声明

　　再看 `@Retention(RetentionPolicy.RUNTIME)` 这是指定需要在什么级别保存该注解信息。`RetentionPolicy` 包括：

- SOURCE 注解只在源文件中被保留，在编译的时候会被丢弃
- CLASS 注解在 class 文件中可用，但会被 VM 丢弃
- RUNTIME 注解在 VM 运行期间也存在

　　另外还有两个元注解用于定义注解，`@Documented` 表示将此注解包含在 Javadoc 中，`Inherited` 表示允许子类继承父类中的注解。

## 编写一个注解解释器

　　上面只是定义了一个注解，现在我们来看一下注解应该怎么使用：

``` java
	public class ExampleTest {
	
	    @Example(id = 1)
	    public void test1(){
	        System.out.println("this is test1()");
	    }
	
	    @Example(id = 2, name = "test2")
	    public void test2(){
	        System.out.println("this is test2()");
	    }
	
	    @Example(id = 3, name = "example name")
	    public void test3(){
	        System.out.println("this is test3()");
	    }
	
	    public void test4(){
	        System.out.println("this is test4()");
	    }
	}
```

　　到这一步，我们好像还是看不出来注解究竟有什么用，好像就是跟注释一样啊。别急，要让注解真正发挥作用，我们需要写一个注解处理器，注解处理器才是真正让注解起作用的部分。

　　编写注解处理器的时候需要用到一些关于反射机制方面的内容，不过是属于比较简单的内容。

``` java
	public class ExampleHandler {
	    public static void examplePrint(Class<?> cl){
	        for(Method m : cl.getDeclaredMethods()){
	            Example e = m.getAnnotation(Example.class);
	            if(e != null){
	                System.out.println("This example id: "+e.id()+" , name: "+e.name());
	            }
	        }
	    }
	
	    public static void main(String args[]){
	        ExampleHandler.examplePrint(ExampleTest.class);
	    }
	}
```

　　可以看到主要是用了反射中的 `getDeclaredMethods()` 和 `getAnnotation` 方法， `examplePrint()` 方法的作用就是对类中的所有方法进行遍历，看是否有 `@Example` 这个注解，如果存在这个注解，那么输出这个注解的 `id` 和 `name` 值。

　　输出结果为：![](http://oc4wmeyj8.bkt.clouddn.com/ExampleHandler%E8%BE%93%E5%87%BA%E7%BB%93%E6%9E%9C.PNG)

　　可以看到有 `@Example` 注解的方法都输出了相应的内容，没有对 `name` 进行赋值的注解也输出了默认的内容，没有 `@Example` 注解的部分并没有相应的输出。

　　以上，就是关于 Java 注解入门的一些基础知识，深入的话还有好多内容可以挖。
