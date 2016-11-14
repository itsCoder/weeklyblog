### 热修复实现：ClassLoader 方式的实现

在之前的文章 [热修复入门：Android 中的 ClassLoader](http://jaeger.itscoder.com/android/2016/08/27/android-classloader.html) 中，讲解了 Android 中的 ClassLoader 工作原理和通过 ClassLoader 实现热修复的可能性。本文结合 [Nuva](https://github.com/jasonross/Nuwa) 项目，来讲讲基于 ClassLoader 方式如何具体实现热修复，阅读本文之前建议先通过前面提到的文章了解下 Android 的 ClassLoader。

#### 实现的几个关键点

在讲解实现思路之前，先回顾下 [热修复入门：Android 中的 ClassLoader](http://jaeger.itscoder.com/android/2016/08/27/android-classloader.html) 文章中提到的几个关键点，这也是 ClassLoader 方式实现热修复的关键：

- 在 Android 中，App 安装到手机后，apk 里面的 class.dex 中的 class 均是通过 PathClassLoader 来加载的。

- DexClassLoader 可以用来加载 SD 卡上加载包含 class.dex 的 .jar 和 .apk 文件

- DexClassLoader 和 PathClassLoader 的基类 BaseDexClassLoader 查找 class 是通过其内部的 `DexPathList pathList` 来查找的

- DexPathList 内部有一个 `Element[] dexElements` 数组，其 `findClass()` 方法（源码如下）的实现就是遍历该数组，查找 class ，一旦找到需要的类，就直接返回，停止遍历：

  ```java
  public Class findClass(String name, List<Throwable> suppressed) {
      for (Element element : dexElements) {
          DexFile dex = element.dexFile;
          if (dex != null) {
              Class clazz = dex.loadClassBinaryName(name, definingContext, suppressed);
              if (clazz != null) {
                  return clazz;
              }
          }
      }
      if (dexElementsSuppressedExceptions != null) {
          suppressed.addAll(Arrays.asList(dexElementsSuppressedExceptions));
      }
      return null;
  }
  ```

#### 实现思路

基于 ClassLoader 方式实现的热修复思路如下图所示：

![](http://ac-QYgvX1CC.clouddn.com/b1c92f1555e7fb4b.jpg)

主要步骤：

1. 假设 MainActivity 中有一个方法`showMsg` ，现在显示的是 “bug” ，需要修复。

   ```java
   public class MainActivity extends AppCompatActivity {
       ...
       public void showMsg() {
           Toast.makeText(this, "bug", Toast.LENGTH_SHORT).show();
       }
   }
   ```

2. 我们修改 `showMsg()` 方法，让其显示正确的结果 “meaasge”。

   ```java
   public void showMsg() {
       Toast.makeText(this, "message", Toast.LENGTH_SHORT).show();
   }
   ```

3. 制作好补丁包，即 patch.jar 文件，该 patch.jar 文件中就包含已经修复了的 dex 文件，注意此时 patch.jar 会包含一个和原来安装 apk 文件中同样的类 `MainActivity` 。

4. 在 Application 的 `onCreate` 方法中检测是否已经下载好补丁包，如果存在补丁包，就通过 DexClassLoader 加载 patch.jar，然后通过反射拿到 DexClassLoader 中的 DexPathList 对象，进而拿到 `Element[] dexElements` 数组，这里标记该 Element 数组为 **newDexElements** 。

5. 还是通过反射，拿到 App 默认的 ClassLoader 即 PathClassLoader 的 DexPathList 对象，进而拿到  Element 数组，这里标记下该数组为 **baseDexElements** 。

6. 将 newDexElements 和 baseDexElements 合成一个新的数组 **allDexElements** ，且保证 newDexElements 中的值在 allDexElements 数组的最前面。

7. 然后还是通过通过反射，将合成的 Element 数组设置给 PathClassLoader 的 DexPathList 对象。

8. 在 Application 完成初始化之后，会开始加载 `MainActivity` ，加载过程就是通过 DexPathList 对象的 `findClass()` 方法来完成的，会从头开始遍历其 Element 数组，会优先查找到之前插入的补丁包中的 dexFile，而原 apk 中的则不会查找到，因此就实现了热修复的目的。


#### 基于 ClassLoader 方式实现需要解决的问题

在对 Nuwa 源码开始解读之前，先说明下在基于 ClassLoader 方式实现热修复需要解决的问题。

- CLASS_ISPREVERIFIED 问题

  odex 文件是 OptimizedDEX 的缩写，表示经过优化的 dex 文件。由于 Android 程序的 apk 文件为 zip 压缩包格式，Dalvik虚拟机每次加载都需要从 apk 中读取 classes.dex 文件，这会耗费很多 cpu 时间，而采用 odex 方式优化的 dex 文件已经包含了加载 dex 必须的依赖库文件列表，Dalvik 虚拟机只需检测并加载所需的依赖库即可执行相应的 dex 文件，大大缩短了读取 dex 文件所需的时间。同时，Android专门提供了一个验证与优化 dex 文件的工具 dexopt，Dalvik 虚拟机在加载一个 dex 文件时，通过指定的验证与优化选项来调用 dexopt 进行相应的验证与优化操作。

  在 dex 优化过程中：

  > 如果某个类直接方法中引用到的类（第一层级关系，不会进行递归搜索）在同一个 dex 中的话，那么这个类就会被打上 **CLASS_ISPREVERIFIED** 标志。

  打上这个标志的类，其引用到的类就只会在该类所在的 dex 中查找，如果没找到，就直接报以下异常：

  ```verilog
  java.lang.IllegalAccessError: Class ref in pre-verified class resolved to unexpected implementation
  ```

  而 ClassLoader 方式实现的热修复，必然需要在 patch.jar 的 dex 文件中查找其他类。为了防止类打上 CLASS_ISPREVERIFIED 标志，我们只需要在每个类中引用一个单独的 dex 中的类即可。这个 dex 我们命名为 hack.dex，其包含一个 `HackLoad.java` ，接下来需要做的就是在除了 Applicaton 类以为的类的默认构造方法中都引用一下 `HackLoad` 类，如下所示：

  ```java
  public class MainActivity extends AppCompatActivity {
      public MainActivity() {
          System.out.println(HackLoad.class);
      }
     
     ...
  }
  ```

  以上插入外部类防止打上 CLASS_ISPREVERIFIED 标志的操作也叫做打桩。

  目前开源的热修复项目插入打桩的代码均是通过 javassist 来实现的，本文这里不做详细介绍了，可以参考一下文章来深入了解：

  - [安卓 App 热补丁动态修复实现 \- 简书](http://www.jianshu.com/p/56facb3732a7)
  - [Java 编程的动态性， 第四部分: 用 Javassist 进行类转换](https://www.ibm.com/developerworks/cn/java/j-dyn0916/)

  > 注：Android 官方增加类的验证过程，并打上 CLASS_ISPREVERIFIED 标志，肯定是为了提升性能和效率的，因此这种解决方案对性能确实存在一定的影响，在微信的 Tinker 方案对比中，也给出了实际的效率对比，差距还是挺大的，因此在使用该方式实现热修复需要了解到这一点。

  ![](http://ac-QYgvX1CC.clouddn.com/04eb03974bad8947.png)

#### Nuva 项目的源码解读

在前面的实现思路分析中，可以说整体思路是比较简单清晰的，按照此思路来，具体的实现其实也不难。接下来就以 Nuwa 项目的源码来解读下具体的实现。

1. 项目结构分析

   ![](http://ac-QYgvX1CC.clouddn.com/473528c66cc757e3.jpg)

   Nuwa 项目的结构如上图所示，可以看出，项目结构并不复杂：

   - `util/AssetUtils.java` Asset 工具类，内部两个方法：复制 Asset 资源和复制文件。
   - `util/DexUtils.java` dex 工具类，主要是实现将 patch.jar 文件中的 dexFile 插入到 PathClassLoader 对应的  Element 数组的前面。
   - `util/ReflectionUtils.java` 反射工具类，实现了两个方法：获取和设置无访问权限域（字段）的值。
   - `Nuwa.java` 项目主类，其包含两个方法：初始化方法，加载补丁方法。

2. Nuva 的实现过程：初始化和加载 dex

   在 Nuwa 项目的使用说明中，需要在 Application 中添加如下代码：

   ```java
         @Override
         protected void attachBaseContext(Context base) {
             super.attachBaseContext(base);
             Nuwa.init(this);
         }
   ```

   直接看 `Nuwa.java` 中的源码：

   ```java
      public class Nuwa {
      
          private static final String TAG = "nuwa";
          private static final String HACK_DEX = "hack.apk";
      
          private static final String DEX_DIR = "nuwa";
          private static final String DEX_OPT_DIR = "nuwaopt";
      
          /**
           * 初始时加载 hack.pak 的 dex 文件,处理打桩
           * @param context
           */
          public static void init(Context context) {
               File dexDir = new File(context.getFilesDir(), DEX_DIR);
               dexDir.mkdir();
       
               String dexPath = null;
               try {
                   dexPath = AssetUtils.copyAsset(context, HACK_DEX, dexDir);
               } catch (IOException e) {
                   Log.e(TAG, "copy " + HACK_DEX + " failed");
                   e.printStackTrace();
               }
       
               loadPatch(context, dexPath);
           }
       
          public static void loadPatch(Context context, String dexPath) {
       
               if (context == null) {
                   Log.e(TAG, "context is null");
                   return;
               }
               if (!new File(dexPath).exists()) {
                   Log.e(TAG, dexPath + " is null");
                   return;
             }
             File dexOptDir = new File(context.getFilesDir(), DEX_OPT_DIR);
             dexOptDir.mkdir();
             try {
                 DexUtils.injectDexAtFirst(dexPath, dexOptDir.getAbsolutePath());
             } catch (Exception e) {
                 Log.e(TAG, "inject " + dexPath + " failed");
                 e.printStackTrace();
             }
         }
      }
   ```
   
   在 `init()` 方法中，通过加载  asset 文件夹中的 hack.apk 文件，将插桩类加载进来，防止之前插桩的那些类报找不到 `HackLoad.class` 异常。这里也可以意识到一点，就是 Application    不应该插桩，否则直接报异常出错。
    
   接下来的 `loadPatch(Context context, String dexPath)` 才是重点，除了在 `init()` 方法中被调用以为，后面加载补丁 patch.jar 时也是使用该方法来加载。其需要两个参数：一个是上下文    context，一个是包含 dex 的 jar 或者 apk 文件的路径。
    
   注意到其中有这么一段代码：
    
   ```java
   File dexOptDir = new File(context.getFilesDir(), DEX_OPT_DIR);
   dexOptDir.mkdir();
   ```
    
   这个得到的是一个存放优化后的 dex 文件的路径，这是 DexClassLoader 类的构造方法所需要的：
    
   ```
   public DexClassLoader(String dexPath, String optimizedDirectory,
           String libraryPath, ClassLoader parent) {
       super(dexPath, new File(optimizedDirectory), libraryPath, parent);
   }
   ```
   
   - `String optimizedDirectory` : 用来缓存优化的 dex 文件的路径，即从 apk 或 jar 文件中提取出来的 dex 文件。该路径不可以为空，且应该是应用私有的，有读写权限的路径（实际上也可以使用外部存储空间，但是这样的话就存在代码注入的风险）。
   
   关于 DexClassLoader 的其他细节，可以阅读本文开头提到的那篇文章。
   
   接下来就是调用 `DexUtils.injectDexAtFirst()` 方法，看该方法的名称就可以知道，是将对应的 dex 注入到所有的 dex 的最前面。

3. 注入补丁的 dex 

   注入补丁的过程主要在 DexUtil 类中：

   ```java
   public class DexUtils {

       public static void injectDexAtFirst(String dexPath, String defaultDexOptPath) throws NoSuchFieldException, IllegalAccessException, ClassNotFoundException {
           DexClassLoader dexClassLoader = new DexClassLoader(dexPath, defaultDexOptPath, dexPath, getPathClassLoader());
           Object baseDexElements = getDexElements(getPathList(getPathClassLoader()));
           Object newDexElements = getDexElements(getPathList(dexClassLoader));
           Object allDexElements = combineArray(newDexElements, baseDexElements);
           Object pathList = getPathList(getPathClassLoader());
           ReflectionUtils.setField(pathList, pathList.getClass(), "dexElements", allDexElements);
       }

       private static PathClassLoader getPathClassLoader() {
           PathClassLoader pathClassLoader = (PathClassLoader) DexUtils.class.getClassLoader();
           return pathClassLoader;
       }

       private static Object getDexElements(Object paramObject)
               throws IllegalArgumentException, NoSuchFieldException, IllegalAccessException {
           return ReflectionUtils.getField(paramObject, paramObject.getClass(), "dexElements");
       }

       private static Object getPathList(Object baseDexClassLoader)
               throws IllegalArgumentException, NoSuchFieldException, IllegalAccessException, ClassNotFoundException {
           return ReflectionUtils.getField(baseDexClassLoader, Class.forName("dalvik.system.BaseDexClassLoader"), "pathList");
       }

       private static Object combineArray(Object firstArray, Object secondArray) {
           Class<?> localClass = firstArray.getClass().getComponentType();
           int firstArrayLength = Array.getLength(firstArray);
           int allLength = firstArrayLength + Array.getLength(secondArray);
           Object result = Array.newInstance(localClass, allLength);
           for (int k = 0; k < allLength; ++k) {
               if (k < firstArrayLength) {
                   Array.set(result, k, Array.get(firstArray, k));
               } else {
                   Array.set(result, k, Array.get(secondArray, k - firstArrayLength));
               }
           }
           return result;
       }
       
   }
   ```

   结合上文实现思路的分析，`injectDexAtFirst()` 方法的流程是很清晰的：
   - 通过 `DexClassLoader` 加载补丁中的 dex 文件，然后反射得到新的 Element 集合—— `newDexElements` ；
   - 拿到 `PathClassLoader` 中的 Element 集合—— `baseDexElements` ；
   - 将 `newDexElements` 和 `baseDexElements` 组合成整个的 Element 组合，组合是放在 `combineArray` 方法中执行的，看看其具体的实现，就可以发现会优先将 newDexElements 中的值放在合成数组的最前面，这也是之前所提到的实现热修复的关键点之一。
   - 将合成后的 `allDexElements` 设置给 PathClassLoader 的 DexPathList 对应的 Element 数组。

   反射工具类的源码如下：

   ```java
   public class ReflectionUtils {
       public static Object getField(Object obj, Class<?> cl, String field)
               throws NoSuchFieldException, IllegalArgumentException, IllegalAccessException {
           Field localField = cl.getDeclaredField(field);
           localField.setAccessible(true);
           return localField.get(obj);
       }
       public static void setField(Object obj, Class<?> cl, String field, Object value)
               throws NoSuchFieldException, IllegalArgumentException, IllegalAccessException {
           Field localField = cl.getDeclaredField(field);
           localField.setAccessible(true);
           localField.set(obj, value);
       }
   }
   ```

   关于反射，你可以通过 [Java基础与提高干货系列——Java反射机制 \- 简书](http://www.jianshu.com/p/1a60d55a94cd) 来了解，本文就不多做探讨了。

   至此，Nuva 的关键代码均解读完毕，就该项目而言，代码量并不多，但是整个实现的思路是很巧妙很清晰的，这也是该项目的关键之处。


#### 后续内容
- 在接下来的系列文章中，还会结合 Nuva 项目，介绍下补丁包 patch.jar 的生成操作。
- 由于本文时间较为仓促，后续会补上实践过程和代码。
   ​
#### 参考资料
- [绕过 Dalvik 验证技术分析](http://blog.sina.com.cn/s/blog_71338cc10102uwgt.html)
- [安卓 App 热补丁动态修复实现 \- 简书](http://www.jianshu.com/p/56facb3732a7)





