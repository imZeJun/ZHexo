---
title: Fragment 知识梳理(1) - Fragment 源码解析
date: 2017-02-20 23:36
categories : Fragment 知识梳理
---
一、概述
官方是从`3.0`开始引入`Fragment`的，在文档中有对于其使用的详细介绍，可以看出来，在它刚出现时大家对于它是十分推崇的。然而随着使用`Fragment`开发的项目越来越多，在一些复杂场景下逐渐暴露出了它的一些问题，对于是否继续使用`Fragment`大家也有不同的看法。目前在项目中也有大量之前留下来的使用 `Fragment`的代码， `monkey`有时会跑出一些莫名奇妙的问题，在这里我们对于使用`Fragment`的好坏先不做评价，而是看看它内部整个的实现过程，学习它的思想，在以后出现问题的时候也方便排查，同时大家可以根据自己的需求来决定是否使用`Fragment`。

# 二、`Fragment`事务的执行过程
在操作`Fragment`时，第一件是就是通过`Activity`的`getFragmentManger()`方法得到一个`FragmentManager `对象：
```
Fragment manager = getFragmentManager();
```
我们看一下`Activity.java`中的这个方法：
```
final FragmentController mFragments = FragmentController.createController(new HostCallbacks())

public FragmentManager getFragmentManager() {
    return mFragments.getFragmentManager();
}

class HostCallbacks extends FragmentHostCallback<Activity> {
    public HostCallbacks() {
        super(Activity.this);
    }
}
```
可以看到`getFragmentManager()`调用的是`FragmentController`的接口，那么我们再看看这个类对应的方法：
```
private final FragmentHostCallback<?> mHost;

public static final FragmentController createController(FragmentHostCallback<?> callbacks) {
   return new FragmentController(callbacks)
}

private FragmentController(FragmentHostCallback<?> callbacks) {
    mHost = callbacks;
}

public FragmentManager getFragmentManager() {
   return mHost.getFragmentManagerImpl();
}
```
原来`FragmentController`是把一个`callback`给包装了起来，真正完成任务的是`FragmentHostCallback`：
```
final FragmentManagerImpl mFragmentManager = new FragmentManagerImpl();

FragmentManagerImpl getFragmentManagerImpl() {
    return mFragmentManagerImpl;
}
```
那么结论就是我们实际得到的是`Activity#mFragments#mHost#mFragmentManagerImpl`这个变量，得到这个实例之后，我们通过它的`beginTransition`方法得到一个事务：
```
FragmentTransation transation = manager.beginTransation();
```
我们看一下这个`FragmentTransation`究竟是个什么东西，`FragmentManagerImpl`是`FragmentManager`的一个内部类，它实现了该接口：
```
final class FragmentManagerImpl extends FragmentManager implements LayoutInflater.Factory2 {

public FragmentTransaction beginTransation() {
    return new BackStackRecord(this);
}

}
```
原来这个`FragmentTransation`也是一个接口，它的实现是`BackStackRecord`，并且在每次 `beginTransaction`时都是返回一个新的事务对象，包括之后进行的后退操作都是通过这个事务对象来管理的，这个对象中保存了创建它的`FragmentManagerImpl`实例。
`BackStackRecord`其实是`BackStackState`的一个内部类，我们平时就是通过它来进行`add、remove、replace、attach、detach、hide、show`等操作的，进入源码看一下这些操作背后都干了些什么：
```
final class BackStackRecord extends FragmentTransation implements FragmentManager.BackStackEntry, Runnable {

public FragmentTransation add(Fragment fragment, String tag) {
    doAddOp(0, fragment, tag, OP_ADD);
    return this;
}

public FragmentTransation add(int containerViewId, Fragment fragment) {
    doAddOp(containerViewId, fragment, null, OP_ADD);
    return this;
}

public FragmentTransation add(int containerViewId, Fragment fragment, String tag) {
    doAddOp(containerViewId, fragment, tag, OP_ADD);
}

public FragmentTransation replace(int containerViewId, Fragment fragment) {
    return replace(containerViewId, fragment, null);
}

public FragmentTransation replace(int containerViewId, Fragment fragment, String tag) {
    if (containerViewId == 0) {
        //replace操作必须要指定containerViewId.
   }
   doAddOp(containerViewId, fragment, tag, OP_REPLACE);
}

public FragmentTransaction remove(Fragment fragment) {
   Op op = new Op();
   op.cmd = OP_REMOVE;
   op.fragment = fragment;
   addOp(op);
}

public FragmentTransaction hide(Fragment fragment) {
   Op op = new Op();
   op.cmd = OP_HIDE;
   op.fragment = fragment;
   addOp(op);
}

public FragmentTransaction show(Fragment fragment) {
   Op op = new Op();
   op.cmd = OP_SHOW;
   op.fragment = fragment;
   addOp(op);
}

public FragmentTransaction attach(Fragment fragment) {
   Op op = new Op();
   op.cmd = OP_ATTACH;
   op.fragment = fragment;
   addOp(op);
}

public FragmentTransaction detach(Fragment fragment) {
   Op op = new Op();
   op.cmd = OP_DETACH;
   op.fragment = fragment;
   addOp(op);
}

//新建一个操作，并给操作赋值。
private void doAddOp(int containerViewId, Fragment fragment, String tag, int opcmd) {
    fragment.mFragmentManager = mManager;
   if (tag != null) {
       if (fragment.mTag != null && !tag.equals(fragment.mTag)) {
             //如果这个 fragment 之前已经有了tag，那么是不允许改变它的。
      }
      fragment.mTag = tag;
   }
   if (containerViewId != 0) {
        if (fragment.mFragmentId != 0 && fragment.mFragmentId != containerViewId) {
            //如果这个fragment已经有了mFragmentId，那么不允许改变它。
       }
       fragment.mContainerId = fragment.mFragmentId = containerViewId;
   }
   Op op = new Op();
   op.cmd = opcmd;
   op.fragment = fragment;
   addOp(op);
}
//把操作添加到链表之中。
void addOp() {
    if (mHead == null) {
        mHead = mTail = op;
    } else {
       op.prev = mTail;
       mTail.next = op;
       mTail = op;
    }
    mNumOp++；
}
}
```
看完上面这段代码，我们可以得到下面这些信息：
- 我们调用这些操作之后仅仅是把这个操作添加到了`BackStackRecord`当中的一个链表。
- 在进行`attach、detach、hide、show、remove`这五个操作时是不需要传入`containerViewId`的，因为在执行这些操作之前这个`Fragment`必然已经经过了`add`操作，它的`containerId`是确定的。
- 在执行`add`操作时，需要保证`Fragment`的`mTag`和`mContainerViewId`在不为空时（也就是这个 `Fragment`实例之前已经执行过`add`操作），它们和新传入的`tag`和`containerViewId`必须是相同的，这是因为在`FragmentManager`的`mActive`列表中保存了所有被添加进去的`Fragment`，而其提供的 `findFragmentById/Tag`正是通过这两个字段作为判断的标准，因此不允许同一个实例在列表当中重复出现两次。
- `replace`操作和`add`类似，它增加了一个额外条件，就是`containerViewId`不为空，因为`replace`需要知道它是对哪个`container`进行操作，后面我们会看到`replace`其实是一个先`remove`再`add`的过程，因此`add 的那些判断条件同样适用。

那么这个链表中的操作什么时候被执行呢，看一下`commit()`操作：
```
public int commit() {
    return commitInternal(false);
}

