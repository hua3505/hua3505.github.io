---
title: ThreadPoolExecutor 中的 ctl 变量
date: 2018-01-08 18:19:47
tags:
- Android
- 源码解析
---
最近在看 Java 线程池的实现，发现里面有一个 int 类型的成员变量，同时表示线程池运行状态和线程数量。理解了一下这块的实现，挺有意思的，所以单独拿出来跟大家分享一下。

### 为什么要研究一个 int 变量

其实一开始，我是在看 execute 方法的实现……

```java
public void execute(Runnable command) {
    if (command == null)
        throw new NullPointerException();
     
    int c = ctl.get();
    // 这里 c 用来获取线程数量
    if (workerCountOf(c) < corePoolSize) {
        if (addWorker(command, true))
            return;
        c = ctl.get();
    }
    // 这里 c 用来判断线程池是否处于运行状态
    if (isRunning(c) && workQueue.offer(command)) {
        int recheck = ctl.get();
        if (!isRunning(recheck) && remove(command))
            reject(command);
        else if (workerCountOf(recheck) == 0)
            addWorker(null, false);
    }
    else if (!addWorker(command, false))
        reject(command);
}
```

我注意到从 ctl（AtomicInteger类型） 中取出来的这个 int 变量 c，它一会儿用来获取线程数量，一会儿又用来判断线程池是否处于运行状态。我很好奇，于是点进去看了这两个方法的实现。

```java
private static int workerCountOf(int c) { 
    return c & CAPACITY; 
}

private static boolean isRunning(int c) {
    return c < SHUTDOWN;
}
```

这下就更疑惑了。CAPACITY 是什么？是怎么通过按位与运算得到线程数量的呢？SHUTDOWN 又是什么？有点意思，我决定好好研究一下它是怎么实现的。

### 怎么实现的

来看一下 ctl 这个成员变量以及相关的值的声明。

```java
private final AtomicInteger ctl = new AtomicInteger(ctlOf(RUNNING, 0));
private static final int COUNT_BITS = Integer.SIZE - 3;      // 29
private static final int CAPACITY   = (1 << COUNT_BITS) - 1; // 00011111 ... ... 11111111

// 状态在高位存储
private static final int RUNNING    = -1 << COUNT_BITS;      // 11100000 ... ... 00000000
private static final int SHUTDOWN   =  0 << COUNT_BITS;      // 00000000 ... ... 00000000
private static final int STOP       =  1 << COUNT_BITS;      // 00100000 ... ... 00000000
private static final int TIDYING    =  2 << COUNT_BITS;      // 01000000 ... ... 00000000
private static final int TERMINATED =  3 << COUNT_BITS;      // 01100000 ... ... 00000000

private static int ctlOf(int rs, int wc) { return rs | wc; }
```

ctl 是一个 AtomicInteger 的类，就是让保存的 int 变量的更新都是原子操作，保证线程安全。 ctlOf 方法就是组合运行状态和工作线程数量。可以看到，ctlOf 方法是通过按位或的方式来实现的。为什么能这样做呢？因为，这里把一个 int 变量拆成两部分来用。前面3位用来表示状态，后面29位用来表示工程线程数量。所以，工作线程数量最大不能超过 2^29-1 ，ThreadPoolExecutor 的设计者也是考虑不太可能超过这个数，暂时就用了29位。

![ctl 变量的位分布](http://upload-images.jianshu.io/upload_images/196189-6a737cbcc0672c52.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240 "ctl 变量的位分布")

了解了 ctl 变量的结构，再回过头来看前面提到的两个方法。

```java
private static int workerCountOf(int c) { 

    return c & CAPACITY; 

}

private static boolean isRunning(int c) {

    return c < SHUTDOWN;

}
```

workerCountOf 方法很好理解，CAPACITY 的值是00011111 ... … 11111111，按位与之后去掉了前面三位，保留了后面四位。所以，拿到的就是工作线程的数量。

isRunning 方法中，直接拿 ctl 的值和 SHUTDOWN 作比较。这个要先知道在 RUNNING 状态下，ctl 的值是什么样的。初始状态，ctl 的值是11100000 ... … 00000000，表示 RUNNING 状态，和0个工作线程。后面，每创建一个新线程，都把 ctl 加一。当有5个工作线程时，ctl 的值是11100000 ... … 00000101。在 RUNNING 状态下，ctl 始终是负值，而 SHUTDOWN 是0，所以可以通过直接比较 ctl 的值来确定状态。

### 思考

善用位运算，有时候可以给我们节省很多空间。但是，在这里，明显不是为了省空间了，因为就算用两个值分开表示状态和工作线程数量，也就8个字节而已。我猜测是为了在多线程环境下保证运行状态和线程数量的统一。把这两个值放到一个 int 变量中，然后用 AtomicInteger 进行存储和读写，就可以保证这两个值始终是统一的。如果用两个变量保存，即使用了 AtomicInteger ，也可能出现一个改了，另一个还没改的情况。
