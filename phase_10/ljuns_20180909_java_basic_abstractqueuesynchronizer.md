---
title: Java 基础--队列同步器(AQS)
date: 2018-09-04 13:12:46
categories: Java 基础
tags: 队列同步器，AQS


---

在 Java 5 之前，Java 程序是靠 synchronized 关键字实现锁的功能的，在 Java 5 之后并发包中提供了 Lock 接口及相关实现类（ReentrantLock、CountDownLatch ...）来实现锁的功能，而这些实现类内部正是用到了 AbstractQueuedSynchronizer 来实现对应的功能。

<!-- more -->

### 简介

队列同步器 AbstractQueuedSynchronizer（简称同步器）是锁和其他同步组件的基础框架，内部使用了一个 int 类型的成员变量来表示同步状态，还使用了一个 FIFO 队列来管理线程的排队工作。

#### 同步状态

同步器内部提供了对同步状态操作的方法，包括设置和获取：

```java
// 同步状态
private volatile int state;

// 获取同步状态，1 表示获取同步状态成功，0 表示获取同步状态失败
protected final int getState() {
    return state;
}

// 设置同步状态
protected final void setState(int newState) {
    state = newState;
}

// 采用 CAS 方式设置同步状态
protected final boolean compareAndSetState(int expect, int update) {
    return unsafe.compareAndSwapInt(this, stateOffset, expect, update);
}
```

同步器中还有几个空方法，在自定义同步器时可以按需重写，当需要操作同步状态时可通过上面三个方法来完成：

```java
// 独占式获取同步状态
protected boolean tryAcquire(int arg) {
    throw new UnsupportedOperationException();
}

// 独占式释放同步状态
protected boolean tryRelease(int arg) {
    throw new UnsupportedOperationException();
}

// 共享式获取同步状态
protected int tryAcquireShared(int arg) {
    throw new UnsupportedOperationException();
}

// 共享式释放同步状态
protected boolean tryReleaseShared(int arg) {
    throw new UnsupportedOperationException();
}

// 是否被当前线程占用
protected boolean isHeldExclusively() {
    throw new UnsupportedOperationException();
}
```

此外，同步器还提供了以下常用的模板方法：

> - acquire(int arg)：独占式获取同步状态，内部是通过 `tryAcquire(int arg)` 方法实现的。
> - acquireInterruptibly(int arg)：与 `acquire(int arg)` 相同，但是该方法可以响应中断。
> - tryAcquireNanos(int arg, long nanosTimeout)：在 `acquireInterruptibly(int arg)` 方法的基础上增加了超时功能。
> - acquireShared(int arg)：共享式获取同步状态，内部是通过 `tryAcquireShared(int arg)` 方法实现的。
> - acquireSharedInterruptibly(int arg)：与 `acquireShared(int arg)` 方法相同，但是该方法可以响应中断。
> - tryAcquireSharedNanos(int arg, long nanosTimeout)：在 `acquireSharedInterruptibly(int arg)` 方法的基础上增加了超时功能。
> - release(int arg)：独占式释放同步状态，内部是通过 `tryRelease(int arg)` 方法实现的。
> - releaseShared(int arg)：共享式释放同步状态，内部是通过 `tryReleaseShared(int arg)` 方法实现的。

#### 同步队列

同步器内部通过 FIFO 队列（双向链表）来管理那些获取同步状态失败的线程。当线程获取同步状态失败后，会将当前线程以及一些状态信息构造成一个节点（Node）添加到队列的尾部，同时阻塞该线程；当同步状态被释放后，会把后继节点的线程唤醒并尝试获取同步状态。

Node 是 AQS 的静态内部类：

