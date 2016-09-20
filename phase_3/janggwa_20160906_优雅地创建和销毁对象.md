## 创建和销毁对象概述

- **何时以及如何创建对象**
- **何时以及如何避免创建对象**
- **如何确保对象适时地销毁**
- **如何管理对象销毁之前必须进行的各种清理动作**

### 一.考虑用静态工厂方法代替构造器

构造器是创建一个对象实例最基本也最通用的方法，大部分开发者在使用某个 class 的时候，首先需要考虑的就是如何构造和初始化一个对象示例，而构造的方式首先考虑到的就是通过构造函数来完成，因此在看 javadoc 中的文档时首先关注的函数也是构造器。然而在有些时候构造器并非我们唯一的选择，我们可以通过静态类工厂的方式来创建 class 的实例，如

```java
public static Boolean valueOf(boolean b) {
    return b?Boolean.TRUE:Boolean.FALSE;
}
```

相比于构造器，静态工厂方法的**优势**：

**1.有意义的名称**

构造方法本身没有名称，不能确切的返回描述返回的对象，具有适当名称的静态工厂方法更容易被使用，产生的代码也更容易被阅读。

**2.不必每次调用的时候创建一个新的对象**

构造方法的每次调用都会重新创建一个新的实例，而静态工厂方法可以预先构建好实例/实例缓存，进行重复利用，从而避免不必要的重复对象的创建。

**3.能返回原返回类型的任何子类型的对象**

构造方法只能返回类本身，而静态方法可以返回它的子类，利用静态使得方法更加灵活。

**4.创建参数化实例的时候，它们使代码变得更加简洁**

```java
Map<String,List<String>> stringListMap = new HashMap<>();
```

由于 Java 在构造函数的调用中无法进行类型的推演，因此也就无法通过构造器的参数类型来实例化指定类型参数的实例化对象。然而通过静态工厂方法则可以利用参数类型推演的优势，避免了类型参数在一次声明中被多次重写所带来的烦忧，见如下代码：

```java
public static <K,V>  HashMap<K,V> newInstance() {
    return new HashMap<K,V>();
}

//调用
Map<String,List<String>> m = HashMap.newInstance();
```

### 二.遇到多个构造器参数时要考虑用构建器

当实例化一个类时，特别是有很多可选的参数，如果我们考虑使用写很多不同参数的构造方法，就会使得可读性变得很差。
这个时候推荐 Builder 模式来创建这个带有很多可选参数的实例对象。

```java
class NutritionFacts {
    private final int servingSize;
    private final int servings;
    private final int calories;
    private final int fat;
    private final int sodium;
    private final int carbohydrate;
    public static class Builder {
        //对象的必选参数
        private final int servingSize;
        private final int servings;
        //对象的可选参数的缺省值初始化
        private int calories = 0;
        private int fat = 0;
        private int carbohydrate = 0;
        private int sodium = 0;
        //只用少数的必选参数作为构造器的函数参数
        public Builder(int servingSize,int servings) {
            this.servingSize = servingSize;
            this.servings = servings;
        }
        public Builder calories(int val) {
            calories = val;
            return this;
        }
      public Builder fat(int val) {
          fat = val;
          return this;
      }
      public Builder carbohydrate(int val) {
          carbohydrate = val;
          return this;
      }
      public Builder sodium(int val) {
          sodium = val;
          return this;
      }
      public NutritionFacts build() {
          return new NutritionFacts(this);
      }
  }
      private NutritionFacts(Builder builder) {
          servings = builder.servings;
          calories = builder.calories;
          fat = builder.fat;
          sodium = builder.sodium;
          carbohydrate = builder.carbohydrate;
      }
}

//使用方式
public static void main(String[] args) {
    NutritionFacts cocaCola = new NutritionFacts.Builder(240, 8).calories(100)
		.sodium(35).carbohydrate(27).build();
    System.out.println(cocaCola);
}
```

### 三.用私有构造器或枚举类型强化 Singleton 属性

在 Java1.5 之前，实现 Singleton 由两种方法。这两种方法都要把构造器保持为私有的，并导出公有的静态成员，以便允许客户端能够访问该类的唯一实例。

**1.将构造函数私有化，直接通过静态公有的 final 域字段获取单实例对象：**

```java
public class Elvis {
    public static final Elvis INSTANCE = new Elvis();
    private Elivs() { ... }  
}       
 
```

