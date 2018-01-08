---
title: 【学习记录】Android深入学习之消息处理机制
date: 2018-01-08 17:17:47
tags:
- Android
- 源码解析
---

# 学习过程
跟着鸿洋_的博客的思路，结合7.0的源码进行学习，同时参考其他好的文章。

# 概述
主要涉及四个类：Looper、Handler、Message、MessageQueue。
Message是消息对象，MessageQueue是消息队列。Looper负责创建消息队列，并进入无限循环不断从消息队列中读取消息。而Handler负责发送消息到消息队列，以及消息的回调处理。

# Looper
### 1. Looper类的作用
源码的类注释中已经把Looper类的作用和使用方法说得很清楚了。
```java
/**
  * Class used to run a message loop for a thread.  Threads by default do
  * not have a message loop associated with them; to create one, call
  * {@link #prepare} in the thread that is to run the loop, and then
  * {@link #loop} to have it process messages until the loop is stopped.
  *
  * <p>Most interaction with a message loop is through the
  * {@link Handler} class.
  *
  * <p>This is a typical example of the implementation of a Looper thread,
  * using the separation of {@link #prepare} and {@link #loop} to create an
  * initial Handler to communicate with the Looper.
  *
  * <pre>
  *  class LooperThread extends Thread {
  *      public Handler mHandler;
  *
  *      public void run() {
  *          Looper.prepare();
  *
  *          mHandler = new Handler() {
  *              public void handleMessage(Message msg) {
  *                  // process incoming messages here
  *              }
  *          };
  *
  *          Looper.loop();
  *      }
  *  }</pre>
  */
```
Looper类的作用就是让线程进行消息循环。如果一个线程需要消息循环，只需要调用Looper类的prepare方法和loop方法就可以了。因此，Looper类中我们主要关注prepare和loop这两个方法，它们都是static方法。
### 2. prepare()方法
```java
private static void prepare(boolean quitAllowed) {
    
    if (sThreadLocal.get() != null) {
        
        throw new RuntimeException("Only one Looper may be created per thread");
    
    }
    
    sThreadLocal.set(new Looper(quitAllowed));

}
```
这里创建了一个Looper对象，然后保存到一个ThreadLocal的静态变量中。当sThreadLocal中取出的对象不为null时，会抛出异常，保证一个线程中只有一个Looper对象。***ThreadLocal后面再研究。***

然后看Looper的构造方法。
```java
private Looper(boolean quitAllowed)  {
    
    mQueue = new MessageQueue(quitAllowed);
    
    mThread = Thread.currentThread();

}
```
创建了一个MessageQueue（消息队列）。

### 3. loop()方法
```java
/**
 * Run the message queue in this thread. Be sure to call
 * {@link #quit()} to end the loop.
 */
public static void loop() {
    final Looper me = myLooper();
    if (me == null) {
        throw new RuntimeException("No Looper; Looper.prepare() wasn't called on this thread.");
    }
    final MessageQueue queue = me.mQueue;

    // Make sure the identity of this thread is that of the local process,
    // and keep track of what that identity token actually is.
    Binder.clearCallingIdentity();
    final long ident = Binder.clearCallingIdentity();

    for (;;) {
        Message msg = queue.next(); // might block
        if (msg == null) {
            // No message indicates that the message queue is quitting.
            return;
        }

        // This must be in a local variable, in case a UI event sets the logger
        final Printer logging = me.mLogging;
        if (logging != null) {
            logging.println(">>>>> Dispatching to " + msg.target + " " +
                    msg.callback + ": " + msg.what);
        }

        final long traceTag = me.mTraceTag;
        if (traceTag != 0) {
            Trace.traceBegin(traceTag, msg.target.getTraceName(msg));
        }
        try {
            msg.target.dispatchMessage(msg);
        } finally {
            if (traceTag != 0) {
                Trace.traceEnd(traceTag);
            }
        }

        if (logging != null) {
            logging.println("<<<<< Finished to " + msg.target + " " + msg.callback);
        }

        // Make sure that during the course of dispatching the
        // identity of the thread wasn't corrupted.
        final long newIdent = Binder.clearCallingIdentity();
        if (ident != newIdent) {
            Log.wtf(TAG, "Thread identity changed from 0x"
                    + Long.toHexString(ident) + " to 0x"
                    + Long.toHexString(newIdent) + " while dispatching to "
                    + msg.target.getClass().getName() + " "
                    + msg.callback + " what=" + msg.what);
        }

        msg.recycleUnchecked();
    }
}
```
***其中，Binder.clearCallingIdentity()的作用不清楚，先忽略。***
执行流程：
1. myLooper方法获取sThreadLocal中保存的Looper实例。因此再loop方法执行前应该先执行prepare方法。
1. 进入无限循环。
    1.   从消息队列中取出一条消息，如果没有消息则会阻塞；如果消息队列已释放，则会返回null，退出消息循环。
    1. 调用msg.target的dispatchMessage方法处理消息。target就是Handler的实例，负责接收处理这个消息。
    1. 回收msg资源。

