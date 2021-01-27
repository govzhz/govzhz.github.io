---
title: ReentrantLock
date: 2019-01-14 19:53:12
toc: true
tags:
 - Java
 - 锁
 - 多线程
categories: JUC
---

&emsp;&emsp;ReentrantLock 是基于 AQS 同步器实现的互斥锁，它支持设置公平锁/非公平锁模式，同时具有可重入性。在这里讨论 ReentrantLock 对这些特性的支持及应用。

## 标准模式

```java
class X {
    
    private final ReentrantLock lock = new ReentrantLock();
    
    public void m() {
    
        lock.lock();
        
        try {

            // ... method body
        
        } finally {
            
            lock.unlock();
        }
    }

}
```

<!-- more -->

## 简介

ReentrantLock 默认使用非公平锁，也可以通过显式的使用公平锁

```java
public ReentrantLock() {
    sync = new NonfairSync();
}

public ReentrantLock(boolean fair) {
    sync = fair ? new FairSync() : new NonfairSync();
}
```

公平锁 FairSync 和非公平锁 NonfairSync 均继承于内部类 Sync，而 Sync 继承 AQS（AbstractQueuedSynchronizer）锁。获取锁和释放锁均在 Sycn 中实现

```java
public void lock() {
    sync.lock();
}

public boolean tryLock() {
    return sync.nonfairTryAcquire(1);
}

public boolean tryLock(long timeout, TimeUnit unit)
        throws InterruptedException {
    return sync.tryAcquireNanos(1, unit.toNanos(timeout));
}

public void unlock() {
    sync.release(1);
}
```

## 非公平锁

```java
static final class NonfairSync extends Sync {
    private static final long serialVersionUID = 7316153563782823691L;

    final void lock() {
        if (compareAndSetState(0, 1))
            setExclusiveOwnerThread(Thread.currentThread());
        else
            acquire(1);
    }

    protected final boolean tryAcquire(int acquires) {
        return nonfairTryAcquire(acquires);
    }
}

abstract static class Sync extends AbstractQueuedSynchronizer {
    private static final long serialVersionUID = -5179523762034025860L;

    abstract void lock();

    /**
     * Performs non-fair tryLock.  tryAcquire is implemented in
     * subclasses, but both need nonfair try for trylock method.
     */
    final boolean nonfairTryAcquire(int acquires) {
        final Thread current = Thread.currentThread();
        int c = getState();
        if (c == 0) {
            if (compareAndSetState(0, acquires)) {
                setExclusiveOwnerThread(current);
                return true;
            }
        }
        else if (current == getExclusiveOwnerThread()) {
            int nextc = c + acquires;
            if (nextc < 0) // overflow
                throw new Error("Maximum lock count exceeded");
            setState(nextc);
            return true;
        }
        return false;
    }
}
```

