---
title: AQS同步器
date: 2019-01-10 19:53:57
toc: true
tags:
 - Java
 - 锁
 - 多线程
categories: JUC
---

&emsp;&emsp;在 java.util.concurrent (JUC) 并发包中，如 ReentrantLock，Semaphore，CountDownLatch 等并发类的同步控制都是基于 AbstractQueuedSynchronizer (简称 AQS) 这个同步器抽象类来实现的。在这里较为深入的讨论同步器抽象类的实现原理与应用。

## AQS简介

AbstractQueuedSynchronizer 内部维护着一个 FIFO 的 CLH 队列，队列中的每个 Node 代表着一个需要获取锁的线程

![](https://zzcoder.oss-cn-hangzhou.aliyuncs.com/juc/aqs01.png)

> {% raw %}<i class="far fa-bell"></i> {% endraw%}
>
> &emsp;&emsp;自旋锁：自旋锁是指当一个线程尝试获取某个锁时，如果该锁已被其他线程占用，就一直循环检测锁是否被释放，而不是立刻进入线程挂起或睡眠状态。
>
> - CLH 锁（Craig, Landin, and Hagersten  locks）：基于链表的可扩展、高性能、公平的自旋锁，它不断轮询前驱的状态，如果发现前驱释放了锁就结束自旋
> - MCS 锁：在当前结点自旋，但由前驱结点通知其结束自旋

AQS 采用的是一种变种的 CLH 队列锁：原始 CLH 是在前驱结点自旋，通过判断 pred.locked 来自旋，而 **AQS 的 CLH 则是根据前驱结点的状态来控制阻塞，不会一直自旋。同时当前驱结点释放锁时会去唤醒该结点使其参与竞争锁。** AQS 的结点的定义如下：

<!-- more -->

```java
static final class Node {

    volatile int waitStatus;

    volatile Node prev;

    volatile Node next;

    volatile Thread thread;

    // 指向 Condition 队列中的后继节点
    Node nextWaiter;
}
```

Node 结点中分别有指向前驱，后继的结点，入队时的线程以及结点状态（Condition 队列本文不涉及）。结点状态会存在以下几种：

- CANCELLED：线程取消
- SIGNAL：当前线程的后继线程被阻塞或者即将被阻塞，当前线程释放锁或者取消后需要唤醒后继线程
- CONDITION：在等待 Condition ，也就是在 Condition 队列中
- PROPAGATE：当头结点处于 PROPAGATE，需要唤醒后继线程，为了保证共享模式下唤醒机制正常
- 0：初始状态

基于上述 Node 的定义，AQS 基本属性如下：

```java
// 队列的头结点
private transient volatile Node head;

// 队列的尾节点
private transient volatile Node tail;

// 同步状态
private volatile int state;
```

## API

AbstractQueuedSynchronizer 的提供的接口主要有两种类型

### 控制同步状态

AbstractQueuedSynchronizer 并不实现同步接口，所有对同步状态的控制都交由子类同步组件控制。比如 tryAcquire 代表由子类控制当前线程是否能独占式获取同步状态成功

| 方法                              | 说明                       |
| --------------------------------- | -------------------------- |
| boolean tryAcquire(int arg)       | 独占式获取同步状态         |
| boolean tryRelease(int arg)       | 独占式释放同步状态         |
| int tryAcquireShared(int arg)     | 共享式获取同步状态         |
| boolean tryReleaseShared(int arg) | 共享式释放同步状态         |
| boolean isHeldExclusively()       | 检测当前线程是否获取独占锁 |

而在多线程环境中对状态的操纵必须确保原子性，因此它还提供了对状态控制的三组 API：

| 方法                                               | 说明                  |
| -------------------------------------------------- | --------------------- |
| int getState()                                     | 获取同步状态          |
| void setState()                                    | 设置同步状态          |
| boolean compareAndSetState(int expect, int update) | 通过 CAS 设置同步状态 |

通过这三组 API，子类可以线程安全的控制同步状态（同时子类需要确保实现是非阻塞的）

### 模板方法

模板方法封装了获取同步状态成功或失败后的在队列中的一系列操作，子类可以直接调用

| 方法                                              | 说明                                                         |
| ------------------------------------------------- | ------------------------------------------------------------ |
| void acquire(int arg)                             | 独占式获取同步状态，该方法将会调用 tryAcquire 尝试获取同步状态。获取成功则返回，获取失败，线程进入同步队列等待。 |
| void acquireInterruptibly(int arg)                | 响应中断版的 acquire                                         |
| boolean tryAcquireNanos(int arg,long nanos)       | 超时 + 响应中断版的 acquire                                  |
| void acquireShared(int arg)                       | 共享式获取同步状态，同一时刻可能会有多个线程获得同步状态。比如读写锁的读锁就是就是调用这个方法获取同步状态的。 |
| void acquireSharedInterruptibly(int arg)          | 响应中断版的 acquireShared                                   |
| boolean tryAcquireSharedNanos(int arg,long nanos) | 超时 + 响应中断版的 acquireShared                            |
| boolean release(int arg)                          | 独占式释放同步状态                                           |
| boolean releaseShared(int arg)                    | 共享式释放同步状态                                           |

## 互斥锁

### acquire

```java
public final void acquire(int arg) {
    if (!tryAcquire(arg) &&
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();
}
```

tryAcquire 方法代表尝试获取一次互斥锁，需要子类根据需求去实现（比如 ReentrantLock 实现了公平锁和非公平锁），通过布尔变量来标志获取状态：

```java
protected boolean tryAcquire(int arg) {
    throw new UnsupportedOperationException();
}
```

若获取失败，则通过 addWaiter 方法将当前线程添加至阻塞队列

```java
private Node addWaiter(Node mode) {
    
    // 将线程封装在Node节点中
    Node node = new Node(Thread.currentThread(), mode);
    
    // CAS 尝试将该节点插在队列尾
    // Try the fast path of enq; backup to full enq on failure
    Node pred = tail;
    if (pred != null) {
        node.prev = pred;
        if (compareAndSetTail(pred, node)) {
            pred.next = node;
            return node;
        }
    }
    
    // 如果不成功则通过自旋的方式插到队尾，直到插入成功
    enq(node);
    return node;
}

private Node enq(final Node node) {
    for (;;) {
        Node t = tail;
        // 设置头结点，初始情况下，头结点是一个空结点(这里不会直接返回，因此即使阻塞队列为空，当前节点仍然是插在空结点之后)
        if (t == null) { // Must initialize
            if (compareAndSetHead(new Node()))
                tail = head;
        // 插入该结点到队尾
        } else {
            node.prev = t;
            if (compareAndSetTail(t, node)) {
                t.next = node;
                return t;
            }
        }
    }
}
```

插入完成后，则会调用 acquireQueued() 方法对该结点进行有限次自旋获取锁，并在到达边界条件后阻塞

```java
final boolean acquireQueued(final Node node, int arg) {
    boolean failed = true;
    try {
        boolean interrupted = false;
        for (;;) {
            final Node p = node.predecessor();
            // 如果该节点的前一节点为头节点，那么它将有资格参与竞争锁
            // 如果获取锁成功，则将当前结点设为头结点
            if (p == head && tryAcquire(arg)) {
                setHead(node);
                p.next = null; // help GC
                failed = false;
                return interrupted;
            }
            
            // 判断线程需不需要阻塞，和 CLH 不同，线程并不总是参与竞争锁，而是仅当线程被唤醒时竞争锁
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                interrupted = true;
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}

private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
    int ws = pred.waitStatus;
    // 如果前驱节点为 SIGNAL 状态，那么在释放锁时会唤醒后继结点
    // 因此这种情况当前结点会阻塞自己
    if (ws == Node.SIGNAL)
        return true;
    
    // 如果前驱节点为 CANCELLED 状态，那么从后向前找到第一个非取消状态的节点
    // 并更新当前结点的前驱为该结点
    if (ws > 0) {
        do {
            node.prev = pred = pred.prev;
        } while (pred.waitStatus > 0);
        pred.next = node;
    } else {
		
        // 如果前驱节点为 0 或 PROPAGATE，那么设置前驱结点的状态为 SIGNAL（可以说这一步才是标志会将每一个节点阻塞的一步）
        compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
    }
    return false;
}

// LockSupport.park(this) 来挂起线程，然后就停在这里了，等待被唤醒
// 返回的时候会先判断是否由线程中断造成的，如果由线程中断造成，在这里会接下去置中断标记
// 而 lockInterruptibly 方法则是抛出异常
private final boolean  parkAndCheckInterrupt() {
    LockSupport.park(this);
    return Thread.interrupted();
}
```

那么总结下 acquire 方法的逻辑：

1. 尝试获取互斥锁，若获取成功则直接返回
2. 若获取失败，则将当前线程添加到阻塞队列尾（CAS 操作插入，自旋直到插入成功为止）
3. 自旋/阻塞获取锁
   1. 尝试获取互斥锁（前驱结点必须为头结点时，当前结点才有资格竞争锁），若获取成功则将当前结点设为头结点后退出
   2. 若前驱结点为 SIGNAL 状态，则阻塞当前结点（唤醒后继续循环 *自旋/阻塞获取锁* ）
   3. 若前驱结点为 CANCELLED 状态，则更新前驱到非取消结点
   4. 若前驱结点为 0 或 PROPAGATE，则设置前驱结点状态为 SIGNAL 状态

### release 

```java
public final boolean release(int arg) {
    if (tryRelease(arg)) {
        Node h = head;
        
        // h == null 的情况就是阻塞队列为空（前面说过，第一个线程持有锁时不会放到头结点中）
        // h.waitStatus = 0，那么其后的结点必定没有阻塞（前面也说过，因为该值是由后继结点来赋值的，然后仅当该结点状态为阻塞状态，后继结点才会将自己阻塞，即 CLH 特性，根据前驱结点状态来控制自己）
        if (h != null && h.waitStatus != 0)
            unparkSuccessor(h);
        return true;
    }
    return false;
}
```

tryRelease 方法代表尝试释放一次互斥锁，需要子类根据需求去实现，通过布尔变量来标志获取状态：

```java
protected boolean tryRelease(int arg) {
    throw new UnsupportedOperationException();
}
```

在释放锁成功后，会判断当前结点状态来唤醒后继结点，即当前结点状态为 SIGNAL 状态时会唤醒后继结点

```java
if (h != null && h.waitStatus != 0)
    unparkSuccessor(h);

private void unparkSuccessor(Node node) {
	
    int ws = node.waitStatus;
    if (ws < 0)
        // 设置头结点状态为 0
        compareAndSetWaitStatus(node, ws, 0);

    // 从队尾往前找，找到 waitStatus <= 0 的所有节点中排在最前面的(> 0 代表节点取消阻塞)
    Node s = node.next;
    if (s == null || s.waitStatus > 0) {
        s = null;
        for (Node t = tail; t != null && t != node; t = t.prev)
            if (t.waitStatus <= 0)
                s = t;
    }
    // 唤醒该节点，也就是头结点的下一个不为取消阻塞状态的节点
    if (s != null)
        LockSupport.unpark(s.thread);
}
```

那么总结下 release 方法的逻辑：

1. 尝试释放一次互斥锁，若释放失败，则直接返回失败
2. 释放成功后，唤醒一个后继结点

## 互斥锁案例

通过以上的理解，可以实现一个简单的互斥锁

```java
public class MutexLock implements Lock {

    private Sync sync;

    public MutexLock() {

        this.sync = new Sync();
    }

    private static class Sync extends AbstractQueuedSynchronizer {

        public Sync() {

            setState(0);
        }

        @Override
        protected boolean tryAcquire(int acquire) {

            if(compareAndSetState(0, 1)){

                setExclusiveOwnerThread(Thread.currentThread());
                return true;
            }

            return false;
        }

        @Override
        protected boolean tryRelease(int release) {

            if(getState() == 0){
                throw new IllegalMonitorStateException();
            }

            setExclusiveOwnerThread(null);
            setState(0);
            return true;

        }
    }

    @Override
    public void lock() {

        sync.acquire(1);
    }

    @Override
    public void lockInterruptibly() throws InterruptedException {

        sync.acquireInterruptibly(1);
    }

    @Override
    public boolean tryLock() {

        return sync.tryAcquire(1);
    }

    @Override
    public boolean tryLock(long time, TimeUnit unit) throws InterruptedException {

        return sync.tryAcquireNanos(1, unit.toNanos(time));
    }

    @Override
    public void unlock() {

        sync.release(1);
    }

    @Override
    public Condition newCondition() {
        return null;
    }
}
```

内部类 Sync 继承 AbstractQueuedSynchronizer，并重载 tryAcquire 和 tryRelease 方法

- tryAcquire 通过 CAS 尝试获取一次同步状态（0 -> 1），若获取成功则设置当前持有锁的线程为自己
- tryRelease 判断同步状态是否为 1，若是则重置同步状态为 0，且设置当前获取锁的线程为 null，否则抛出异常（互斥锁的释放不会有并发）

我们可以写个简单的并发计数测试：

```java
public class Main {

    // 计数
    private static int count;

    public static void main(String[] args) {

        final Mutex mutex = new Mutex();

        ExecutorService executorService = new ThreadPoolExecutor(
                3, 5, 60, TimeUnit.SECONDS,
                new ArrayBlockingQueue<Runnable>(100),
                new ThreadPoolExecutor.DiscardOldestPolicy()
        );

        count = 0;

        int threadCnt = 10;
        for (int i = 0; i < threadCnt; i++){

            executorService.execute(new Runnable() {
                @Override
                public void run() {

                    for(int i = 0; i < 10000; i++){

                        mutex.lock();

                        try {
                            count++;
                        }finally {
                            mutex.unlock();
                        }
                    }

                }
            });
        }

        executorService.shutdown();

        try {
            executorService.awaitTermination(1, TimeUnit.DAYS);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }

        System.out.println("assert " + threadCnt * 10000 + " = " + count + " is true");
    }
}
```

正常输出为：

```
assert 100000 = 100000 is true
```

## 共享锁

### acquireShared

```java
public final void acquireShared(int arg) {
    if (tryAcquireShared(arg) < 0)
        doAcquireShared(arg);
}
```

tryAcquireShared 方法尝试获取一次共享锁，需要子类根据需求去实现。但和互斥锁不同的是，它以整型作为状态标志，负数代表获取失败，非负数代表获取成功，0 代表成功但之后的竞争线程不会成功

```java
protected int tryAcquireShared(int arg) {
    throw new UnsupportedOperationException();
}
```

在获取共享锁失败时，会调用 doAcquireShared 将当前线程添加至阻塞队列并自旋获取共享锁

```java
private void doAcquireShared(int arg) {
    
    // 将当前线程添加至阻塞队列
    final Node node = addWaiter(Node.SHARED);
    boolean failed = true;
    try {
        boolean interrupted = false;
        for (;;) {
            final Node p = node.predecessor();
            // 如果该节点的前一节点为头节点，那么它将有资格参与竞争锁
            // 如果获取锁成功，则将当前结点设为头结点
            if (p == head) {
                int r = tryAcquireShared(arg);
                if (r >= 0) {
                    // 和互斥锁不同的点，共享锁会在获取锁成功后唤醒后继结点
                    setHeadAndPropagate(node, r);
                    p.next = null; // help GC
                    if (interrupted)
                        selfInterrupt();
                    failed = false;
                    return;
                }
            }
            // 判断线程需不需要阻塞
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

它的大体逻辑和互斥锁的自旋获取锁逻辑相同，但是它们之间有个很重要的不同点，即共享锁在获取锁成功后调用 setHeadAndPropagate 来唤醒后继结点

```java
private void setHeadAndPropagate(Node node, int propagate) {
    Node h = head; // Record old head for check below
    // 设为头结点
    setHead(node);
    
    // 唤醒后继结点
    if (propagate > 0 || h == null || h.waitStatus < 0) {
        Node s = node.next;
        if (s == null || s.isShared())
            doReleaseShared();
    }
}

private void doReleaseShared() {
	
    for (;;) {
        Node h = head;
        if (h != null && h != tail) {
            int ws = h.waitStatus;
            // 如果头结点处于 SIGNAL 状态，唤醒后继结点
            if (ws == Node.SIGNAL) {
                if (!compareAndSetWaitStatus(h, Node.SIGNAL, 0))
                    continue;            // loop to recheck cases
                unparkSuccessor(h);
            }
            // 如果头结点处于 0 的状态，设置头结点状态为 PROPAGATE
            // 这是为了解决共享锁的并发唤醒后继结点导致极端情况下存在线程永远无法唤醒的情况
            else if (ws == 0 &&
                     !compareAndSetWaitStatus(h, 0, Node.PROPAGATE))
	            continue;                // loop on failed CAS
        }
        if (h == head)                   // loop if head changed
            break;
    }
}
```

在判断是否需要唤醒后继结点这步，它的判断逻辑是 propagate > 0 || h.waitStatus < 0：

- propagate > 0 ：tryAcquireShared 方法的返回值，代表当前线程获取共享锁成功（按理说 propagate = 0 的情况也属于获取锁成功，为什么不加进去呢？这是因为当 propagate = 0 时代表当前已经没有共享资源了，所以唤醒也没有意义了）
- h.waitStatus < 0 ：头结点状态为 SIGNAL 或 PROPAGATE 时

{% blockquote %}

{% raw %}<i class="far fa-bell"></i> {% endraw%}

&emsp;&emsp;在共享锁中会存在 PROPAGATE 状态：

- 获取共享锁成功后，如果头结点状态为 0（unparkSuccessor 时会将头结点状态设为0），会将头结点状态设为 PROPAGATE

  {% codeblock lang:java %}
  else if (ws == 0 &&
  ​    !compareAndSetWaitStatus(h, 0, Node.PROPAGATE))
  ​	continue;                // loop on failed CAS
  {% endcodeblock %}

- 判断后继结点是否需要唤醒时会判断头结点的状态 propagate > 0 || h.waitStatus < 0

之所以需要这个状态是因为共享锁的 *唤醒后继结点*  操作是并发操作，同时 propagate = 0 的情况不会唤醒后继结点，因此在一些极端情况下会存在阻塞结点无法被唤醒的情况

{% endblockquote %}

那么我们总结下获取共享锁的逻辑：

1. 尝试获取共享锁，若获取成功则直接返回
2. 若获取失败，则将当前线程添加到阻塞队列尾（CAS 操作插入，自旋直到插入成功为止）
3. 自旋/阻塞获取锁
   1. 尝试获取共享锁（前驱结点必须为头结点时，当前结点才有资格竞争锁），若获取成功则尝试唤醒一个**后继结点**（唤醒的结点如果获取锁成功又会继续唤醒接下去的结点）
   2. 前驱结点为 SIGNAL 状态，则阻塞当前结点（唤醒后继续循环 *自旋/阻塞获取锁* ）
   3. 前驱结点为 CANCELLED 状态，则更新前驱到非取消结点
   4. 前驱结点为 0 或 PROPAGATE，则设置前驱结点状态为 SIGNAL 状态

> 不知道大家注意到了没有，在获取共享锁时，若新线程直接通过 tryAcquireShared 获取锁成功，它是不会入 Node 结点的，那么它也就不会去传播式的唤醒 CLH 队列中的后继节点了，这和上面的结论是否存在矛盾呢？其实这是正常的，我们可以考虑正在和新线程争抢共享锁的结点（头结点的后继结点），如果它抢到了共享锁，那么它会去唤醒后继节点；如果连它都抢不到锁，那么唤醒后继节点已经没有必要了。这个时候只需要等某个持有共享锁的线程释放锁来唤醒就可以了

### releaseShared

```java
public final boolean releaseShared(int arg) {
    if (tryReleaseShared(arg)) {
        doReleaseShared();
        return true;
    }
    return false;
}
```

tryReleaseShared 方法代表尝试释放一次共享锁，需要子类根据需求去实现，通过布尔变量来标志获取状态：

```java
protected boolean tryReleaseShared(int arg) {
    throw new UnsupportedOperationException();
}
```

释放成功后会调用 doReleaseShared 尝试唤醒一个后继结点，上面已经解释了。

## 共享锁案例

基于以上分析，我们也可以实现一个同时允许 N 个线程进入的共享锁

```java
public class ShareLock implements Lock {

    private Sync sync;

    public ShareLock(Integer permit) {

        this.sync = new Sync(permit);
    }

    private static class Sync extends AbstractQueuedSynchronizer{

        Sync(int permit){

            setState(permit);
        }

        @Override
        protected int tryAcquireShared(int acquire) {

            for(;;){

                int expect = getState();
                int update = expect - acquire;

                if(update < 0 || compareAndSetState(expect, update)){

                    return update;
                }
            }
        }

        @Override
        protected boolean tryReleaseShared(int release) {

            for(;;){

                int expect = getState();
                int update = expect + release;

                if(compareAndSetState(expect, update)){

                    return true;
                }
            }
        }
    }

    @Override
    public void lock() {

        sync.acquireShared(1);
    }

    @Override
    public void lockInterruptibly() throws InterruptedException {

        sync.acquireSharedInterruptibly(1);
    }

    @Override
    public boolean tryLock() {

        return sync.tryAcquireShared(1) >= 0;
    }

    @Override
    public boolean tryLock(long time, TimeUnit unit) throws InterruptedException {

        return sync.tryAcquireSharedNanos(1, unit.toNanos(time));
    }

    @Override
    public void unlock() {

        sync.releaseShared(1);
    }

    @Override
    public Condition newCondition() {
        return null;
    }
}
```

接下来对共享锁进行简单的测试：

```java
public class Main{

    public static void main(String[] args) {

        final ShareLock shareLock = new ShareLock(2);

        ExecutorService executorService = new ThreadPoolExecutor(
                5, 5, 60, TimeUnit.SECONDS,
                new ArrayBlockingQueue<Runnable>(100),
                new ThreadPoolExecutor.DiscardOldestPolicy()
        );

        int threadCnt = 10;
        for (int i = 0; i < threadCnt; i++){

            executorService.execute(new Runnable() {
                @Override
                public void run() {

                    shareLock.lock();

                    try {

                        System.out.println(Thread.currentThread().getName() + ": is running");

                        Thread.sleep(2000);

                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    } finally {

                        shareLock.unlock();

                    }
                }
            });
        }
        
        executorService.shutdown();

        try {
            executorService.awaitTermination(1, TimeUnit.DAYS);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}
```

测试程序创建了一个允许最多两个线程同时进入的共享锁。因此正常情况下，日志会成双打印。

## 中断

- thread.interrupt()：中断线程，将会设置该线程的中断状态位，即设置为 true（**不会中断一个正在运行的线程，而是中断阻塞的线程**）
- thread.interrupted()：判断某个线程是否已被发送过中断请求，该方法调用后会将中断标示位清除，即重新设置为 false
- Thread.currentThread().isInterrupted()：判断某个线程是否已被发送过中断请求，不会将中断标示位清除

如果一个线程处于了阻塞状态（如线程调用了 thread.sleep、thread.join、thread.wait、1.5 中的 condition.await、以及可中断的通道上的 I/O 操作方法后可进入阻塞状态），线程在检查中断标示时如果发现中断标示为 true，则会在这些阻塞方法调用处抛出 InterruptedException 异常，并且在抛出异常后立即将线程的中断标示位清除，即重新设置为 false。而如果线程处于非阻塞状态，则需要通过判断 Thread.interrupted() 或者 Thread.isInterrupted() 来循环检测

{% blockquote %}

{% raw %}<i class="far fa-bell"></i> {% endraw%}

1. Synchronized 在获锁的过程中是不能被中断的，意思是说如果产生了死锁，则不可能被中断

2. LockSupport 的 park 方法阻塞，能够响应中断，但是不会抛出 InterruptedException 异常

3. 一个支持中断线程的程序的标准处理模式

   {% codeblock lang:java %}
   public void run() {
   ​    try {

   ​        // do something

   ​        // 1. !Thread.currentThread().isInterrupted() 确保在非阻塞时能响应中断
   ​        // 2. try-catch 后对 InterruptedException 处理确保阻塞时对中断进行处理
   ​        while (!Thread.currentThread().isInterrupted()&& more work to do) {
   ​            do more work 
   ​        }
   ​    } catch (InterruptedException e) {
   ​        //线程在 wait 或 sleep 期间被中断了
   ​    } finally {
   ​        //线程结束前做一些清理工作
   ​    }
   }
   {% endcodeblock %}
   {% endblockquote %}

在之前所说的 acquire，ascquireShared 方法均不支持中断操作

```java
if (shouldParkAfterFailedAcquire(p, node) &&
    parkAndCheckInterrupt())
    interrupted = true;
```

它们在 LockSupport.park 响应中断后只是置一个中断标记，但是并不会处理，仍然自旋获取锁直到获取成功或阻塞。而 acquireInterruptibly，acquireSharedInterruptibly 方法支持中断操作

```java
if (shouldParkAfterFailedAcquire(p, node) &&
    parkAndCheckInterrupt())
    throw new InterruptedException();
```

它们会在 LockSupport.park 响应中断后抛出 InterruptedException 异常结束线程

## 参考

- [AbstractQueuedSynchronizer 原理分析 - 独占/共享模式](https://segmentfault.com/a/1190000014721183)
- [AbstractQueuedSynchronizer的介绍和原理分析](http://ifeve.com/introduce-abstractqueuedsynchronizer/)
- [AbstractQueuedSynchronizer源码解读](https://www.cnblogs.com/micrari/p/6937995.html)