### 4. Looper类小结
1. 每个线程最多只能有一个Looper对象。每个Looper对象创建并持有一个MessageQueue对象。
2. 通过调用Looper.loop方法使当前线程进入消息循环。当前线程的Looper对象循环从消息队列中取出消息，交由相应的Handler对象去处理。

# Handler
### 1. Handler类的作用
还是看源码中的类注释。
```java
/**
 * A Handler allows you to send and process {@link Message} and Runnable
 * objects associated with a thread's {@link MessageQueue}.  Each Handler
 * instance is associated with a single thread and that thread's message
 * queue.  When you create a new Handler, it is bound to the thread /
 * message queue of the thread that is creating it -- from that point on,
 * it will deliver messages and runnables to that message queue and execute
 * them as they come out of the message queue.
 * 
 * <p>There are two main uses for a Handler: (1) to schedule messages and
 * runnables to be executed as some point in the future; and (2) to enqueue
 * an action to be performed on a different thread than your own.
 * 
 * <p>Scheduling messages is accomplished with the
 * {@link #post}, {@link #postAtTime(Runnable, long)},
 * {@link #postDelayed}, {@link #sendEmptyMessage},
 * {@link #sendMessage}, {@link #sendMessageAtTime}, and
 * {@link #sendMessageDelayed} methods.  The <em>post</em> versions allow
 * you to enqueue Runnable objects to be called by the message queue when
 * they are received; the <em>sendMessage</em> versions allow you to enqueue
 * a {@link Message} object containing a bundle of data that will be
 * processed by the Handler's {@link #handleMessage} method (requiring that
 * you implement a subclass of Handler).
 * 
 * <p>When posting or sending to a Handler, you can either
 * allow the item to be processed as soon as the message queue is ready
 * to do so, or specify a delay before it gets processed or absolute time for
 * it to be processed.  The latter two allow you to implement timeouts,
 * ticks, and other timing-based behavior.
 * 
 * <p>When a
 * process is created for your application, its main thread is dedicated to
 * running a message queue that takes care of managing the top-level
 * application objects (activities, broadcast receivers, etc) and any windows
 * they create.  You can create your own threads, and communicate back with
 * the main application thread through a Handler.  This is done by calling
 * the same <em>post</em> or <em>sendMessage</em> methods as before, but from
 * your new thread.  The given Runnable or Message will then be scheduled
 * in the Handler's message queue and processed when appropriate.
 */
```
Handler用于发送和处理Message和Runnable对象。Handler对象在创建时关联当前线程的MessageQueue，且每个Handler对象只能关联一个MessageQueue。Handler对象发送Message和Runnable对象到关联的MessageQueue，然后当它们从MessageQueue中移出时，负责执行它们。

Handler的主要用途有两个：
1. 定时执行message或runnable。
1. 让其他线程执行某个操作。比如，在非UI线程发送一个消息，让UI线程更新界面。

Handler中重要的有以下几组方法：
1. 构造方法
1. sendMessage方法
1. post方法
1. dispatchMessage方法

