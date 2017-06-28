---
title: Framework 源码解析知识梳理(3) - 应用进程之间的通信实现
date: 2017-05-16 23:49
categories : Framework 源码解析知识梳理
---
# 一、前言
在 [Framework 源码解析知识梳理(1) - 应用进程与 AMS 的通信实现](http://www.jianshu.com/p/f26a76e19a2d) 和 [Framework 源码解析知识梳理(2) - 应用进程与 WMS 的通信实现](http://www.jianshu.com/p/0fd68bf4aa5b) 这两篇文章中，我们介绍了应用进程与`AMS`以及`WMS`之间的通信实现，但是逻辑还是比较绕的，为了方便大家更好地理解，我们介绍一下大家见得比较多的应用进程间通信的实现。

# 二、例子
说起应用进程之间的通信，相信大家都不陌生，应用进程之间通信最常用的方式就是`AIDL`，下面，我们先演示一个`AIDL`的简单例子，接下来，我们再分析它的内部实现。
## 2.1 服务端
### 2.1.1 编写 AIDL 文件
第一件事，就是服务端需要声明自己可以为客户端提供什么功能，而这一声明则需要通过一个`.aidl`文件来实现，我们先简要介绍介绍一下`AIDL`文件：
- `AIDL`文件的后缀为`*.aidl`
- 对于`AIDL`默认支持的数据类型，是不需要导入包的，这些默认的数据类型包括：
 - 基本数据类型：` byte/short/int/long/float/double/boolean/char`
 - `String/CharSequence`
 - `List<T>`：其中`T`必须是`AIDL`支持的类型，或者是其它`AIDL`生成的接口，或者是实现了`Parcelable`接口的对象，`List`支持泛型
 - `Map`：它的要求和`List`类似，但是不支持泛型
- `Tag`标签：对于接口方法中的形参，我们需要用`in/out/inout`三种关键词去修饰：
 - `in`：表示服务端会收到客户端的完整对象，但是在服务端对这个对象的修改不会同步给客户端
 - `out`：表示服务端会收到客户端的空对象，它对这个对象的修改会同步到客户端
 - `inout`：表示服务端会收到客户端的完整对象，它对这个对象的修改会同步到客户端

说了这么多，我们最常见的需求无非就是两个：让复杂对象实现`Parcelable`接口实现传输以及定义接口方法。

**(1) Parcelable 的实现**

我们可以很方便地通过`AS`插件来让某个对象实现`Parcelable`接口，并自动补全要实现的方法：
![](http://upload-images.jianshu.io/upload_images/1949836-0560bf55a03ec35c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
安装完之后重启，我们原本的对象为：
```
public class InObject {

    private int inData;

    public int getInData() {
        return inData;
    }

    public void setInData(int inData) {
        this.inData = inData;
    }
    
}
```
在文件的空白处，点击右键`Generate -> Parcelable`：
![](http://upload-images.jianshu.io/upload_images/1949836-ba7f5e859dc1712b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
它就会帮我们实现`Parcelable`中的接口：
```
public class InObject implements Parcelable {

    private int inData;

    public int getInData() {
        return inData;
    }

    public void setInData(int inData) {
        this.inData = inData;
    }

    public InObject() {}

    @Override
    public int describeContents() {
        return 0;
    }

    @Override
    public void writeToParcel(Parcel dest, int flags) {
        dest.writeInt(this.inData);
    }

    protected InObject(Parcel in) {
        this.inData = in.readInt();
    }

    public static final Parcelable.Creator<InObject> CREATOR = new Parcelable.Creator<InObject>() {
        @Override
        public InObject createFromParcel(Parcel source) {
            return new InObject(source);
        }

        @Override
        public InObject[] newArray(int size) {
            return new InObject[size];
        }
    };
}
```

**(2) 编写 AIDL 文件**

点击`File -> New -> AIDL -> AIDL File`之后，会多出一个名叫`aidl`的文件夹，之后我们所有需要用到的`aidl`文件都被存放在这里：
![](http://upload-images.jianshu.io/upload_images/1949836-ceb3c27ef9002926.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
初始时候，`AS`为我们生成的`AIDL`文件为：
```
// AIDLInterface.aidl
package com.demo.lizejun.binderdemoclient;

// Declare any non-default types here with import statements

interface AIDLInterface {
    /**
     * Demonstrates some basic types that you can use as parameters
     * and return values in AIDL.
     */
    void basicTypes(int anInt, long aLong, boolean aBoolean, float aFloat,
            double aDouble, String aString);
}
```
当我们编译之后，就会在下面这个路径中生成一个`Java`文件：
```
/*
 * This file is auto-generated.  DO NOT MODIFY.
 * Original file: /home/lizejun/Repository/RepoGithub/BinderDemo/app/src/main/aidl/com/demo/lizejun/binderdemoclient/AIDLInterface.aidl
 */
package com.demo.lizejun.binderdemoclient;
// Declare any non-default types here with import statements

public interface AIDLInterface extends android.os.IInterface {
    /**
     * Local-side IPC implementation stub class.
     */
    public static abstract class Stub extends android.os.Binder implements com.demo.lizejun.binderdemoclient.AIDLInterface {
        private static final java.lang.String DESCRIPTOR = "com.demo.lizejun.binderdemoclient.AIDLInterface";

        /**
         * Construct the stub at attach it to the interface.
         */
        public Stub() {
            this.attachInterface(this, DESCRIPTOR);
        }

        /**
         * Cast an IBinder object into an com.demo.lizejun.binderdemoclient.AIDLInterface interface,
         * generating a proxy if needed.
         */
        public static com.demo.lizejun.binderdemoclient.AIDLInterface asInterface(android.os.IBinder obj) {
            if ((obj == null)) {
                return null;
            }
            android.os.IInterface iin = obj.queryLocalInterface(DESCRIPTOR);
            if (((iin != null) && (iin instanceof com.demo.lizejun.binderdemoclient.AIDLInterface))) {
                return ((com.demo.lizejun.binderdemoclient.AIDLInterface) iin);
            }
            return new com.demo.lizejun.binderdemoclient.AIDLInterface.Stub.Proxy(obj);
        }

        @Override
        public android.os.IBinder asBinder() {
            return this;
        }

        @Override
        public boolean onTransact(int code, android.os.Parcel data, android.os.Parcel reply, int flags) throws android.os.RemoteException {
            switch (code) {
                case INTERFACE_TRANSACTION: {
                    reply.writeString(DESCRIPTOR);
                    return true;
                }
                case TRANSACTION_basicTypes: {
                    data.enforceInterface(DESCRIPTOR);
                    int _arg0;
                    _arg0 = data.readInt();
                    long _arg1;
                    _arg1 = data.readLong();
                    boolean _arg2;
                    _arg2 = (0 != data.readInt());
                    float _arg3;
                    _arg3 = data.readFloat();
                    double _arg4;
                    _arg4 = data.readDouble();
                    java.lang.String _arg5;
                    _arg5 = data.readString();
                    this.basicTypes(_arg0, _arg1, _arg2, _arg3, _arg4, _arg5);
                    reply.writeNoException();
                    return true;
                }
            }
            return super.onTransact(code, data, reply, flags);
        }

        private static class Proxy implements com.demo.lizejun.binderdemoclient.AIDLInterface {
            private android.os.IBinder mRemote;

            Proxy(android.os.IBinder remote) {
                mRemote = remote;
            }

            @Override
            public android.os.IBinder asBinder() {
                return mRemote;
            }

            public java.lang.String getInterfaceDescriptor() {
                return DESCRIPTOR;
            }

            /**
             * Demonstrates some basic types that you can use as parameters
             * and return values in AIDL.
             */
            @Override
            public void basicTypes(int anInt, long aLong, boolean aBoolean, float aFloat, double aDouble, java.lang.String aString) throws android.os.RemoteException {
                android.os.Parcel _data = android.os.Parcel.obtain();
                android.os.Parcel _reply = android.os.Parcel.obtain();
                try {
                    _data.writeInterfaceToken(DESCRIPTOR);
                    _data.writeInt(anInt);
                    _data.writeLong(aLong);
                    _data.writeInt(((aBoolean) ? (1) : (0)));
                    _data.writeFloat(aFloat);
                    _data.writeDouble(aDouble);
                    _data.writeString(aString);
                    mRemote.transact(Stub.TRANSACTION_basicTypes, _data, _reply, 0);
                    _reply.readException();
                } finally {
                    _reply.recycle();
                    _data.recycle();
                }
            }
        }

        static final int TRANSACTION_basicTypes = (android.os.IBinder.FIRST_CALL_TRANSACTION + 0);
    }

    /**
     * Demonstrates some basic types that you can use as parameters
     * and return values in AIDL.
     */
    public void basicTypes(int anInt, long aLong, boolean aBoolean, float aFloat, double aDouble, java.lang.String aString) throws android.os.RemoteException;
}
```
整个结构图为：
![](http://upload-images.jianshu.io/upload_images/1949836-5eb19c93ac9952cf.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
有没有感觉在 [Framework 源码解析知识梳理(1) - 应用进程与 AMS 的通信实现](http://www.jianshu.com/p/f26a76e19a2d) 和 [Framework 源码解析知识梳理(2) - 应用进程与 WMS 的通信实现](http://www.jianshu.com/p/0fd68bf4aa5b) 中也见到过类似的东西 `asInterface/asBinder/transact/onTransact/Stub/Proxy...`，这个我们之后再来解释，下面我们介绍服务端的第二步操作。
### 2.1.2 编写 Service　
既然服务端已经定义好了接口，那么接下来就服务端就需要实现这些接口：
```
public class AIDLService extends Service {

    private final AIDLInterface.Stub mBinder = new AIDLInterface.Stub() {
        @Override
        public void basicTypes(int anInt, long aLong, boolean aBoolean, float aFloat, double aDouble, String aString) throws RemoteException {
            Log.d("basicTypes", "basicTypes");
        }
    };

    @Nullable
    @Override
    public IBinder onBind(Intent intent) {
        return mBinder;
    }
}
```
在这个`Service`中，我们只需要实现`AIDLInterface.Stub`接口中的`basicTypes`接口就可以了，它就是我们在`AIDL`文件中定义的接口，再把这个对象通过`onBinde`方法返回。
### 2.1.3 声明 Service
最后一步，在`AndroidManifest.xml`文件中声明这个`Service`：
```
        <service
            android:name=".server.AIDLService"
            android:enabled="true"
            android:exported="true">
        </service>
```
## 2.2 客户端
### 2.2.1 编写 AIDL 文件
和服务端类似，客户端也需要和服务端一样，生成一个相同的`AIDL`文件，这里就不多说了：
```
// AIDLInterface.aidl
package com.demo.lizejun.binderdemoclient;

// Declare any non-default types here with import statements

interface AIDLInterface {
    /**
     * Demonstrates some basic types that you can use as parameters
     * and return values in AIDL.
     */
    void basicTypes(int anInt, long aLong, boolean aBoolean, float aFloat,
            double aDouble, String aString);
}
```
和服务端类似，我们也会得到一个由`aidl`生成的`Java`接口文件。
## 2.2.2 绑定服务，并调用
**(1) 绑定服务**
```
    private void bind() {
        Intent intent = new Intent();
        intent.setComponent(new ComponentName("com.demo.lizejun.binderdemo", "com.demo.lizejun.binderdemo.server.AIDLService"));
        boolean result = bindService(intent, mServiceConnection, BIND_AUTO_CREATE);
        Log.d("bind", "result=" + result);
    }

    private ServiceConnection mServiceConnection = new ServiceConnection() {

        @Override
        public void onServiceConnected(ComponentName name, IBinder service) {
            mBinder = AIDLInterface.Stub.asInterface(service);
        }

        @Override
        public void onServiceDisconnected(ComponentName name) {
            mBinder = null;
        }
    };
```
**(2) 调用接口**

```
    public void sayHello(View view) {
        try {
            if (mBinder != null) {
                mBinder.basicTypes(0, 0, false, 0, 0, "aaa");
            }
        } catch (RemoteException e) {
            e.printStackTrace();
        }
    }
```
之后我们打印出响应的`log`：
![](http://upload-images.jianshu.io/upload_images/1949836-61590cfbe9a2b590.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

# 三、实现原理
下面，我们就一起来分析一下通过`AIDL`实现的进程间通信的原理。

**(1) 客户端获得服务端的远程代理对象 IBinder **

我们从客户端说起，当我们在客户端调用了`bindService`方法之后，就会启动服务端实现的`AIDLService `，而在该`AIDLService`的`onBind()`方法中返回了`AIDLInterface.Stub`的实现类，绑定成功之后，客户端就通过`onServiceConnected`的回调，得到了它在客户端进程的远程代理对象`IBinder`。

**(2) AIDLInterface.Stub.asInterface(IBinder)**

在拿到这个`IBinder`对象之后，我们通过`AIDLInterface.Stub.asInterface(IBinder)`方法，对这个`IBinder`进行了一层包装，转换成为`AIDLInterface`接口类，那么这个`AIDLInterface`是怎么来的呢，它就是通过我们在客户端定义的`aidl`文件在编译时生成的，可以看到，最终`asInterface`方法会返回给我们一个`AIDLInterface`的实现类`Proxy`，`IBinder`则被保存为它内部的一个成员变量`mRemote`。
![](http://upload-images.jianshu.io/upload_images/1949836-5c0253a3ba2bf9c0.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
也就是说，我们在客户端中保存的`mBinder`实际上是一个由`AIDL`文件所生成的`Proxy`对象。

**(3) 调用 AIDLInterface 的接口方法**

`Proxy`实现了`AIDLInterface`中定义的接口方法，当我们调用它的接口方法时：
![](http://upload-images.jianshu.io/upload_images/1949836-f5fd5a8f97f37bed.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
实际上是通过它内部的`mRemote`对象的`transact`方法发送消息：
![](http://upload-images.jianshu.io/upload_images/1949836-806450199d906587.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

**(4) 服务端接收消息**

那么客户端发送的这一消息去哪里了呢，回想一下在服务端中通过`onBind()`放回的`IBinder`，它其实是由`AIDL`文件生成的`AIDLInterface.java`中的内部类`Stub`的实现，而在`Stub`中有一个回调函数`onTransact`，当我们通过客户端发送消息之后，那么服务端的`Stub`对象就会通过`onTransact`收到这一发送的消息：
![](http://upload-images.jianshu.io/upload_images/1949836-64d53a0cfc5c4ea0.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

**(5) 调用子类的处理逻辑**

在`onTransact`方法中，又会去调用`AIDLInterface`定义的接口：
![](http://upload-images.jianshu.io/upload_images/1949836-73279f7130b3cb9c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

而我们在服务端`AIDLService`中实现了该接口，因此，最终就打印出了我们在上面所看到的文字：
![](http://upload-images.jianshu.io/upload_images/1949836-ea825fdb0d972006.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

# 四、进一步讨论
现在，我们通过这一进程间的通信过程来复习一下前面两篇文章中讨论的应用进程与`AMS`和`WMS`之间的通信实现。

在 [Framework 源码解析知识梳理(1) - 应用进程与 AMS 的通信实现](http://www.jianshu.com/p/f26a76e19a2d) 中，`AMS`所在进程在收到消息之后，进行了下面的处理逻辑：
![](http://upload-images.jianshu.io/upload_images/1949836-d8b382d739c2337c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
这里面的处理逻辑就相当于我们在客户端中`onServiceConnected`中做的工作一样，`IApplicationThread`实际上是一个`ApplicaionThreadProxy`对象，和我们上面`AIDLInterface`实际上是一个`AIDLInterface.Stub.Proxy`的原理是一样的。当我们调用了它的接口方法时，就是通过内部的`mRemote`发送了消息。

在`AMS`通信中，接收消息的是应用进程中的`ApplicationThread`，而我们上面例子中接收消息的是服务端进程的`AIDLInterface.Stub`的实现类`mBinder`，同样是在`onTransact()`方法中接收消息，再由子类去实现处理的逻辑。

[Framework 源码解析知识梳理(2) - 应用进程与 WMS 的通信实现](http://www.jianshu.com/p/0fd68bf4aa5b) 中的原理就更好解释了，因为它就是通过`AIDL`来实现的，我们从客户端向`WMS`所在进程建立会话时，是通过`IWindowSession`来实现的，会话过程中`IWindowSession`就对应于`AIDLInterface`，而`WMS`中的`IWindowSession.Stub`的实现类`Session`，就对应于上面我们在`AIDLService`中定义的`AIDLInterface.Stub`的实现类`mBinder`。

# 五、小结
其实`AIDL`并没有什么神秘的东西，它的本质就是`Binder`通信，我们定义`aidl`文件的目的，主要有两个：
- 为了让应用进程间通信的实现者不用再去编写`transact/onTransact`里面的代码，因为这些东西和业务逻辑是无关的，只不过是简单的发送消息、接收消息。
- 让服务端、客户端只关心接口的定义。

如果我们明白了`AIDL`的原理，那么我们完全可以不用定义`AIDL`文件，自己去参考由`AIDL`文件所生成的`Java`文件的逻辑，进行消息的发送和处理。







