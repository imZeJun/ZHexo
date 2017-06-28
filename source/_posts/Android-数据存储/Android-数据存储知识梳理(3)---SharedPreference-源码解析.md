---
title: Android 数据存储知识梳理(3) - SharedPreference 源码解析
date: 2017-05-05 10:14
categories : Android 数据存储知识梳理
---
# 一、概述
`SharedPreferences`在开发当中常被用作保存一些类似于配置项这类轻量级的数据，它采用键值对的格式，将数据存储在`xml`文件当中，并保存在`data/data/{应用包名}/shared_prefs`下：
![](http://upload-images.jianshu.io/upload_images/1949836-b8c6dd3a700ce215.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
今天我们就来一起研究一下`SP`的实现原理。
# 二、SP 源码解析
## 2.1 获取 SharedPreferences 对象
在通过`SP`进行读写操作时，首先需要获得一个`SharedPreferences`对象，`SharedPreferences`是一个接口，它定义了系列读写的接口，其实现类为`SharedPreferencesImpl`、在实际过程中，我们一般通过`Application、Activity、Service`的下面这个方法来获取`SP`对象：
```
public SharedPreferences getSharedPreferences(String name, int mode)
```
来获取`SharedPreferences`实例，而它们最终都是调用到`ContextImpl`的`getSharedPreferences`方法，下面是整个调用的结构：
![](http://upload-images.jianshu.io/upload_images/1949836-7b1a073104001c1c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
在`ContextImpl`当中，`SharedPreferences`是以一个静态双重`ArrayMap`的结构来保存的：
```
private static ArrayMap<String, ArrayMap<String, SharedPreferencesImpl>> sSharedPrefs;
```
下面，我们看一下获取`SP`实例的过程：
```
    public SharedPreferences getSharedPreferences(String name, int mode) {
        SharedPreferencesImpl sp;
        synchronized (ContextImpl.class) {
            if (sSharedPrefs == null) {
                sSharedPrefs = new ArrayMap<String, ArrayMap<String, SharedPreferencesImpl>>();
            }
            //1.第一个维度是包名.
            final String packageName = getPackageName();
            ArrayMap<String, SharedPreferencesImpl> packagePrefs = sSharedPrefs.get(packageName);
            if (packagePrefs == null) {
                packagePrefs = new ArrayMap<String, SharedPreferencesImpl>();
                sSharedPrefs.put(packageName, packagePrefs);
            }
            //2.第二个维度就是调用get方法时传入的name，并且如果已经存在了那么直接返回
            sp = packagePrefs.get(name);
            if (sp == null) {
                File prefsFile = getSharedPrefsFile(name);
                sp = new SharedPreferencesImpl(prefsFile, mode);
                packagePrefs.put(name, sp);
                return sp;
            }
        }

        return sp;
    }
```
在上面，我们看到`SharedPreferencesImpl`的构造传入了一个和`name`相关联的`File`，它就是我们在第一节当中所说的`xml`文件，在构造函数中，会去预先读取这个`xml`文件当中的内容：
```
SharedPreferencesImpl(File file, int mode) {
        //..
        startLoadFromDisk(); //读取xml文件的内容
}
```
这里启动了一个异步的线程，需要注意的是这里会将标志位`mLoad`置为`false`，后面我们会谈到这个标志的作用：
```
    private void startLoadFromDisk() {
        synchronized (this) {
            mLoaded = false;
        }
        new Thread("SharedPreferencesImpl-load") {
            public void run() {
                synchronized (SharedPreferencesImpl.this) {
                    loadFromDiskLocked();
                }
            }
        }.start();
    }
```
在`loadFromDiskLocked`中，将`xml`文件中的内容保存到`Map`当中，在读取完毕之后，唤醒之前有可能阻塞的读写线程：

```
    private Map<String, Object> mMap;

    private void loadFromDiskLocked() {
        //1.如果已经在加载，那么返回.
        if (mLoaded) {
            return;
        }

        //...
        //2.最终保存到map当中
        map = XmlUtils.readMapXml(str);
        mMap = map;

        //...
        //3.由于读写操作只有在mLoaded变量为true时才可进行，因此它们有可能阻塞在调用读写操作的方法上，因此这里需要唤醒它们。
        notifyAll();
    }
```
从`SP`对象的获取过程来看，我们可以得出下面几个结论：
- 与某个`name`所对应的`SP`对象需要等到调用`getSharedPreferences`才会被创建
- 对于同一进程而言，在`Activity/Application/Service`获取`SP`对象时，如果`name`相同，它们实际上获取到的是同一个`SP`对象
- 由于使用的是静态容器来保存，因此即使`Activity/Service`销毁了，它之前创建的`SP`对象也不会被释放，而`SP`中的数据又是用`Map`来保存的，也就是说，我们只要调用了某个`name`相关联的`getSharedPreferences`方法，那么和该`name`对应的`xml`文件中的数据都会被读到内存当中，并且一直到进程被结束。

## 2.2 通过 SharedPreferences 进行读取操作
读取的操作很简单，它其实就是从之间预先读取的`mMap`当中去取出对应的数据，以`getBoolean`为例：
```
    public boolean getBoolean(String key, boolean defValue) {
        synchronized (this) {
            awaitLoadedLocked();
            Boolean v = (Boolean)mMap.get(key);
            return v != null ? v : defValue;
        }
    }
```
这里唯一需要关心的是`awaitLoadedLocked`方法：
```
    private void awaitLoadedLocked() {
        //这里如果判断没有加载完毕，那么会进入无限等待状态
        while (!mLoaded) {
            try {
                wait();
            } catch (InterruptedException unused) {}
        }
    }
```
在这个方法中，会去检查`mLoaded`标志位是否为`true`，如果不为`true`，那么说明没有加载完毕，该线程会释放它所持有的锁，进入等待状态，直到`loadFromDiskLocked`加载完`xml`文件中的内容调用`notifyAll()`后，该线程才被唤醒。

从读取操作来看，我们可以得出以下两个结论：
- 任何时刻读取操作，读取的都是内存中的值，而并不是`xml`文件的值。
- 在调用读取方法时，如果构造函数中的预读取线程没有执行完毕，那么将会导致读取的线程进入等待状态。

## 2.3 通过 SharedPreferences 进行写入操作
### 2.3.1 获取 EditorImpl
当我们需要通过`SharedPreferences`写入信息时，那么首先需要通过`.edit()`获得一个`Editor`对象，这里和读取操作类似，都是需要等到预加载的线程执行完毕：
```
    public Editor edit() {
        synchronized (this) {
            awaitLoadedLocked();
        }
        return new EditorImpl();
    }
```
`Editor`的实现类为`EditorImpl`，以`putString`为例：
```
    public final class EditorImpl implements Editor {

        private final Map<String, Object> mModified = Maps.newHashMap();
        private boolean mClear = false;

        public Editor putString(String key, @Nullable String value) {
            synchronized (this) {
                mModified.put(key, value);
                return this;
            }
        }
   }
```
由上面的代码可以看出，当我们调用`Editor`的`putXXX`方法时，实际上并没有保存到`SP`的`mMap`当中，而仅仅是保存到通过`.edit()`返回的`EditorImpl`的临时变量当中。
### 2.3.2 apply 和 commit 方法
我们通过`editor`写入的数据，最终需要等到调用`editor`的`apply`和`commit`方法，才会写入到内存和`xml`这两个地方。
#### (a) apply
下面，我们先看比较常用的`apply`方法：
```
        public void apply() {
            //1.将修改操作提交到内存当中.
            final MemoryCommitResult mcr = commitToMemory();
           
            //2.写入文件当中
            SharedPreferencesImpl.this.enqueueDiskWrite(mcr, postWriteRunnable); //postWriteRunnable在写入文件完成后进行一些收尾操作.
            
            //3.只要写入到内存当中，就通知监听者.
            notifyListeners(mcr);
        }
```
整个`apply`分为三个步骤：
- 通过`commitToMemory`写入到内存中
- 通过`enqueueDiskWrite`写入到磁盘中
- 通知监听者

其中第一个步骤很好理解，就是根据`editor`中的内容，确定哪些是需要更新的数据，然后把`SP`当中的`mMap`变量进行更新，之后将变化的内容封装成`MemoryCommitResult`结构体。

我们主要看一下第二步，是如何写入磁盘当中的：
```
    private void enqueueDiskWrite(final MemoryCommitResult mcr,
                                  final Runnable postWriteRunnable) {
        //1.写入磁盘任务的runnable.
        final Runnable writeToDiskRunnable = new Runnable() {
                public void run() {
                    //1.1 写入磁盘
                    synchronized (mWritingToDiskLock) {
                        writeToFile(mcr);
                    }
                    //....执行收尾操作.
                    if (postWriteRunnable != null) {
                        postWriteRunnable.run();
                    }
                }
            };
        
        //2.这里如果是通过apply方法调用过来的，那么为false
        final boolean isFromSyncCommit = (postWriteRunnable == null);

        if (isFromSyncCommit) { //apply 方法不走这里
                //...
                writeToDiskRunnable.run();
                return;
        }

        QueuedWork.singleThreadExecutor().execute(writeToDiskRunnable);
    }
```
可以看出，如果调用`apply`方法，那么对于`xml`文件的写入是在异步线程当中进行的。
#### (b) commit 
如果调用的`commit`方法，那么执行的是如下操作：
```
       public boolean commit() {
            //1.写入内存
            MemoryCommitResult mcr = commitToMemory();
            //2.写入文件
            SharedPreferencesImpl.this.enqueueDiskWrite(mcr, null); //由于是同步进行，所以把收尾操作放到Runnable当中.
            //在这里执行收尾操作..
            //3.通知监听
            notifyListeners(mcr);
            return mcr.writeToDiskResult;
        }
```
当使用`commit`方法时，和`apply`类似，都是三步操作，只不过第二步在写入文件的时候，传入的`Runnable`为`null`，因此，对于写入文件的操作是同步的，因此，如果我们在主线程当中调用了`commit`方法，那么实际上是在主线程进行`IO`操作。

#### (c) 回调时机
- 对于`apply`方法，由于它对于文件的写入是异步的，但是`notifyListener`方法不会等到真正写入完成时才通知监听者，因此监听者在收到回调或者`apply`返回时，对于`SP`数据的改变只是写入到了内存当中，并没有写入到文件当中。
- 对于`commit`方法，由于它对于文件的写入是同步的，因此可以保证监听者收到回调时或者`commit`方法返回后，改变已经被写入到了文件当中。

## 2.4 监听 SP 的变化
如果希望监听`SP`的变化，那么可以通过下面的这两个方法：
```
    public void registerOnSharedPreferenceChangeListener(OnSharedPreferenceChangeListener listener) {
        synchronized(this) {
            mListeners.put(listener, mContent);
        }
    }

    public void unregisterOnSharedPreferenceChangeListener(OnSharedPreferenceChangeListener listener) {
        synchronized(this) {
            mListeners.remove(listener);
        }
    }
```
由于对应于`Name`的`SP`在进程中是实际上是一个单例模式，因此，我们可以做到在进程中的任何地方改变`SP`的数据，都能收到监听。
