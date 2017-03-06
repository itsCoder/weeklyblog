在我们日常的工作中经常需要在应用程序中保持一个唯一的实例，如：IO处理，数据库操作等，由于这些对象都要占用重要的系统资源，所以我们必须限制这些实例的创建或始终使用一个公用的实例，这就是我们今天要介绍的——单例模式（Singleton）。

## 定义
>单例模式，又称单件模式或者单子模式，指的是一个类只有一个实例，并且提供一个全局访问点。

## 实现思路
在单例的类中设置一个 private 静态变量instance，instance 类型为当前类，用来持有单例唯一的实例。
将（无参数）构造器设置为 private，避免外部使用 new 构造多个实例。
提供一个 public 的静态方法，如 getInstance，用来返回该类的唯一实例 instance。

## 类图
![](https://rawgit.com/jasonim/design-patterns/develop/zh/creator-mode/singleton/image/singleton.png)

## 几种实现方式
由于使用场景不同，出现不同写法和模式，它们分别:
+ 懒汉式
+ 恶汉式
+ 双重校验锁
+ 枚举
+ 静态内部类

由于枚举使用场景场景较少， 下面就不介绍，感兴趣的可以自行解决。

### 饿汉式
饿汉式指的是单例的实例在类装载时进行创建。由于是在类装时候创建， 所以能够保证线程安全。如果单例类的构造方法中没有包含过多的操作处理，饿汉式其实是可以接受的。
```
public class SingleInstance {
  private final static SingleInstance instance = new SingleInstance();

  public static SingleInstance getInstance() {
      return instance;
  }
}
```

不足：
+ 如果构造方法中存在过多的处理，会导致加载这个类时比较慢，可能引起性能问题。
+ 如果使用饿汉式的话，只进行了类的装载，并没有实质的调用，会造成资源的浪费。

### 懒汉式
懒汉式指的是单例实例在第一次使用时进行创建。这种情况下避免了上面饿汉式可能遇到的问题。
```
public class SingleInstance {
  private static SingleInstance instance;
  private SingleInstance() {
  }

  public static SingleInstance getInstance() {
      if (null == instance) {
          instance = new SingleInstance();
      }
      return instance;
  }
}
```
但是如果上面的代码在多线程并发的情况下就会发生问题， 因为它们存在共同的「临界资源」 instance, 比如线程A进入 `null == instance` 这段代码块，而在A线程未创建完成实例时，这时线程B也进入了该代码块，必然会造成两个实例的产生。

所以如果多线程这里要考虑加锁同步。代码实现如下：
```
public class SingleInstance {
  private static SingleInstance instance;
  private SingleInstance() {
  }

  public static synchronized SingleInstance getInstance() {
      if (null == instance) {
          instance = new SingleInstance();
      }
      return instance;
  }
}
```
如果使用 synchronized 修饰 getInstance 方法后必然会导致性能下降，而且 getInstance 是一个被频繁调用的方法。虽然这种方法能解决问题，但是不推荐使用在多线程的情况下。所以伟大人类又想到了 「双重检查加锁」。

### 双重校验锁
伟大人类想到首先进入该方法时进行 `null == sInstance` 检查，如果第一次检查通过，即没有实例创建，则进入 synchronized 控制的同步块,并再次检查实例是否创建，如果仍未创建，则创建该实例。
```
public class SingleInstance {
  private static SingleInstance instance;
  private SingleInstance() {
  }

  public static SingleInstance getInstance() {
      if (null == instance) {
          synchronized (SingleInstance.class) {
              if(null == instance) {
                  instance = new SingleInstance();
              }
          }
      }
      return instance;
  }
}
```

双重检查加锁保证了多线程下只创建一个实例，并且加锁代码块只在实例创建的之前进行同步。如果实例已经创建后，进入该方法，则不会执行到同步块的代码。

### 静态内部类

```
public class SingleInstance {
  private SingleInstance() {
  }

  public static SingleInstance getInstance() {
      return SingleInstanceHolder.sInstance;
  }

  private static class SingleInstanceHolder {
      private static SingleInstance sInstance = new SingleInstance();
  }
}
```
上面的代码 Singleton 类被装载了，instance 不一定被初始化。因为 SingletonHolder 类没有被主动使用，只有显示通过调用 getInstance 方法时，才会显示装载 SingletonHolder 类，从而实例化 instance。想象一下，如果实例化 instance 很消耗资源，我想让它延迟加载， 上面就能这种方式就能达到。

**优点**:
+ 单例模式（Singleton）会控制其实例对象的数量，从而确保访问对象的唯一性。
+ 实例控制：单例模式防止其它对象对自己的实例化，确保所有的对象都访问一个实例。
+ 伸缩性：因为由类自己来控制实例化进程，类就在改变实例化进程上有相应的伸缩性。


**缺点**:
+ 系统开销。虽然这个系统开销看起来很小，但是每次引用这个类实例的时候都要进行实例是否存在的检查。这个问题可以通过静态实例来解决。
+ 使用多个类加载器加载单例类，也会导致创建多个实例并存的问题。
+ 使用反射，虽然构造器为非公开，但是在反射面前就不起作用了。
+ 对象生命周期。因为单例模式没有提出对象的销毁， 所以使用时容易造成内存泄漏， 例如在 Android 中在 Activity 中使用单例， 所以我们要额外小心。

## 使用场景
+ 系统只需要一个实例对象，如系统要求提供一个唯一的序列号生成器，或者需要考虑资源消耗太大而只允许创建一个对象。
+ 不要使用单例模式存取全局变量。这违背了单例模式的用意，最好放到对应类的静态成员中。
+ 在一个系统中要求一个类只有一个实例时才应当使用单例模式。反过来，如果一个类可以有几个实例共存，就需要对单例模式进行改进，使之成为多例模式

## Android 系统中的应用
在 Android 系统中， 大量使用单例模式， 我们来看一下。

饿汉式：
```
public class CallManager {
    ...
    // Singleton instance
    private static final CallManager INSTANCE = new CallManager();

    public static CallManager getInstance() {
        return INSTANCE;
    }
    ....
}
```
懒汉式非线程安全实现方式:
```
class SnackbarManager {
    .....
    private static SnackbarManager sSnackbarManager;

    static SnackbarManager getInstance() {
        if (sSnackbarManager == null) {
            sSnackbarManager = new SnackbarManager();
        }
        return sSnackbarManager;
    }
}
```


懒汉式线程安全实现方式:

```
public class SystemConfig {
    ...
    static SystemConfig sInstance;
    ...
    public static SystemConfig getInstance() {
        synchronized (SystemConfig.class) {
            if (sInstance == null) {
                sInstance = new SystemConfig();
            }
            return sInstance;
        }
    }
}
```

## 总结
**饿汉 VS 懒汉**：
+饿汉：声明实例引用时就会被实例化
+ 懒汉：静态方法第一次调用前不被实例化，即懒加载。对于创建实例代价大的， 且不一定使用时，这种方式可以减少开销

一般的情况下，构造方法没有太多处理时，我会使用「饿汉」方式， 因为它简单易懂，而且在JVM层实现了线程安全（如果不是多个类加载器环境）。只有在要明确实现延迟加载效果时我才会使用「静态内部类」方式。
