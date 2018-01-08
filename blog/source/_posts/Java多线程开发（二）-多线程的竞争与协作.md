---
title: Java多线程开发（二）| 多线程的竞争与协作
date: 2018-01-08 18:26:33
tags:
- Java
---
# 0. 前言
　　使用多线程的过程中，主要要解决的是两类问题：
1. 多个线程共享资源
2. 多个线程的协作

　　线程就像独立的个体，每个线程都有各自的任务。为了完成各自的任务，会去获取自己需要的资源，可能会和其他线程产生竞争。但每个线程的任务，最终都是为了实现共同的目标，线程与线程之间需要相互配合。而我们要做的，就是建立一种机制，让多个线程能合理地竞争，有效地合作。这么一想，管理多线程就像管理团队一样。团队的任务拆解到个人，每个人有各自的任务和目标，在执行过程中会用到相同的资源，成员之间也需要沟通和协作。管理团队不容易，管理多线程也要小心谨慎。
　　对 Java 的多线程机制不了解的同学，可以先看我的上一篇文章：[Java多线程开发（一）| 基本的线程机制](http://www.jianshu.com/p/d094f6adb78e)。

# 1. 多个线程共享资源

## 1.1 不正确地访问资源
　　多个线程经常有需要共享的资源。有些是因为资源本身有限，比如打印机；有一些是出于协作的需要，比如共享变量。当两个以上的线程同时访问相同的资源时，很容易出现问题。
　　举个例子演示一下。想了很久找不出很好的例子，就用《Thinking in Java》书上的例子吧。

```java
public abstract class IntGenerator {
    private volatile boolean mCanceled = false;
    public abstract int next();
  
    public void cancel() {
        mCanceled = true;
    }
  
    public boolean isCanceled() {
        return mCanceled;
    }
}

public class EvenChecker implements Runnable {
    private IntGenerator mGenerator;
    private final int mId;

    public EvenChecker(IntGenerator generator, int ident) {
        mGenerator = generator;
        mId = ident;
    }
  
    public void run() {
        while (!mGenerator.isCanceled()) {
            int val = mGenerator.next();
            if (val % 2 != 0) {
                System.out.println(val + " not even");
                mGenerator.cancel();
            }
        }
    }
  
    public static void test(IntGenerator generator, int count) {
        System.out.println("Press Control-C to exit");
        ExecutorService executor = Executors.newCachedThreadPool();
        for (int i = 0; i < count; ++i) {
            executor.execute(new EvenChecker(generator, i));
        }
        executor.shutdown();
    }
  
    public static void test(IntGenerator generator) {
        test(generator, 10);
    }
}
```

　　IntGenerator 是产生数字的抽象类，EvenChecker 创建多个线程去调用 IntGenerator 的 next() 方法生成数字，并检测数字是否是偶数。

```java
public class EvenGenerator extends IntGenerator {
    private int mCurrentEvenValue = 0;
  
    @Override
    public int next() {
        ++mCurrentEvenValue;
        Thread.yield();
        ++mCurrentEvenValue;
        return mCurrentEvenValue;
    }
  
    public static void main(String[] args) {
        EvenChecker.test(new EvenGenerator());
    }
}
// 输出：
11 not even
15 not even
17 not even
19 not even
21 not even
```

　　EvenGenerator 用于生成偶数。在两个自增操作之间，增加了一个 Thread.yield() ，是为了更快地观察到现象。EvenGenerator 的 next() 被多个线程调用，mCurrentEvenValue 在这里就是多个线程共享的变量。从输出结果可以看到，由于线程在第一次自增操作之后切换，next() 返回的值会出现奇数。程序处于不正常的状态。

## 1.2 解决共享资源竞争问题

　　为了解决线程竞争共享资源导致的问题，通常让线程以序列化的方式访问资源，即同一时刻只能有一个线程访问资源。当一个线程在访问资源时，就不允许另一个线程访问，线程间的这种制约关系叫互斥。而同一时间只能被一个线程访问的资源叫临界资源，访问临界资源的代码块叫临界区。一般通过对临界区加锁，实现线程的互斥。

　　在 Java 中，对临界区加锁常用的方式有两种：
- synchronized 关键字
- Lock 对象

###  1.2.1 synchronized 关键字

　　Java 中的 synchronized 关键字，为防止资源冲突提供了内置支持。当执行到 synchronized 关键字保护的代码块时，首先要获取锁，然后才能执行代码，执行完成后释放锁。synchronized 关键字的使用有如下几种形式：
- 修饰方法，synchronized void fun() {...}
- 修饰静态方法，synchronized static fun() {...}
- 包裹代码块，synchronized (obj) {...}

　　虽然 synchronized 关键字的使用有不同的形式，但本质上是一样的，都是对对象加锁。在 Java 中，所有对象都含有一个锁（源码注释中叫 monitor），synchronized 就是获取对象的锁。再回过头看一下 synchronized 的几种使用形式：

- 修饰方法，是对调用该方法的对象加锁
- 修饰 static 方法，是对 Class 对象加锁
- 修饰语句块，是对指定对象加锁

　　一个线程可以多次获得同一个对象的锁，比如在 synchronized 方法中调用另一个 synchronized 方法。JVM 会记录对象加锁的次数，已经获得锁的线程再次获得锁，计数加1。计数为0时，锁才被完全释放，其他线程才能获得这个锁。
　　使用 synchronized 关键字修改一下前面的例子，把 next() 方法用 synchronized 保护起来。这次不管运行多久，都不会出现 next() 返回奇数的情况了。

```java
public class SynchronizedEvenGenerator extends IntGenerator {
    private int mCurrentEvenValue = 0;

    @Override
    public synchronized int next() {
        ++mCurrentEvenValue;
        Thread.yield();
        ++mCurrentEvenValue;
        return mCurrentEvenValue;
    }

    public static void main(String[] args) {
        EvenChecker.test(new SynchronizedEvenGenerator());
    }
}
```

### 1.2.2 Lock 对象

　　除了内置的 synchronized 关键字外。还可以使用 java.util.concurrent.locks 类库中 Lock 的对象来实现互斥。Lock 是一个接口。需要显示地创建 Lock 对象，然后调用 lock() 方法去获取锁，调用 unlock() 方法去释放锁。

```java
public interface Lock {
    void lock();
    void lockInterruptibly() throws InterruptedException;
    boolean tryLock();
    boolean tryLock(long time, TimeUnit unit) throws InterruptedException;
    void unlock();
    Condition newCondition();
}
```

　　下面是使用 Lock 重写的 EventGenerator ，简单展示一下 Lock 的使用方法。注意，使用 Lock 时，尽量用 try-finally , 把 unlock() 方法的调用放到 finally 中。避免中途出现异常，导致锁无法被释放。

```java
public class MutexEvenGenerator extends IntGenerator {

    private int mCurrentEvenValue = 0;
    private Lock mLock = new ReentrantLock();

    @Override
    public int next() {
        try {
            mLock.lock();
            ++mCurrentEvenValue;
            Thread.yield();
            ++mCurrentEvenValue;
            return mCurrentEvenValue;
        } finally {
            // try-finally 的惯用法，保证锁能释放
            mLock.unlock();
        }
    }

    public static void main(String[] args) {
        EvenChecker.test(new MutexEvenGenerator());
    }
}
```

　　和 synchronized 相比，Lock 的使用更麻烦，但同时也更灵活。因为锁的创建、加锁和释放都由我们控制，可以实现 synchronized 做不到的需求。下面列举一些 Lock 能做到，而 synchronized 无法实现的功能：

- 尝试获取锁，但如果锁已经被占用会直接返回而不阻塞；也可以设置尝试获取锁的等待时间，超时之后就返回。使用 Lock 的 tryLock() 方法即可。
- 等待锁导致阻塞时，能够被 interrupt() 方法打断。
- 更细粒度的控制，可以实现代码块之间交叉的加锁。比如，遍历链表时，要在释放当前节点的锁之前获取下一个节点的锁。

　　关于 synchronized 关键字和 Lock 对象的选择，尽量用 synchronized 。当 synchronized 满足不了需求的时候，才考虑用 Lock 。synchronized 使用更简单，而且相对来说更加安全。
　　java.util.concurrent.lock 包中，有三种锁。
- Lock ，普通锁接口，有一个实现：ReentrantLock。
- ReadWriteLock，包含读锁和写锁两个锁，这里的锁是 Lock 对象。也有一个实现：ReentrantReadWriteLock。
- StampedLock，包含乐观读锁、悲观读锁、写锁三种模式锁，是一个具体的实现类。1.8才引入的，与 Lock 和 ReadWriteLock 完全无关。

　　对于 ReentrantReadWriteLock，有一点需要注意。

>Thread1: A.readlock().lock() -> …  已经拿到读锁
>
>Thread2: A.writelock().lock() ->.. .请求写锁
>
>Thread3: A.readlock().lock() ->… 等待，一直到 Thread2 获取到然后又释放写锁。

　　第一个线程获取了读锁，第二个线程在等待获取写锁。这时其他线程想要获取读锁的话，得等待第二个线程获取到写锁，做完事情，释放写锁。

## 1.3 原子性、可见性

**原子性：**一个操作是不可中断的，开始执行就一定会完全执行完。这种操作被称为原子操作。
　　除了 long 和 dobule 之外的基本类型变量的读取和写入操作，都是原子性的。JVM 可以将 long 和 dobue 的读取和写入当成两个 32 位操作来执行，在中间可以发生切换。JVM 规范里面说明对 long 和 dobule 的读写操作不要求原子性，但是加了 volatile 关键字的 long 和 dobule 变量，必须是原子性的。
　　java.util.concurrent.atomic 类库提供了一系列原子操作的类，比如 AtomicInteger、AtomicLong、AtomicReference 等。它们能提供自增、赋值并读取等扩展的原子操作。

**可见性：**也可以叫作可视性、一致性。
　　在多处理器或者多核处理器系统中，一个任务修改了某个共享变量，但这个修改暂时只存在处理器的缓存中，还没有更新到主存中。对于运行在其他处理器上的任务来说，这个变量的修改时不可见的，它们看到的还是修改前的值。在 Java 的内存模型中，每个线程都有单独的工作内存，线程内变量的修改会先缓存在工作内存中。
![线程、主存、工作内存之间的交互关系](http://upload-images.jianshu.io/upload_images/196189-41cd2e6531de15dd.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240 "线程、主存、工作内存之间的交互关系")
　　volatile 关键字可以保证可见性，使变量的修改立即写入到主存中，读取时也从主存中读取变量的最新值。synchronized 可以保证加锁的方法或者代码块中的修改的可见性。
　　volatile 同时还能提供一定的有序性。对 volatile 变量的读写在一些情况下不会被重排序。具体可以参考这篇文章：[深入理解Java内存模型（四）——volatile](http://www.infoq.com/cn/articles/java-memory-model-4)。[Java内存访问重排序的研究](http://tech.meituan.com/java-memory-reordering.html)。

## 1.４ 线程本地存储 

　　有一些变量不希望被其他线程共享，那可以使用 ThreadLocal。顾名思义，通过 ThreadLocal ，每个线程可以存储只有自身能访问的变量。ThreadLocal 对象本身只有一个，但是它里面存的值是每个线程一个，完全独立的。Thread中持有 ThreadLocalMap的对象threadLocals，这个map完全由ThreadLocal创建和维护。ThreadLocal对象，获取到当前线程的ThreadLocalMap，往里面存值，所以每个线程的值都是独立的。ThreadLocalMap的key是ThreadLocal对象，value就是ThreadLocal要保存的值。
　　来看下面的例子。

```java
private static ThreadLocal<Integer> sVal = ThreadLocal.withInitial(() -> (1));

public static void main(String[] args) {
    ExecutorService executorService = Executors.newCachedThreadPool();
    for (int i = 0; i < 10; i++) {
        executorService.execute(() -> {
            sVal.set(sVal.get() + 1);
            System.out.println(Thread.currentThread().getName() + " " + sVal.get());
        });
    }
    executorService.shutdown();
}
// 输出：
pool-1-thread-1 2
pool-1-thread-2 2
pool-1-thread-3 2
pool-1-thread-2 3
pool-1-thread-1 3
pool-1-thread-3 3
pool-1-thread-2 4
pool-1-thread-2 5
pool-1-thread-2 6
pool-1-thread-4 2
```

　　因为 ThreadLocal 的对象 sVal 中存储的值应该是每个线程独立的，理论上输出都是 2 才对，但是最终的输出结果甚至有 6。这是因为，我们使用了线程池，而这里的任务执行时间太短，线程被复用了。这意味着线程中保存的值还是之前那个，所以会一直累加上来了。所以在使用线程池时，如果任务依赖 ThreadLocal 储存的值，那就需要小心了。

# 2 线程之间的协作

　　线程之间共享资源的问题解决了，接下来学习如何让线程之间彼此协作了。线程之间的协作，说白了，就是某些部分任务需要等待其他部分任务完成后，才能继续进行。我们可以通过 Object 的 wait() 和 notify() 方法来实现，也可以通过 Condition 对象的 await() 和 signal() 方法来实现。

## 2.1 wait()、notifyAll() 和 notify()

　　某些任务在执行时，会依赖某个条件，只有条件满足了才能继续执行，而这个条件是有其他任务改变的。如果我们只是不断地循环去检测这个条件，将会导致线程无意义地占用 CPU 资源，这被称为忙等待。而 wait() 可以在等待条件变化时，将任务挂起，等到 notify() 或者 notifyAll() 方法被调用时才会唤醒去检测条件是否满足。
　　ｗait() 、notify() 和 notifyAll() 都是 Object 类的方法。基于对象锁（monitor）实现。所以，调用这几个方法前，必须要先获得相应对象的锁，不然会抛出 IllegalMonitorStateExecption 异常。获取对象锁的方式就是使用 synchronized 关键字。
　　调用 wait() 方法挂起任务时，线程会释放获取到的对象锁，以让其他线程能够获得这个对象的锁。流程是这样的：在调用 wait() 之前，已经获得了该对象的锁；调用 wait() 的时候，会释放对象锁；而在从 wait() 唤醒之前，线程会重新获得对象锁。
　　下面是一个给汽车打蜡的例子，两个线程，一个负责打蜡，一个负责抛光。抛光任务要等待打蜡任务完成，而下一层打蜡任务要等待上一层蜡被抛光。使用 wait() 和 notify() 来进行任务同步。

```java
public class WaxOMatic {
    public static void main(String[] args) {
        ExecutorService executorService = Executors.newCachedThreadPool();
        Car car = new Car();
        executorService.execute(new WaxOn(car));
        executorService.execute(new WaxOff(car));
        try {
            Thread.sleep(5000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        executorService.shutdownNow();
    }
}

class Car {
    private boolean waxOn = false;
    synchronized void waxed() {
        waxOn = true;
        notifyAll();
    }
    synchronized void buffed() {
        waxOn = false;
        notifyAll();
    }
    synchronized void waitForWaxing() throws InterruptedException {
        while (!waxOn) {
            wait();
        }
    }
    synchronized void waitForBuffing() throws InterruptedException {
        while (waxOn) {
            wait();
        }
    }
}

class WaxOn implements Runnable {
    private Car car;
    WaxOn(Car car) {
        this.car = car;
    }
    @Override
    public void run() {
        try {
            while (!Thread.interrupted()) {
                System.out.println("Wax On! " + System.currentTimeMillis());
                Thread.sleep(200);
                car.waxed();
                car.waitForBuffing();
            }
        } catch (InterruptedException e) {
            System.out.println("Exiting via interrupt");
        }
        System.out.println("Ending Wax On task");
    }
}

class WaxOff implements Runnable {
    private Car car;
    WaxOff(Car car) {
        this.car = car;
    }
    @Override
    public void run() {
        try {
            while (!Thread.interrupted()) {
                car.waitForWaxing();
                System.out.println("Wax Off! " + System.currentTimeMillis());
                Thread.sleep(200);
                car.buffed();
            }
        } catch (InterruptedException e) {
            System.out.println("Exiting via interrupt");
        }
        System.out.println("Ending Wax On task");
    }
}
```

　　一般使用 wait() 方法，应该用一个 while 循环去检查条件。因为无法保证唤醒的时候，条件是否满足，在每次唤醒后去检查才能保证安全。有很多可能性导致唤醒的时候条件并不满足：

1. 有多个任务因为相同的原因在等待。第一个被唤醒的任务先进行了处理，条件又变成不满足了。

2. 被唤醒时，其他任务做了一些操作，影响到了当前条件。
3. 多个任务因为不同的原因在等待。因为要唤醒其他任务而被连带唤醒了。

   ```java
   synchronized (monitor)
   while (someCondition) {
       monitor.wait();
   }
   ```
### 错失的信号

　　当两个线程使用 wait() / notify() / notifyAll() 进行协作时，有可能出现错过某个信号的情况。

```java
T1:
synchronized(sharedMonitor) {
    <setup condition for T2>
    someCondition = false;
    sharedMonitor.notify();
}

T2:
while (someCondition) {
    // Point1. 在这个时间点被切换
    synchronized(sharedMonitor) {
        sharedMonitor.wait();  
    }
}
```

　　T2 线程先执行，判断条件 someCondition 为 true。在 Point1 这个时间点被切换了。T1 执行，someCondition 变为 false，并且调用 notify() 。T2 继续执行，这时已经晚了，T2 调用 wait() 进行等待。而 notify() 已经错过了，T2 将一直在 wait() ，等不到唤醒的信号。这个也算是共享资源的竞争导致的问题。为了避免这种情况，我们应该把对条件的判断和 wait() 放在同一个 synchronized 代码块中。

```java
T2:
synchronized(sharedMonitor) {
    while (someCondition) {
        sharedMonitor.wait();  
    }
}
```

### notify() 还是 notifyAll()

　　notify() 和 notifyAll() 。如果有多个线程在同一个对象上等待，notify() 会选择一个唤醒。因此，使用 notify() 的时候得确保唤醒的是你想要的线程。不然的话，还是只能用 notifyAll() 。

### 2.2 使用 Lock 和 Condition 对象

　　跟互斥一样，除了 Java 内建的 wait() 和 notify() 上，java.util.concurrent 类库中还提供了显示的工具来进行线程的同步。就是 Condition，也是一个接口。使用 Condition 对象，可以调用 await() 来挂起一个任务，通过调用 signal() 或者 signalAll() 来唤醒任务。
　　Condition 是一个接口，但是 Condition 类的文档注释中对 Condition 的实现做了严格的要求。Condition 对象应该关联到 Lock， 应该通过 Lock 对象的 newCondition() 方法生成。Lock 和 Condition 的使用，基本上就和 synchronized 和 wait()/notify() 的用法差不多。调用 await()、singnal() 和 singnalAll() 等方法前，应该先对关联的 Lock 对象上锁，不然要抛出 IllegalMonitorStateException 异常。调用 await() 挂起任务时，会释放关联的锁，被唤醒前，重新获得锁。
　　使用 Lock 和 Condition 好处是：Lock 和 Condition 都是自己创建的，使用上更灵活。比如，每个等待的条件可以专门弄一个 Condition 对象，这样可以更精确地控制唤醒的线程，优化程序的执行效率。
　　下面是用 Lock 和 Condition 重写的给汽车打蜡的例子。

```java
public class WaxOMaticWithCondition {
    public static void main(String[] args) {
        ExecutorService executorService = Executors.newCachedThreadPool();
        Car car = new Car();
        executorService.execute(new WaxOn(car));
        executorService.execute(new WaxOff(car));
        try {
            Thread.sleep(5000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        executorService.shutdownNow();
    }
    static class Car {
        private boolean waxOn = false;
        private Lock lock = new ReentrantLock();
        private Condition condition = lock.newCondition();

        void waxed() {
            lock.lock();
            try {
                waxOn = true;
                condition.signalAll();
            } finally {
                // 养成好习惯，把 unlock 放到 finally 中，避免异常导致锁无法释放
                lock.unlock();
            }
        }
        void buffed() {
            lock.lock();
            try {
                waxOn = false;
                condition.signalAll();
            } finally {
                lock.unlock();
            }
        }
        void waitForWaxing() throws InterruptedException {
            lock.lock();
            try {
                while (!waxOn) {
                    condition.await();
                }
            } finally {
                lock.unlock();
            }
        }
        void waitForBuffing() throws InterruptedException {
            lock.lock();
            try {
                while (waxOn) {
                    condition.await();
                }
            } finally {
                lock.unlock();
            }
        }
    }
    static class WaxOn implements Runnable {
        private Car car;
        WaxOn(Car car) {
            this.car = car;
        }
        @Override
        public void run() {
            try {
                while (!Thread.interrupted()) {
                    System.out.println("Wax On! " + System.currentTimeMillis());
                    Thread.sleep(200);
                    car.waxed();
                    car.waitForBuffing();
                }
            } catch (InterruptedException e) {
                System.out.println("Exiting via interrupt");
            }
            System.out.println("Ending Wax On task");
        }
    }
    static class WaxOff implements Runnable {
        private Car car;
        WaxOff(Car car) {
            this.car = car;
        }
        @Override
        public void run() {
            try {
                while (!Thread.interrupted()) {
                    car.waitForWaxing();
                    System.out.println("Wax Off! " + System.currentTimeMillis());
                    Thread.sleep(200);
                    car.buffed();
                }
            } catch (InterruptedException e) {
                System.out.println("Exiting via interrupt");
            }
            System.out.println("Ending Wax Off task");
        }
    }
}
```



### 2.3 阻塞队列

　　阻塞队列（BlockingQueue），解决“生产者-消费者”问题的神器，因为它内部为我们实现了线程之间的同步，可以极大地简化代码。当某个线程试图从阻塞队列中取一个对象是，如果队列为空，线程会被挂起等待。往阻塞队列添加对象也是一样，当队列已满时，线程会被挂起等待。

主要关注以下几个方法：

添加元素的方法：

- put(E)。队列已满时，挂起线程，一直到队列中有空位。
- offer(E): boolean。队列已满时，返回 false，表示添加失败。
- offer(E, long, TimeUnit): boolean。队列已满时，等待一段时间。超时后，返回 false，表示添加失败。
- add(E): boolean。添加成功，返回 true 。队列已满时，抛出异常。AbstractQueue 的实现是直接调用 offer(E) ，失败就抛出异常。

取出元素的方法：

- take(): E。队列为空时，挂起线程，一直到有新的元素插入。
- pull(long, TimeUnit): 队列为空时，挂起线程，等待一段时间。超时后，返回 null 。

　　线程池（ThreadPoolExecutor ）就是依赖于阻塞队列实现的，任务被添加到阻塞队列中，线程从阻塞队列中获取任务。

# 3. 结束线程

　　前面提到的 synchronized 和 Lock 等，都会导致线程阻塞。那么，怎么结束阻塞状态的线程呢？先来看看 Java 中的线程有哪几种状态。

1. 新建（new）。线程刚被创建时，短暂地处于这种状态
2. 就绪状态（RUNNABLE）。线程正在 JVM 中执行。但是线程可能正在等待操作系统的处理器资源。
3. 阻塞状态（BLOCKED、WAITING、TIMED_WAITING）。根据阻塞的原因区分了三种，这里我们详细区分。
4. 终止状态（Terminated）。

　　为什么没有RUNNING状态，因为线程获得处理器资源去实际执行，这是操作系统负责的，JVM 不管这事儿。对JVM 来说，它只知道线程当前是就绪状态，至于是在等待处理器还是实际在执行并不关心。具体可以看这篇文章。[Java 线程状态之 RUNNABLE](https://my.oschina.net/goldenshaw/blog/705397)。

进入阻塞状态的原因：

1. sleep
2. wait
3. 等待输入输出
4. synchronized 或者 Lock

　　对阻塞状态的线程调用 interrupt() 方法，将抛出 InterruptedException。可以用来中断线程。如果不想或者不方便直接对线程对象操作。也可以通过 executor 的 submit 方法执行 Runable，拿到 Future 对象，调用 Future 对象的 cancel 方法。cancel 方法实际上也是调用 interrupt。
　　sleep wait 等阻塞可以被 interrupt() 中断，但 io 和 synchronized阻塞 不能被中断。Lock 有 lockInterruptibly() 方法，可以被中断。对于像 io 引起的阻塞，可以通过关闭底层资源来结束线程。线程池的 shutdownNow() 方法也是调用 interrupt() 方法的，也无法结束被 io 和 synchronized 阻塞的线程。

# 4. 死锁

　　对于临界资源，我们使用互斥锁来保证同一时刻只能有一个线程在访问。但是，使用互斥锁也容易出现死锁的问题。死锁，指的是两个或两个以上的线程（进程），每个线程持有部分资源，而又等待其他线程的资源，形成环路地等待，导致所有线程都永远无法拿到所有所需的资源。

产生死锁的四个必要条件：

1. 互斥条件。
2. 不可抢占条件。
3. 占有且申请条件。
4. 环路等待条件。

预防死锁的办法。破坏一个条件即可。

1. 打破互斥条件。让资源允许共享。
2. 打破不可抢占条件。线程请求不到资源时，释放已获取的资源。
3. 打破占有且申请条件。线程一次性申请所有资源。
4. 打破环路等待条件。把资源排序，所有线程按顺序请求资源，不会出现环路等待的情况。

# 5. 结语 
　　多线程开发主要要解决的问题是竞争与协作。对于竞争，有 synchronized 关键字和 Lock 对象两种方式来实现互斥。对于协作，也有 synchronized+wait()/notify() 和 Lock + Condition 两种方式实现任务的等待。至此，我们已经对 Java 多线程开发有了较为全面地理解。下一篇文章将研究一下 Java 并发包下给我们提供的一些好用的工具。
