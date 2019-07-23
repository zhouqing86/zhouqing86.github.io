---
layout: post
comments: false
categories: Java基础
date:   2019-06-15 00:30:54
title: AbstractQueuedSynchronizer源码阅读 Java 1.8
---

<div id="toc"></div>

> 本文基于Java 1.8.0_121-b13版本

AbstractQueuedSynchronizer类是用来构建锁或者其他同步组件的基础框架类。

因为此文较长，后续的针对成员变量和逻辑的分析需要花时间阅读。大部分人应该只是想大概了解下AbstractQueuedSynchronizer的使用，可以直接跳到使用章节。

AbstractQueuedSynchronizer被许多并发工具类类所使用或直接继承，如`ReentrantLock`,`ReentrantReadWriteLock`,`CountDownLatch`, `Semaphore`和`ThreadPoolExecutor`。

## 什么是AbstractQueuedSynchronizer

这里先引用其源码类里的一段英文说明:

```
Provides a framework for implementing blocking locks and related synchronizers (semaphores, events, etc) that rely on first-in-first-out (FIFO) wait queues.This class is designed to be a useful basis for most kinds of synchronizers that rely on a single atomic int value to represent state.
```

`AbstractQueuedSynchronizer`是一个`抽象类`，但是英文说明称其为`框架`，瞬间高大上了。这个类为`依赖先进先出队列`的`阻塞锁`和`相关同步器`提供了基础实现。对于大部分同步器，只是依赖于对于一个整型`state`的同步修改。

这里其实可以得到很多的信息：

- `AbstractQueuedSynchronizer`中使用了先进先出队列

- `AbstractQueuedSynchronizer`有一个整型的`state`

而同步器`synchronizer`背后的`acquire`逻辑是:

```
while (synchronization state does not allow acquire) {
  enqueue current thread if not already queued;
  possibly block current thread;
}
dequeue current thread if it was queued;
```

`release`的逻辑是:

```
update synchronization state;
if (state may permit a blocked thread to acquire)
unblock one or more queued threads;
```

## 静态类

### Node

Node子类是为了等待队列而设计的。等待队列是一个支持CLH锁的队列。CLH锁常用的是自旋锁，而这里使用的是阻塞同步器的方式。

#### CLH锁

CLH锁即Craig, Landin, and Hagersten (CLH) locks。CLH锁是一个自旋锁。能确保无饥饿性。提供先来先服务的公平性。

在Doug Lead论文里AQS使用CLH(Craig, Landin, and Hagersten)锁的原因是其比MCS(Mellor-Crummey and Scott)锁更容易去处理取消和超时。

CLH一般的实现如下，AQS中的实现略有不同。

```
public class CLHLock implements Lock {
    AtomicReference<QNode> tail = new AtomicReference<QNode>(new QNode());
    ThreadLocal<QNode> myPred;
    ThreadLocal<QNode> myNode;

    public CLHLock() {
        tail = new AtomicReference<QNode>(new QNode());
        myNode = new ThreadLocal<QNode>() {
            protected QNode initialValue() {
                return new QNode();
            }
        };
        myPred = new ThreadLocal<QNode>() {
            protected QNode initialValue() {
                return null;
            }
        };
    }

    @Override
    public void lock() {
        QNode qnode = myNode.get();
        qnode.locked = true;
        QNode pred = tail.getAndSet(qnode);
        myPred.set(pred);
        while (pred.locked) {
        }
    }

    @Override
    public void lockInterruptibly() throws InterruptedException {

    }

    @Override
    public boolean tryLock() {
        return false;
    }

    @Override
    public boolean tryLock(long time, TimeUnit unit) throws InterruptedException {
        return false;
    }

    @Override
    public void unlock() {
        QNode qnode = myNode.get();
        qnode.locked = false;
        myNode.set(myPred.get());
    }

    @Override
    public Condition newCondition() {
        return null;
    }

    static class QNode {
        public boolean locked;
    }
}
```

AQS中的CLH Queue维护着一个双向同步队列。不仅仅有`tail`节点，也定义了`head`节点。对于每个节点，不仅仅有`pred`，也有`next`。Doug Lea在他的论文里说明为什么要引入`next`：

```
The next link is treated only as an optimized path. If a node's successor does not appear to exist (or appears to be cancelled) via its next field, it is always possible to start at the tail of the list and traverse backwards using the pred field to accurately check if there really is one.
```