### 2. 构造方法
Handler中有很多构造方法，但是最终分别进入到两个构造方法中。先来看下这两个构造方法有什么不同。
```java
public Handler(Looper looper, Callback callback, boolean async) {
    mLooper = looper;
    mQueue = looper.mQueue;
    mCallback = callback;
    mAsynchronous = async;
}
```
```java
public Handler(Callback callback, boolean async) {
    if (FIND_POTENTIAL_LEAKS) {
        final Class<? extends Handler> klass = getClass();
        if ((klass.isAnonymousClass() || klass.isMemberClass() || klass.isLocalClass()) &&
                (klass.getModifiers() & Modifier.STATIC) == 0) {
            Log.w(TAG, "The following Handler class should be static or leaks might occur: " +
                klass.getCanonicalName());
        }
    }

    mLooper = Looper.myLooper();
    if (mLooper == null) {
        throw new RuntimeException(
            "Can't create handler inside thread that has not called Looper.prepare()");
    }
    mQueue = mLooper.mQueue;
    mCallback = callback;
    mAsynchronous = async;
}
```
第二个构造方法前面那段主要是当Handler的之类不是static类时，警告可能会导致内存泄漏。忽略这个，两个方法的区别就在于mLooper这个成员变量的来源。第一个方法由参数传入，而第二个方法是获取当前线程的Looper。可以看到，构造方法里面就是对几个成员变量赋值而已。这里先了解一下这几个成员变量的作用。
- mLooper：Looper，消息循环
- mQueue：MessageQueue，消息队列，就是mLooper中持有的那个
- mCallback：Handler.CallBack，Handler中声明的接口，用于处理消息
- mAsynchronous：boolean，发送的消息是否是异步的。那么，这个异步到底是什么意思呢？我们在后面MessageQueue的next方法中再去详细了解。

### 3. sendMessage方法
sendMessage有一系列的方法：
- sendMessage(Message msg): boolean
- sendEmptyMessage(int what): boolean
- sendEmptyMessageDelayed(int what, long delayMillis): boolean
- sendMessageDelayed(Message msg, long delayMillis): boolean
- sendEmptyMessageAtTime(int what, long uptimeMillis): boolean
- sendMessageAtTime(Message msg, long uptimeMillis): boolean
- sendMessageAtFrontOfQueue(Message msg): boolean 

其中，sendMessage、sendEmptyMessage、sendMessageDelayed、sendEmptyMessageDelayed和sendEmptyMessageAtTime方法都调用sendMessageAtTime方法。所以，我们重点看sendMessageAtTime方法和sendMessageAtFrontOfQueue方法。

**sendMessageAtTime方法**
```java
public boolean sendMessageAtTime(Message msg, long uptimeMillis) {
    MessageQueue queue = mQueue;
    if (queue == null) {
        RuntimeException e = new RuntimeException(
                this + " sendMessageAtTime() called with no mQueue");
        Log.w("Looper", e.getMessage(), e);
        return false;
    }
    return enqueueMessage(queue, msg, uptimeMillis);
}
```
enquueMessage方法只是调用MessageQueue的enqueueMessage方法。
```java
private boolean enqueueMessage(MessageQueue queue, Message msg, long uptimeMillis) {
    msg.target = this;
    if (mAsynchronous) {
        msg.setAsynchronous(true);
    }
    return queue.enqueueMessage(msg, uptimeMillis);
}
```
uptimeMillis是消息分发的时间，基于SystemClock.uptimeMillis()。比如，sendMessageDelayed方法中使用SystemClock.uptimeMillis()加上延迟的时间。在消息队列中，消息是按分发时间的先后排列的。因此，这里的msg会插入到所有分发时间在uptimeMillis之前的消息后面。

**sendMessageAtFrontOfQueue方法**
```java
public final boolean sendMessageAtFrontOfQueue(Message msg) {
    MessageQueue queue = mQueue;
    if (queue == null) {
        RuntimeException e = new RuntimeException(
            this + " sendMessageAtTime() called with no mQueue");
        Log.w("Looper", e.getMessage(), e);
        return false;
    }
    return enqueueMessage(queue, msg, 0);
}
```
顾名思义，sendMessageAtFrontOfQueue方法就是把消息插入到队列的最前面。和sendMessageAtTime唯一的不同是，在调用enqueueMessage方法时传的uptimeMillis参数是0。用0来表示插入到消息队列最前面，也是比较自然的做法。