int commitInternal(boolean allowStateLoss) {
    if (mCommitted) {
        //已经有事务处理，抛出异常。
    }
    mCommited = true;
    if (mAddToBackState) {
         mIndex = mManager.allocBackStackIndex(this);
    } else {
         mIndex = -1;
    }
    mManager.enqueueAction(this, allowStateLoss);
}
```
它最后调用了`FragmentManager`的`enqueueAction`方法，我们进去看一下里面做了什么：
```
public void enqueueAction(Runnable action, boolean allowStateLoss) {    
    if (!allowStateLoss) {        
        checkStateLoss();    
    }    
    synchronized (this) {        
        if (mDestroyed || mHost == null) {            
           //如果Activity已经被销毁，那么抛出异常。          
        }       
        if (mPendingActions == null) {            
            mPendingActions = new ArrayList<Runnable>();        
        }
        //因为BackStackRecord实现了Runnable接口，把加入到其中。        
        mPendingActions.add(action);        
        if (mPendingActions.size() == 1) {        
           mHost.getHandler().removeCallbacks(mExecCommit);            
           mHost.getHandler().post(mExecCommit);        
        }   
    }
}

Runnable mExecCommit = new Runnable {
    public void run() {
        execPendingActions();
    }
}

public boolean execPendingActions() {
   if (mExecutingActions) {
      //已经有操作在执行，抛出异常
   }
   if (Looper.myLooper() != mHost.getHandler().getLooper()) {
       //不在主线程中执行，抛出异常
   }
   boolean didSomething = false;
   while (true) {
      int numActions;
      sychronized(this) {
          if (mPendingActions == null || mPendingActions.size == 0) {
               break;
          }
          //期望需要执行的事务个数
          numActions = mPendingActions.size();
          if (mTmpActions == null || mTmpActions.length < numActions) {
              mTmpActions = new Runnable[numActions];
          }
          mPendingActions.toArray(mTmpActions);
          mPendingActions.clear();
          mHost.getHandler().removeCallbacks(mExecCommit);
       }
       mExecutingActions = true;
       for (int i = 0; i < numActions; i++) {
            //好吧，这里就又回调了BackStackRecord的run()方法
            mTmpActions[i].run();
            mTmpActions[i] = null;
       }
       mExecutingActions = false;
       didSomething = true; 
   }
}
```
- 在调用`commit`之后，把`BackStackRecord`加入到`FragmentManagerImpl`的`mPendingActions`中，而且通过查看`FragmentManager`的源码也可以发现，所有对`mPendingActions`的添加操作只有这一个地方调用，
- 当其大小为1时，会通过主线程的`Handler post`一个`Runnable mExecCommit`出去，当这个`Runnable`执行时调用`execPendingActions()`方法。
- `execPendingActions`它会拿出这个`mPendingActions`当中的所有`Runnable`执行（如果它是 `BackStackRecord`调用过来的，那么就是调用`BackStackRecord`的`run`方法），并把这个列表清空。在每个`BackStackRecord`的`run`方法执行时，它是通过遍历`BackStackRecord`链表当中每个节点的`cmd`来判断我们之前通过`FragmentTransation`加入期望执行的那些操作的。
- 可以看出`execPendingActions`这个方法很关键，因为它决定了我们添加的操作什么时候会被执行，我们看下还有那些地方调用到了它，为什么要分析这个呢，因为我们在项目当中发现很多来自于`Fragment`的异常都是由我们后面谈论的`moveToState`方法抛出的，而`moveToState`执行的原因是`execPendingActions`被调用了，因此了解它被调用的时机是我们追踪问题的关键，关于调用的时机，我们都总结在下面的注释当中了：

```
<!-- Activity.java -->
final void performStart() {
    mFragments.execPendingActions(); //有可能在 Activity 的 onStart 方法执行前
    mInstrumentation.callActivityOnStart(this);
}

