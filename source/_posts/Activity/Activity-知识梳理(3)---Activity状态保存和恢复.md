---
title: Activity 知识梳理(3) - Activity状态保存和恢复
date: 2017-02-20 23:18
categories : Activity 知识梳理
---
# 一、概述
在开发过程中，不可避免地会遇到`Activity`被回收的场景， `Activity`被回收有两种情况：主动和被动。
- 当`Activity`是被主动回收时，例如按下了`Back`键，那么这时候是无法恢复的，因为系统认为你已经不再需要它了。
- 在被动回收的情况下，虽然这个`Activity`的实例已经被销毁了，但是系统在新建一个`Activity`实例的时候，会带上先前被回收`Activity`的信息，这些信息是被存储在`Bundle`的键值对，这里面有些是系统帮我们读写的，例如`Activity`当中`View`的状态，这部分的信息并不需要我们担心。（为了系统能帮我们恢复状态，必须要为每个`View`都指定一个`id`），如果我们希望保存更多临时的信息，而这些信息又没有必要写入到持久化的存储当中，这时候我们应该重写`onSaveInstanceState`方法，在该回调当中会传入一个`Bundle`对象，只需要把保存的信息放到里面，之后再在`onRestoreInstanceState`和`onCreate`方法中把它读取出来就可以了，这里面有两点需要注意的：
- `onRestoreInstance`只有在`Bundle`不为空时才会回调，而在`onCreate`方法当中是通过判空操作来判断是否有需要恢复的状态的。
- 在重写`onRestoreInstanceState`方法时，应当先调用`super`方法，这样由系统负责保存的部分才能够恢复。

# 二、疑问
关于状态的保存和恢复，实现方法很简单，我们主要了解一下它内部的原理，主要有这么几个问题：

# 2.1 对于`View`的状态，是怎么做到自动恢复的
由于文档要求我们必须调用`super`方法，那么就可以知道保存`View`状态的代码入口必然在`Activity`当中，我们看下`Activity`的默认实现：

```
<!-- Activity.java -->
protected void onSaveInstanceState(Bundle outState){
    outState.putBundle(WINDOW_HIERARCHY_TAG, mWindow.saveHierarchyState);
}

//看到mWindow，自然就想到了PhoneWindow.java
<!-- PhoneWindow.java -->
public Bundle saveHierarchyState() {
     mContentParent.saveHierarchyState(states); //这里是保存View状态的地方。  
     outState.putInt(FOCUSED_ID_TAG, focusedView.getId()); //保存Focus。
     outState.putSparseParcelableArray(PANELS_TAG, panelStates); //保存Panel状态。
}

<!-- View.java -->
public void saveHierachyState(SparseArray<Parcelable> container) {
   dispatchSaveInstanceState(container);
}
protected void dispatchSaveInstceState(Parcelable> container) {
    Parcelable state = onSaveInstanceState();
    container.put(mID, state);
}
```
整个保存`View`状态的流程如下：
- 调用`Activity`的`onSaveInstanceState`方法
- 该方法又调用`mWindow.saveHierarchyState`，把返回的结果保存到`WINDOW_HIERARCHY_TAG`这个`Key`对应的`Value`中
- `mWindow`的实现类`PhoneWindow`当中：
 - 调用根布局的`saveHierarchyState`方法，这里面会从根布局按树形结构遍历，调用每个`ViewGroup/View`的`onSaveInstanceState `
 - 保存`FoucusView`

同时，我们也可以得到结论，保存的前提有两个
 - `View`的子类必须实现了`onSaveInstanceState`
 - 它必须要有一个`ID`，这个`ID`作为`Bundle`的`key`，这也为我们实现自定义 View 时，需要保存状态提供了思路。

# 2.2 `onSaveInstanceState`调用时机
下面看下第二个问题，有了前面分析`Activity`生命周期的经验，我们直接在`ActivityThread`中找相关的代码：
```
<!-- ActivityThread.java -->
//在ActivityThread中，一共有3处调用了 saveInstanceState

private void handleRelaunchActivity(ActivityClientRecord tmp) {
    performPauseActivity(r.token, false, r.isPreHoneycomb());
    if (r.state == null && !r.stopped && !r.isPreHoneycomb) {
        //保存
    }
}

final Bundle performPauseActivity(ActivityRecord r, boolean finished, boolean saveState) {
   if (!r.activity.mFinished && saveState) { //这里saveState的条件是!r.isPreHoneycomb。
       //保存
   }
   mInstrumentation.callActivityOnPause(r.activity);
}

//这个函数调用的地方有：
1.private void handleWindowVisibility(IBinder token, boolean show) //这里saveState是false
2.public void performStopActivity(IBinder token, boolean saveState) //LocalActivityManager调过来的也是false
3.private void handleStopActivity(IBinder token, boolean show, int configChanges) //这里是true，通过 ActivityManagerSupervisor 调过来的

private void performStopActivityInner(ActivityClientRecord r, StopInfo info, boolean keepShow, boolean saveState) {
    if (r.state == null) {
        //保存  
    }
    r.activity.performStop();
}
```
看完这三个调用的地方，问题又解决了，结论就是：
- 如果是`honeycomb`之后的版本，那么在`performPauseActivity`时是不会保存的，而对于`honeycomb`之前的版本，会在回调`onPause() `之前保存。
- 如果在`performPauseActivity`时没有保存，那么在执行`performStopActivityInner` 时`r.state`为空并且是从 `handleStopActivity`过来的，那么会在`onPause()`和`onStop()`之间保存状态，其它情况下不会保存状态。
- 在`ReLaunchAcitivity`时，也会保存状态。

# 2.3 `onRestoreInstanceState`调用时机
在`Activity`的生命周期解析中分析`onStart()`方法的时候我们已经知道，它是在`onStart()`和`onResume()`之间被调用的。
