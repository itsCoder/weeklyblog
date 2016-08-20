## 热修复入门：ClassLoader 机制

> 对于 Java 程序来说，编写程序就是编写类，运行程序也就是运行类（编译得到的 class 文件），其中起到关键作用的就是类加载器 ClassLoader。

### ClassLoader 简介

任何一个 Java 程序都是由若干个 class 文件组成的一个完整的 Java 程序，在程序运行时，需要将 class 文件加载到 JVM 中才可以使用，负责加载这些 class 文件的就是 Java 的类加载（ClassLoader）机制。![](http://ac-qygvx1cc.clouddn.com/78e71017bdd24420.jpeg)

因此 ClassLoader 的作用简单来说就是加载 class 文件，提供给程序运行时使用。

##### ClassLoader 的双亲委托模型（Parent *Delegation Model* ）

先来看 jdk 中的 ClassLoader 类的构造方法，其需要传入一个父类加载器，并持有该引用。

```java
protected ClassLoader(ClassLoader parent) {
    this(checkCreateClassLoader(), parent);
}
```

当类加载器收到加载类或资源的请求时，通常都是先委托给父类加载器加载，也就是说只有当父类加载器找不到指定类或资源时，自身才会执行实际的类加载过程，具体的加载过程如下：

1. 源 ClassLoader 先判断该 Class 是否已加载，如果已加载，则直接返回 Class，如果没有则委托给父类加载器。
2. 父类加载器判断是否加载过该 Class，如果已加载，则直接返回 Class，如果没有则委托给祖父类加载器。
3. 依此类推，直到始祖类加载器（引用类加载器）。
4. 始祖类加载器判断是否加载过该 Class，如果已加载，则直接返回 Class，如果没有则尝试从其对应的类路径下寻找 class 字节码文件并载入。如果载入成功，则直接返回 Class，如果载入失败，则委托给始祖类加载器的子类加载器。
5. 始祖类加载器的子类加载器尝试从其对应的类路径下寻找 class 字节码文件并载入。如果载入成功，则直接返回 Class，如果载入失败，则委托给始祖类加载器的孙类加载器。
6. 依此类推，直到源 ClassLoader。
7. 源 ClassLoader 尝试从其对应的类路径下寻找 class 字节码文件并载入。如果载入成功，则直接返回 Class，如果载入失败，源 ClassLoader 不会再委托其子类加载器，而是抛出异常。

如果需要详细了解 ClassLoader 的信息，可以借助以下文章深入了解：

- [JVM 的工作原理,层次结构以及 GC 工作原理](https://segmentfault.com/a/1190000002579346)
- [深入分析Java ClassLoader原理](http://blog.csdn.net/xyang81/article/details/7292380)
- [类加载机制：全盘负责和双亲委托](http://blog.csdn.net/zhangzeyuaaa/article/details/42499839)

### Android 中的 ClassLoader

Android 的 Dalvik/ART 虚拟机如同标准 Java 的 JVM 虚拟机一样，也是同样需要加载 class 文件到内存中来使用，但是在 ClassLoader 的加载细节上会有略微的差别。

#### Android 中的 dex 文件

Android 应用打包成 apk 文件时，class 文件会被打包成一个或者多个 dex 文件。将一个 apk 文件后缀改成 .zip 格式解压后（也可以直接解压，apk 文件本质是个 zip 文件），里面就有 class.dex 文件，由于 Android 的 65K 问题（不要纠结是 64K 还是 65K），使用 MultiDex 就会生成多个 dex 文件。

![](http://ac-QYgvX1CC.clouddn.com/3c5e66e9e048d343.jpg)

当 Android 系统安装一个应用的时候，会针对不同平台对 Dex 进行优化，这个过程由一个专门的工具来处理，叫 DexOpt 。DexOpt 是在第一次加载 Dex 文件的时候执行的，该过程会生成一个 ODEX 文件，即 Optimised Dex。执行 ODEX 的效率会比直接执行 Dex 文件的效率要高很多，加快 App 的启动和响应。

ODEX 相关的细节可以阅读以下文章扩展：

- [ART 和 Dalvik](http://www.mywiki.cn/hovercool/index.php/ART%E5%92%8CDalvik)
- [ODEX格式及生成过程](http://www.jianshu.com/p/242abfb7eb7f) 
- [What are ODEX files in Android](http://stackoverflow.com/questions/9593527/what-are-odex-files-in-android) 

> 注：本人的 5.0 机器 ODEX 优化后的文件是在 `/data/dalvilk-cache` 文件夹下的，6.0 机器该文件夹下只有 framework 和部分内置的 App 的优化后的 dex 文件，查找相关资料后没有找到明确的说法，目前猜测和 ROM 有关系，后续再深究下这个问题。

![](http://ac-QYgvX1CC.clouddn.com/b79b994f71a47130.png)

总之，Android 中的 Dalvik/ART 无法像 JVM 那样 **直接** 加载 class 文件和 jar 文件中的 class，需要通过 dx 工具来优化转换成 Dalvik byte code 才行，只能通过 dex 或者 包含 dex 的jar、apk 文件来加载（注意 odex 文件后缀可能是 .dex 或 .odex，也属于 dex 文件），因此 Android 中的 ClassLoader 工作就交给了 BaseDexClassLoader 来处理。

> 注：如果 jar 文件包含有 dex 文件，此时 jar 文件也是可以用来加载的，不过实际加载的还是其中的 dex 文件，不要弄混淆了。

#### BaseDexClassLoader 及其子类

在 Android 开发者官网上的 [ClassLoader](https://developer.android.com/reference/java/lang/ClassLoader.html) 的文档说明中我们可以看到，ClassLoader 是个抽象类，其具体实现的子类有 `BaseDexClassLoader` 和 ` SecureClassLoader` 。

SecureClassLoader 的子类是 `URLClassLoader` ，其只能用来加载 jar 文件，这在 Android 的 Dalvik/ART  上没法使用的。

BaseDexClassLoader 的子类是 `PathClassLoader` 和 `DexClassLoader` 。

##### PathClassLoader

PathClassLoader 在应用启动时创建，从 data/app/… 安装目录下加载 apk 文件。

其有 2 个构造函数，如下所示，这里遵从之前提到的双亲委托模型：

```java
public PathClassLoader(String dexPath, ClassLoader parent) {
    super(dexPath, null, null, parent);
}

public PathClassLoader(String dexPath, String libraryPath,
        ClassLoader parent) {
    super(dexPath, null, libraryPath, parent);
}
```

- `dexPath` :  包含 dex 的 jar 文件或 apk 文件的路径集，多个以文件分隔符分隔，默认是“：”
- `libraryPath` : 包含 C/C++ 库的路径集，多个同样以文件分隔符分隔，可以为空

PathClassLoader 里面除了这 2 个构造方法以外就没有其他的代码了，具体的实现都是在 BaseDexClassLoader 里面，其 dexPath 比较受限制，一般是已经安装应用的 apk 文件路径。

在 Android 中，App 安装到手机后，apk 里面的 class.dex 中的 class 均是通过 PathClassLoader 来加载的。

我们可以新建一个项目来验证下，在 MainActivity 中添加如下代码：

```java
public class MainActivity extends AppCompatActivity {

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        ClassLoader loader = MainActivity.class.getClassLoader();
        while (loader != null) {
            System.out.println(loader.toString());
            loader = loader.getParent();
        }
    }
}
```

输出结果是：

```
 I/System.out: dalvik.system.PathClassLoader[DexPathList[[zip file "/data/app/com.jaeger.testclassloader-2/base.apk"],nativeLibraryDirectories=[/vendor/lib, /system/lib]]]
 I/System.out: java.lang.BootClassLoader@1d9c6226
```

`/data/app/com.jaeger.testclassloader-2/base.apk` 就是示例应用安装在手机上的位置。

BootClassLoader 是 PathClassLoader 的父加载器，其在系统启动时创建，在 App 启动时会将该对象传进来，具体的调用在 `com.android.internal.os.ZygoteInit` 的 `main()` 方法中调用了 `preload()` ， 然后调用 `preloadClasses()` 方法，在该方法内部调用了 Class 的 `forName()` 方法：

```java
Class.forName(line, true, null);
```

`forName()` 方法源码如下，方法内部获取到 BootClassLoader 实例：

```java
public static Class<?> forName(String className, boolean shouldInitialize,
        ClassLoader classLoader) throws ClassNotFoundException {
    if (classLoader == null) {
        classLoader = BootClassLoader.getInstance();
    }
    // Catch an Exception thrown by the underlying native code. It wraps
    // up everything inside a ClassNotFoundException, even if e.g. an
    // Error occurred during initialization. This as a workaround for
    // an ExceptionInInitializerError that's also wrapped. It is actually
    // expected to be thrown. Maybe the same goes for other errors.
    // Not wrapping up all the errors will break android though.
    Class<?> result;
    try {
        result = classForName(className, shouldInitialize, classLoader);
    } catch (ClassNotFoundException e) {
        Throwable cause = e.getCause();
        if (cause instanceof LinkageError) {
            throw (LinkageError) cause;
        }
        throw e;
    }
    return result;
}
```

而 PathClassLoader 的实例化又是在哪进行的呢？在源码中寻找下其构造方法调用的地方，结果如下：

![](http://ac-QYgvX1CC.clouddn.com/a0a44b8cac607cae.jpg)

其中：

- 在 ZygoteInit 中的调用是用来启动相关的系统服务

- 在 ApplicationLoaders 中用来加载系统安装过的 apk，用来加载 apk 内的 class ，其调用是在 LoadApk 类中的 `getClassLoader()` 方法中调用的，得到的就是 PathClassLoader：

  ```java
  mClassLoader = ApplicationLoaders.getDefault().getClassLoader(zip, lib,
          mBaseClassLoader);
  ```

##### DexClassLoader

介绍 DexClassLoader 之前，先来看看其官方描述：

> A class loader that loads classes from .jar and .apk filescontaining a  classes.dex entry. This can be used to execute code notinstalled as part of an application.

很明显，对比 PathClassLoader 只能加载已经安装应用的 dex 或 apk 文件，DexClassLoader 则没有此限制，可以从 SD 卡上加载包含 class.dex 的 .jar 和 .apk 文件，这也是插件化和热修复的基础，在不需要安装应用的情况下，完成需要使用的 dex 的加载。

DexClassLoader 的源码里面只有一个构造方法，这里也是遵从双亲委托模型：

```java
public DexClassLoader(String dexPath, String optimizedDirectory,
        String libraryPath, ClassLoader parent) {
    super(dexPath, new File(optimizedDirectory), libraryPath, parent);
}
```

参数说明：

- `String dexPath` : 包含 class.dex 的 apk、jar 文件路径 ，多个用文件分隔符(默认是 ：)分隔

- `String optimizedDirectory` : 用来缓存优化的 dex 文件的路径，即从 apk 或 jar 文件中提取出来的 dex 文件。该路径不可以为空，且应该是应用私有的，有读写权限的路径（实际上也可以使用外部存储空间，但是这样的话就存在代码注入的风险），可以通过以下方式来创建一个这样的路径：

  ```java
  File dexOutputDir = context.getCodeCacheDir();
  ```

  > 注：后续发现，getCodeCacheDir() 方法只能在 API 21 以上可以使用。 

- `String libraryPath` : 存储 C/C++ 库文件的路径集

- `ClassLoader parent `: 父类加载器，遵从双亲委托模型

简单介绍了 PathClassLoader 和 DexClassLoader，但这两者都是对 BaseDexClassLoader 的一层简单封装，真正的实现都在 BaseClassLoader 内。

#####  BaseClassLoader 源码分析

先来看一眼 BaseClassLoader 的结构：

![](http://ac-QYgvX1CC.clouddn.com/a6f9824c199cf304.jpg)

其中有个重要的字段 `private final DexPathList pathList` ，其继承 ClassLoader 实现的 `findClass()` 、`findResource()` 均是基于 pathList 来实现的（省略了部分源码）：

```java
@Override
protected Class<?> findClass(String name) throws ClassNotFoundException {
    List<Throwable> suppressedExceptions = new ArrayList<Throwable>();
    Class c = pathList.findClass(name, suppressedExceptions);
    ...
    return c;
}
@Override
protected URL findResource(String name) {
    return pathList.findResource(name);
}
@Override
protected Enumeration<URL> findResources(String name) {
    return pathList.findResources(name);
}
@Override
public String findLibrary(String name) {
    return pathList.findLibrary(name);
}
```

那么重要的部分则是在 DexPathList 类的内部了，DexPathList 的构造方法也较为简单，和之前介绍的类似：

```java
public DexPathList(ClassLoader definingContext, String dexPath,
        String libraryPath, File optimizedDirectory) {
    ...
}
```

接受之前传进来的包含 dex 的 apk/jar/dex 的路径集、native 库的路径集和缓存优化的 dex 文件的路径，然后调用 `makePathElements()` 方法生成一个 `Element[] dexElements` 数组，Element 是 DexPathList 的一个嵌套类，其有以下字段：

```java
static class Element {
	private final File dir;
	private final boolean isDirectory;
	private final File zip;
	private final DexFile dexFile;
	private ZipFile zipFile;
	private boolean initialized;
}
```

`makePathElements() ` 是如何生成 Element 数组的？继续看源码：

```java
private static Element[] makePathElements(List<File> files, File optimizedDirectory,
                                          List<IOException> suppressedExceptions) {
    List<Element> elements = new ArrayList<>();
    // 遍历所有的包含 dex 的文件
    for (File file : files) {
        File zip = null;
        File dir = new File("");
        DexFile dex = null;
        String path = file.getPath();
        String name = file.getName();
        // 判断是不是 zip 类型
        if (path.contains(zipSeparator)) {
            String split[] = path.split(zipSeparator, 2);
            zip = new File(split[0]);
            dir = new File(split[1]);
        } else if (file.isDirectory()) {
            // 如果是文件夹,则直接添加 Element,这个一般是用来处理 native 库和资源文件
            elements.add(new Element(file, true, null, null));
        } else if (file.isFile()) {
            // 直接是 .dex 文件,而不是 zip/jar 文件(apk 归为 zip),则直接加载 dex 文件
            if (name.endsWith(DEX_SUFFIX)) {
                try {
                    dex = loadDexFile(file, optimizedDirectory);
                } catch (IOException ex) {
                    System.logE("Unable to load dex file: " + file, ex);
                }
            } else {
                // 如果是 zip/jar 文件(apk 归为 zip),则将 file 值赋给 zip 字段,再加载 dex 文件
                zip = file;
                try {
                    dex = loadDexFile(file, optimizedDirectory);
                } catch (IOException suppressed) {
                    suppressedExceptions.add(suppressed);
                }
            }
        } else {
            System.logW("ClassLoader referenced unknown path: " + file);
        }
        if ((zip != null) || (dex != null)) {
            elements.add(new Element(dir, false, zip, dex));
        }
    }
    // list 转为数组
    return elements.toArray(new Element[elements.size()]);
}
```

`loadDexFile()` 方法最终会调用 JNI 层的方法来读取 dex 文件，这里不再深入探究，有兴趣的可以阅读 [从源码分析 Android dexClassLoader 加载机制原理](http://blog.csdn.net/nanzhiwen666/article/details/50515895) 这篇文章深入了解。

接下来看以下 DexPathList 的 `findClass()` 方法，其根据传入的完整的类名来加载对应的 class，源码如下：

```
public Class findClass(String name, List<Throwable> suppressed) {
	// 遍历 dexElements 数组，依次寻找对应的 class，一旦找到就终止遍历
    for (Element element : dexElements) {
        DexFile dex = element.dexFile;
        if (dex != null) {
            Class clazz = dex.loadClassBinaryName(name, definingContext, suppressed);
            if (clazz != null) {
                return clazz;
            }
        }
    }
    // 抛出异常
    if (dexElementsSuppressedExceptions != null) {
        suppressed.addAll(Arrays.asList(dexElementsSuppressedExceptions));
    }
    return null;
} 
```

这里有关于热修复实现的一个点，就是将补丁 dex 文件放到 dexElements 数组前面，这样在加载 class 时，优先找到补丁包中的 dex 文件，加载到 class 之后就不再寻找，从而原来的 apk 文件中同名的类就不会再使用，从而达到修复的目的，虽然说起来较为简单，但是实现起来还有很多细节需要注意，本文先热身，后期再分析具体实现。

至此，BaseDexClassLader 寻找 class 的路线就清晰了：

1. 当传入一个完整的类名，调用 BaseDexClassLader 的 `findClass(String name) ` 方法
2. BaseDexClassLader 的 findClass 方法会交给 DexPathList 的 `findClass(String name, List<Throwable> suppressed ` 方法处理
3. 在 DexPathList 方法的内部，会遍历 dexFile ，通过 DexFile 的 `dex.loadClassBinaryName(name, definingContext, suppressed)` 来完成类的加载

###### 实际使用

需要注意到的是，在项目中使用 BaseDexClassLoader 或者 DexClassLoader 去加载某个 dex 或者 apk 中的 class 时，是无法调用 `findClass()` 方法的，因为该方法是包访问权限，你需要调用 `loadClass(String className)` ，该方法其实是 BaseDexClassLoader 的父类 ClassLoader 内实现的：

```java
public Class<?> loadClass(String className) throws ClassNotFoundException {
    return loadClass(className, false);
}

protected Class<?> loadClass(String className, boolean resolve) throws ClassNotFoundException {
    Class<?> clazz = findLoadedClass(className);
    if (clazz == null) {
        ClassNotFoundException suppressed = null;
        try {
            clazz = parent.loadClass(className, false);
        } catch (ClassNotFoundException e) {
            suppressed = e;
        }
        if (clazz == null) {
            try {
                clazz = findClass(className);
            } catch (ClassNotFoundException e) {
                e.addSuppressed(suppressed);
                throw e;
            }
        }
    }
    return clazz;
}
```

上面这段代码结合之前提到的双亲委托模型就很好理解了，先查找是否已经加载过，如果没有就交给父 ClassLoader 去加载，如果父类加载器没有找到，才调用当前 ClassLoader 来加载，此时就是调用上面分析的 `findClass() ` 方法了。

###  ClassLoader 使用示例

上面说了这么多理论知识，只说不练假把式，接下来实战：从 SD 卡中动态加载一个包含 class.dex 的 jar 文件，加载其中的类，并调用其方法。

1.  新建一个 Java 项目，包含两个文件：`ISayHello.java` 和 `HelloAndroid.java`

```java
   package com.jaeger;

   public interface ISayHello {
       String say();
   }
```

```java
   package com.jaeger;

   public class HelloAndroid implements ISayHello {
       @Override
       public String say() {
           return "Hello Android";
       }
   }
```

2. 导出 jar 包

   这一步使用 IntelliJ IDEA 导出有点问题，最终我是用 Eclipse 导出 jar 包的。

   ![](http://ac-QYgvX1CC.clouddn.com/88ede5c72013c55b.jpg)

3. 使用 SDK 目录 > platform-tools 里面的 dx 工具生成包含 class.dex 的 jar 包

   将上一步生成的 `sayhello.jar` 放到 你的 SDK 下的 platform-tools 文件夹下，使用下面的命令生成 dex 化的 jar 文件，其中是 output 后面的 `sayhello_dex.jar` 就是最终生成的 jar 包。

   ```
   dx --dex --output=sayhello_dex.jar sayhello.jar
   ```

   生成 `sayhello_dex.jar` 之后，用解压解压后就会发现其已经包含了 class.dex 文件了。

   ![](http://ac-QYgvX1CC.clouddn.com/ba0d600fc2a90e2d.jpg)

4. 将 `sayhello_dex.jar` 文件拷贝到手机存储空间的根目录，不一定是内存卡。

   ![](http://ac-QYgvX1CC.clouddn.com/7efba4a5a816a8e1.png)

5. 新建一个 Android 项目，在 MainActivity 中添加如下的代码：

   ```java
   public class MainActivity extends AppCompatActivity {
       private static final String TAG = "TestClassLoader";
       private TextView mTvInfo;
       private Button mBtnLoad;

       @Override
       protected void onCreate(Bundle savedInstanceState) {
           super.onCreate(savedInstanceState);
           setContentView(R.layout.activity_main);

           mTvInfo = (TextView) findViewById(R.id.tv_info);
           mBtnLoad = (Button) findViewById(R.id.btn_load);
           mBtnLoad.setOnClickListener(new View.OnClickListener() {
               @Override
               public void onClick(View view) {
                   // 获取到包含 class.dex 的 jar 包文件
                   final File jarFile =
                       new File(Environment.getExternalStorageDirectory().getPath() + File.separator + "sayhello_dex.jar");
                   
                   // 如果没有读权限,确定你在 AndroidManifest 中是否声明了读写权限
                   Log.d(TAG, jarFile.canRead() + "");

                   if (!jarFile.exists()) {
                       Log.e(TAG, "sayhello_dex.jar not exists");
                       return;
                   }

                   // getCodeCacheDir() 方法在 API 21 才能使用,实际测试替换成 getExternalCacheDir() 等也是可以的
                   // 只要有读写权限的路径均可
                   DexClassLoader dexClassLoader =
                       new DexClassLoader(jarFile.getAbsolutePath(), getExternalCacheDir().getAbsolutePath(), null, getClassLoader());
                   try {
                       // 加载 HelloAndroid 类
                       Class clazz = dexClassLoader.loadClass("com.jaeger.HelloAndroid");
                       // 强转成 ISayHello, 注意 ISayHello 的包名需要和 jar 包中的
                       ISayHello iSayHello = (ISayHello) clazz.newInstance();
                       mTvInfo.setText(iSayHello.say());
                   } catch (ClassNotFoundException e) {
                       e.printStackTrace();
                   } catch (InstantiationException e) {
                       e.printStackTrace();
                   } catch (IllegalAccessException e) {
                       e.printStackTrace();
                   }
               }
           });
       }
   }
   ```

   同时需要新建一个和第一步创建的 Java 项目中包名一致的 `ISayHello` 接口：

   ```java
   package com.jaeger;

   public interface ISayHello {
       String say();
   }
   ```

   这里需要注意几点：

   - 因为需要从存储空间中读取 jar 文件，需要在 AndroidManifest 中声明读写权限
   - ISayHello 接口的包名必须一致
   - `getCodeCacheDir()` 方法在 API 21 才能使用,实际测试替换成 `getExternalCacheDir()` 等也是可以的

6. 接下来就是运行，运行的结果如图，和预期的一样，完美收工。

   ![](http://ac-QYgvX1CC.clouddn.com/4b94e8fbecf66b72.png)

7. 示例代码以及 jar 包上传到 GitHub 了，请前往 [这里](https://github.com/laobie/TestClassLoader) 去查看。

#### 总结

顺着相关资料，从源头开始分析起，然后再进入源码，理清楚具体的运行机制，最终通过简单的示例验证了分析的结果。

在这过程中，查阅了不少资料，也阅读了很多前辈的博文，受益匪浅，这也是现在技术圈内很好的氛围。同时在分析中也发现自己对 Android 底层了解相当薄弱，这也是今后需要多学习的地方。

鉴于自己能力有限，如果本文中有遗漏或者错误的地方，请在评论区指出或者通过邮件等方式联系我，谢谢。

#### 参考资料

[JVM 的工作原理,层次结构以及 GC 工作原理](https://segmentfault.com/a/1190000002579346)

[深入分析Java ClassLoader原理](http://blog.csdn.net/xyang81/article/details/7292380)

[Android动态加载基础 ClassLoader工作机制](https://segmentfault.com/a/1190000004062880)

[各大热补丁方案分析和比较](http://blog.zhaiyifan.cn/2015/11/20/HotPatchCompare/)

[android-dynamical-loading](https://github.com/kaedea/android-dynamical-loading)

[dex分包变形记](http://bugly.qq.com/bbs/forum.php?mod=viewthread&tid=193)

[Android ClassLoader机制](http://blog.csdn.net/mr_liabill/article/details/50497055)

[彻底搞懂 Java ClassLoader](http://weli.iteye.com/blog/1682625)

[从源码分析 Android dexClassLoader 加载机制原理](http://blog.csdn.net/nanzhiwen666/article/details/50515895)

[码农故事](http://www.iloveandroid.net/)