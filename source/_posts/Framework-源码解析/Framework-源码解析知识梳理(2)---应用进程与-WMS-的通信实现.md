---
title: Framework 源码解析知识梳理(2) - 应用进程与 WMS 的通信实现
date: 2017-05-14 12:20
categories : Framework 源码解析知识梳理
---
# 一、前言
在 [Framework 源码解析知识梳理(1) - 应用进程与 AMS 的通信实现](http://www.jianshu.com/p/f26a76e19a2d) 这篇文章中，我们分析了应用进程和`AMS`之间的通信实现，我们今天讨论一下应用进程和`WindowManagerService`之间的通信实现。

在之前的分析中，我们分两个部分来介绍了应用进程与`AMS`之间的通信：
- 应用进程发送消息到`AMS`进程
- `AMS`发送消息到应用进程

现在，我们也按照一样的讨论，分为这两个方向来介绍应用进程与`WMS`之间的通信实现，整个通信的过程会涉及到下面的这些类，其中加粗的线就是整个通信实现的调用路径。
![](http://upload-images.jianshu.io/upload_images/1949836-81611d40e4bd3bec.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


# 二、WindowManagerImpl & WindowManagerGlobal 
在`AMS`的讨论中，我们以在应用进程中启动`Activity`为例子进行了介绍，今天，我们选取另一个大家很常见的例子：`Activity`启动之后，是如何将界面添加到屏幕上的。
## 2.1 WindowManagerImpl 
在 [View 绘制体系知识梳理(2) - setContentView 源码解析](http://www.jianshu.com/p/ee98bab25fdf) 这篇文章中，我们介绍了`DecorView`的相关知识，它就对应于我们需要添加到屏幕上的`View`的根节点，而这一添加的过程就需要涉及到和`WMS`之间的通信，它是在`ActivityThread`的下面这个方法中实现的：
```
<!-- ActivityThread.java -->

final void handleResumeActivity(IBinder token, boolean clearHide, boolean isForward, boolean reallyResume, int seq, String reason) {
        r = performResumeActivity(token, clearHide, reason);
        if (r != null) {
            if (r.window == null && !a.mFinished && willBeVisible) {
                r.window = r.activity.getWindow();
                View decor = r.window.getDecorView();
                decor.setVisibility(View.INVISIBLE);
                ViewManager wm = a.getWindowManager();
                WindowManager.LayoutParams l = r.window.getAttributes();
                a.mDecor = decor;
                l.type = WindowManager.LayoutParams.TYPE_BASE_APPLICATION;
                l.softInputMode |= forwardBit;
                if (r.mPreserveWindow) {
                    a.mWindowAdded = true;
                    r.mPreserveWindow = false;
                    ViewRootImpl impl = decor.getViewRootImpl();
                    if (impl != null) {
                        impl.notifyChildRebuilt();
                    }
                }
                if (a.mVisibleFromClient && !a.mWindowAdded) {
                    a.mWindowAdded = true;
                    wm.addView(decor, l);
                }
            } else if (!willBeVisible) {
                if (localLOGV) Slog.v(
                    TAG, "Launch " + r + " mStartedActivity set");
                r.hideForNow = true;
            }
        }
    }
```
上面的代码中，关键的是下面这几个步骤：
```
//(1) 通过 Activity 获得 Window 的实现类 PhoneWindow
r.window = r.activity.getWindow();

//(2) 通过 PhoneWindow 获得 DecorView
View decor = r.window.getDecorView();

//(3) 通过 Activity 获得 ViewManager 的实现类 WindowManagerImpl
ViewManager wm = a.getWindowManager();
WindowManager.LayoutParams l = r.window.getAttributes();

//(4) 通过 WindowManagerImpl 添加 DecorView
wm.addView(decor, l);
```
**(1) 通过 Activity 获得 Window 的实现类 PhoneWindow**

这里首先调用了`Activity`的`getWindow()`方法：
```
<!-- Activity.java -->

public Window getWindow() {
    return mWindow;
}
```
而这个`mWindow`是在`Activity.attach(xxxx)`中被赋值的，它其实是`Window`的实现类`PhoneWindow`，`PhoneWindow`的构造函数中传入了`Activity`以及`parentWindow`：
```
<!-- Activity.java -->

final void attach(Context context, ActivityThread aThread,
            Instrumentation instr, IBinder token, int ident,
            Application application, Intent intent, ActivityInfo info,
            CharSequence title, Activity parent, String id,
            NonConfigurationInstances lastNonConfigurationInstances,
            Configuration config, String referrer, IVoiceInteractor voiceInteractor,
            Window window) {
        //....
        mWindow = new PhoneWindow(this, window);
}
```
第一步的分析就结束了，大家要记得一个结论：
> 通过`Activity`的`getWindow()`返回的是`PhoneWindow`对象，如果以后需要查看`mWindow`调用的函数，那么应当首先去`PhoneWindow.java`中查看是否有对应的实现，如果没有，那么再去`Window.java`中寻找。

对应于整个流程图的中的这个部分：
![](http://upload-images.jianshu.io/upload_images/1949836-e4a25cb5d6873025.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
**(2) 通过 PhoneWindow 获得 DecorView **

在第二步中，通过第一步返回的`PhoneWindow`获得`DecorView`，这个`mDecor`就是我们在  [View 绘制体系知识梳理(2) - setContentView 源码解析](http://www.jianshu.com/p/ee98bab25fdf) 所介绍的`DecorView`：
```
<!-- PhoneWindow.java -->

@Override
public final View getDecorView() {
    if (mDecor == null || mForceDecorInstall) {
        installDecor();
    }
    return mDecor;
}
```
**(3) 通过 Activity 获得 ViewManager 的实现类 WindowManagerImpl**

下面，我们看第三步，这里通过`Activity`的`getWindowManager()`返回了一个`ViewManager`的实现类：
```
<!-- Activity.java -->

public WindowManager getWindowManager() {
    return mWindowManager;
}
```
和`mWindow`类似，它也是在`Activity`的`attach`方法中赋值的：
```
<!-- Activity.java -->

    final void attach(Context context, ActivityThread aThread,
            Instrumentation instr, IBinder token, int ident,
            Application application, Intent intent, ActivityInfo info,
            CharSequence title, Activity parent, String id,
            NonConfigurationInstances lastNonConfigurationInstances,
            Configuration config, String referrer, IVoiceInteractor voiceInteractor,
            Window window) {
        //...
        mWindow = new PhoneWindow(this, window);
        //...
        mWindow.setWindowManager( 
                (WindowManager)context.getSystemService(Context.WINDOW_SERVICE),
                mToken, 
                mComponent.flattenToString(),
                (info.flags & ActivityInfo.FLAG_HARDWARE_ACCELERATED) != 0);
        if (mParent != null) {
            mWindow.setContainer(mParent.getWindow());
        }
        mWindowManager = mWindow.getWindowManager();
        mCurrentConfig = config;
    }
```
它通过`Window`的`getWindowManager()`返回，我们看一下`Window`的这个方法：
```
<!-- Window.java -->

public WindowManager getWindowManager() {
    return mWindowManager;
}
```
`PhoneWindow`和`Activity`类似，也有一个`mWindowManager`变量，我们再去看一下它被赋值的地方：
```
<!-- Activity.java -->

public void setWindowManager(WindowManager wm, IBinder appToken, String appName, boolean hardwareAccelerated) {
        mAppToken = appToken;
        mAppName = appName;
        mHardwareAccelerated = hardwareAccelerated || SystemProperties.getBoolean(PROPERTY_HARDWARE_UI, false);
        if (wm == null) {
            wm = (WindowManager) mContext.getSystemService(Context.WINDOW_SERVICE);
        }
        mWindowManager = ((WindowManagerImpl) wm).createLocalWindowManager(this);
}
```
在`createLocalWindowManager`返回的是一个`WindowManagerImpl`对象：
```
<!-- WindowManagerImpl.java -->

public WindowManagerImpl createLocalWindowManager(Window parentWindow) {
    return new WindowManagerImpl(mContext, parentWindow);
}
```
这样第三步的结论就是：
> 通过`Activity`的`getWindowManager()`方法返回的是它内部的`mWindowManager`对象，而这个对象是通过`Window`中的`mWindowManager`得到的，它其实是`ViewManager`接口的实现类`WindowManagerImpl`。

`ViewManager`和`WindowManagerImpl`的关系为：
![](http://upload-images.jianshu.io/upload_images/1949836-fc0fc541b4cc6191.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

**(4) 通过 WindowManagerImpl 添加 DecorView**

在第四步中，我们通过`ViewManager`的`addView(View, WindowManager.LayoutParams)`方法添加`DecorView`，经过前面的分析，我们知道它其实是一个`WindowManagerImpl`对象，因此，我们去看一下它所实现的`addView`方法：
```
public final class WindowManagerImpl implements WindowManager {

    private final WindowManagerGlobal mGlobal = WindowManagerGlobal.getInstance();
    private final Context mContext;
    private final Window mParentWindow;

    private WindowManagerImpl(Context context, Window parentWindow) {
        mContext = context;
        mParentWindow = parentWindow;
    }

    @Override
    public void addView(@NonNull View view, @NonNull ViewGroup.LayoutParams params) {
        applyDefaultToken(params);
        mGlobal.addView(view, params, mContext.getDisplay(), mParentWindow);
    }
}
```
可以看到`WindowManagerImpl`什么都没有做，它只是一个代理类，真正去做工作的是`mGlobal`，并且这个`mGlobal`使用了单例模式，也就是说，同一个进程中的所有`Activity`，调用的是同一个`WindowManagerGlobal`对象。

那么，我们下面分析的重点就集中在了`WindowManagerGlobal`上了。

## 2.2 WindowManagerGlobal
我们来看`WindowManagerGlobal`的`addView`方法，这里的第一个参数就是前面传递过来的`mDecor`：
```
<!-- WindowManagerGlobal -->

    public void addView(View view, ViewGroup.LayoutParams params,
            Display display, Window parentWindow) {
        //...
        ViewRootImpl root;
        View panelParentView = null;
        //....
        synchronized (mLock) {
            //...
            root = new ViewRootImpl(view.getContext(), display);
            mViews.add(view);
            mRoots.add(root);
        }
        try {
            root.setView(view, wparams, panelParentView);
        } catch (RuntimeException e) {
            throw e;
        }
    }
```
在`addView(xxx)`方法中，会生成一个`ViewRootImpl`对象，并调用它的`setView(xxx)`方法把它和`DecorView`和它关联起来，与`WMS`通信的逻辑都是由`ViewRootImpl`负责的，`WindowManagerGlobal`则负责用来管理应用进程当中的所有`ViewRootImpl`，对应于整个框架图的部分为：
![](http://upload-images.jianshu.io/upload_images/1949836-bd4e8f93d011b7c2.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
在介绍`ViewRootImpl`之前，我们还要先讲一下在`WindowManagerGlobal`中比较重要的两个静态变量：
```
<!-- WindowManagerGlobal.java -->

private static IWindowManager sWindowManagerService;
private static IWindowSession sWindowSession;
```
**(1) sWindowManagerService 为管理者进程在应用进程中的代理对象**
```
<!-- WindowManagerGlobal.java -->

sWindowManagerService = IWindowManager.Stub.asInterface(ServiceManager.getService("window"));
```
**(2) sWindowSession 为应用进程和管理者进程之间的会话**

```
<!-- WindowManagerGlobal.java -->

InputMethodManager imm = InputMethodManager.getInstance();
IWindowManager windowManager = getWindowManagerService();
sWindowSession = windowManager.openSession(
    new IWindowSessionCallback.Stub() {
        @Override
        public void onAnimatorScaleChanged(float scale) {
            ValueAnimator.setDurationScale(scale);
        }
    }, imm.getClient(), imm.getInputContext());
```
这个会话的方向为从应用进程到管理者进程，通过这个会话，应用进程就可以向管理者进程发送消息，而发送消息的逻辑则是通过`ViewRootImpl`来实现的，下面我们就来看一下这个最重要的类是如何实现的。

# 三、ViewRootImpl
`ViewRootImpl`实现了`ViewParent`接口
![](http://upload-images.jianshu.io/upload_images/1949836-0244f1bd9bd91b9b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
在这个类中有三个最关键的变量：
```
<!-- ViewRootImpl.java -->

View mView;
IWindowSession mWindowSession;
IWindow W;
```
- `mView(View)`：这就是我们在`WindowManagerGlobal`中通过`setView()`传递进来的，对应于`Activity`中的`DecorView`。

```
<!-- ViewRootImpl.java -->

public void setView(View view, WindowManager.LayoutParams attrs, View panelParentView) {
        synchronized (this) {
            if (mView == null) {
                mView = view;
                //....
            }
        }
}
```
- `mWindowSession(IWindowSession)`：代表了从**应用进程到管理者进程**的会话，它其实就是`WindowManagerGlobal`中的`sWindowSession`，对于同一个进程，会复用同一个会话。

```
<!-- ViewRootImpl.java -->

public ViewRootImpl(Context context, Display display) {
    mWindowSession = WindowManagerGlobal.getWindowSession();
    //...
}
```
- `mWindow(W)`：代表了从**管理者进程到应用进程**的会话，是在`ViewRootImpl`中定义的一个内部类。

```
<!-- ViewRootImpl.java -->

    public ViewRootImpl(Context context, Display display) {
        mWindow = new W(this);
    }

    static class W extends IWindow.Stub {

        private final WeakReference<ViewRootImpl> mViewAncestor;
        private final IWindowSession mWindowSession;

        W(ViewRootImpl viewAncestor) {
            mViewAncestor = new WeakReference<ViewRootImpl>(viewAncestor);
            mWindowSession = viewAncestor.mWindowSession;
        }
    }
```
## 3.1 从应用进程到管理者进程
`IWindowSession`是应用进程到管理者进程的会话，它定义了管理者进程所支持的调用接口，通过`IWindowSession`内部的管理者进程的远程代理对象，我们就可以实现从应用进程向管理者进程发送消息。

而在管理者进程中，通过`WindowManagerService`来处理来自各个应用进程的消息，在`WMS`中有一个`Session`列表，所有从应用进程到管理进程的会话都保存在该列表中。
```
<!-- WindowManagerService.java -->

final ArraySet<Session> mSessions = new ArraySet<>();
```
`Session`则实现了`IWindowSession.Stub`接口：
![](http://upload-images.jianshu.io/upload_images/1949836-d251cd2c5747290f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
当它收到消息之后，就会回调`IWindowSession`的接口方法。

我们举一个例子，在`ViewRootImpl`的`setView(xxx)`方法中，调用了`IWindowSession`的下面这个接口方法：
```
<!-- ViewRootImpl.java -->

    public void setView(View view, WindowManager.LayoutParams attrs, View panelParentView) {
        //....
        res = mWindowSession.addToDisplay(mWindow, mSeq, mWindowAttributes,
            getHostVisibility(), mDisplay.getDisplayId(),
            mAttachInfo.mContentInsets, mAttachInfo.mStableInsets,
            mAttachInfo.mOutsets, mInputChannel);
    }
```
最终这一跨进程的调用会回调到该应用进程在管理者进程中对应的`Session`对象的回调方法中：
```
<!-- Session.java -->

    @Override
    public int addToDisplay(IWindow window, int seq, WindowManager.LayoutParams attrs,
            int viewVisibility, int displayId, Rect outContentInsets, Rect outStableInsets,
            Rect outOutsets, InputChannel outInputChannel) {
        return mService.addWindow(this, window, seq, attrs, viewVisibility, displayId,
                outContentInsets, outStableInsets, outOutsets, outInputChannel);
    }
```
## 3.2 从管理者进程到应用进程
如果我们希望实现从管理者进程发送消息到应用进程，那么也需要一个应用进程在管理者进程的代理对象。

在调用`addToDisplay`时，我们传入的第一个参数是`mWindow`，前面我们介绍过，它实现了`IWindow.Stub`接口：
![](http://upload-images.jianshu.io/upload_images/1949836-a9aa1d33d7d81d67.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
这样管理者进程在`Session`的`addToDisplay`方法被回调时，就可以获得一个远程代理对象，它就可以通过`IWindow`中定义的接口方法，实现从管理者进程到应用进程的通信。

在`Session`的`addToDisplay()`方法中，会调用`WMS`的`addWindow`方法，而在`addWindow`方法中，它会创建一个`WindowState`对象，一个进程中的每个`ViewRootImpl`会对应于一个`IWindow`会话，它们被保存在`WMS`的下面这个`HashMap`中：
```
<!-- WindowManagerService.java -->

final HashMap<IBinder, WindowState> mWindowMap = new HashMap<>();
```
其中`key`值就表示应用进程在管理者进程中的远程代理对象，例如我们在`WMS`中调用了下面这个方法：
```
<!-- WindowState.java -->

mClient.windowFocusChanged(focused, inTouchMode);
```
那么应用进程中`IWindow.Stub`的实现的`ViewRootImpl.W`类的对应方法就会被回调，在该回调方法中又会调用`ViewRootImpl`的方法：
```
<!-- ViewRootImpl.java -->

        @Override
        public void windowFocusChanged(boolean hasFocus, boolean inTouchMode) {
            final ViewRootImpl viewAncestor = mViewAncestor.get();
            if (viewAncestor != null) {
                viewAncestor.windowFocusChanged(hasFocus, inTouchMode);
            }
        }
```
而在`ViewRootImpl`的`windowFocusChanged`方法中，会通过它内部的一个`ViewRootHandler`发送消息，`ViewRootHandler`的`Looper`是和应用进程中的主线程所绑定的，因此它就可以在`handleMessage`进行后续逻辑处理。
```
<!-- ViewRootImpl.java -->

    final ViewRootHandler mHandler = new ViewRootHandler();

    public void windowFocusChanged(boolean hasFocus, boolean inTouchMode) {
        Message msg = Message.obtain();
        msg.what = MSG_WINDOW_FOCUS_CHANGED;
        msg.arg1 = hasFocus ? 1 : 0;
        msg.arg2 = inTouchMode ? 1 : 0;
        mHandler.sendMessage(msg);
    }
```
# 四、小结
做一个简单的总结，应用进程与`WMS`之间通信是通过`WindowManagerGlobal`中`ViewRootImpl`来管理的，`ViewRootImpl`中的`IWindowSession`对应于从应用进程到`WMS`的通信，而`IWindow`对应于从管理者进程到应用进程的通信。
