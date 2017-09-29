---
layout: post
comments: false
categories: "Java基础"
date:   2017-09-29 00:00:54
title: Java多线程介绍
---

<div id="toc"></div>

偶然的契机又接触了一把Java的多线程，按照现在的习惯，学习东西的时候都是要总结一下的，如是有了这篇文章。

多线程在Java上看来都是并行的，即便CPU只有一个物理处理器。处理器会做时间分片，每个时间分片里只有一个线程运行。但是由于时间分片很小，所以看起来两个线程就是并行执行的。而如果CPU有多处理器，Java程序是不需要做任何修改的，但是不同处理器上的线程是真正的同时运行。

Java的作者在设计Java语言就考虑到了对多线程的支持，Object类的wait()和notify()方法，synchronized关键字，java.lang包下的Thread, ThreadGroup。Java程序中可以很快的入门多线程编程，但是想成为多线程编程的大师却是很难的。


## 开启多线程

> Take is cheaper, show me the code

### Thread类与Runnable接口

在Java 1.5之前有两种方式可以创建可以当做线程运行的类：

- 继承自Thread类，覆盖run方法

  ```
  public class TwoThread extends Thread{
      @Override
      public void run() {
          for (int i = 0; i < 10; i++) {
              System.out.println("New Thread!");
          }
      }

      public static void main(String[] args) {
          TwoThread tt = new TwoThread();
          tt.start();
      }
  }
  ```

- 实现Runnable接口
  ```
  public class TwoThreadWithRunnable implements Runnable{
      @Override
      public void run() {
          for (int i = 0; i < 10; i++) {
              System.out.println("New Thread with runnable!");
          }
      }

      public static void main(String[] args) {
          TwoThreadWithRunnable tt = new TwoThreadWithRunnable();
          new Thread(tt).start();
      }
  }
  ```

从实现方式看出，两者的区别在于一个是集成而另一个是接口，Runnable接口的使用是更灵活的，而且Runnable接口可以更方便的在线程之间共享数据。另本身Thread类也实现了Runnable接口。

另需要注意的是通过start方法来启动线程，新启的线程在可运行状态，等待CPU调度其到运行状态。而直接调用线程的run方法无法达到新启线程的效果。

Thread类相关常用的一些方法

- Thread.currentThread()获取当前代码所运行的线程

- Thread.currentThread().setName()与Thread.currentThread().getName()。建议调用start之前setName

- 查看某个thread是否还在运行, tt.isAlive

- Thread.sleep是一个可以让线程休息一段时间的方法

- Thread的构造方法，Thread(), Thread(Runnable target), Thread(ThreadGroup, Runnable, String)等

- 获取进程的优先级， t.getPriority(), setPriority()可以修改优先级

### Callable接口

Java 1.5后引入引入了Callable接口，与Runnable接口定义稍有不同，下面直接贴代码做对比。

```
@FunctionalInterface
public interface Callable<V> {
    V call() throws Exception;
}
```

```
@FunctionalInterface
public interface Runnable {
    void run();
}
```

这里先忽略Java 1.8增加的@FunctionalInterface接口。
从代码中可以看出这两个接口的明显区别是，call方法是需要返回对象V的。而run方法不需要返回方法。
这就意味着call方法可以返回线程计算后的值。那么问题来了，如何将多个Callable接口的返回的值合并到一起呢？譬如将不同线程计算的结果相加。这时我们就需要用到Future类了。

```
import java.util.LinkedList;
import java.util.List;
import java.util.Random;
import java.util.concurrent.*;

class TestCallable implements Callable<Integer>{

    @Override
    public Integer call() throws Exception {
        System.out.println(Thread.currentThread().getName() + " are running");
        return new Random().nextInt(100);
    }
}

public class MyCallable {
    public static void main(String[] args) {
        List<Future<Integer>> futureList = new LinkedList<>();
        ExecutorService executorService = Executors.newFixedThreadPool(10);
        for(int i=0; i<1000; i++) {
            futureList.add(executorService.submit(new TestCallable()));
        }

        int sum = 0;
        for(Future<Integer> future: futureList) {
            try {
                sum += future.get();
            } catch (InterruptedException e) {
                e.printStackTrace();
            } catch (ExecutionException e) {
                e.printStackTrace();
            }
        }

        System.out.println("main thread exit! sum=" + sum);
    }
}
```

