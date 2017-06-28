---
title: Framework 源码解析知识梳理(1) - 应用进程与 AMS 的通信实现
date: 2017-05-13 12:36
categories : Framework 源码解析知识梳理
---
一、前言
我们在许多和`Framework`解析相关的文章中，都会看到`ActivityManagerService`这个类，但是在上层的应用开发中，却很少直接会使用到他。

那么我们为什么要学习它的呢，一个最直接的好处就是它是我们理解应用程序启动过程的基础，只有把和`ActivityManagerService`以及和它相关的类的关系都理解透了，我们才能理清应用程序启动的过程。

今天这篇文章，我们就来对`ActivityManagerService`与应用进程之间的通信方式做一个简单的总结，这里我们根据进程之间的通信方向，分为两个部分来讨论：
- 从应用程序进程到管理者进程

  **(a) IActivityManager**
  **(b) ActivityManagerNative**
  **(c) ActivityManagerProxy**
  **(d) ActivityManagerService**

- 从管理者进程到应用程序进程

  **(a) IApplicationThread**
  **(b) ApplicationThreadNative**
  **(c) ApplicationThreadProxy**
  **(d) ApplicationThread**

# 二、从应用程序进程到管理者进程
在这一方向上的通信，应用程序进程作为客户端，而管理者进程则作为服务端。举一个最简单的例子，当我们启动一个`Activity`，就需要通知全局的管理者，让它去负责启动，这个全局的管理者就运行在另外一个进程，这里就涉及到了从应用程序进程到管理者进程的通信。

