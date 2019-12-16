# JAVA

## 并发

### JUC
[juc学习路线](https://segmentfault.com/a/1190000015558984)
线程的状态流转图
![avatar](https://img-blog.csdn.net/20150722204022168)
#### juc-locks锁框架
早期的JDK版本中，仅仅提供了synchronizd、wait、notify等等比较底层的多线程同步工具，开发人员如果需要开发复杂的多线程应用，通常需要基于JDK提供的这些基础工具进行封装，开发自己的工具类。JDK1.5+后，Doug Lea根据一系列常见的多线程设计模式，设计了JUC并发包，其中java.util.concurrent.locks包下提供了一系列基础的锁工具，用以对synchronizd、wait、notify等进行补充、增强。

java.util.concurrent.locks包的结构如下：
![avatar](https://upload-images.jianshu.io/upload_images/10462182-b682c51008fdbb07.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/600)

包内接口和类的简单UML图如下：
![avatar](https://upload-images.jianshu.io/upload_images/10462182-579a7e4e808d1336.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/600)
#### (1)接口说明
##### 一、Lock接口简介
Lock接口可以视为synchronized的增强版，提供了更灵活的功能。该接口提供了限时锁等待、锁中断、锁尝试等功能。
**1.1 接口定义**
该接口的方法声明如下：
![avatar](https://upload-images.jianshu.io/upload_images/10462182-e3d9a7ae3580d149.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/700)
需要注意lock()和lockInterruptibly()这两个方法的区别：
- lock()方法类似于使用synchronized关键字加锁，如果锁不可用，出于线程调度目的，将禁用当前线程，并且在获得锁之前，该线程将一直处于休眠状态。
- lockInterruptibly()方法顾名思义，就是如果锁不可用，那么当前正在等待的线程是可以被中断的，这比synchronized关键字更加灵活。

**1.2 使用示例**
可以看到，Lock作为一种同步器，一般会用一个finally语句块确保锁最终会释放。
```Java
Lock lock = ...;
if (lock.tryLock()) {
    try {
        // manipulate protected state
    } finally {
        lock.unlock();
    }
} else {
    // perform alternative actions
}
```

##### 二、Condition接口简介
Condition可以看做是Obejct类的wait()、notify()、notifyAll()方法的替代品，与Lock配合使用。
当线程执行condition对象的await方法时，当前线程会立即释放锁，并进入对象的等待区，等待其它线程唤醒或中断。

*JUC在实现Conditon对象时，其实是通过实现AQS框架，来实现了一个Condition等待队列，这个在后面讲AQS框架时会详细介绍，目前只要了解Condition如何使用即可。*

**2.1 接口定义**
![avatar](https://upload-images.jianshu.io/upload_images/10462182-791238139928a012.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/700)

**2.2 使用示例**
假定有一个缓冲队列，支持 put 和 take 方法。如果试图在空队列中执行 take 操作，则线程将一直阻塞，直到队列中有可用元素；如果试图在满队列上执行 put 操作，则线程也将一直阻塞，直到队列不满。（这是一个类似于栈的数据结构）
```Java
class BoundedBuffer {
    final Lock lock = new ReentrantLock();
    final Condition notFull = lock.newCondition();
    final Condition notEmpty = lock.newCondition();

    final Object[] items = new Object[100];
    int putptr, takeptr, count;

    public void put(Object x) throws InterruptedException {
        lock.lock();
        try {
            while (count == items.length)    //防止虚假唤醒，Condition的await调用一般会放在一个循环判断中
                notFull.await();
            items[putptr] = x;
            if (++putptr == items.length)
                putptr = 0;
            ++count;
            notEmpty.signal();
        } finally {
            lock.unlock();
        }
    }

    public Object take() throws InterruptedException {
        lock.lock();
        try {
            while (count == 0)
                notEmpty.await();
            Object x = items[takeptr];
            if (++takeptr == items.length)
                takeptr = 0;
            --count;
            notFull.signal();
            return x;
        } finally {
            lock.unlock();
        }
    }
}
```
等待 Condition 时，为了防止发生“虚假唤醒”， Condition 一般都是在一个循环中被等待，并测试正被等待的状态声明，如上述代码注释部分。
虽然上面这个示例程序即使不用while，改用if判断也不会出现问题，但是最佳实践还是做while循环判断——[Guarded Suspension](https://segmentfault.com/a/1190000015558585)模式，以防遗漏情况。

##### 三、ReadWriteLock接口简介
ReadWriteLock接口是一个单独的接口（未继承Lock接口），该接口提供了获取读锁和写锁的方法。
*所谓读写锁，是一对相关的锁——读锁和写锁，读锁用于只读操作，写锁用于写入操作。读锁可以由多个线程同时保持，而写锁是独占的，只能由一个线程获取。*
**3.1 接口定义**
![avatar](https://upload-images.jianshu.io/upload_images/10462182-2d24c7aee140a3af.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/420)

**3.2 使用注意**
读写锁的阻塞情况如下图：
![avatar](https://upload-images.jianshu.io/upload_images/10462182-2ea0e462868f42e8.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
举个例子，假设我有一份共享数据——订单金额，大多数情况下，线程只会进行高频的数据访问（读取订单金额），数据修改（修改订单金额）的频率较低。
那么一般情况下，如果采用互斥锁，读/写和读/读都是互斥的，性能显然不如采用读写锁。
另外，由于读写锁本身的实现就远比独占锁复杂，因此，读写锁比较适用于以下情形：
- 高频次的读操作，相对较低频次的写操作；
- 读操作所用时间不会太短。（否则读写锁本身的复杂实现所带来的开销会成为主要消耗成本）

#### (2)ReentrantLock 的使用
##### 一、ReentrantLock类简介
ReentrantLock类，实现了Lock接口，是一种可重入的独占锁，它具有与使用 synchronized 相同的一些基本行为和语义，但功能更强大。ReentrantLock内部通过内部类实现了AQS框架(AbstractQueuedSynchronizer)的API来实现独占锁的功能。
**1.1 类声明**
![avatar](https://upload-images.jianshu.io/upload_images/10462182-3f4af1dafcda268a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/600)
**1.2 构造声明**
ReentrantLock类提供了两类构造器：
![avatar](https://upload-images.jianshu.io/upload_images/10462182-8cbf55700b4262cb.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/600)
ReentrantLock类的其中一个构造器提供了指定公平策略 / 非公平策略的功能，默认为非公平策略。
**注意：一般情况下，使用公平策略的程序在多线程访问时，总体吞吐量（即速度很慢，常常极其慢）比较低，因为此时在线程调度上面的开销比较大。**
**1.3 使用方式**
ReentrantLock的典型调用方式如下：
```Java
class X {
    private final ReentrantLock lock = new ReentrantLock();
    // ...
    public void m() {
        lock.lock(); // block until condition holds
        try {
            // ... method body
        } finally {
            lock.unlock();
        }
    }
}
```

##### 二、ReentrantLock类原理
ReentrantLock的源码非常简单，它通过内部类实现了AQS框架，Lock接口的实现仅仅是对AQS的api的简单封装.

#### (3)ReentrantReadWriteLock的使用
##### 一、ReentrantReadWriteLock类简介
ReentrantReadWriteLock类，顾名思义，是一种读写锁，它是ReadWriteLock接口的直接实现，该类在内部实现了具体独占锁特点的写锁，以及具有共享锁特点的读锁，和ReentrantLock一样，ReentrantReadWriteLock类也是通过定义内部类实现AQS框架的API来实现独占/共享的功能。

ReentrantReadWriteLock类具有如下特点：
**1.1 支持公平/非公平策略**
与ReadWriteLock类一样，ReentrantReadWriteLock对象在构造时，可以传入参数指定是公平锁还是非公平锁。
![avatar](https://upload-images.jianshu.io/upload_images/10462182-0ee6def2495c80f2.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/900)

**1.2 支持锁重入**
- 同一读线程在获取了读锁后还可以获取读锁;
- 同一写线程在获取了写锁之后既可以再次获取写锁又可以获取读锁；


**1.3 支持锁降级**
所谓锁降级，就是：先获取写锁，然后获取读锁，最后释放写锁，这样写锁就降级成了读锁。但是，读锁不能升级到写锁。简言之，就是：
**写锁可以降级成读锁，读锁不能升级成写锁。**

**问题**：ReentrantWriteLock 同个线程首先获取写锁后可以接着获取读锁（锁降级）以及写锁，但是获取读锁后就只能获取读锁，这个是出于什么考虑？
**答**：
这不就是读写锁的目的吗？写锁是排他锁，在一个线程持有写锁时，可以保证没有任何其他线程持有锁，相当于独占，所以它可以再持有读锁。但是反过来却不行，因为读锁并不是排他的，其他线程可能正在持有读锁或写锁，所以不能随意升级。

**1.4 Condition条件支持**
ReentrantReadWriteLock的内部读锁类、写锁类实现了Lock接口，所以可以通过newCondition()方法获取Condition对象。但是这里要注意，读锁是没法获取Condition对象的，读锁调用newCondition() 方法会直接抛出UnsupportedOperationException。

*我们知道，condition的作用其实是对Object类的wait()和notify()的增强，是为了让线程在指定对象上等待，是一种线程之间进行协调的工具。
当线程调用condition对象的await方法时，必须拿到和这个condition对象关联的锁。由于线程对读锁的访问是不受限制的（在写锁未被占用的情况下），那么即使拿到了和读锁关联的condition对象也是没有意义的，因为读线程之前不需要进行协调。*

**1.5 使用示例**
使用ReentrantReadWriteLock控制对TreeMap的访问（利用读锁控制读操作的访问，利用写锁控制修改操作的访问），将TreeMap包装成一个线程安全的集合，并且利用了读写锁的特性来提高并发访问。
```java
public class RWTreeMap {
    private final Map<String, Data> m = new TreeMap<String, Data>();
    private final ReentrantReadWriteLock rwl = new ReentrantReadWriteLock();
    private final Lock r = rwl.readLock();
    private final Lock w = rwl.writeLock();

    public Data get(String key) {
        r.lock();
        try {
            return m.get(key);
        } finally {
            r.unlock();
        }
    }

    public String[] allKeys() {
        r.lock();
        try {
            return (String[]) m.keySet().toArray();
        } finally {
            r.unlock();
        }
    }

    public Data put(String key, Data value) {
        w.lock();
        try {
            return m.put(key, value);
        } finally {
            w.unlock();
        }
    }

    public void clear() {
        w.lock();
        try {
            m.clear();
        } finally {
            w.unlock();
        }
    }
}
```
##### 二、ReentrantReadWriteLock类/方法声明
**2.1 类声明**
![avatar](https://upload-images.jianshu.io/upload_images/10462182-adf9297c699e65e3.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
内部嵌套类声明：
ReentrantReadWriteLock类有两个内部嵌套类ReadLock和WriteLock，这两个内部类的实例会在ReentrantReadWriteLock类的构造器中创建，并通过ReentrantReadWriteLock类的readLock()和writeLock()方法访问。
**ReadLock**：
![avatar](https://upload-images.jianshu.io/upload_images/10462182-59dc4c0458a11e27.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
**WriteLock**:
![avatar](https://upload-images.jianshu.io/upload_images/10462182-cc2716200ec9b40b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

**2.2 方法声明**
ReentrantReadWriteLock类的核心方法其实就两个：readLock()和writeLock()，其它都是一些用来监控系统状态的方法，返回的都是某一时刻点的近似值。
![avatar](https://upload-images.jianshu.io/upload_images/10462182-f0b0f0d0af464999.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

#### LockSupport 工具类
##### 一、LockSupport类简介
LockSupport类，是JUC包中的一个工具类，是用来创建锁和其他同步类的基本线程阻塞原语。

LockSupport类的核心方法其实就两个：park()和unark()，其中park()方法用来阻塞当前调用线程，unpark()方法用于唤醒指定线程。
这其实和Object类的wait()和signial()方法有些类似，但是LockSupport的这两种方法从语意上讲比Object类的方法更清晰，而且可以针对指定线程进行阻塞和唤醒。

**1.1 使用示例**
假设现在需要实现一种FIFO类型的独占锁，可以把这种锁看成是ReentrantLock的公平锁简单版本，且是不可重入的，就是说当一个线程获得锁后，其它等待线程以FIFO的调度方式等待获取锁。
```Java
public class FIFOMutex {
    private final AtomicBoolean locked = new AtomicBoolean(false);
    private final Queue<Thread> waiters = new ConcurrentLinkedQueue<Thread>();

    public void lock() {
        Thread current = Thread.currentThread();
        waiters.add(current);

        // 如果当前线程不在队首，或锁已被占用，则当前线程阻塞
        // NOTE：这个判断的意图其实就是：锁必须由队首元素拿到
        while (waiters.peek() != current || !locked.compareAndSet(false, true)) {
            LockSupport.park(this);
        }
        waiters.remove(); // 删除队首元素
    }

    public void unlock() {
        locked.set(false);
        LockSupport.unpark(waiters.peek());
    }
}
```
上述FIFOMutex 类的实现中，当判断锁已被占用时，会调用LockSupport.park(this)方法，将当前调用线程阻塞；当使用完锁时，会调用LockSupport.unpark(waiters.peek())方法将等待队列中的队首线程唤醒。

通过LockSupport的这两个方法，可以很方便的阻塞和唤醒线程。但是LockSupport的使用过程中还需要注意以下几点：
- park方法的调用一般要方法一个循环判断体里面。
如上述示例中的：
```Java
while (waiters.peek() != current || !locked.compareAndSet(false, true)) {
    LockSupport.park(this);
}
```
之所以这样做，是为了防止线程被唤醒后，不进行判断而意外继续向下执行，这其实是一种Guarded Suspension的多线程设计模式。

-  park方法是会响应中断的，但是不会抛出异常。(也就是说如果当前调用线程被中断，则会立即返回但不会抛出中断异常)

- park的重载方法park(Object blocker)，会传入一个blocker对象，所谓Blocker对象，其实就是当前线程调用时所在调用对象（如上述示例中的FIFOMutex对象）。该对象一般供监视、诊断工具确定线程受阻塞的原因时使用。

##### 二、LockSupport类/方法声明
**类声明**
![avatar](https://upload-images.jianshu.io/upload_images/10462182-ee8cdff500cf54aa.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/300)
**方法声明**
![avatar](https://upload-images.jianshu.io/upload_images/10462182-0ceb080d026b7486.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


### juc-atomic原子类框架

### juc-sync 同步器框架

### juc-collections 集合框架
这里的juc-collections集合框架，是指java.util.concurrent包下的一些同步集合类，按类型划分可以分为：符号表、队列、Set集合、列表四大类，每个类都有自己适合的使用场景，整个juc-collections集合框架的结构如下图：
![avatar](https://image-static.segmentfault.com/295/701/2957013004-5b7bef685a046_articlex)
其中阻塞队列的分类及特性如下表：
队列特征|有界队列|近似无界队列|无界队列|特殊队列
:-: | :-: | :-: | :-: | :-:
有锁算法 | ArrayBlockingQueue | LinkedBlockingQueue、LinkedBlockingDeque | / |  PriorityBlockingQueue、DelayQueue
无锁算法 | / | / | LinkedTransferQueue | SynchronousQueue

#### 9.阻塞队列简介——BlockingQueue
**简介**
BlockingQueue继承了Queue接口，提供了一些阻塞方法，主要作用如下：
- 当线程向队列中插入元素时，如果队列已满，则阻塞线程，直到队列有空闲位置（非满）；
- 当线程从队列中取元素（删除队列元素）时，如果队列未空，则阻塞线程，直到队列有元素；

操作类型 	| 抛出异常 |	返回特殊值 | 	阻塞线程 |	超时
:-: | :-: | :-: | :-: | :-:
插入 | add(e) |	offer(e)  | put(e) |	offer(e, time, unit)
删除 |remove() |	poll() |	take() |	poll(time, unit)
读取 |	element() |	peek() |	/  |	/
可以看到，对于每种基本方法，“抛出异常”和“返回特殊值”的方法定义和Queue是完全一样的。BlockingQueue只是增加了两类和阻塞相关的方法：put(e)、take()；offer(e, time, unit)、poll(time, unit)。

put(e)和take()方法会一直阻塞调用线程，直到线程被中断或队列状态可用；offer(e, time, unit)和poll(time, unit)方法会限时阻塞调用线程，直到超时或线程被中断或队列状态可用。

```Java
public interface BlockingQueue<E> extends Queue<E> {
    /**
     * 插入元素e至队尾, 如果队列已满, 则阻塞调用线程直到队列有空闲空间.
     */
    void put(E e) throws InterruptedException;

    /**
     * 插入元素e至队列, 如果队列已满, 则限时阻塞调用线程，直到队列有空闲空间或超时.
     */
    boolean offer(E e, long timeout, TimeUnit unit)
        throws InterruptedException;

    /**
     * 从队首删除元素，如果队列为空, 则阻塞调用线程直到队列中有元素.
     */
    E take() throws InterruptedException;

    /**
     * 从队首删除元素，如果队列为空, 则限时阻塞调用线程，直到队列中有元素或超时.
     */
    E poll(long timeout, TimeUnit unit) throws InterruptedException;

    // ...
}
```

#### ArrayBlockingQueue
**简介**
ArrayBlockingQueue是在JDK1.5时，随着J.U.C包引入的一种阻塞队列，它实现了BlockingQueue接口，底层基于数组实现：
![avatar](https://image-static.segmentfault.com/273/434/2734343052-5b9232618f01b_articlex)
ArrayBlockingQueue是一种有界阻塞队列，在初始构造的时候需要指定队列的容量。具有如下特点：
1.队列的容量一旦在构造时指定，后续不能改变;
2.插入元素时，在队尾进行；删除元素时，在队首进行；
3.队列满时，调用特定方法插入元素会阻塞线程；队列空时，删除元素也会阻塞线程；
4.支持公平/非公平策略，默认为非公平策略。

*这里的公平策略，是指当线程从阻塞到唤醒后，以最初请求的顺序（FIFO）来添加或删除元素；非公平策略指线程被唤醒后，谁先抢占到锁，谁就能往队列中添加/删除顺序，是随机的。*

**原理**




### juc-executors 执行器框架

countdownlatch
```Java
public class CountDownLatchTest {

    public static void main(String[] args) {
        final CountDownLatch latch = new CountDownLatch(2);
        System.out.println("主线程开始执行…… ……");
        //第一个子线程执行
        ExecutorService es1 = Executors.newSingleThreadExecutor();
        es1.execute(new Runnable() {
            @Override
            public void run() {
                try {
                    Thread.sleep(3000);
                    System.out.println("子线程："+Thread.currentThread().getName()+"执行");
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                latch.countDown();
            }
        });
        es1.shutdown();

        //第二个子线程执行
        ExecutorService es2 = Executors.newSingleThreadExecutor();
        es2.execute(new Runnable() {
            @Override
            public void run() {
                try {
                    Thread.sleep(3000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                System.out.println("子线程："+Thread.currentThread().getName()+"执行");
                latch.countDown();
            }
        });
        es2.shutdown();
        System.out.println("等待两个线程执行完毕…… ……");
        try {
            latch.await();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println("两个子线程都执行完毕，继续执行主线程");
    }
}

```
