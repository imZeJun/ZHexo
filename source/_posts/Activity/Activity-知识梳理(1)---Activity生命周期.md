---
title: Activity 知识梳理(1) - Activity 生命周期
date: 2017-02-20 20:51
categories : Activity 知识梳理
---
# 一、概述
学习`Activity`生命周期，首先我们要明白，学习它的目的不仅在于要知道有哪些生命周期，而是在于明白各回掉函数调用的时机，以便在合适的时机进行正确的操作，如初始化变量、页面展示、数据操作、资源回收等。平时的工作习惯都是，`onCreate(xxx)`初始化，`onResume()`注册、拉取数据，`onPause()`反注册，`onDestroy()`释放资源，这篇文章总结了一些和关键生命周期相关联的一些要点。

# 二、金字塔模型
![Activity 生命周期金字塔模型.png](http://upload-images.jianshu.io/upload_images/1949836-2584d4850ed0d2dd.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
在官方文档中，把`Activity`的生命周期理解成为一个金字塔模型，它是基于下面两点：
- 金字塔的每个**阶梯**表示**`Activity`所处的状态**
- 各**回调函数**则是**各状态转换过程当中所经过的路径**

这个模型中包含了`Activity`的六种状态：
- `Created`：创建完成
- `Started`：可见
- `Resumed`：可见
- `Paused`：部分可见
- `Stopped`：不可见
- `Destroyed`：销毁

在这六种状态当中，只有`Resumed`、`Paused`、`Stopped`这几种状态在用户没有进一步操作时会保持在该状态，而其余的，都会在执行完相应的回调函数后快速跳过。

# 三、关键生命周期

## 3.1 `protected void onCreate(Bundle savedInstanceState)`
- 该方法被回调时，意味着一个新的`Activity`被创建了。
- 由于当前处于一个新的`Activity`实体对象当中，所有的东西都是未初始化的，我们一般需要做的事情包括调用`setContentView`方法设置该`Activity`的布局，初始化类成员变量。
- `onCreate(xxx)`方法执行完之后，`Activity`就进入了`Created`状态，然而它并不会在这个状态停留，系统会接着回调`onStart()` 方法由`Created`状态进入到`Started`状态。
- 注意到，`onCreate(xxxx)`是所有这些回调方法中唯一有参的，该参数保存了上次由于`Activity`被动回收时所保存的数据。

## 3.2 `protected void onStart()`
- `onStart()`方法执行完后，`Activity`就进入了`Started`状态，它也同样不会在该状态停留，而是接着回调 `onResume()`方法进入`Resumed`状态。
- `onStart()`被回调的情况有两种：
 - 从`Created`状态过来
 - 从`Stopped`状态过来，从这种状态过来还会先经过`onRestart()`方法。
- 由于它也会从`Stopped`状态跳转过来，因此如果我们在`onStop()`当中反注册了一些广播，或者释放了一些资源，那么在这里需要重新注册或者初始化，可以认为，`onStart()`和`onStop()`是成对的关系。
- `Created`和`Started`都不是持久性的状态，那么为什么要提供一个`onStart()`回调给开发者呢，直接由`Created`状态或者是 `Stopped`状态经过`onResume()`这条路走到`Resumed`状态不就可以吗，那么我们就要分析一下从 `onCreate()`到`onStart()`，再到`onResume()`的过程中，做了哪些其它的操作，这有利于我们在平时的开发中区分这两个回调的使用场景，我们来看一下源码。

首先我们看一下`onStart()`方法调用的地方，通过下面这段代码，我们可以知道`Activity`的`onStart()`最初是通过`Activity#performStart()`方法调用过来的：
```
<!-- Activity.java -->
private Instrumentaion mInstrumentation;

final void performStart() {
    mInstrumentation.callActivityOnStart(this);
}

<!-- Instrumentaion.java -->
public void callActivityOnStart(Activity activity) {
   activity.onStart();
}
```
而`Activity#performStart()`方法是由`ActivityThread#performLaunchActivity(`调过来的：
```
<!-- ActivityThread.java -->
private final void handleLaunchActivity(ActivityClientRecord r, Intent customIntent) {
    Activity a = performLaunchActivity(r, customIntent); //performCreate, performStart()
    if (a != null) { 
        ....
        handleResumeActivity(r.token, false, r.isForward, !r.activity.mFinished && !r.startsNotResumed); //performResume()
       ....
   }
}
//首先看一下调用performCreate, performStart的地方。
private final Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent) {
    mInstrumentation.callActivityOnCreate(activity, r.state); //performCreate
    ....
    if (!r.activity.mFinished) { 
        activity.performStart(); 
        r.stopped = false; 
    }
    if (!r.activity.mFinished) { 
       if (r.state != null) {         
           mInstrumentation.callActivityOnRestoreInstanceState(activity, r.state);  //1.onRestoreIntanceState()
       } 
    } 
    if (!r.activity.mFinished) { 
        activity.mCalled = false; 
        mInstrumentation.callActivityOnPostCreate(activity, r.state); //2.onPostCreate
        if (!activity.mCalled) { 
            throw new SuperNotCalledException( "Activity " + r.intent.getComponent().toShortString() + " did not call through to super.onPostCreate()"); 
       } 
    }
    ...    
}
//这是performResume的入口。
final void handleResumeActivity(IBinder token, boolean clearHide, boolean isForward, boolean reallyResume) {        
    ActivityClientRecord r = performResumeActivity(token, clearHide);
}
//最后看一下performResume真正执行的地方。
public final ActivityClientRecord performResumeActivity(IBinder token, boolean clearHide) {
    try { 
        if (r.pendingIntents != null) { 
            deliverNewIntents(r, r.pendingIntents); //3.onNewIntent()
            r.pendingIntents = null; 
        } 
       if (r.pendingResults != null) { 
            deliverResults(r, r.pendingResults); //4.onActivityResult()
            r.pendingResults = null; 
      } 
      r.activity.performResume(); 
}
```
- 通过上面这段代码，我们可以得出以下几点启发：
 - `onStart()`到`onResume()`的过程中，还可能会回调`onRestoreInstanceState/onPostCreate/onNewIntent/onActvitiyResult`这几个方法。
 - 如果应用退到后台，再次被启动（`onNewIntent`），或者通过`startActivityForResult`方法启动另一个 `Activity`得到结果返回（`onActivityResult`）的时候，在`onStart()`方法当中得到的并不是这两个回调的最新结果，因为上面的这两个方法并没有调用，而在`onResume()`当中，这点是可以得到保证的。 

## 3.3 `protected void onResume()`
- 该方法被回调时，意味着`Activity`已经完全可见了，但这个完全可见的概念并不等同于`Activity`所属的`Window`被`Attach`了，因为在`Activity`初次启动的时候，`Attach`的操作是在回调`onResume()`之后的，也就是下面的这段代码

```
final void handleResumeActivity(IBinder token, boolean clearHide, boolean isForward, boolean reallyResume) {
    ....
    ActivityClientRecord r = performResumeActivity(token, clearHide);
    ....
    if (r.window == null && !a.mFinished && willBeVisible) {
        r.window = r.activity.getWindow();
        View decor = r.window.getDecorView();
        WindowManager.LayoutParams l = r.window.getAttributes();
        ...
        wm.addView(decor, l);
    }
}
```
- 在`onResume()`方法被回调时，由于`DecorView`并不一定`Attach`了，因此这时候我们获取布局内某些`View` 的宽高得到的值有可能是不正确的，既然`onResume()`当中不能保证，那么`onStart()`方法也是同理，所有需要在`attachToWindow`之后才能执行或是期望得到正确结果的操作都需要注意这一点。
- 在`onResume`当中，我们一般会做这么一些事：在可见时重新拉取一些需要及时刷新的数据、注册 `ContentProvider`的监听，最重要的是在`onPause()`中的一些释放操作要在这里面恢复回来。

## 3.4 `protected void onPause()`
- 该方法被回调时，意味着`Activity`部分不可见，例如一个半透明的界面覆盖在了上面，这时候只要`Activity `仍然有一部分可见，那么它会一直保持在`Paused`状态。
- 如果用户从`Paused`状态回到`Resumed`状态，只会回调`onResume`方法。
- 在`onPause()`方法中，应该暂停正在进行的页面操作，例如正在播放的视频，或者释放相机这些多媒体资源。
- 在`onPause()`当中，可以保存一些必要数据到持久化的存储，例如正在编写的信息草稿。
- 不应该在这里执行耗时的操作，因为新界面启动的时候会先回调当前页面的`onPause()`方法，所以如果进行了耗时的操作，那么会影响到新界面的启动时间，官方文档的建议是这些操作应该放到 `onStop()`当中去做，其实在`onStop()`中也不应当做耗时的操作，因为它也是在主线程当中的，而在主线程中永远不应该进行耗时的操作。
- 释放系统资源，例如先前注册的广播、使用的传感器（如`GPS`）、以及一些仅当页面获得焦点时才需要的资源。
- 当处于`Paused`状态时，`Activity`的实例是被保存在内存中的，因此在其重新回到`Resumed`状态的过程中，不需要重新初始化任何的组件。

## 3.5 `protected void onStop()`
- 该方法被回调时，表明`Activity`已经完全不可见了。
- 在任何场景下，系统都会先回调`onPause()`，之后再回调`onStop()`。
- 官方文档有谈到，当`onStop()`执行完之后，系统有可能会销毁这个`Activity`实例，在某些极端情况下，有可能不会回调`onDestroy()`方法，因此，我们需要在`onStop()`当中释放掉一些资源来避免内存泄漏，而 `onDestory()`中需要释放的是和`Activity`相关的资源，如线程之类的（这点平时在工作中很少用，一般我们都是在`onDestroy()`中释放所有资源，而且也没有去实现`onStart()` 方法，究竟什么时候不会走`onDestroy()`，这点值得研究，目前的猜测是该`Activity`在别的地方被引用了，导致其不能回收）。
- 当`Activity`从`Stopped`状态回到前台时，会先回调`onRestart()`方法，然而我们更多的是使用`onStart()` 方法作为`onStop()`的对应关系，因为无论是从`Stopped`状态，还是从`Created`状态跳转到`Resumed`状态，都是需要初始化必要的资源的，而它们经过的共同路径是`onStart()`。

## 3.6 `protected void onDestroy()`
- 该方法被回调时，表明`Activity`是真正要被销毁了，
- 此时应当释放掉所有用到的资源，避免内存泄漏。
