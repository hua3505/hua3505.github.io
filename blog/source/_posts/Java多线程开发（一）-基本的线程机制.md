---
title: Java多线程开发（一）| 基本的线程机制
date: 2018-01-08 18:23:05
tags:
- Java
---
### 0. 前言

　　Java 为了实现跨平台，在语言层面上实现了多线程。我们只需要熟悉 Java 这一套多线程机制就行了，比 C/C++ 要容易多了。

### 1. 定义任务

　　我们编写程序，最终是为了完成特定的任务。为了更有效的利用系统资源，我们把任务合理地划分成多个子任务，放到多个线程中来执行。所以，首先我们需要一种描述任务的方式。在 Java 中，一般我们都用 Runable 接口来定义任务。

```java
public interface Runnable {
    // 在run方法中定义任务
    public void run();
}
```

　　想要定义任务，只需要实现 Runable 接口，然后在 run() 方法中写上执行步骤。请注意，Runable 只是定义了一个任务，本身不会去启动一个新线程来执行。看下面的例子。可以看到，在外面直接打印的线程名和在 Runable 的 run() 方法中打印的线程名是相同的。

```java
public static void main(String[] args) {
    System.out.println(Thread.currentThread().getName());
    Runnable runnable = new Runnable() {
        @Override
        public void run() {
            System.out.println(Thread.currentThread().getName());
        }
    };
    runnable.run();
}
```

> 输出结果：
>
> main
> main
<!-- more -->
### 2. Thread类 

​	要让任务在新的线程执行，最直接的方法是用它来创建一个 Thread 类。这里用 Thinking in Java 书上的例子来展示 Thread 类的使用。

```java
/**
 * 显示发射之前的倒计时
 */
public class LiftOff implements Runnable {
    private static int sTaskCount = 0;
    private final int mId = sTaskCount++;
    protected int mCountDown = 10;

    public LiftOff() {
    }

    public LiftOff(int countDown) {
        mCountDown = countDown;
    }

    private String status() {
        return "#" + mId + "(" +
                ((mCountDown > 0) ? mCountDown : "Liftoff!") + "), ";
    }

    @Override
    public void run() {
        while (mCountDown-- > 0) {
            System.out.print(status());
            Thread.yield(); // Thread.yield() 是对线程调度器的一种建议，表示当前线程准备让出处理器
        }
    }
}
```
　　LiftOff 任务会显示发射前的倒计时。注意在 run() 方法中调用的 Thread.yield() 方法。这个方法的作用是对线程调度器的一种建议，表示当前线程可以让出处理器。当然，线程调度器不一定会真的切换执行线程。LifiOff 任务整个执行时间实际上很短，如果不使用 Thread.yield()  很可能直到任务执行完成线程调度器才会切换新的线程，不利于观察多线程的效果。

```java
public class MoreBasicThreads {
    public static void main(String[] args) {
        for (int i = 0; i < 5; i++) {
            new Thread(new LiftOff()).start();
        }
        System.out.println("Waiting for Liftoff");
    }
}

输出结果：
#0(9), #1(9), #2(9), #3(9), #0(8), Waiting for Liftoff
#4(9), #1(8), #2(8), #3(8), #0(7), #4(8), #1(7), #2(7), #3(7), #0(6), #4(7), #1(6), #2(6), #3(6), #0(5), #4(6), #1(5), #2(5), #3(5), #0(4), #4(5), #1(4), #2(4), #3(4), #0(3), #4(4), #1(3), #2(3), #3(3), #0(2), #4(3), #1(2), #2(2), #3(2), #0(1), #4(2), #1(1), #2(1), #3(1), #0(Liftoff!), #4(1), #1(Liftoff!), #2(Liftoff!), #3(Liftoff!), #4(Liftoff!), 
```

​	Thread的构造器接收 Runable 对象，并在调用 start() 方法之后启动新的线程去执行 Runable中的 run() 方法。输出结果很有意思。我们启动了5个发射前的倒计时任务，“ Waiting for Liftoff ” 在倒计时没完成之前就输出了，这证明现在的任务确实是在新的线程执行的。各个任务的倒计时混杂在一起，说明不同任务的执行线程在被不断的换进换出。

### 3. 使用Executor

