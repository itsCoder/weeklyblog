多线程对于 Android 开发者来说是基础。而且这类知识在计算机里也是很重要的一环，所以很有必要整理一番。

<!-- more -->

>- 文章来源：itsCoder 的 [WeeklyBolg](https://github.com/itsCoder/weeklyblog) 项目
>- itsCoder主页：[http://itscoder.com/](http://itscoder.com/)
>- 作者：[谢三弟](https://github.com/xcc3641)
>- 审阅者：[Jaeger](https://github.com/laobie)


#### 目录

- [目录](#目录)
- [多线程的实现](#多线程的实现)
  - [Thread 源码](#Thread-源码)
  - [线程的几个重要的函数](#线程的几个重要的函数)
  - [Wait() 的实践](#Wait-的实践)
  - [Join() 的实践](#Join-的实践)
  - [Yield() 的实践](#Yield-的实践)
- [总结与参考](#总结与参考)


#### 多线程的实现

来上代码：

```Java
// 最常见的两种方法启动新的线程
public static void startThread() {
    // 覆盖 run 方法
    new Thread() {
        @Override
        public void run() {
            // 耗时操作
        }
    }.start();

    // 传入 Runnable 对象
    new Thread(new Runnable() {
        public void run() {
            // 耗时操作
        }
    }).start();
}
```

其实第一个就是在 Thread 里覆写了 `run()` 函数，第二个是给 Thread 传了一个 Runnable 对象，在 Runnable 对象 `run()` 方法里进行耗时操作。
以前没有怎么考虑过他们两者的关系，今天我们来具体看看到底是什么鬼？

##### Thread 源码

进入 Thread 源码我们看看：

```Java
public class Thread implements Runnable {
    /* What will be run. */
    private Runnable target;

    /* The group of this thread */
    private ThreadGroup group;


    public Thread() {
        init(null, null, "Thread-" + nextThreadNum(), 0);
    }
    public Thread(Runnable target) {
    init(null, target, "Thread-" + nextThreadNum(), 0);
    }
}
```

源码很长，我进行了一点分割。一点一点的来解析看看。
我们首先知道 Thread 也是一个 Runnable ，它实现了 Runnable 接口，并且在 Thread 类中有一个 Runnable 类型的 target 对象。

构造方法里我们都会调用 `init()` 方法，接下来看看在该方法里做了如何的初始化配置。

```Java
    private void init(ThreadGroup g, Runnable target, String name,
                  long stackSize) {
    init(g, target, name, stackSize, null);
    }

    private void init(ThreadGroup g, Runnable target, String name,
                  long stackSize, AccessControlContext acc) {

    Thread parent = currentThread();
    SecurityManager security = System.getSecurityManager();
    // group 参数如果为 null ，则获得当前线程的 group（线程组）
    if (g == null) {
            g = parent.getThreadGroup();
    }
    // 代码省略

    this.group = g;
    this.daemon = parent.isDaemon();
    this.priority = parent.getPriority();
    // 设置 target（ Runnable 类型 ）
    this.target = target;

    }


    public synchronized void start() {

    // 将当前线程加入线程组
    group.add(this);

    boolean started = false;
    try {
        // 启动 native 方法启动新的线程
        start0();
        started = true;
    } finally {
        // 代码省略
    }

   private native void start0();



   @Override
   public void run() {
       if (target != null) {
        target.run();
    }
}

```
从上我们可以明白，最终被线程执行的任务是 Runnable ，Thread 只是对 Runnable 的一个包装，并且通过一些状态对 Thread 进行管理和调度。
当启动一个线程时，如果 Thread 的 target 不为空，则会在子线程中执行这个 target 的 `run()` 函数，否则虚拟机就会执行该线程自身的 `run()` 函数。

##### 线程的几个重要的函数

- wait()
    当一个线程执行到 wait() 方法时，它就进入到一个和该对象相关的等待池中，同时失去（释放）了对象的机锁，使得其他线程可以访问。用户可以使用 notify 、notifyAll 或者指定睡眠时间来唤醒当前等待池中的线程。
    注意：`wait() notify() notifyAll()` 必须放在 `synchronized` block 中，否则会抛出异常。
- sleep()
    该函数是 Thread 的静态函数，作用是使调用线程进入睡眠状态。因为 `sleep()` 是 Thread 类的静态方法，因此他不能改变对象的机锁。所以，当在一个 `synchronized` 块中调用 `sleep()` 方法时，线程虽然休眠了，但是对象的机锁并没有被释放，其他线程无法访问这个对象。
- join()
    等待目标线程执行完成之后继续执行。
- yield()
    线程礼让。目前线程由运行状态转换为就绪状态，也就是让出执行权限，让其他线程得以优先执行，但其他线程能否优先执行未知。

在源码中，查看 Thread 里的 State ，对几种状态解释的很清楚。
> NEW 状态是指线程刚创建，尚未启动
>
> RUNNABLE 状态是线程正在正常运行中，当然可能会有某种耗时计算 / IO 等待的操作 / CPU 时间片切换等, 这个状态下发生的等待一般是其他系统资源, 而不是锁, Sleep 等
>
> BLOCKED  这个状态下，是在多个线程有同步操作的场景, 比如正在等待另一个线程的 synchronized 块的执行释放，或者可重入的 synchronized 块里别人调用 wait() 方法，也就是这里是线程在等待进入临界区
>
> WAITING  这个状态下是指线程拥有了某个锁之后，调用了他的 wait 方法，等待其他线程 / 锁拥有者调用 notify / notifyAll 一遍该线程可以继续下一步操作，这里要区分 BLOCKED 和 WATING ，一个是在临界点外面等待进入， 一个是在临界点里面 wait 等待别人 notify ， 线程调用了 join 方法 进入另外的线程的时候, 也会进入 WAITING 状态，等待被他 join 的线程执行结束
>
> TIMED_WAITING  这个状态就是有限的 (时间限制) 的 WAITING， 一般出现在调用 `wait(long), join(long)` 等情况下，另外，一个线程 sleep 后, 也会进入 TIMED_WAITING 状态
>
> TERMINATED 这个状态下表示 该线程的 run 方法已经执行完毕了, 基本上就等于死亡了 (当时如果线程被持久持有, 可能不会被回收)


##### Wait() 的实践

我们来看一段，`wait()` 的用途和效果。
```Java
	static void waitAndNotifyAll() {

		System.out.println("主线程运行");

		Thread thread = new WaitThread();
		thread.start();
		long startTime = System.currentTimeMillis();
		try {
			synchronized (sLockOject) {
				System.out.println("主线程等待");
				sLockOject.wait();
			}
		} catch (Exception e) {
		}

		long timeMs = System.currentTimeMillis() - startTime;
		System.out.println("主线程继续 —-> 等待耗时：" + timeMs + " ms");

	}

	static class WaitThread extends Thread {

		@Override
		public void run() {
			try {
				synchronized (sLockOject) {
					System.out.println("进入子线程");
					Thread.sleep(3000);
					System.out.println("唤醒主线程");
					sLockOject.notifyAll();
				}
			} catch (Exception e) {
			}
		}

	}
```

在 `waitAndNotifyAll()` 函数里，会启动一个 WaitThread 线程，在该线程中将会调用 sleep 函数睡眠 3 秒。线程启动之后在主线程调用 sLockOject 的 `wait()` 函数，使主线程进入等待状态，此时将不会继续执行。等 WaitThread 在 `run()` 函数沉睡了 3 秒后会调用 sLockOject 的 `notifyAll()` 函数，此时就会重新唤醒正在等待中的主线程，因此会继续往下执行。

结果如下：

> 主线程运行
> 主线程等待
> 进入子线程
> 唤醒主线程
> 主线程继续 —-> 等待耗时：3005 ms

`wait()、notify()` 机制通常用于等待机制的实现，当条件未满足时调用 wait 进入等待状态，一旦条件满足，调用 `notify` 或 `notifyAll` 唤醒等待的线程继续执行。



<div class="tip">

对于这里细节可能会有一些疑问。</br>

在子线程启动的时候，`run()` 函数里面已经持有了该对象锁。</br>

但是真实环境下，其实是主线程先持有对象锁，然后调用 `wait()` 进入等待区并且释放锁等待唤醒。

</div>

这个问题涉及到 JNI 代码，目前我只能从理论上来解释这个问题。
我们都知道一个线程 `start()` 并不是马上启动，而是需要 CPU 分配资源的，根据目前运行来看，分配资源的时间大于 Java 虚拟机运行指令的时间，所以主线程比子线程先拿到锁。
我们还可以知道一点，控制台打印出的时间是 **3005 ms** ，在代码里我们只等待了 3s 多出来的 5ms （这个数字会浮动）我们可以推断是，子线程获取 CPU 的时间加上唤醒主线程的时间。

##### Join() 的实践

`join()` 的注释上面写着：
> Waits for this thread to die.

意思是，阻塞当前调用 `join()` 函数所在的线程，直到接收线程执行完毕之后再继续。
我们来看看实践代码：


```Java
public class JoinThread {

	public static void main(String[] args) {
		joinDemo();
	}

	public static void joinDemo() {
		Worker worker1 = new Worker("work-1");
		Worker worker2 = new Worker("work-2");
		worker1.start();
		System.out.println("启动线程 1 ");

		try {
			// 调用 worker1 的 join 函数，主线，程会阻塞直到 woker1 执行完成
			worker1.join();
			System.out.println("启动线程 2");
			// 再启动线程 2 ，并且调用线程 2 的 join 函数，主线程会阻塞直到 woker2 执行完成
			worker2.start();
			worker2.join();
		} catch (InterruptedException e) {
			e.printStackTrace();
		}

		System.out.println("主线程继续执行");
	}

	static class Worker extends Thread {
		public Worker(String name) {
			super(name);
		}

		@Override
		public void run() {

			try {
				Thread.sleep(2000);
			} catch (InterruptedException e) {

				e.printStackTrace();
			}

			System.out.println("work in " + getName());

		}
	}
}
```

运行之后我们得到：
> 启动线程 1
> work in work-1
> 启动线程 2
> work in work-2
> 主线程继续执行

在 `joinDemo()` 方法里我们创建两个子线程，然后启动了 work1 线程，下一步调用了 woker1 的 `join()` 函数。此时，主线程会进入阻塞状态，直到 work1 执行完毕之后才开始继续执行。因为 Worker 的 `run()` 方法里会休眠 2 秒，因此线程每次调用了 `join()` 方法实际上都会阻塞 2 秒，直到 `run()` 方法执行完毕再继续。
所以，上述代码逻辑其实就是：

__启动线程1__ —-> __等待线程 1 执行完毕__ —-> __启动线程2__ —-> __等待线程 2 执行完毕__ —-> __继续执行主线程代码__

##### Yield() 的实践

`yield()` 是 Thread 的静态方法，注释上说：
> A hint to the scheduler that the current thread is willing to yield its current use of a processor. The scheduler is free to ignore this hint.

大致意思是说：当前线程让出执行时间给其他的线程。
我们都知道，线程的执行是有时间片的，每个线程轮流占用 CPU 固定时间，执行周期到了之后让出执行权给其他线程。
`yield()` 就是主动让出执行权给其他线程。

来看看我们实践的代码：
```Java
public class YieldThreadTest {

	public static void main(String[] args) {
		YieldTread t1 = new YieldTread("thread-1");
		YieldTread t2 = new YieldTread("thread-2");
		t1.start();
		t2.start();
	}

	public static class YieldTread extends Thread {

		public YieldTread(String name) {
			super(name);
		}

		public synchronized void run() {
			for (int i = 0; i < 5; i++) {
				System.out.printf("%s 优先级为 [%d] -------> %d\n", this.getName(), this.getPriority(), i);
				// 当 i 为 2 时，调用当前线程的 yield 函数
				if (i == 2) {
					Thread.yield();

				}
			}
		}

	}

}
```
在 `main()` 方法里创建了两个 YieldTread 线程，控制台输出结果如下：

> thread-1 优先级为 [5] -------> 0
> thread-1 优先级为 [5] -------> 1
> thread-1 优先级为 [5] -------> 2
>
> thread-2 优先级为 [5] -------> 0
> thread-2 优先级为 [5] -------> 1
> thread-2 优先级为 [5] -------> 2
>
> thread-1 优先级为 [5] -------> 3
> thread-1 优先级为 [5] -------> 4
> thread-2 优先级为 [5] -------> 3
> thread-2 优先级为 [5] -------> 4

通常情况下 t1 首先执行，让 t1 的 `run()` 函数执行到了 i 等于 2 时让出当前线程的执行时间。所以我们看到前三行都是 t1 在执行，让出执行时间后 t2 开始执行。后面逻辑简单思考下就得知了，这里也不做过多诠释。

因此，调用 `yield()` 就是让出当前线程的执行权，这样一来让其他线程得到优先执行。


#### 总结与参考

本章内容属于线程的基础，具体生产环境内是下一次我会做的笔记，关于线程池。
但是这章内容也及其重要，因为它是后面的基础。
正确理解才能让我们对各种线程问题有方向和思路。

__参考读物__：
- [Android 开发进阶 --- 从小工到专家](https://book.douban.com/subject/26744163/)
- [我是一个线程](http://mp.weixin.qq.com/s?__biz=MzAxOTc0NzExNg==&mid=416915373&idx=1&sn=f80a13b099237534a3ef777d511d831a&scene=0#wechat_redirect)
