---
title: Framework 源码解析知识梳理(6) - ContentProvider 源码解析
date: 2017-06-26 21:24
categories : Framework 源码解析知识梳理
---
# 一、前言
在 [Framework 源码解析知识梳理(5) - startService 源码分析](http://www.jianshu.com/p/b66712e5ba0f) 中，我们分析了`Service`启动的内部实现原理，今天，我们趁热打铁，看一下`Android`中的四大组件中另一个组件`ContentProvider`。

# 二、源码解析
在分析之前，先上一张整个的流程图，大家在后面绕晕了以后，可以参考这张图进行对照：
![](http://upload-images.jianshu.io/upload_images/1949836-bfe0ed32d80a9a14.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
## 2.1 ContentResolver 获取过程
在使用`ContentProvider`来进行数据的增删改查时，第一步就是要通过`getContentResolver()`，获得一个`ContentResolver`对象，该方法实际上调用了基类中的`mBase`变量，也就是`ContextImpl`中的`getContentResolver()`方法，并返回它其中的`mContentResolver`变量。
```
    //ContextImpl.java

    @Override
    public ContentResolver getContentResolver() {
        return mContentResolver;
    }
```
而该`mContentResolver`是在`ContextImpl`的构造函数中初始化的，这其实和我们之前在 [插件化知识梳理(9) - 资源的动态加载示例及源码分析](http://www.jianshu.com/p/86dbf0360348) 中所分析的`getResources()`方法返回一个`Resources`对象的过程类似。
```
    private ContextImpl(ContextImpl container, ActivityThread mainThread,
            LoadedApk packageInfo, IBinder activityToken, UserHandle user, boolean restricted,
            Display display, Configuration overrideConfiguration, int createDisplayWithId) {
        //...
        mContentResolver = new ApplicationContentResolver(this, mainThread, user);
    }
```
这上面的`ApplicationContentResolver`是`ContentResolver`的子类：
![](http://upload-images.jianshu.io/upload_images/1949836-1f02c3995d56b32d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
## 2.2 简单的查询过程
现在，我们以`ContentResolver`所提供的`query`方法为例，对`ContentProvider`的调用过程进行一次简单的走读：
```
    public final @Nullable Cursor query(final @NonNull Uri uri, @Nullable String[] projection,
            @Nullable String selection, @Nullable String[] selectionArgs,
            @Nullable String sortOrder, @Nullable CancellationSignal cancellationSignal) {
        Preconditions.checkNotNull(uri, "uri");
        //1.获取ContentProvider接口。
        IContentProvider unstableProvider = acquireUnstableProvider(uri);
        if (unstableProvider == null) {
            return null;
        }
        IContentProvider stableProvider = null;
        Cursor qCursor = null;
        try {
            long startTime = SystemClock.uptimeMillis();

            ICancellationSignal remoteCancellationSignal = null;
            if (cancellationSignal != null) {
                cancellationSignal.throwIfCanceled();
                //2.创建取消信号量。
                remoteCancellationSignal = unstableProvider.createCancellationSignal();
                cancellationSignal.setRemote(remoteCancellationSignal);
            }
            try {
                //3.调用IContentProvider的query方法。
                qCursor = unstableProvider.query(mPackageName, uri, projection,
                        selection, selectionArgs, sortOrder, remoteCancellationSignal);
            } catch (DeadObjectException e) {
                //如果发生了异常，那么销毁unstableProvider对象，重新获取一个stableProvider对象。
                unstableProviderDied(unstableProvider);
                stableProvider = acquireProvider(uri);
                //如果stableProvider对象还是为空，那么直接返回空。
                if (stableProvider == null) {
                    return null;
                }
                //调用stableProvider进行查询。
                qCursor = stableProvider.query(mPackageName, uri, projection,
                        selection, selectionArgs, sortOrder, remoteCancellationSignal);
            }
            if (qCursor == null) {
                return null;
            }

            // Force query execution.  Might fail and throw a runtime exception here.
            qCursor.getCount();
            long durationMillis = SystemClock.uptimeMillis() - startTime;
            maybeLogQueryToEventLog(durationMillis, uri, projection, selection, sortOrder);

            //用CursorWrapperInner把qCursor包裹起来。
            CursorWrapperInner wrapper = new CursorWrapperInner(qCursor,
                    stableProvider != null ? stableProvider : acquireProvider(uri));
            stableProvider = null;
            qCursor = null;
            return wrapper;
        } catch (RemoteException e) {
            // Arbitrary and not worth documenting, as Activity
            // Manager will kill this process shortly anyway.
            return null;
        } finally {
            if (qCursor != null) {
                qCursor.close();
            }
            if (cancellationSignal != null) {
                cancellationSignal.setRemote(null);
            }
            if (unstableProvider != null) {
                releaseUnstableProvider(unstableProvider);
            }
            if (stableProvider != null) {
                releaseProvider(stableProvider);
            }
        }
    }
```
我们对上面的流程进行一个简单的梳理：
- 通过`acquireUnstableProvider`获取一个`unstableProvider`实例，按字面上的翻译它是一个不稳定的`ContentProvider`。
- 通过第一步中获取的`unstableProvider`实例进行查询，如果查询成功，那么得到`qCursor`对象；如果`ContentProvider`所对应的进程已经死亡，那么将会释放`unstableProvider`对象，再通过调用`acquireProvider`方法重新得到一个`stableProvider`，它和`unstableProvider`相同，都是实现了`IContentProvider`接口，之后在通过它来查询得到`qCursor`。
- 把第二步中获得的`qCursor`用`CursorWrapperInner`包裹起来，这里需要注意的是第二个参数，如果是通过`unstableProvider`查询得到的`qCursor`，那么将需要调用`acquireProvider`，并将返回值传入。

那么，我们接下来就要分析通过`acquireUnstableProvider`、`acquireProvider`获取`IContentProvider`的过程。

## 2.3 IContentProvider 获取过程
首先，通过`acquireUnstableProvider`方法根据`Uri`中的`authority`字段，调用`acquireUnstableProvider(Context c, String auth)`方法：
![](http://upload-images.jianshu.io/upload_images/1949836-f2fffd1ec39e1e12.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
该方法是由我们前面看到的`ApplicationContentResolver`所实现的：
![](http://upload-images.jianshu.io/upload_images/1949836-6944c5161b924669.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
可以看到，这里调用了`mMainThread`的`acquireProvider`方法，它实际上是一个`ActivityThread`实例，其实现为：
```
       public final IContentProvider acquireProvider(
            Context c, String auth, int userId, boolean stable) {
        //首先从缓存中获取，如果获取到就直接返回。
        final IContentProvider provider = acquireExistingProvider(c, auth, userId, stable);
        if (provider != null) {
            return provider;
        }

        IActivityManager.ContentProviderHolder holder = null;
        try {
            //如果缓存当中没有，那么首先通过AMS进行获取。
            holder = ActivityManagerNative.getDefault().getContentProvider(
                    getApplicationThread(), auth, userId, stable);
        } catch (RemoteException ex) {
        }
        if (holder == null) {
            Slog.e(TAG, "Failed to find provider info for " + auth);
            return null;
        }

        //根据返回的holder信息进行安装。
        holder = installProvider(c, holder, holder.info,
                true /*noisy*/, holder.noReleaseNeeded, stable);
        return holder.provider;
    }
```
这里，首先会去缓存中查找`IContentProvider`，如果没有找到，那么在调用`AMS`的方法去查找，获取一个`ContentProviderHolder`对象。
### 2.3.1 调用者进程不存在缓存的情况
在这种情况下面，会执行两步操作：
- 第一步：通过`ActivityManagerService`获取`ContentProviderHolder`
- 第二步：通过返回的`ContentProviderHolder`中的信息进行安装

#### 第一步，通过 ActivityManagerService 获取 ContentProviderHolder
这里我们先假设没有缓存的情况，通过 [Framework 源码解析知识梳理(1) - 应用进程与 AMS 的通信实现](http://www.jianshu.com/p/f26a76e19a2d) 中学到的知识，我们知道它最终会调用到`ActivityManagerService`的下面这个方法：
![](http://upload-images.jianshu.io/upload_images/1949836-2f6a2a43672c01c7.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
接下来最终会调用到`getContentProviderImpl`方法返回一个`ContentProviderHolder`对象，这个方法比较长，就不贴代码了，直接说结论，这里会分为以下几种情况：

**(a) ContentProvider 所在进程已经启动，并且已经该 ContentProvider 已经被安装**

这种情况下，直接返回该`ContentProviderHolder`即可：
![](http://upload-images.jianshu.io/upload_images/1949836-224e841f62688d2e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
**(b) ContentProvider 所在进程已经启动，但是该 ContentProvider 没有被安装**

此时，就需要通过`ApplicationThread`对象，再和`ContentProvider`所在的进程进行交互，以返回一个`ContentProviderHolder`实例：
![](http://upload-images.jianshu.io/upload_images/1949836-d4d402789ab27cf0.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
经过`Binder`通信，那么最终会调用到`ContentProvider`所在进程的下面这个方法：
![](http://upload-images.jianshu.io/upload_images/1949836-ebab3f6b19fe2f66.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
这里面调用有调用了内部的`installContentProviders`方法：
![](http://upload-images.jianshu.io/upload_images/1949836-432f6f4ca1dc04a0.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
这里的操作分为两步：
- 安装：根据传过来的`List<ProviderInfo>`对象，通过`installProvider`方法进行安装，并将结果存放在`List<ContentProviderHolder>`列表中。
- 发布：将安装的结果，再通过一次消息传递，返回给`ActivityManagerService`。

**(b-1) 安装过程**

在这一步当中，传入的第二个参数`holder`为`null`，因此会根据`Provider`的名字，动态地加载该类，并调用它的`attachInfo`方法：
![](http://upload-images.jianshu.io/upload_images/1949836-03e535cdb11da272.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
我们上面的有两个`Provider`：
- `localProvider`，类型为`ContentProvider`
- `provider`，类型为`Transport`

`provider`是通过`localProvider`的`getIContentProvider`方法获得的，它是`ContentProvider`的一个内部类，它的作用就是作为`ContentProvider`在远程调用者中的一个代理对象，也就是说，`ContentProvider`的使用者是通过获取`ContentProvider`所在进程的一个代理类`Transport`，再通过这个`Transport`对象调用到`ContentProvider`进行查询的：
![](http://upload-images.jianshu.io/upload_images/1949836-90586e4113507a87.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
接下来，还会去调用`localProvider`的`attachInfo`方法，这里面会初始化权限相关的信息，最终会执行`ContentProvider`的`onCreate()`方法：
![](http://upload-images.jianshu.io/upload_images/1949836-7072beeb3fb61730.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
假设上面我们获得的`localProvider`不为空，那么会执行下面的逻辑：
![](http://upload-images.jianshu.io/upload_images/1949836-eb13cdc13fbbf894.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
这里面，我们会生成一个`ProviderClientRecord`对象，其内部包含了下面几个变量：
![](http://upload-images.jianshu.io/upload_images/1949836-2b424f64ee4795a2.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
- `mNames`：`ContentProvider`对象的`authority`
- `mProvider`：远程代理对象
- `mLocalProvider`：本地对象
- `mHolder`：返回给`AMS`的数据结构，`AMS`再会把它返回给`ContentProvider`的调用者，`mHolder`的类型为`IActivityManager.ContentProviderHolder`，其内部包含的数据结构为：
![](http://upload-images.jianshu.io/upload_images/1949836-814ddb85b16a5191.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

关于`ContentProviderHolder`和`ProviderClientRecord`，其继承族谱如下图所示：
![](http://upload-images.jianshu.io/upload_images/1949836-b0ed85668d4bed4d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
**(b-2) 发布过程**

发布过程，其实就是调用了`ActivityManagerService`的`publishContentProviders`方法，将在`ContentProvider`拥有者所创建的`List<ContentProviderHolder>`保存起来：
![](http://upload-images.jianshu.io/upload_images/1949836-99783225f7ef0517.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

**(c) ContentProvider 所在进程没有启动**

在这种情况下，就需要先通过`startProcessLocked`启动`ContentProvider`所在进程，等待进程启动完毕之后，再进行安装。
![](http://upload-images.jianshu.io/upload_images/1949836-ad263b7c2a5e94dd.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
#### 第二步，利用返回的 ContentProviderHolder 中的信息，进行安装
在第一步中，通过`ActivityManagerService`，我们最终获得了`ContentProviderHolder`对象，接下来就是调用`installProvider`方法，这里和我们之前在第一步中的`(b-1)`中所看到的`installProvider`其实是同一个方法，区别在于，之前我们分析的`installProvider`传入的`holder`参数为空，下面，我们就来看一下当`holder`参数不为空时最终会走到下面的逻辑：
![](http://upload-images.jianshu.io/upload_images/1949836-4b856a735bba80b9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
在`installProviderAuthoritiesLocked`方法中，会将它缓存在`mProviderMap`当中。
![](http://upload-images.jianshu.io/upload_images/1949836-4cd4c7fbf2f5bb28.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
### 2.3.2 调用者进程存在缓存的情况
当调用者进程存在缓存时，会调用`acquireExistingProvider`方法，这里面就会通过我们前面所看到的`mProviderMap`进行查找：
![](http://upload-images.jianshu.io/upload_images/1949836-b31258f57779c902.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
# 三、小结
这篇文章拖了一个星期，总算是完成了，源码看的真的头晕，其实最终看下来，发现整个调用过程，和我们之前分析过的 [Framework 源码解析知识梳理(5) - startService 源码分析](http://www.jianshu.com/p/b66712e5ba0f) 很类似，究其根本，就是调用者进程、所有者进程和`ActivityManagerService`进程的三方调用。