### 4. post方法
post方法把Runable对象添加到消息队列中，也有一系列方法。
- post(Runnable r): boolean
- postAtTime(Runnable r, long uptimeMillis): boolean
- postAtTime(Runnable r, Object token, long uptimeMillis): boolean
- postDelayed(Runnable r, long delayMillis): boolean
- postAtFrontOfQueue(Runnable r): boolean

这些方法实现都类似，都是通过getPostMessage方法，获取一个Message，同时把Runnable存到Message中，然后调用sendMessage方法。
```java
public final boolean post(Runnable r)
{
   return  sendMessageDelayed(getPostMessage(r), 0);
}
```
比较特殊的是带有token参数的postAtTime方法，这里的token是传给消息接收者的数据，会保存到Message的成员变量obj中。
```java
private static Message getPostMessage(Runnable r, Object token) {
    Message m = Message.obtain();
    m.obj = token;
    m.callback = r;
    return m;
}
```

### 5. dispatchMessage方法
dispatchMessage方法就是前面Looper中调用的处理消息的方法。
> msg.target.dispatchMessage(msg);

先看dispatchMessage方法的源码。
```java
public void dispatchMessage(Message msg) {
    if (msg.callback != null) {
        handleCallback(msg);
    } else {
        if (mCallback != null) {
            if (mCallback.handleMessage(msg)) {
                return;
            }
        }
        handleMessage(msg);
    }
}
```
如果消息中带有callback，直接执行callback（就是Message带的Runnable）；如果Handler中设置了CallBack，则调用CallBack来处理消息；前面两个都没有的话，才是调用handleMessage方法。handleMessage是空方法，子类可以重写该方法来处理消息。

### 6. Handler类小结
1. Handler创建时会关联一个Looper和Looper中持有的MessageQueue。
1. Handler可以在任意线程发送消息到关联的MessageQueue。
1. Handler在关联的Looper所在线程处理自己发送的消息。
1. Hander主要用于定时任务和在其他线程执行任务。

# MessageQueue
看完了Looper和Handler，已经基本理清了消息机制。再来看一下消息队列是怎么实现的。主要看一下把消息加入队列和从队列中取消息的实现。

### 1. 消息加入队列
MessageQueue的enqueueMessage方法负责把消息加入队列中。就是Handler中添加消息到消息队列调用的方法。下面贴的代码中省略了一下不影响主要逻辑的部分。
```java
boolean enqueueMessage(Message msg, long when) {
    ......

    synchronized (this) {
        ......

        msg.markInUse();
        msg.when = when;
        Message p = mMessages;
        boolean needWake;
        if (p == null || when == 0 || when < p.when) {
            msg.next = p;
            mMessages = msg;
            needWake = mBlocked;
        } else {
            needWake = mBlocked && p.target == null && msg.isAsynchronous();
            Message prev;
            for (;;) {
                prev = p;
                p = p.next;
                if (p == null || when < p.when) {
                    break;
                }
                if (needWake && p.isAsynchronous()) {
                    needWake = false;
                }
            }
            msg.next = p; 
            prev.next = msg;
        }

        if (needWake) {
            nativeWake(mPtr);
        }
    }
    return true;
}
```
成员变量mMessages是消息队列的头部，是一个Message对象。MessageQueue中通过Message的next形成一个链表。如果待插入的消息的分发时间是0，表示直接插入到队列头部。如果队列头部Message为空，或者待插入消息的分发时间小于队列头部Message，也把消息插入到队列头部。如果不满足插入到头部的条件的话，就遍历消息队列，按分发时间找到合适的插入位置。

