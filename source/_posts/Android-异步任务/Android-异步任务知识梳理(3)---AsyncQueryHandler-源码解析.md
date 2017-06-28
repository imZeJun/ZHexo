---
title: Android 异步任务知识梳理(3) - AsyncQueryHandler 源码解析
date: 2017-02-20 22:10
categories : Android 异步任务知识梳理
---
# 一、概述
`AsyncQueryHandler`方便了我们对`ContentProvider`进行增、删、改、查，此外，我们通过学习它的原理可以更好地理解`HandlerThread`，学习如何在项目中使用它。

# 二、源码
`AsyncQueryHandler`中的关键是`mWorkerThreadHandler`，它在其`handleMessage`中进行操作，因为它在构造时传入的`Looper`所关联的`Thread`并不是主线程，因此所有在`handleMessage`中的操作都是异步的，这个变量的初始化时在其构造函数中：
```
public AsyncQueryHandler(ContentResolver cr) {    
    super();    
    mResolver = new WeakReference<ContentResolver>(cr);    
    synchronized (AsyncQueryHandler.class) {        
        if (sLooper == null) {            
            HandlerThread thread = new HandlerThread("AsyncQueryWorker");            
            thread.start();            
            sLooper = thread.getLooper();        
        }    
    }    
    mWorkerThreadHandler = createHandler(sLooper);
}

protected Handler createHandler(Looper looper) {    
    return new WorkerHandler(looper);
}
```
可以看到`sLooper`只有在第一次实例化`AsyncQueryHandler`才会生成，因此当我们采用默认实现时，并且在多个地方实例化不同的`AsyncQueryHandler`对象，每个对象对应的是不同的`WorkerHandler`，但是 `WorkerHandler`关联到的是同一个`Looper`，我们通过它执行的所有任务是放在一个队列当中顺序执行的。如果我们不希望运行在默认的`Looper`中，那么也可以通过重写`createHandler`来传入一个另外的 Looper。因为`AsyncQueueHandler`的增、删、改、查的原理都是相同的，因此我们单独看一下增加的操作，就可以理解它的思想了：
```
public final void startInsert(int token, Object cookie, Uri uri, ContentValues initialValues) {     
    Message msg = mWorkerThreadHandler.obtainMessage(token);    
    msg.arg1 = EVENT_ARG_INSERT;    
    WorkerArgs args = new WorkerArgs();    
    args.handler = this;    
    args.uri = uri;    
    args.cookie = cookie;    
    args.values = initialValues;    
    msg.obj = args;    
    mWorkerThreadHandler.sendMessage(msg);
}

protected class WorkerHandler extends Handler {    
    public WorkerHandler(Looper looper) {         
        super(looper);    
    }
    public void handleMessage(Message msg) {
        final ContentResolver resolver = mResolver.get();
        if (resolver == null) return;
        WorkerArgs args = (WorkerArgs) msg.obj;
          int token = msg.what;
          int event = msg.arg1;
          switch (event) {
              case EVENT_ARG_INSERT:    
                  args.result = resolver.insert(args.uri, args.values);    
                  break;
          }
          Message reply = args.handler.obtainMessage(token);
          reply.obj = args;
          reply.arg1 = msg.arg1;
          reply.sendToTarget();
    }
}

@Override
public void handleMessage(Message msg) {    
    WorkerArgs args = (WorkerArgs) msg.obj;
    int token = msg.what;
    int event = msg.arg1;
    switch (event) {
        case EVENT_ARG_INSERT:    
            onInsertComplete(token, args.cookie, (Uri) args.result);    
            break;
    }
}
```
当我们调用了插入方法之后，整个过程如下：
- `mWorkerThreadHandler`发送一条消息，该消息当中带有**插入相关的所有参数**以及**`AsyncQueryHandler`子类的实例**。
- 在`mWorkderThreadHanlder`的`handleMessage`中，它调用`ContentResolver`的对应插入方法进行插入，它和`mWorkderThreadHandler`关联的`Looper`是运行在同一个线程当中的。
- 插入完毕之后，通过消息当中传入的`AsyncQueryHandler`子类的实例将执行的结果发送回去，在其`AsyncQueryHandler`的`handleMessage(Message message)`方法中，回调抽象方法`onInsertComplete(token, args.cookie, (Uri) args.result) `，子类通过实现该方法来获取执行的结果。