**2.通过公有域成员的方式返回单实例对象**：

```java
public class Elvis {
    public static final Elvis INSTANCE = new Elvis();
    private Elivs() { ... }
    public static Elvis getInstance() { 
        return INSTANCE; 
    }
    public void leaveTheBuilding() { ... }
}
```

Java1.5 起，实现 Singleton 还有第三种方法。只需编写一个包含单个元素的枚举类型。**单类型的枚举类型是实现单利模式的最佳方法**。

```java
public enum Elvis {
    INSTANCE;
    public void leaveTheBuilding() { ... }
}
```

### 四.通过私有构造器强化不可实例化的能力

不可实例化是指当前的类只包含 静态方法和静态域的类。在 Java 中，只有当类不包含显示的构造器时，编译器才会生成缺省的构造器，因为只要让类包含私有的构造器，它就不能被实例化。

```java
public class BitmapUtils {
    private BitmapUtils() {
        throw new AssertionError();
    }
}
```

### 五.避免创建不必要的对象

```java
String s = new String("stringette");
String s = "stringette";
```

比较这两行代码，上面的语句每次执行的时候都会创建一个新的 String 实例，但是这些创建对象的动作全都是不必要的。而下面的语句只用了一个 String 实例，而不是每次执行的时候都创建一个新的实例。由于 String 被实现为不可变对象，JVM 底层将其实现为常量池，所有值等于 "stringette" 的对象实例共享同一对象地址，而且还可以保证，对于所有在同一 JVM 中运行的代码，只要他们包含相同的字符串字面常量，该对象就会被重用。

**除了重用不可变的对象之外，也可以重用那些已知不会被修改的可变对象。**

```java
public class Person {
    private final Date birthDate;
    //判断该婴儿是否是在生育高峰期出生的。
    public boolean isBabyBoomer {
        Calender c = Calendar.getInstance(TimeZone.getTimeZone("GMT"));
        c.set(1946,Calendar.JANUARY,1,0,0,0);
        Date dstart = c.getTime();
        c.set(1965,Calendar.JANUARY,1,0,0,0);
        Date dend = c.getTime();
        return birthDate.compareTo(dstart) >= 0 && birthDate.compareTo(dend) < 0;
    }
}
    
public class Person {
    private static final Date BOOM_START;
    private static final Date BOOM_END;
        
    static {
        Calender c = Calendar.getInstance(TimeZone.getTimeZone("GMT"));
        c.set(1946,Calendar.JANUARY,1,0,0,0);
        BOOM_START = c.getTime();
        c.set(1965,Calendar.JANUARY,1,0,0,0);
        BOOM_END = c.getTime();
    }
    public boolean isBabyBoomer() {
        return birthDate.compareTo(BOOM_START) >= 0 && birthDate.compareTo(BOOM_END) < 0;
    }
}
```

改进后的 Person 类只是在初始化的时候创建 Calender、TimeZone 和 Date 实例一次，而不是在每次调用 isBabyBoomer 方法时都创建一次它们。如果该方法会被频繁调用，效率的提升将会极为显著。

### 六.消除过期对象的引用

内存泄露：如果一个栈先是增长，然后收缩，那么，从栈中弹出来的对象是不会被当做垃圾回收的，即使使用栈的程序不再引用这些对象，它们也不会被回收。这是因为，栈内部维护着对这些对象的过期引用。由于过期引用的存在, GC 并不会去回收它们,所以需要我们手动清空这些引用。比如 ：

```java
public Object pop() {
    if(size==0) throw new EmptyStackException();
    Object result = elements[--size];
    elements[size] = null; //Eliminate obsolete reference
    return result;
}
```

### 七.避免使用终结方法

**终结方法的好处**

- 当对象的所有者忘记调用前面段落中建议的显式终止方法时，终结方法可以充当“安全网”
- 终止非关键的本地资源

**终结方法的缺点**

- 终结方法(finalizer)通常是不可预测的，也是很危险的，一般情况下是不必要的。使用终结方法会导致行为不稳定、降低性能，以及可移植性问题。
- 终结方法不能保证会被及时地执行
- 不应该依赖终结方法来更新重要的持久状态
- 使用终结方法有严重的性能损失

### 总结

以上是我对阅读了《 Effective Java 》第一篇之后的一些总结，希望可以帮助大家写出更优雅的代码。