### 2. 从队列中获取下一条消息
在Looper中已经看到，MessageQueue的next方法负责从队列中取下一条消息。
```java
Message next() {
    ...

    int pendingIdleHandlerCount = -1; // -1 only during first iteration
    int nextPollTimeoutMillis = 0;
    for (;;) {
        if (nextPollTimeoutMillis != 0) {
            Binder.flushPendingCommands();
        }

        nativePollOnce(ptr, nextPollTimeoutMillis);

        synchronized (this) {
            // 找下一个消息。找到的话就返回这个消息。
            final long now = SystemClock.uptimeMillis();
            Message prevMsg = null;
            Message msg = mMessages;
            if (msg != null && msg.target == null) {
                // 被Barrier阻塞。找队列中的下一个异步消息。
                do {
                    prevMsg = msg;
                    msg = msg.next;
                } while (msg != null && !msg.isAsynchronous());
            }
            if (msg != null) {
                if (now < msg.when) {
                    // 下一个消息的分发时间还没到。设置一个时间，等到消息准备分发时再唤醒。
                    nextPollTimeoutMillis = (int) Math.min(msg.when - now, Integer.MAX_VALUE);
                } else {
                    // 有一个可以分发的消息。
                    mBlocked = false;
                    if (prevMsg != null) {
                        prevMsg.next = msg.next;
                    } else {
                        mMessages = msg.next;
                    }
                    msg.next = null;
                    if (DEBUG) Log.v(TAG, "Returning message: " + msg);
                    msg.markInUse();
                    return msg;
                }
            } else {
                // 没有消息，也可能是消息都被Barrier阻塞了。
                nextPollTimeoutMillis = -1;
            }

            // 消息循环准备退出。释放资源，并且返回null。
            if (mQuitting) {
                dispose();
                return null;
            }

            // pendingIdleHandlerCount初始值为-1，所以第一次会去获取IdleHandlers的数量。
            // IdleHandler在需要等待下一条消息时去运行，因为这时是空闲的。
            if (pendingIdleHandlerCount < 0
                    && (mMessages == null || now < mMessages.when)) {
                pendingIdleHandlerCount = mIdleHandlers.size();
            }
            if (pendingIdleHandlerCount <= 0) {
                // No idle handlers to run.  Loop and wait some more.
                mBlocked = true;
                continue;
            }

            if (mPendingIdleHandlers == null) {
                mPendingIdleHandlers = new IdleHandler[Math.max(pendingIdleHandlerCount, 4)];
            }
            mPendingIdleHandlers = mIdleHandlers.toArray(mPendingIdleHandlers);
        }

        // 执行IdleHandler
        // 只在第一次迭代时会执行，因为后面会把pendingIdleHandlerCount设成0
        for (int i = 0; i < pendingIdleHandlerCount; i++) {
            final IdleHandler idler = mPendingIdleHandlers[i];
            mPendingIdleHandlers[i] = null; // release the reference to the handler

            boolean keep = false;
            try {
                keep = idler.queueIdle();
            } catch (Throwable t) {
                Log.wtf(TAG, "IdleHandler threw exception", t);
            }

            if (!keep) {
                synchronized (this) {
                    mIdleHandlers.remove(idler);
                }
            }
        }

        // 把pendingIdleHandlerCount设成0，后面的迭代不会再去执行IdleHandlers。
        pendingIdleHandlerCount = 0;

        // 执行完IdleHandler，可能已经有新的消息了，所以不需要再等待了。
        nextPollTimeoutMillis = 0;
    }
}
```
这个方法还是比较复杂的，所以我画了个流程图，能够看得清楚一些。


