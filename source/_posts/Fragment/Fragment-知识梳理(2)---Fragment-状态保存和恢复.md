---
title: Fragment 知识梳理(2) - Fragment 状态保存和恢复
date: 2017-02-20 23:54
categories : Fragment 知识梳理
---
# 一、概述
在前一篇文章中，我们对于`Fragment`的事务和生命周期有了直观的理解，这篇文章我们来分析一下`Fragment`状态的保存和恢复。

# 二、状态保存
有了分析`Activity `的经验，我们看一下`Activity`中和`Fragment`状态保存相关的代码：
```
protected void onSaveInstanceState(Bundle outState) {       
    Parcelable p = mFragments.saveAllState();    
    if (p != null) {        
        outState.putParcelable(FRAGMENTS_TAG, p);
    }    
}    
```
前面已经分析过，`mFragments `只是一个代理，真正执行的代码还是在`FragmentManager `中，这个方法返回值是`FragmentManagerState`：
```
Parcelable saveAllState() {
    //先执行完之前所有剩余的操作。    
    execPendingActions();    
    if (HONEYCOMB) {
        mStateSaved = true;    
    }
    ...
    FragmentManagerState fms = new FragmentManagerState();
    fms.mActive = active;
    fms.mAdded = added;
    fms.mBackStack = backStack;
    return fms;
}
```
下面我们分三个部分来看一下保存的逻辑：
```
//如果mActive为空，那么什么其它的也不保存了。
if (mActive == null || mActive.size() <= 0) {       
    return null;
}
//开始遍历mActive列表。
int N = mActive.size();
FragmentState[] active = new FragmentState[N];
for (int i = 0; i < N; i++) {
    Fragment f = mActive.get(i);
    if (f != null) {
         //f.mIndex不许为空。
         //这里面保存了基本的属性：mClassName, mIndex, mFromLayout, mFragmentId, ContainerId, mTag, mRetainInstance, mDetach, mArguments.
         FragmentState fs = new FragmentState(f);
         active[i] = fs;
         if (f.mState > Fragment.INITALZING && fs.mSavedFragmentState == null) 
             fs.mSavedFragmentState = saveFragmentBasicState(f);
             if (f.mTarget != null) {
             //保存mTarget 到 mSavedFragmentState
             //保存mTargetRequestCode 到 mSavedFragmentState
         } else {
             fs.mSavedFragmentState = f.mSavedFragmentState;
         }
    }
}
```
其中，`saveFragmentBasicState`保存的是额外的属性，我们看一下这个方法：
```
Bundle saveFragmentBasicState(Fragment f) {
    f.performSaveInstanceState(mStateBundle); //调用 Fragment 的 onSaveInstanceState 方法，让使用者保存额外的数据; 调用 childFragmentManager 的 saveAllState 方法。
    if (f.mView != null) {
       //保存mInnverView 的状态，赋值给mSavedViewState。
        saveFragmentViewState(f); 
    }
    if (!mUserVisibleHint) {
        //保存mUserVisibleHint
    }
    return result;
}
```
- 第一部分，对于`mActive`列表，它主要保存了一下这些属性：
 - `mClassName,mIndex,mFromLayout,mFragmentId,ContainerId,mTag,mRetainInstance,mDetach,mArguments`
 - `mSavedFragmentState：childFragment.saveAllState、mSavedViewState、
mUserVisibleHint、mTarget、mTargetRequestCode、Fragment.onSaveInstanceState`

- 第二部分，处理`mAdded`列表：
 - 遍历` mAdded `列表，保存 `mAdded `的 `mIndex`，是一个int[]数组

- 最后处理`BackStackRecord`，保存成为 `BackStackState`
```
遍历 BackStackRecord，保存成为 BackStackState[]
final int[] mOps;//int[]数组，从head到tail的排列，保存{op.cmd, op.fragment.mIndex, op.enterAnim, op.exitAnim, op.popEnterAnim, op.popExitAnim, removed.size, removeIndex1, removeIndex2,...}
final int mTransition;
final int mTransitionStyle;
final String mName; //最后一次的name.
final int mIndex; //最后一次的index.
```

# 三、状态恢复
我们看一下` Activity` 中状态恢复的代码是在`onCreate`方法中调用的：
```
protected void onCreate(@Nullable Bundle savedInstanceState) {
    if (savedInstanceState != null) {    
        Parcelable p = savedInstanceState.getParcelable(FRAGMENTS_TAG);        
        mFragments.restoreAllState(p, mLastNonConfigurationInstances != null ? mLastNonConfigurationInstances.fragments : null);
    }
     mFragments.dispatchCreate();
}
```
很好理解，就是我们在第二步中保存的那三个部分，那么这个`mLastNonConfigurationInstances`又是什么呢，我们看一下它被赋值的地方：
```
final void attach(...,NonConfigurationInstances lastNonConfigurationInstances.. ) {
    mLastNonConfigurationInstances = lastNonConfigurationInstances;
}

final void performResume() {
     mLastNonConfigurationInstances  = null;
}
```
原来是`attach`方法传进来的，并且在`performResume`的时候会被置为`null`，我们进`ActivityThread`看一下：
```
private Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent) {
    activity.attach(...., r.lastNonConfigurationInstances,...);
}
```
原来是被放在`ActivityClientRecord`当中了，那么它就跟`Activity`的生命周期无关了，在看一下它被赋值的地方：
```
这个函数调用的地方有两个：
1.通过`AMS`调过来时`getNonConfigInstance`为`false`
2.通过`handleRelaunchAcitivity 调过来为`true`

private ActivityClientRecord performDestroyActivity(IBinder token, boolean finishing, int configChanges, boolean getNonConfigInstance) {
    if (getNonConfigInstance) {    
        r.lastNonConfigurationInstances = r.activity.retainNonConfigurationInstances();
    }
}
```
再看一下`retainNonConfigurationInstances`都保存了哪些信息：
```
NonConfigurationInstances retainNonConfigurationInstances() {    
    Object activity = onRetainNonConfigurationInstance();    
    HashMap<String, Object> children = onRetainNonConfigurationChildInstances();   
    List<Fragment> fragments = mFragments.retainNonConfig();    
    ArrayMap<String, LoaderManager> loaders = mFragments.retainLoaderNonConfig();    
    if (activity == null && children == null && fragments == null && loaders == null && mVoiceInteractor == null) {        
        return null;    
    }    
    NonConfigurationInstances nci = new NonConfigurationInstances();    
    nci.activity = activity;    
    nci.children = children;    
    nci.fragments = fragments;    
    nci.loaders = loaders;    
    if (mVoiceInteractor != null) {        
        mVoiceInteractor.retainInstance();        
       nci.voiceInteractor = mVoiceInteractor;    
    }    
    return nci;
}
```
它保存了一个`Fragment`的列表，这个列表是通过`FragmentManager`的`retainNonConfig`返回的：
```
ArrayList<Fragment> retainNonConfig() {
    //遍历mActive列表：
   if (f != null && f.mRetainInstance) {
        fragments.add(f);
        f.mRetaining = true;
        f.mTargetIndex = f.mTarget != null ? f.mTarget.mIndex : -1;
   }
   return fragments;
}
```
原来它是把`mActive`中`mRetainInstance`为`true`的`Fragment`保存了，而且设置了这个标识位的`Fragment `不会执行`onCreate、onDestory`，并且不会从`mActive`列表中移除，具体的源码：
```
if (!f.mRetaining) {    
    f.performCreate(f.mSavedFragmentState);
}

if (!f.mRetaining) {    
    f.performDestroy();
}

if (!f.mRetaining) {    
    makeInactive(f);
}
```
