> 转载请注明出处：[多进程中安全的使用SharedPreferences](http://melodyxxx.com/2016/08/04/%E5%A4%9A%E8%BF%9B%E7%A8%8B%E4%B8%AD%E5%AE%89%E5%85%A8%E7%9A%84%E4%BD%BF%E7%94%A8SharedPreferences/) 



### 一、一般情况下多进程中直接使用SharedPreferences的影响

SharedPreferences是Android中一种轻量级的存储解决方案，底层采用的是XML文件并且通过键值对的方法来管理数据，所以一般用来存储一些App的配置文件和一些轻量的数据。SharedPreferences使用起来也比较方便简单，但是在多进程情况下使用，SharedPreferences就并不显得那么友好了。在多进程中使用SharedPreferences可能会造成读取到数据为脏数据，还有可能在读写的时候造成数据的丢失。

下面分析一下原因：

系统对SharedPreferences的读写是有一定的缓存机制的，通俗点意思就是操作SharedPreferences时，每个进程都会有一份它的缓存，你对它的操作都会先写到缓存内，然后系统会在合适的时机将缓存里的数据写到文件内。所以在多进程环境中，每个进程都会有一份其缓存，所以在另一个进程中操作了SharedPreferences，另一个进程并不能及时更新数据，这样在读取数据的时候极有可能读取到的不是最新的值，依旧是旧的数据。


我也看到过网上的解决方案，将MODE_PRIVATE改成MODE_WORLD_READABLE或者MODE_WORLD_WRITEABLE之类的，我也尝试过，但还是失败，这两个参数在API 17开始就被废弃了，并且在Android N开始使用这两个参数会抛出安全异常，下面是[官方的文档](https://developer.android.com/reference/android/content/Context.html#MODE_WORLD_READABLE)：

> Note: The constants MODE_WORLD_READABLE and MODE_WORLD_WRITEABLE have been deprecated since API level 17
> **As of N attempting to use this mode will throw a SecurityException.**


### 二、解决方案

前面也也说道在多进程环境下使用SharedPreferences会不安全，所以我们要做的就是把所有对SharedPreferences的操作放在一个进程，其他进程的想要对SharedPreferences操作，则将他们的操作全部转移到这个进程，所以这样使用SharedPreferences就也没什么问题了。

所以需要用到跨进程传递数据了，Android中的跨进程通信(IPC)的解决方案也有很多，例如使用AIDL、Messenger、ContentProvider、文件共享机制、Socket通信等。AIDL使用起来相对其他的方案比较麻烦。。

**解决方案：**
**使用ContentProvider封装SharedPreferences的所有操作,ContentProvider的底层使用就是AIDL，只不过ContentProvider已经为我们做了很好的封装了**

下面是大致的实现流程图：

![流程图](http://i.imgur.com/yuV0Gbl.png)


### 三、具体实现

这里就不介绍ContentProvider的基本使用用法了，需要说明的是这里用不到ContentProvider的query、getType、insert、delete、update这几个方法，因为这几个方法是系统方便我们用ContentProvider封装数据库的操作的，显然，我们这里是自己封装SharedPreferences，不需要用到数据库，所以这几个方法用不到。而我们需要用到它的call方法，下面看call方法的参数：

``` java
@Override
public Bundle call(String method, String arg, Bundle extras) {
	return null;
}
```
参数列表：
- method: 根据这个method的值，我们就可以知道调用方想要执行什么操作
- arg: 这个参数可以让调用发传递简单的String类型的数据过来，要想传递其他类型的数据需要使用到第三个参数Bundle
- extras: 上面已经说了，可以让调用方传递一些数据过来，因为涉及到IPC操作，所以需要使用Bundle

这里我们首先创建一个BasePreferencesProvider抽象类继承ContentProvider，内部先空实现query、getType、insert、delete、update这个几个方法，因为在本例中这几个方法用不到，所以为了主ContentProvider类的结构清晰，所以这里先空实现这几个方法.

``` java
public abstract class BasePreferencesProvider extends ContentProvider {
    @Nullable
    @Override
    public Cursor query(Uri uri, String[] projection, String selection, String[] selectionArgs, String sortOrder) {
        return null;
    }

    @Nullable
    @Override
    public String getType(Uri uri) {
        return null;
    }

    @Nullable
    @Override
    public Uri insert(Uri uri, ContentValues values) {
        return null;
    }

    @Override
    public int delete(Uri uri, String selection, String[] selectionArgs) {
        return 0;
    }

    @Override
    public int update(Uri uri, ContentValues values, String selection, String[] selectionArgs) {
        return 0;
    }
}
```

然后创建我们的PreferencesProvider继承上面的BasePreferencesProvider，PreferencesProvider在里面封装了对SharedPreferences的所有操作。当然，这里为了简单，只实现了String和boolean的实现，其他类型的操作实现类似。

``` java
public class PreferencesProvider extends BasePreferencesProvider {

    // putString()方法标识
    public static final String METHOD_PUT_STRING = "put_string";
    // getString()方法标识
    public static final String METHOD_GET_STRING = "get_string";
    // putBoolean()方法标识
    public static final String METHOD_PUT_BOOLEAN = "put_boolean";
    // getBoolean()方法标识
    public static final String METHOD_GET_BOOLEAN = "get_boolean";

    public static final String EXTRA_KEY = "key";
    public static final String EXTRA_VALUE = "value";
    public static final String EXTRA_DEFAULT_VALUE = "default_value";

    private SharedPreferences mPreferences;

    @Override
    public boolean onCreate() {
        // Provider创建时获取SharedPreferences
        mPreferences = getContext().getSharedPreferences("app_config", Context.MODE_PRIVATE);
        return false;
    }

    @Nullable
    @Override
    public Bundle call(String method, String arg, Bundle extras) {
        // 用于将数据返回给调用方，例如getString()、getBoolean()
        Bundle replyData = null;
        switch (method) {
            case METHOD_PUT_STRING: {
                String key = extras.getString(EXTRA_KEY);
                String value = extras.getString(EXTRA_VALUE);
                // 将值存起来 - putString()
                mPreferences.edit().putString(key, value).commit();
                break;
            }
            case METHOD_GET_STRING: {
                String key = extras.getString(EXTRA_KEY);
                String defValue = extras.getString(EXTRA_DEFAULT_VALUE);
                // 获取到的值 - getString()
                String value = mPreferences.getString(key, defValue);
                replyData = new Bundle();
                // 将获取到的值放进Bundle
                replyData.putString(EXTRA_VALUE, value);
                break;
            }
            case METHOD_PUT_BOOLEAN: {
                String key = extras.getString(EXTRA_KEY);
                boolean value = extras.getBoolean(EXTRA_VALUE);
                // 将值存起来 - putBoolean()
                mPreferences.edit().putBoolean(key, value).commit();
                break;
            }
            case METHOD_GET_BOOLEAN: {
                String key = extras.getString(EXTRA_KEY);
                boolean defValue = extras.getBoolean(EXTRA_DEFAULT_VALUE);
                // 获取到的值 - getBoolean()
                boolean value = mPreferences.getBoolean(key, defValue);
                replyData = new Bundle();
                replyData.putBoolean(EXTRA_VALUE, value);
                break;
            }
        }
        // 将获取到的值返回给调用方，若为put操作，replyData则为null
        return replyData;
    }
}
```

上面可以看到，我们首先定义了4个Method操作方法标识。在执行call方法内，根据调用方传进来的method的值，来执行对应的操作。上面例子中只实现了putString()、getString()、putBoolean()、getBoolean()这四个操作，其他类似int等操作实现一模一样。

编写完Provider后，不要忘记在AndroidManifest.xml文件中注册我们的Provider
``` xml
<provider
    android:name=".provi.PreferencesProvider"
    android:authorities="com.melodyxxx.sharedpreferencesdemo.sp"/>
```
这样注册Provider后，Provider是运行在主进程的，也可以指定让其运行在其他进程，如下：
``` xml
<provider
    android:name=".provi.PreferencesProvider"
    android:authorities="com.melodyxxx.sharedpreferencesdemo.sp"
    android:process=":remote"/>
```
这样就讲其指定运行在私有的remote进程了


最后为了方便调用方调用，我们还需要创建PreferencesUtils类再封装一层操作：

``` java
public class PreferencesUtils {

    private static final Uri sUri = Uri.parse("content://com.melodyxxx.sharedpreferencesdemo.sp");

    public static void putString(Context context, String key, String value) {
        Bundle data = new Bundle();
        data.putString(PreferencesProvider.EXTRA_KEY, key);
        data.putString(PreferencesProvider.EXTRA_VALUE, value);
        context.getContentResolver().call(sUri, PreferencesProvider.METHOD_PUT_STRING, null, data);
    }

    public static String getString(Context context, String key, String defValue) {
        String value = null;
        Bundle data = new Bundle();
        data.putString(PreferencesProvider.EXTRA_KEY, key);
        data.putString(PreferencesProvider.EXTRA_DEFAULT_VALUE, defValue);
        Bundle replyData = context.getContentResolver().call(sUri, PreferencesProvider.METHOD_GET_STRING, null, data);
        return replyData.getString(PreferencesProvider.EXTRA_VALUE);
    }

    public static void putBoolean(Context context, String key, boolean value) {
        Bundle data = new Bundle();
        data.putString(PreferencesProvider.EXTRA_KEY, key);
        data.putBoolean(PreferencesProvider.EXTRA_VALUE, value);
        context.getContentResolver().call(sUri, PreferencesProvider.METHOD_PUT_BOOLEAN, null, data);
    }

    public static boolean getBoolean(Context context, String key, boolean defValue) {
        Bundle data = new Bundle();
        data.putString(PreferencesProvider.EXTRA_KEY, key);
        data.putBoolean(PreferencesProvider.EXTRA_DEFAULT_VALUE, defValue);
        Bundle replyData = context.getContentResolver().call(sUri, PreferencesProvider.METHOD_GET_BOOLEAN, null, data);
        return replyData.getBoolean(PreferencesProvider.EXTRA_VALUE);
    }

}
```

上面的封装也是很简单的，注意Uri不要指定错了。同样也写了对应的putString()、getString()、putBoolean()、getBoolean()、内部都是根据指定的Uri调用了PreferencesProvider的call方法。在多进程情况下时，这里调用是跨进程的，所有对SharedPreferences的操作最终都会在PreferencesProvider所在的remote进程中完成，从而保证了SharedPreferences读写的安全性，保证了在各个进程读取到的数据是正确的。

最后，就是验证了，我将此例子放到自己的项目中运行测试了下，在运行在主进程中的Activity中通过PreferencesUtils修改值后，然后另一个在其他进程中的Service立马通过PreferencesUtils将值取出打印：

put操作:
```
Log.d("sp_test", "Put String :  " + namesList.get(position));
PreferencesUtils.putString(LiveWallpaperSettingsActivity.this, "weather_type", namesList.get(position));
```

get操作:
```
String weather_type = PreferencesUtils.getString(context, "weather_type", "unknown");
Log.d("sp_test", "Get String :  " + weather_type);
```


最后运行看打印的Log:

![](http://i.imgur.com/ixQq3FF.png)

根据Log打印的PID可以看到这里put和get操作不在同一进程，取出值是正确的，所以这种利用ContentProvider封装SharedPreferences是可行的。

### 四、最后：

本文只探讨的是SharedPreferences在多进程中如何安全的使用，当然在多进程也可以不使用SharedPreferences，例如可以使用数据库来存储配置文件和其他的数据，方法也有很多。由于技术有限，文中难免写的不对地方，也欢迎指正。