这里用到线程池，固定大小为10个线程，代码中使用futureList来存储每一个被调用的TestCallable。这里TestCallable创建了1000个TestCallable来产生100以内的随机整数。

而后，通过future.get()去获取每个TestCallable中call方法返回的结果。最后加和求得1000个线程产生的随机整数的和。关于Future的get方法，其获取异步执行的结果，如果没有结果可用，此方法会阻塞直到异步计算完成。

所以在从futureList中获取结果部分是同步的：获取完线程1的结果再获取线程2的结果，如此依次。那能不能稍做优化呢，如先结束的线程可以把计算结果先加到总和中，当然是可以的，这里就要用到。

```
sum = 0;
try {
    for(int i=0; i<1000; i++) {
        Future<Integer> future = completionService.take();
        sum += future.get();
    }
} catch (InterruptedException e) {
    e.printStackTrace();
} catch (ExecutionException e) {
    e.printStackTrace();
}
```

## 线程同步

但线程程序是不需要考虑线程同步问题，但是多个线程同时操作某个变量时，需要考虑线程同步问题。如何来解释同步问题呢，看如下代码：

```
i = 10;
i = i + 1;
```

如果两个线程A, B同时走到i = i + 1; 在做计算之前，A, B获取的值都是10， 两个线程都做了i+1操作，而后写入i, 最后i的值将是11。而不是我们期望的12。所以实际上两个线程运行的环境，i = i + 1后的值可能是11，也可能是12，这种不确定性不是我们想要的。所以需要引入方法保障线程执行的同步。

### synchronized关键字

被synchronized修饰的代码或者类就是线程同步访问的了，保证一个时间段最多只有一个线程访问那一段代码。

synchronized关键字可以用来修饰方法，当一个线程访问类的某个synchronized实例方法时，其他线程除了不可以访问当前synchronized方法外，也不能访问synchronized修饰的其他实例方法。原因是synchronized默认加的是对象锁。
synchronized关键字也可以用来修饰代码块，synchronized(this) { }
synchronized关键字中使用class级别的锁，synchronized(ClassName.class) { }，注意类锁和对象锁是不同的。


### 锁

synchronized是对象锁，其对性能影响较大。Java 1.5后新增了一个java.util.concurrent包来支持同步。 其中包含ReentrantLock和ReentrantReadWriteLock等。下面是一个使用ReentrantLock的来锁定账户金额的例子，一个线程执行1万次的存钱操作，而另一个线程执行1万次的取钱操作。最后希望得到的结果是不变的。

```
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.TimeUnit;
import java.util.concurrent.locks.ReentrantLock;

class Account {
    private String name;
    private int money;
    private ReentrantLock lock = new ReentrantLock();

    public Account(String name, int money) {
        this.name = name;
        this.money = money;
    }

    public void deposit(int amount) {
        try {
            lock.lock();
            this.money += amount;
        } finally {
            lock.unlock();
        }
    }

    public void draw(int amount) {
        try {
            lock.lock();
            this.money -= amount;
        } finally {
            lock.unlock();
        }
    }

    public int getMoney() {
        return this.money;
    }
}


public class ReentrantLockTest {

    public static void main(String[] args) {
        Account account = new Account("我的账户", 100);
        ExecutorService executorService = Executors.newFixedThreadPool(2);
        Runnable depositThread = new Runnable() {
            @Override
            public void run() {
                for(int i=0; i<10000; i++) {
                    account.deposit(10);
                }
            }
        };

        Runnable drawThread = new Runnable() {
            @Override
            public void run() {
                for(int i=0; i<10000; i++) {
                    account.draw(10);
                }
            }
        };

        executorService.execute(depositThread);
        executorService.execute(drawThread);

        executorService.shutdown(); //只是不能再提交新任务，等待执行的任务不受影响

        try
        {
            // awaitTermination返回false即超时会继续循环，返回true即线程池中的线程执行完成主线程跳出循环往下执行，每隔10秒循环一次
            while (!executorService.awaitTermination(10, TimeUnit.SECONDS));
        }
        catch (InterruptedException e)
        {
            e.printStackTrace();
        }

        System.out.println("final account money: " + account.getMoney());
    }
}

```

注意上面的代码中，lock.unlock是放到finally中。