AQS中的同步队列的Node节点并没有像常规的CLH Queue一样对状态进行自旋。因为AQS并不是真正需要一个`lock`方法。其`acquire`方法会首先尝试调用`tryAcquire`方法来获取资源，如果没有获取成功，则创建Node节点并插入到同步队列中，同时当前线程会进行`LockSupport.park`操作。

#### 成员变量

- `static final Node SHARED = new Node()`共享锁静态变量，当前节点为共享锁时，将复制`nextWaiter`为`SHARED`

- `static final Node EXCLUSIVE = null`独占锁静态变量

- `volatile int waitStatus`，Node节点的状态，初始化为0，waitStatus>0表示取消状态，而waitStatus<0表示有效状态。
  值为-1表示`SIGNAL`，被标识为该等待唤醒状态的后继结点，当其前继结点的线程释放了同步锁或被取消，将会通知该后继结点的线程执行。说白了，就是处于唤醒状态，只要前继结点释放锁，就会通知标识为SIGNAL状态的后继结点的线程执行。
  值为-2表示`CONDITION`，与Condition相关，该标识的结点处于等待队列中，结点的线程等待在Condition上，当其他线程调用了Condition的signal()方法后，CONDITION状态的结点将从等待队列转移到同步队列中，等待获取同步锁。
  值为-3表示`PROPAGATE`，与共享模式相关，在共享模式中，该状态标识结点的线程处于可运行状态。

- `volatile Node prev`，指向当前节点的前向节点

- `volatile Node next`，指向当前节点的后续节点

- `volatile Thread thread`，当前节点对应的线程

- `Node nextWaiter`，注意不要被`nextWaiter`的变量名锁迷惑，其可以指向下一个等待某个条件的节点，或特殊表示为静态变量`SHARED`。

#### 方法

- `Node(Thread thread, Node mode) `构造函数提供给AQS的`addWaiter`使用，创建`Node`节点时设置其`thread`和`nextWaiter`的值。

- `Node(Thread thread, int waitStatus)`构造函数提供给条件等待队列使用，条件等待代理的每个节点也是`Node`对象，这样如果条件满足，可以很方便的把某一个节点从条件等待队列迁移到AQS的等待队列中。

- `isShared`方法判断是不是共享模式节点，返回`nextWaiter == SHARED`

- `predecessor`返回前置节点`prev`，注意这里会对`prev`是否为null做判断，如果为null就会抛出`NullPointerException`异常

### ConditionObject

#### Condition接口

Condition接口中定义的方法是`await`, `signal`和`signalAll`:

- 其与Object种的`wait`, `notify`和`notifyAll`是不同的，Object里的这些方法是配合`synchronized`关键字来使用。而Condition接口的这些方法往往是配合`Lock`来使用。

- 当每个锁上有多个等待条件时，可以优先使用Condition，这样可以具体一个Condition控制一个条件等待

- `void await() throws InterruptedException` 如果当前线程中断，则抛出`InterruptedException`异常

- `void awaitUninterruptibly()`方法意味着即便线程被中断，Condition将继续等待

- `long awaitNanos(long nanosTimeout) throws InterruptedException`方法将返回剩余的纳秒数，如果前线程中断，则抛出`InterruptedException`异常

- `boolean await(long time, TimeUnit unit) throws InterruptedException;`,如果等待的时间消耗完，返回false，否则方法结束时返回时为true

- `boolean awaitUntil(Date deadline) throws InterruptedException;`,等待某个绝对时间为止

#### 实现

ConditionObject中有两个变量：

- `private transient Node firstWaiter;`条件队列的第头结点

- `private transient Node lastWaiter;`条件队列的最后一个结点

实现的方法：

- addConditionWaiter方法，创建新结点并使其成为条件等待队列的最后一个结点。此方法中也考虑移除已经被取消的节点

- signal方法，通知条件等待队列的头结点，如果头结点对应的线程没有被取消的话，就从条件等待队列中移除并加入AQS的等待队列，在Doug Lea论文中描述其为

  ```
  transfer the first node from condition queue to lock queue;
  ```

- signalAll方法，将通知整个等待队列，并将等待队列的所有节点都迁移至AQS的等待队列

- await方法，这个方法是允许中断的。此方法会创建节点并添加到条件等待队列中，而后会释放占用了的AQS的状态。线程会park直到其不在条件等待队列中，否则其就在AQS的等待队列中，去尝试竞争锁。在Doug Lea的论文中，描述await的逻辑步骤:

  ```
  create and add new node to condition queue;
  release lock;
  block until node is on lock queue;
  re-acquire lock;
  ```

