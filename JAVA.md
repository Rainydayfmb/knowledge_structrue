# JAVA

## 并发

### 多线程
#### 一、本章概述
本章将继续以ReentrantLock的调用为例，说明AbstractQueuedSynchronizer提供的Conditon等待功能。
#### 二、Condition接口的实现

#### ABCABC 按照顺序执行
```Java
import java.util.concurrent.Semaphore;
public class ABC_Semaphore {
    // 以A开始的信号量,初始信号量数量为1
    private static Semaphore A = new Semaphore(1);
    // B、C信号量,A完成后开始,初始信号数量为0
    private static Semaphore B = new Semaphore(0);
    private static Semaphore C = new Semaphore(0);
    static class ThreadA extends Thread {
        @Override
        public void run() {
            try {
                for (int i = 0; i < 10; i++) {
                    A.acquire();// A获取信号执行,A信号量减1,当A为0时将无法继续获得该信号量
                    System.out.print("A");
                    B.release();// B释放信号，B信号量加1（初始为0），此时可以获取B信号量
                }
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }
    static class ThreadB extends Thread {
        @Override
        public void run() {
            try {
                for (int i = 0; i < 10; i++) {
                    B.acquire();
                    System.out.print("B");
                    C.release();
                }
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }
    static class ThreadC extends Thread {
        @Override
        public void run() {
            try {
                for (int i = 0; i < 10; i++) {
                    C.acquire();
                    System.out.println("C");
                    A.release();
                }
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }

    public static void main(String[] args) throws InterruptedException {
        new ThreadA().start();
        new ThreadB().start();
        new ThreadC().start();
    }
}

```

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



####  <span id = "lockSupport">(4)LockSupport 工具类</span>
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

#### (5)AbstractQueuedSynchronizer 综述

##### 一、AQS简介
AQS框架提供了一套通用的机制来管理同步状态（synchronization state）、阻塞/唤醒线程、管理等待队列。ReentrantLock、CountDownLatch、CyclicBarrier都是通过AQS实现的，这些同步器的主要区别其实就是对同步状态（synchronization state）的定义不同。

AQS框架，分离了构建同步器时的一系列关注点，它的所有操作都围绕着资源——同步状态（**synchronization state**）来展开，并替用户解决了如下问题：
- 资源是可以被同时访问？还是在同一时间只能被一个线程访问？（共享/独占功能）
- 访问资源的线程如何进行并发管理？（等待队列）
- 如果线程等不及资源了，如何从等待队列退出？（超时/中断）

AQS将剩下的问题留给了用户：**什么是资源，如何定义资源是否可以被访问**？
我们来看下几个常见的同步器对这一问题的定义：
同步器 | 资源的定义
 :-:  | :-:
 ReentrantLock | 资源表示独占锁。state表示锁可用；为1表示被占用；为N表示被重入的次数
 CountDownLatch | 资源表示倒数计数器。state 0 表示计数器归零，所有的线程可以访问资源；为N表示计数器为归零，所有的线程需要阻塞；
 Semaphore | 资源表示信号量或者令牌。State《=0表示没有令牌可用，搜有线程需要阻塞；大于0表示有令牌可用，线程每获取一个令牌，State减1，线程每释放一个令牌state加1
 ReentrantReadWriteLock | 资源表示共享的读锁和独占的写锁。state逻辑上被分为两个16位的unsigned short,分别记录读锁被多少线程使用和写锁被重入的次数

**1.1 提供一套模板框架**
AQS通过暴露以下API来让让用户自己解决上面提到的“如何定义资源是否可以被访问”的问题：
钩子方法 | 描述
:-:     | :-:
tryAcquire | 排它获取（资源数）
tryRelease | 排它释放（资源数）
tryAcquireShared | 共享获取（资源数）
tryReleaseShared | 共享获取（资源数）
isHeldExclusively | 是否排它状态

**1.2 支持中断、超时**
使用了AQS框架的同步器，都支持下面的操作：
- 阻塞和非阻塞（例如tryLock）同步；
- 可选的超时设置，让调用者可以放弃等待；
- 可中断的阻塞操作。

**1.3 支持独占模式和共享模式**

**1.4 支持Condition条件等待**
Condition接口，可以看做是Obejct类的wait()、notify()、notifyAll()方法的替代品，与Lock配合使用。
AQS框架内部通过一个内部类ConditionObject，实现了Condition接口，以此来为子类提供条件等待的功能。