```java
static final class Node {
    // 共享式
    static final Node SHARED = new Node();
    // 独占式
    static final Node EXCLUSIVE = null;    

    // 等待状态
    static final int CANCELLED =  1;
    static final int SIGNAL    = -1;
    static final int CONDITION = -2;
    static final int PROPAGATE = -3;
    
    /**
     * 节点的等待状态有以下几种：
     * CANCELLED：由于超时或中断导致取消，节点的状态将不再变化
     * SIGNAL：后继节点处于等待状态，如果当前节点释放同步状态或被取消就会通知后继节点，让后继节点去获取同步状态
     * CONDITION：节点在 Condition 队列（等待队列），此时不会去获取同步状态，直到调用 signal() 方法将其转移到同步队列
     * PROPAGATE：下一次共享式获取同步状态将会传播下去
     * 0：初始状态
     */
    volatile int waitStatus;
    
    // 前驱节点
    volatile Node prev;
    
    // 后继节点
    volatile Node next;
    
    // 当前节点的线程
    volatile Thread thread;
    
    Node nextWaiter;
    // 是否共享
    final boolean isShared() {
        return nextWaiter == SHARED;
    }
    
    // 获取前驱节点
    final Node predecessor() throws NullPointerException {
        Node p = prev;
        if (p == null)
            throw new NullPointerException();
        else
            return p;
    }
    
    Node() {}
    
    Node(Thread thread, Node mode) {     // Used by addWaiter
        this.nextWaiter = mode;
        this.thread = thread;
    }
    
    Node(Thread thread, int waitStatus) { // Used by Condition
        this.waitStatus = waitStatus;
        this.thread = thread;
    }
}
```

节点（Node）是构成同步队列的基础，同步器拥有头节点和尾节点的引用。获取同步状态失败的线程会构建成节点被添加到同步队列的尾部。
同步队列示意图：

<center><img src="https://ws1.sinaimg.cn/large/929c4bfbgy1fusv1r6i2ej20pi06wt92.jpg" width="80%" height="80%" /></center>

当一个线程获取同步状态成功后，其他线程则无法获取同步状态，转而被构造成节点添加到同步队列的尾部。这个添加过程必须是线程安全的，所以同步器提供了一个基于 CAS 的方法 `compareAndSetTail(Node expect, Node update)` 来完成添加。

<center><img src="https://s1.ax2x.com/2018/08/31/5BPXNi.gif" width="80%" height="80%" /></center>

头节点在释放同步状态后会通知后继节点，当后继节点获取同步状态成功后将自己设置为头节点。同步器同样提供了一个基于 CAS 的方法 `compareAndSetHead(Node update)`。

<center><img src="https://i.niupic.com/images/2018/08/31/5z32.gif" width="80%" height="80%" /></center>

### 实现分析

下面将从独占式获取与释放同步状态、共享式获取与释放同步状态来分析同步器的实现。

#### 独占式获取与释放同步状态

首先看一个自定义独占式同步器用法的示例：

```java
// 这是一个独占锁
class Mutex implements Lock, java.io.Serializable {
    // 推荐自定义静态内部类继承 AbstractQueuedSynchronizer
    private static class Sync extends AbstractQueuedSynchronizer {
    
        // 判断是否同步状态
        protected boolean isHeldExclusively() {
            return getState() == 1;
        }
        
        // 如果状态为 0 就获取同步状态
        public boolean tryAcquire(int acquires) {
            assert acquires == 1; // Otherwise unused
            if (compareAndSetState(0, 1)) {
                setExclusiveOwnerThread(Thread.currentThread());
                return true;
            }
            return false;
        }
        
        // 如果状态为 1 就释放同步状态
        protected boolean tryRelease(int releases) {
            assert releases == 1; // Otherwise unused
            if (getState() == 0) throw new IllegalMonitorStateException();
            setExclusiveOwnerThread(null);
            setState(0);
            return true;
        }
        
        // 提供一个 Condition
        Condition newCondition() { return new ConditionObject(); }
        
        private void readObject(ObjectInputStream s)
                throws IOException, ClassNotFoundException {
            s.defaultReadObject();
            setState(0); // reset to unlocked state
        }
    }
    
    // 将所有操作代理到 Sync
    private final Sync sync = new Sync();
    // 暴露给外部使用
    public void lock() { sync.acquire(1); }
    public boolean tryLock() { return sync.tryAcquire(1); }
    public void unlock() { sync.release(1); }
    public Condition newCondition() { return sync.newCondition(); }
    public boolean isLocked() { return sync.isHeldExclusively(); }
    public boolean hasQueuedThreads() { return sync.hasQueuedThreads(); }
    public void lockInterruptibly() throws InterruptedException {
        sync.acquireInterruptibly(1);
    }
    public boolean tryLock(long timeout, TimeUnit unit)
            throws InterruptedException {
        return sync.tryAcquireNanos(1, unit.toNanos(timeout));
    }
}
```

Mutex 是一个独占锁，在同一个时刻只允许一个线程占有锁，Sync 是个继承了 AbstractQueuedSynchronizer 的静态内部类，重写了同步器的空方法并实现了具体的逻辑，这种方式是官方所推荐的。

