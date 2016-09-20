---
title: 利用动态代理来做点事
toc: true
date: 2016-09-3 15:30:23
tags: java
categories: About Java
description:
feature:
---

 本篇文章介绍动态代理在 Java 和 Android 中的一些应用。

<!-- more -->

### 引入

本篇文章主要介绍一下动态代理在 Java 和 Android 中的应用。了解动态代理之前，你需要知道什么是代理以及什么是静态代理。由于篇幅所限，本文就不介绍这些了。你可以点击[这里查看](https://zhuanlan.zhihu.com/p/20376806)或者[知乎上的讨论](https://www.zhihu.com/question/20794107)，你也可以自行 Google 一下。

## 一. 概念介绍

动态代理是 Java 一大特性。它的显著优势就是**无侵入式的扩展代码**。通俗来讲就是可以用来做**方法的增强**，让你可以在不修改源码的情况下，增强一些方法或者功能，在方法执行前后做任何你想做的事情。具体应用的话，比如可以添加调用日志，做事务控制等。

目前动态代理主要分为 Java 自己提供的动态代理和 CGLIB 类似框架。Java 自带的动态代理是需要接口来辅助的。CGLIB 这种则是直接修改字节码。


## 二. 在 Java 中的应用 

### 2.1 实现功能

我们通过一个实例来看看 JDK 中时如何来使用动态代理的，我们要实现的效果是不改变原有代码结构的情况下，在调用一个方法前面插入另外的方法，达到在调用原方法前后打印日志信息的目的。

### 2.2 定义接口及其实现

由于JDK 动态代理只能为接口**创建代理**，故提供一个接口以及其实现。Dog 接口定义如下：

``` java
interface Dog{
    void info();
    void run();
}
```

Dog 的具体实现如下：

``` java
class GunDog implements Dog{

    @Override
    public void info() {
        System.out.println("我是一只狗");
    }

    @Override
    public void run() {
        System.out.println("我迅速奔跑");
    }
}
```

### 2.2 增强方法

接着提供一个 DogUtil,里面包含着两个通用方法，也就是我们希望在执行 GunDog 里面的**两个方法的前后**能够执行下面的两个通用方法：

``` java
class DogUtil{
	// 模拟方法一
    public void method1(){
        System.out.println("==模拟第一个通用方法==");
    }
	// 模拟方法二
    public void method2(){
        System.out.println("==模拟第二个通用方法==");
    }
}
```

### 2.3 实现 InnovationHandler 接口 

动态代理的核心环节就是要实现 InvocaitonHandler 接口。每个动态代理类都必须要实现 InvocationHandler 接口，并且每个代理类的实例都关联到了一个 InvocationHandler 接口实现类的实例对象，当我们通过代理对象调用一个方法的时候，这个方法的调用就会被转发为由 InvocationHandler 这个接口的 invoke() 方法来进行调用，具体的实现如下：

``` java
// 实现调度处理器接口
class MyInvocationHandler implements InvocationHandler{
   // 需要被代理的那个对象
    private Object target;

    public void setTarget(Object target){
        this.target = target;
    } 
  	// 当我们去调用被代理对象的方法时，invoke 方法将会取而代之。
    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {

        DogUtil du = new DogUtil();
      	// 调用 GunDog 中的 info() 或者 run() 方法前
        du.method1();
      	// 用反射去调用 Dog class 中的方法
        Object result = method.invoke(target,args);// 注释一
      	// 调用 GunDog 中的 info() 或者 run() 方法后
        du.method2();
        return result;
    }
}
```

对于上述代码，我们最直接的疑惑就是 method.invoke(target,args) 方法中的参数是什么意思？在哪里被调用的？先不要着急，我们先把过程走一遍，然后再仔细分析。

### 2.4 生成代理对象

提供一个工厂方法，为指定的 target 生成代理对象：

``` java
class MyProxyFactory{

    public static Object getProxy(Object target) throws Exception{
		// 实例化 MyInvocationHandler 对象
        MyInvocationHandler handler = new MyInvocationHandler();
        handler.setTarget(target);
      	// 调用 proxy 的 newProxyInstance() 方法传入三个参数,创建代理类实例
        return Proxy.newProxyInstance(target.getClass().getClassLoader(),
                target.getClass().getInterfaces(),handler);
    }
}
```

上面的 newProxyInstance() 中的三个参数还是很重要的，我们稍微解释一下三个参数的含义

* 参数一：类加载器对象即用哪个类加载器来加载这个代理类到 JVM 的方法区


* 参数二：接口表明该代理类需要实现的接口


* 参数三：是调用处理器类实例即 InvocationHandler 的实现的实例对象

这个函数是 JDK 为了程序员方便创建代理对象而封装的一个函数，因此你调用`newProxyInstance()`时直接创建了代理对象。

### 2.5 测试结果

最后写一个测试用例：

``` java
// 实例化一个被代理对象
Dog target = new GunDog();// 注意二
// 用被代理对象去产生一个代理类对象 dog
Dog dog = (Dog) MyProxyFactory.getProxy(target);
// 代理对象调用被代理对象的方法啦
dog.info();
dog.run();
```

结果如下：

``` java
==模拟第一个通用方法==
我是一只狗
==模拟第二个通用方法==
==模拟第一个通用方法==
我迅速奔跑
==模拟第二个通用方法==
```

### 2.6 原理分析

如上我们用代理对象 dog 去调用了本不属于它的 info() 和  run() 方法。实际上，当调用代理对象的每个方法时，都会被替换执行 InvocationHandler 对象的 invoke() 方法。

当然啦，此时我们不得不进去 invoke() 方法中看看啦，也就是 2.3 小节中的 invoke(target,args) 方法。等等，method.invoke() 中的 method 是啥啊？其实这个就是我们想要 **调用代理对象中的方法**。那 invoke(target,args) 中的两个参数是什么啊？哈哈，往上看一眼呗，target 原来就是**被代理对象啊**，那 args 呢？这不容易知道，但是笔者查阅资料，发现该参数会传入被代理对象的方法中，由于本例子不需要这些参数，所以不用多做考虑啦。然而我们进去并没有发现什么可用的信息：

``` java
@CallerSensitive
public Object invoke(Object obj, Object... args)
     throws IllegalAccessException, IllegalArgumentException,
        InvocationTargetException{
    if (!override) {
            if (!Reflection.quickCheckMemberAccess(clazz, modifiers)) {
                Class<?> caller = Reflection.getCallerClass();
                checkAccess(caller, clazz, obj, modifiers);
            }
    }
   MethodAccessor ma = methodAccessor;             // read volatile
  	if (ma == null) {
            ma = acquireMethodAccessor();
     }
  return ma.invoke(obj, args);
}
```

那我们得调转枪头，看看另一个切入点 Proxy.newProxyInstance() 啦。贴出部分源码如下：

``` java
@CallerSensitive
public static Object newProxyInstance(ClassLoader loader,
                                          Class<?>[] interfaces,
                                          InvocationHandler h)
        throws IllegalArgumentException
    {
        Objects.requireNonNull(h);

        final Class<?>[] intfs = interfaces.clone();
        final SecurityManager sm = System.getSecurityManager();
        if (sm != null) {
            checkProxyAccess(Reflection.getCallerClass(), loader, intfs);
        }
      	// 调用 getProxyClass0() 方法
 		Class<?> cl = getProxyClass0(loader, intfs);
 		final Constructor<?> cons = cl.getConstructor(constructorParams);
		final InvocationHandler ih = h;
    // 返回代理类的实例
 	return cons.newInstance(new Object[]{h});
}
```

上述代码中，除了最后返回的代理类实例，我们为一个的突破口就是 getProxyClass0() 方法啦，跟进去看看呗：

``` java
private static Class<?> getProxyClass0(ClassLoader loader,
                                           Class<?>... interfaces) {
   if (interfaces.length > 65535) {
            throw new IllegalArgumentException("interface limit exceeded");
    }
	// proxyClassCache 来做一个中间的缓存
    return proxyClassCache.get(loader, interfaces);
}
```

我们可以跟进 proxyClassCache 对象所对应的类中去看看，但是由于篇幅原因(有兴趣可以跟进这个类中看看)，就大概说说吧。顾名思义其作用是一个缓存。将所有生成的代理类都存入一个 Map 中，由生成的接口与其对应。当已经拥有此代理类时则直接取出，无此代理类时则动态生成。此外还做了一些多线程的设计。由上继续跟，发现一个生成代理类的工厂类，不用说，里面肯定有能够生成代理类相关的内容：

``` java
private static final class ProxyClassFactory
        implements BiFunction<ClassLoader, Class<?>[], Class<?>>
    {
        // prefix for all proxy class names
        private static final String proxyClassNamePrefix = "$Proxy";

        // next number to use for generation of unique proxy class names
        private static final AtomicLong nextUniqueNumber = new AtomicLong();

        @Override
        public Class<?> apply(ClassLoader loader, Class<?>[] interfaces) {
          	// ...
          	// 注释一
            byte[] proxyClassFile = ProxyGenerator.generateProxyClass(
                proxyName, interfaces, accessFlags);
            try {
              	// 注释二
                return defineClass0(loader, proxyName,
                                    proxyClassFile, 0, proxyClassFile.length);
            } catch (ClassFormatError e) {
        
         }
     }
}
```

我们理解一下注释一处的代码，这个地方提前说一下 generateProxyClass() 用来生成代理类的二进制字节码数据。下面我们会去验证的。不过你可以从 byte[] proxyClassFile 的定义猜到一点。注释二处再调用 defineClass0() 方法加载类。其作用应该就类似于 ClassLoader.loadClass。

接着跟下去，其实也没啥必要了，不过还是要看一下，为了验证之前的猜想：

``` java
 public static byte[] generateProxyClass(final String var0, Class<?>[] var1, int var2) {
        ProxyGenerator var3 = new ProxyGenerator(var0, var1, var2);
        final byte[] var4 = var3.generateClassFile();
        if(saveGeneratedFiles) {
            AccessController.doPrivileged(new PrivilegedAction() {
                public Void run() {
                    try {
                        int var1 = var0.lastIndexOf(46);
                        Path var2;
                        if(var1 > 0) {
                            Path var3 = Paths.get(var0.substring(0, var1).replace('.', File.separatorChar), new String[0]);
                            Files.createDirectories(var3, new FileAttribute[0]);
                            var2 = var3.resolve(var0.substring(var1 + 1, var0.length()) + ".class");
                        } else {
                            var2 = Paths.get(var0 + ".class", new String[0]);
                        }
						// 将数据写进文件
                        Files.write(var2, var4, new OpenOption[0]);
                        return null;
                    } catch (IOException var4x) {
                        throw new InternalError("I/O exception saving generated file: " + var4x);
                    }
                }
            });
        }
    return var4;
}
```

好了，以上就差不多了。其实也没必要挖这么深，对于这些我们只需要明白，最后**会生成**代理类的二进制字节码文件，并且会在运行时动态的加载该字节码文件。我们借助于反编译工具 dump 一下生成的 .class 文件，其核心代码如下：

```java
public final class $Proxy1 extends Proxy implements Subject{
    private InvocationHandler h;
    private $Proxy1(){}
    public $Proxy1(InvocationHandler h){
        this.h = h;
    }
    public int request(int i){
      	// 创建 method 对象
        Method method = Subject.class.getMethod("request", new Class[]{int.class});
      	// 调用了 invoke() 方法
        return (Integer)h.invoke(this, method, new Object[]{new Integer(i)}); 
    }
}
```

看到上面，我们大家想必都已经明白了整个流程。在我们的入口处即 newProxyInstance() 动态生成的代理类对象 $Proxy1 会去回调 MyInvocationHandler 的 invoke() 方法的。

上述的简单例子，帮助我们初步理解了动态代理到底是怎么一个回事，以及动态代理的基本用法。接下来我们看看动态代理在 Android 中的一个典型应用。

## 二. 在 Android 的应用

大名鼎鼎的 retrofit2.0 相信大家都已经用过了，里面的原理大家也应该略知一二。现在我们就在本文第一部分的的基础上看看 Retrofit2.0 中怎么用到动态代理以及使用动态代理带来的好处。下面用 retrofit2.0 来作为通信支持，来访问一下 GitHub 获得用户基本信息。

首先来根据 Retrofit2.0 的用法定义一个 GitHubApi 接口：

``` java
public interface GitHubApi {
    @GET("users/{user}")
    Call<Repo> listInfos(@Path("user") String user);
}
```

实体类 Repo 的代码如下：

``` java
public class Repo{
  private String blog;
  // 省略 get and set
}
```

MainActivity 中的主要方法如下实现：

``` java
btnSearch.setOnClickListener(new View.OnClickListener() {
  @Override
  public void onClick(View v) {
    Retrofit retrofit = new Retrofit.Builder()
             .baseUrl(API_URL)
             .addConverterFactory(GsonConverterFactory.create())
             .build();
    // 注释一
   	GitHubApi github = retrofit.create(GitHubApi.class);
    // 注释二
    Call<Repo> call = github.listInfos("wuchangfeng");
         call.enqueue(new Callback<Repo>() {
           @Override
            public void onResponse(Call<Repo> call, Response<Repo> response) {
                  // 打印出查询信息
                 Log.d(TAG,response.body().getBlog());
            }
           @Override
           public void onFailure(Call<Repo> call, Throwable t) {
                 Log.d(TAG,t.getMessage());
                 }
            });
      }
});
```

OK. 如上代码，就能获取 GitHub 账号个人的信息了。接下来在如上**注释一**处，我们是否应该有点疑问？我们传入进去一个 GithubApi 接口的 .class 对象进去，结果又给我们返回了一个 GithubApi 接口的对象？那不是多此一举吗？为了一探究竟。我们进入 create()  方法中看看：

``` java
public <T> T create(final Class<T> service) {
  	//...
  	// 注释三 返回代理类的实例
    return (T) Proxy.newProxyInstance(service.getClassLoader(), new Class<?>[] { service },
        new InvocationHandler() {
          private final Platform platform = Platform.get();
			// 注释四
          @Override public Object invoke(Object proxy, Method method, Object... args)
              throws Throwable {
            // 不用关心
            if (method.getDeclaringClass() == Object.class) {
              return method.invoke(this, args);
            }
            // 兼容 java8
            if (platform.isDefaultMethod(method)) {
              return platform.invokeDefaultMethod(method, service, proxy, args);
            }
            // 注释五
            ServiceMethod serviceMethod = loadServiceMethod(method);
            OkHttpCall okHttpCall = new OkHttpCall<>(serviceMethod, args);
            // 注释六
            return serviceMethod.callAdapter.adapt(okHttpCall);
          }
        });
  }
```

咦，代码总体结构上我们是不是似曾相识，特别从注释三开始。没错，是动态代理。从这里我们就大概知道了，retrofit2.0 内部也应用了动态代理技术。从而我们大致上也应该知道注释一处返回的 github 对象，已经是一个 **GitHupApi 接口的代理对象啦**。

 还记得我们在**注释二**处的 github.listInfos("allen") 方法嘛，虽然我们用**代理对象去调用 listinfos() 方法**，但是基于前面的分析，实际上是去调用**代理对象的 invoke() 方法啦**。想必代理对象的 invoke() 方法内必定影藏了很多黑科技。

我们接着看看 invoke() 中的三个参数，有了**一部分中**的分析，我们很容易知道：

* proxy 即 github 对象

* args 参数,实际上需要传入的参数，在本例中就是 "allen"

* method 即在接口 GitHubApi 中函数定义，明显我们可以获取到 @GET 注解以及参数 ，再次贴出其定义如下

  ``` java
  @GET("users/{user}")
  Call<Repo> listInfos(@Path("user") String user);
  ```

总体上而言，我们知道了当我们去调用 listInfos() 时，会去调用代理对象的 invoke() 方法，其实我们可以从注释五追踪下去，但是没啥必要了(文章侧重点在动态代理的应用)。而至此我们需要明白的就是为什么使用动态代理对象？再看注释五处，得到的 serviceMethod 和 okhttpcall 这意味 retrofit 内部已经将 GitHubApi 内部的方法转换成代理对象的方法并且 @GET 注解也已经转换成 HTTP 请求了。接下来都是 okHttp 的事情了...

而到这里我们也明白了动态代理的作用啦：基于其本身的特性，是在运行时动态生成代理类，灵活性可见一斑。另外 Retrofit2.0 中基于接口生成动态代理类的，并且接口的规范上设置了注解。HTTP 的请求就这样被简化了。另一个方面记得我们开始定义的那个 GitHubApi 的接口吗？想想，按照正常的 Java 编程逻辑，我们是不是应该去实现接口，并且实现其里面的逻辑方法？如果没有注解的使用和动态代理，我们将不可避免的去写许多类似于 XXXImpl 的实现类。

因为文章是分析动态代理在 Retrofit2.0 中的作用，所以就不继续深挖下去了。

## 三. 总结与分析

文章主要讲述动态代理在 Java 和 Android 的一些应用。其实对于动态代理，我们可以从两个方面来分析和学习：动态和代理。动态生成代码相对于静态编译期所带来的好处是非常明显的。而对于代理，我们可以从设计模式中的代理模式略窥一二。

在**一部分中**我们是想在调方法前后加上日志打印的功能。当然，这种功能太小了，这里也只是举个例子。一般的逻辑思维，是在调用方法前后，去硬编码调用其他的方法，而由此也会让程序出现耦合...而如上所示的动态代理能够解决完美的避免耦合发生的问题。

而在**二部分中**我们稍微探究了一下动态代理在 Retrofit2.0 中的应用。Retrofit2.0 用注解来描述一个 HTTP 请求，将其抽象成一个接口，接着**动态的**将这个接口的注解翻译成一个 HTTP 请求，并且最后再执行这个HTTP请求。

最后，本来想分析分析 Android 中 Hook 技术的。也成功的根据作者的文章实现了 Demo。后来改造了一下，一直没有成功，发邮件问了原作者，也没得到回应。同时也按照自己的思路写了一半关于 Hook 的文章，但是后来发现很难离开原作者的思路。大概是本人没有理解以及原作者写的太好了的原因吧。如果你有兴趣了解 Hook 技术，推荐你看[Android插件化原理解析——Hook机制之动态代理](http://weishu.me/2016/01/28/understand-plugin-framework-proxy-hook/)。

 ## 四. 参考

* 疯狂 Java 第四版
* Retorfit2.0 官方文档





