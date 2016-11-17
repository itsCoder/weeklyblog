title: Android 单元测试-Mockito
Date: 2016-11-08 06:30:16
---

以前我在**[内功之自动化测试](http://hujiandong.com/2016/04/10/how_code_auto_test/)**中说到测试在项目中的重要性。单元测试是一个个「点」（细胞）的重构，是重构的基石，今天我们说单元测试中如何使用 **[Mock](https://en.wikipedia.org/wiki/Mock_object)** 及 **[Mockito](http://mockito.org/)** 的。
## Mock 概念
所谓的mock就是创建一个类的虚假的对象，在测试环境中，用来替换掉真实的对象，主要提供两大功能：
+ 验证这个对象的某些方法的调用情况，调用了多少次，参数是什么等等
+ 指定这个对象的某些方法的行为，返回特定的值，或者是执行特定的动作

要使用Mock，一般需要用到mock框架，这篇文章我们使用 Mockito 这个框架，这个是Java界使用最广泛的一个mock框架。
## 在 Gradle 添加 Mockito 依赖
```
repositories { jcenter() }
dependencies { testCompile "org.mockito:mockito-core:1.+" }
```
## 如何使用？
我们首先看一下官方的例子:
```java
//mock creation List
mockedList = mock(List.class);
//using mock object - it does not throw any "unexpected interaction" exception
mockedList.add("one");
// selective, explicit, highly readable verification
verify(mockedList).add("one");
```


一般使用 Mockito 需要执行下面三步:
+ 模拟并替换测试代码中外部依赖。
+ 执行测试代码
+ 验证测试代码是否被正确的执行

创建 Mock 对象的方式：
+ mock(toMockObject.class)
+ 注解的方式 @Mock，注意要利用注解， 首先要告诉 Mockito 框架， 可以`@Rule public MockitoRule mockitoRule = MockitoJUnit.rule);` 或者它的实现`MockitoAnnotations.initMocks(target);`

### 误区一
上面的例子我们进行修改 ==> 为不同于上面的地方，如下:
```java
//mock creation List
mock(List.class);
List<String> list = new ArrayList<>();
//using mock object - it does not throw any "unexpected interaction" exception
list.add("one");
// selective, explicit, highly readable verification
verify(list).add("one");

```
运行发现如下错误：

```
org.mockito.exceptions.misusing.NotAMockException: 
Argument passed to verify() is of type ArrayList and is not a mock!
Make sure you place the parenthesis correctly!
See the examples of correct verifications:
verify(mock).someMethod();
verify(mock, times(10)).someMethod();
verify(mock, atLeastOnce()).someMethod();
```

这就是mock的误区一：
**Mockito.mock()并不是mock一整个类，而是根据传进去的一个类，mock出属于这个类的一个对象，并且返回这个mock对象；而传进去的这个类本身并没有改变，用这个类new出来的对象也没有受到任何改变！**

结合上面的例子，Mockito.mock(ArrayList.class);只是返回了一个属于ArrayList这个类的一个mock对象。ArrayList这个类本身没有受到任何影响，而 list 不是一个mock对象。Mockito.verify()的参数必须是mock对象，也就是说，Mockito只能验证mock对象的方法调用情况。因此，上面那种写法就出错了。

### 误区二
我们先看一个例子：
```java
public class LoginPresenter {
    private UserLoginTask mAuthTask;
    
    public void login(String email, String password) {
        mAuthTask = new UserLoginTask(email, password);
        
        //执行登录操作
        mAuthTask.execute();
    }
}
```
上面是一个登录操作， 现在我们来验证 login() 函数， 因为它没有返回值，这时候我们只要验证 execute() 有没有执行就可以了。

```java
    @Test
    public void testLogin() throws Exception {
        UserLoginTask mockLoginTask = mock(UserLoginTask.class);
        LoginPresenter loginPresenter = new LoginPresenter();
        loginPresenter.login("jason@gmail.com", "123456");
        
        //验证是否执行 excute()
        verify(mockLoginTask).excute();
    }
```
由于 UserLoginTask 继承 AsyncTask， 所以会报错，同误区一中的问题一样（not  mock），这时候我们需要用到 Robolectric 框架，这个可以参考我**[以前写的](http://hujiandong.com/2016/05/20/android-unit-test/)**。解决这个问题以后你会发现，fuck，怎么还有问题， 什么鬼。。

![](http://7xnilf.com1.z0.glb.clouddn.com/mock_wanted.png)

mock的误区二：
**mock出来的对象并不会自动替换掉正式代码里面的对象，你必须要有某种方式把mock对象应用到正式代码里面。**

这个时候我们可以通过 构造方式将依赖传进去，就 OK 了。

```java
public class LoginPresenter {
    private UserLoginTask mAuthTask; //===>
    
    public LoginPresenter(UserLoginTask mAuthTask) {
        //TODO test argument
        //执行登录操作
        mAuthTask.execute(email, password);
    }
}
```

修改测试用例
```
…
LoginPresenter loginPresenter = new LoginPresenter(mockLonginTask); //==>
```
运行发现终于成功了， 不容易。。
### 验证方法调用及参数
使用Mockito，验证一个对象的方法调用情况：
Mockito.verify(objectToVerify).methodToVerify(arguments);
其中 objectToVerify 和 methodToVerify 对应上面的 mockedList 和 add，表示验证 mockedList 的 add 方法是否传入参数是 one。

很多时候你并不关心被调用方法的参数具体是什么，或者是你也不知道，你只关心这个方法得到调用了就行。这种情况下，Mockito提供了一系列的any方法，来表示任何的参数都行。

anyString()表示任何一个字符串都可以。类似anyString，还有anyInt, anyLong, anyDouble等等。anyObject表示任何对象，any(clazz)表示任何属于clazz的对象。
### 指定mock对象的某些方法的行为
那么接下来，我们就来介绍mock的第二大作用，先介绍其中的第一点：指定mock对象的某个方法返回特定的值。
我们见面的 login() 进行修改， 添加对网络的判断， 代码如下：

```java
public void login(String email, String password) {
    //TODO test argument
    if(!NetManager.isConnected()) { //添加网络判断===>
        return;
    }
    
    //执行登录操作
    mAuthTask.execute(email, password);
}
```

修改测试代码：
```java
NetManagerWraper netManagerWraper = mock(NetManagerWraper.class); //==>
when(netManagerWraper.isConnected()).thenReturn(false); ==>
```
下面我们说说怎么样指定一个方法执行特定的动作，这个功能一般是用在目标的方法是void类型的时候。
现在假设我们的LoginPresenter的login()方法是这样的：
```java
//执行登录操作, 并且处理网络返回
mAuthTask.execute(email, password, new NetworkCallBack() {
                         @Override
                         public void onSuccess(Object data) {

                         }

                         @Override
                         public void onFailed(int code, String msg) {

                         }
                         });
```

我们想进一步测试传给 NetworkCallback 里面的代码，验证 view 得到了更新等等。在测试环境下，我们并不想依赖 excute 的真实逻辑，而是让 mAuthTask
直接调用传入的NetworkCallback的onSuccess或onFailed方法。这种指定mock对象执行特定的动作的写法如下：
`Mockito.doAnswer(desiredAnswer).when(mockObject).targetMethod(args);
测试代码如下：

```java
doAnswer(new Answer() {
             @Override
             public Object answer(InvocationOnMock invocation) throws Throwable {
             //这里可以获得传给performLogin的参数
             Object[] arguments = invocation.getArguments();

             //callback是第三个参数
             NetworkCallBack callback = (NetworkCallBack) arguments[2];

             callback.onSuccess(null);
             return null;
             }
             }).when(mockLonginTask).execute(anyString(), anyString(), any(NetworkCallBack.class));
```

我们想在调用某些无返回值函数的时候抛出异常，那么可以使用doThrow 方法。如果想简单的指定目标方法“什么都不做”，那么可以使用Mockito.doNothing()。如果你想让目标方法调用真实的逻辑，可以使用Mockito.doCallRealMethod()（默认不是的， 请看下文）。

## Spy
如果我们不指定 mock 对象方法的行为， 那么他是不是走真实逻辑呢？ 答案是否定的。如果没我们不指定它的行为，对于mock对象的所有非void方法都将返回默认值int，long类型方法将返回0，boolean方法将返回false，对象方法将返回null等等；而void方法将什么都不做。

然而很多时候，你希望达到这样的效果：除非指定，否者调用这个对象的默认实现，同时又能拥有验证方法调用的功能。这正好是spy对象所能实现的效果。
创建Spy方式：
+ Mockito.spy(toMockObject);
+ 通过注解的方式@Spy

```java
    @Test
    public void testSpy() {
        NetManagerWraper spy = spy(new NetManagerWraper());
        assertTrue(spy.isConnected());
        when(spy.isConnected()).thenReturn(false);
    }
```
**spy 与 mock 的唯一区别就是默认行为不一样： spy 对象的方法默认调用真实的逻辑，mock 对象的方法默认什么都不做，或直接返回默认值。**

## Mockito Annotation
通过 Mockito 注解可以快速创建 Mock 对象， 这样对于我们这样的懒人来说，是不是很爽。 上面我也简单提到了， 我们可以通过 mock() 和 @Mock 创建， @Mock 就是通过注解的方式创建的， 由于我们使用注解， 当然我们要告诉 Mockito 框架， 不然它怎样知道你使用注解了， 难道它是神吗？ 加上@Rule 就行了， 这样JUnit Rule（？）就会每个测试方法测试前进行检查。添加方法：`@Rule public MockitoRule mockitoRule = MockitoJUnit.rule();`， 当然创建 spy 对象也可以加 @Spy。
## 写在最后
上面的就是 Mockito 的基本使用， 当然由于并不是很全，更全面的可以看**[这里](http://www.vogella.com/tutorials/Mockito/article.html)**。基本的概念都有了， 下面的就是多用， 在项目中发现问题， 然后带着问题去查看文档。 Android 下还可以用 Dagger 动态依赖框架进行测试， 由于涉及到 Dagger 框架的使用，这个我们可以单独来说， 前提是你知道 Dagger 怎样用。

上面代码在 **[Github](https://github.com/jasonim/JasonAndroidSimple/tree/master/mocksample) 上**。

## 参考

http://www.vogella.com/tutorials/Mockito/article.html

http://chriszou.com/2016/04/29/android-unit-testing-mockito.html

https://medium.com/square-corner-blog/mockito-on-android-88f84656910
