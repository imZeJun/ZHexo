---
title: Loader 知识梳理(1) - LoaderManager初探
date: 2017-03-07 21:08
categories : Loader 知识梳理
---
# 一、概述
刚开始学习`Loader`的时候，只是使用`CursorLoader`把它当作加载封装在`ContentProvider`中的数据的一种方式，但是如果我们学会了如何取定义自己的`Loader`，那么将不仅仅局限于读取`ContentProvider`的数据，在谷歌的蓝图框架中，就有一个分支是介绍如何使用`Loader`来实现数据的异步读取：
> [`https://github.com/googlesamples/android-architecture/tree/todo-mvp-loaders/`](https://github.com/googlesamples/android-architecture/tree/todo-mvp-loaders/)

我们现在来学习一下`Loader`的实现原理，这将帮助我们知道如何自定义自己的`Loader`来进行异步数据的加载。

# 二、`Activity`和`LoaderManager`的桥梁 - `FragmentHostCallback`
如果我们把`Loader`比喻为异步任务的执行者，那么`LoaderManager`就是这些执行者的管理者，而`LoaderManager`对于`Loader`的管理又会依赖于`Activity/Fragment`的生命周期。
在整个系统当中，`LoaderManager`和`Activity/Fragment`之间的关系是通过`FragmentHostCallback`这个中介者维系的，当`Activity`或者`Fragment`的关键生命周期被回调时，会调用`FragmentHostCallback`的对应方法，它再通过内部持有的`LoaderManager`实例来控制每个`LoaderManager`内的`Loader`。
在`FragmentHostCallback`当中，和`Loader`有关的成员变量包括：
```
    /** The loader managers for individual fragments [i.e. Fragment#getLoaderManager()] */
    private ArrayMap<String, LoaderManager> mAllLoaderManagers;
    /** Whether or not fragment loaders should retain their state */
    private boolean mRetainLoaders;
    /** The loader manger for the fragment host [i.e. Activity#getLoaderManager()] */
    private LoaderManagerImpl mLoaderManager;
    private boolean mCheckedForLoaderManager;
    /** Whether or not the fragment host loader manager was started */
    private boolean mLoadersStarted;
```
- `mAllLoaderManagers`：和`Fragment`关联的`LoaderManager`，每个`Fragment`对应一个`LoaderManager`。
- `mRetainLoaders`：**`Fragment`的`Loader`**是否要保持它们的状态。
- `mLoaderManager`：和`Fragment`宿主关联的`LoaderManager`。
- `mCheckedForLoaderManager`：当`Fragment`的宿主的`LoaderManager`被创建以后，该标志位变为`true`。
- `mLoadersStarted`：**`Fragment`的宿主的`Loader`**是否已经启动。

## `FragmentHostCallback`的`doXXX`和`Activity`的对象关系
下面是整理的表格：
- `restoreLoaderNonConfig`  <- `onCreate`
- `reportLoaderStart`  <- `performStart`
- `doLoaderStart` <- `onStart/retainNonConfigurationInstances`
- `doLoaderStop(true/false)`  <- `performStop/retainNonConfigurationInstances`
- `retainLoaderNonConfig`  <- `retainNonConfigurationInstances`
- `doLoaderDestroy`  <- `performDestroy`
- `doLoaderRetain` <- `null`

其中有个函数比较陌生，`retainNonConfigurationInstances`，我们看一下它的含义：
```
    NonConfigurationInstances retainNonConfigurationInstances() {
        Object activity = onRetainNonConfigurationInstance();
        HashMap<String, Object> children = onRetainNonConfigurationChildInstances();
        FragmentManagerNonConfig fragments = mFragments.retainNestedNonConfig();
        
        //由于要求保存loader的状态，所以我们需要标志loader，为此，我们需要在将它交给下个Activity之前重启一下loader
        mFragments.doLoaderStart();
        mFragments.doLoaderStop(true);
        ArrayMap<String, LoaderManager> loaders = mFragments.retainLoaderNonConfig();

        if (activity == null && children == null && fragments == null && loaders == null
                && mVoiceInteractor == null) {
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
我们看到，它保存了大量的信息，最后返回一个`NonConfigurationInstances`，因此我们猜测它和`onSaveInstance`的作用是类似的，在`attach`方法中，传入了`lastNonConfigurationInstances`，之后我们就可以通过`getLastNonConfigurationInstance`来得到它，但是需要注意，这个变量在`performResume`之后就会清空。
通过`ActivityThread`的源码，我们可以看到，这个方法是在`onStop`到`onDestory`之间调用的。
```
//调用onStop()
r.activity.performStop(r.mPreserveWindow);
//调用retainNonConfigurationInstances
r.lastNonConfigurationInstances = r.activity.retainNonConfigurationInstances();
//调用onDestroy().
mInstrumentation.callActivityOnDestroy(r.activity);
```
总结下来，就是一下几点：
- 在`onStart`时启动`Loader`
- 在`onStop`时停止`Loader`
- 在`onDestory`时销毁`Loader`
- 在配置发生变化时保存`Loader`

# 三、`LoaderManager/LoaderManagerImpl`的含义
通过上面，我们就可以了解系统是怎么根据`Activity/Fragment`的生命周期来自动管理`Loader`的了，现在，我们来看一下`LoaderManagerImpl`的具体实现，这两个类的关系是：
- `LoaderManager`：这是一个抽象类，它内部定义了`LoaderCallbacks`接口，在`loader`的状态发生改变时会通过这个回调通知使用者，此外，它还定义了三个关键的抽象方法，调用者只需要使用这三个方法就能完成数据的异步加载。
- `LoaderManagerImpl`：继承于`LoaderManager`，真正地实现了`Loader`的管理。

# 四、`LoaderManager`的接口定义
```
public abstract class LoaderManager {
    /**
     * Callback interface for a client to interact with the manager.
     */
    public interface LoaderCallbacks<D> {
        /**
         * Instantiate and return a new Loader for the given ID.
         *
         * @param id The ID whose loader is to be created.
         * @param args Any arguments supplied by the caller.
         * @return Return a new Loader instance that is ready to start loading.
         */
        public Loader<D> onCreateLoader(int id, Bundle args);

        /**
         * Called when a previously created loader has finished its load.  Note
         * that normally an application is <em>not</em> allowed to commit fragment
         * transactions while in this call, since it can happen after an
         * activity's state is saved.  See {@link FragmentManager#beginTransaction()
         * FragmentManager.openTransaction()} for further discussion on this.
         * 
         * <p>This function is guaranteed to be called prior to the release of
         * the last data that was supplied for this Loader.  At this point
         * you should remove all use of the old data (since it will be released
         * soon), but should not do your own release of the data since its Loader
         * owns it and will take care of that.  The Loader will take care of
         * management of its data so you don't have to.  In particular:
         *
         * <ul>
         * <li> <p>The Loader will monitor for changes to the data, and report
         * them to you through new calls here.  You should not monitor the
         * data yourself.  For example, if the data is a {@link android.database.Cursor}
         * and you place it in a {@link android.widget.CursorAdapter}, use
         * the {@link android.widget.CursorAdapter#CursorAdapter(android.content.Context,
         * android.database.Cursor, int)} constructor <em>without</em> passing
         * in either {@link android.widget.CursorAdapter#FLAG_AUTO_REQUERY}
         * or {@link android.widget.CursorAdapter#FLAG_REGISTER_CONTENT_OBSERVER}
         * (that is, use 0 for the flags argument).  This prevents the CursorAdapter
         * from doing its own observing of the Cursor, which is not needed since
         * when a change happens you will get a new Cursor throw another call
         * here.
         * <li> The Loader will release the data once it knows the application
         * is no longer using it.  For example, if the data is
         * a {@link android.database.Cursor} from a {@link android.content.CursorLoader},
         * you should not call close() on it yourself.  If the Cursor is being placed in a
         * {@link android.widget.CursorAdapter}, you should use the
         * {@link android.widget.CursorAdapter#swapCursor(android.database.Cursor)}
         * method so that the old Cursor is not closed.
         * </ul>
         *
         * @param loader The Loader that has finished.
         * @param data The data generated by the Loader.
         */
        public void onLoadFinished(Loader<D> loader, D data);

        /**
         * Called when a previously created loader is being reset, and thus
         * making its data unavailable.  The application should at this point
         * remove any references it has to the Loader's data.
         *
         * @param loader The Loader that is being reset.
         */
        public void onLoaderReset(Loader<D> loader);
    }
    
    /**
     * Ensures a loader is initialized and active.  If the loader doesn't
     * already exist, one is created and (if the activity/fragment is currently
     * started) starts the loader.  Otherwise the last created
     * loader is re-used.
     *
     * <p>In either case, the given callback is associated with the loader, and
     * will be called as the loader state changes.  If at the point of call
     * the caller is in its started state, and the requested loader
     * already exists and has generated its data, then
     * callback {@link LoaderCallbacks#onLoadFinished} will
     * be called immediately (inside of this function), so you must be prepared
     * for this to happen.
     *
     * @param id A unique identifier for this loader.  Can be whatever you want.
     * Identifiers are scoped to a particular LoaderManager instance.
     * @param args Optional arguments to supply to the loader at construction.
     * If a loader already exists (a new one does not need to be created), this
     * parameter will be ignored and the last arguments continue to be used.
     * @param callback Interface the LoaderManager will call to report about
     * changes in the state of the loader.  Required.
     */
    public abstract <D> Loader<D> initLoader(int id, Bundle args,
            LoaderManager.LoaderCallbacks<D> callback);

    /**
     * Starts a new or restarts an existing {@link android.content.Loader} in
     * this manager, registers the callbacks to it,
     * and (if the activity/fragment is currently started) starts loading it.
     * If a loader with the same id has previously been
     * started it will automatically be destroyed when the new loader completes
     * its work. The callback will be delivered before the old loader
     * is destroyed.
     *
     * @param id A unique identifier for this loader.  Can be whatever you want.
     * Identifiers are scoped to a particular LoaderManager instance.
     * @param args Optional arguments to supply to the loader at construction.
     * @param callback Interface the LoaderManager will call to report about
     * changes in the state of the loader.  Required.
     */
    public abstract <D> Loader<D> restartLoader(int id, Bundle args,
            LoaderManager.LoaderCallbacks<D> callback);

    /**
     * Stops and removes the loader with the given ID.  If this loader
     * had previously reported data to the client through
     * {@link LoaderCallbacks#onLoadFinished(Loader, Object)}, a call
     * will be made to {@link LoaderCallbacks#onLoaderReset(Loader)}.
     */
    public abstract void destroyLoader(int id);

    /**
     * Return the Loader with the given id or null if no matching Loader
     * is found.
     */
    public abstract <D> Loader<D> getLoader(int id);

    /**
     * Returns true if any loaders managed are currently running and have not
     * returned data to the application yet.
     */
    public boolean hasRunningLoaders() { return false; }
}
```
这一部分，我们先根据源码的注释对这些方法有一个大概的了解：
- `public Loader<D> onCreateLoader(int id, Bundle args)`
 - 当`LoaderManager`需要创建一个`Loader`时，回调该函数来要求使用者提供一个`Loader`，而`id`为这个`Loader`的唯一标识。
- `public void onLoadFinished(Loader<D> loader, D data)`
 - 当之前创建的`Loader`完成了任务之后回调，`data`就是得到的数据。
 - 回调时，可能`Activity`已经调用了`onSaveInstanceState`，因此不建议在其中提交`Fragment`事务。
 - 这个方法会保证数据资源在被释放之前调用，例如，当使用`CursorLoader`时，`LoaderManager`会负责`cursor`的关闭。
 - `LoaderManager`会主动监听数据的变化。
- `public void onLoaderReset(Loader<D> loader)`
 - 当先前创建的某个`Loader`被`reset`时回调。
 - 调用者应当在收到该回调以后移除与旧`Loader`有关的数据。

- `public abstract <D> Loader<D> initLoader(int id, Bundle args, LoaderManager.LoaderCallbacks<D> callback)`
 - 用来初始化和激活`Loader`，`args`一般用来放入查询的条件。
 - 如果`id`对应的`Loader`之前不存在，那么会创建一个新的，如果此时`Activity/Fragment`已经处于`started`状态，那么会启动这个`Loader`。
 - 如果`id`对应的`Loader`之前存在，那么会复用之前的`Loader`，并且忽略`Bundle`参数，它仅仅是使用新的`callback`。
 - 如果调用此方法时，满足`2`个条件：调用者处于`started`状态、`Loader`已经存在并且产生了数据，那么`onLoadFinished`会立刻被回调。
 - 这个方法一般来说应该在组件被初始化调用。
- `public abstract <D> Loader<D> restartLoader(int id, Bundle args, LoaderManager.LoaderCallbacks<D> callback)`
 - 启动一个新的`Loader`或者重新启动一个旧的`Loader`，如果此时`Activity/Fragment`已经处于`Started`状态，那么会开始`loading`过程。
 - 如果一个相同`id`的`loader`之前已经存在了，那么当新的`loader`完成工作之后，会销毁旧的`loader`，在旧的`Loader`已经被`destroyed`之前，会回调对应的`callback`。
 - 因为`initLoader`会忽略`Bundle`参数，所以当我们的查询需要依赖于`bundle`内的参数时，那么就需要使用这个方法。
- `public abstract void destroyLoader(int id)`
 - 停止或者移除对应`id`的`loader`。
 - 如果这个`loader`之前已经回调过了`onLoadFinished`方法，那么`onLoaderReset`会被回调，参数就是要销毁的那个`Loader`实例。
- `public abstract <D> Loader<D> getLoader(int id)`
 - 返回对应`id`的`loader`。
- `public boolean hasRunningLoaders()`
 - 是否有正在运行，但是没有返回数据的`loader`。

#五、`LoaderInfo`
`LoaderInfo` 包装了 `Loader`，其中包含了状态变量提供给 `LoaderManager`，并且在构造时候传入了 `LoaderManager.LoaderCallbacks<Object>`，这也是回调给我们调用者的地方，里面的逻辑很复杂，我们主要关注这3个方法在什么时候被调用：
```
    final class LoaderInfo implements Loader.OnLoadCompleteListener<Object>,
            Loader.OnLoadCanceledListener<Object> {
        final int mId; //唯一标识 Loader。
        final Bundle mArgs; //查询参数。
        LoaderManager.LoaderCallbacks<Object> mCallbacks; //给调用者的回调。
        Loader<Object> mLoader;
        boolean mHaveData;
        boolean mDeliveredData;
        Object mData;
        @SuppressWarnings("hiding")
        boolean mStarted;
        @SuppressWarnings("hiding")
        boolean mRetaining;
        @SuppressWarnings("hiding")
        boolean mRetainingStarted;
        boolean mReportNextStart;
        boolean mDestroyed;
        boolean mListenerRegistered;

        LoaderInfo mPendingLoader;
        
        public LoaderInfo(int id, Bundle args, LoaderManager.LoaderCallbacks<Object> callbacks) {
            mId = id;
            mArgs = args;
            mCallbacks = callbacks;
        }
        
        void start() {
            if (mRetaining && mRetainingStarted) {
                //Activity中正在恢复状态，所以我们什么也不做。
                mStarted = true;
                return;
            }
            if (mStarted) {
                //已经开始了，那么返回。
                return;
            }
            mStarted = true;
            //如果Loader没有创建，那么创建让用户去创建它。
            if (mLoader == null && mCallbacks != null) {
                mLoader = mCallbacks.onCreateLoader(mId, mArgs); //onCreateLoader()
            }
            if (mLoader != null) {
                if (mLoader.getClass().isMemberClass()
                        && !Modifier.isStatic(mLoader.getClass().getModifiers())) {
                    throw new IllegalArgumentException(
                            "Object returned from onCreateLoader must not be a non-static inner member class: "
                            + mLoader);
                }
                //注册监听，onLoadCanceled和OnLoadCanceledListener，因为LoaderInfo实现了这两个接口，因此把它自己传进去。
                if (!mListenerRegistered) {
                    mLoader.registerListener(mId, this);
                    mLoader.registerOnLoadCanceledListener(this);
                    mListenerRegistered = true;
                }
                //Loader开始工作。
                mLoader.startLoading();
            }
        }
        
        //恢复之前的状态。
        void retain() {
            if (DEBUG) Log.v(TAG, "  Retaining: " + this);
            mRetaining = true; //正在恢复
            mRetainingStarted = mStarted; //恢复时的状态
            mStarted = false; 
            mCallbacks = null;
        }
        
        //状态恢复完之后调用。
        void finishRetain() {
            if (mRetaining) {
                if (DEBUG) Log.v(TAG, "  Finished Retaining: " + this);
                mRetaining = false;
                if (mStarted != mRetainingStarted) {
                    if (!mStarted) {
                        //如果在恢复完后发现，它已经不处于Started状态，那么停止。
                        stop();
                    }
                }
            }

            if (mStarted && mHaveData && !mReportNextStart) {
                // This loader has retained its data, either completely across
                // a configuration change or just whatever the last data set
                // was after being restarted from a stop, and now at the point of
                // finishing the retain we find we remain started, have
                // our data, and the owner has a new callback...  so
                // let's deliver the data now.
                callOnLoadFinished(mLoader, mData);
            }
        }
        
        void reportStart() {
            if (mStarted) {
                if (mReportNextStart) {
                    mReportNextStart = false;
                    if (mHaveData) {
                        callOnLoadFinished(mLoader, mData);
                    }
                }
            }
        }

        void stop() {
            if (DEBUG) Log.v(TAG, "  Stopping: " + this);
            mStarted = false;
            if (!mRetaining) {
                if (mLoader != null && mListenerRegistered) {
                    // Let the loader know we're done with it
                    mListenerRegistered = false;
                    mLoader.unregisterListener(this);
                    mLoader.unregisterOnLoadCanceledListener(this);
                    mLoader.stopLoading();
                }
            }
        }

        void cancel() {
            if (DEBUG) Log.v(TAG, "  Canceling: " + this);
            if (mStarted && mLoader != null && mListenerRegistered) {
                if (!mLoader.cancelLoad()) {
                    onLoadCanceled(mLoader);
                }
            }
        }

        void destroy() {
            if (DEBUG) Log.v(TAG, "  Destroying: " + this);
            mDestroyed = true;
            boolean needReset = mDeliveredData;
            mDeliveredData = false;
            if (mCallbacks != null && mLoader != null && mHaveData && needReset) {
                if (DEBUG) Log.v(TAG, "  Reseting: " + this);
                String lastBecause = null;
                if (mHost != null) {
                    lastBecause = mHost.mFragmentManager.mNoTransactionsBecause;
                    mHost.mFragmentManager.mNoTransactionsBecause = "onLoaderReset";
                }
                try {
                    mCallbacks.onLoaderReset(mLoader);
                } finally {
                    if (mHost != null) {
                        mHost.mFragmentManager.mNoTransactionsBecause = lastBecause;
                    }
                }
            }
            mCallbacks = null;
            mData = null;
            mHaveData = false;
            if (mLoader != null) {
                if (mListenerRegistered) {
                    mListenerRegistered = false;
                    mLoader.unregisterListener(this);
                    mLoader.unregisterOnLoadCanceledListener(this);
                }
                mLoader.reset();
            }
            if (mPendingLoader != null) {
                mPendingLoader.destroy();
            }
        }

        @Override
        public void onLoadCanceled(Loader<Object> loader) {
            if (DEBUG) Log.v(TAG, "onLoadCanceled: " + this);

            if (mDestroyed) {
                if (DEBUG) Log.v(TAG, "  Ignoring load canceled -- destroyed");
                return;
            }

            if (mLoaders.get(mId) != this) {
                // This cancellation message is not coming from the current active loader.
                // We don't care about it.
                if (DEBUG) Log.v(TAG, "  Ignoring load canceled -- not active");
                return;
            }

            LoaderInfo pending = mPendingLoader;
            if (pending != null) {
                // There is a new request pending and we were just
                // waiting for the old one to cancel or complete before starting
                // it.  So now it is time, switch over to the new loader.
                if (DEBUG) Log.v(TAG, "  Switching to pending loader: " + pending);
                mPendingLoader = null;
                mLoaders.put(mId, null);
                destroy();
                installLoader(pending);
            }
        }

        @Override
        public void onLoadComplete(Loader<Object> loader, Object data) {
            if (DEBUG) Log.v(TAG, "onLoadComplete: " + this);
            
            if (mDestroyed) {
                if (DEBUG) Log.v(TAG, "  Ignoring load complete -- destroyed");
                return;
            }

            if (mLoaders.get(mId) != this) {
                // This data is not coming from the current active loader.
                // We don't care about it.
                if (DEBUG) Log.v(TAG, "  Ignoring load complete -- not active");
                return;
            }
            
            LoaderInfo pending = mPendingLoader;
            if (pending != null) {
                // There is a new request pending and we were just
                // waiting for the old one to complete before starting
                // it.  So now it is time, switch over to the new loader.
                if (DEBUG) Log.v(TAG, "  Switching to pending loader: " + pending);
                mPendingLoader = null;
                mLoaders.put(mId, null);
                destroy();
                installLoader(pending);
                return;
            }
            
            // Notify of the new data so the app can switch out the old data before
            // we try to destroy it.
            if (mData != data || !mHaveData) {
                mData = data;
                mHaveData = true;
                if (mStarted) {
                    callOnLoadFinished(loader, data);
                }
            }

            //if (DEBUG) Log.v(TAG, "  onLoadFinished returned: " + this);

            // We have now given the application the new loader with its
            // loaded data, so it should have stopped using the previous
            // loader.  If there is a previous loader on the inactive list,
            // clean it up.
            LoaderInfo info = mInactiveLoaders.get(mId);
            if (info != null && info != this) {
                info.mDeliveredData = false;
                info.destroy();
                mInactiveLoaders.remove(mId);
            }

            if (mHost != null && !hasRunningLoaders()) {
                mHost.mFragmentManager.startPendingDeferredFragments();
            }
        }

        void callOnLoadFinished(Loader<Object> loader, Object data) {
            if (mCallbacks != null) {
                String lastBecause = null;
                if (mHost != null) {
                    lastBecause = mHost.mFragmentManager.mNoTransactionsBecause;
                    mHost.mFragmentManager.mNoTransactionsBecause = "onLoadFinished";
                }
                try {
                    if (DEBUG) Log.v(TAG, "  onLoadFinished in " + loader + ": "
                            + loader.dataToString(data));
                    mCallbacks.onLoadFinished(loader, data);
                } finally {
                    if (mHost != null) {
                        mHost.mFragmentManager.mNoTransactionsBecause = lastBecause;
                    }
                }
                mDeliveredData = true;
            }
        }
        
    }
```

`onCreateLoader`：在 `start()` 方法中，如果我们发现 `mLoader` 没有创建，那么通知调用者创建它。

`onLoaderReset`：在 `destroy()` 方法中，也就是`Loader`被销毁时调用，它的调用需要满足以下条件：
 - `mHaveData == true：mHaveData` 被置为 `true` 的地方是在 `onLoadComplete` 中判断到有新的数据，并且之前 `mHaveData == false`，在 `onDestroy` 时置为 `false`。
 - `mDeliveredData == true`：它在 `callOnLoadFinished` 时被置为 `true`，成功地回调了调用者的 `onLoadFinished` 
 - 这两个条件一结合，就可以知道这是一个已经递交过数据的`loader`，所以在`destory`的时候，就要通知调用者`loader`被替换了。

# 六、`LoaderManagerImpl`实现的三个关键方法
## 6.1 `initLoader`的实现
```
public <D> Loader<D> initLoader(int id, Bundle args, LoaderManager.LoaderCallbacks<D> callback) {
    //createAndInstallLoader方法正在执行，抛出异常。
    if (mCreatingLoader) {    
        throw new IllegalStateException("Called while creating a loader");
    }
    LoaderInfo info = mLoaders.get(id);
    if (info == null) {
        info = createAndInstallLoader(id, args,  (LoaderManager.LoaderCallbacks<Object>)callback);
    } else {
        info.mCallbacks = (LoaderManager.LoaderCallbacks<Object>)callback;
    }
    //如果已经有数据，并且处于LoaderManager处于Started状态，那么立刻返回。
    if (info.mHaveData && mStarted) {
        info.callOnLoadFinished(info.mLoader, info.mData);
    }
    return (Loader<D>) info.mLoader;
}

private LoaderInfo createAndInstallLoader(int id, Bundle args, LoaderManager.LoaderCallbacks<Object> callback) {    
    try {        
        mCreatingLoader = true; 
        //调用者创建loader，在主线程中执行。       
        LoaderInfo info = createLoader(id, args, callback);        
        installLoader(info);        
        return info;    
    } finally {        
        mCreatingLoader = false;    
    }
}

private LoaderInfo createLoader(int id, Bundle args, LoaderManager.LoaderCallbacks<Object> callback) {    
    LoaderInfo info = new LoaderInfo(id, args,  callback);   
    Loader<Object> loader = callback.onCreateLoader(id, args);    
    info.mLoader = loader;    
    return info;
}

void installLoader(LoaderInfo info) {
    mLoaders.put(info.mId, info);
    //如果已经处于mStarted状态，说明错过了doStart方法，那么只有自己启动了。
    if (mStarted) {
        info.start();
    }
}
```

## 6.2 `restartLoader`的实现
```
    public <D> Loader<D> restartLoader(int id, Bundle args, LoaderManager.LoaderCallbacks<D> callback) {
        if (mCreatingLoader) {
            throw new IllegalStateException("Called while creating a loader");
        }
        
        LoaderInfo info = mLoaders.get(id);
        if (DEBUG) Log.v(TAG, "restartLoader in " + this + ": args=" + args);
        if (info != null) {
            //这个mInactive列表是restartLoader的关键。
            LoaderInfo inactive = mInactiveLoaders.get(id);
            if (inactive != null) {
                //如果info已经有了数据，那么取消它。
                if (info.mHaveData) {
                    if (DEBUG) Log.v(TAG, "  Removing last inactive loader: " + info);
                    inactive.mDeliveredData = false;
                    inactive.destroy();
                    info.mLoader.abandon();
                    mInactiveLoaders.put(id, info);
                } else {
                    //info没有开始，那么直接把它移除。
                    if (!info.mStarted) {
                        if (DEBUG) Log.v(TAG, "  Current loader is stopped; replacing");
                        mLoaders.put(id, null);
                        info.destroy();
                    //info已经开始了。
                    } else {
                        //先取消。
                        info.cancel();
                        if (info.mPendingLoader != null) {
                            if (DEBUG) Log.v(TAG, "  Removing pending loader: " + info.mPendingLoader);
                            info.mPendingLoader.destroy();
                            info.mPendingLoader = null;
                        }
                        //inactive && !mHaveData && mStarted，那么最新的Loader保存在mPendingLoader这个变量当中。
                        info.mPendingLoader = createLoader(id, args, 
                                (LoaderManager.LoaderCallbacks<Object>) callback);
                        return (Loader<D>) info.mPendingLoader.mLoader;
                    }
                }
            //如果调用restartLoader时已经有了相同id的Loader，那么保存在这个列表中进行跟踪。
            } else {
                info.mLoader.abandon();
                mInactiveLoaders.put(id, info);
            }
        }
        
        info = createAndInstallLoader(id, args,  (LoaderManager.LoaderCallbacks<Object>)callback);
        return (Loader<D>) info.mLoader;
    }
```
代码的逻辑比较复杂，我们理一理：
- 在`mLoaders`中不存在相同`id`的`LoaderInfo`情况下，`initLoader`和`restartLoader`的行为是一致的。
- 在`mLoaders`中存在相同`id`的`LoaderInfo`情况下：
 + `initLoader`不会新建`LoaderInfo`，也不会改变`Bundle`的值，仅仅是替换`info.mCallbacks`的实例。
 + `restartLoader`除了会新建一个全新的`Loader`之外，还会有这么一套逻辑，它主要和 `mInactiveLoaders`以及它内部`LoaderInfo`所处的状态有关有关，这个列表用来跟踪调用者希望替换的旧`LoaderInfo`：
   +  如果要被替换的`LoaderInfo`没有被跟踪，那么调用`info.mLoader.abandon()`，再把它加入到跟踪列表，然后会新建一个全新的`LoaderInfo`放入`mLoaders`。
   +  如果要替换的`LoaderInfo`还处在被跟踪的状态，那么再去判断它内部的状态：
     + 已经有数据，调用`info.destroy()`，`info.mLoader.abandon()`，并继续跟踪。
     + 没有数据：
        + 还没有开始，调用`info.destroy()`，直接在`mLoaders`中把对应`id`的位置置为`null`。
        + 已经开始了，那么先`info.cancel()`，然后把新建的`Loader`赋值给`LoaderInfo.mPendingLoader` ，这时候`mLoaders`中就有两个`Loader`了，这是唯一没有新建`LoaderInfo`的情况，即希望替换但是还没有执行完毕的`Loader`以及这个新创建的`Loader`。

## 6.3  `destroyLoader`的实现
```
    public void destroyLoader(int id) {
        if (mCreatingLoader) {
            throw new IllegalStateException("Called while creating a loader");
        }
        
        if (DEBUG) Log.v(TAG, "destroyLoader in " + this + " of " + id);
        int idx = mLoaders.indexOfKey(id);
        if (idx >= 0) {
            LoaderInfo info = mLoaders.valueAt(idx);
            mLoaders.removeAt(idx);
            info.destroy();
        }
        idx = mInactiveLoaders.indexOfKey(id);
        if (idx >= 0) {
            LoaderInfo info = mInactiveLoaders.valueAt(idx);
            mInactiveLoaders.removeAt(idx);
            info.destroy();
        }
        if (mHost != null && !hasRunningLoaders()) {
            mHost.mFragmentManager.startPendingDeferredFragments();
        }
    }
```
