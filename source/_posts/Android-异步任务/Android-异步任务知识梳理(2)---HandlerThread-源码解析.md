---
title: Android 异步任务知识梳理(2) - HandlerThread 源码解析
date: 2017-02-20 21:59
categories : Android 异步任务知识梳理
---
#一、概述
刚开始接触`HandlerThread`是在看`AsyncQueryHandler`源码的时候，第一次眼看到`HandlerThread`这个名字，就在想这到底是个`Handler`还是个`Thread`，后来看了源码才发现它其实就是一个`Thread`，只不过可以通过它的`getLooper`返回的`Looper`作为`Handler`构造函数的参数，那么这个新建`Handler`所有`message`的执行都是在这个对应的`Thread`当中的，关于在项目当中如何用好`HandlerThread`，`AsyncQueryHandler`无疑是一个最好的例子。

# 二、源码
`HandlerThread`的代码很少，我们一点点看：
```
int mPriority; //线程的优先级
int mTid = -1; //线程对应的pid，结束时为-1。
Looper mLooper; //对应的looper
```
再看一下`run`方法：
```
<!-- HandlerThread.java -->
@Override
public  void run() {    
    mTid = Process.myTid();    
    Looper.prepare();    
    synchronized (this) {        
        mLooper = Looper.myLooper();         
        notifyAll();    
    }     
    Process.setThreadPriority(mPriority);     
    onLooperPrepared();    
    Looper.loop();    
    mTid = -1;
}
```
第一步先调用了`Looper.prepare()`方法来初始该线程的`Looper`，它其实是新建了一个`Looper`（这个 `Looper`是可以退出循环的），保存到`sThreadLocal`变量当中，这个变量是`static final ThreadLocal<Looper>`类型的，也就是每个`Thread`有一份不同的实例，并且在每个线程中也只允许初始化一次：
```
<!-- Looper.java -->

private Looper(boolean quitAllowed) {    
    mQueue = new MessageQueue(quitAllowed);    
    mThread = Thread.currentThread();
}

public static void prepare() {    
     prepare(true);
}

private static void prepare(boolean quitAllowed) {    
    if (sThreadLocal.get() != null) {        
        throw new RuntimeException("Only one Looper may be created per thread");    
    }    
    sThreadLocal.set(new Looper(quitAllowed));
}
```
之后拿出这个新建的`Looper`保存起来，同时设置线程的优先级，接着调用`Looper.loop()`方法，在这之前会回调给`onLooperPrepared`，因为如果`Looper.loop()`开始执行后，那么就是一个无限循环，用户没法进行初始化操作了：
```
<!-- Looper.java -->
public static void loop() {
     final Looper me = myLooper();
     if (me == null) {
         //exception;
     }
     final MessageQueue queue = me.mQueue;
     Binder.clearCallingIdentity();
     final long ident = Binder.clearCallingIdentity();
     for (;;) {
        //这里面会调用 nativePollOnce(ptr, nextPollTimeoutMillis) 等待有新的消息出现，否则就一直阻塞在这里;
        Message msg = queue.next();
        //如果新的消息为null，那么就退出这个循环。
        if (message == null) {
            return;
        }
        //分发有效的消息。
        msg.target.dispatchMessage(msg);
        msg.recycleUnchecked();
     }
}

<!-- MessageQueue.java -->
Message next() {
   final long ptr = mPtr;
   if (ptr == null) {
      return null;
   }
   for (;;) {
       //会休眠在这里，直到通过mPtr被唤醒。 
       nativePollOnce(ptr, nextPollTimeoutMillis);
       //被唤醒后，从队列中去除消息进行处理。
   }
}
```
接下来看一下`getLooper()`方法，如果此时`mLooper`还没有被初始化，那么它会一直休眠在这里：
```
<!-- HandlerThread.java -->
public Looper getLooper() {
    //如果线程没有存活，那么退出。    
    if (!isAlive()) {        
        return null;    
    }          
    synchronized (this) {
        //如果线程存活，但是没有开始运行，那么等待。        
        while (isAlive() && mLooper == null) {             
            try {                
                wait();            
            } catch (InterruptedException e) {}         
        }    
     }    
     return mLooper;
}
```
这下来看一下退出的两个方法，它们的区别就是`safe`方法会保证当前队列当中的`Message`都被执行完毕，但那些`delay`的消息不会被递送。
```
<!-- HandlerThread.java -->
public boolean quit() {    
    Looper looper = getLooper();    
    if (looper != null) {        
        looper.quit();        
        return true;    
    }    
    return false;
}

public boolean quitSafely() {    
    Looper looper = getLooper();    
    if (looper != null) {        
        looper.quitSafely();        
        return true;    
    }    
    return false;
}
```
`quit`其实是调用了和`Looper`关联的`MessageQueue`对应的`quit(boolean safe)`方法：
```
<!-- Looper.java -->
public void quit() {    
    mQueue.quit(false);
}

public void quitSafely() {      
    mQueue.quit(true);
}

<!-- MessageQueue.java -->
void quit(boolean safe) {
    if (!mQuitAllowed) {    
        throw new IllegalStateException("Main thread not allowed to quit.");
    }
    
    synchronized (this) {    
        if (mQuitting) {        
            return;    
        }    
        mQuitting = true;    
        if (safe) {        
            removeAllFutureMessagesLocked();     
        } else {        
            removeAllMessagesLocked();    
        }
        //唤醒循环，使得 loop 知道，mQuitting 的变化    
        nativeWake(mPtr);
    }
}
```
可以看到这里把`mQuitting`置为了`true`，之后通过`nativeWake`使`Message.next()`中 原先阻塞的 `nativePollOnce`继续往下执行并且会返回`null`，当`Looper.loop()`中的`queue.next()`返回`null` ，那么 `Looper.loop()`也跟着退出了。
由此可见当我们实例化一个`HandlerThread`对象，之后又调用了`quitXXX`方法，那么它就停止了这个循环，相当于这个`Looper`对象在这个线程中虽然存在，但是无法通过它做任何操作了，即使我们第二次执行它的`run`方法，调用`Looper.prepare()`的时候，也会因为`sThreadLocal.get() != null`而抛出异常。
# 三、结论
可以看到，`HandlerThread`就是创建了一个线程，当我们调用它的`start()`方法时，也就是执行它的`run`方法，其实就是创建一个与他相关联的`Looper`，并让这个`Looper`跑起来，这个`Looper`中所有消息的处理都是在这个新建的线程当中的，调用者可以通过这个`Looper`进行后续的操作。
