# 设计模式实践系列（一）

最近在看 [《Head First Design Pattern》](https://book.douban.com/subject/1400656/)（顺便推荐一下😄），深感结合实例例子才能理解好各种模式，下面通过一个小场景，来实践一下最近学的四种设计模式（装饰器模式、工厂方法模式、观察者模式和策略模式）。

项目地址： [DesignPatternStudy/src/practice/practice01 at master · brucezz/DesignPatternStudy](https://github.com/brucezz/DesignPatternStudy/tree/master/src/practice/practice01)

我们平常最离不开的就是手机了，那么我们的故事就从**买手机**开始。

我们需要个实体类 `Phone`，以及继承自 `Phone` 的一系列不同品牌的手机。

```java
public abstract class Phone {

    public abstract String getBrand();// 品牌
	
    //...
}
```

不同品牌的手机单独实现一下 `getBrand()` 方法就好了。

```java
public class NexusPhone extends Phone {
    @Override
    public String getBrand() {
        return "Nexus";
    }
}

public class OneplusPhone extends Phone {
    @Override
    public String getBrand() {
        return "Oneplus";
    }
}

// ...
```



不同的手机，自然有自己的 feature， 比如有的手机屏幕大，有的手机续航时间长，还有的音质不错~

这里就要用到第一个模式啦 —— **装饰器模式**。先继承 `Phone` 得到一个装饰器类 `PhoneDecorator`。

```java
public abstract class PhoneDecorator extends Phone {
    protected Phone phone;

    public PhoneDecorator(Phone phone) {
        this.phone = phone;
    }
}
```

这个装饰器类内部维护了一个 `Phone` 对象，方便对其进行装饰，同时继承了 `Phone`，保证装饰后的对象类型与原来的类型保持一致，不会影响到使用。

```java
/**
 * 大屏幕
 */
public class LargeScreen extends PhoneDecorator {
    public LargeScreen(Phone phone) {
        super(phone);
    }

    @Override
    public String getBrand() {
        return phone.getBrand() + ", Large Screen";
    }
}
```

如果想要得到一台大屏幕的 Nexus，可以这样`Phone nexus = new LargeScreen(new NexusPhone());`

其他装饰器可以去参考项目代码，不贴啦。

有了手机，就该有手机店了。我们这里的手机店是专卖店啦，都只卖各自类型的手机。首先我们还是得一个接口（在设计模式里面，接口泛指父类或者接口这种超类型）来定义统一的卖手机的方法。实现如下：

```java
public abstract class PhoneStore {

    public Phone sale() {
        Phone phone = producePhone();// 生产
        phone.pack();// 打包
        phone.transport();// 运输
        return phone;
    }

    protected abstract Phone producePhone();

    public abstract String getName();
}
```

这里用到了另一个模式 —— **工厂方法模式**。将实例化 `Phone` 对象的过程延迟到子类中进行，由子类即具体的手机专卖店来决定手机的类型。这样一来，不同品牌的专卖店就能卖各自家的手机了。

```java
public class NexusStore extends PhoneStore {
    @Override
    public Phone producePhone() {
        return new LongBatteryLife(new LargeScreen(new HiFi(new NexusPhone())));
    }

    @Override
    public String getName() {
        return "NexusStore";
    }
}
```

到这里，`PhoneStore` 也实现好了，消费者可以直接通过 `Phone nexus = new NexusStore().sale();` 来买一台 nexus。 

整个故事的剧情就是这样：

```java
public class Main {

    public static void main(String[] args) {
        Customer jaeger = new Customer("Jaeger", "Nexus");
        jaeger.hello();

        NexusStore store = new NexusStore();
        Phone phone = store.sale();

        jaeger.setPhone(phone);
        jaeger.hello();
    }
}
```

输出：

```
[Jaeger] 嘿, 我是 Jaeger, 我喜欢 Nexus 这个牌子, 不过我还没有手机诶 :(
[Nexus, HiFi, Large Screen, Long Battery Life Phone] 打包中 ...
[Nexus, HiFi, Large Screen, Long Battery Life Phone] 运输中 ...
[Jaeger] 我有手机啦! Nexus, HiFi, Large Screen, Long Battery Life
[Jaeger] 嘿, 我是 Jaeger, 我喜欢 Nexus 这个牌子, 我现在有一个手机: Nexus, HiFi, Large Screen, Long Battery Life 
```

骚年，你以为这样故事就完结了？图样图森破啊！
其实 [Jaeger](https://github.com/laobie) 去买手机的时候，手机没货啦 😅
然后店员告诉他：你在这里登记一下，然后三天之后手机到货了，我们会主动联系你的，到时候你再来买手机，肯定有货！

这个登记和通知的过程嘛，就要用到第三个模式 —— **观察者模式**。手机店为被观察对象， [Jaeger](https://github.com/laobie) 是观察者。首先观察者要向被观察者进行注册，也就是登记的过程。然后被观察者在状态改变时，会通知所有注册的观察者。

```java
public abstract class PhoneStore {

    /**
     * 等待购买的顾客的列表
     */
    protected List<Waiting> waitList = new ArrayList<>();

    public Phone sale() {
        if (isPhoneValid()) {
            Phone phone = producePhone();
            phone.pack();
            phone.transport();
            return phone;
        } else {
            System.out.println("[" + getName() + "] 现在手机卖完啦, 你先登记一下, 等三天之后手机到货了我们再通知你~");
            days = 0;
            return null;
        }
    }

    public void register(Waiting waiting) {
        if (!waitList.contains(waiting)) {
            waitList.add(waiting);
        }
    }

    public void unRegister(Customer customer) {
        waitList.remove(customer);
    }

    public void notifyAllCustomers() {
        for (Waiting waiting : waitList) {
            waiting.onPhoneValid(this);
        }
    }

    private int days = 3;

    /**
     * 等待三天
     */
    public void oneDayPassed() {
        days++;
        System.out.println(days + " 天过去了 ...");
        if (days >= 3) {
            notifyAllCustomers();
        }
    }

    private boolean isPhoneValid() {
        return days >= 3;
    }

    protected abstract Phone producePhone();

    public abstract String getName();

    public interface Waiting {
        void onPhoneValid(PhoneStore store);
    }
}
```

```java
public class Customer implements PhoneStore.Waiting {

    public String name;
    public String favorite;
    public Phone phone;

    /**
     * @param name     名字
     * @param favorite 喜欢的牌子
     */
    public Customer(String name, String favorite) {
        this.name = name;
        this.favorite = favorite;
    }

    public void setPhone(Phone phone) {
        System.out.println("[" + name + "] " +
                (this.phone == null ? "我有" : "我换") +
                "手机啦! " + phone.getBrand());
        this.phone = phone;
    }

    /**
     * 商店通知说,有货啦!
     */
    @Override
    public void onPhoneValid(PhoneStore store) {
        setPhone(store.sale());
    }

    @Override
    public boolean equals(Object obj) {
        return obj instanceof Customer && name.equals(((Customer) obj).name);
    }

    @Override
    public String toString() {
        return String.format("[%s] 嘿, 我是 %s, 我喜欢 %s 这个牌子, " + (phone == null
                ? "不过我还没有手机诶 :("
                : "我现在有一个手机: " + phone.getBrand() + " ^_^"), name, name, favorite);
    }

    public void hello() {
        System.out.println(toString());
    }
}
```

这里用 `oneDayPassed()` 方法来模拟时间的流逝，计数累计到 3 时，为手机有货的状态。
让消费者实现 `PhoneStore.Waiting` 这个接口来定义收到通知时的操作。商店内部维护了一个列表，把所有等待的消费者都加进去了，等到手机有货了，挨个通知消费者，让其回来买手机。

通过实现观察者模式，解决了手机卖完的尴尬了。。

```java
public class Main {

    public static void main(String[] args) {
        Customer jaeger = new Customer("Jaeger", "Nexus");
        jaeger.hello();

        PhoneStore store = new NexusStore();
        Phone phone = store.sale();

        if (phone == null) {
            // 等三天
            store.register(jaeger);
            store.oneDayPassed();
            store.oneDayPassed();
            store.oneDayPassed();
        }

        jaeger.hello();
    }
}
```

```
[Jaeger] 嘿, 我是 Jaeger, 我喜欢 Nexus 这个牌子, 不过我还没有手机诶 :(
[NexusStore] 现在手机卖完啦, 你先登记一下, 等三天之后手机到货了我们再通知你~
1 天过去了 ...
2 天过去了 ...
3 天过去了 ...
[Nexus, HiFi, Large Screen, Long Battery Life Phone] 打包中 ...
[Nexus, HiFi, Large Screen, Long Battery Life Phone] 运输中 ...
[Jaeger] 我有手机啦! Nexus, HiFi, Large Screen, Long Battery Life
[Jaeger] 嘿, 我是 Jaeger, 我喜欢 Nexus 这个牌子, 我现在有一个手机: Nexus, HiFi, Large Screen, Long Battery Life ^_^
```

最后， [Jaeger](https://github.com/laobie) 觉得这个 Nexus 手机用着不爽，要买最新的 1+3 😒。这个不难，直接去专卖店买一个就好啦。

```java
public class Main {

    public static void main(String[] args) {
        // ... 省略

        // 换个手机
        store = new OneplusStore();
        phone = store.sale();
        jaeger.setPhone(phone);
        jaeger.hello();
    }
}
```

其实这里包含了最后一种模式 —— **策略模式**。`Custom` 对象中持有一个 `Phone` 手机对象，这是一个抽象类型，并不是一个具体的类。这样的好处就是可以方便地用 `Phone` 的任意子类去替换 `Custom` 中的手机，而不会影响到内部其他地方的使用。



吼了，故事就扯到这里了，通过  [Jaeger](https://github.com/laobie) 买手机的经历，实践了一下这四个设计模式，总结一下：

- 装饰器模式：动态地给一个对象增加一些额外的职责，比生成子类更为灵活。
- 工厂方法模式：定义一个用于创建对象的接口，让子类决定将哪一个类实例化
- 观察者模式：定义对象之间的一种一对多依赖关系，使得每当一个对象状态发生改变时，其相关依赖对象皆得到通知并被自动更新。
- 策略模式：定义一系列策略，将每一个策略封装起来，并让它们可以相互替换，让他们的变化独立语调用者。




其他设计模式待续……

我的设计模式练习代码：[brucezz/DesignPatternStudy: Design Pattern in Java](https://github.com/brucezz/DesignPatternStudy)

对于设计模式我也只是初学，有一些自己的理解，欢迎来讨论指点 😄

>  最后的最后，感谢好机油  [Jaeger](https://github.com/laobie) 倾情出演😆！