## 一些常量和成员变量

- `private transient volatile Node head;`，等待队列的头结点

- `private transient volatile Node tail`，等待队列的尾结点

- `private volatile int state;`，AQS中的同步状态

- `private static final Unsafe unsafe = Unsafe.getUnsafe();` 为了支持`CAS`提供的`Unsafe`对象

- 其他一些支持`CAS的对象`

```
//state成员变量的偏移
private static final long stateOffset;

//head成员变量的偏移
private static final long headOffset;

//tail成员变量的偏移
private static final long tailOffset;

//Node对象的waitStatus成员变量的偏移
private static final long waitStatusOffset;

//Node对象的nextOffset成员变量的偏移
private static final long nextOffset;
```

## 定义的重要方法

这里并不会对每个方法的逻辑进行分析，只是简单给出方法列表

public方法：

- acquire

- acquireInterruptibly

- acquireShared

- acquireSharedInterruptibly

- release

- releaseShared

- tryAcquireSharedNanos

- hasQueuedThreads

- hasContended

- getFirstQueuedThread

- isQueued

- hasQueuedPredecessors

- getQueueLength

- getQueuedThreads

- getExclusiveQueuedThreads

- getSharedQueuedThreads

- owns

- hasWaiters

- getWaitingThreads


非public方法：

- acquireQueued

- apparentlyFirstQueuedIsExclusive

- isOnSyncQueue

- transferForSignal

- fullyRelease

私有方法：

- addWaiter

- setHead

- unparkSuccessor

- doReleaseShared

- setHeadAndPropagate

- cancelAcquire

- shouldParkAfterFailedAcquire

- parkAndCheckInterrupt

- doAcquireInterruptibly

- doAcquireNanos

- doAcquireShared

- doAcquireSharedInterruptibly

- doAcquireSharedNanos

- fullGetFirstQueuedThread

- findNodeFromTail


## 使用

定义子类来继承`AbstractQueuedSynchronizer`，覆盖如下方法：

- tryAcquire

- tryRelease

- tryAcquireShared

- tryReleaseShared

- isHeldExclusively

定义的子类中可以使用AQS子类提供的`getState`，`setState`和`compareAndSetState`等方法。

经典实践：

- tryAcquire 使用 ""test-and-test-and-set"的方式

### 自定义Mutex

```
class Mutex {
  class Sync extends AbstractQueuedSynchronizer {
    public boolean tryAcquire(int ignore) {
      return compareAndSetState(0, 1);
    }
    public boolean tryRelease(int ignore) {
      setState(0);
      return true;
    }
  }
  private final Sync sync = new Sync();
  public void lock() { sync.acquire(0); }
  public void unlock() { sync.release(0); }
}
```

### 独占还是共享

AQS中`acquire`方法是为排它锁设计的，而`acquireShared`方法是为共享锁设计的。两者的不同之处也在于其调用函数的不同。

`acquire`调用的是`tryAcquire`方法，此方法定义为`protected boolean tryAcquire(int arg)`。意味着一个线程获取状态时只能是`true`或`false`。

而`acquireShared`调用的是`tryAcquireShared`，此方法定义为`protected int tryAcquireShared(int arg)`。当`tryAcquireShared`返回为负则获取状态state失败，0表示获取状态成功但剩下的状态为0，如果大于0则表示剩下的状态的值。

使用`tryAcquire`或`tryAcquireShared`获取到状态失败，在代码实现层面

- 独占模式的方法`acquire`主要调用了`boolean acquireQueued(final Node node, int arg) `方法

- 共享模式的方法`acquireShared`主要调用了`void doAcquireShared(int arg)`方法

- `acquireQueued`和`doAcquireShared`实现的逻辑基本差不多

  ```
  将节点添加添加到等待队列
  在一个for死循环中：
    检查当前节点的前节点是否为等待队列的head
      尝试获取可用state值，获取成功则设置当前节点为等待队列的头结点（且将当前结点的thread和prev都设置为空)
      获取可用state值失败则先判断是否需要park当前线程，如果需要park则对想成进行LockSupport.park
  ```