ReentrantReadWriteLock类保留了两个锁，写锁和读锁。其是为了保障：
- 多个线程可以同时读数据，但是读数据的时候数据不能被更新(写);
- 最多一个线程可以在某一个时间修改数据，线程在写时其他写线程和多线程都需要被阻塞;
- 有线程在读时，写线程将会被阻塞。

```
ReadWriteLock rwLock = new ReentrantReadWriteLock();
Lock readLock = rwLock.readLock();
Lock writeLock = rwLock.writeLock();
```

### 使用Blocking Queue
LinkedBlockingQueue是一个单向链表实现的阻塞队列。该队列按 FIFO（先进先出）排序元素，新元素插入到队列的尾部，并且队列获取操作会获得位于队列头部的元素。

LinkedBlockingQueue对插入和取出都用了不同的锁（takeLock和putLock），对于插入操作，通过“插入锁putLock”进行同步；对于取出操作，通过“取出锁takeLock”进行同步。同时其提供了两种Condition(notEmpty和notFull)

以下是使用LinkedBlockingDeque来解决生产者和消费者多线程问题的代码，使用了put和take方法:

```
import java.util.UUID;
import java.util.concurrent.BlockingQueue;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.LinkedBlockingDeque;

class Task {
    private String name;

    public String getName() {
        return name;
    }

    public Task(String name) {
        this.name = UUID.randomUUID().toString() + " " +  name;
    }

    public void run() {
        System.out.println("In task: " + name);
    }
}

class Consumer implements  Runnable {
    private BlockingQueue<Task> blockingQueue;
    private String name;

    public Consumer(BlockingQueue<Task> blockingQueue, String name) {
        this.blockingQueue = blockingQueue;
        this.name = name;
    }

    @Override
    public void run() {
        while (true) {
            try {
                Task task = blockingQueue.take();
                System.out.println(name + " are running task " + task.getName());
                task.run();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }

    }
}

class Producer implements Runnable {
    private BlockingQueue<Task> blockingQueue;

    public Producer(BlockingQueue<Task> blockingQueue) {
        this.blockingQueue = blockingQueue;
    }

    @Override
    public void run() {
        while (true){
            try {
                Task task = new Task("product");
                blockingQueue.put(task);
                System.out.println("Producer are putting task " + task.getName());
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }
}


public class BlockingQueueTest {
    public static void main(String[] args) {

        BlockingQueue<Task> blockingQueue = new LinkedBlockingDeque<>(10);
        ExecutorService es = Executors.newFixedThreadPool(6);


        for(int i=0; i<3; i++) {
            es.execute(new Producer(blockingQueue));
        }

        for(int i=0; i<3; i++) {
            es.execute(new Consumer(blockingQueue, "Consumer" + i));
        }

    }
}
```

BlockingQueue<E>定义了阻塞队列的常用方法，尤其是三种添加元素的方法，我们要多加注意，当队列满时：
- add()方法会抛出异常
- offer()方法返回false
- put()方法会阻塞

### CAS

在java的util.concurrent.atomic包中提供了创建了原子类型变量的工具类，其底层都使用CAS的技术。

目前的处理器基本都支持CAS，只不过不同的厂家的实现不一样罢了。CAS有三个操作数：内存值V、旧的预期值A、要修改的值B，当且仅当预期值A和内存值V相同时，将内存值修改为B并返回true，否则什么都不做并返回false。

使用了atomic包中的类，可以让代码看起来更清爽了。

```
class Account {
    private AtomicInteger money;

    public AtomicInteger getMoney() {
        return money;
    }

    public void deposit(int amount) {
        money.addAndGet(amount);
    }
}
```

### 关于volatile
在并发编程中，我们通常会遇到以下三个问题：原子性问题，可见性问题，有序性问题。volatile能够保障可见性问题和有序性问题，但是对原子性问题却无能为力。

使用了volatile修饰的变量：

- 如果线程改变了其值，会强制将修改的值立即写入主存，而不是在CPU的各级缓存上溜达

- 如果A线程改变了其值，B线程的CPU缓存中的值就会立即失效，如果B线程想去读这个值，得重新从主存里读

使用volatile的场景有:

- 状态标记量

```
volatile boolean flag = false;

while(!flag){
    doSomething();
}

public void setFlag() {
    flag = true;
}
```

