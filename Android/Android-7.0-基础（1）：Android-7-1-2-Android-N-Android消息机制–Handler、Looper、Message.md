---
title: Android N 基础（1）：Android 7.1.2 Android消息处理机制分析（从JAVA层到NATIVE层）– Handler、Looper、Message
cover: https://raw.githubusercontent.com/zhoujinjianx/PicGo/master/hexo.themes/bing-wallpaper-2018.04.01.jpg
categories: 
  - Android
tags:
  - Android
toc: true
abbrlink: 20170808
date: 2017-08-08 09:25:00
---


> ● framework/base/core/java/andorid/os/Handler.java

> ● framework/base/core/java/andorid/os/Looper.java

> ● framework/base/core/java/andorid/os/Message.java

> ● framework/base/core/java/andorid/os/MessageQueue.java

> ● framework/base/core/java/andorid/os/MessageQueue.java

> ● framework/base/core/jni/android_os_MessageQueue.cpp

> ● framework/base/core/java/andorid/os/Looper.java (Java层）

> ● system/core/libutils/Looper.cpp ( Native层)

> ● framework/base/native/android/looper.cpp (ALoop对象)

> ● framework/native/include/android/looper.h


## Ⅰ、Android消息机制(Java层)

### 一、Android消息机制相关类、概念（Java层）

**主线程（UI线程）** 定义：当程序第一次启动时，Android会同时启动一条主线程（Main Thread） 作用：主线程主要负责处理与UI相关的事件

**Message（消息）** 定义：Handler接收和处理的消息对象（Bean对象） 作用：通信时相关信息的存放和传递

**ThreadLocal** 定义：线程内部的数据存储类 作用：负责存储和获取本线程的Looper

**MessageQueue（消息队列）** 定义：采用单链表的数据结构来存储消息列表 作用：用来存放通过Handler发过来的Message，按照先进先出执行

**Handler（处理者）** 定义：Message的主要处理者 作用：负责发送Message到消息队列&处理Looper分派过来的Message

**Looper（循环器）** 定义：扮演Message Queue和Handler之间桥梁的角色 作用： 消息循环：循环取出Message Queue的Message 消息派发：将取出的Message交付给相应的Handler

### 二、类关系图（Java层）

![Markdown](https://raw.githubusercontent.com/zhoujinjianx/PicGo/master/android.message/01-android-handler-looper-message.jpg)

> ● Looper有一个MessageQueue消息队列；
● MessageQueue有一组待处理的Message；
●Message中有一个用于处理消息的Handler；
● Handler中有Looper和MessageQueue。

#### 典型实例:

先展示一个典型的关于Handler/Looper的线程

```java
class LooperThread extends Thread {
    public Handler mHandler;

    public void run() {
        Looper.prepare();

        mHandler = new Handler() {
            public void handleMessage(Message msg) {
                //TODO 定义消息处理逻辑.
            }
        };

        Looper.loop();
    }
}
```

接下来，围绕着这个实例展开详细分析。

### 三、Looper源码分析（Java层）

对于无参的情况，默认调用prepare(true)，表示的是这个Looper运行退出，而对于false的情况则表示当前Looper不运行退出。

```java
private static void prepare(boolean quitAllowed) {
    //每个线程只允许执行一次该方法，第二次执行时线程的TLS已有数据，则会抛出异常。
    if (sThreadLocal.get() != null) {
        throw new RuntimeException("Only one Looper may be created per thread");
    }
    //创建Looper对象，并保存到当前线程的TLS区域
    sThreadLocal.set(new Looper(quitAllowed));
}
```

### 3.1 知识：ThreadLocal介绍

这里的sThreadLocal是ThreadLocal类型，下面，先说说ThreadLocal。

ThreadLocal： 线程本地存储区（Thread Local Storage，简称为TLS），每个线程都有自己的私有的本地存储区域，不同线程之间彼此不能访问对方的TLS区域。TLS常用的操作方法：

![Markdown](https://raw.githubusercontent.com/zhoujinjianx/PicGo/master/android.message/02-android-handler-looper-message-_Thread_Local.png)

```java
ThreadLocal.set(T value)：将value存储到当前线程的TLS区域，源码如下：

  public void set(T value) {
      Thread currentThread = Thread.currentThread(); //获取当前线程
      Values values = values(currentThread); //查找当前线程的本地储存区
      if (values == null) {
          //当线程本地存储区，尚未存储该线程相关信息时，则创建Values对象
          values = initializeValues(currentThread);
      }
      //保存数据value到当前线程this
      values.put(this, value);
  }
```

ThreadLocal.get()：获取当前线程TLS区域的数据，源码如下：

```java
  public T get() {
      Thread currentThread = Thread.currentThread(); //获取当前线程
      Values values = values(currentThread); //查找当前线程的本地储存区
      if (values != null) {
          Object[] table = values.table;
          int index = hash & values.mask;
          if (this.reference == table[index]) {
              return (T) table[index + 1]; //返回当前线程储存区中的数据
          }
      } else {
          //创建Values对象
          values = initializeValues(currentThread);
      }
      return (T) values.getAfterMiss(this); //从目标线程存储区没有查询是则返回null
  }
```

ThreadLocal的get()和set()方法操作的类型都是泛型，接着回到前面提到的sThreadLocal变量，其定义如下：

```java
static final ThreadLocal<Looper> sThreadLocal = new ThreadLocal<Looper>()
```

可见sThreadLocal的get()和set()操作的类型都是Looper类型。

### (1) Looper.prepare()

Looper.prepare()在每个线程只允许执行一次，该方法会创建Looper对象，Looper的构造方法中会创建一个MessageQueue对象，再将Looper对象保存到当前线程TLS。

对于Looper类型的构造方法如下：

```java
private Looper(boolean quitAllowed) {
    mQueue = new MessageQueue(quitAllowed);  //创建MessageQueue对象.
    mThread = Thread.currentThread();  //记录当前线程.
}
```

另外，与prepare()相近功能的，还有一个prepareMainLooper()方法，该方法主要在ActivityThread类中使用。

```java
public static void prepareMainLooper() {
    prepare(false); //设置不允许退出的Looper
    synchronized (Looper.class) {
        //将当前的Looper保存为主Looper，每个线程只允许执行一次。
        if (sMainLooper != null) {
            throw new IllegalStateException("The main Looper has already been prepared.");
        }
        sMainLooper = myLooper();
    }
}
```

Looper.prepare()在每个线程只允许执行一次，该方法会创建Looper对象，Looper的构造方法中会创建一个MessageQueue对象，再将Looper对象保存到当前线程TLS。

对于Looper类型的构造方法如下：

```java
private Looper(boolean quitAllowed) {
    mQueue = new MessageQueue(quitAllowed);  //创建MessageQueue对象.稍后详细介绍
    mThread = Thread.currentThread();  //记录当前线程.
}
```

另外，与prepare()相近功能的，还有一个prepareMainLooper()方法，该方法主要在主线程（UI线程）ActivityThread类中使用。

```java
public static void prepareMainLooper() {
    prepare(false); //设置不允许退出的Looper
    synchronized (Looper.class) {
        //将当前的Looper保存为主Looper，每个线程只允许执行一次。
        if (sMainLooper != null) {
            throw new IllegalStateException("The main Looper has already been prepared.");
        }
        sMainLooper = myLooper();
    }
}
```

### （2）创建MessageQueue()

MessageQueue是消息机制的Java层和C++层的连接纽带，大部分核心方法都交给native层来处理，其中MessageQueue类中涉及的native方法如下：

```java
private native static long nativeInit();
private native void nativePollOnce(long ptr, int timeoutMillis);
private native static void nativeWake(long ptr);
```

关于这些native方法的介绍，见第二节：Android消息机制(native篇)。

```java
MessageQueue(boolean quitAllowed) {
    mQuitAllowed = quitAllowed;
    //通过native方法初始化消息队列，其中mPtr是供native代码使用
    mPtr = nativeInit();
}
```

> **MessageQueue创建过程总结**：

> **1、Looper的prepare或者prepareMainLooper静态方法被调用，将一个Looper对象保存在ThreadLocal里面。**
**2、Looper对象的初始化方法里，首先会新建一个MessageQueue对象。**
**3、MessageQueue对象的初始化方法通过JNI初始化C++层的NativeMessageQueue对象。**
**4、NativeMessageQueue对象在创建过程中，会初始化一个C++层的Looper对象。**
**5、c层的Looper对象在创建的过程中，会在内部创建一个管道（pipe），并将这个管道的读写fd都保存在 mWakeReadPipeFd和mWakeWritePipeFd中。然后新建一个epoll实例，并将两个fd注册进去。**
**6、利用epoll的机制，可以做到当管道没有消息时，线程睡眠在读端的fd上，当其他线程往管道写数据时，本线程便会被唤醒以进行消息处理。**

### (3) Looper.loop()

```java
public static void loop() {
    final Looper me = myLooper();  //获取TLS存储的Looper对象
    if (me == null) {
        throw new RuntimeException("No Looper; Looper.prepare() wasn't called on this thread.");
    }
    final MessageQueue queue = me.mQueue;  //获取Looper对象中的消息队列

    Binder.clearCallingIdentity();
    //确保在权限检查时基于本地进程，而不是基于最初调用进程。
    final long ident = Binder.clearCallingIdentity();

    for (;;) { //进入loop的主循环方法
        Message msg = queue.next(); //可能会阻塞 May be block
        if (msg == null) { //没有消息，则退出循环
            return;
        }

        Printer logging = me.mLogging;  //默认为null，可通过setMessageLogging()方法来指定输出，用于debug功能
        if (logging != null) {
            logging.println(">>>>> Dispatching to " + msg.target + " " +
                    msg.callback + ": " + msg.what);
        }
        msg.target.dispatchMessage(msg); //用于分发Message
        if (logging != null) {
            logging.println("<<<<< Finished to " + msg.target + " " + msg.callback);
        }

        final long newIdent = Binder.clearCallingIdentity(); //确保分发过程中identity不会损坏
        if (ident != newIdent) {
             //打印identity改变的log，在分发消息过程中是不希望身份被改变的。
        }
        msg.recycleUnchecked();  //将Message放入消息池
    }
}
```

loop()进入循环模式，不断重复下面的操作，直到没有消息时退出循环

读取MessageQueue的下一条Message；把Message分发给相应的target； A1：Looper.loop()循环中的msg.target是什么时候被赋值的？handler.sendMessage()最终会进入MessageQueue.enqueueMessage()，就是在这里面复制的。 稍后再handler.sendMessage()详细介绍。 再把分发后的Message回收到消息池，以便重复利用。Looper.loop()是消息处理的核心部分。

#### 3.1 MessageQueue.next()

```java
Message next() {
    final long ptr = mPtr;
    if (ptr == 0) { //当消息循环已经退出，则直接返回
        return null;
    }
    int pendingIdleHandlerCount = -1; // 循环迭代的首次为-1
    int nextPollTimeoutMillis = 0;
    for (;;) {
        if (nextPollTimeoutMillis != 0) {
            Binder.flushPendingCommands();
        }
        //阻塞操作，当等待nextPollTimeoutMillis时长，或者消息队列被唤醒，都会返回
        nativePollOnce(ptr, nextPollTimeoutMillis);
        synchronized (this) {
            final long now = SystemClock.uptimeMillis();
            Message prevMsg = null;
            Message msg = mMessages;
            if (msg != null && msg.target == null) {
                //当消息Handler为空时，查询MessageQueue中的下一条异步消息msg，则退出循环。
                do {
                    prevMsg = msg;
                    msg = msg.next;
                } while (msg != null && !msg.isAsynchronous());
            }
            if (msg != null) {
                if (now < msg.when) {
                    //当异步消息触发时间大于当前时间，则设置下一次轮询的超时时长
                    nextPollTimeoutMillis = (int) Math.min(msg.when - now, Integer.MAX_VALUE);
                } else {
                    // 获取一条消息，并返回
                    mBlocked = false;
                    if (prevMsg != null) {
                        prevMsg.next = msg.next;
                    } else {
                        mMessages = msg.next;
                    }
                    msg.next = null;
                    //设置消息的使用状态，即flags |= FLAG_IN_USE
                    msg.markInUse();
                    return msg;   //成功地获取MessageQueue中的下一条即将要执行的消息
                }
            } else {
                //没有消息
                nextPollTimeoutMillis = -1;
            }
            //消息正在退出，返回null
            if (mQuitting) {
                dispose();
                return null;
            }
            //当消息队列为空，或者是消息队列的第一个消息时
            if (pendingIdleHandlerCount < 0 && (mMessages == null || now < mMessages.when)) {
                pendingIdleHandlerCount = mIdleHandlers.size();
            }
            if (pendingIdleHandlerCount <= 0) {
                //没有idle handlers 需要运行，则循环并等待。
                mBlocked = true;
                continue;
            }
            if (mPendingIdleHandlers == null) {
                mPendingIdleHandlers = new IdleHandler[Math.max(pendingIdleHandlerCount, 4)];
            }
            mPendingIdleHandlers = mIdleHandlers.toArray(mPendingIdleHandlers);
        }
        //只有第一次循环时，会运行idle handlers，执行完成后，重置pendingIdleHandlerCount为0.
        for (int i = 0; i < pendingIdleHandlerCount; i++) {
            final IdleHandler idler = mPendingIdleHandlers[i];
            mPendingIdleHandlers[i] = null; //去掉handler的引用
            boolean keep = false;
            try {
                keep = idler.queueIdle();  //idle时执行的方法
            } catch (Throwable t) {
                Log.wtf(TAG, "IdleHandler threw exception", t);
            }
            if (!keep) {
                synchronized (this) {
                    mIdleHandlers.remove(idler);
                }
            }
        }
        //重置idle handler个数为0，以保证不会再次重复运行
        pendingIdleHandlerCount = 0;
        //当调用一个空闲handler时，一个新message能够被分发，因此无需等待可以直接查询pending message.
        nextPollTimeoutMillis = 0;
    }
}
```

nativePollOnce()在native做了大量的工作，Android消息机制(native篇)稍后详细分析。

> **消息循环Looper.loop()总结：**

> **首先通过调用Looper的loop方法开始消息监听。loop方法里会调用MessageQueue的next方法。next方法会堵塞线程直到有消息到来为止。**

> **next方法通过调用nativePollOnce方法来监听事件。next方法内部逻辑如下所示(简化)：**

> **a.进入死循环，以参数timout=0调用nativePollOnce方法。**

> **b.如果消息队列中有消息，nativePollOnce方法会将消息保存在mMessage成员中。nativePollOnce方法返回后立刻检查mMessage成员是否为空。**

> **c.如果mMessage不为空，那么检查它指定的运行时间。如果比当前时间要前，那么马上返回这个mMessage，否则设置> timeout为两者之差，进入下一次循环。**

> **d. 如果mMessage为空，那么设置timeout为-1，即下次循环nativePollOnce永久堵塞。 nativePollOnce方法内部利用epoll机制在之前建立的管道上等待数据写入。接收到数据后马上读取并返回结果。**

这里先提一下为什么会阻塞，稍后再Android消息处理(Native层)分析，主要是底层使用了Linux epoll： [Linux IO模式及 select、poll、epoll详解](https://segmentfault.com/a/1190000003063859)

#### 3.2 Looper.quit()

```java
public void quit() {
    mQueue.quit(false); //消息移除
}

public void quitSafely() {
    mQueue.quit(true); //安全地消息移除
}
```

Looper.quit()方法的实现最终调用的是MessageQueue.quit()方法

```java
MessageQueue.quit()

void quit(boolean safe) {
        // 当mQuitAllowed为false，表示不运行退出，强行调用quit()会抛出异常
        if (!mQuitAllowed) {
            throw new IllegalStateException("Main thread not allowed to quit.");
        }
        synchronized (this) {
            if (mQuitting) { //防止多次执行退出操作
                return;
            }
            mQuitting = true;
            if (safe) {
                removeAllFutureMessagesLocked(); //移除尚未触发的所有消息
            } else {
                removeAllMessagesLocked(); //移除所有的消息
            }
            //mQuitting=false，那么认定为 mPtr != 0
            nativeWake(mPtr);
        }
    }
```

消息退出的方式：

当safe =true时，只移除尚未触发的所有消息，对于正在触发的消息并不移除； 当safe =flase时，移除所有的消息

前面构造了Looper 、MessageQueue，假设此时没有message处理，Looper.loop()会阻塞在MessageQueue.next()。 接下来讲一下Handler如何发送和处理消息。

### 四、异步处理大师 Handler()

### (1) 构造Handler()

### 1.1 无参构造Handler()

```java
public Handler() {
    this(null, false);
}
public Handler(Callback callback, boolean async) {
    //匿名类、内部类或本地类都必须申明为static，否则会警告可能出现内存泄露
    if (FIND_POTENTIAL_LEAKS) {
        final Class<? extends Handler> klass = getClass();
        if ((klass.isAnonymousClass() || klass.isMemberClass() || klass.isLocalClass()) &&
                (klass.getModifiers() & Modifier.STATIC) == 0) {
            Log.w(TAG, "The following Handler class should be static or leaks might occur: " +
                klass.getCanonicalName());
        }
    }
    //必须先执行Looper.prepare()，才能获取Looper对象，否则为null.
    mLooper = Looper.myLooper();  //从当前线程的TLS中获取Looper对象
    if (mLooper == null) {
        throw new RuntimeException("");
    }
    mQueue = mLooper.mQueue; //消息队列，来自Looper对象
    mCallback = callback;  //回调方法
    mAsynchronous = async; //设置消息是否为异步处理方式
}
```

对于Handler的无参构造方法，默认采用当前线程TLS中的Looper对象，并且callback回调方法为null，且消息为同步处理方式。只要执行的Looper.prepare()方法，那么便可以获取有效的Looper对象。

### 1.2 有参构造Handler()

```java
public Handler(Looper looper) {
    this(looper, null, false);
}

public Handler(Looper looper, Callback callback, boolean async) {
    mLooper = looper;
    mQueue = looper.mQueue;
    mCallback = callback;
    mAsynchronous = async;
}
```

Handler类在构造方法中，可指定Looper，Callback回调方法以及消息的处理方式(同步或异步)，对于无参的handler，默认是当前线程的Looper。

### (2) 使用 Handler发送消息Handler.sendMessage()、Handler.post(Ruunable r)

第一种方式：sendMessage(Message msg)

![Markdown](https://raw.githubusercontent.com/zhoujinjianx/PicGo/master/android.message/03-android-handler-looper-message-java-sendmessage.png)

```java
//从这里开始
public final boolean sendEmptyMessage(int what)
{
    return sendEmptyMessageDelayed(what, 0);
}

//往下追踪
public final boolean sendEmptyMessageDelayed(int what, long delayMillis) {
    Message msg = Message.obtain();
    msg.what = what;
    return sendMessageDelayed(msg, delayMillis);
}

//往下追踪
public final boolean sendMessageDelayed(Message msg, long delayMillis)
{
    if (delayMillis < 0) {
        delayMillis = 0;
    }
    return sendMessageAtTime(msg, SystemClock.uptimeMillis() + delayMillis);
}

//往下追踪
public boolean sendMessageAtTime(Message msg, long uptimeMillis) {
    //直接获取MessageQueue
    MessageQueue queue = mQueue;
    if (queue == null) {
        RuntimeException e = new RuntimeException(
                this + " sendMessageAtTime() called with no mQueue");
        Log.w("Looper", e.getMessage(), e);
        return false;
    }
    return enqueueMessage(queue, msg, uptimeMillis);
}

//调用sendMessage方法其实最后是调用了enqueueMessage方法
private boolean enqueueMessage(MessageQueue queue, Message msg, long uptimeMillis) {
    //为msg.target赋值为this，也就是把当前的handler作为msg的target属性
    //如果大家还记得Looper的loop()方法会取出每个msg然后执行msg.target.dispatchMessage(msg)去处理消息，其实就是派发给相应的Handler
    msg.target = this;
    if (mAsynchronous) {
        msg.setAsynchronous(true);
    }
    //最终调用queue的enqueueMessage的方法，也就是说handler发出的消息，最终会保存到消息队列中去
    return queue.enqueueMessage(msg, uptimeMillis);
}
```

enqueueMessage()方法中将msg.target赋值为this，也就是把当前的handler作为msg的target属性 //如果大家还记得Looper的loop()方法会取出每个msg然后执行msg.target.dispatchMessage(msg)去处理消息，其实就是派发给相应的Handler

### 2.1 MessageQueue.enqueueMessage() 添加一条消息到消息队列

```java
boolean enqueueMessage(Message msg, long when) {
    // 每一个普通Message必须有一个target
    if (msg.target == null) {
        throw new IllegalArgumentException("Message must have a target.");
    }
    if (msg.isInUse()) {
        throw new IllegalStateException(msg + " This message is already in use.");
    }
    synchronized (this) {
        if (mQuitting) {  //正在退出时，回收msg，加入到消息池
            msg.recycle();
            return false;
        }
        msg.markInUse();
        msg.when = when;
        Message p = mMessages;
        boolean needWake;
        if (p == null || when == 0 || when < p.when) {
            //p为null(代表MessageQueue没有消息） 或者msg的触发时间是队列中最早的， 则进入该该分支
            msg.next = p;
            mMessages = msg;
            needWake = mBlocked; //当阻塞时需要唤醒
        } else {
            //将消息按时间顺序插入到MessageQueue。一般地，不需要唤醒事件队列，除非
            //消息队头存在barrier，并且同时Message是队列中最早的异步消息。
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
        //消息没有退出，我们认为此时mPtr != 0
        if (needWake) {
            nativeWake(mPtr);
        }
    }
    return true;
}
```

MessageQueue是按照Message触发时间的先后顺序排列的，队头的消息是将要最早触发的消息。当有消息需要加入消息队列时，会从队列头开始遍历，直到找到消息应该插入的合适位置，以保证所有消息的时间顺序。

第二种方式：post(Ruunable r)

```java
public final boolean post(Runnable r)
{
   return  sendMessageDelayed(getPostMessage(r), 0);
}
```

其实post()方法最终也会保存到消息队列中去，和上面不同的是它传进来的一个Runnable对象，执行了getPostMessage()方法，我们往下追踪

```java
private static Message getPostMessage(Runnable r) {
    Message m = Message.obtain();
    m.callback = r;
    return m;
}
```

实质上就是将这个Runnable保存在Message的变量中，这就导致了我们下面处理消息的时候有两种不同方案

> **Handler发送消息总结：**

> **1、Handler对象在创建时会保存当前线程的looper和MessageQueue，如果传入Callback的话也会保存起来。**
**2、用户调用handler对象的sendMessage方法，传入msg对象。handler通过调用MessageQueue的enqueueMessage方法将消息压入MessageQueue。**
**3、enqueueMessage方法会将传入的消息对象根据触发时间（when）插入到message queue中。然后判断是否要唤醒等待中的队列。**
**a. 如果插在队列中间。说明该消息不需要马上处理，不需要由这个消息来唤醒队列。**
**b. 如果插在队列头部（或者when=0），则表明要马上处理这个消息。如果当前队列正在堵塞，则需要唤醒它进行处理。**
**4、如果需要唤醒队列，则通过nativeWake方法，往前面提到的管道中写入一个"W"字符，令nativePollOnce方法返回。**

### (4) Handler处理消息

在Looper.loop()中，当发现有消息时，调用消息的目标handler，执行dispatchMessage()方法来分发消息。

```java
public void dispatchMessage(Message msg) {
    if (msg.callback != null) {
        //1\. post()方法的处理方法
        handleCallback(msg);
    } else {
        if (mCallback != null) {
            if (mCallback.handleMessage(msg)) {
                return;
            }
        }
        //2\. sendMessage()方法的处理方法
        handleMessage(msg);
    }
}

//1\. post()方法的最终处理方法
private static void handleCallback(Message message) {
    message.callback.run();
}

//2\. sendMessage()方法的最终处理方法
public void handleMessage(Message msg) {
}
```

> **处理消息总结：**

> **Looper对象的loop方法里面的queue.next方法如果返回了message，那么handler的dispatchMessage会被调用。**
**a. 如果新建Handler的时候传入了callback实例，那么callback的handleMessage方法会被调用。**
**b.如果是通过post方法向handler传入runnable对象的，那么runnable对象的run方法会被调用。**
**c.其他情况下，handler方法的handleMessage会被调用。**

### (五)总结

**（1）简洁总结图示：**

![Markdown](https://raw.githubusercontent.com/zhoujinjianx/PicGo/master/android.message/05-android-handler-looper-message-java.jpg)

图解：

● Handler通过sendMessage()发送Message到MessageQueue队列；
● Looper通过loop()，不断提取出达到触发条件的Message，并将Message交给target来处理；
● 经过dispatchMessage()后，交回给Handler的handleMessage()来进行相应地处理。
● 将Message加入MessageQueue时，处往管道写入字符，可以会唤醒loop线程；如果MessageQueue中没有Message，并处于Idle状态，则会执行IdelHandler接口中的方法，往往用于做一些清理性地工作。

消息分发的优先级：

1、Message的回调方法：message.callback.run()，优先级最高； 2、Handler的回调方法：Handler.mCallback.handleMessage(msg)，优先级仅次于1； 3、Handler的默认方法：Handler.handleMessage(msg)，优先级最低。

**详细总结图示：**

![Markdown](https://raw.githubusercontent.com/zhoujinjianx/PicGo/master/android.message/06-android-handler-looper-message-detail.jpg)

● Looper调用prepare()进行初始化，创建了一个与当前线程对应的Looper对象（通过ThreadLocal实现），并且初始化了一个与当前Looper对应的MessageQueue对象。

● Looper调用静态方法loop()开始消息循环，通过MessageQueue.next()方法获取Message对象。

● 当获取到一个Message对象时，让Message的发送者（target）去处理它。

● Message对象包括数据，发送者（Handler），可执行代码段（Runnable）三个部分组成。

● Handler可以在一个已经Looper.prepare()的线程中初始化，如果线程没有初始化Looper，创建Handler对象会失败

● 一个线程的执行流中可以构造多个Handler对象，它们都往同一个MessageQueue中发消息，消息也只会分发给对应的Handler处理。

● Handler将消息发送到MessageQueue中，Message的target域会引用自己的发送者，Looper从MessageQueue中取出来后，再交给发送这个Message的Handler去处理。

● Message可以直接添加一个Runnable对象，当这条消息被处理的时候，直接执行Runnable.run()方法。

--------------------------------------------------------------------------------

## Ⅱ、Android消息机制(Native层)

在前面讲解了Java层的消息处理机制，其中MessageQueue类里面涉及到多个native方法，除了MessageQueue的native方法，native层本身也有一套完整的消息机制，用于处理native的消息。在整个消息机制中，而MessageQueue是连接Java层和Native层的纽带，换言之，Java层可以向MessageQueue消息队列中添加消息，Native层也可以向MessageQueue消息队列中添加消息。

Native层类的关系图： 
![Markdown](https://raw.githubusercontent.com/zhoujinjianx/PicGo/master/android.message/07-android-handler-looper-message-native-message.png)

### (一) MessageQueue 初始化（Native 层）

接着从Java层MessageQueue初始化开始分析：

### Step 1：MessageQueue()

```java
MessageQueue(boolean quitAllowed) {
    mQuitAllowed = quitAllowed;
    //通过native方法初始化消息队列，其中mPtr是供native代码使用
    mPtr = nativeInit();
}
```

![Markdown](https://raw.githubusercontent.com/zhoujinjianx/PicGo/master/android.message/08-android-handler-looper-message-native_init.png)

### Step 2：android_os_MessageQueue_nativeInit()

```c
==> android_os_MessageQueue.cpp
static jlong android_os_MessageQueue_nativeInit(JNIEnv* env, jclass clazz) {
    NativeMessageQueue* nativeMessageQueue = new NativeMessageQueue(); //初始化native消息队列 【3】
    if (!nativeMessageQueue) {
        jniThrowRuntimeException(env, "Unable to allocate native queue");
        return 0;
    }
    nativeMessageQueue->incStrong(env);
    return reinterpret_cast<jlong>(nativeMessageQueue);
}
```

在nativeInit中，new了一个Native层的MessageQueue的对象

### Step 3：NativeMessageQueue()

```c
==> android_os_MessageQueue.cpp

NativeMessageQueue::NativeMessageQueue() : mPollEnv(NULL), mPollObj(NULL), mExceptionObj(NULL) {
    mLooper = Looper::getForThread(); //获取TLS中的Looper对象
    if (mLooper == NULL) {
        mLooper = new Looper(false); //创建native层的Looper
        Looper::setForThread(mLooper); //保存native层的Looper到TLS中
    }
}
```

在NativeMessageQueue的构造函数中获得了一个Native层的Looper对象，Native层的Looper也使用了线程本地存储，注意new Looper时传入了参数false。

Looper::getForThread()，功能类比于Java层的Looper.myLooper(); Looper::setForThread(mLooper)，功能类比于Java层的ThreadLocal.set();

MessageQueue是在Java层与Native层有着紧密的联系，但是此次Native层的Looper与Java层的Looper没有任何的关系，可以发现native基本等价于用C++重写了Java的Looper逻辑，故可以发现很多功能类似的地方。

### (二) Looper初始化（Native层）

```c
Looper::Looper(bool allowNonCallbacks) :
        mAllowNonCallbacks(allowNonCallbacks), mSendingMessage(false),
        mPolling(false), mEpollFd(-1), mEpollRebuildRequired(false),
        mNextRequestSeq(0), mResponseIndex(0), mNextMessageUptime(LLONG_MAX) {
    mWakeEventFd = eventfd(0, EFD_NONBLOCK); //构造唤醒事件的fd
    AutoMutex _l(mLock);
    rebuildEpollLocked();  //重建Epoll事件
}

void Looper::rebuildEpollLocked() {
    if (mEpollFd >= 0) {
        close(mEpollFd); //关闭旧的epoll实例
    }
    mEpollFd = epoll_create(EPOLL_SIZE_HINT); //创建新的epoll实例，并注册wake管道
    struct epoll_event eventItem;
    memset(& eventItem, 0, sizeof(epoll_event)); //把未使用的数据区域进行置0操作
    eventItem.events = EPOLLIN; //可读事件
    eventItem.data.fd = mWakeEventFd;
    //将唤醒事件(mWakeEventFd)添加到epoll实例(mEpollFd)
    int result = epoll_ctl(mEpollFd, EPOLL_CTL_ADD, mWakeEventFd, & eventItem);

    for (size_t i = 0; i < mRequests.size(); i++) {
        const Request& request = mRequests.valueAt(i);
        struct epoll_event eventItem;
        request.initEventItem(&eventItem);
        //将request队列的事件，分别添加到epoll实例
        int epollResult = epoll_ctl(mEpollFd, EPOLL_CTL_ADD, request.fd, & eventItem);
        if (epollResult < 0) {
            ALOGE("Error adding epoll events for fd %d while rebuilding epoll set, errno=%d", request.fd, errno);
        }
    }
}
```

通过eventfd创建mWakeEventFd用于线程间通信去唤醒Looper的，当需要唤醒Looper时，就往里面写1 创建用于监听epoll_event的mEpollFd，并初始化mEpollFd要监听的epoll_event类型 通过epoll_ctl将mWakeEventFd注册到mEpollFd中，当mWakeEventFd有事件可读则唤醒Looper 如果mRequests不为空的话，说明前面注册了有要监听的fd，则遍历mRequests中的Request，将它初始化为epoll_event并通过epoll_ctl注册到mEpollFd中，当有可读事件同样唤醒Looper

### (三) nativePollOnce()

我们从前面分析知道，Looper.loop()方法被调用后，会启动一个无限循环，而在这个循环中，调用了MessageQueue的next()方法以获取下一条消息，而next()方法中会首先调用nativePollOnce()方法，这个方法的作用在之前说过是阻塞，达到超时时间或有新的消息到达时得到eventFd的通知再唤醒消息队列，其实这个方法也是native消息处理的开始。

nativePollOnce用于提取消息队列中的消息，提取消息的调用链，如下：

![Markdown](https://raw.githubusercontent.com/zhoujinjianx/PicGo/master/android.message/09-android-handler-looper-message-poll_once.png)

下面来进一步来看看调用链的过程：

### Step 1：MessageQueue.next()

```java
==> MessageQueue.java

Message next() {
    final long ptr = mPtr;
    if (ptr == 0) {
        return null;
    }

    for (;;) {
        ...
        nativePollOnce(ptr, nextPollTimeoutMillis); //阻塞操作
        ...
    }
```

### Step 2：android_os_MessageQueue_nativePollOnce()

```c
==> android_os_MessageQueue.cpp

static void android_os_MessageQueue_nativePollOnce(JNIEnv* env, jobject obj, jlong ptr, jint timeoutMillis) {
    //将Java层传递下来的mPtr转换为nativeMessageQueue
    NativeMessageQueue* nativeMessageQueue = reinterpret_cast<NativeMessageQueue*>(ptr);
    nativeMessageQueue->pollOnce(env, obj, timeoutMillis);
}
```

### Step 3：NativeMessageQueue::pollOnce()

```c
==> android_os_MessageQueue.cpp

void NativeMessageQueue::pollOnce(JNIEnv* env, jobject pollObj, int timeoutMillis) {
    mPollEnv = env;
    mPollObj = pollObj;
    mLooper->pollOnce(timeoutMillis);
    mPollObj = NULL;
    mPollEnv = NULL;
    if (mExceptionObj) {
        env->Throw(mExceptionObj);
        env->DeleteLocalRef(mExceptionObj);
        mExceptionObj = NULL;
    }
}
```

### Step 4：Looper::pollOnce()

```c
==> Looper.cpp

int Looper::pollOnce(int timeoutMillis, int* outFd, int* outEvents, void** outData) {
    int result = 0;
    for (;;) {
        // 先处理没有Callback方法的 Response事件
        while (mResponseIndex < mResponses.size()) {
            const Response& response = mResponses.itemAt(mResponseIndex++);
            int ident = response.request.ident;
            if (ident >= 0) { //ident大于0，则表示没有callback, 因为POLL_CALLBACK = -2,
                int fd = response.request.fd;
                int events = response.events;
                void* data = response.request.data;
                if (outFd != NULL) *outFd = fd;
                if (outEvents != NULL) *outEvents = events;
                if (outData != NULL) *outData = data;
                return ident;
            }
        }
        if (result != 0) {
            if (outFd != NULL) *outFd = 0;
            if (outEvents != NULL) *outEvents = 0;
            if (outData != NULL) *outData = NULL;
            return result;
        }
        // 再处理内部轮询
        result = pollInner(timeoutMillis);
    }
}
```

### Step 5 ：Looper::pollInner()：

```c
==> Looper.cpp


void Looper::awoken() {
    uint64_t counter;
    //不断读取管道数据，目的就是为了清空管道内容
    TEMP_FAILURE_RETRY(read(mWakeEventFd, &counter, sizeof(uint64_t)));
}

int Looper::pollInner(int timeoutMillis) {
    ...
    int result = POLL_WAKE;
    mResponses.clear();
    mResponseIndex = 0;
    mPolling = true; //即将处于idle状态
    struct epoll_event eventItems[EPOLL_MAX_EVENTS]; //fd最大个数为16
    //等待事件发生或者超时，在nativeWake()方法，向管道写端写入字符，则该方法会返回；
    int eventCount = epoll_wait(mEpollFd, eventItems, EPOLL_MAX_EVENTS, timeoutMillis);

    mPolling = false; //不再处于idle状态
    mLock.lock();  //请求锁
    if (mEpollRebuildRequired) {
        mEpollRebuildRequired = false;
        rebuildEpollLocked();  // epoll重建，直接跳转Done;
        goto Done;
    }
    if (eventCount < 0) {
        if (errno == EINTR) {
            goto Done;
        }
        result = POLL_ERROR; // epoll事件个数小于0，发生错误，直接跳转Done;
        goto Done;
    }
    if (eventCount == 0) {  //epoll事件个数等于0，发生超时，直接跳转Done;
        result = POLL_TIMEOUT;
        goto Done;
    }

    //循环遍历，处理所有的事件
    for (int i = 0; i < eventCount; i++) {
        int fd = eventItems[i].data.fd;
        uint32_t epollEvents = eventItems[i].events;
        if (fd == mWakeEventFd) {
            if (epollEvents & EPOLLIN) {
                awoken(); //已经唤醒了，则读取并清空管道数据
            }
        } else {
            ssize_t requestIndex = mRequests.indexOfKey(fd);
            if (requestIndex >= 0) {
                int events = 0;
                if (epollEvents & EPOLLIN) events |= EVENT_INPUT;
                if (epollEvents & EPOLLOUT) events |= EVENT_OUTPUT;
                if (epollEvents & EPOLLERR) events |= EVENT_ERROR;
                if (epollEvents & EPOLLHUP) events |= EVENT_HANGUP;
                //处理request，生成对应的reponse对象，push到响应数组
                pushResponse(events, mRequests.valueAt(requestIndex));
            }
        }
    }
Done: ;
    //再处理Native的Message，调用相应回调方法
    mNextMessageUptime = LLONG_MAX;
    while (mMessageEnvelopes.size() != 0) {
        nsecs_t now = systemTime(SYSTEM_TIME_MONOTONIC);
        const MessageEnvelope& messageEnvelope = mMessageEnvelopes.itemAt(0);
        if (messageEnvelope.uptime <= now) {
            {
                sp<MessageHandler> handler = messageEnvelope.handler;
                Message message = messageEnvelope.message;
                mMessageEnvelopes.removeAt(0);
                mSendingMessage = true;
                mLock.unlock();  //释放锁
                handler->handleMessage(message);  // 处理消息事件
            }
            mLock.lock();  //请求锁
            mSendingMessage = false;
            result = POLL_CALLBACK; // 发生回调
        } else {
            mNextMessageUptime = messageEnvelope.uptime;
            break;
        }
    }
    mLock.unlock(); //释放锁

    //处理带有Callback()方法的Response事件，执行Reponse相应的回调方法
    for (size_t i = 0; i < mResponses.size(); i++) {
        Response& response = mResponses.editItemAt(i);
        if (response.request.ident == POLL_CALLBACK) {
            int fd = response.request.fd;
            int events = response.events;
            void* data = response.request.data;
            // 处理请求的回调方法
            int callbackResult = response.request.callback->handleEvent(fd, events, data);
            if (callbackResult == 0) {
                removeFd(fd, response.request.seq); //移除fd
            }
            response.request.callback.clear(); //清除reponse引用的回调方法
            result = POLL_CALLBACK;  // 发生回调
        }
    }
    return result;
}
```

pollInner()方法比较长也是native消息机制的核心，我们拆成几个部分看。

### 5.1 Request 与 Response

```c
int result = POLL_WAKE;
mResponses.clear();
mResponseIndex = 0;
mPolling = true;

struct epoll_event eventItems[EPOLL_MAX_EVENTS];
int eventCount = epoll_wait(mEpollFd, eventItems, EPOLL_MAX_EVENTS, timeoutMillis);// 第7行
mPolling = false;
mLock.lock();
...
for (int i = 0; i < eventCount; i++) {//第11行
    int fd = eventItems[i].data.fd;
    uint32_t epollEvents = eventItems[i].events;
    if (fd == mWakeEventFd) {
        if (epollEvents & EPOLLIN) {
            awoken();
        } else {
            ALOGW("Ignoring unexpected epoll events 0x%x on wake event fd.", epollEvents);
        }
    } else {
        ssize_t requestIndex = mRequests.indexOfKey(fd);
        if (requestIndex >= 0) {
            int events = 0;
            if (epollEvents & EPOLLIN) events |= EVENT_INPUT;
            if (epollEvents & EPOLLOUT) events |= EVENT_OUTPUT;
            if (epollEvents & EPOLLERR) events |= EVENT_ERROR;
            if (epollEvents & EPOLLHUP) events |= EVENT_HANGUP;
            pushResponse(events, mRequests.valueAt(requestIndex));// 第28行
        } else {
            ALOGW("Ignoring unexpected epoll events 0x%x on fd %d that is "
                    "no longer registered.", epollEvents, fd);
        }
    }
}
```

当第7行系统调用epoll_wait()返回时，说明因注册的fd有消息或达到超时，在第11行就对收到的唤醒events进行遍历，首先判断有消息的fd是不是用于唤醒的mWakeEventFd，如果不是的话，说明是系统调用addFd()方法设置的自定义fd（后面会讲）。那么我们需要对这个事件作出响应。

第21到28行就对这个event做处理，首先，我们以这个fd为key从mRequests中找到他的索引，这个mRequests是我们在addFd()方法一并注册的以fd为key，Request为value的映射表。找到request之后，28行调用pushResponse()方法去建立response：

```c
void Looper::pushResponse(int events, const Request& request) {
    Response response;
    response.events = events;
    response.request = request;
    mResponses.push(response);
}
```

现在我们要处理的任务已经被封装成了一个Response对象，等待被处理，那么真正的处理在哪里呢？

在上面的代码与处理response的代码中间夹着的是处理MessageEnvelope的代码，我们后面再讲这段，现在到处理response的代码：

```c
 for (size_t i = 0; i < mResponses.size(); i++) {
    Response& response = mResponses.editItemAt(i);
    if (response.request.ident == POLL_CALLBACK) {
        int fd = response.request.fd;
        int events = response.events;
        void* data = response.request.data;
        int callbackResult = response.request.callback->handleEvent(fd, events, data);
        if (callbackResult == 0) {
            removeFd(fd, response.request.seq);
        }
        response.request.callback.clear();
        result = POLL_CALLBACK;
    }
}
```

遍历所有response对象，取出之前注册的request对象的信息，然后调用了request.callback->handleEvent()方法进行回调，如果该回调返回0，则调用removeFd()方法取消这个fd的注册。

> **再梳理一遍这个过程：注册的自定义fd被消息唤醒，从mRequests中以fd为key找到对应的注册好的request对象然后生成response对象，在MessageEnvelop处理完毕之后处理response，调用request中的callback的handleEvent()方法。**

那么addFd()注册自定义fd与removeFd()取消注册是如何实现的呢？

### 5.2 addFd()

```c
int Looper::addFd(int fd, int ident, int events, const sp<LooperCallback>& callback, void* data) {
...
{ // acquire lock
    AutoMutex _l(mLock);

    Request request;//第6-13行
    request.fd = fd;
    request.ident = ident;
    request.events = events;
    request.seq = mNextRequestSeq++;
    request.callback = callback;
    request.data = data; // 第6-13行 end
    if (mNextRequestSeq == -1) mNextRequestSeq = 0; // reserve sequence number -1

    struct epoll_event eventItem;
    request.initEventItem(&eventItem);

    ssize_t requestIndex = mRequests.indexOfKey(fd);
    if (requestIndex < 0) { // 第19行
        int epollResult = epoll_ctl(mEpollFd, EPOLL_CTL_ADD, fd, & eventItem); //第20行
        if (epollResult < 0) {
            ALOGE("Error adding epoll events for fd %d: %s", fd, strerror(errno));
            return -1;
        }
        mRequests.add(fd, request); //第25行
    } else {
        int epollResult = epoll_ctl(mEpollFd, EPOLL_CTL_MOD, fd, & eventItem); // 第27行
        if (epollResult < 0) {
            if (errno == ENOENT) {
                epollResult = epoll_ctl(mEpollFd, EPOLL_CTL_ADD, fd, & eventItem);
                if (epollResult < 0) {
                    ALOGE("Error modifying or adding epoll events for fd %d: %s",
                            fd, strerror(errno));
                    return -1;
                }
                scheduleEpollRebuildLocked();
            } else {
                ALOGE("Error modifying epoll events for fd %d: %s", fd, strerror(errno));
                return -1;
            }
        }
        mRequests.replaceValueAt(requestIndex, request);
    }
} // release lock
return 1;
}
```

第6-13行使用传入的参数初始化了request对象，然后16行由request来初始化注册epoll使用的event。19行根据mRequests.indexOfKey()方法取出的值来判断fd是否已经注册，如果未注册，则在20行进行系统调用epoll_ctl()注册新监听并在25行将fd与request存入mRequest，如果已注册，则在27行更新注册并在42行更新request。

这就是自定义fd设置的过程：保存request并使用epoll_ctl系统调用注册fd的监听。

### 5.3 removeFd()

```c
int Looper::removeFd(int fd, int seq) {
    { // acquire lock
        AutoMutex _l(mLock);
        ssize_t requestIndex = mRequests.indexOfKey(fd);
        if (requestIndex < 0) {
            return 0;
        }
        if (seq != -1 && mRequests.valueAt(requestIndex).seq != seq) {
            return 0;
        }
        mRequests.removeItemsAt(requestIndex);

        int epollResult = epoll_ctl(mEpollFd, EPOLL_CTL_DEL, fd, NULL);
        if (epollResult < 0) {
            if (seq != -1 && (errno == EBADF || errno == ENOENT)) {
                scheduleEpollRebuildLocked();
            } else {
                ALOGE("Error removing epoll events for fd %d: %s", fd, strerror(errno));
                scheduleEpollRebuildLocked();
                return -1;
            }
        }
    } // release lock
    return 1;
}
```

解除的过程相反，在第11行删除mRequests中的键值对，然后在第13行系统调用epoll_ctl()解除fd的epoll注册。

### MessageEnvelope消息处理

之前说到，在request生成response到response的处理中间有一段代码执行了MessageEnvelop消息的处理，这个顺序保证了MessageEnvelop优先于fd引起的request的处理。

现在我们来看这段代码：

```c
mNextMessageUptime = LLONG_MAX;
while (mMessageEnvelopes.size() != 0) {
    nsecs_t now = systemTime(SYSTEM_TIME_MONOTONIC);
    const MessageEnvelope& messageEnvelope = mMessageEnvelopes.itemAt(0); // 第4行
    if (messageEnvelope.uptime <= now) {
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
        mNextMessageUptime = messageEnvelope.uptime;
        break;
    }
}
```

可以看到mMessageEnvelopes容器中存储了所有的消息，第4行从首位置取出一条消息，随后进行时间判断，如果时间到达，先移出容器，与java层比较相似都是调用了handler的handleMessage()来进行消息的处理。

那么MessageEnvelope是如何添加的呢？

Native Looper提供了一套与java层MessageQueue类似的方法，用于添加MessageEnvelope：

```c
void Looper::sendMessageAtTime(nsecs_t uptime, const sp<MessageHandler>& handler,
        const Message& message) {
    size_t i = 0;
    { // acquire lock
        AutoMutex _l(mLock);

        size_t messageCount = mMessageEnvelopes.size();
        while (i < messageCount && uptime >= mMessageEnvelopes.itemAt(i).uptime) {
            i += 1;
        }

        MessageEnvelope messageEnvelope(uptime, handler, message);
        mMessageEnvelopes.insertAt(messageEnvelope, i, 1);

        if (mSendingMessage) {
            return;
        }
    } // release lock
    if (i == 0) {
        wake();
    }
}
```

小结: 现在我们看到，其实Native中的消息机制有两个方面，一方面是通过addFd()注册的自定义fd触发消息处理，通过mRequests保存的request对象中的callback进行消息处理。另一方面是通过与java层类似的MessageEnvelop消息对象进行处理，调用的是该对象handler域的handleMessage()方法，与java层非常类似。优先级是先处理MessageEnvelop再处理request。

一些思考 现在消息机制全部内容分析下来，我们可以看到android的消息机制不算复杂，分为native与java两个部分，这两个部分分别有自己的消息处理机制，其中关键的超时与唤醒部分是借助了linux系统epoll机制来实现的。

连接java与native层消息处理过程的是next()方法中的nativePollOnce()，java层消息循环先调用它，自身阻塞，进入native的消息处理，在native消息处理完毕后返回，再进行java层的消息处理，正是因为如此，如果我们在处理java层消息的时候执行了耗时或阻塞的任务（甚至阻塞了整个主线程），整个java层的消息循环就会阻塞，也无法进一步进入native层的消息处理，也就无法响应例如触摸事件这样的消息，导致ANR的发生。这也就是我们不应在主线程中执行这类任务的原因。

### (四) 唤醒 nativeWake()

在添加消息到消息队列enqueueMessage(), 或者把消息从消息队列中全部移除quit()，再有需要时都会调用 nativeWake方法。包含唤醒过程的添加消息的调用链，nativeWake用于唤醒功能，如下： 
![Markdown](https://raw.githubusercontent.com/zhoujinjianx/PicGo/master/android.message/10-android-handler-looper-message-native_wake.png)

下面来进一步来看看调用链的过程：

### Step 1 ：MessageQueue.enqueueMessage()

```java
==> MessageQueue.java

boolean enqueueMessage(Message msg, long when) {
    ... //将Message按时间顺序插入MessageQueue
    if (needWake) {
            nativeWake(mPtr);
        }
}
```

往消息队列添加Message时，需要根据mBlocked情况来决定是否需要调用nativeWake。

### Step 2 ：android_os_MessageQueue_nativeWake()

```c
==> android_os_MessageQueue.cpp

static void android_os_MessageQueue_nativeWake(JNIEnv* env, jclass clazz, jlong ptr) {
    NativeMessageQueue* nativeMessageQueue = reinterpret_cast<NativeMessageQueue*>(ptr);
    nativeMessageQueue->wake();
}
```

### Step 3 ：NativeMessageQueue::wake()

```c
==> android_os_MessageQueue.cpp

void NativeMessageQueue::wake() {
    mLooper->wake();
}
```

### Step 4 ：Looper::wake()

```c
==> Looper.cpp

void Looper::wake() {
    uint64_t inc = 1;
    // 向管道mWakeEventFd写入字符1
    ssize_t nWrite = TEMP_FAILURE_RETRY(write(mWakeEventFd, &inc, sizeof(uint64_t)));
    if (nWrite != sizeof(uint64_t)) {
        if (errno != EAGAIN) {
            ALOGW("Could not write wake signal, errno=%d", errno);
        }
    }
}
其中TEMP_FAILURE_RETRY 是一个宏定义， 当执行write失败后，会不断重复执行，直到执行成功为止。
```

### (五) 发送消息sendMessage（Native层）

在前面Android消息机制(Java层)文中，讲述了Java层如何向MessageQueue类中添加消息，那么接下来讲讲Native层如何向MessageQueue发送消息。

### Step 1 ：sendMessage()

```c
void Looper::sendMessage(const sp<MessageHandler>& handler, const Message& message) {
    nsecs_t now = systemTime(SYSTEM_TIME_MONOTONIC);
    sendMessageAtTime(now, handler, message);
}
```

### Step 2 ：sendMessageDelayed()

```c
void Looper::sendMessageDelayed(nsecs_t uptimeDelay, const sp<MessageHandler>& handler,
        const Message& message) {
    nsecs_t now = systemTime(SYSTEM_TIME_MONOTONIC);
    sendMessageAtTime(now + uptimeDelay, handler, message);
}
```

sendMessage(),sendMessageDelayed() 都是调用sendMessageAtTime()来完成消息插入。

### Step 3 ：sendMessageAtTime()

```c
void Looper::sendMessageAtTime(nsecs_t uptime, const sp<MessageHandler>& handler,
        const Message& message) {
    size_t i = 0;
    { //请求锁
        AutoMutex _l(mLock);
        size_t messageCount = mMessageEnvelopes.size();
        //找到message应该插入的位置i
        while (i < messageCount && uptime >= mMessageEnvelopes.itemAt(i).uptime) {
            i += 1;
        }
        MessageEnvelope messageEnvelope(uptime, handler, message);
        mMessageEnvelopes.insertAt(messageEnvelope, i, 1);
        //如果当前正在发送消息，那么不再调用wake()，直接返回。
        if (mSendingMessage) {
            return;
        }
    } //释放锁
    //当把消息加入到消息队列的头部时，需要唤醒poll循环。
    if (i == 0) {
        wake();
    }
}
```

### (六) 处理消息MessageHandler.handleMessage() && LooperCallback.handleEvent()（Native层）

其实Native中的消息机制有两个方面，一方面是通过addFd()注册的自定义fd触发消息处理，通过mRequests保存的request对象中的callback进行消息处理。即调用LooperCallback的handleEvent()处理

另一方面是通过与java层类似的MessageEnvelop消息对象进行处理，调用的是该对象handler域的handleMessage()方法，与java层非常类似。优先级是先处理MessageEnvelop再处理request。即调用MessageHandler类的handleMessage()处理

6.1 MessageHandler类：调用MessageHandler类的handleMessage()处理消息

```c
class MessageHandler : public virtual RefBase {
protected:
    virtual ~MessageHandler() { }
public:
    virtual void handleMessage(const Message& message) = 0;
};
```

6.2 LooperCallback.handleEvent)：用于处理指定的文件描述符的poll事件

```c
LooperCallback类
class LooperCallback : public virtual RefBase {
protected:
virtual ~LooperCallback() { }
public:
//用于处理指定的文件描述符的poll事件
virtual int handleEvent(int fd, int events, void* data) = 0;
};
```

### (七) Native消息机制使用实例：SurfaceFlinger Native消息处理

在 【Android 7.1.2(Android N) Activity-Window加载显示流程】中讲到 App请求创建Surface创建过程中，SurfaceFlinger会处理Native消息，此处便是Native消息机制使用的一个具体实例。

```c
 status_t Client::createSurface(
        const String8& name,
        uint32_t w, uint32_t h, PixelFormat format, uint32_t flags,
        sp<IBinder>* handle,
        sp<IGraphicBufferProducer>* gbp){
    /*
     * createSurface must be called from the GL thread so that it can
     * have access to the GL context.
     */

    class MessageCreateLayer : public MessageBase {
        SurfaceFlinger* flinger;
        Client* client;
        sp<IBinder>* handle;
        sp<IGraphicBufferProducer>* gbp;
        status_t result;
        const String8& name;
        uint32_t w, h;
        PixelFormat format;
        uint32_t flags;
    public:
        MessageCreateLayer(SurfaceFlinger* flinger,
                const String8& name, Client* client,
                uint32_t w, uint32_t h, PixelFormat format, uint32_t flags,
                sp<IBinder>* handle,
                sp<IGraphicBufferProducer>* gbp)
            : flinger(flinger), client(client),
              handle(handle), gbp(gbp), result(NO_ERROR),
              name(name), w(w), h(h), format(format), flags(flags) {
        }
        status_t getResult() const { return result; }
        virtual bool handler() {
            result = flinger->createLayer(name, client, w, h, format, flags,
                    handle, gbp);
            return true;
        }
    };

    sp<MessageBase> msg = new MessageCreateLayer(mFlinger.get(),
            name, this, w, h, format, flags, handle, gbp);
    mFlinger->postMessageSync(msg);
    return static_cast<MessageCreateLayer*>( msg.get() )->getResult();
    }
```

Client将应用程序创建Surface的请求转换为异步消息投递到SurfaceFlinger的消息队列中，将创建Surface的任务转交给SurfaceFlinger。 函数首先将请求创建的Surface参数封装为MessageCreateSurface对象，然后调用SurfaceFlinger的postMessageSync函数往SurfaceFlinger的消息队列中发送一个同步消息，当消息处理完后，通过调用消息msg的getResult()函数来得到创建的Surface。

```c
status_t SurfaceFlinger::postMessageSync(const sp<MessageBase>& msg,  
    nsecs_t reltime, uint32_t flags) {  
//往消息队列中发送一个消息  
status_t res = mEventQueue.postMessage(msg, reltime);  
//消息发送成功后，当前线程等待消息处理  
if (res == NO_ERROR) {  
    msg->wait();  
}  
return res;  
}  

status_t MessageQueue::postMessage(
    const sp<MessageBase>& messageHandler, nsecs_t relTime)
    {
const Message dummyMessage;
//将messageHandler对象和dummyMessage消息对象发送到消息循环Looper对象中
if (relTime > 0) {
    mLooper->sendMessageDelayed(relTime, messageHandler, dummyMessage);
} else {
    mLooper->sendMessage(messageHandler, dummyMessage);
}
return NO_ERROR;
}
```

关于消息循环Looper对象的消息发送函数sendMessage的调用流程请看前面讲解。 这里再次贴上关于消息插入代码：

```c
void Looper::sendMessageAtTime(nsecs_t uptime, const sp<MessageHandler>& handler,  
    const Message& message) {  
size_t i = 0;  
{ // acquire lock  
    AutoMutex _l(mLock);  
    //获取消息队列中保存的消息个数  
    size_t messageCount = mMessageEnvelopes.size();  
    //按时间排序，查找当前消息应该插入的位置  
    while (i < messageCount && uptime >= mMessageEnvelopes.itemAt(i).uptime) {  
        i += 1;  
    }  
    //将Message消息及消息处理Handler封装为MessageEnvelope对象，并插入到消息队列mMessageEnvelopes中  
    MessageEnvelope messageEnvelope(uptime, handler, message);  
    mMessageEnvelopes.insertAt(messageEnvelope, i, 1);  
    if (mSendingMessage) {  
        return;  
    }  
} // release lock  
//唤醒消息循环线程以及时处理消息  
if (i == 0) {  
    wake();  
}  
}
```

到此消息发送就完成了，由于发送的是一个同步消息，因此消息发送线程此刻进入睡眠等待状态，而消息循环线程被唤醒起来处理消息，消息处理过程如下：

```c
//所有C++层的消息都封装为MessageEnvelope类型的变量并保存到mMessageEnvelopes链表中  
while (mMessageEnvelopes.size() != 0) {  
    nsecs_t now = systemTime(SYSTEM_TIME_MONOTONIC);  
    const MessageEnvelope& messageEnvelope = mMessageEnvelopes.itemAt(0);  
    //处理当前时刻之前的所有消息  
    if (messageEnvelope.uptime <= now) {  
        {   
            //取出处理该消息的Hanlder  
            sp<MessageHandler> handler = messageEnvelope.handler;  
            //取出该消息描述符  
            Message message = messageEnvelope.message;  
            //从mMessageEnvelopes链表中移除该消息  
            mMessageEnvelopes.removeAt(0);  
            //表示当前消息循环线程正在处理消息，处于唤醒状态，因此消息发送线程无需唤醒消息循环线程  
            mSendingMessage = true;  
            mLock.unlock();  
            //调用该消息Handler对象的handleMessage函数来处理该消息  
            handler->handleMessage(message);  
        } // release handler  
        mLock.lock();  
        mSendingMessage = false;  
        result = ALOOPER_POLL_CALLBACK;  
    } else {  
        // The last message left at the head of the queue determines the next wakeup time.  
        mNextMessageUptime = messageEnvelope.uptime;  
        break;  
    }  
}
```

消息处理过程就是调用该消息的Handler对象的handleMessage函数来完成，由于创建Surface时，往消息队列中发送的Handler对象类型为MessageCreateSurface，因此必定会调用该类的handleMessage函数来处理Surface创建消息。但该类并未实现 handleMessage函数，同时该类继承于MessageBase，由此可见其父类MessageBase必定实现了handleMessage函数：

```c
void MessageBase::handleMessage(const Message&) {  
    this->handler();  
    barrier.open();  
};
```

该函数首先调用其子类的handler()函数处理消息，然后唤醒消息发送线程，表明发往消息队列中的消息已得到处理，消息发送线程可以往下执行了。由于MessageCreateSurface是MessageBase的子类，因此该类必定实现了handler()函数来处理Surface创建消息：

``` c
class MessageCreateSurface : public MessageBase {  
    sp<ISurface> result;  
    SurfaceFlinger* flinger;  
    ISurfaceComposerClient::surface_data_t* params;  
    Client* client;  
    const String8& name;  
    DisplayID display;  
    uint32_t w, h;  
    PixelFormat format;  
    uint32_t flags;  
public:  
    MessageCreateSurface(SurfaceFlinger* flinger,  
            ISurfaceComposerClient::surface_data_t* params,  
            const String8& name, Client* client,  
            DisplayID display, uint32_t w, uint32_t h, PixelFormat format,  
            uint32_t flags)  
        : flinger(flinger), params(params), client(client), name(name),  
          display(display), w(w), h(h), format(format), flags(flags)  
    {  
    }
    sp<ISurface> getResult() const { return result; }  

    virtual bool handler() {  
        result = flinger->createSurface(params, name, client,display, w, h, format, flags);  
        return true;  
    }  
};
```

这里又调用SurfaceFlinger的createSurface函数来创建Surface。绕了一圈又回到SurfaceFlinger，为什么要这么做呢？因为在同一时刻可以有多个应用程序请求SurfaceFlinger为其创建Surface，通过消息队列可以实现请求排队，然后SurfaceFlinger依次为应用程序创建Surface。
 **图解：** 
![Markdown](https://raw.githubusercontent.com/zhoujinjianx/PicGo/master/android.message/11-android-handler-looper-message-java-c.png)

红色虚线关系：Java层和Native层的MessageQueue通过JNI建立关联，彼此之间能相互调用，搞明白这个互调关系，也就搞明白了Java如何调用C++代码，c代码又是如何调用Java代码。 蓝色虚线关系：Handler/Looper/Message这三大类Java层与Native层并没有任何的真正关联，只是分别在Java层和Native层的handler消息模型中具有相似的功能。都是彼此独立的，各自实现相应的逻辑。 WeakMessageHandler继承于MessageHandler类，NativeMessageQueue继承于MessageQueue类 另外，消息处理流程是先处理Native Message，再处理Native Request，最后处理Java Message。理解了该流程，也就明白有时上层消息很少，但响应时间却较长的真正原因。

## 总结：

![Markdown](https://raw.githubusercontent.com/zhoujinjianx/PicGo/master/android.message/12-android-handler-looper-message-structure.png)

## 参考文档（特别感谢）：

[Android消息处理机制(Handler、Looper、MessageQueue与Message)](http://www.cnblogs.com/angeldevil/p/3340644.html)
[android的消息处理机制（图+源码分析）----Looper,Handler,Message](http://www.cnblogs.com/codingmyworld/archive/2011/09/12/2174255.html)
[Android进阶----Android消息机制之Looper、Handler、MessageQueen](http://blog.csdn.net/qq_30379689/article/details/53394061)
[Handler、Looper、Message、MessageQueue 基础流程分析图解](https://www.diycode.cc/topics/671)
[Android 中线程间通信原理分析：Looper, MessageQueue, Handler](https://segmentfault.com/a/1190000003063859) 
[Android消息机制1-Handler(Java层)](http://gityuan.com/2015/12/26/handler-message-framework/)
[Android消息机制2-Handler(Native层)](http://gityuan.com/2015/12/27/handler-message-native/)
[ANDROID消息机制，从JAVA层到NATIVE层剖析](http://www.cheelok.com/aosp/196) [Android应用程序消息处理机制](https://segmentfault.com/a/1190000002982318)
[Android 消息机制（三）Native层消息机制](http://www.hustmeituan.club/android-xiao-xi-ji-zhi-san-nativeceng-xiao-xi-ji-zhi.html)