final void performResume() {
    performRestart();
    mFragments.execPendingActions(); //如果是从 Stopped 过来的，那么有可能在 onStart 到 onResume 之间。
    ....
    mInstrumentation.callActivityOnResume(this);
    ....
    mFragments.dispatchResume(); 
    mFragments.execPendingActions(); //有可能在 onResume 到 onPause 之间。
}

<!-- FragmentManager.java -->
public void dispatchDestroy() { //这个调用在 onDestroy 之前。
    execPendingActions();
}

public boolean popBackStackImmediate() {
    executePendingTransactions();
}

public boolean popBackStackImmediate(String name, int flags) {
    executePendingTransactions();
}

```
关于`FragmentManager`的讨论我们先暂时放一放，看一下`BackStackRecord`是怎么执行链表内部的操作的：
```
public void run() {
    Op op = mHead;
    while (op != null) {
        switch(op.cmd) {
             case OP_ADD:
                 Fragment f = op.fragment;
                 f.mNextAnim = op.enterAnim;
                 mManager.addFragment(f, false);
                 break;
            case OP_REPLACE: {
                Fragment f = op.fragment;
                int containerId = f.mContainerId;
                if (mManager.mAdded != null) {
                    //遍历 mAdded列表，找到 containerId 相同的 old Fragment.
                    if (old == f) {
                        op.fragment = f = null;
                    } else {
                        if (op.removed == null) {
                             op.removed = new ArrayList<Fragment>();
                        }
                        op.removed.add(old); //这里要把replace之前的记下来是为了后退栈准备的。
                        if (mAddToBackStack) {
                             old.mBackStackNesting += 1;
                        }
                        mManager.removeFragment(old, transition, transitionStyle);
                    }
                }
                if (f != null) { 
                     f.addFragment(f, false);
                }
                break;
                //后面的remove,hide,show,attach,detach就是调用了FragmentManager中相应的方法，没什么特别的，就不贴出来了
                case xxx:
                   
            }
        }
        op = op.next;
    }
    //mCurState此时为FragmentManager当前的状态，其余的参数不用管。    
    mManager.moveToState(mManager.mCurState, mTransition, mTransitionStyle, true); 
    if (mAddToBackStack) {
        mManager.addBackStackState(this);
    } 
}
```
我们来看一下`FragmentManagerImpl`对应的`addFragment`等操作：
```
public void addFragment(Fragment fragment, boolean moveToStateNow) {
   makeActive(fragment); //加入到mActive列表中。
   if (!fragment.mDetached) {
       if (mAdd.contains(fragment)) {
           //已经在mAdded列表，抛出异常。
       }
       mAdded.add(fragment);
       fragment.mAdded = true;
       fragment.mRemoving = false;
       if (moveToStateNow) {
           moveToState(fragment);
       }
   }
}