- double check

```
class Singleton{
    private volatile static Singleton instance = null;

    private Singleton() {

    }

    public static Singleton getInstance() {
        if(instance==null) {
            synchronized (Singleton.class) {
                if(instance==null)
                    instance = new Singleton();
            }
        }
        return instance;
    }
}
```

### ThreadLocal
ThreadLocal管理的变量在不同的线程中可以是不同的副本，可以是不同的值。其实现方式是定义了一个ThreadLocalMap，去存储不同的线程下这个变量不同的副本。因而使得不同的线程之间互相不影响。

```
public class ThreadLocalTest {
    public static void main(String[] args) {
        ThreadLocal<String> threadLocal = new ThreadLocal<>();
        threadLocal.set("helloworld!");

        Thread thread = new Thread(() -> {
            threadLocal.set("happy holiday!");
            System.out.println("Child thread: " + threadLocal.get());
        });

        thread.start();

        try {
            thread.join();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }

        System.out.println("Main thread: " + threadLocal.get());
    }
}
```

### CycleBarrier

CycleBarrier是为了让一组线程到达某一个屏障，可以使得所有的线程（根据数量）达到时才开始下一步。

CyclicBarrier c = new CyclicBarrier(2);

各个线程使用c.await()方法告诉CycleBarrier已达到屏障，如果有两个线程到达了这个屏障，各个线程才可以继续往下执行。

### Semaphore

Semaphore是用来保护一个或者多个共享资源的访问，Semaphore内部维护了一个计数器，其值为可以访问的共享资源的个数。一个线程要访问共享资源，先获得信号量，如果信号量的计数器值大于1，意味着有共享资源可以访问，则使其计数器值减去1，再访问共享资源。

```
Semaphore semaphore = new Semaphore(10,true);  
semaphore.acquire();
semaphore.release();  
```

## 线程池

### 创建线程池

Java通过Executors提供四种线程池：

- Executors.newCachedThreadPool()创建的是一个弹性线程池

- Executors.newCachedThreadPool(10) 创建了一个固定大小为10的线程池，定长线程池的大小最好根据系统资源进行设置。如Runtime.getRuntime().availableProcessors()

- Executors.newScheduledThreadPool(5) 创建了一个周期任务执行的线程池，大小为5，调用schedule方法来提交线程，设定执行时间等

- Executors.newSingleThreadExecutor()创建的是一个单线程的线程池