在之前的 [AQS同步器](http://zzcoder.cn/2019/01/10/AbstractQueuedSynchronizer%E5%90%8C%E6%AD%A5%E5%99%A8/) 提到过，AbstractQueuedSynchronizer 为子类提供了需要实现的 tryAcquire 模板方法，非公平锁获取锁调用的底层核心方法是 nonfairTryAcquire。首先基于 AQS 实现获取互斥锁的标准实现：**当 state 为 0 时代表没有线程持有锁，因此尝试获取锁，如果获取锁成功则将当前线程设为持有锁的线程**：

```java
int c = getState();
if (c == 0) {
    if (compareAndSetState(0, acquires)) {
        setExclusiveOwnerThread(current);
        return true;
    }
}
```

但和普通的互斥锁不同的是，ReentrantLock 还需要支持可重入性：**当 state 不为 0（即存在线程持有锁），会继续判断持有锁的是否为当前线程，如果是则允许当前线程获取锁，并将 state + 1。**

```java
else if (current == getExclusiveOwnerThread()) {
    int nextc = c + acquires;
    if (nextc < 0) // overflow
        throw new Error("Maximum lock count exceeded");
    setState(nextc);
    return true;
}
```

那么释放逻辑也需要对重入性额外处理

```java
protected final boolean tryRelease(int releases) {
    int c = getState() - releases;
    if (Thread.currentThread() != getExclusiveOwnerThread())
        throw new IllegalMonitorStateException();
    boolean free = false;
    if (c == 0) {
        free = true;
        setExclusiveOwnerThread(null);
    }
    setState(c);
    return free;
}
```

**首先确保释放锁的线程为持有锁的线程，接下去确保重入次数和释放次数相同（即 state = 0）才认为释放锁完成，才会将持有锁的线程设为空。**

> {% raw %}<i class="far fa-bell"></i> {% endraw%}
>
> &emsp;&emsp;ReentrantLock 的非公平锁模式意味着多个线程获取锁的顺序并不是按照申请锁的顺序，会存在“线程饥饿”的问题

## 公平锁

```java
static final class FairSync extends Sync {
    private static final long serialVersionUID = -3000897897090466540L;

    final void lock() {
        acquire(1);
    }

    /**
     * Fair version of tryAcquire.  Don't grant access unless
     * recursive call or no waiters or is first.
     */
    protected final boolean tryAcquire(int acquires) {
        final Thread current = Thread.currentThread();
        int c = getState();
        if (c == 0) {
            // 先判断当前线程是否位于等待队列中的第一个
            if (!hasQueuedPredecessors() &&
                compareAndSetState(0, acquires)) {
                setExclusiveOwnerThread(current);
                return true;
            }
        }
        else if (current == getExclusiveOwnerThread()) {
            int nextc = c + acquires;
            if (nextc < 0)
                throw new Error("Maximum lock count exceeded");
            setState(nextc);
            return true;
        }
        return false;
    }
}
```

公平锁和非公平锁唯一的区别在于，它通过 hasQueuedPredecessors 确保当前线程是否位于等待队列中的第一个时才会尝试竞争锁

```java
if (!hasQueuedPredecessors() &&
    compareAndSetState(0, acquires)) {
    setExclusiveOwnerThread(current);
    return true;
}

public final boolean hasQueuedPredecessors() {
    // The correctness of this depends on head being initialized
    // before tail and on head.next being accurate if the current
    // thread is first in queue.
    Node t = tail; // Read fields in reverse initialization order
    Node h = head;
    Node s;
    return h != t &&
        ((s = h.next) == null || s.thread != Thread.currentThread());
}
```

那么也就是说**公平模式的获取锁会先判断当前线程是否位于等待队列中的第一个，若不是则直接加入等待队列来确保多个线程按照申请锁的顺序来获取锁**

> {% raw %}<i class="far fa-bell"></i> {% endraw%}
>
> &emsp;&emsp;公平模式可以解决线程饥饿问题，但相比非公平模式，也会使得更多的线程阻塞，产生更多 CPU 唤醒阻塞线程的开销而影响吞吐量

## Condition

Condition 是一个多线程间协调通信工具类，在 AQS 中实现，子类可以创建 Condition 实现类

```java
final ConditionObject newCondition() {
    return new ConditionObject();
}
```

### await

```java
public final void await() throws InterruptedException {
    if (Thread.interrupted())
        throw new InterruptedException();
    
    // 添加到 Condition 队列
    Node node = addConditionWaiter();
    
    // 释放锁
    int savedState = fullyRelease(node);
    int interruptMode = 0;
    
    // 判断是否在 AQS 队列，如果不在则阻塞
    // 唤醒时会将当前线程重新插入 AQS 队列尾
    while (!isOnSyncQueue(node)) {
        LockSupport.park(this);
        if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
            break;
    }
    
    // 自旋获取锁直到重新阻塞
    if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
        interruptMode = REINTERRUPT;
    if (node.nextWaiter != null) // clean up if cancelled
        unlinkCancelledWaiters();
    if (interruptMode != 0)
        reportInterruptAfterWait(interruptMode);
}
```

Condition 存在自己的队列，在 Condition 队列就意味着线程需要 signal 方法唤醒。await 方法主要做以下几步：

1. 将当前线程加入 Condition 队列尾
2. 释放锁，即从 AQS 队列中退出（因此线程不会同时存在于 AQS 队列和 Condition 队列）
3. 阻塞当前线程等待唤醒（唤醒时会将当前线程重新插入 AQS 队列尾，然后当它的前驱结点释放锁后 unpark 唤醒，唤醒后自旋/阻塞获取锁）

### signal

```java
public final void signal() {
    
    // 确保当前线程持有锁
    if (!isHeldExclusively())
        throw new IllegalMonitorStateException();
    Node first = firstWaiter;
    if (first != null)
        
        // 将 Condition 队列中的首结点加入 AQS 队列
        doSignal(first);
}
```

signal 方法用于唤醒处于 Condition 队列中的首结点，但注意它并不是立刻唤醒

```java
private void doSignal(Node first) {
    do {
        // 移除头结点
        if ( (firstWaiter = first.nextWaiter) == null)
            lastWaiter = null;
        first.nextWaiter = null;
    } while (!transferForSignal(first) &&
             (first = firstWaiter) != null);
}

final boolean transferForSignal(Node node) {

    if (!compareAndSetWaitStatus(node, Node.CONDITION, 0))
        return false;

    Node p = enq(node);
    int ws = p.waitStatus;
    
    // 仅在前驱节点的状态处于取消状态或设置前驱节点状态为 SIGNAL 失败时才会直接唤醒
    // 大部分情况都不会在这里唤醒
    if (ws > 0 || !compareAndSetWaitStatus(p, ws, Node.SIGNAL))
        LockSupport.unpark(node.thread);
    return true;
}
```

signal 方法的主要逻辑如下：

1. 首先它会将头结点从 Condition 队列取出
2. 然后通过 enq 将当前线程加入 AQS 队列尾
3. 仅在前驱节点的状态处于取消状态或设置前驱节点状态为 SIGNAL 失败时才会直接唤醒，否则是等待它在 AQS 队列的前驱结点释放锁后唤醒（这样它的前驱结点为头结点，它才有资格获取锁，唤醒才有意义）

### signalAll

```java
public final void signalAll() {
    if (!isHeldExclusively())
        throw new IllegalMonitorStateException();
    Node first = firstWaiter;
    if (first != null)
        doSignalAll(first);
}

private void doSignalAll(Node first) {
    lastWaiter = firstWaiter = null;
    do {
        Node next = first.nextWaiter;
        first.nextWaiter = null;
        transferForSignal(first);
        first = next;
    } while (first != null);
}
```

signalAll 和 signal 的区别就在于它会遍历 Condition 队列，把所有 Condition 队列中的结点放入 AQS 队列等待唤醒。

### 应用

一个经典的应用：生产者/消费者模式

```java
import java.util.LinkedList;
import java.util.Queue;
import java.util.Random;
import java.util.concurrent.locks.Condition;
import java.util.concurrent.locks.ReentrantLock;

public class ProducerConsumer {

    private Queue<Integer> queue;
    private Integer max;
    private final ReentrantLock lock = new ReentrantLock();
    private final Condition empty = lock.newCondition();
    private final Condition full = lock.newCondition();

    public ProducerConsumer(Queue<Integer> queue, Integer max) {
        this.queue = queue;
        this.max = max;
    }

    public void produce(){
        new Thread(() -> {
            Random random = new Random();
            for(;;){
                lock.lock();
                try {
                    while (queue.size() >= max) {
                        try {
                            full.await();
                        } catch (InterruptedException e) {
                            e.printStackTrace();
                        }
                    }

                    int num = random.nextInt();
                    if(queue.size() >= max) {
                        break;
                    }
                    queue.offer(num);
                    empty.signalAll();
                }finally {
                    lock.unlock();
                }
            }
            System.out.println("Not safe");
        }).start();
    }

    public void consume(){
        new Thread(() -> {
            for(;;) {
                lock.lock();
                try {
                    while (queue.isEmpty()) {
                        try {                
                            empty.await();
                        } catch (InterruptedException e) {
                            e.printStackTrace();
                        }
                    }
                    queue.poll();
                    full.signalAll();
                } finally {
                    lock.unlock();
                }
            }
        }).start();
    }

    public static void main(String[] args) {
        Queue<Integer> queue = new LinkedList<>();
        int max = 10;
      
        ProducerConsumer producerConsumer = new ProducerConsumer(queue, max);
        producerConsumer.produce();
        producerConsumer.produce();
        producerConsumer.consume();
        producerConsumer.consume();
    }

}
```

正常情况，不会出现 `Not safe`

> {% raw %}<i class="far fa-bell"></i> {% endraw%}
>
> &emsp;&emsp;Synchronized + wait/notify 的组合和 Lock + Condition 组合具有类似的功能，性能上的差别也不是很大，但它们仍然有许多区别。这里举几个典型的例子：
>
> - Lock + Condition 可以选择公平/非公平模式，而 Synchronized + wait/notify 只能是非公平的
> - Lock + Condition 可以唤醒指定 Condition，而 Synchronized + wait/notify 不能指定
> - Lock + Condition 可以设置超时时间，而 Synchronized + wait/notify 只能等待唤醒或中断

