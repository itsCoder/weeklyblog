# 在 Android 中使用 Java8 的特性

根据 [Android 官网](https://developer.android.com/guide/platform/j8-jack.html?hl=zh-cn)的说明，在开发面向 Android N 的应用时，可以使用 Java8 语言功能。目前 Android 只支持一部分 Java8 的特性：

- [Lambda 表达式](https://docs.oracle.com/javase/tutorial/java/javaOO/lambdaexpressions.html)
- [方法引用](https://docs.oracle.com/javase/tutorial/java/javaOO/methodreferences.html)
- [默认和静态接口方法](https://docs.oracle.com/javase/tutorial/java/IandI/defaultmethods.html)
- [重复注解](https://docs.oracle.com/javase/tutorial/java/annotations/repeating.html)

其中，只有前两者可以兼容 API 23 以下的版本。


## Lambda 表达式

从一个实际例子来引入 lamdba 的使用。

有一组 Person 对象（具体实现不复杂，参考[这里](https://docs.oracle.com/javase/tutorial/java/javaOO/examples/Person.java)），需要通过年龄大小来过滤出满足要求的对象，然后对其进行输出操作，实现很简单，如下：

```java
public static void printPersonsOlderThan(List<Person> roster, int age) {
    for (Person p : roster) {
        if (p.getAge() >= age) {
            p.printPerson();
        }
    }
}
```

如果我的过滤条件变更了，就必须修改这个方法的代码，比如我现在根据年龄上下限进行过滤：

```java
public static void printPersonsWithinAgeRange(List<Person> roster, int low, int high) {
    for (Person p : roster) {
        if (low <= p.getAge() && p.getAge() <= high) {
            p.printPerson();
        }
    }
}
```

这样一来，过滤条件经常变更的话，需要频繁修改这个方法。根据面向对象的思想，封装变化，把经常改变的逻辑封装起来，有外部来决定。这里我把过滤条件封装到 `CheckPerson` 接口里，根据不同的过滤条件去实现这个接口即可。

```java
@FunctionalInterface
public interface CheckPerson {
    boolean test(Person p);
}

public static void printPersons(List<Person> roster, CheckPerson tester) {
    for (Person p : roster) {
        if (tester.test(p)) {
            p.printPerson();
        }
    }
}

// 实际使用

List<Person> roster = Person.createRoster(); // 制造一些数据

printPersons(roster, new CheckPerson() {
    @Override
    public boolean test(Person p) {
        return p.getGender() == Person.Sex.MALE
                && p.getAge() >= 18
                && p.getAge() <= 25;
    }
});
```

`CheckPerson` 接口是一个**函数式接口**(functional interface)，即**仅有一个抽象方法**的接口。 因此实现这个接口的时候可以忽略掉方法名称，使用 Lambda 表达式来替代匿名类。

```java
printPersons(roster,
        p -> p.getGender() == Person.Sex.MALE
                && p.getAge() >= 18
                && p.getAge() <= 25
);
```

其实在 Java8 的包中，已经内置了一些标准的函数式接口。比如 `CheckPerson` 接收一个对象，然后输出一个 boolean 值。可以使用 `java.util.function.Predicate<T>` 来替代，它相当于 RxJava 中的 `Func1<T, Boolean>`，接收一个对象，返回布尔值。

```java
public static void printPersonsWithPredicate(List<Person> roster, Predicate<Person> tester) {
    for (Person p : roster) {
        if (tester.test(p)) {
            p.printPerson();
        }
    }
}

// 使用起来并没有什么差别
printPersonsWithPredicate(roster,
        p -> p.getGender() == Person.Sex.MALE
                && p.getAge() >= 18
                && p.getAge() <= 25
);
```

这是，我并不满足于仅仅把过滤条件封装起来，还想把过滤之后对 Person 对象的操作也封装起来，便于修改。可以用另外一个标准的函数式接口 `java.util.function.Consumer<T>`，它相当于 RxJava 中的 `Action1<T>`，接收一个对象，返回 void。

```java
public static void processPersons(List<Person> roster, Predicate<Person> tester, Consumer<Person> block) {
    for (Person person : roster) {
        if (tester.test(person)) {
            block.accept(person);
        }
    }
}

// 具体使用
processPersons(roster,
        p -> p.getGender() == Person.Sex.MALE
                && p.getAge() >= 18
                && p.getAge() <= 25,
        p -> p.printPerson()
);

```

如果我的处理过程中有数据转换的过程，可以用 `java.util.function.Function<T, F>` 将其封装起来，这个接口相当于 RxJava 中的 `Func1<T, F>`，接收一个类型的对象，返回另外个类型的对象，达到数据转换的目的。比如例子中，把 Person 转换成 String 对象。

```java
public static void processPersonsWithFunction(
        List<Person> roster,
        Predicate<Person> tester,
        Function<Person, String> mapper,
        Consumer<String> block) {
    for (Person person : roster) {
        if (tester.test(person)) {
            String data = mapper.apply(person);
            block.accept(data);
        }
    }
}

// 实际使用
processPersonsWithFunction(roster,
        p -> p.getGender() == Person.Sex.MALE
                && p.getAge() >= 18
                && p.getAge() <= 25,
        p -> p.getEmailAddress(), // 获取 Person 对象的 email 字符串
        email -> System.out.println(email)
);
```

最后可以把数据源也封装成一个 `java.lang.Iterable<T>` 对象。

```java
public static <X, Y> void processElements(
        Iterable<X> source,
        Predicate<X> tester,
        Function<X, Y> mapper,
        Consumer<Y> block) {
    for (X x : source) {
        if (tester.test(x)) {
            Y data = mapper.apply(x);
            block.accept(data);
        }
    }
}

// 实际使用
Iterable<Person> source = roster;
Predicate<Person> tester = p -> p.getGender() == Person.Sex.MALE
        && p.getAge() >= 18
        && p.getAge() <= 25;
Function<Person, String> mapper = p -> p.getEmailAddress();
Consumer<String> block = email -> System.out.println(email);
processElements(roster, tester, mapper, block);
```

在 Java8 中也可以把 Collections 对象快速转换成 Stream 来使用方便的操作符。

```java
roster.stream()                                     // 获取数据流
    .filter(                                        // 根据 Predicate 过滤数据
            p -> p.getGender() == Person.Sex.MALE
                    && p.getAge() >= 18
                    && p.getAge() <= 25)
    .map(p -> p.getEmailAddress())                  // 根据 Function 转换数据
    .forEach(email -> System.out.println(email));   // 对数据执行操作(消费数据)


```



#### Lambda 写法

基本写法：

```java
p -> p.getGender() == Person.Sex.MALE 
    && p.getAge() >= 18
    && p.getAge() <= 25

// 没有参数
() -> System.out.println("Hello lambda")

// 参数多于 1 个
(x, y) -> x + y
```

参数列表，如果只有一个参数，可以省略掉括号，其他情况需要写上一对括号。

需要注意的是， 箭头 `->` 后面必须是一个单独的表达式（expression）或者是一个语句块（statement block）。

```java
// 表达式
p -> p.getGender() == Person.Sex.MALE 
    && p.getAge() >= 18
    && p.getAge() <= 25
  
// 代码块
p -> {
    return p.getGender() == Person.Sex.MALE
        && p.getAge() >= 18
        && p.getAge() <= 25;
}
```



## 方法引用

当你使用一个 lambda 表达式的时候，如果它仅仅是调用了一下已有的方法，并没有做其他任何操作，就可以把它转换成**方法引用**。方法引用有四种写法，下面一一介绍。

```java
// 先制造一些数据， 供后面的例子使用
List<Person> roster = Person.createRoster();
Person[] rosterAsArray = roster.toArray(new Person[roster.size()]);
```

#### 引用静态方法

首先在 Person 类中有一个静态方法，通过年龄比较大小：

```java
// Person.java
public static int compareByAge(Person a, Person b) {
    return a.birthday.compareTo(b.birthday);
}
```

```java
//MethodReferencesTest.java
// 原来的写法，传入匿名类
Arrays.sort(rosterAsArray, new Comparator<Person>() {
    @Override
    public int compare(Person o1, Person o2) {
        return Person.compareByAge(o1, o2);
    }
});

// 写成 lambda 形式
Arrays.sort(rosterAsArray, (a, b) -> Person.compareByAge(a, b));
// 转换成方法引用 =>
Arrays.sort(rosterAsArray, Person::compareByAge);
```



#### 引用具体实例的方法

```java
class ComparisonProvider {
    public int compareByName(Person a, Person b) {
        return a.getName().compareTo(b.getName());
    }

    public int compareByAge(Person a, Person b) {
        return a.getBirthday().compareTo(b.getBirthday());
    }
}

ComparisonProvider comparisonProvider = new ComparisonProvider();

// lambda 形式
Arrays.sort(rosterAsArray, (p1, p2) -> comparisonProvider.compareByAge(p1, p2));
// 转换成方法引用 =>
Arrays.sort(rosterAsArray, comparisonProvider::compareByName);
```



#### 引用特定类型的对象的实例方法

Person 实现一下 `Comparable<T>` 接口，会有一个 `compareTo(Person)` 方法。

```java
public class Person implements Comparable<Person> {

    @Override
    public int compareTo(Person o) {
        return Person.compareByAge(this, o); // 复用之前静态方法的逻辑
    }
  
    // 其他忽略
}
```

```java
// lambda 形式
Arrays.sort(rosterAsArray, (p1, p2) -> p1.compareTo(p2));
// 转换成方法引用 =>
Arrays.sort(rosterAsArray, Person::compareTo);
```

#### 引用构造方法

有一个 `transferElements` 方法，将 `SOURCE` 类型的集合转换成 `DEST` 类型的集合。

```java
public static <T, SOURCE extends Collection<T>, DEST extends Collection<T>>
DEST transferElements(
        SOURCE sourceCollection,
        Supplier<DEST> collectionFactory) {

    DEST result = collectionFactory.get();

    for (T t : sourceCollection) {
        result.add(t);
    }
    return result;
}
```

 其中 `java.util.function.Supplier<T>` 也是标准的函数式接口，它有一个 `get()` 方法来获取所提供的对象。

```java
// 匿名类形式
Set<Person> rosterSet = transferElements(roster, new Supplier<Set<Person>>() {
    @Override
    public Set<Person> get() {
        return new HashSet<Person>();
    }
});
// lambda 形式
Set<Person> rosterSet = transferElements(roster, () -> new HashSet<>());
// 转换成方法引用 =>
Set<Person> rosterSet = transferElements(roster, HashSet::new);
```

lambda 表达式中直接 new 了一个 HashSet，相当于调用了 HashSet 的构造方法，故可以写成 `HashSet::new` 方法引用的形式。



## 静态和默认接口方法

在 Java8 之前，接口不允许有默认实现，如果接口的两个实现类有同样的实现逻辑，就得写重复代码了。现在接口可以通过关键字 `default` 实现默认方法，另外接口还可以实现静态方法。

```java
public interface SampleInterface {
    default int test() {
        System.out.println("SampleInterface default impl");
        return staticTest() + 666;
    }

    static int staticTest() {
        return 100;
    }
}
```

```java
public class SampleTest {
    public static void main(String[] args) {
        int test = new SampleInterfaceImpl1().test();
        System.out.println(test);

        int test2 = new SampleInterfaceImpl2().test();
        System.out.println(test2);
    }

    static class SampleInterfaceImpl1 implements SampleInterface {
        @Override
        public int test() {
            System.out.println("SampleInterfaceImpl1 override");
            return SampleInterface.staticTest() + 233;
        }
    }

    static class SampleInterfaceImpl2 implements SampleInterface {
        // 不需要实现 test 方法
    }
}
```

最后输出结果：

```
SampleInterfaceImpl1 override
333
SampleInterface default impl
766
```

使用接口的默认方法可以减少代码重复，静态方法也可以方便地封装一些通用逻辑。

## 重复注解

重复注解就是允许在同一申明类型（类，属性，或方法）多次使用同一个注解。

```java
@Repeatable(Schedules.class) // 指定存储 Schedule 的注解
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface Schedule {
    String dayOfWeek() default "Mon";

    String dayOfMonth() default "first";

    int hour() default 12;
}

@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface Schedules {
    Schedule[] value(); // 存储 Schedule
}
```

使用时可以通过 `AnnotatedElement.getAnnotationsByType()` 方法来获取到注解，然后进行相应的处理。

```java
public class AnnotationTest {

    @Schedule(dayOfMonth = "last")
    @Schedule(dayOfWeek = "Fri", hour = 9)
    public void doSomethingWork() {
        System.out.println("doSomethingWork");
        try {
            Method method = AnnotationTest.class.getMethod("doSomethingWork");
            Schedule[] schedules = method.getAnnotationsByType(Schedule.class);
            for (Schedule schedule : schedules) {
                System.out.println("Schedule: " + schedule.dayOfWeek() + ", " + schedule.dayOfMonth() + ", " + schedule.hour());
            }
        } catch (NoSuchMethodException e) {
            e.printStackTrace();
        }
    }

    public static void main(String[] args) {
        new AnnotationTest().doSomethingWork();
    }
}
```

输出如下：

```
doSomethingWork
Schedule: Mon, last, 12
Schedule: Fri, first, 9
```

使用就是这么简单~

## 在 Android 中使用这些特性

在主 module (app) 的 `build.gradle` 里配置，开启 jack 编译器，使用 Java8 进行编译。
如果要体验接口的默认方法等特性，minSdkVersion 需要指定为 24 (Android N)。

```groovy
android {
    compileSdkVersion 24
    buildToolsVersion "24.0.0"

    defaultConfig {
        applicationId "me.brucezz.sharedelementdemo"
        minSdkVersion 14
        targetSdkVersion 24
        versionCode 1
        versionName "1.0"

        // 开启 jack 编译
        jackOptions {
            enabled true
        }

    }

    compileOptions {
        // 指定用 Java8 编译
        sourceCompatibility JavaVersion.VERSION_1_8
        targetCompatibility JavaVersion.VERSION_1_8
    }
}
```


## Reference

- 文中代码大部分来自于 [Oracle 官方文档教程](https://docs.oracle.com/javase/tutorial/)
- [在 Android N 预览版中使用 Java 8 的新特性](http://blog.zhaiyifan.cn/2016/04/20/trans-java-8-in-android-n-preview/)