以上的几种创建方式都是基于ThreadPoolExecutor实现的。关于ThreadPoolExecutor的具体原理，可查看[深入理解java线程池—ThreadPoolExecutor](http://www.jianshu.com/p/ade771d2c9c0)


### shutdown与shutdownNow

shutdown()后线程池将变成shutdown状态，此时不接收新任务，但会处理完正在运行的 和 在阻塞队列中等待处理的任务。

shutdownNow()后线程池将变成stop状态，此时不接收新任务，不再处理在阻塞队列中等待的任务，还会尝试中断正在处理中的工作线程。

### execute与submit

execute与submit的区别是：

- submit返回Future，而execute不会，所以

- excecute只接受Runnable作为其参数


### 等待线程池的结束
- while(!executor.isTerminated())或者while (!executor.awaitTermination(10,TimeUnit.SECONDS))可以用来判断线程池有没有结束，

- 使用CountDownLatch，CountDownLatch是通过一个计数器来实现的，计数器的初始值为线程的数量。每当一个线程完成了自己的任务后，计数器的值就会减1。当计数器值到达0时，它表示所有的线程已经完成了任务，然后在闭锁上等待的线程就可以恢复执行任务。

```
import java.util.concurrent.CountDownLatch;

class Worker {
    private String name;
    private int operationTime;

    public Worker(String name, int operationTime) {
        this.name = name;
        this.operationTime = operationTime;
    }

    public void run() {
        System.out.println(name + " is running!");
        try {
            Thread.sleep(operationTime);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println(name + " finished!");
    }
}

class WorkerThread implements Runnable {
    private Worker worker;
    private CountDownLatch countDownLatch;

    public WorkerThread(Worker worker, CountDownLatch countDownLatch) {
        this.worker = worker;
        this.countDownLatch = countDownLatch;
    }

    @Override
    public void run() {
        worker.run();
        countDownLatch.countDown();
    }
}

public class CountDownWatchTest {

    public static void main(String[] args) {
        CountDownLatch countDownLatch = new CountDownLatch(2);
        Worker worker1 = new Worker("Thread 1", 1000);
        Worker worker2 = new Worker("Thread 2", 3000);

        new Thread(new WorkerThread(worker1, countDownLatch)).start();
        new Thread(new WorkerThread(worker2, countDownLatch)).start();

        try {
            countDownLatch.await();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println("All thread finished!");
    }

}
```

## 线程状态迁移

这里就不对线程状态迁移画图描述，但是需要了解线程具有的状态：

- 新建状态

- 就绪状态

- 运行状态

- 阻塞状态

- 死亡状态


## 线程间通信

### 共享内存

所谓的共享内存，其实就是多个线程持有一个相同的对象，因而可以调用其方法来改变对象中的成员变量的值。

### wait(), notify(), notifyAll(), join()

wait方法，和notify方法都是Object类就拥有的final方法，不可以被覆盖。我们可以利用wait()来让一个线程在某些条件下暂停运行。可以用 notify 和 notifyAll 来通知那些等待中的线程重新开始运行。不同之处在于，notify 仅仅通知一个线程，并且我们不知道哪个线程会收到通知，然而 notifyAll 会通知所有等待中的线程。

join()是Thread类的一个方法, t.join()方法阻塞调用此方法的线程，直到线程t完成，此线程再继续。如控制A, B, C线程的执行顺序，主线程等待子线程执行结束，都可以用join来实现。

### Condition

Condition 将 Object 监视器方法（wait、notify 和 notifyAll）分解成截然不同的对象，以便通过将这些对象与任意 Lock 实现组合使用。

在Condition中，用await()替换wait()，用signal()替换notify()，用signalAll()替换notifyAll()。

需要注意的是Condition是被绑定到Lock上的，要创建一个Lock的Condition必须用newCondition()方法。

```
final Lock lock = new ReentrantLock();  
final Condition notFull  = lock.newCondition();
final Condition notEmpty = lock.newCondition();
```

### 管道通信

在Java的JDK中提供了4个类来使线程间可以相互通信：
1）PipedInputSteam和PipedOutputStream
2）PipedReader和PipedWriter



## 其他

### Java的Collections类
synchronizedCollection
synchronizedList
synchronizedMap
synchronizedSet
synchronizedSortedMap
synchronizedSortedSet


List list = Collections.synchronizedList(new ArrayList());

### ConcurrentHashMap

ConcurrentHashMap是一个经常被使用的数据结构，相比于Hashtable以及Collections.synchronizedMap()，ConcurrentHashMap在线程安全的基础上提供了更好的写并发能力。起源中使用大量的volatile, final, CAS，值得一读。

## 参考文档

- [Java主线程等待子线程、线程池](http://blog.csdn.net/xiao__gui/article/details/9213413)

- [Java ReadWriteLock and ReentrantReadWriteLock Example](http://www.codejava.net/java-core/concurrency/java-readwritelock-and-reentrantreadwritelock-example)

- [java笔记--关于线程同步（7种同步方式）](http://www.cnblogs.com/XHJT/p/3897440.html)

- [Java LinkedBlockingQueue](http://www.jianshu.com/p/9394b257fdde)

- [什么时候使用CountDownLatch](http://www.importnew.com/15731.html)

- [java中的CAS](http://www.jianshu.com/p/fb6e91b013cc)

- [Java并发编程：volatile关键字解析](http://www.importnew.com/18126.html)

- [并发工具类（二）同步屏障CyclicBarrier](http://ifeve.com/concurrency-cyclicbarrier/)

- [Java中Semaphore(信号量)的使用](http://blog.csdn.net/zbc1090549839/article/details/53389602)

- [Java四种线程池的使用](http://cuisuqiang.iteye.com/blog/2019372)

- [深入理解java线程池—ThreadPoolExecutor](http://www.jianshu.com/p/ade771d2c9c0)

- [Java通过管道进行进程间通信](http://blog.csdn.net/x_i_y_u_e/article/details/51290061)

<script type="text/javascript">
$(document).ready(function() {
    $('#toc').toc({ listType: 'ul', title: "<i>目录</i>" });
});
</script>