　　java.util.concurrent 包中的执行器（ Executor ），可以帮我们管理Thread对象。 Executor 是一个接口，只有一个方法，就是 execute 。当我们把一个 Runable 交给 Executor 去执行，它可能会启动一个新的线程、或者从线程池中选择一个线程、甚至直接使用当前线程。但是，这些我们都不需要关心，我们只需要选择合适的 Executor 的实现，然后把任务扔给它去执行就好了。

```java
public interface Executor {

    /**
     * Executes the given command at some time in the future.  The command
     * may execute in a new thread, in a pooled thread, or in the calling
     * thread, at the discretion of the {@code Executor} implementation.
     *
     * @param command the runnable task
     * @throws RejectedExecutionException if this task cannot be
     * accepted for execution
     * @throws NullPointerException if command is null
     */
    void execute(Runnable command);
}
```

　　先来看一个具体的使用示例。在这个示例中，我们通过 Executors 来创建了一个 线程池 CachedThreadPool。并通过这个线程池来执行5个发射前的倒计时任务。

```java
public class CachedThreadPool {
    public static void main(String[] args) {
        ExecutorService exectorService = Executors.newCachedThreadPool();
        for (int i=0; i<5;++i) {
            exectorService.execute(new LiftOff());
        }
        exectorService.shutdown();
    }
}
```

　　在上面的示例中有几个需要解释的概念：

- Executors：一个工厂和工具类。
- ExecutorService：有生命周期的 Executor 。也是一个接口，继承于 Executor 。
- CachedThreadPool：线程池。当新任务过来，会首先找池中有没有可用的线程，没有才新建线程。

　　在 Executors 中还定义了另外三种线程池：FixedThreadPool 、SingleThreadPool 、 ScheduledThreadPool  （也提供了单线程的 ScheduledThreadPool ）。FixedThreadPool 线程数量是稳定的，线程创建后不会销毁，达到设定的数量后，不再创建新线程。SingleThreadExecutor 是只能有一个线程的线程池。而 ScheduledThreadPool  可以定时执行任务。现在把上面的示例中的 CachedThreadPool 换成 FixedThreadPool ，最大线程数量为3。

```java
public class FixedThreadPool {
    public static void main(String[] args) {
        ExecutorService exector = Executors.newFixedThreadPool(3);
        for (int i=0; i<5;++i) {
            exector.execute(new LiftOff());
        }
        exector.shutdown();
    }
}

输出结果：
#1(9), #2(9), #0(9), #1(8), #2(8), #0(8), #1(7), #2(7), #0(7), #1(6), #0(6), #2(6), #1(5), #0(5), #2(5), #1(4), #0(4), #2(4), #1(3), #0(3), #2(3), #0(2), #1(2), #0(1), #2(2), #0(Liftoff!), #1(1), #2(1), #1(Liftoff!), #2(Liftoff!), #3(9), #4(9), #3(8), #4(8), #3(7), #4(7), #3(6), #4(6), #3(5), #4(5), #3(4), #4(4), #3(3), #4(3), #3(2), #4(2), #3(1), #4(1), #3(Liftoff!), #4(Liftoff!), 
```

　　从输出结果可以看到，只有三个任务在同时执行。后面两个任务等前面的任务执行完成了，才开始执行。

#### 对线程池的进一步研究

　　来看一下 Executors 中这四种线程池是怎么创建的。

```java
public static ExecutorService newCachedThreadPool() {
    return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                                  60L, TimeUnit.SECONDS,
                                  new SynchronousQueue<Runnable>());
}

public static ExecutorService newFixedThreadPool(int nThreads) {
    return new ThreadPoolExecutor(nThreads, nThreads,
                                  0L, TimeUnit.MILLISECONDS,
                                  new LinkedBlockingQueue<Runnable>());
}

public static ExecutorService newSingleThreadExecutor() {
    return new FinalizableDelegatedExecutorService
        (new ThreadPoolExecutor(1, 1,
                                0L, TimeUnit.MILLISECONDS,
                                new LinkedBlockingQueue<Runnable>()));
}

public static ScheduledExecutorService newScheduledThreadPool(int corePoolSize) {
    return new ScheduledThreadPoolExecutor(corePoolSize);
}
public class ScheduledThreadPoolExecutor
        extends ThreadPoolExecutor
        implements ScheduledExecutorService {
    ...
}       
```

