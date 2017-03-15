---
title: 学习总结-- Android 多进程和 6 种 IPC 方式
date: 2017-02-27 21:38:32
category: 学习笔记
tags: IPC

---

>- 文章来源：itsCoder 的 [WeeklyBolg](https://github.com/itsCoder/weeklyblog) 项目
>- itsCoder主页：[http://itscoder.com/](http://itscoder.com/)
>- 作者：[ljuns](https://ljuns.github.io/)
>- 审阅者：[]()

之前对于 IPC 机制真的是完全懵逼啊，根本就不知道 IPC 是个什么东西，通过这些天的学习算是有了一定的了解，但也仅限于使用方式，对于原理的东西还是很懵逼的，所以先来个 IPC 方式的总结（对原理还没有理解透彻，这里只是用法的小结）。

IPC 是 Inter-Process Communication 的缩写，含义为进程间通信或者跨进程通信，是指两个进程之间进行数据交换的过程。在说到 IPC 之前有必要简单介绍一下 Android 中的多进程模式。
## Android 中的多进程模式
### 开启多进程模式
在 Android 中使用多进程一般来说只有两种方法：给四大组件(Activity、Service、Receiver、ContentProvider)在 AndroidMenifest 中指定 android:process 属性；另一种非常规的方法是通过 JNI 在 native 层去 fork 一个新的进程，这种方法比较特殊，先不考虑。

默认情况(也就是没有为某个组件指定 android:process 属性的情况)下，此时的进程名是包名。通常情况下，进程可以分为私有进程和全局进程：进程名以“：”开头的进程属于当前应用的私有进程，其他应用的组件不可以和它在同一个进程中；进程名不以“：”开头的进程属于全局进程，其他应用通过 ShareUID 方式可以和它在同一个进程中。

### 多进程造成的问题
开启多进程很简单，通过为四大组件指定 android:process 属性即可，但是在实际开发中可能会遇到各种问题，总结如下：

1. 静态成员和单例模式完全失效；
2. 线程同步机制完全失效；
3. SharedPreferences 的可靠性下降；
4. Application 会多次创建。

简单的分析一下出现这几个问题的原因，第 1 个问题：Android 为每个应用分配了一个独立的虚拟机，或者说为每个进程都分配了一个独立的虚拟机，不同的虚拟机在内存的分配上有不同的地址空间，这就导致在不同的虚拟机中访问同一个对象会产生多份副本。

第 2 个问题本质上和第 1 个问题是类似的，既然不是同一块内存，那么不管锁对象还是锁全局类都无法保证线程同步，因为不同进程锁的不是同一个对象。

第 3 个问题是因为 SharedPreferences 不支持两个进程同时去执行写操作，否则会导致一定几率的数据丢失，这是因为不同的进程都有一份 SharedPreferences 的内存缓存。

第 4 个问题：运行在同一个进程中的组件是属于同一个虚拟机和同一个 Application，也就是说运行在不同进程中的组件是属于两个不同的虚拟机和 Application 的。

大概分析了下多进程带来的问题，接下来就要学习如何解决这些问题。

## Android 中的 IPC 方式
### 使用 Bundle 
四大组件中的三大组件（Activity、Service、Receiver）都支持在 Intent 中传递 Bundle 数据，由于 Bundle 实现了 Parcelable 接口，所以它可以方便地在不同的进程间传输。当在一个进程中启动另一个进程的 Activity 、Service 和 Receiver 时，可以在 Bundle 中附加需要传输给远程进程的信息，并通过 Intent 发送出去。

**注意：传输的数据必须能够被序列化，比如基本类型、实现了 Parcelabel 接口的对象、实现了 Serializable 接口的对象以及一些 Android 支持的特殊对象**

### 使用文件共享
共享文件也是一种不错的进程间通信方式，两个进程通过读\写同一个文件来交换数据，由于 Android 系统基于 Linux ，使得其并发读\写文件可以没有限制地进行。下面将模拟这么个例子：在 MainActivity 中序列化一个 User 对象到 sd 卡上，然后在 SecondActivity 中去反序列化。首先创建一个 User 类，并实现 Serializable 进行序列化(使用 Parcelable 进行序列化也是可以的)，如果不懂什么是序列化，自己 Google 去。

``` java
public class User implements Serializable{

    private static final long serialVersionUID = 1L;

    private int userId;
    private String userName;
    private boolean isMale;

    public User(int userId, String userName, boolean isMale) {
        this.userId = userId;
        this.userName = userName;
        this.isMale = isMale;
    }

   // 以下省略 get()、set()、toString() 方法
   ...
}
```

然后在 MainActivity 中进行保存(MyConstants.BASE_PATH：是 sd 卡的存储路径，`BASE_PATH = Environment.getExternalStorageDirectory()
.getPath() + "/ljuns/basepath/"`)：

``` java
private void persistToFile() {
    new Thread(new Runnable() {
        @Override
        public void run() {
            // 创建基本目录
            File dir = new File(MyConstants.BASE_PATH);
            if (!dir.exists()) {
                // 创建多级目录
                dir.mkdirs();
            }
            // 文件存储目录
            File cacheFile = new File(MyConstants.CACHE_FILE_PATH);
            try {
                ObjectOutputStream out = new ObjectOutputStream(
                        new FileOutputStream(cacheFile));
                User user = new User(1, "ljuns", true);
                Log.d("ljuns", "run1: " + user.toString());
                out.writeObject(user);
            } catch (IOException e) {
                e.printStackTrace();
            } finally {
                MyUtils.close(out);
            }
        }
    }).start();
}
```

最后在 SecondActivity 中恢复数据(`CACHE_FILE_PATH = BASE_PATH + "usercache"`)：

``` java
private void recoverFromFile() {
    new Thread(new Runnable() {
        @Override
        public void run() {
            // 文件存储路径
            File cacheFile = new File(MyConstants.CACHE_FILE_PATH);
            if (cacheFile.exists()) {
                try {
                    ObjectInputStream in = new ObjectInputStream(
                            		new FileInputStream(cacheFile));
                    User user = (User) in.readObject();
                    Log.d("ljuns", "run2: " + user.toString());
                } catch (IOException e) {
                    e.printStackTrace();
                } catch (ClassNotFoundException e) {
                    e.printStackTrace();
                } finally {
                    MyUtils.close(in);
                }
            }
        }
    }).start();
}
```

通过上面的步骤的运行结果可以发现两个 user 对象的内容是相同的，本质上还是两个对象。

__还有一点需要注意:__

>SharedPreferences 是 Android 提供的轻量级存储方案，它通过键值对的方式来存储数据，在底层实现上采用了 XML 文件来存储键值对。从本质上来说 SharedPreferences 也属于文件的一种，但是由于系统对它的读\写有一定的缓存策略，即在内存中会有一份 SharedPreferences 文件的缓存，因此在多进程模式下系统对它的读\写就变得不可靠，当面对高并发的读\写有很大几率会丢失数据，所以不建议在进程间通信中使用 SharedPreferences 。

### 使用 Messenger
暂且不深究 Messenger 是个什么东东，只需知道可以通过它在不同进程中传递 Message 对象来实现数据的进程间传递。直接来看看它的用法：

首先需要在服务端创建一个 Service 来处理客户端的连接请求，同时创建一个 Handler 对象用来接收客户端发送的消息，然后通过 Handler 对象创建一个 Messenger 对象，在 Service 的 onBind() 方法中返回这个 Messenger 对象底层的 Binder，客户端通过该 Messenger 给服务端发送消息。

``` java
public class MessengerService extends Service {

    private static class MessengerHandler extends Handler {
        @Override
        public void handleMessage(Message msg) {
            switch (msg.what) {
                case 0:
     				// 接受客户端发送的消息
                    Log.d("ljuns", "service: " + msg.getData().getString("msg"));

                    // 给客户端返回消息
                    Messenger messenger = msg.replyTo;
                    Message message = Message.obtain(null, 1);
                    Bundle bundle = new Bundle();
                    bundle.putString("reply", "嗯，你的信息已收到");
                    message.setData(bundle);
                    try {
                        messenger.send(message);
                    } catch (RemoteException e) {
                        e.printStackTrace();
                    }
                    break;
                default:
                    super.handleMessage(msg);
            }
        }
    }

    // 创建 Messenger 对象
    private final Messenger messenger = 
    							new Messenger(new MessengerHandler());

    @Override
    public IBinder onBind(Intent intent) {
        return messenger.getBinder();
    }
}
```

服务端通过 `Messenger messenger = msg.replyTo` 取出客户端用来接收消息的 Messenger 对象，再通过该 Messenger 对象给客户端发送消息。为什么可以通过 `msg.replyTo` 获取到一个 Messenger 对象呢？那是因为在客户端中把一个接收消息的 Messenger 对象放入 Message 的 replyTo 参数中，所以才能在这通过 replyTo 参数取出来。

不要忘了还需对这个 Service 进行注册，让其运行在单独的进程中：

``` xml
<service
    android:name=".messenger.MessengerService"
    android:process=":remote" />
```

客户端的实现：首先需要绑定远程进程的 MessengerService，绑定成功后根据返回的 binder 对象创建 Messenger 对象，该 Messenger 对象正是服务端用来接收消息的，在客户端中使用该 Messenger 对象向服务端发送消息。

``` java
public class MessengerActivity extends AppCompatActivity {

    private Messenger mMessenger;
    private Messenger mReplyMessenger = 
    						new Messenger(new MessengerHandler());

    /**
     * 接收消息
     */
    private class MessengerHandler extends Handler {
        @Override
        public void handleMessage(Message msg) {
            switch (msg.what) {
                case 1:
                    Log.d("ljuns", "client: " + msg.getData().getString("reply"));
                    break;
                default:
                    super.handleMessage(msg);
            }
        }
    }

    /**
     * 发送消息
     */
    private ServiceConnection connection = new ServiceConnection() {
        @Override
        public void onServiceConnected(ComponentName componentName, IBinder binder) {
            // 创建 Messenger 对象
            mMessenger = new Messenger(binder);
            Message msg = Message.obtain(null,0);
            Bundle data = new Bundle();
            data.putString("msg", "Hello, this is client");
            msg.setData(data);

            msg.replyTo = mReplyMessenger;
            try {
                // 发送消息
                mMessenger.send(msg);
            } catch (RemoteException e) {
                e.printStackTrace();
            }
        }

        @Override
        public void onServiceDisconnected(ComponentName componentName) {}
    };

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_messenger);

        // 绑定 Service
        Intent intent = new Intent(this, MessengerService.class);
        bindService(intent, connection, Service.BIND_AUTO_CREATE);
    }

    @Override
    protected void onDestroy() {
        unbindService(connection);
        super.onDestroy();
    }
}
```

`msg.replyTo = mReplyMessenger` 正是这句代码把客户端用来接收消息的 Messenger 通过 Message 的 replyTo 参数传递给服务端。运行程序打印 log 就会发现一个客户端与服务端进行通信的小例子就完成了，而且是跨进程进行通信。总结一下上面的客户端与服务端的交互过程：

1. 服务端接收消息的 Messenger 对象会通过 onBind() 方法返回给客户端，客户端取出该 Messenger 对象并通过该 Messenger 对象给服务端发送消息；
2. 客户端接收消息的 Messenger 对象会通过 Message 的参数 replyTo 传递给服务端，服务端取出该 Messenger 对象并用该 Messenger 对象给客户端发送消息。

### 使用 AIDL
前面的 Messenger 的作用主要是为了传递消息，如果需要跨进程调用服务端的方法，那就需要用到 AIDL 了，但是 AIDL 并不支持所有的数据类型，仅支持如下类型：

> * 基本数据类型(int、long、char、boolean、double等)；
* String 和 CharSequence；
* List：只支持 ArrayList，每个元素都必须能够被 AIDL 支持；
* Map：只支持 HashMap，里面每个元素都必须能够被 AIDL 支持，包括 key 和 value；
* Parcelable：所有实现了 Parcelable 接口的对象；
* AIDL：所有的 AIDL 接口本身也可以在 AIDL 文件中使用。

下面来看看如何使用 AIDL 进行进程间的通信。

AIDL 接口：首先创建了一个后缀为 .aidl 的文件：IBookManager.aidl，在里面声明了一个接口和两个方法，这两个方法是暴露给客户端的：

``` java
package cn.ljuns.androidgrowing.aidl;
import cn.ljuns.androidgrowing.aidl.Book;

interface IBookManager {
    List<Book> getBookList();
    void addBook(in Book book);
}
```

__注意:__

1. 在 IBookManager.aidl 中用到了 Book 这个类，这个类实现了 Parcelable 接口并且和 IBookManager.aidl 位于同一个包中，由于 AIDL 的规范，这里必须手动调用 `import cn.ljuns.androidgrowing.aidl.Book;` 进行导入。
2. 如果 AIDL 文件中用到了自定义的 Parcelable 对象，那么必须新建一个和该对象同名的 AIDL 文件，并在其中声明为 Parcelable 类型。在 IBookManager.aidl 中用到了自定义的 Parcelable 对象（Book），所以此时还需创建 Book.aidl：

``` java
package cn.ljuns.androidgrowing.aidl;
parcelable Book;
```

AIDL 接口创建好了，接下来到服务端了，首先要创建一个 Service 用来监听客户端的连接请求，然后实现创建好的 AIDL 接口：

``` java
public class AIDLService extends Service {

    private static final String TAG = "ljuns";

    // CopyOnWriteArrayList 支持并发读\写
    private CopyOnWriteArrayList<Book> bookList = 
    									new CopyOnWriteArrayList<>();

    // 实现 IBookManager.Stub 接口
    private IBinder binder = new IBookManager.Stub() {

        @Override
        public List<Book> getBookList() throws RemoteException {
            return bookList;
        }

        @Override
        public void addBook(Book book) throws RemoteException {
            bookList.add(book);
        }
    };

    @Override
    public void onCreate() {
        super.onCreate();
        bookList.add(new Book(1, "Android"));
        bookList.add(new Book(2, "iOS"));
    }

    public IBinder onBind(Intent intent) {
        return binder;
    }
}
```

在这里使用了 CopyOnWriteArrayList 是因为它支持并发读\写。AIDL 中所支持的是抽象的 List，虽然此时返回的是 CopyOnWriteArrayList，但是在 Binder 中会按照 List 的规范去访问数据并最终形成一个新的 ArrayList 传递给客户端，所以上面有说到 AIDL 只支持 ArrayList 和这里的 CopyOnWriteArrayList 并不矛盾。

客户端的实现就比较简单了，首先是绑定服务端，并在绑定成功后将服务端返回的 Binder 对象转换成 AIDL 接口，然后通过该 Binder 对象就可以去调用服务端的方法了：

``` java
public class AIDLActivity extends AppCompatActivity {

    private static final int MESSAGE_NEW_BOOK_ARRIVED = 1;

    private ServiceConnection connection = new ServiceConnection() {
        /**
         * 绑定成功
         * @param componentName
         * @param iBinder
         */
        @Override
        public void onServiceConnected(ComponentName componentName, IBinder iBinder) {

            // 将服务端返回的 Binder 对象转成 AIDL 接口所属的类型
            IBookManager manager = IBookManager.Stub.asInterface(iBinder);
            mManager = manager;
            // 调用远程服务端的方法
            try {
                List<Book> bookList = manager.getBookList();
                log(bookList);

                manager.addBook(new Book(3, "Android 开发艺术探索"));
                log(manager.getBookList());
            } catch (RemoteException e) {
                e.printStackTrace();
            }
        }

        @Override
        public void onServiceDisconnected(ComponentName componentName) {}
    };

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_aidl);
        // 绑定 Service
        Intent intent = new Intent(this, AIDLService.class);
        bindService(intent, connection, Service.BIND_AUTO_CREATE);
    }

    @Override
    protected void onDestroy() {
        unbindService(connection);
        super.onDestroy();
    }

    /**
     * 打印
     * @param list：数据源
     */
    private void log(List<Book> list) {
        for (Book book : list) {
            Log.d("ljuns", "id: " + book.bookId + ", name: " + book.bookName);
        }
    }
}
```

使用 AIDL 跨进程通信的基本用法就是这样的，还有很多比较复杂的用法就不细说，有兴趣的自己找找相关的资料。

### 使用ContentProvider
ContentProvider 是 Android 中提供的专门用于不同应用间进行数据共享的方式，它天生适合进程间通信。

ContentProvider 主要以表格的形式来组织数据，并且可以包含多张表，对于每个表格来说，它们都具有行和列的层次性，行往往对应一条记录，而列对应一条记录中的一个字段。ContentProvider 对底层的数据存储方式没有任何要求，既可以使用 SQLite 数据库，也可以使用普通文件，甚至可以采用内存中的一个对象来进行数据的存储。

ContentProvider 属于 Android 四大组件之一，关于用法不再细说，有一点需要注意：在注册 ContentProvider 的时候需要给它设置 **android:authorities** 属性，该属性是 ContentProvider 的唯一标识，通过这个属性外部应用就可以访问本应用的 ContentProvider，因此 android:authorities 必须是唯一的。

### 使用 Socket

>Socket 也称为“套接字”，分为流式套接字和用户数据报套接字两种，分别对应于网络的传输控制层中的 TCP 和 UDP 协议。TCP 协议是面向连接的协议，提供稳定的双向通信功能，TCP 连接的建立需要经历“三次握手”才能完成，为了提供稳定的数据传输功能，其本身提供了超时重传机制，因此具有很高的稳定性。

>UDP 是无连接的，提供不稳定的单向通信功能，在性能上具有更好的效率，其缺点是不保证数据一定能够正确传输，尤其是在网络拥塞的情况下。

下面演示一个跨进程的聊天程序，仅仅是演示跨进程的通信：

![](http://oaydqd1yy.bkt.clouddn.com/gif/Kapture%202017-03-02%20at%2015.56.29.gif)

接下来看看具体的实现，首先使用 Socket 进行通信，需要声明权限：

 ``` xml
 <uses-permission android:name="android.permission.INTERNET" />
 <uses-permission android:name="android.permission.ACCESS_NETWORK_STATE" />
 ```

服务端：当 Service 启动时新建一个线程并在这个线程中建立 TCP 服务，然后就等待客户端的连接请求。当有客户端连接时就会生成一个新的 Socket，通过 Socket 就可以实现客户端和服务端通信了。当服务端输入流的返回值是 null 时说明客户端断开了连接，此时服务端也会关闭对应的 Socket 并结束通信。

服务端的全部代码（已经做了详细注释）：

``` java
public class SocketService extends Service {

    private boolean mIsServiceDestory = false;
    // 返回消息集合
    private String[] mDefinedMessages = new String[]{
            "你好啊",
            "你叫啥？",
            "你知道吗？我可是可以和多个人同时聊天的哦",
            "给你讲个笑话：据说爱笑的人运气不会太差，不知道真假"
    };

    @Override
    public void onCreate() {
        new Thread(new TcpService()).start();
        super.onCreate();
    }

    @Override
    public IBinder onBind(Intent intent) {
        return null;
    }

    @Override
    public void onDestroy() {
        super.onDestroy();
        mIsServiceDestory = true;
    }

    /**
     * 开启线程建立 TCP 服务
     */
    private class TcpService implements Runnable {

        @SuppressWarnings("resource")
        @Override
        public void run() {
            ServerSocket serverSocket = null;
            try {
                // 建立 TCP 服务，监听 8688 端口
                serverSocket = new ServerSocket(8688);
            } catch (IOException e) {
                System.out.println("establish tcp server failed, port: 8688");
                e.printStackTrace();
                return;
            }
            // 等待客户端连接请求
            while (!mIsServiceDestory) {
                try {
                    // 当客户端连接时生成一个新的 Socket
                    final Socket clientSocket = serverSocket.accept();
                    System.out.println("accept");
                    new Thread(new Runnable() {
                        @Override
                        public void run() {
                            responseClient(clientSocket);
                        }
                    }).start();
                } catch (IOException e) {
                    e.printStackTrace();
                }
            }
        }
    }

    /**
     * 响应客户端
     * @param clientSocket：客户端 Socket
     */
    private void responseClient(Socket clientSocket) {
        try {
            // 用于接收客户端消息
            BufferedReader in = new BufferedReader(
                    new InputStreamReader(clientSocket.getInputStream()));
            // 用于向客户端发送消息
            PrintWriter out = new PrintWriter(new BufferedWriter(
                    new OutputStreamWriter(clientSocket.getOutputStream())), true);
            out.println("欢迎来到聊天室");
            while (!mIsServiceDestory) {
                // 读取客户端发送的消息
                String str = in.readLine();
                System.out.println("msg from client: " + str);
                // 当接收到客户端消息为 null 时为客户端断开连接
                if (str == null) {
                    break;
                }
                int i = new Random().nextInt(mDefinedMessages.length);
                // 服务端返回的消息

                String msg = mDefinedMessages[i];
                try {
                    Thread.sleep(500);
                    out.println(msg);
                    System.out.println("send: " + msg);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }

            }
            System.out.println("client quit.");

            // 关闭流
            if (out != null) {
                out.close();
            }
            if (in != null) {
                in.close();
            }
            if (clientSocket != null) {
                clientSocket.close();
            }
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}
```

客户端：当客户端启动时在 onCreate() 方法中开启一个线程去连接服务端 Socket，为了确保能够连接成功，采用超时重连的策略，即每次连接失败后都会重新尝试建立连接。连接成功后通过 while 持续地读取服务端发送过来的消息，直到客户端退出。

客户端的全部代码：

``` java
public class SocketActivity extends AppCompatActivity
        implements View.OnClickListener{

    private static final int MESSAGE_RECEIVE_NEW_MSG = 1;
    private static final int MESSAGE_SOCKET_CONNECTED = 2;

    private TextView mOutput;
    private EditText mInput;
    private Button mSend;
    private PrintWriter mPrintWriter;
    private Socket mClientSocket;

    @SuppressLint("HandlerLeak")
    private Handler handler = new Handler() {
        @Override
        public void handleMessage(Message msg) {
            switch (msg.what) {
                case MESSAGE_RECEIVE_NEW_MSG:
                    // 显示消息
                    mOutput.setText(mOutput.getText() + (String)msg.obj);
                    break;
                case MESSAGE_SOCKET_CONNECTED:
                    // 设置按钮为可点击
                    mSend.setEnabled(true);
                    break;
                default:
                    super.handleMessage(msg);
            }
        }
    };

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_socket);

        findView();
        Intent intent = new Intent(this, SocketService.class);
        startService(intent);
        new Thread(new Runnable() {
            @Override
            public void run() {
                connectTCPService();
            }
        }).start();
    }

    /**
     * 连接服务端 Socket
     * 采取超时重连策略
     */
    private void connectTCPService() {
        Socket socket = null;
        while (socket == null) {
            try {
                socket = new Socket("localhost", 8688);
                mClientSocket = socket;
                // 用于向服务端发送消息
                mPrintWriter = new PrintWriter(new BufferedWriter(
                			new OutputStreamWriter(socket.getOutputStream())), true);
                // 连接成功时设置按钮为可点击
                handler.sendEmptyMessage(MESSAGE_SOCKET_CONNECTED);
                System.out.println("connect server success");
            } catch (IOException e) {
                try {
                    Thread.sleep(1000);
                } catch (InterruptedException e1) {
                    e1.printStackTrace();
                }
                e.printStackTrace();
                System.out.println("connect tcp server failed, retry...");
            }
        }

        try {
            // 用于接收服务端的消息
            BufferedReader in = new BufferedReader(
                    new InputStreamReader(socket.getInputStream()));
            // 不断的去读取从服务端返回的消息
            while (!SocketActivity.this.isFinishing()) {
                // 读取从服务端返回的消息
                String msg = in.readLine();
                System.out.println("receive: " + msg);
                if (msg != null) {
                    String time = formateDateTime(System.currentTimeMillis());
                    String showMsg = "server " + time + ": " + msg + "\n";
                    // 显示服务端返回的消息
                    handler.obtainMessage(MESSAGE_RECEIVE_NEW_MSG, showMsg).sendToTarget();
                }
            }
            System.out.println("quit...");
            // 关闭流
            if (mPrintWriter != null) {
                mPrintWriter.close();
            }
            if (in != null) {
                in.close();
            }
            if (socket != null) {
                socket.close();
            }
        } catch (IOException e) {
            e.printStackTrace();
        }
    }

    /**
     * 格式化时间
     * @param l
     * @return
     */
    @SuppressLint("SimpleDateFormat")
    private String formateDateTime(long l) {
        return new SimpleDateFormat("(HH:mm:ss)").format(new Date(l));
    }

    private void findView() {
        mOutput = (TextView) findViewById(R.id.tv_output);
        mInput = (EditText) findViewById(R.id.et_input);
        mSend = (Button) findViewById(R.id.btn_send);
        mSend.setOnClickListener(this);
    }

    @Override
    public void onClick(View view) {
        if (view == mSend) {
            // 获取输入的内容
            String msg = mInput.getText().toString().trim();
            if (!TextUtils.isEmpty(msg) && mPrintWriter != null) {
                // 发送给服务端
                mPrintWriter.println(msg);
                mPrintWriter.flush();
                // 设置输入框为空
                mInput.setText("");
                String time = formateDateTime(System.currentTimeMillis());
                String showMsg = "client " + time + ": " + msg + "\n";
                // 显示客户端发送的消息
                handler.obtainMessage(MESSAGE_RECEIVE_NEW_MSG, showMsg).sendToTarget();
            }
        }
    }

    @Override
    protected void onDestroy() {
        super.onDestroy();
        if (mClientSocket != null) {
            try {
                mClientSocket.shutdownInput();
                mClientSocket.close();
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
    }
}
```

## IPC 方式比较
上面一共实现了六种 IPC 方式，各种方式之间都有哪些联系和区别呢？总结一下：

> | 方式 | 优点 | 缺点 | 适用场景 |
| :--: | :--: | :--: | :------: |
| Bundle | 简单易用 | 只能传输 Bundle 支持的数据类型 |        四大组件间的进程间通信 |
| 文件共享 | 简单易用 | 不适合高并发场景，并且无法做到进程间的即时通信 | 无并发访问情形，交换简单的数据实时性不高的场景 |
| AIDL | 功能强大，支持一对多并发通信，支持实时通信 | 使用稍复杂，需要处理好线程同步 | 一对多通信且有 RPC 需求 |
| Messenger | 功能一般，支持一对多串形通信，支持实时通信 | 不能很好处理高并发情形，不支持 RPC ，数据通过 Messenger 进行传输，因此只能传输 Bundle 支持的数据类型 | 低并发的一对多即时通信，无 RPC 需求，或者无需返回结果的 RPC 需求 |
| ContentProvider | 在数据源访问方面功能强大，支持一对多并发数据共享，可通过 Call 方法扩展其他操作 | 可以理解为受约束的 AIDL ，主要提供数据源的 CRUD 操作 | 一对多的进程间的数据共享 |
| Socket | 功能强大，可以通过网络传输字节流，支持一对多并发实时通信 | 实现细节稍微有点烦琐，不支持直接的 RPC | 网络数据交换 |


能看到这里首先恭喜你耐心见长了，其次要感谢赏光，有什么问题可以留言交流。