![MessageQueue的next方法流程图.png](http://upload-images.jianshu.io/upload_images/196189-beb18a631eb164d6.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240  "MessageQueue的next方法流程图.png")
通过流程图已经能理清next方法的执行过程。但是，还有两个需要解释的地方。一个是Barrier和异步消息，另一个是nativePollOnce这个方法。

#### Barrier和异步消息
Barrier是什么？
Barrier是阻塞器，会阻塞消息队列。它也是一个Message对象，只不过它的target是null。从next方法中可以看到，在Barrier后面所有消息，除了异步消息外都无法执行。
MessageQueue中对外提供了post和remove的方法。
```java
public int postSyncBarrier();
public void removeSyncBarrier(int token);
```
调用post方法时，会创建一个空的Message对象，时间设为当前的系统时间，同时生成一个token，保存在Message中。这个Message对象会像普通的消息一样被插入到消息队列中。调用remove方法时，根据token从消息队列中找到相应的Barrier，然后移除。
看一个具体的例子，这是ViewRootImpl中的一段代码。
```java
void scheduleTraversals() {
    ...
    mTraversalBarrier = mHandler.getLooper().getQueue().postSyncBarrier();
    ...
}

void unscheduleTraversals() {
    ...
    mHandler.getLooper().getQueue().removeSyncBarrier(mTraversalBarrier);
    ...
}
```
当要阻塞消息队列时，通过Handler获取到MessageQueue，然后直接调用MessageQueue的postSyncBarrier方法，保存下token。需要取消消息队列的阻塞时，通过先前保存的token去移除Barrier。
#### nativePollOnce方法
nativePollOnce是一个native方法。它的实现在frameworks/base/code/jni目录下的android_os_MessageQueue.cpp中。想要了解怎么找到这个实现的，可以阅读这篇文章：[Android JNI原理分析](http://gityuan.com/2016/05/28/android-jni/)。

```c++
static void android_os_MessageQueue_nativePollOnce(JNIEnv* env, jobject obj, jlong ptr, jint timeoutMillis) {
    NativeMessageQueue* nativeMessageQueue = reinterpret_cast<NativeMessageQueue*>(ptr);
    nativeMessageQueue->pollOnce(env, obj, timeoutMillis);
}
```
这里去调用了NativeMessageQueue的pollOnce方法。NativeMessageQueue的对象是在Java层的MessageQueue创建时，同时创建的。
```c++
void NativeMessageQueue::pollOnce(JNIEnv* env, jobject pollObj, int timeoutMillis) {
    ...
    mLooper->pollOnce(timeoutMillis);
    ...
}
```
这里调用mLooper的pollOnce方法。这里的mLooper是JNI层的Looper，是在创建NativeMessageQueue时创建的。这个类的实现在system/core/libutils/Looper.cpp中。
```c++
int Looper::pollOnce(int timeoutMillis, int* outFd, int* outEvents, void** outData) {
    int result = 0;
    for (;;) {
        ...

        if (result != 0) {
            ...
            return result;
        }

        result = pollInner(timeoutMillis);
    }
}
```
我们把pollOnce方法中不太重要的部分都去掉，只留下最主要的部分。实际上就是循环去调用pollInner方法，当pollInner方法的返回结果不为0时，这个方法就可以返回了。下面来看一下pollInner方法的实现。
```c++
int Looper::pollInner(int timeoutMillis) {

    // Adjust the timeout based on when the next message is due.
    if (timeoutMillis != 0 && mNextMessageUptime != LLONG_MAX) {
        nsecs_t now = systemTime(SYSTEM_TIME_MONOTONIC);
        int messageTimeoutMillis = toMillisecondTimeoutDelay(now, mNextMessageUptime);
        if (messageTimeoutMillis >= 0
                && (timeoutMillis < 0 || messageTimeoutMillis < timeoutMillis)) {
            timeoutMillis = messageTimeoutMillis;
        }
    }

    // Poll.
    int result = POLL_WAKE;
    ...

    struct epoll_event eventItems[EPOLL_MAX_EVENTS];
    int eventCount = epoll_wait(mEpollFd, eventItems, EPOLL_MAX_EVENTS, timeoutMillis);


    // Rebuild epoll set if needed.
    if (mEpollRebuildRequired) {
        mEpollRebuildRequired = false;
        rebuildEpollLocked();
        goto Done;
    }

    // Check for poll error.
    if (eventCount < 0) {
        if (errno == EINTR) {
            goto Done;
        }
        ALOGW("Poll failed with an unexpected error: %s", strerror(errno));
        result = POLL_ERROR;
        goto Done;
    }

    // Check for poll timeout.
    if (eventCount == 0) {
        result = POLL_TIMEOUT;
        goto Done;
    }

    for (int i = 0; i < eventCount; i++) {
        int fd = eventItems[i].data.fd;
        uint32_t epollEvents = eventItems[i].events;
        if (fd == mWakeEventFd) {
            if (epollEvents & EPOLLIN) {
                awoken();
            } else {
                ALOGW("Ignoring unexpected epoll events 0x%x on wake event fd.", epollEvents);
            }
        } else {
            ...
        }
    }
Done: ;

    // Invoke pending message callbacks.
    mNextMessageUptime = LLONG_MAX;
    while (mMessageEnvelopes.size() != 0) {
        nsecs_t now = systemTime(SYSTEM_TIME_MONOTONIC);
        const MessageEnvelope& messageEnvelope = mMessageEnvelopes.itemAt(0);
        if (messageEnvelope.uptime <= now) {
            // Remove the envelope from the list.
            // We keep a strong reference to the handler until the call to handleMessage
            // finishes.  Then we drop it so that the handler can be deleted *before*
            // we reacquire our lock.
            { // obtain handler
                sp<MessageHandler> handler = messageEnvelope.handler;
                Message message = messageEnvelope.message;
                mMessageEnvelopes.removeAt(0);
                mSendingMessage = true;
                mLock.unlock();

                handler->handleMessage(message);
            } // release handler

            mLock.lock();
            mSendingMessage = false;
            result = POLL_CALLBACK;
        } else {
            // The last message left at the head of the queue determines the next wakeup time.
            mNextMessageUptime = messageEnvelope.uptime;
            break;
        }
    }

    ...

    // Invoke all response callbacks.
    for (size_t i = 0; i < mResponses.size(); i++) {
        Response& response = mResponses.editItemAt(i);
        if (response.request.ident == POLL_CALLBACK) {
            int fd = response.request.fd;
            int events = response.events;
            void* data = response.request.data;

            // Invoke the callback.  Note that the file descriptor may be closed by
            // the callback (and potentially even reused) before the function returns so
            // we need to be a little careful when removing the file descriptor afterwards.
            int callbackResult = response.request.callback->handleEvent(fd, events, data);
            if (callbackResult == 0) {
                removeFd(fd, response.request.seq);
            }

            // Clear the callback reference in the response structure promptly because we
            // will not clear the response vector itself until the next poll.
            response.request.callback.clear();
            result = POLL_CALLBACK;
        }
    }
    return result;
}
```

pollInner方法中调用了epoll_wait()，等待消息到来，或者等到超时返回。如果有消息到来，则进行处理。

poolInner方法的流程：（摘自[[M0]Android Native层Looper详解](http://blog.csdn.net/chwan_gogogo/article/details/46953549)）
- 调整timeout：
- mNextMessageUptime 是 消息队列 mMessageEnvelopes 中最近一个即将要被处理的message的时间点。
- 所以需要根据mNextMessageUptime 与 调用者传下来的timeoutMillis 比较计算出一个最小的timeout，这将决定epoll_wait() 可能会阻塞多久才会返回。
- epoll_wait()：
epoll_wait()这里会阻塞，在三种情况下回返回，返回值eventCount为上来的epoll event数量。 
- 出现错误返回， eventCount < 0; 
- timeout返回，eventCount = 0，表明监听的文件描述符中都没有事件发生，将直接进行native message的处理； 
- 监听的文件描述符中有事件发生导致的返回，eventCount > 0; 有eventCount 数量的epoll event 上来。

- 处理epoll_wait() 返回的epoll events. 
判断epoll event 是哪个fd上发生的事件 
- 如果是mWakeEventFd，则执行awoken(). awoken() 只是将数据read出来，然后继续往下处理了。其目的也就是使epoll_wait() 从阻塞中返回。
- 如果是通过Looper.addFd() 接口加入到epoll监听队列的fd，并不是立马处理，而是先push到mResponses，后面再处理。
- 处理消息队列 mMessageEnvelopes 中的Message. 
- 如果还没有到处理时间，就更新一下mNextMessageUptime
- 处理刚才放入mResponses中的事件. 
- 只处理 ident 为POLL_CALLBACK的事件。其他事件在 pollOnce 中处理

# 参考资料
1. [Android 异步消息处理机制 让你深入理解 Looper、Handler、Message三者关系](http://blog.csdn.net/lmj623565791/article/details/38377229)
1. [android的消息处理机制（图+源码分析）——Looper,Handler,Message](http://www.cnblogs.com/codingmyworld/archive/2011/09/12/2174255.html)
1. [Android消息处理机制(Handler、Looper、MessageQueue与Message)](http://www.cnblogs.com/angeldevil/p/3340644.html)(里面关于同步消息、异步消息讲得很清楚)
1. [Android应用程序消息处理机制（Looper、Handler）分析](http://blog.csdn.net/luoshengyang/article/details/6817933/)
1. [Android JNI原理分析](http://gityuan.com/2016/05/28/android-jni/)
1. [[M0]Android Native层Looper详解](http://blog.csdn.net/chwan_gogogo/article/details/46953549)