　　前面三种线程池，都是直接创建了 ThreadPoolExecutor 类的对象。ScheduledThreadPool  因为要实现定时功能，创建的是 ScheduledThreadPoolExecutor 类的对象。但 ScheduledThreadPoolExecutor 也是继承自ThreadPoolExecutor 。所以我们主要关注一下 ThreadPoolExecutor 。下面这个构造方法是参数最全的一个。

```java
/**
 * Creates a new {@code ThreadPoolExecutor} with the given initial
 * parameters.
 *
 * @param corePoolSize 线程池中维持的线程数量。
 *                     当线程数量不超过这个数时，即使线程处于空闲状态也不会被销毁，会一直等待任务到来。
 *                     但是如果设置 allowCoreThreadTimeOut 为 true，corePoolSize 就不再有效了。
 * @param maximumPoolSize 线程池中线程的最大数量。
 * @param keepAliveTime 当线程数量超过了 corePoolSize 时，多余的线程销毁前等待的时间。
 * @param unit keepAliveTime 的时间单位
 * @param workQueue 用来管理待执行任务的队列。
 * @param threadFactory 创建线程的工厂。
 * @param handler RejectedExecutionHandler 接口的实现对象。用于处理任务被拒绝执行的情况。
 *                被拒绝的原因可能是所有线程正在执行任务而任务队列容量又满了
 */
public ThreadPoolExecutor(int corePoolSize,
                          int maximumPoolSize,
                          long keepAliveTime,
                          TimeUnit unit,
                          BlockingQueue<Runnable> workQueue,
                          ThreadFactory threadFactory,
                          RejectedExecutionHandler handler) {
    ...
}
```

　　理解了这些参数，就很容易理解 Executors 中创建的几种线程池。当这几种线程池都不能满足需求的时候，我们直接可以通过 ThreadPoolExecutor 的构造方法来创建一个合适的线程池。那么，ThreadPoolExecutor 是怎么调度线程来执行任务的呢？