##### 二、AQS方法说明
**2.1 CAS操作**
CAS，即CompareAndSet，在Java中CAS操作的实现都委托给一个名为UnSafe类，关于Unsafe类，以后会专门详细介绍该类，目前只要知道，通过该类可以实现对字段的原子操作。
方法名 | 修饰符 | 描述
:-: | :-: | :-:
compareAndSetState | protected final | CAS修改同步状态值
compareAndSetHead  | private final | CAS修改等待队列的头指针
compareAndSetTail  | private final | CAS修改等待队列的尾指针
compareAndSetWaitStatus | private static final | CAS修改结点的等待状态
compareAndSetNext | private static final | CAS修改结点的next指针

**2.2 等待队列的核心操作**
方法名 | 修饰符 | 描述
:-: | :-: | :-:
enq | private | 入队操作
addWaiter | private | 入队操作
setHead | private | 设置头结点
unparkSuccessor | private | 唤醒后继结点
doReleaseShared | private | 释放共享结点
setHeadAndPropagate | private | 设置头结点并传播唤醒

**2.3 资源的获取操作**
方法名 | 修饰符 | 描述
:-: | :-: | :-:
cancelAcquire | private | 取消获取资源
shouldParkAfterFailedAcquire | private static | 判断是否阻塞当前调用线程
acquireQueued | final | 尝试获取资源,获取失败尝试阻塞线程
doAcquireInterruptibly | private | 独占地获取资源（响应中断）
doAcquireNanos | private | 独占地获取资源（限时等待）
doAcquireShared | private | 共享地获取资源
doAcquireSharedInterruptibly | private | 共享地获取资源（响应中断）
doAcquireSharedNanos | private | 共享地获取资源（限时等待）


方法名 | 修饰符 | 描述
:-: | :-: | :-:
acquire |	public final |	独占地获取资源
acquireInterruptibly 	| public final 	| 独占地获取资源（响应中断）
acquireInterruptibly 	| public final 	| 独占地获取资源（限时等待）
acquireShared |	public final 	|共享地获取资源
acquireSharedInterruptibly 	|public final 	|共享地获取资源（响应中断）
tryAcquireSharedNanos |	public final 	| 共享地获取资源（限时等待）

**2.4 资源的释放操作**
方法名 | 修饰符 | 描述
:-: | :-: | :-:
release 	| public final |	释放独占资源
releaseShared |	public final |	释放共享资源

##### 三、AQS原理简述
AQS的所有操作都围绕着资源-**同步状态**（synchronization state）来展开因此，围绕着资源，衍生出三个基本问题：
- 同步状态（synchronization state）的管理
- 阻塞/唤醒线程的操作
- 线程等待队列的管理