##### 独占式获取同步状态

调用同步器的 `acquire(int arg)` 方法可以获取同步状态，该方法不响应中断操作。也就是说当线程获取同步状态失败后会加入到同步队列中，如果此时对线程进行中断操作，线程不会从同步队列中移出。
下面看看 `acquire(int arg)` 方法的实现：

```java
public final void acquire(int arg) {
    if (!tryAcquire(arg) && acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();
}
```

`acquire(int arg)` 方法的代码虽然少，但是做的事却不少，接下来分几步进行介绍。

###### 获取同步状态

这一步是通过 `tryAcquire(arg)` 方法来完成的，而这个方法是需要重写的。

###### 构建节点

如果获取同步状态失败，会用当前线程和其他状态信息构建一个节点：

```java
private Node addWaiter(Node mode) {
    // 构建 Node，参数 mode 是 Node.EXCLUSIVE，表示独占式
    Node node = new Node(Thread.currentThread(), mode);
    // 尝试快速添加
    Node pred = tail;
    if (pred != null) {
        node.prev = pred;
        // 添加到尾节点
        if (compareAndSetTail(pred, node)) {
            pred.next = node;
            return node;
        }
    }
    // 添加到同步队列
    enq(node);
    return node;
}
```

###### 添加到同步队列

添加节点到同步队列，这里有两种情况：一种是同步队列为空的时候，也就是说当前线程是第二个获取同步状态的，此时还没有头节点和尾节点，然后添加了一个空节点（`new Node()`）；另一种情况是同步队列不为空的时候：

```java
private Node enq(final Node node) {
    for (;;) {
        Node t = tail;
        if (t == null) {
             // 此时同步队列为空，添加第一个节点
            if (compareAndSetHead(new Node()))
                tail = head;
        } else {
            node.prev = t;
            // 添加到同步队列尾部
            if (compareAndSetTail(t, node)) {
                t.next = node;
                return t;
            }
        }
    }
}
```

###### 自旋

这个过程其实就是当前节点在死循环中获取同步状态：

```java
final boolean acquireQueued(final Node node, int arg) {
    boolean failed = true;
    try {
        boolean interrupted = false;
        for (;;) {
            final Node p = node.predecessor();
            // 只有前驱节点是头节点并且当前节点获取同步状态成功才退出循环
            if (p == head && tryAcquire(arg)) {
                setHead(node);
                p.next = null; // help GC
                failed = false;
                return interrupted;
            }
            // 更新等待状态并阻塞线程
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                interrupted = true;
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}

// 设置当前节点为头节点
private void setHead(Node node) {
    head = node;
    node.thread = null;
    node.prev = null;
}

// 阻塞当前线程
private final boolean parkAndCheckInterrupt() {
    LockSupport.park(this);
    return Thread.interrupted();
}
```

独占式的获取同步状态经过前面四步就完成了，画个流程图加深下印象：

<center><img src="https://ws1.sinaimg.cn/large/929c4bfbgy1fv4ewpsgt4j216y0piwi8.jpg" width="80%" height="80%" /></center>

在获取同步状态时，同步器维护了一个同步队列，获取状态失败的线程都会被构建成节点加入到队列中并进行自旋；移出队列的条件是前驱节点为头节点且成功获取了同步状态。

##### 独占式释放同步状态

线程获取同步状态成功并执行完相关逻辑后就要释放同步状态，后继节点就可以继续获取同步状态。而调用同步器的 `release(int arg)` 方法可以释放同步状态：

```java
public final boolean release(int arg) {
    // tryRelease() 需要重写
    if (tryRelease(arg)) {
        Node h = head;
        if (h != null && h.waitStatus != 0)
            unparkSuccessor(h);
        return true;
    }
    return false;
}

// 唤醒后继节点
private void unparkSuccessor(Node node) {
    // 获取、修改头节点等待状态
    int ws = node.waitStatus;
    if (ws < 0) compareAndSetWaitStatus(node, ws, 0);
    
    Node s = node.next;
    if (s == null || s.waitStatus > 0) {
        s = null;
        for (Node t = tail; t != null && t != node; t = t.prev)
            if (t.waitStatus <= 0)
                s = t;
    }
    if (s != null)
        // 解除阻塞
        LockSupport.unpark(s.thread);
}
```

所以释放同步状态就是修改同步状态，并且唤醒后继节点的线程。

#### 共享式获取与释放同步状态