　　从 execute() 方法入手去理解。其中 ctl 只是一个原子操作的 int 型（AtomicInteger类）变量，但可以同时保存线程池状态和线程数量。我在另一篇文章中专门分析了这个 [ctl 的实现](http://wolfxu.com/2018/01/08/ThreadPoolExecutor-%E4%B8%AD%E7%9A%84-ctl-%E5%8F%98%E9%87%8F/)。

```java
public void execute(Runnable command) {
    if (command == null)
        throw new NullPointerException();
 
    int c = ctl.get();
    if (workerCountOf(c) < corePoolSize) {
        if (addWorker(command, true))
            return;
        c = ctl.get();
    }
    if (isRunning(c) && workQueue.offer(command)) {
        int recheck = ctl.get();
        if (! isRunning(recheck) && remove(command))
            reject(command);
        else if (workerCountOf(recheck) == 0)
            addWorker(null, false);
    }
    else if (!addWorker(command, false))
        reject(command);
}
```

　　如果线程数量小于核心线程数量，就创建新的线程来执行任务；不然就添加到任务队列中；如果添加到任务队列失败，就创建新的线程来执行；如果创建线程再失败（可能是线程池不再是RUNNING状态，或者线程数量已经达到了最大线程数量），就只能拒绝任务了。

　　上面说的线程，实际上都通过 Worker 来管理，每个 Worker 对象持有一个线程。而 Woker 实现了 Runable 接口，会在自己的管理的线程中来执行。Worker 的 run() 方法就是直接调用了 runWorker 这个方法。

```java
final void runWorker(Worker w) {
    ...
    Runnable task = w.firstTask;
    w.firstTask = null;
    ...
    while (task != null || (task = getTask()) != null) {
        ...
        task.run();
        ...
    }
    ...
    processWorkerExit(w, completedAbruptly);
}   
```

　　如果在创建 Worker 时，就指定了一个任务，会先执行这个任务。后面就是循环，不断地从任务队列获取任务去执行。获取任务时，核心线程会一直等待获取到新的任务。而一般线程会设置一个超时时间，这个时间就是创建线程池时指定的 keepAliveTime。超时之后，就退出循环了，Worker 的使命完成，马上会被释放。有两点要补充一下：

1. 核心线程和一般线程没有区分，只是去 getTask 时，根据当前线程的数量是否大于核心线程数量来决定要不要一直等待。
2. 可以设置 allowCoreThreadTimeOut 为 true，让核心线程获取任务时也会超时。

　　现在我们基本上搞清楚了线程池是如何调度线程来执行任务的。再来回顾一下前面 Executors 中创建的几种线程池。

　　Executors中创建的CachedThreadPoollExecutor，是用的同步队列，只有当前有线程在等待任务时，才能加入，实际上也不在队列中管理，是直接扔给了执行线程去执行。所以CachedThreadPool中，当新任务到来时，如果线程数小于核心线程数，是直接创建，不然就看当前有没有在等待任务的线程，有就交给该线程执行，没有就创建一个新线程去执行。

　　Executors中创建的FixedThreadPoolExecutor和SingleThreadlExecutor，都是核心数量等于最大数量，且它们的任务队列是无限容量的。当新任务到来时，如果线程数小于核心线程数，创建新线程去执行，不然就加到任务队列中等待。

　　最后，研究一下 ScheduledThreadPoolExecutor 是怎么实现定时任务的。ScheduledThreadPoolExecutor 实现了 ScheduledExecutorService 接口中的 schedule 等方法。调用 schedule() 方法时，会把需要定时执行的任务打包在 ScheduledFutureTask 对象中，然后加入到等待执行的队列中去。

```java
public ScheduledFuture<?> schedule(Runnable command, long delay, TimeUnit unit) {
    if (command == null || unit == null)
        throw new NullPointerException();
    RunnableScheduledFuture<?> t = decorateTask(command,
        new ScheduledFutureTask<Void>(command, null,
                                      triggerTime(delay, unit)));
    delayedExecute(t);
    return t;
}
```

　　ScheduledThreadPoolExecutor 中用 DelayedWorkQueue 来管理等待执行的任务。添加时，会根据执行时间，把任务排到队列中合适的位置，保证队列中的任务按执行时间先后排列。取出时，取队列头部的任务，如果队列头部没有任务，或者任务的执行时间还没到，就要等待。

```java
private void delayedExecute(RunnableScheduledFuture<?> task) {
    if (isShutdown())
        reject(task);
    else {
        super.getQueue().add(task);
        if (isShutdown() &&
            !canRunInCurrentRunState(task.isPeriodic()) &&
            remove(task))
            task.cancel(false);
        else
            ensurePrestart();
    }
} 
```

　　delayedExecute() 方法中，先把任务加入到任务队列中，然后调用 ensurePrestart() 方法去启动一个新线程（线程数量小于限定的核心线程数量才会启动新线程）。这个线程就会去队列中等待任务，任务队列会在任务执行时间到时返回任务给线程去执行。这样就实现了定时任务的执行。

### 4. 从任务中产生返回值

　　前面我们用 Runable 来定义任务，但是 Runable 执行完成后不会有返回值。当需要返回值时，可以实现 Callable 接口。Callable 需要通过 ExecutorService 中声明的 submit() 方法去执行。

```java
public interface Callable<V> {
    V call() throws Exception;
}
```
　　下面举一个计算年级平均分的例子。为了简化，假定每个班学生人数都是50人。为了计算年级平均分，要让各班去计算各自的总分。每个班计算总分的过程用 Callable 去执行。
```java
private static final int STUDENT_NUM_OF_EACH_CLASS = 50;

static class ClassScoreCaculator implements Callable<Integer> {
    private List<Integer> loadScore() {
        List<Integer> scoreList = new ArrayList<>();
        for (int i = 0; i < STUDENT_NUM_OF_EACH_CLASS; ++i) {
            scoreList.add((int) (Math.random() * 100));
        }
        return scoreList;
    }

    @Override
    public Integer call() throws Exception {
        List<Integer> scoreList = loadScore();
        Integer sum = 0;
        for (Integer score : scoreList) {
            sum += score;
        }
        return sum;
    }
}

public static void main(String[] args) {
    List<Future<Integer>> results = new ArrayList<>();
    ExecutorService executor = Executors.newCachedThreadPool();
    for (int i = 0; i < 12; ++i) {
        results.add(executor.submit(new ClassScoreCaculator()));
    }
    int sumScore = 0;
    for (Future<Integer> result : results) {
        try {
            sumScore += result.get();
        } catch (InterruptedException | ExecutionException e) {
            e.printStackTrace();
        }
    }
    int average = sumScore / (STUDENT_NUM_OF_EACH_CLASS * 12);
    System.out.print("average score is " + average);
}   
```

　　submit() 方法会返回 Future 对象。可以用 get() 方法去获取执行结果，get() 方法会一直阻塞，直到 Callable 执行完成返回结果。如果不希望阻塞，可以先用 isDone() 方法查询是否执行完成。也使用带超时时间参数的 get() 方法。注意到 get() 方法会抛出两种异常：InterruptedException 和 ExecutionException 。其中，InterruptedException 是调用 Future 对象的 cancel() 方法去取消任务时，可能会中断线程而抛出的异常。而 ExecutionException ，是执行任务过程中的异常。因为，Callable 的 call() 方法是会抛出异常的，这个异常会被封装到 ExecutionException 中抛出。

### 5. 休眠

　　当我们需要任务暂停一段时间，可以使用线程的 sleep() 方法。在线程休眠过程中，可能会有其他线程尝试中断当前线程，这时 sleep() 方法会抛出 InterruptedException ，结束休眠。我们可以在 catch 到中断异常之后，选择尽快结束当前线程的执行任务，当然也可以忽略，选择继续执行。

``` java
public static native void sleep(long millis) throws InterruptedException;
public static void sleep(long millis, int nanos) throws InterruptedException {...}
```

### 6. 优先级

　　线程的优先级表示线程的重要性，线程调度器倾向于让优先级高的线程先执行。可以用 Thread 的 getPriority() 方法读取线程的优先级，通过 setPriority() 方法可以修改线程的优先级。目前 Java 中的线程是映射到底层操作系统的线程，通过底层操作系统来实现的。所以优先级也被映射到底层操作系统中的线程优先级。但是，不同操作系统的优先级级别数量、策略都有所不同，Java 中的 10 个优先级并不能映射得很好。Thinking in Java 书上建议，调整优先级时，只使用 MAX_PRIORITY、NORM_PRIORITY 和 MIN_PRIORITY 三种级别。由于不同操作系统的线程调度策略不一样，因此我们在开发时不应该依赖于线程的执行顺序。

### 7. 让步

　　通过 Thread 的 yield() 方法，可以给线程调度器一个建议：当前线程的工作告一段落，可以让出 CPU 给其他线程使用了。当然，这只是一个建议，没有任何机制能保证它一定被采纳。所以，我们在开发时也不应该依赖于 yield() 方法。

### 8. 后台线程

　　后台（daemon）线程，也有就守护线程的。关于后台线程需要了解的主要有三点：

- 当所有非后台线程结束，程序也就会结束，所有的后台进程都被杀死。因此，不要把必须执行的任务放到后台线程中。
- 通过 setDaemon(true) 可以把线程标记为后台线程。这个方法要在线程开始运行之前调用，不然会抛出异常。
- 后台线程中创建的线程会被自动设成后台线程。原理是线程初始化的时候会获取当前线程的 daemon ，来设置自己的 daemon 。

　　下面看一个使用后台线程的例子。

```java
public class DaemonThreadStudy {
    private static class DaemonThreadFactory implements ThreadFactory {
        @Override
        public Thread newThread(Runnable r) {
            Thread thread = new Thread(r);
            thread.setDaemon(true);
            return thread;
        }
    }

    private static class DaemonFromFactory implements Runnable {
        @Override
        public void run() {
            try {
                while (true) {
                    Thread.sleep(100);
                    System.out.println(Thread.currentThread() + " " + this);
                }
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }

    public static void main(String[] args) throws InterruptedException {
        Executor executor = Executors.newCachedThreadPool(new DaemonThreadFactory());
        for (int i = 0; i < 10; i++) {
            executor.execute(new DaemonFromFactory());
        }
        System.out.println("All daemons started");
        Thread.sleep(100);
    }
}
```

### 9. join

　　一个线程如果要等待另一个线程执行完成，可以调用另一个线程的 join() 方法。调用 join() 方法之后，当前线程将被挂起，等待另一个线程执行结束。join() 方法也有一个带等待时间参数的重载版本，等待时间到了后，不管等待的线程是否执行完成都会返回。来看一个使用 join() 方法的例子。

```java
public static void main(String[] args) {
    Thread sleeper = new Thread(new Runnable() {
        @Override
        public void run() {
            try {
                Thread.sleep(10000);
            } catch (InterruptedException e) {
                System.out.println(Thread.currentThread().getName() + " interrupted");
            }
            System.out.println(Thread.currentThread().getName() + " has awakend");
        }
    }, "Sleeper");
    Thread joiner = new Thread(new Runnable() {
        @Override
        public void run() {
            try {
                sleeper.join();
            } catch (InterruptedException e) {
                System.out.println(Thread.currentThread().getName() + " interrupted");
            }
            System.out.println(Thread.currentThread().getName() + " join completed");
        }
    }, "Joiner");
    sleeper.start();
    joiner.start();
}
```

　　在上面的例子中，sleeper 休眠 10 秒，而 joiner 会一直等待 sleeper 执行完成。注意，join() 方法和 sleep() 方法一样会抛出中断异常。也就是说，线程在等待时，也可以通过调用 interrupt 方法去中断它。

　　来看一下 join() 方法的实现。

```java
public final synchronized void join(long millis) throws InterruptedException {
    long base = System.currentTimeMillis();
    long now = 0;

    if (millis < 0) {
        throw new IllegalArgumentException("timeout value is negative");
    }

    if (millis == 0) {
        while (isAlive()) {
            wait(0);
        }
    } else {
        while (isAlive()) {
            long delay = millis - now;
            if (delay <= 0) {
                break;
            }
            wait(delay);
            now = System.currentTimeMillis() - base;
        }
    }
}
```

　　实际上就是调用了线程对象的 wait() 方法。循环判断线程是否执行结束，没结束就继续 wait()。如果设置了超时时间的话，会在时间到了之后结束 wait()，退出循环。按照注释中的说法，Thread 对象在 terminate 的时候，会调用 notifyAll() 。这样，wait() 方法就能返回，join() 方法也就执行完了。为什么需要设个循环去判断 isAlive() 呢，因为我们有可能在程序的其他地方去调用被等待的线程对象的 notify() 和 notifyAll() 方法。如果没有循环的，join() 就会直接返回，不会等到线程执行结束了。

　　测试在其他地方调用被等待的线程的 notify() 方法时，还发现调用一个对象的 wait()、notify()、notifyAll() 等方法都需要先成为这个对象的 monitor 所有者，不然会抛出 IllegalMonitorStateException 异常。成为一个对象的 monitor 所有者有三种方法：

- 在这个对象的 synchronize 的方法中
- 在 synchronize 这个对象的代码块中
- 如果这个对象是 Class 类的对象，可以在类的静态的 synchronize 的方法中

　　其实三种方法本质上都是一样的，就是在调用 wait()、notify() 方法之前，得先对对象做 synchronize 。前面两种就不用说了。第三种方法，由于 Class 类的特殊性，类的静态的 synchronize 的方法，实际上就是对 Class 对象做的 synchronize。

### 10. 线程组

　　ThreadGroup ，这个东西没太大作用。看了书和很多资料，都说没什么意义。看了 ThreadGroup 类的源码，就是持有一个线程数组和一个线程组数组，方便进行统一操作，比如：interrupt() 方法。除此之外，还能通过 activeCount() 方法获取一下线程组内的线程数量。有些作用的是，ThreadGroup 可以对线程运行中没有被捕获的异常做处理。

### 11. 捕获异常

　　由于线程的特性，我们无法捕获从线程中逃逸的异常。一旦异常逃出任务的 run() 方法，就会向外传播 。我们需要用特殊的方式捕获线程中逃出的异常。在 Java 1.5 以前只能用线程组来捕获，在 1.5 版本之后，就有更好的方式可以处理了。

```java
class ExceptionRunable implements Runnable {
    @Override
    public void run() {
        throw new RuntimeException();
    }
}

class NativeExceptionHandling {
    public static void main(String[] args) {
        try {
            Executor executor = Executors.newCachedThreadPool();
            executor.execute(new ExceptionRunable());
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```

　　在上面的示例中，我们执行了一个会抛出异常的任务，并尝试用 try catch 去捕获异常，很显然，这是没有作用的，因为异常是在新的线程中抛出的。那么，我们改怎么去捕获这种异常呢？Java 1.5 引入了一个新的接口 Thread.UncaughtExceptionHandler，我们可以给线程设置一个异常处理器，去处理没有被捕获的异常。

```java
class MyUncaughtExceptionHandler implements Thread.UncaughtExceptionHandler {

    @Override
    public void uncaughtException(Thread t, Throwable e) {
        System.out.println("caught " + e);
    }
}

class HandlerThreadFactory implements ThreadFactory {

    @Override
    public Thread newThread(Runnable r) {
        Thread thread = new Thread(r);
        thread.setUncaughtExceptionHandler(new MyUncaughtExceptionHandler());
        return thread;
    }
}

class CaptureUncaughtException {
    public static void main(String[] args) {
        Executor executor = Executors.newCachedThreadPool(new HandlerThreadFactory());
        executor.execute(new ExceptionRunable());
    }
}
```

　　这里我们使用了 HandlerThreadFactory 来创建线程，通过调用 Thread 的成员方法 setUncaughtExceptionHandler() 给每个线程设置了 UncaughtExceptionHandler 。线程运行中没有被捕获的异常，会被扔给 UncaughtExceptionHandler 来处理，而不会向外传递。

　　进一步研究，看异常是怎么被传到处理器中的。先看 Thread 类中的 dispatchUncaughtException()  方法，这个方法是由 JVM 去调用的。之前的流程应该就是线程执行任务后，有没捕获的异常，然后 JVM 调用线程的 dispatchUncaughtException() 方法来处理异常。然后，获取异常处理器，把异常交给异常处理器的 uncaughtException() 方法。如果该线程对象设置了异常处理器，就用自身的，否则就交给线程组处理（ThreadGroup 也实现了 UncaughtExceptionHandler 接口）。

```java
public class Thread {
    ...
    private volatile UncaughtExceptionHandler uncaughtExceptionHandler;
    private static volatile UncaughtExceptionHandler defaultUncaughtExceptionHandler;
    private ThreadGroup group;
    ...
    /**
     * Dispatch an uncaught exception to the handler. This method is
     * intended to be called only by the JVM.
     */
    private void dispatchUncaughtException(Throwable e) {
        getUncaughtExceptionHandler().uncaughtException(this, e);
    }

    public UncaughtExceptionHandler getUncaughtExceptionHandler() {
        return uncaughtExceptionHandler != null ?
            uncaughtExceptionHandler : group;
    }
    ...
}
```
　　线程组中的处理流程是：首先找父线程组的处理方法；其次找线程中设置的默认异常处理器；都找不到就直接打印异常堆栈。
```java
public class ThreadGroup {
    ...
    public void uncaughtException(Thread t, Throwable e) {
        if (parent != null) {
            parent.uncaughtException(t, e);
        } else {
            Thread.UncaughtExceptionHandler ueh =
                Thread.getDefaultUncaughtExceptionHandler();
            if (ueh != null) {
                ueh.uncaughtException(t, e);
            } else if (!(e instanceof ThreadDeath)) {
                System.err.print("Exception in thread \""
                                 + t.getName() + "\" ");
                e.printStackTrace(System.err);
            }
        }
    }
    ...
}
```

　　总结一下，线程执行中没捕获的异常优先扔给线程对象中设置的异常处理器，其次给线程组，如果都没处理，会看是否设置了 Thread 类的默认异常处理器。

　　看到这里，我产生了一个疑问，按照这种机制，没捕获的异常最多是打个错误信息，而不会导致程序 crash 。那么，为什么在 android 中，异常会导致应用 crash 呢。原来，Android 在所有进程启动时，都给 Thread 设置了 defaultUncaughtExceptionHandler ，遇到异常时会让应用 crash 。想了解更多内容，请看这篇文章 [理解Android Crash处理流程](http://gityuan.com/2016/06/24/app-crash/) 。

### 12. 结语
　　这篇文章是我阅读《 Thinking In Java 》书中并发一章第2节，并结合源码以及测试的学习记录。对 Java 基础线程机制的学习到此就告一段落了。下一篇文章学习多线程开发的两个主要问题的解决：[Java多线程开发（二）| 多线程的竞争与协作](http://wolfxu.com/2018/01/08/Java%E5%A4%9A%E7%BA%BF%E7%A8%8B%E5%BC%80%E5%8F%91%EF%BC%88%E4%BA%8C%EF%BC%89-%E5%A4%9A%E7%BA%BF%E7%A8%8B%E7%9A%84%E7%AB%9E%E4%BA%89%E4%B8%8E%E5%8D%8F%E4%BD%9C/)。