- 共享模式下的设置当前头结点函数`setHeadAndPropagate`

  ```
  //设置等待队列的head结点，并检查后续处于SHARE模式下的等待节点。
  private void setHeadAndPropagate(Node node, int propagate) {
      Node h = head; // Record old head for check below
      setHead(node);

      // propagate > 0表示调用者要求需要往下传递
      // 检查setHead前和setHead后的head节点的waitStatus状态，注意这里并不是h.waitStatus==PROPAGATE
      // 因为PROPAGATE可能会转换为SIGNAL，这里会存在不必要的线程唤醒，but only when there are multiple racing acquires/releases
      if (propagate > 0 || h == null || h.waitStatus < 0 ||
          (h = head) == null || h.waitStatus < 0) {
          Node s = node.next;

          //Next结点为空，则其可能为一个不知道的中间状态
          if (s == null || s.isShared())
              doReleaseShared();
      }
  }
  ```

  `doReleaseShared`方法会通知下一个节点且保证这种通知能传递到后续节点，所以引入了for死循环去处理；而独占模式的`release`只需要调用head节点的`unparkSuccessor`即可，不需要考虑后续节点。

### 公平和不公平

通过AQS支持对公平性的控制，尽管等待队列是基于FIFO队列，同步器可以是不公平的，那么如何实现这种不公平呢？acquire方法中，`tryAcquire`的调用是先于队列操作的，这样新的获取线程可能会偷取状态。

```
public final void acquire(int arg) {
    if (!tryAcquire(arg) &&
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();
}
```

不公平体现在没有加入等待队列的当前线程在获取状态时，会与等待队列的头结点竞争，会出现等待队列头结点没有竞争到状态，但没有加入到等待队列的线程相反获取竞争到了状态，造成了不公平的状态。不过这种方式能提供更好的性能。Doug Lea中论文中如实说：

```
This barging FIFO strategy generally provides higher aggregate
throughput than other techniques. It reduces the time during
which a contended lock is available but no thread has it because
the intended next thread is in the process of unblocking.
```

前面说的是不公平，那如果需要强公平性，则需要在实现`tryAcquire`时添加逻辑：如如果当前进程不再队列头，则直接返回false。

最后我们看看ReentrantLock中的`FairSync`和`NonfairSync`是怎么实现的:

FairSync中的`tryAcquire`方法`Don't grant access unless recursive call or no waiters or is first`:

```
protected final boolean tryAcquire(int acquires) {
   final Thread current = Thread.currentThread();
   int c = getState();

   //状态为0标识号是没有被锁
   if (c == 0) {
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
```

这里的`hasQueuedPredecessors`方法就是为公平锁而设计的：