这一个方向上通信过程所涉及到的类包括：
![](http://upload-images.jianshu.io/upload_images/1949836-cb52ae8bbe56ba5e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 2.1 应用程序进程向管理者进程发送消息
当我们想要启动一个`Activity`，会首先调用到`Activity`的：
```
    @Override
    public void startActivityForResult(
            String who, Intent intent, int requestCode, @Nullable Bundle options) {
        //...
        Instrumentation.ActivityResult ar =
            mInstrumentation.execStartActivity(
                this, mMainThread.getApplicationThread(), mToken, who,
                intent, requestCode, options);
        //...
    }
```
之后调用到`Instrumentation`的：
```
    public ActivityResult execStartActivity(
            Context who, IBinder contextThread, IBinder token, Activity target,
            Intent intent, int requestCode, Bundle options) {
            //....
            int result = ActivityManagerNative.getDefault()
                .startActivity(whoThread, who.getBasePackageName(), intent,
                        intent.resolveTypeIfNeeded(who.getContentResolver()),
                        token, target != null ? target.mEmbeddedID : null,
                        requestCode, 0, null, options);
            //...
    }
```
这里我们看到了上面`UML`图中`ActivityManagerNative`，它的`getDefault()`方法返回的是一个`IActivityManager`的实现类：
```
    private static final Singleton<IActivityManager> gDefault = new Singleton<IActivityManager>() {
        protected IActivityManager create() {
            //1.得到一个代理对象
            IBinder b = ServiceManager.getService("activity");
            //2.将这个代理对象再经过一层封装
            IActivityManager am = asInterface(b);
            return am;
        }
    };

    static public IActivityManager getDefault() {
        return gDefault.get();
    }
```
这里，对于`gDefault`变量有两点说明：

- 这是一个`static`类型的变量，因此在程序中的任何地方调用`getDefault()`方法访问的是内存中的同一个对象
- 这里采用了`Singleton`模式，也就是懒汉模式的单例，只有第一次调用`get()`方法时，才会通过`create()`方法来创建一个对象

`create()`做了两件事：

- 通过`ServieManager`得到`IBinder`，这个`IBinder`是管理进程在应用程序进程的代理对象，通过`IBinder`的`transact`方法，我们就可以向管理进程发送消息：

```
public boolean transact(int code, Parcel data, Parcel reply, int flags)
```
- 将`IBinder`传入`asInterface(IBinder b)`构建一个`IActivityManager`的实现类，可以看到，这里返回的是一个`ActivityManagerProxy`对象：

```
    static public IActivityManager asInterface(IBinder obj) {
        if (obj == null) {
            return null;
        }
        //这一步先忽略....
        IActivityManager in = (IActivityManager)obj.queryLocalInterface(descriptor);
        if (in != null) {
            return in;
        }
        return new ActivityManagerProxy(obj);
    }
```
下面，我们在来看一下这个`ActivityManagerProxy`，它实现了`IActivityManager`接口，我们可以看到它所实现的`IActivityManager`接口方法都是通过构造这个对象时所传入的`IBinder.transact(xxxx)`来调用的，这些方法之间的区别就在于消息的类型以及参数。
```
class ActivityManagerProxy implements IActivityManager {

    public ActivityManagerProxy(IBinder remote) {
        mRemote = remote;
    }

    public IBinder asBinder() {
        return mRemote;
    }

    public int startActivity(IApplicationThread caller, String callingPackage, Intent intent,
            String resolvedType, IBinder resultTo, String resultWho, int requestCode,
            int startFlags, ProfilerInfo profilerInfo, Bundle options) throws RemoteException {
        Parcel data = Parcel.obtain();
        Parcel reply = Parcel.obtain();
        //这个也很重要，我们之后分析..
        data.writeStrongBinder(caller != null ? caller.asBinder() : null);
        //发送消息到管理者进程...
        mRemote.transact(START_ACTIVITY_TRANSACTION, data, reply, 0);
        reply.readException();
        int result = reply.readInt();
        reply.recycle();
        data.recycle();
        return result;
    }
 }
```
经过上面的分析，我们用一句话总结：
> 应用程序进程通过`ActivityManagerProxy`内部的`IBinder.transact(...)`向管理者进程发送消息，这个`IBinder`是管理者进程在应用程序进程的一个代理对象，它是通过`ServieManager`获得的。

## 2.2 管理者进程处理消息
下面，我们看一下管理者进程对于消息的处理，在管理者进程中，最终是通过`ActivityManagerService`对各个应用程序进行管理的。

它继承了`ActivityManagerNative`类，并重写了`Binder`类的`onTransact(...)`方法，我们前面通过`ActivityManagerProxy`的`IBinder`对象发送的消息最终会调用到管理者进程中的这个函数当中，`ActivityManagerNative`对该方法进行了重写：
```
  @Override
  public boolean onTransact(int code, Parcel data, Parcel reply, int flags) throws RemoteException {
    switch (code) {
        case START_ACTIVITY_TRANSACTION:
            data.enforceInterface(IActivityManager.descriptor);
            IBinder b = data.readStrongBinder();
            IApplicationThread app = ApplicationThreadNative.asInterface(b);
            String callingPackage = data.readString();
            Intent intent = Intent.CREATOR.createFromParcel(data);
            String resolvedType = data.readString();
            IBinder resultTo = data.readStrongBinder();
            String resultWho = data.readString();
            int requestCode = data.readInt();
            int startFlags = data.readInt();
            ProfilerInfo profilerInfo = data.readInt() != 0
                    ? ProfilerInfo.CREATOR.createFromParcel(data) : null;
            Bundle options = data.readInt() != 0
                    ? Bundle.CREATOR.createFromParcel(data) : null;
            //这里在管理者进程进行处理操作....
            int result = startActivity(app, callingPackage, intent, resolvedType,
                    resultTo, resultWho, requestCode, startFlags, profilerInfo, options);
            reply.writeNoException();
            reply.writeInt(result);
            return true;
      //...
  }
```
在`onTransact(xxx)`方法中，会根据收到的消息类型，调用`IActivityManager`接口中所定义的不同接口，而`ActivityManagerNative`是没有实现这些接口的，真正的处理在`ActivityManagerService`中，`ActivityManagerService`开始进行一系列复杂的操作，这里之后我们介绍应用程序启动过程的时候再详细分析。
```
    @Override
    public final int startActivity(IApplicationThread caller, String callingPackage,
            Intent intent, String resolvedType, IBinder resultTo, String resultWho, int requestCode,
            int startFlags, ProfilerInfo profilerInfo, Bundle bOptions) {
        return startActivityAsUser(caller, callingPackage, intent, resolvedType, resultTo,
                resultWho, requestCode, startFlags, profilerInfo, bOptions,
                UserHandle.getCallingUserId());
    }
```
同样的，我们也用一句话总结：
> 管理者进程通过`onTransact(xxxx)`处理应用程序发送过来的消息

# 三、从管理者进程到应用程序进程
接着，我们考虑另一个方向上的通信方式，从管理者进程到应用程序进程，这一方向上的通信过程涉及到下面的类：
![](http://upload-images.jianshu.io/upload_images/1949836-e8f9c8e3fa3fa081.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
可以看到，`ApplicationThread`的整个框架和上面很类似，只不过在这一个方向上，管理者进程作为客户端，而应用程序进行则作为服务端。
## 3.1 管理者进程向应用程序进程发送消息
前面我们分析的时候，应用程序进程向管理者进程发送消息的时候，是通过`IBinder`这个管理者进程在应用程序进程中的代理对象来实现的，而这个`IBinder`则是通过`ServiceManager`获取的：
```
IBinder b = ServiceManager.getService("activity");
```
同理，如果管理者进程希望向应用程序进程发送消息，那么它也必须设法得到一个应用程序进程在它这边的代理对象。

我们回忆一下，在第二节的分析中，应用者进程向管理者进程发送消息的同时，通过`writeStringBinder`，放入了下面这个对象：
```
public int startActivity(IApplicationThread caller, ...) {
    //...
    data.writeStrongBinder(caller != null ? caller.asBinder() : null);
    //...
    mRemote.transact(START_ACTIVITY_TRANSACTION, data, reply, 0);
}
```
这个`caller`是在最开始调用`startActivityForResult`时传入的：
```
    ActivityThread mMainThread;

    @Override
    public void startActivityForResult(
            String who, Intent intent, int requestCode, @Nullable Bundle options) {
        //...
        Instrumentation.ActivityResult ar =
            mInstrumentation.execStartActivity(
                this, mMainThread.getApplicationThread(), mToken, who,
                intent, requestCode, options);
        //...
    }
```
通过查看`ActivityThread`的代码，我们可以看到它其实是一个定义在`ApplicationThread`中的`ApplicationThread`对象，它的`asBinder`实现是在`ApplicationThreadNative`当中：
```
   public IBinder asBinder() {
       return this;
   }
```
在管理者进程接收消息的时候，就可以通过`readStrongBinder`获得这个`ApplicationThread`对象在管理者进程的代理对象`IBinder`：
```
  @Override
  public boolean onTransact(int code, Parcel data, Parcel reply, int flags)
            throws RemoteException {
      switch (code) {
        case START_ACTIVITY_TRANSACTION:
            //取出传入的ApplicationThread对象，之后调用asInterface方法..
            IBinder b = data.readStrongBinder();
            IApplicationThread app = ApplicationThreadNative.asInterface(b);
            //...
            int result = startActivity(app, callingPackage, intent, resolvedType,
                    resultTo, resultWho, requestCode, startFlags, profilerInfo, options);
            return true;
     }
  }
```
接着，它再通过`asInterface(IBinder xx)`方法把传入的代理对象通过`ApplicationThreadProxy`进行了一层封装：
```
    static public IApplicationThread asInterface(IBinder obj) {
        if (obj == null) {
            return null;
        }
        IApplicationThread in = (IApplicationThread) obj.queryLocalInterface(descriptor);
        if (in != null) {
            return in;
        }
        return new ApplicationThreadProxy(obj);
    }
```
之后，管理者进程就可以通过这个代理对象的`transact(xxxx)`方法向应用程序进程发送消息了：
```
class ApplicationThreadProxy implements IApplicationThread {

    private final IBinder mRemote;

    public ApplicationThreadProxy(IBinder remote) {
        mRemote = remote;
    }

    public final IBinder asBinder() {
        return mRemote;
    }

    public final void schedulePauseActivity(IBinder token, boolean finished,
            boolean userLeaving, int configChanges, boolean dontReport) throws RemoteException {
        //....
        mRemote.transact(SCHEDULE_PAUSE_ACTIVITY_TRANSACTION, data, null, IBinder.FLAG_ONEWAY);
        //...
    }
```
这一个过程可以总结为：
> 管理者进程通过`ApplicationThreadProxy`内部的`IBinder`向应用程序进程发送消息，这个`IBinder`是应用程序进程在管理者进程的代理对象，它是在管理者进程接收应用程序进程发送过来的消息中获得的。

## 3.2 用户进程接收消息
在用户程序进程中，`ApplicationThread`的`onTransact(....)`就可以收到管理者进程发送的消息，之后再调用`ApplicationThread`所实现的`IApplicationThread`的接口方法进行消息的处理：
```
  @Override
  public boolean onTransact(int code, Parcel data, Parcel reply, int flags)
            throws RemoteException {
        switch (code) {
        case SCHEDULE_PAUSE_ACTIVITY_TRANSACTION: 
            data.enforceInterface(IApplicationThread.descriptor);
            IBinder b = data.readStrongBinder();
            boolean finished = data.readInt() != 0;
            boolean userLeaving = data.readInt() != 0;
            int configChanges = data.readInt();
            boolean dontReport = data.readInt() != 0;
            schedulePauseActivity(b, finished, userLeaving, configChanges, dontReport);
            return true;
  }
```
# 三、小结
以上就是应用程序进程和管理者进程之间的通信方式，究其根本，都是通过获取对方进程的代理对象的`transact(xxxx)`方法发送消息，而对方进程则在`onTransact(xxxx)`方法中进行消息的处理，从而实现了进程之间的通信。
