转载请附原文链接：[ThreadPoolExecutor源码学习笔记](http://extremej.itscoder.com/threadpoolexecutor_source/)

**大部分分析以注释形式写在源码中**

本篇笔记将从 ThreadPoolExecutor 的一次使用上来分析源码，主要涉及线程池创建，execute 的步骤，任务添加到阻塞队列，线程从阻塞队列中拿取任务执行，线程的回收，线程池的终止。

涉及到的类有

- Executors — 获取线程池

- ThreadPoolExecutor — 线程池

- Worker — 工作线程

- LinkedBlockingQueue — 阻塞队列

- RejectedExecutionHandler — 任务拒绝处理器（实在不知道什么翻译～）

  ​

### 线程池的获取

我们知道可以通过 Executors 来获取不同类型的线程池，那么就从 Executors 来开始看它是如何返回不同类型的线程池的，看看我们常用的一些方法

```java
//获取一个固定线程池
public static ExecutorService newFixedThreadPool(int nThreads, ThreadFactory threadFactory) {
    return new ThreadPoolExecutor(nThreads, nThreads,
                                  0L, TimeUnit.MILLISECONDS,
                                  new LinkedBlockingQueue<Runnable>(),
                                  threadFactory);
}
//获取只有一个线程的池子
public static ExecutorService newSingleThreadExecutor() {
        return new FinalizableDelegatedExecutorService
            (new ThreadPoolExecutor(1, 1,
                                    0L, TimeUnit.MILLISECONDS,
                                    new LinkedBlockingQueue<Runnable>()));
    }
//获取一个缓存线程池，可变
public static ExecutorService newCachedThreadPool() {
        return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                                      60L, TimeUnit.SECONDS,
                                      new SynchronousQueue<Runnable>());
    }
```

从上面的三个方法可以发现其实都是 new 了一个 ThreadPoolExecutor ，但是传入的参数不同，我们进到这个构造方法中去一探究竟，看看不同的参数到底代表了什么

```java
//构造方法
public ThreadPoolExecutor(int corePoolSize,
                          int maximumPoolSize,
                          long keepAliveTime,
                          TimeUnit unit,
                          BlockingQueue<Runnable> workQueue,
                          ThreadFactory threadFactory,
                          RejectedExecutionHandler handler) {
    if (corePoolSize < 0 ||
        maximumPoolSize <= 0 ||
        maximumPoolSize < corePoolSize ||
        keepAliveTime < 0)
        throw new IllegalArgumentException();
    if (workQueue == null || threadFactory == null || handler == null)
        throw new NullPointerException();
    this.corePoolSize = corePoolSize;
    this.maximumPoolSize = maximumPoolSize;
    this.workQueue = workQueue;
    this.keepAliveTime = unit.toNanos(keepAliveTime);
    this.threadFactory = threadFactory;
    this.handler = handler;
}
```

通过参数名称以及注释可以知道这几个参数的作用分别是

- corePoolSize — 核心线程数，即允许闲置的线程数目
- maximumPoolSize — 最大线程数，即这个线程池的容量
- keepAliveTime — 非核心线程的闲置存活时间
- unit — 上一个参数的单位
- workQueue — 任务队列（阻塞队列）
- threadFacotry —  线程创建工厂
- handler — 当线程池或者任务队列容量已满时用于 reject

这里要明白一件事情，**核心线程只是通过数目来判断，而不是说先创建的线程就是核心线程。**

这句话可能有点难懂，我大概解释一下，线程池这个核心线程数的用处就是来判断当前这个闲置线程是否应该回收，那么什么是闲置线程呢？一个线程执行完了一个任务后，会去阻塞队列里面取新的任务，在取到任务之前它就是一个闲置的线程，**取任务的方法有两个，一个是一直阻塞直到取出任务，另一个是一定时间内阻塞直到取出任务或者超时，如果超时这个线程就会被回收，我们知道核心线程一般不会被回收。**

**线程在取任务的时候，线程池会比较当前的有效线程数和允许的核心线程数，如果小于当前的核心线程数则使用第一个方法取任务，也就是没有超时回收，如果大于核心线程数，则使用第二个，一旦超时就回收，所以，并没有绝对的核心线程，只要这个线程出于闲置状态就有被回收的可能。**

**还有一种情况是设置了线程池允许核心线程超时回收，那么无论线程数有多少，统统会使用第二个方法取任务。**

### 任务的执行

#### A.状态属性

在看源码之前先了解一下 ThreadPoolExecutor 的几个状态属性，这对后面的源码阅读有很重要的作用，ThreadPoolExecutor 有五种状态

```java
private static final int RUNNING    = -1 << COUNT_BITS; 
private static final int SHUTDOWN   =  0 << COUNT_BITS;
private static final int STOP       =  1 << COUNT_BITS;
private static final int TIDYING    =  2 << COUNT_BITS;
private static final int TERMINATED =  3 << COUNT_BITS;
```

从上到下依次是

- RUNNING — 运行状态，可以添加新任务，也可以处理阻塞队列中的任务
- SHUTDOWN — 待关闭状态，不再接受新的任务，会继续处理阻塞队列中的任务
- STOP — 停止状态，此时的线程池不处理任何任务
- TIDYING — 整理状态，也可以理解为预终结状态，这个时候任务都处理完毕，池中无有效线程
- TERMINATED — 终止状态

#### B.execute(Runnable command)

当获取到了一个线程池之后，需要它来执行异步任务，也就是 execute(Runnable) ,传入一个 runnable 对象，在 run 方法中执行我们的代码，那么来看一下 execute() 是怎么工作的，因为源码的注释解释得十分清楚，这里将注释也贴出来。简单翻译一下，当 execute 被调用时总共有三种情况。

- **如果当前的有效线程数小于核心线程数，则试图创建一个新的 worker 线程**
- **如果上面一步失败了，则试图将任务添加到阻塞队列中，并且要再一次判断需要不需要回滚队列，或者说创建线程（后面会详细说明）**
- **如果上面两步都失败了，则会试图强行创建一个线程来执行这个任务，如果还是失败，扔掉这个任务**

了解了这三个步骤，来看看源码，源码中调用了 addworker 方法，这是创建线程的方法，会在后面讲到

```java
public void execute(Runnable command) {
    if (command == null)
        throw new NullPointerException();
    /*
     * Proceed in 3 steps:
     *
     * 1. If fewer than corePoolSize threads are running, try to
     * start a new thread with the given command as its first
     * task.  The call to addWorker atomically checks runState and
     * workerCount, and so prevents false alarms that would add
     * threads when it shouldn't, by returning false.
     *
     * 2. If a task can be successfully queued, then we still need
     * to double-check whether we should have added a thread
     * (because existing ones died since last checking) or that
     * the pool shut down since entry into this method. So we
     * recheck state and if necessary roll back the enqueuing if
     * stopped, or start a new thread if there are none.
     *
     * 3. If we cannot queue task, then we try to add a new
     * thread.  If it fails, we know we are shut down or saturated
     * and so reject the task.
     */
    int c = ctl.get();
  //1.判断有效线程数是否小于核心线程数
    if (workerCountOf(c) < corePoolSize) {
      //创建新线程
        if (addWorker(command, true))
            return;
        c = ctl.get();
    }
  //2.分开来看，首先判断当前的池子是否是处于 running 状态
  //因为只有 running 状态才可以接收新任务
  //接下来判断能否成功添加到队列中，如果队列满了或者其他情况则会跳到下一步
    if (isRunning(c) && workQueue.offer(command)) {
      //再次检查池子的状态，如果进入了非 running 状态，回滚队列，扔掉这个任务
        int recheck = ctl.get();
        if (! isRunning(recheck) && remove(command))
            reject(command);
      //如果处于 running 状态则检查当前的有效线程，如果没有则创建一个线程
        else if (workerCountOf(recheck) == 0)
            addWorker(null, false);
    }
  //3.前两步失败了，就强行创建线程，成功会返回true，如果失败扔掉这个任务
    else if (!addWorker(command, false))
        reject(command);
}
```

解释一下第二步，为什么要recheck

**当这个任务被添加到了阻塞队列前，池子处于 RUNNING 状态，但如果在添加到队列成功后，池子进入了 SHUTDOWN 状态或者其他状态，这时候是不应该再接收新的任务的，所以需要把这个任务从队列中移除，并且 reject**

**同样，在没有添加到队列前，可能有一个有效线程，但添加完任务后，这个线程闲置超时或者因为异常被干掉了，这时候需要创建一个新的线程来执行任务**

为了更直观的理解一个任务的执行过程，我画了一张图：

![执行流程.png](http://7xtakx.com1.z0.glb.clouddn.com/threadpoolexecutor.png)

#### C .addWorker()

前一步把 execute 的流程捋了一遍，里面多次出现了 addWorker() 方法，前文说到这是个创建线程的方法，来看看 addWorker 做了些什么，这个方法代码比较长，我们拆开来一点一点看

- 第一部分 — 判断各种基础异常

```java
private boolean addWorker(Runnable firstTask, boolean core) {
    retry:
    for (;;) {
        int c = ctl.get();
        int rs = runStateOf(c);

      // Check if queue empty only if necessary.
      // 检查线程池状态，队列状态，以及 firstask ，拆开来看
      // 这段代码看起来异常的蛋疼,转换一下逻辑即
     //rs>= SHUTDOWN && (rs != SHUTDOWN || firstTask != null ||workQueue.isEmpty())
      // 总结起来就是 当前处于非 Running 状态,并且这三种情况
      // 1. 不是处于 SHUTDOWN 状态，不能再创建线程
      // 2. 有新的任务 (因为不能再接收新的任务)
      // 3. 阻塞队列中已经没有任务 (不需要再创建线程)
        if (rs >= SHUTDOWN &&
            ! (rs == SHUTDOWN &&
               firstTask == null &&
               ! workQueue.isEmpty()))
            return false;
		
        for (;;) {
            int wc = workerCountOf(c);//当前有效线程数目
           // 根据传入的参数确定以核心线程数还是最大线程数作为判断条件
            if (wc >= CAPACITY ||
                wc >= (core ? corePoolSize : maximumPoolSize))
              // 大于容量 或者指定的线程数，不允许创建
                return false;
            if (compareAndIncrementWorkerCount(c))
                break retry;
            c = ctl.get();  // Re-read ctl
            if (runStateOf(c) != rs)
                continue retry;
            // else CAS failed due to workerCount change; retry inner loop
        }
    }
  ｝
```

- 第二部分 — 试图创建线程

创建一个Worker(什么东西？下文会讲解，这里把它就当成是一个线程的容器)

```java
boolean workerStarted = false;//标记 worker 开启状态
boolean workerAdded = false;//标记 worker 添加状态
Worker w = null;
try {
    w = new Worker(firstTask); //将这个任务作为 worker 的第一个任务传入
    final Thread t = w.thread;//通过 worker 获取到一个线程
    if (t != null) {
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            // Recheck while holding lock.
            // Back out on ThreadFactory failure or if
            // shut down before lock acquired.
            int rs = runStateOf(ctl.get());
			
           // running状态，或者 shutdown 状态但是没有新的任务
            if (rs < SHUTDOWN ||(rs == SHUTDOWN && firstTask == null)) {
                if (t.isAlive()) // precheck that t is startable
                    throw new IllegalThreadStateException();
                // 将这个 worker 添加到线程池中
              	workers.add(w); 
                int s = workers.size();
                if (s > largestPoolSize)
                    largestPoolSize = s;
              //标记worker添加成功
                workerAdded = true;
            }
        } finally {
            mainLock.unlock();
        }
      	// 如果 worker 创建成功，开启线程
        if (workerAdded) {
            t.start();
            workerStarted = true;
        }
    }
} finally {
    if (! workerStarted)
        addWorkerFailed(w);
}
return workerStarted;
```

上面代码从逻辑层面来看不算难懂，到这里一个任务到达后，ThreadPoolExecutor 的处理就结束了，那么任务又是怎么被添加到阻塞队列中，线程是如何从队列中取出任务，上文中的 Worker 又是什么东西？

一个一个来，先来看看 Worker 到底是什么

#### D.Worker

Worker 是 ThreadPoolExecutor 的一个内部类，实现了 Runnable 接口，继承自 AbstractQueuedSynchronizer,这又是个什么鬼？？？我也不造～可以看看这篇文章

[《Java并发包源码学习之AQS框架（一）概述》](http://www.cnblogs.com/zhanjindong/p/java-concurrent-package-aqs-overview.html)

简单来说，Worker实现了 lock 和 unLock 方法来标示当前线程的状态是否为闲置

```java
public void lock()        { acquire(1); }
public boolean tryLock()  { return tryAcquire(1); }
public void unlock()      { release(1); }
public boolean isLocked() { return isHeldExclusively(); }
```

上一节创建线程成功后调用 t.start() 而这个线程又是 Worker 的成员变量

```java
Worker(Runnable firstTask) {
    setState(-1); // inhibit interrupts until runWorker
    this.firstTask = firstTask;
    this.thread = getThreadFactory().newThread(this);
}
```

可以看到这里将 Worker 作为 Runnable 参数创建了一个新的线程，我们知道 Thread 接收一个 Runnable 对象后 start 运行的是 Runnable 的 run 方法，Worker 的 run 方法调用了 runWorker ,这个方法里面就是取出任务执行的逻辑

```java
public void run() {
    runWorker(this);
}

final void runWorker(Worker w) {
        Thread wt = Thread.currentThread();
        Runnable task = w.firstTask; // 获取到 worker 的第一个任务
        w.firstTask = null;
        w.unlock(); // 标记为闲置，还没有开始任务 允许打断
        boolean completedAbruptly = true; // 异常退出标记
        try {
          // 循环取出任务，如果第一个任务不为空，或者从队列中拿到了任务
          // 只要这两个条件满足，会一直循环，直到没有任务，正常退出，或者异常退出
            while (task != null || (task = getTask()) != null) {
                w.lock();// 该线程标记为非闲置
                // If pool is stopping, ensure thread is interrupted;
                // if not, ensure thread is not interrupted.  This
                // requires a recheck in second case to deal with
                // shutdownNow race while clearing interrupt
              // 翻译注释：1.如果线程池STOPPING状态，需要中断线程
              // 2.Thread.interrupted()是一个native方法，返回当前线程是否有被等待中断的请求
              // 3.第二个条件成立时，检查线程池状态，如果为STOP，并且没有被中断，则中断线程
                if ((runStateAtLeast(ctl.get(), STOP) ||
                     (Thread.interrupted() &&
                      runStateAtLeast(ctl.get(), STOP))) &&
                    !wt.isInterrupted())
                    wt.interrupt();
              // 执行任务
                try {
                    beforeExecute(wt, task);// 执行前
                    Throwable thrown = null;
                    try {
                        task.run(); // 执行任务
                    } catch (RuntimeException x) {
                        thrown = x; throw x;
                    } catch (Error x) {
                        thrown = x; throw x;
                    } catch (Throwable x) {
                        thrown = x; throw new Error(x);
                    } finally {
                        afterExecute(task, thrown); // 执行结束
                    }
                } finally {
                    task = null; // 将 worker 的任务置空
                    w.completedTasks++; 
                    w.unlock(); // 释放锁，进入闲置状态
                }
            }// 循环结束
            completedAbruptly = false; // 标记为正常退出
        } finally {
          // 干掉 worker
            processWorkerExit(w, completedAbruptly);
        }
    }
```

这里弄清楚了一件事情，进入循环准备执行任务时，worker 加锁标记为非闲置，任务执行完毕或者出现异常，worker 释放锁，进入闲置状态。

**也就是当一个 worker 执行任务前或者执行完任务，到取出下一个任务期间，都是闲置状态可以被打断**

上面取出任务调用了 getTask() ，诶～为什么有一个死循环，别着急，慢慢看来。上面的代码可以知道如果 getTask 返回任务则执行，如果返回为 null 则 worker 需要被回收

```java
private Runnable getTask() {
  // 标记取任务是否超时
    boolean timedOut = false; // Did the last poll() time out?
    for (;;) {
        int c = ctl.get();
        int rs = runStateOf(c);

        // Check if queue empty only if necessary.
      	// 如果线程池状态为 STOP 或者 SHUTDOWN 并且队列已经为空，回收 wroker
        if (rs >= SHUTDOWN && (rs >= STOP || workQueue.isEmpty())) {
            decrementWorkerCount();
            return null;
        }
		//获取当前有效线程数
        int wc = workerCountOf(c);

        // Are workers subject to culling?
      	// timed 用来标记当前的 worker 是否设置超时时间，
      	// 还记得获取线程池的时候 可以设置核心线程超时时间
      	//1.允许核心线程超时回收(即所有线程) 2.当前有效线程超过核心线程数(需要回收)
      	// 如果timed == false 则该worker不会被回收，如果没有取到任务 会一直阻塞
        boolean timed = allowCoreThreadTimeOut || wc > corePoolSize;
		
      	// 回收线程条件
      	// 1. 有效线程数已经大于了线程池的最大线程数或者设置了超时回收并且已经超时
      	// 2. 有效线程数大于1或者队列任务已经为空
      	// 只有当上面1和2 同时满足时 则试图回收线程
        if ((wc > maximumPoolSize || (timed && timedOut))
            && (wc > 1 || workQueue.isEmpty())) {
          // 如果减少workercount成功 直接回收
            if (compareAndDecrementWorkerCount(c))
                return null;
          // 否则重走循环，从第一个判断条件处回收
            continue;
        }
		// 取任务
        try {
          // 根据是否设置超时回收来选择不同的取任务的方式
          // poll 方法取任务会有超时时间，超过时间则返回null
          // take 方法没有超时时间，阻塞式方法
            Runnable r = timed ?
                workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS) :
                workQueue.take();
          // 如果任务不为空返回任务
            if (r != null)
                return r;
          // 否则标记超时 进入下一次循环等待回收
            timedOut = true;
        } catch (InterruptedException retry) {
          // 如果出现异常，试图重试
            timedOut = false;
        }
    }
}
```

getTask() 方法逻辑也捋得差不多了，这里又出现了两个新的方法，workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS) 和 workQueue.take() ，这两个都是阻塞队列的方法，来看看它们又各自是怎么实现的

#### E.LinkedBlockingQueue — 阻塞队列

ThreadPoolExecutor 使用的是链表结构的阻塞队列，实现了 BlockingQueue 接口，而 BlockingQueue 则是继承自 Queue 接口，再上层就是 Collection 接口。

因为本篇笔记主要是分析 ThreadPoolExecutor 的原理，所以不会详细介绍 LinkedBlockingQueue 中的其它代码，主要介绍这里所用的方法，首先来看一下上文所提到的 take()

```java
public E take() throws InterruptedException {
    E x; // 任务
    int c = -1; // 取出任务后的剩余任务数量
    final AtomicInteger count = this.count; // 当前任务数量
    final ReentrantLock takeLock = this.takeLock; // 加锁防止并发
    takeLock.lockInterruptibly();
    try {
      // 如果队列数量为空，则一直循环，阻塞线程
        while (count.get() == 0) {
            notEmpty.await();
        }
      // 取出任务
        x = dequeue();
      // 任务数量减一
        c = count.getAndDecrement();
        if (c > 1)
            notEmpty.signal();// 标记队列非空
    } finally {
        takeLock.unlock(); // 释放锁
    }
  //
    if (c == capacity)
        signalNotFull();//标记队列已满
    return x;// 返回任务
}
```

上面的代码可以知道 take 方法会一直阻塞直到队列有新的任务为止

接下来是 poll 方法，可以看到几乎与 take 方法相同，唯一的区别是在阻塞的循环代码块里面加了时间判断，如果超时则直接返回为空，不会一直阻塞下去

```java
public E poll(long timeout, TimeUnit unit) throws InterruptedException {
    E x = null; // 存放的任务
    int c = -1; 
    long nanos = unit.toNanos(timeout); // 超时时间
    final AtomicInteger count = this.count; // 队列中的数量
    final ReentrantLock takeLock = this.takeLock; // 加锁防止并发
    takeLock.lockInterruptibly();
    try {
      // 如果队列为空，则不断的循环
        while (count.get() == 0) {
          // 如果当倒计时小于0 即超时时间到 则返回空
            if (nanos <= 0)
                return null;
          // 让线程等待
            nanos = notEmpty.awaitNanos(nanos);
        }
        x = dequeue(); // 取出一个任务
        c = count.getAndDecrement(); // 取出后的队列数量
        if (c > 1)
            notEmpty.signal(); // 标记非空
    } finally {
        takeLock.unlock(); // 释放锁
    }
    if (c == capacity)
        signalNotFull(); // 标记队列已满
    return x; // 返回任务
}
```

### 线程池的回收及终止

前一节分析了任务的执行流程及原理，也留下了一个问题，worker 是如何被回收的呢？线程池该如何管理呢？回到上一节的 runWorker() 方法中，还记得最后调用了一个方法

```java
processWorkerExit(w, completedAbruptly);
```

这个方法传入了两个参数，第一个是当前的 Woker ,第二个是标记异常退出的标识

首先判断是否为异常退出，如果是异常退出的话需要手动调整线程数量，如果是正常回收的，getTask 方法里面已经手动调整过了，不记得的小伙伴可以看看前文的代码，找找 decrementWorkerCount(),

```java
private void processWorkerExit(Worker w, boolean completedAbruptly) {
    if (completedAbruptly) // If abrupt, then workerCount wasn't adjusted
        decrementWorkerCount();
	// 加锁
    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
  	// 记录线程池完成的任务总数，从 workers 中移除该 worker
    try {
        completedTaskCount += w.completedTasks;
        workers.remove(w);
    } finally {
        mainLock.unlock();
    }
	
    tryTerminate();//尝试关闭池子

    int c = ctl.get();
  // 以下的代码是判断需不需要给线程池创建一个新的线程
  // 如果线程池的状态是 RUNNING 或者 SHUTDOWN 进一步判断需不需要创建
    if (runStateLessThan(c, STOP)) {
      // 如果为异常退出直接创建，如果不是异常退出进入判断
        if (!completedAbruptly) {
          // 获取线程池应该存在的最小线程数 如果设置了超时 则是0，否则是核心线程数
            int min = allowCoreThreadTimeOut ? 0 : corePoolSize;
          // 如果 min 是0 但是队列又不为空，则 min 应该是1
            if (min == 0 && ! workQueue.isEmpty())
                min = 1;
          //如果当前池中的有效线程数大于等于最小线程数 则不需要创建
            if (workerCountOf(c) >= min)
                return; // replacement not needed
        }
      // 创建线程
        addWorker(null, false);
    }
}
```

上面的代码中调用了 tryTerminate() 方法，这个方法是用于终止线程池的，又是一个 for 循环，从代码结构来看是异常情况的重试机制。还是老方法，慢慢来看总共做了几件事情

```java
final void tryTerminate() {
    for (;;) {
        int c = ctl.get();
      // 如果处于这三种情况不需要关闭线程池
      // 1. Running 状态
      // 2. SHUTDOWN 状态并且任务队列不为空，不能终止
      // 3. TIDYING 或者 TERMINATE 状态，说明已经在关闭了 不需要重复关闭
        if (isRunning(c) ||
            runStateAtLeast(c, TIDYING) ||
            (runStateOf(c) == SHUTDOWN && ! workQueue.isEmpty()))
            return;
      // 进入到关闭线程池的代码，如果线程池中还有线程，则需要打断线程
        if (workerCountOf(c) != 0) { // Eligible to terminate 可以关闭池子
          // 打断闲置线程，只打断一个
            interruptIdleWorkers(ONLY_ONE);
            return;
          // 如果有两个以上怎么办？只打断一个？
          // 这里只打断一个是因为 worker 回收的时候都会进入到该方法中来，可以回去再看看
          // runWorker方法最后的代码
        }
		
      	// 线程已经回收完毕，准备关闭线程池
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();// 加锁
        try {
          //  将状态改变为 TIDYING 并且即将调用 terminated
            if (ctl.compareAndSet(c, ctlOf(TIDYING, 0))) {
                try {
                    terminated(); // 终止线程池
                } finally {
                    ctl.set(ctlOf(TERMINATED, 0)); // 改变状态
                    termination.signalAll();
                }
                return;
            }
        } finally {
            mainLock.unlock();
          // 如果终止失败会重试
        }
        // else retry on failed CAS
    }
}
```

尝试终止线程池的代码分析完了，好像就结束了～但作为好奇宝宝，我们是不是应该看看如何打断闲置线程，以及 terminated 中做了什么呢？来吧，继续装逼

先来看打断线程

```java
private void interruptIdleWorkers(boolean onlyOne) {
    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();//加锁～
    try {
      // 遍历线程池中的 wroker
        for (Worker w : workers) {
            Thread t = w.thread;
          // 如果线程没有被中断，并且能够获取到 worker的锁(说明是闲置线程)
            if (!t.isInterrupted() && w.tryLock()) {
                try {
                    t.interrupt();// 中断线程
                } catch (SecurityException ignore) {
                } finally {
                    w.unlock();
                }
            }
          // 只中断一个 worker 跳出循环，否则会将所有的闲置线程都中断
            if (onlyOne)
                break;
        }
    } finally {
        mainLock.unlock();// 释放锁
    }
}
```

有同学开始装逼了，说我们是好奇宝宝，t.interrupt() 方法也应该看，嗯～没错，但这里是调用了 native 方法，会 c 的可以去看看装逼，我就算了～

好了，再来看看 terminate, 是不是很坑爹？ terminated 里面神！马！也！没！干！。。。淡定，其实这个方法类似于 Activity 的生命周期方法，允许你在被终止时做一些事情，默认的线程池没有什么要做的事情，当然什么也没写啦～

```java
/**
 * Method invoked when the Executor has terminated.  Default
 * implementation does nothing. Note: To properly nest multiple
 * overridings, subclasses should generally invoke
 * {@code super.terminated} within this method.
 */
protected void terminated() { }
```

### 异常处理

还记得前面讲到，出现各种异常情况，添加队列失败等等，只是笼统的说了一句扔掉，当然代码实现不可能是简单一句扔掉就完了。回到 execute() 方法中找到 reject() 任务，看看究竟是怎么处理的

```java
final void reject(Runnable command) {
    handler.rejectedExecution(command, this);
}
```

还记得在创建线程池的时候，初始化了一个 handler — RejectedExecutionHandler

这是一个接口，只有一个方法,接收两个参数

```java
void rejectedExecution(Runnable r, ThreadPoolExecutor executor);
```

既然是一个接口，那么肯定有他的实现类，我们先不急着看所有实现类，先来看看这里的 handler 可能是什么，记得在使用 Executors 获取线程池调用构造方法的时候并没有传入 handler 参数，那么 ThreadPoolExecutor 应该会有一个默认的 handler

```java
private static final RejectedExecutionHandler defaultHandler =
    new AbortPolicy();

public static class AbortPolicy implements RejectedExecutionHandler {
        public AbortPolicy() { }

        public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
            throw new RejectedExecutionException("Task " + r.toString() +
                                                 " rejected from " +
                                                 e.toString());
        }
    }
```

默认 handler 是 AbortPolicy ,这个类实现了 rejectedExecution() 方法，抛了一个 Runtime 异常，也就是说当任务添加失败，就会抛出异常。这个类在 AsyncTask 引发了一场血案～所以在 API19 以后修改了 AsyncTask 的部分代码逻辑，这里就不细说啦，会在下一篇 AsyncTask 的笔记中分析。

实际上，在 ThreadPoolExecutor 中除了 AbortPolicy 外还实现了三种不同类型的 handler

- CallerRunsPolicy — 在 线程池没有 shutdown 的前提下，会直接在执行 execute 方法的线程里执行这个任务

```java
public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
    if (!e.isShutdown()) {
        r.run();
    }
}
```

- DiscardPolicy — 啥也不干，默默地丢掉任务～不信你看

```java
public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
}
```

- DiscardOldestPolicy — 丢弃掉队列中未执行的，最老的任务，也就是任务队列排头的任务，然后再试图在执行一次

```java
public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
    if (!e.isShutdown()) {
        e.getQueue().poll();
        e.execute(r);
    }
}
```



### 总结

其实我不想做任何概念性的总结，原因是我之前没有开始学习源码的时候也看过很多源码分析的文章，大部分文章都会总结一些概念，这些概念本身可能是没有错的，起码是作者自己对源码的理解，但是文字所传达的思想真的是有限的，有时候因为概念的模糊，反而会被带入一个误区，并且长时间的无法转变。

我自己一开始对线程池的理解其实是有偏差的，宏观上可能没有大的问题，但在细节上有很大的误区，通过自己耐心的阅读源码分析后学习到了很多东西。

非要总结的话就给一点我阅读源码的小思路吧：

- 一定要使用过，起码能完整的使用。如果没有用过很难把流程捋清楚
- 从使用的角度作为突破口，一步步的去寻找线索
- 一开始看不需要每一句都弄得很清楚，比如一个方法，应该先搞清楚这个方法里面做了几件事，核心的逻辑是什么
- 在捋清了整体逻辑后，再去看细节上的实现
- 实在无法理解的内容，再看看别人的文章，因为有了源码的基础，再看别人的文章能够有自己的思路
- 与你的好基友探讨，你会发现每个人有不同的角度去理解源码，找到最合适你的那一种

以上是我的一点拙见。

最后感谢我的好基友  — **[阿语](http://yongyu.itscoder.com/)**，在与他的探讨中我走出了误区并有了很多新的理解。