**3.1 同步状态**
同步状态，其实就是资源。AQS使用单个int（32位）来保存同步状态，并暴露出getState、setState以及compareAndSetState操作来读取和更新这个状态。
```Java
/**
 * 同步状态.
 */
private volatile int state;

protected final int getState() {
    return state;
}

protected final void setState(int newState) {
    state = newState;
}
/**
 * 以原子的方式更新同步状态.
 * 利用Unsafe类实现
 */
protected final boolean compareAndSetState(int expect, int update) {
    return unsafe.compareAndSwapInt(this, stateOffset, expect, update);
}
```
**3.2 线程的阻塞/唤醒**
在JDK1.5之前，除了内置的监视器机制外，没有其它方法可以安全且便捷得阻塞和唤醒当前线程。
JDK1.5以后，java.util.concurrent.locks包提供了[LockSupport](#lockSupport)类来作为线程阻塞和唤醒的工具。

**3.3 等待队列**
等待队列，是AQS框架的核心，整个框架的关键其实就是如何在并发状态下管理被阻塞的线程。
//todo CLH锁

1.节点定义
CLH队列中的结点是对线程的包装，结点一共有两种类型：
- 独占（EXCLUSIVE）：CANCELLED(1)、SIGNAL(-1)、CONDITION(-2)
- 共享（SHARED）：CANCELLED(1)、SIGNAL(-1)、PROPAGATE(-3)。

结点状态 |	值 |	描述
:-: | :-: | :-:
CANCELLED |	1 |	取消。表示后驱结点被中断或超时，需要移出队列
SIGNAL    | -1 | 发信号。表示后驱结点被阻塞了（当前结点在入队后、阻塞前，应确保将其prev结点类型改为SIGNAL，以便prev结点取消或释放时将当前结点唤醒。）
CONDITION | -2 | Condition专用。表示当前结点在Condition队列中，因为等待某个条件而被阻塞了
PROPAGATE | -3 | 传播。适用于共享模式（比如连续的读操作结点可以依次进入临界区，设为PROPAGATE有助于实现这种迭代操作。）
INITIAL | 0 | 默认。新结点会处于这种状态

AQS使用CLH队列实现线程的结构管理，而CLH结构正是用前一结点某一属性表示当前结点的状态，之所以这种做是因为在双向链表的结构下，这样更容易实现取消和超时功能。

- next指针：用于维护队列顺序，当临界区的资源被释放时，头结点通过next指针找到队首结点。
- prev指针：用于在结点（线程）被取消时，让当前结点的前驱直接指向当前结点的后驱完成出队动作。

```Java
static final class Node {

    // 共享模式结点
    static final Node SHARED = new Node();

    // 独占模式结点
    static final Node EXCLUSIVE = null;

    static final int CANCELLED =  1;

    static final int SIGNAL    = -1;

    static final int CONDITION = -2;

    static final int PROPAGATE = -3;

    /**
    * INITAL：      0 - 默认，新结点会处于这种状态。
    * CANCELLED：   1 - 取消，表示后续结点被中断或超时，需要移出队列；
    * SIGNAL：      -1- 发信号，表示后续结点被阻塞了；（当前结点在入队后、阻塞前，应确保将其prev结点类型改为SIGNAL，以便prev结点取消或释放时将当前结点唤醒。）
    * CONDITION：   -2- Condition专用，表示当前结点在Condition队列中，因为等待某个条件而被阻塞了；
    * PROPAGATE：   -3- 传播，适用于共享模式。（比如连续的读操作结点可以依次进入临界区，设为PROPAGATE有助于实现这种迭代操作。）
    *
    * waitStatus表示的是后续结点状态，这是因为AQS中使用CLH队列实现线程的结构管理，而CLH结构正是用前一结点某一属性表示当前结点的状态，这样更容易实现取消和超时功能。
    */
    volatile int waitStatus;

    // 前驱指针
    volatile Node prev;

    // 后驱指针
    volatile Node next;

    // 结点所包装的线程
    volatile Thread thread;

    // Condition队列使用，存储condition队列中的后继节点
    Node nextWaiter;

    Node() {
    }

    Node(Thread thread, Node mode) {
        this.nextWaiter = mode;
        this.thread = thread;
    }
}
```

2.队列定义
对于CLH队列，当线程请求资源时，如果请求不到，会将线程包装成结点，将其挂载在队列尾部。
CLH队列的示意图如下：
①初始状态，队列head和tail都指向空
![avatar](https://image-static.segmentfault.com/740/737/740737339-5b5d4f6e5066a_articlex)

②首个线程入队，先创建一个空的头结点，然后以自旋的方式不断尝试插入一个包含当前线程的新结点
![avatar](https://image-static.segmentfault.com/410/300/4103009523-5b5d4f91e85ff_articlex)
![avatar](https://image-static.segmentfault.com/311/891/3118914716-5b5d4f9f14dd5_articlex)

```Java
/**
 * 以自旋的方式不断尝试插入结点至队列尾部
 *
 * @return 当前结点的前驱结点
 */
private Node enq(final Node node) {
    for (; ; ) {
        Node t = tail;
        if (t == null) { // 如果队列为空，则创建一个空的head结点
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

##### 四、总结
本章简要介绍了AQS的思想和原理，读者可以参考Doug Lea的论文，进一步了解AQS。
直接阅读AQS的源码比较漫无目的，后续章节，将从ReentrantLock、CountDownLatch的使用入手，讲解AQS的独占功能和共享功能。

#### (7)Condition 原理

### juc-atomic原子类框架
早期的JDK版本中，如果要并发的对Integer、Long、Double之类的Java原始类型或引用类型进行操作，一般都需要通过锁来控制并发，以防数据不一致。

从JDK1.5开始，引入了java.util.concurrent.atomic工具包，该包提供了许多Java原始/引用类型的映射类，如AtomicInteger、AtomicLong、AtomicBoolean，这些类可以通过一种“无锁算法”，线程安全的操作Integer、Long、Boolean等原始类型。

**关于CAS**
在计算机科学中，比较和交换（Conmpare And Swap）是用于实现多线程同步的原子指令。 它将内存位置的内容与给定值进行比较，只有在相同的情况下，将该内存位置的内容修改为新的给定值。 这是作为单个原子操作完成的。 原子性保证新值基于最新信息计算; 如果该值在同一时间被另一个线程更新，则写入将失败。 操作结果必须说明是否进行替换; 这可以通过一个简单的布尔响应（这个变体通常称为比较和设置），或通过返回从内存位置读取的值来完成（摘自维基本科）

**CAS在JUC中的运用**
我们看一下JUC中非常重要的一个类AbstractQueuedSynchronizer，作为JAVA中多种锁实现的父类，其中有很多地方使用到了CAS操作以提升并发的效率
![avatar](https://mmbiz.qpic.cn/mmbiz_png/wXExy1PE3KMlm3PC5npFw0XOibb4Zq4Ao7N8P7eibCORiawibZsJraj4BfSOncSkljpAvalTiadlsCiczs9qjrRHiaZtw/640?wx_fmt=png)

**ABA问题**
所以JAVA中提供了AtomicStampedReference/AtomicMarkableReference来处理会发生ABA问题的场景，主要是在对象中额外再增加一个标记来标识对象是否有过变更。

### juc-sync 同步器框架

#### (3)信号量——Semaphore

##### 一、Semaphore简介
信号量，作用类似于许可证。有时，我们需要控制同时访问共享资源的最大线程数量，比如处于系统的性能考虑需要限流，或者共享资源是稀缺资源，我们需要一种方法能够协调各个线程，保证合理地使用公共资源。
semaphore维护了一个许可集：
1.当有线程需要访问资源的时候，通过先获取(acquire)；如果许可不够用了，线程会一直等待，直达徐克可用；
2.当线程使用完资源后，可以归还(release)许可，以供其他线程的使用。

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

### 框架

#### spring
##### spring的ioc和aop
- ioc：依赖注入的思想是通过反射机制实现的，在实例化一个类时，它通过反射调用类中set方法将事先保存在HashMap中的类属性注入到类中。
- aop:实现AOP的技术，主要分为两大类：一是采用动态代理技术，利用截取消息的方式，对该消息进行装饰，以取代原有对象行为的执行；二是采用静态织入的方式，引入特定的语法创建“方面”，从而使得编译器可以在编译期间织入有关“方面”的代码。

**Spring实现AOP**：JDK动态代理和CGLIB代理 JDK动态代理：其代理对象必须是某个接口的实现，它是通过在运行期间创建一个接口的实现类来完成对目标对象的代理；其核心的两个类是InvocationHandler和Proxy。 CGLIB代理：实现原理类似于JDK动态代理，只是它在运行期间生成的代理对象是针对目标类扩展的子类。CGLIB是高效的代码生成包，底层是依靠ASM（开源的java字节码编辑类库）操作字节码实现的，性能比JDK强；需要引入包asm.jar和cglib.jar。使用AspectJ注入式切面和@AspectJ注解驱动的切面实际上底层也是通过动态代理实现的。

**动态代理和静态代理**：
**动态代理优点**：
动态代理与静态代理相比较，最大的好处是接口中声明的所有方法都被转移到调用处理器一个集中的方法中处理（InvocationHandler.invoke）。这样，在接口方法数量比较多的时候，我们可以进行灵活处理，而不需要像静态代理那样每一个方法进行中转。而且动态代理的应用使我们的类职责更加单一，复用性更强




#### springmvc


##### springmvc的请求流程
[参考](https://www.jianshu.com/p/8a20c547e245)
![avatar](https://upload-images.jianshu.io/upload_images/5220087-3c0f59d3c39a12dd.png?imageMogr2/auto-orient/strip|imageView2/2/w/1002/format/webp)
1.用户发送请求至前端控制器DispatcherServlet
2.DispatcherServlet收到请求调用处理器映射器HandlerMapping。
3.处理器映射器根据请求url找到具体的处理器，生成处理器执行链HandlerExecutionChain(包括处理器对象和处理器拦截器)一并返回给DispatcherServlet。
4.DispatcherServlet根据处理器Handler获取处理器适配器HandlerAdapter执行HandlerAdapter处理一系列的操作，如：参数封装，数据格式转换，数据验证等操作
5.执行处理器Handler(Controller，也叫页面控制器)。
6.Handler执行完成返回ModelAndView
7.HandlerAdapter将Handler执行结果ModelAndView返回到DispatcherServlet
8.DispatcherServlet将ModelAndView传给ViewReslover视图解析器
9.ViewReslover解析后返回具体View
10.DispatcherServlet对View进行渲染视图（即将模型数据model填充至视图中）。
11.DispatcherServlet响应用户。

##### springmvc的拦截器和过滤器的区别
[参考](https://blog.csdn.net/zxd1435513775/article/details/80556034)
**过滤器**
filter的使用位置,我们在配置web.xml时会配置字符编码，让所有的请求都需要进行字符编码的设置。
过滤器依赖于servlet容器。在实现上，基于回调函数。它可以对几乎所有请求进行过滤，但是缺点是一个过滤器实例只能在容器初始化时调用一次。使用过滤器的目的，是用来做一些过滤操作，获取我们想要获取的数据，比如：在Javaweb中，对传入的request、response提前过滤掉一些信息，或者提前设置一些参数，然后再传入servlet或者Controller进行业务逻辑操作。通常用的场景是：在过滤器中修改字符编码（CharacterEncodingFilter）、在过滤器中修改HttpServletRequest的一些参数（XSSFilter(自定义过滤器)），如：过滤低俗文字、危险字符等。

**拦截器**
拦截器的配置一般在SpringMVC的配置文件中，使用Interceptors标签。

它依赖于web框架，在SpringMVC中就是依赖于SpringMVC框架。在实现上,基于Java的反射机制，属于面向切面编程（AOP）的一种运用，就是在service或者一个方法前，调用一个方法，或者在方法后，调用一个方法，比如动态代理就是拦截器的简单实现，在调用方法前打印出字符串（或者做其它业务逻辑的操作），也可以在调用方法后打印出字符串，甚至在抛出异常的时候做业务逻辑的操作。**由于拦截器是基于web框架的调用，因此可以使用Spring的依赖注入（DI）进行一些业务操作，同时一个拦截器实例在一个controller生命周期之内可以多次调用。但是缺点是只能对controller请求进行拦截，对其他的一些比如直接访问静态资源的请求则没办法进行拦截处理**。

**总结**
对于上述过滤器和拦截器的测试，可以得到如下结论：
（1）、Filter需要在web.xml中配置，依赖于Servlet；
（2）、Interceptor需要在SpringMVC中配置，依赖于框架；
（3）、Filter的执行顺序在Interceptor之前，具体的流程见下图；
![avatar](https://img-blog.csdn.net/20180603133007923?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3p4ZDE0MzU1MTM3NzU=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)
（4）、两者的本质区别：拦截器（Interceptor）是基于Java的反射机制，而过滤器（Filter）是基于函数回调。从灵活性上说拦截器功能更强大些，Filter能做的事情，都能做，而且可以在请求前，请求后执行，比较灵活。Filter主要是针对URL地址做一个编码的事情、过滤掉没用的参数、安全校验（比较泛的，比如登录不登录之类），太细的话，还是建议用interceptor。不过还是根据不同情况选择合适的。

## JVM

### java运行时数据区域
[参考](https://www.cnblogs.com/czwbig/p/11127124.html)
下图是 JDK8 之后的 JVM 内存布局。
![avatar](https://upload-images.jianshu.io/upload_images/14923529-c0cbbccaa6858ca1.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
jdk8之前的内存区域图：
![avatar](https://upload-images.jianshu.io/upload_images/14923529-ae8d342e3d4432ad.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
#### 程序计数器
是一块较小的内存空间，它可以看作是当前线程所执行的字节码的行号指示器。
由于 Java 虚拟机的多线程是通过线程轮流切换并分配处理器执行时间的方式来实现的，在任何一个确定的时刻，一个处理器内核都只会执行一条线程中的指令。
因此，为了线程切换后能恢复到正确的执行位置，每条线程都需要有一个独立的程序计数器，各条线程之间计数器互不影响，独立存储，我们称这类内存区域为“**线程私有**”的内存。
如果线程正在执行的是一个 Java 方法，这个计数器记录的是正在执行的虚拟机字节码指令的地址；如果正在执行的是 Native 方法，这个计数器值则为空（Undefined）。此内存区域是唯一一个在 Java 虚拟机规范中没有规定任何 OutOfMemoryError 情况的区域。
#### Java虚拟机栈
与程序计数器一样，Java 虚拟机栈（Java Virtual Machine Stacks）也是**线程私有**的，它的生命周期与线程相同。
**虚拟机栈描述的是 Java 方法执行的内存模型**：每个方法在执行的同时都会创建一个栈帧（Stack Frame，是方法运行时的基础数据结构）用于存储局部变量表、操作数栈、动态链接、方法出口等信息。每一个方法从调用直至执行完成的过程，就对应着一个栈帧在虚拟机栈中入栈到出栈的过程。
在活动线程中，只有位千栈顶的帧才是有效的，称为当前栈帧。正在执行的方法称为当前方法，栈帧是方法运行的基本结构。在执行引擎运行时，所有指令都只能针对当前栈帧进行操作。
![avatar](https://upload-images.jianshu.io/upload_images/14923529-7a6d6e02c15fff2f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
##### 1. 局部变量表
局部变量表是**存放方法参数和局部变量**的区域。 局部变量没有准备阶段， 必须显式初始化。如果是非静态方法，则在 index[0] 位置上存储的是方法所属对象的实例引用，一个引用变量占 4 个字节，随后存储的是参数和局部变量。字节码指令中的 STORE 指令就是将操作栈中计算完成的局部变呈写回局部变量表的存储空间内。

虚拟机栈规定了两种异常状况：如果线程请求的栈深度大于虚拟机所允许的深度，将抛出 StackOverflowError 异常；如果虚拟机栈可以动态扩展（当前大部分的 Java 虚拟机都可动态扩展），如果扩展时无法申请到足够的内存，就会抛出 OutOfMemoryError 异常。

##### 2. 操作栈
操作栈是个初始状态为空的桶式结构栈。在方法执行过程中， 会有各种指令往栈中写入和提取信息。JVM 的执行引擎是基于栈的执行引擎， 其中的栈指的就是操作栈。字节码指令集的定义都是基于栈类型的，栈的深度在方法元信息的 stack 属性中。

##### 3.动态连接
每个栈帧中包含一个在常量池中对当前方法的引用， 目的是支持方法调用过程的动态连接。

##### 4.方法返回地址
方法执行时有两种退出情况：
- 正常退出，即正常执行到任何方法的返回字节码指令，如 RETURN、IRETURN、ARETURN 等；
- 异常退出。

无论何种退出情况，都将返回至方法当前被调用的位置。方法退出的过程相当于弹出当前栈帧，退出可能有三种方式：
- 返回值压入上层调用栈帧。
- 异常信息抛给能够处理的栈帧。
- PC计数器指向方法调用后的下一条指令。

#### 本地方法栈
本地方法栈（Native Method Stack）与虚拟机栈所发挥的作用是非常相似的，它们之间的区别不过是虚拟机栈为虚拟机执行 Java 方法（也就是字节码）服务，而本地方法栈则为虚拟机使用到的 Native 方法服务。Sun HotSpot 虚拟机直接就把本地方法栈和虚拟机栈合二为一。与虚拟机栈一样，本地方法栈区域也会抛出 StackOverflowError 和 OutOfMemoryError 异常。

#### Java堆
对于大多数应用来说，Java 堆（Java Heap）是 Java 虚拟机所管理的内存中最大的一块。Java 堆是被所有线程共享的一块内存区域，在虚拟机启动时创建。此内存区域的唯一目的就是存放对象实例，几乎所有的对象实例都在这里分配内存。

堆是垃圾收集器管理的主要区域，因此很多时候也被称做“GC堆”（Garbage Collected Heap）。从内存回收的角度来看，由于现在收集器基本都采用分代收集算法，所以 Java 堆中还可以细分为：新生代和老年代；再细致一点的有 Eden 空间、From Survivor 空间、To Survivor 空间等。从内存分配的角度来看，线程共享的 Java 堆中可能划分出多个线程私有的分配缓冲区（Thread Local Allocation Buffer,TLAB）。

Java 堆可以处于物理上不连续的内存空间中，只要逻辑上是连续的即可，当前主流的虚拟机都是按照可扩展来实现的（通过 -Xmx 和 -Xms 控制）。如果在堆中没有内存完成实例分配，并且堆也无法再扩展时，将会抛出 OutOfMemoryError 异常。

#### 方法区
方法区（Method Area）与 Java 堆一样，是各个线程共享的内存区域，它用于存储已被虚拟机加载的类信息、常量、静态变量、即时编译器编译后的代码等数据。虽然 Java 虚拟机规范把方法区描述为堆的一个逻辑部分，但是它却有一个别名叫做 Non-Heap（非堆），目的应该是与 Java 堆区分开来。

Java 虚拟机规范对方法区的限制非常宽松，除了和 Java 堆一样不需要连续的内存和可以选择固定大小或者可扩展外，还可以选择不实现垃圾收集。垃圾收集行为在这个区域是比较少出现的，其内存回收目标主要是针对常量池的回收和对类型的卸载。当方法区无法满足内存分配需求时，将抛出 OutOfMemoryError 异常。

**JDK8 之前，Hotspot 中方法区的实现是永久代（Perm），JDK8 开始使用元空间（Metaspace），以前永久代所有内容的字符串常量移至堆内存，其他内容移至元空间，元空间直接在本地内存分配。**

为什么要使用元空间取代永久代的实现？
- 字符串存在永久代中，容易出现性能问题和内存溢出。
- 类及方法的信息等比较难确定其大小，因此对于永久代的大小指定比较困难，太小容易出现永久代溢出，太大则容易导致老年代溢出。
- 永久代会为 GC 带来不必要的复杂度，并且回收效率偏低。
- 将 HotSpot 与 JRockit 合二为一。

##### 运行时常量池
运行时常量池（Runtime Constant Pool）是方法区的一部分。Class 文件中除了有类的版本、字段、方法、接口等描述信息外，还有一项信息是常量池（Constant Pool Table），用于存放编译期生成的各种字面量和符号引用，这部分内容将在类加载后进入方法区的运行时常量池中存放。

一般来说，除了保存 Class 文件中描述的符号引用外，还会把翻译出来的直接引用也存储在运行时常量池中。

运行时常量池相对于 Class 文件常量池的另外一个重要特征是具备动态性，Java 语言并不要求常量一定只有编译期才能产生，也就是并非预置入 Class 文件中常量池的内容才能进入方法区运行时常量池，运行期间也可能将新的常量放入池中，这种特性被开发人员利用得比较多的便是 String 类的 intern() 方法。

既然运行时常量池是方法区的一部分，自然受到方法区内存的限制，当常量池无法再申请到内存时会抛出 OutOfMemoryError 异常。

##### 直接内存
直接内存（Direct Memory）并不是虚拟机运行时数据区的一部分，也不是 Java 虚拟机规范中定义的内存区域。

在 JDK 1.4 中新加入了 NIO，引入了一种基于通道（Channel）与缓冲区（Buffer）的 I/O 方式，它可以使用 Native 函数库直接分配堆外内存，然后通过一个存储在 Java 堆中的 DirectByteBuffer 对象作为这块内存的引用进行操作。这样能在一些场景中显著提高性能，因为避免了在 Java 堆和 Native 堆中来回复制数据。

显然，本机直接内存的分配不会受到 Java 堆大小的限制，但是，既然是内存，肯定还是会受到本机总内存（包括 RAM 以及 SWAP 区或者分页文件）大小以及处理器寻址空间的限制。服务器管理员在配置虚拟机参数时，会根据实际内存设置 -Xmx 等参数信息，但经常忽略直接内存，使得各个内存区域总和大于物理内存限制（包括物理的和操作系统级的限制），从而导致动态扩展时出现 OutOfMemoryError 异常。

![avatar](https://upload-images.jianshu.io/upload_images/14923529-3d9e650ad915bc0e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 垃圾回收都有哪些方法
[参考一](https://juejin.im/post/5b85ea54e51d4538dd08f601)
[参考二](https://www.cnblogs.com/aspirant/p/8662690.html)
#### 哪些内存需要回收
- 程序计数器、虚拟机栈、本地方法栈3个区域随线程而生、随线程而灭，不需要回收
- java堆和方法区这部分内存的分配和回收是动态的，正式垃圾收集器所需要关注的部分。

#### 内存回收的策略
**JDK1.8之前的堆内存示意图：**
![avatar](https://user-gold-cdn.xitu.io/2018/8/25/16570344a29c3433?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

从上图可以看出堆内存的分为新生代、老年代和永久代。新生代又被进一步分为：Eden 区＋Survior1 区＋Survior2 区。值得注意的是，在 JDK 1.8中移除整个永久代，取而代之的是一个叫元空间（Metaspace）的区域（永久代使用的是JVM的堆内存空间，而元空间使用的是物理内存，直接受到本机的物理内存限制）。
![avatar](https://user-gold-cdn.xitu.io/2018/8/29/165831b636b1f7ec?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)
**1.1 对象优先在eden区分配**
目前主流的垃圾收集器都会采用分代回收算法，因此需要将堆内存分为新生代和老年代，这样我们就可以根据各个年代的特点选择合适的垃圾收集算法。

大多数情况下，对象在新生代中 eden 区分配。当 eden 区没有足够空间进行分配时，虚拟机将发起一次Minor GC.下面我们来进行实际测试以下。

**在测试之前我们先来看看 Minor Gc和Full GC 有什么不同呢？**
- 新生代GC（Minor GC）:指发生新生代的的垃圾收集动作，Minor GC非常频繁，回收速度一般也比较快。一般情况下，当新对象生成，并且在Eden申请空间失败时，就会触发Scavenge GC，对Eden区域进行GC，清除非存活对象，并且把尚且存活的对象移动到Survivor区。

- 老年代GC（Major GC/Full GC）:指发生在老年代的GC，出现了Major GC经常会伴随至少一次的Minor GC（并非绝对），Major GC的速度一般会比Minor GC的慢10倍以上。

对整个堆进行整理，包括Young、Tenured和Perm。Full GC因为需要对整个堆进行回收，所以比Scavenge GC要慢，因此应该尽可能减少Full GC的次数。在对JVM调优的过程中，很大一部分工作就是对于Full GC的调节。有如下原因可能导致Full GC：
a) 年老代（Tenured）被写满；
b) 持久代（Perm）被写满；
c) System.gc()被显示调用；
d) 上一次GC之后Heap的各域分配策略动态变化；

#### 垃圾回收算法
![avatar](https://user-gold-cdn.xitu.io/2018/8/29/165831b674db685f?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

##### 标记-清除算法

算法分为“标记”和“清除”阶段：首先标记出所有需要回收的对象，在标记完成后统一回收所有被标记的对象。它是最基础的收集算法，效率也很高，但是会带来两个明显的问题：
1.效率问题
2.空间问题（标记清除后会产生大量不连续的碎片）
![avatar](https://user-gold-cdn.xitu.io/2018/8/29/165831b67608523b?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

##### 复制算法

为了解决效率问题，“复制”收集算法出现了。它可以将内存分为大小相同的两块，每次使用其中的一块。当这一块的内存使用完后，就将还存活的对象复制到另一块去，然后再把使用的空间一次清理掉。这样就使每次的内存回收都是对内存区间的一半进行回收。

##### 标记整理算法
根据老年代的特点特出的一种标记算法，标记过程仍然与“标记-清除”算法一样，但后续步骤不是直接对可回收对象回收，而是让所有存活的对象向一段移动，然后直接清理掉端边界以外的内存。
![avatar](https://user-gold-cdn.xitu.io/2018/8/29/165831b69685fdbc?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

##### 分代收集算法
当前虚拟机的垃圾手机都采用分代收集算法，这种算法没有什么新的思想，只是根据对象存活周期的不同将内存分为几块。一般将java堆分为新生代和老年代，这样我们就可以根据各个年代的特点选择合适的垃圾收集算法。

比如在新生代中，每次收集都会有大量对象死去，所以可以选择复制算法，只需要付出少量对象的复制成本就可以完成每次垃圾收集。而老年代的对象存活几率是比较高的，而且没有额外的空间对它进行分配担保，所以我们必须选择“标记-清楚”或“标记-整理”算法进行垃圾收集。

#### 垃圾收集器
![avatar](https://user-gold-cdn.xitu.io/2018/8/29/165831b6960d1419?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)
如果说收集算法是内存回收的方法论，那么垃圾收集器就是内存回收的具体实现。

##### Serial收集器
Serial（串行）收集器收集器是最基本、历史最悠久的垃圾收集器了。大家看名字就知道这个收集器是一个单线程收集器了。它的 “单线程” 的意义不仅仅意味着它只会使用一条垃圾收集线程去完成垃圾收集工作，更重要的是它在进行垃圾收集工作的时候必须暂停其他所有的工作线程（ "Stop The World" ），直到它收集结束。
![avatar](https://user-gold-cdn.xitu.io/2018/8/29/165831b697e79b83?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

缺点：但线程会阻塞其他工作线程，而且STD的时长较长。
优点：当然有，它简单而高效（与其他收集器的单线程相比）。Serial收集器由于没有线程交互的开销，自然可以获得很高的单线程收集效率。Serial收集器对于运行在Client模式下的虚拟机来说是个不错的选择。


##### ParNew收集器

ParNew收集器其实就是Serial收集器的多线程版本，除了使用多线程进行垃圾收集外，其余行为（控制参数、收集算法、回收策略等等）和Serial收集器完全一样。
![avatar](https://user-gold-cdn.xitu.io/2018/8/29/165831b699d5cd86?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

**并行和并发概念补充：**
**并行（Parallel）** ：指多条垃圾收集线程并行工作，但此时用户线程仍然处于等待状态。
**并发（Concurrent）**：指用户线程与垃圾收集线程同时执行（但不一定是并行，可能会交替执行），用户程序在继续运行，而垃圾收集器运行在另一个CPU上。

##### Parallel Scavenge收集器
Parallel Scavenge 收集器类似于ParNew 收集器。 那么它有什么特别之处呢？
-XX:+UseParallelGC
    使用Parallel收集器+ 老年代串行
-XX:+UseParallelOldGC
    使用Parallel收集器+ 老年代并行

##### CMS收集器
CMS（Concurrent Mark Sweep）收集器是一种以获取最短回收停顿时间为目标的收集器。它而非常符合在注重用户体验的应用上使用。

CMS（Concurrent Mark Sweep）收集器是HotSpot虚拟机第一款真正意义上的并发收集器，它第一次实现了让垃圾收集线程与用户线程（基本上）同时工作。
![avatar](https://user-gold-cdn.xitu.io/2018/8/29/165831b69cca82d9?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)






















### 虚拟机内存和栈外内存