public void removeFragment(Fragment fragment, int transition, int transitionStyle) {
    final boolean inactive = !fragment.isInBackStack();
    if (!fragment.mDetach || inactive) {
        if (mAdded != null) {
            mAdded.remove(fragment); //从mAdded列表中移除。
        }
        fragment.mAdded = false;
        fragment.mRemoving = true;
        //这里会根据是否加入后退栈来判断新的状态，最后会影响到Fragment生命周期的调用，如果是没有加入后退栈的，那么会多调用onDestroy、onDetach方法。
        moveToState(fragment, inactive ? Fragment.INITIALZING : Fragment.CREATED, transition, transitionStyle, false); 
    }
}

public void hideFragment(Fragment fragment, int transition, int transitionStyle) {
    if (!fragment.mHidden) {
       fragment.mHidden = true;
       if (fragment.mView != null) {
          fragment.mView.setVisibility(View.GONE);
       }
    }
    fragment.onHiddenChanged(true);
}

public void showFragment(Fragment fragment, int transition, int transitionStyle) {
    if (fragment.mHidden) {
        fragment.mHidden = false;
        if (fragment.mView != null) {
             fragment.mView.setVisibility(View.VISIBLE);
        }
        fragment.onHiddenChanged(false);
    }
}

public void detachFragment(Fragment fragment, int transition, int transitionStyle) {
    if (!fragment.mDetached) {
        fragment.mDetached = true;
        if (fragment.mAdded) {
             if (mAdded != null) {
                 mAdded.remove(fragment);
             }
             fragment.mAdded = false;
             moveToState(fragment, Fragment.CREATED, transition, transitionStyle, false);
        }
    }
}