共享式与独占式最大的区别在于：同一个时刻能否允许多个线程同时获取到同步状态。

再来看自定义的共享式同步器的示例：

```java
class BooleanLatch {
    private static class Sync extends AbstractQueuedSynchronizer {
        boolean isSignalled() { 
            return getState() != 0; 
        }
        // 返回大于等于 0 表示获取同步状态成功
        protected int tryAcquireShared(int ignore) {
            return isSignalled() ? 1 : -1;
        }
        // 释放同步状态
        protected boolean tryReleaseShared(int ignore) {
            setState(1);
            return true;
        }
    }
    
    // 将所有操作代理到 Sync
    private final Sync sync = new Sync();
    // 暴露给外部使用
    public boolean isSignalled() { return sync.isSignalled(); }
    public void signal() { sync.releaseShared(1); }
    public void await() throws InterruptedException {
        sync.acquireSharedInterruptibly(1);
    }
}
```

BooleanLatch 是一个共享锁，在同一个时刻允许多个线程占有锁，Sync 是个继承了 AbstractQueuedSynchronizer 的静态内部类。

##### 共享式获取同步状态

调用同步器的 `acquireShared(int arg)` 方法可以获取同步状态。下面看看代码实现：

```java
public final void acquireShared(int arg) {
    // 大于等于 0 表示获取同步状态成功
    if (tryAcquireShared(arg) < 0)
        doAcquireShared(arg);
}

// 自旋
private void doAcquireShared(int arg) {
    // addWaiter() 过程和独占式基本是一致的，唯一不同是的此时传入了 Node.SHARED，表示共享式
    final Node node = addWaiter(Node.SHARED);
    boolean failed = true;
    try {
        boolean interrupted = false;
        for (;;) {
            // 获取前驱节点
            final Node p = node.predecessor();
            if (p == head) {
                // 获取同步状态，大于等于 0 表示获取成功
                int r = tryAcquireShared(arg);
                if (r >= 0) {
                    setHeadAndPropagate(node, r);
                    p.next = null; // help GC
                    if (interrupted)
                        selfInterrupt();
                    failed = false;
                    return;
                }
            }
            // 更新等待状态并阻塞线程
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                interrupted = true;
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}
```

在 `acquireShared()` 方法中尝试调用 `tryAcquireShared()` 方法获取同步状态，当 `tryAcquireShared()` 的返回值大于等于 0 表示获取同步状态成功。`doAcquireShared()` 方法表示自旋，跳出循环的条件是前驱节点是头节点并且 `tryAcquireShared()` 返回值大于等于 0。

##### 共享式释放同步状态

和独占式一样，共享式也需要释放同步状态，通过调用同步器的 `releaseShared(int arg)` 方法释放同步状态：

```java
public final boolean releaseShared(int arg) {
    if (tryReleaseShared(arg)) {
        doReleaseShared();
        return true;
    }
    return false;
}

// 修改等待状态
private void doReleaseShared() {
    for (;;) {
        Node h = head;
        if (h != null && h != tail) {
            int ws = h.waitStatus;
            if (ws == Node.SIGNAL) {
                if (!compareAndSetWaitStatus(h, Node.SIGNAL, 0))
                    continue;            // loop to recheck cases
                // 唤醒后继节点
                unparkSuccessor(h);
            }
            else if (ws == 0 &&
                     !compareAndSetWaitStatus(h, 0, Node.PROPAGATE))
                continue;                // loop on failed CAS
        }
        if (h == head)                   // loop if head changed
            break;
    }
}
```

在 `releaseShared()` 方法中尝试调用 `tryReleaseShared()` 方法释放同步状态，当 `tryReleaseShared()` 返回 true 表示同步状态已经修改成功。`doReleaseShared()` 方法主要是修改头节点的等待状态以及唤醒后继节点的线程。

#### 小结

获取同步状态失败后会把当前线程以及其他一些状态信息构建成节点添加到同步队列中（如果此时同步队列是空的，那么会先添加一个空节点，然后再添加这个节点），如果是独占式获取，新构建的节点是 `Node.EXCLUSIVE`，否则是 `Node.SHARED` ，加入到同步队列后节点会自旋（死循环获取同步状态）。

释放同步状态成功后，将会唤醒后继节点的线程，后继节点会在自旋状态中获取到同步状态，然后从同步队列中移除。

#### 参考

《Java 并发编程的艺术--方腾飞、魏鹏、程晓明》