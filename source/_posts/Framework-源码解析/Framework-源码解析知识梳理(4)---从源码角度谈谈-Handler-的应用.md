---
title: Framework 源码解析知识梳理(4) - 从源码角度谈谈 Handler 的应用
date: 2017-05-18 23:24
categories : Framework 源码解析知识梳理
---
# 一、前言
其实，这篇文章本来应该叫做"`Handler`源码解析"的，但是想想写`Handler`源码的文章太多了，还是缓一缓，先写些不一样的，今天，我们从源码的角度来总结一下应用`Handler`的一些场景的原理。

# 二、Handler 的应用
## 2.1 线程间的通信
在平时的开发当中，`Handler`最常见的用法就是用于线程之间的通信，特别是当我们在子线程中去处理耗时的任务，当任务完成之后，我们希望将结果发送到主线程中进行处理，那么就会使用到`Handler`，基本的思想如下图所示：
![](http://upload-images.jianshu.io/upload_images/1949836-6ec25b13b993215b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
当我们在子线程中通过`Handler`放入在主线程中的循环队列时，那么主线程就会收到消息，之后，我们就可以在主线程中进行后续的处理，例如下面这样：
```
public class MainActivity extends AppCompatActivity {

    private Handler mHandler = new Handler() {
        @Override
        public void handleMessage(Message msg) {
            Log.d("Handler", "HandleMessageThreadId=" + Thread.currentThread().getId());
        }
    };

    private void startThread() {
        new Thread() {
            @Override
            public void run() {
                super.run();
                Log.d("Handler", "ThreadId=" + Thread.currentThread().getId());
                mHandler.sendEmptyMessage(0);
            }
        }.start();
    }

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        Log.d("Handler", "MainThreadId=" + Thread.currentThread().getId());
        startThread();
    }
    
}
```
打印出上面的线程`Id`，可以看到最后`handleMessage`收到消息的时候是在主线程当中：
![](http://upload-images.jianshu.io/upload_images/1949836-58b0e938232c759a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
## 2.2 实现延时操作
除了线程间的通信之外，我们还可以通过`Handler`提供的`sendXXXDelay`方法，实现延时操作，延时操作有两种目的：
- 一种是希望让不那么紧急的任务延后执行，例如在应用启动过程中，我们在`onCreate`方法中的任务不是那个紧急，那么可以通过`Handler`发送一个延时消息出去，让它不占用主线程去渲染布局的资源，从而提高应用的启动速度。
- 另一种就是防抖动操作，我们收到一个命令，并不是马上执行它，而是通过`Handler`的`sendxxxDelay`方法延时，如果在这段事件内又有一个相同的命令来到了，那么就把之前的消息移除，再放入一个新的延时消息。
例如下面的例子，当我们点击一个按钮之后，我们不立刻执行任务，而是一段时间之后仍然没有收到第二次点击事件采取执行任务：
```
    private Handler mDelayHandler = new Handler() {

        @Override
        public void handleMessage(Message msg) {
            super.handleMessage(msg);
            Log.d("Handler", "handleMessage");
        }

    };

    private void performClickDelay() {
        Log.d("Handler", "performClickDelay");
        mDelayHandler.removeMessages(1);
        mDelayHandler.sendEmptyMessageDelayed(1, 500);
    }

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        mTv = (TextView) findViewById(R.id.tv);
        mTv.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                performClickDelay();
            }
        });
    }
```
我们连续多次点击按钮之后，只收到了一次消息：
![](http://upload-images.jianshu.io/upload_images/1949836-830a5e2079fc9a47.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 2.3 使用`HandlerThread`在异步线程执行耗时操作
由于`Looper`其实是线程的一个私有变量（`ThreadLocal`），主线程可以有`Looper`，子线程同样也可以有`Looper`，只不过从子线程中的`Looper`中的`MessageQueue`中取出消息之后，是在子线程当中处理的，那么我们就可以通过它来执行异步操作，其基本思想如下图所示：
![](http://upload-images.jianshu.io/upload_images/1949836-dd57fa71a78fb077.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
`HandlerThread`就是基于这一思想来实现的，关于`HandlerThread`的内部实现，可以参考之前的这篇文章 [Android 异步任务知识梳理(2) - HandlerThread 源码解析](http://www.jianshu.com/p/3d0ebbde4ffe)，它的本质就是当异步线程启动之后，会初始化这个线程私有的`Looper`，因此，当我们通过`HandlerThread`中的`getLooper()`方法获得这个`Looper`之后，在通过这个`Looper`来创建一个`Handler`，那么这个`Handler`的`handleMessage`回调时所在的线程就是这个异步线程。

下面是一个简单的事例：
```
    private void useHandlerThread() {
        final HandlerThread handlerThread = new HandlerThread("handlerThread");
        System.out.println("MainThreadId=" + Thread.currentThread().getId() + ",handlerThreadId=" + handlerThread.getId());
        handlerThread.start();
        MyHandler handler = new MyHandler(handlerThread.getLooper());
        handler.sendEmptyMessage(0);
    }

    private class MyHandler extends Handler {

        MyHandler(Looper looper) {
            super(looper);
        }

        @Override
        public void handleMessage(Message msg) {
            super.handleMessage(msg);
            System.out.println("handleMessageThreadId=" + Thread.currentThread().getId());
        }
    }
```

我们打印出`handleMessage`的执行时的线程`ID`，可以看到它就是`HandlerThread`的线程`ID`，因此我们就可以通过发送消息的方式，在异步线程执行一些耗时的操作：
![](http://upload-images.jianshu.io/upload_images/1949836-788285a62101b5b1.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 2.4 使用`AsyncQueryHandler`对`ContentProvider`进行操作
如果你的本地数据使用`ContentProvider`封装的，那么对这些数据的增删改查操作，为了不影响主线程，应该在子线程中进行操作，而`AsyncQueryHandler`就是系统为我们提供的很方便的工具类，它通过`Handler + HandlerThread`的方式，实现了异步的增删改查。

它是在`HandlerThread`的基础之上扩展出来的，其基本思想如下图所示，这一框架就保证了发起命令和接收回调是在同一个线程，而任务的执行则是在另一个线程：
![](http://upload-images.jianshu.io/upload_images/1949836-f8fde24555131bb1.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
具体的实现原理可以参考这篇文章 [Android 异步任务知识梳理(3) - AsyncQueryHandler 源码解析](http://www.jianshu.com/p/deaf243cdfcc)。

当然`AsyncQueryHandler`限制了只能操作通过`ContentProvider`封装的数据，我们可以参考它的思想，进行扩展，实现对于数据库的增删改查。

## 2.5 使用`Handler`机制检测应用中的卡顿问题
对于每个应用程序来说，它的入口函数为`ActivityThread`中的`main()`方法：
```
   public static void main(String[] args) {
        //...
        Looper.prepareMainLooper();
        //...
        Looper.loop();
        throw new RuntimeException("Main thread loop unexpectedly exited");
    }
```
而在`main()`函数的最后，则构建一个在主线程中的循环队列，之后应用程序收到的事件之后，还主线程就会被唤醒，进行事件的处理，假如这一处理的事件过长，那么就会发生`ANR`，因此，我们就可以通过计算主线程中对于单次消息的处理时间，从而间隔地判断是否存在卡顿问题。
那么要怎么知道主线程中单次消息的处理时间呢，我们查看`Looper`的源码：
```
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
            if (traceTag != 0 && Trace.isTagEnabled(traceTag)) {
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
对于消息的处理对应这句：
```
msg.target.dispatchMessage(msg);
```
而在这句话的前后，则调用了`Printer`类打印，那么我们就可以通过这两次打印的之间的时长来获得单次消息的处理时间。
```
public class MainLoopMonitor {

    private static final String MSG_START = ">>>>> Dispatching";
    private static final String MSG_END = "<<<<< Finished";
    private static final int TIME = 1000;
    private Handler mExecuteHandler;
    private Runnable mExecuteRunnable;

    private static class Holder {
        private static final MainLoopMonitor INSTANCE = new MainLoopMonitor();
    }

    public static MainLoopMonitor getInstance() {
        return Holder.INSTANCE;
    }

    private MainLoopMonitor() {
        HandlerThread monitorThread = new HandlerThread("LooperMonitor");
        monitorThread.start();
        mExecuteHandler = new Handler(monitorThread.getLooper());
        mExecuteRunnable = new ExecutorRunnable();
    }

    public void startMonitor() {
        Looper.getMainLooper().setMessageLogging(new Printer() {
            @Override
            public void println(String x) {
                if (x.startsWith(MSG_START)) {
                    mExecuteHandler.postDelayed(mExecuteRunnable, TIME);
                } else if (x.startsWith(MSG_END)) {
                    mExecuteHandler.removeCallbacks(mExecuteRunnable);
                }
            }
        });
    }
    
    private class ExecutorRunnable implements Runnable {

        @Override
        public void run() {
            StringBuilder sb = new StringBuilder();
            StackTraceElement[] stackTrace = Looper.getMainLooper().getThread().getStackTrace();
            for (StackTraceElement s : stackTrace) {
                sb.append(s).append("\n");
            }
            System.out.println("MainLooperMonitor:" + sb.toString());
        }
    }
}
```
假如我们像下面这样，在主线程中进行了耗时的操作：
```
public void anrButton(View view) {
        for (int i = 0; i < (1 << 30); i++) {
            int b = 6;
        }
}
```
那么就会打印出下面的堆栈信息：
![](http://upload-images.jianshu.io/upload_images/1949836-a0d68852b7e81b8d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
# 三、Handler 使用注意事项
在介绍完`Handler`的这些用法，我们再来总结一下使用`Handler`是比较容易犯的一些错误。
## 3.1 内存泄漏
一个比较常见的错误就是将定义`Handler`的子类时将它作为`Activity`的内部类，而由于内部类会默认持有外部类的引用，因此，如果这个内部类的实例在`Activity`试图被回收的时候，没有被销毁掉，那么就会导致`Activity`无法被回收，从而引起内存泄漏。

而`Handler`实例无法被销毁掉最常见的情况就是，我们通过它发送了一个延时消息出去，此时这个消息会被放入到该线程所对应的`Looper`中的`MessageQueue`当中，而该消息为了能在得到执行之后，回调到对应的`Handler`，因此它会持有这个`Handler`的实例。

从引用链的角度来看就是下面的情况：
![](http://upload-images.jianshu.io/upload_images/1949836-2c8f4ac46ea9891e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

这个大家见得很多，就不多举例子了。
## 3.2 在子线程中实例化 new Handler()
在子线程中`new Handler()`有下面两个需要注意的点：

**(1) 在 new Handler() 之前调用 Looper.prepare() 方法**

另外一个错误就是我们在子线程当中，直接调用`new Handler()`，那么这时候会抛出异常，例如下面这样：
```
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_handler);
        new Thread() {

            @Override
            public void run() {
                mHandler = new MyHandler();
            }

        }.start();
    }
```
抛出的异常为：
![](http://upload-images.jianshu.io/upload_images/1949836-46c332ab8f5d7c40.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
原因是在`Handler`的构造函数中，会去检查当前线程的`Looper`是否已经被初始化：
```
    public Handler(Callback callback, boolean async) {
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
如果没有，那么就会抛出上面代码当中的异常，正确的做法，是需要保证我们在`new Handler`之前，要保证当前线程的`Looper`对象已经被初始化，也就是`Looper`当中的下面这个方法被调用：
```
     /** Initialize the current thread as a looper.
      * This gives you a chance to create handlers that then reference
      * this looper, before actually starting the loop. Be sure to call
      * {@link #loop()} after calling this method, and end it by calling
      * {@link #quit()}.
      */
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
**(2) 如果希望通过该 Looper 接收消息那么要调用 Looper.loop() 方法，并且需要放在最后一句调用**

假如我们只调用了`prepare()`方法，仅仅只是初始化了一个线程的私有的变量，此时是无法通过这个`Looper`构建的`Handler`来接收消息的，例如下面这样：
```
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_handler);
        new Thread() {

            @Override
            public void run() {
                Looper.prepare();
                mHandler = new MyHandler();
            }
            
        }.start();
    }

    public void anrButton(View view) {
        mHandler.sendEmptyMessage(0);
    }

    private static class MyHandler extends Handler {

        @Override
        public void handleMessage(Message msg) {
            Log.e("TAG", "handleMessageId=" + Thread.currentThread().getId());
        }
    }
```
此时我们应当调用`loop`方法让这个循环队列开始工作，并且该调用一定要位于最后一句，因为一旦调用了`loop`方法，那么它就会进入等待 -> 收到消息 -> 唤醒 -> 处理消息 -> 等待的循环过程，而这一过程只有等到`loop`方法返回的时候才会结束，因此，我们初始化`Handler`的语句要放在`loop`方法之前，类似于下面这样：
```
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_handler);
        new Thread() {

            @Override
            public void run() {
                Looper.prepare();
                mHandler = new MyHandler();
                //最后一句再调用
                Looper.loop();
            }

        }.start();
    }
```
# 四、小结
`Handler`的机制似乎已经成为面试必问的题目，如果大家能在回答完内部的实现原理，再根据这些实现原理引申出上面的几个应用和注意事项，可以加分不少哦~