public void attachFragment(Fragment fragment, int transition, int transitionStyle) {
    if (fragment.mDetached) {
         if (!fragment.mAdded) {
              if (mAdded.contains(fragment)) {
                  //mAdded列表中已经有，抛出异常，
              }
              mAdded.add(fragment);
              fragment.mAdded = true;
              moveToState(fragment, mCurState, transition, transitionStyle, false);
         }
    }

}
```
这里的操作很多，我们需要明白以下几点：
- `mActive`和`mAdded`的区别：`mActive`表示执行过`add`操作，并且其没有被移除（移除表示的是在不加入后退栈的情况下被`removeFragment`），所有被动改变`Fragment`状态的调用都是遍历这个列表；而 `mAdded`则表示执行过`add`操作，并且没有执行`detachFragment/removeFragment`，也就是说`Fragment` 是存在容器当中的，但是有可能是被隐藏的（`hideFragment`）。
- `attachFragment` 必须保证其状态是 `mDetach` 的，而该属性的默认值是 `false`，只有在执行过 `detach` 方法后，才能执行，执行它会把 `f.mView` 重新加入到 `container` 中。
- `detachFragment` 会把 `f.mView` 从 `container` 中移除。
- `removeFragment` 和 `detachFragment` 会强制改变 `Fragment` 的状态，这是因为它们需要改变 `Fragment` 在布局中的位置，而这通过被动地接收 `FragmentManager` 状态（即所在` Activity `的状态）是无法实现的。在 `removeFragment` 时，会根据是否加入后退栈来区分，如果假如了后退栈，因为有可能之后会回退，而回退时遍历的是 `mActive` 列表，如果把它的状态置为`Fragment.INITIALZING`，那么在 `moveToState`方法中就会走到最后一步，把它从`mActive`列表中移除，就找不到了也就无法恢复，因此这种情况下`Fragment`最终的状态的和`detachFragment`是相同的。
- 在加入后退栈时，`detachFragment` 时，会把 `mDetach`置为`true`，这种情况下之后可以执行 `attachFragment `操作但不能执行 `addFragment `操作；`removeFragment` 之后可以执行 `addFragment `操作但不能执行 `attachFragment `操作。
- `showFragment` 和 `hideFragment` 并不会主动改变 `Fragment` 的状态，它仅仅是回调 `onHiddenChanged`方法，其状态还是跟着 `FragmentManager` 来走。

```
mManager.moveToState(mManager.mCurState, mTransition, mTransitionStyle, true); 
```
这个方法才是 `Fragment `的核心，它的思想就是根据 `Fragment` 期望进入的状态和之前的状态进行对比，从而调用 `Fragment `相应的生命周期，那么问题就来，什么是期望进入的状态呢，我认为可以这么理解：
- 用户没有进行主动操作，但是 `Fragment` 和 `FragmentManager `的状态不一致，这时需要发生变化。
- 用户进行了主动操作，无论` Fragment `和 `FragmentManager `的状态是否一致，因为 `Fragment` 的状态发生了变化，因此这时也需要执行。

这里我们为了简便起见先看一下和生命周期有关的代码：

```

static final int INITIALIZING = 0;     // Not yet created.
static final int CREATED = 1;          // Created.
static final int ACTIVITY_CREATED = 2; // The activity has finished its creation.
static final int STOPPED = 3;          // Fully created, not started.
static final int STARTED = 4;          // Created and started, not resumed.
static final int RESUMED = 5;          // Created started and resumed.

void moveToState(int newState, int transit, int transitStyle, boolean always) {
   if (!always && mCurState == newState) {
      return;
   }
   mCurState = newState;
   if (mActive != null) {
       for (int i = 0; i < mActive.size; i++) {
           //在addFragment中通过makeActive加入进去
           Fragment f = mActive.get(i);
           moveToState(f, newState, transit, transitStyle, false);
       }
   }
}