```
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

如果此函数返回true，则能确保当前线程是在等待队列的队列头。要想看懂`hasQueuedPredecessors`方法方法，也需要了解下`enq`方法，了解head，tail初始化时是一个`new Node()`:

```
private Node enq(final Node node) {
    for (;;) {
        Node t = tail;
        if (t == null) { // Must initialize
            if (compareAndSetHead(new Node()))
                tail = head;
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

### 超时

在实现中，在对临界区进行竞争时，我们并不希望无限等待，希望有超时机制来保证我们等待一段时间如果还没有获取到锁则退出竞争。
AQS中为了满足这种需求提供了方法如`public final boolean tryAcquireNanos(int arg, long nanosTimeout)`。

关于`tryAcquireNanos`:

- 独占模式使用，如果线程被中断，将抛出InterruptedException异常

- 如果超时未竞争到对state的获取，返回false

- 在具体实现中也会调用一次`tryAcquire`方法，如果没有获取到，则调用`doAcquireNanos`方法

  ```
  private boolean doAcquireNanos(int arg, long nanosTimeout)
          throws InterruptedException {
      if (nanosTimeout <= 0L)
          return false;
      final long deadline = System.nanoTime() + nanosTimeout;
      final Node node = addWaiter(Node.EXCLUSIVE);
      boolean failed = true;
      try {
          for (;;) {
              final Node p = node.predecessor();
              if (p == head && tryAcquire(arg)) {
                  setHead(node);
                  p.next = null; // help GC
                  failed = false;
                  return true;
              }
              nanosTimeout = deadline - System.nanoTime();
              if (nanosTimeout <= 0L)
                  return false;
              if (shouldParkAfterFailedAcquire(p, node) &&
                  nanosTimeout > spinForTimeoutThreshold)
                  LockSupport.parkNanos(this, nanosTimeout);
              if (Thread.interrupted())
                  throw new InterruptedException();
          }
      } finally {
          if (failed)
              cancelAcquire(node);
      }
  }
  ```

  其基本逻辑与`acquireQueued`方法类似，只是增加了对于超时的处理。在for循环中，如果`tryAcquire`没有没有竞争到state状态，会首先判断有没有超时，如果超时就直接返回。否则如果需要park当前线程，则调用LockSupport.parkNanos来park一段时间。

  另对于共享模式，也有`tryAcquireSharedNanos`和`doAcquireSharedNanos`的存在。

### 中断和非中断

在介绍AQS中的中断之前，我们先了解下`Thread`的`interrupt`：

- 此操作会将线程的中断标志位置位。如果线程sleep()处于timed_waiting状态，则还会抛出InterruptException；如果线程在运行态则不会受此影响。

- isInterrupted()可以用来读取线程的中断标志位，如果线程已经被中断则返回true，否则返回false

- interrupted()方法读取线程的中断标志位`Tests whether this thread has been interrupted.`，并会重置，重置的意思如果这个方法两次接连调用，第二次调用将会返回false，除非在第二次调用之前线程又被中断了

这里很重要的一点是当线程在运行时，调用Thread的`interrupt`方法只是改变线程中断标志位的状态，并不会抛出异常，也并不会真正退出线程。

在AQS，有`acquireInterruptibly`和`acquireSharedInterruptibly`两个方式支持当线程被中断时会终止线程处理。

这里拿`acquireInterruptibly`来看：

```
public final void acquireInterruptibly(int arg)
        throws InterruptedException {
    if (Thread.interrupted())
        throw new InterruptedException();
    if (!tryAcquire(arg))
        doAcquireInterruptibly(arg);
}
```

- 当线程是已经被中断，则直接返回`InterruptedException`

- 调用`doAcquireInterruptibly`中有一段逻辑会抛出`InterruptedException`异常

  ```
  if (shouldParkAfterFailedAcquire(p, node) &&
      parkAndCheckInterrupt())
      throw new InterruptedException();
  ```

  这段逻辑与`acquireQueued`中明显不同，acquireQueued是不支持中断终止线程的。acquireQueued中只是改变了`interrupted`的值，并且即使改变了`interrupted`的值也不会返回，要等到`tryAcquire`竞争到state状态才返回。

  ```
  if (shouldParkAfterFailedAcquire(p, node) &&
      parkAndCheckInterrupt())
      interrupted = true;
  ```

  如果`acquireQueued`返回了true，意味着中断标志位为true，`acquire`就会调用`selfInterrupt`方法，`selfInterrupt`也不会抛出异常，只是调用了`Thread.currentThread().interrupt()`。

## JDK中AQS的一些使用者

### ReentrantLock直接使用了AQS

ReentrantLock支持公平锁和非公平锁，公平锁内部实现FairSync,非公平锁内部实现是NonfairSync。

`FairSync`和`NonfairSync`最终都是集成了AQS。为什么不是ReentrantLock直接继承AQS呢？因为ReentrantLock是实现了锁的接口，其并不需要暴露AQS的所有共有方法。

其中关于非公平锁的`tryAcquire`的实现:

```
protected final boolean tryAcquire(int acquires) {
    return nonfairTryAcquire(acquires);
}
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
```

公平锁的`tryAcquire`实现:

```
protected final boolean tryAcquire(int acquires) {
    final Thread current = Thread.currentThread();
    int c = getState();
    if (c == 0) {
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
```

关于这里的公平与不公平可以查看`使用`章节的`公平和不公平`。

### CountDownLatch直接使用了AQS

CountDownLatch的`await`方法调用的是`sync.acquireSharedInterruptibly(1);`, Sync内部类是基于共享模式的AQS子类。

```
private static final class Sync extends AbstractQueuedSynchronizer {
    private static final long serialVersionUID = 4982264981922014374L;

    Sync(int count) {
        setState(count);
    }

    int getCount() {
        return getState();
    }

    protected int tryAcquireShared(int acquires) {
        return (getState() == 0) ? 1 : -1;
    }

    protected boolean tryReleaseShared(int releases) {
        // Decrement count; signal when transition to zero
        for (;;) {
            int c = getState();
            if (c == 0)
                return false;
            int nextc = c-1;
            if (compareAndSetState(c, nextc))
                return nextc == 0;
        }
    }
}
```

### CyclicBarrier没有直接使用AQS

可以看到CyclicBarrier并没有直接使用AQS来定义自己的Sync同步辅助类。但是其使用的ReentrantLock默认定义了NonfairSync。

CyclicBarrier对失败的同步尝试使用全或无中断模型：如果某个线程由于中断、故障或超时而过早地离开屏障点，那么在该屏障点等待的所有其他线程也将通过{BrokenBarrierException}异常离开(或者{InterruptedException}如果它们同时被中断)。

CyclicBarrier使用ReentranLock进行加锁，使用Condition的await、signal及signalAll方法进行线程间通信。

成员变量:

```
/** 定义一个排他可重入锁 */  
private final ReentrantLock lock = new ReentrantLock();  

/** 创建一个等待队列 */  
private final Condition trip = lock.newCondition();  
```

如何当所有线程都到达某一点时，唤醒所有线程呢？

```
private void breakBarrier() {  
    //设置一代线程组对应的栅栏状态为true（意思就是栅栏被破坏了）  
    generation.broken = true;  
    //重新设置计数count值  
    count = parties;  
    //唤醒所有等待的线程  
    trip.signalAll();  
}
```

其`dowait`方法的代码如:

```
private int dowait(boolean timed, long nanos)
    throws InterruptedException, BrokenBarrierException,
           TimeoutException {
    final ReentrantLock lock = this.lock;

    //先上锁，防止竞争
    lock.lock();
    try {
        final Generation g = generation;

        if (g.broken)
            throw new BrokenBarrierException();

        if (Thread.interrupted()) {
            breakBarrier();
            throw new InterruptedException();
        }

        int index = --count;
        //最后一个线程到达了位置
        if (index == 0) {  // tripped
            boolean ranAction = false;
            try {
                final Runnable command = barrierCommand;
                if (command != null)
                    command.run();
                ranAction = true;

                //将调用trip.signalAll通知所有线程，ConditionObject的signalAll其实是把条件等待队列的Node全部迁移到AQS的同步队列中
                //同时其会更新generation为new Generation()，表示其可以可以支持下一线程组的加入。
                nextGeneration();

                return 0;
            } finally {
                if (!ranAction)

                    //调用trip.signalAll通知所有线程
                    breakBarrier();
            }
        }

        // loop until tripped, broken, interrupted, or timed out
        for (;;) {
            try {
                if (!timed)
                    //条件的await方法会释放掉队锁的占用，同时当前线程加入条件等待队列进行等待，那么其他线程就能获取锁了
                    trip.await();
                else if (nanos > 0L)
                    nanos = trip.awaitNanos(nanos);
            } catch (InterruptedException ie) {
                if (g == generation && ! g.broken) {
                    breakBarrier();
                    throw ie;
                } else {
                    // We're about to finish waiting even if we had not
                    // been interrupted, so this interrupt is deemed to
                    // "belong" to subsequent execution.
                    Thread.currentThread().interrupt();
                }
            }

            if (g.broken)
                throw new BrokenBarrierException();

            if (g != generation)
                return index;

            if (timed && nanos <= 0L) {
                breakBarrier();
                throw new TimeoutException();
            }
        }
    } finally {
        lock.unlock();
    }
}
```


### FutureTask没有使用AQS

FutureTask在Java 1.6底层是使用了`AbstractQueuedSynchronizer`，但是Java 1.7后不再使用。原因是这个BUG:

[JDK-8016247 : ThreadPoolExecutor may interrupt the wrong task](https://bugs.java.com/bugdatabase/view_bug.do?bug_id=8016247)

之前的`FutureTask.Sync.innerCancel`方法中，如下实现会有问题：

```
if (mayInterruptIfRunning) {
    Thread r = runner;
    if (r != null)
        r.interrupt();
}
```

因为我们使用线程池时：

```
ThreadPoolExecutor executor = ...;
executor.submit(task1).cancel(true);
executor.submit(task2);
```

可能会出现在`r.interrupt()`之前线程的task就停止了。这时线程池如果在当前线程上分配task2，如果线程继续执行`r.interrupt()`，就会把task2终止掉。


## 参考文章

- [AbstractQueuedSynchronizer源码解读](https://www.cnblogs.com/micrari/p/6937995.html)

- [Doug Lea的AQS论文](http://gee.cs.oswego.edu/dl/papers/aqs.pdf)

- [AQS源码分析](https://www.jianshu.com/p/497a8cfeef63)

- [CLH队列锁](https://blog.csdn.net/varyall/article/details/80317488)

- [Java并发之AQS详解](https://www.cnblogs.com/waterystone/p/4920797.html)

- [Thread详解](https://www.cnblogs.com/waterystone/p/4920007.html)

<script type="text/javascript">
$(document).ready(function() {
    $('#toc').toc({ listType: 'ul', title: "<i>目录</i>" });
});
</script>