void moveToState(Fragment f, int newState, int transit, int transitionStyle, boolean keepActive) {
    if (f.mState < newState) { //新的状态高于当前状态
        switch(f.mState) {
             case Fragment.INITIALZING:
                 f.onAttach(mHost.getContext());
                 if (f.mParentFragment == null) {
                     mHost.onAttachFragment(f);
                 }
                 if (!f.mRetaining) {
                     f.performCreate(f.mSavedFragmentState);
                 }
                 if (f.mFromLayout) {
                     f.mView = f.performCreateView(f.getLayoutInflater(f.mSavedFragmentState), null, f.mSavedFragmentState);
                     if (f.mHidden) f.mView.setVisibility(View.GONE);
                     f.onViewCreated(f.mView, f.mSavedFragmentState);
                 }
             case Fragment.CREATED:
                  if (newState > Fragment.CREATED) {
                      if (!f.mFromLayout) {
                          ViewGroup container = null;
                          if (f.mCotainerId != null) {
                               cotainer = (ViewGroup) mContainer.onFindViewById(f.mContainerId);
                               if (container == null && !f.mRestored) {
                                  //no view found
                               }
                          }
                          f.mContainer = container;
                          f.mView = f.performCreateView(f.getLayoutInflater(f.mSavedFragmentState), null, f.mSavedFragmentState); 
                          if (f.mView != null) {
                               f.mInnerView = f.mView;
                               if (Build.VERSION.SDK >= 11) {
                                   ViewCompact.setSaveFromParentEnable(f.mView, false);
                               } else {
                                   f.mView = NoSaveStateFrameLayout.wap(f.mView);
                               }
                               if (f.mHidden) f.mView.setVisibility(View.GONE);
                               f.onViewCreated(f.mView, f.mSavedFragmentState);
                          } else {
                               f.mInnerView = null;
                          }
                      }
                      f.performActivityCreated(f.mSavedFragmentState);
                  }
             case Fragment.ACTIVITY_CRREATED:
             case Fragment.STOPPED:
                 if (newState > Fragment.STOPPED) {
                      f.performStart();
                 }
             case Fragment.STARTED:
                 if (newState > Fragment.STARTED) {
                      f.performResume();
                 }
        }
    } else if (f.mState > newState) { //新的状态低于当前状态
        switch(f.mState) {
            case Fragment.RESUMED:
                if (newState < Fragment.RESUMED) {
                    f.performPause();
                }
            case Fragment.STARTED:
                if (newState < Fragment.STARTED) {
                    f.performStop();
                }
            case Fragment.STOPPED::
                if (newState < Fragment.STOPPED) {
                    f.performReallyStop();
                }
            case Fragment.ACTIVITY_CREATED:
                if (newState < Fragment.ACTIVITY_CREATED) {
                    f.performDestroyView(); //调用onDestory()
                    if (f.mView != null && f.mContainer != null) {
                        f.mContainer.removeView(f.mView); //把Fragment的View从视图中移除。
                   }
                }
            case Fragment.CREATED:
                if (newState < Fragment.CREATED) {
                      if (f.mAnimationAway != null) {
                          ...
                      } else {
                           if (!f.mRetaining) {
                               f.performDestory();
                           } else {
                              f.mState = Fragment.INITIALIZING;
                           }
                           f.onDetach();
                           if (!f.mRetaining) {
                               makeInActive(f);//把Fragment从mActive中移除，并把Fragment的所有状态恢复成初始状态，相当于它是一个全新的Fragment。
                           } else {
                               f.mHost = null;
                               f.mParentFragment = null;
                               f.mFragmentManager = null;
                               f.mChildFragmentManager = null;
                           }
                      }
                }
        }
}
```
到 `moveToState `方法中，我们看到了许多熟悉的面孔，就是我们平时最常谈到的 `Fragment` 的生命周期，通过这段代码我们就可以对它的生命周期有更加直观的理解。
