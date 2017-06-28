---
title: Loader 知识梳理(3) - 自定义Loader
date: 2017-03-08 13:16
categories : Loader 知识梳理
---
# 一、概述
前面我们比较关注的是`LoaderManager`的原理和使用，如果我们只是需要通过`LoaderManager`来读取`ContentProvider`封装的数据，那么采用系统已经定义好的`CursorLoader`就可以了，但是有的时候，我们的数据可能不是用`ContentProvider`封装的，那么这时候就要学会如何定义自己的`Loader`。

# 二、`Loader<D>`类
对于`LoaderManager`来说，它所看到的只是一个`Loader`类，并调用它对应的方法，这些方法分为三类：
- 状态标志：`isXXX`
- 调用接口：`startLoading/stopLoading/reset/cancelLoad/forceLoad/onReset`
- 调用接口之后的回调：`onXXX`

```
public class Loader<D> {
    
    public void deliverResult(D data) {
        if (mListener != null) {
            mListener.onLoadComplete(this, data);
        }
    }

    public void deliverCancellation() {
        if (mOnLoadCanceledListener != null) {
            mOnLoadCanceledListener.onLoadCanceled(this);
        }
    }

    //在startLoading执行完，且没有调用stopLoading或reset。
    public boolean isStarted() {
        return mStarted;
    }

    //在该状态下，不应该上报新的数据，并且应该保持着最后一次上报的数据直到被reset。
    public boolean isAbandoned() {
        return mAbandoned;
    }

    //没有启动，或者调用了reset方法。
    public boolean isReset() {
        return mReset;
    }
    
    <!-- 开始任务相关 -->
    //开始任务。    
    public final void startLoading() {
        mStarted = true;
        mReset = false;
        mAbandoned = false;
        onStartLoading();
    }
    
    //开始任务后回调该方法，调用者重写该方法来加载数据。
    protected void onStartLoading() {}

    <!-- 取消任务相关 -->   
   //取消任务。
   //1.返回 false 表示不能取消，有可能是已经完成，或者 startLoading 还没有被调用。
   //2.这不是一个立刻的过程，因为加载是在后台线程中运行的。
   public boolean cancelLoad() {
        return onCancelLoad();
    }
    
    //调用者重写该方法来做取消后的操作。
    protected boolean onCancelLoad() {
        return false;
    }
 
    <!-- 强制刷新数据相关 -->   
    public void forceLoad() {
        onForceLoad();
    }

    protected void onForceLoad() {}

    <!-- 停止任务相关 --> 
    public void stopLoading() {
        mStarted = false;
        onStopLoading();
    }

    protected void onStopLoading() {}
  
   <!-- 废弃任务相关 -->
    public void abandon() {
        mAbandoned = true;
        onAbandon();
    }

    protected void onAbandon() {}

    public void reset() {
        onReset();
        mReset = true;
        mStarted = false;
        mAbandoned = false;
        mContentChanged = false;
        mProcessingChange = false;
    }

    protected void onReset() {}
    
    //得到mContentChanged的值，并把mContentChanged设为false
    public boolean takeContentChanged() {
        boolean res = mContentChanged;
        mContentChanged = false;
        mProcessingChange |= res;
        return res;
    }
    
    //表明正在处理变化。
    public void commitContentChanged() {
        mProcessingChange = false;
    }
    
    //如果正在处理变化，那么停止它，并且把mContentChanged设为true。
    public void rollbackContentChanged() {
        if (mProcessingChange) {
            mContentChanged = true;
        }
    }

    //如果当前是start状态，那么收到变化的通知就立即重新加载，否则记录下这个标志mContentChanged。
    public void onContentChanged() {
        if (mStarted) {
            forceLoad();
        } else {
            mContentChanged = true;
        }
    }

}
```
可以看到，在`Loader<D>`中，提供的是给`LoaderManager`调用的接口，但是它并没有真正地执行操作，而是在`startLoading`等方法之后，提供了`onStartLoading`等回调，让子类在这些回调中去执行对应的操作。
# 三、AsyncTaskLoader<D>
对于`Loader`的使用者来说，它更加希望自己只用处理业务的逻辑，而不用再去关心如何把耗时的任务放到异步线程中。因此系统帮我们实现了一个`Loader`的实现类`AsyncTaskLoader<D>`，里面封装了一个`AsyncTask`用来执行耗时操作。但是它也是一个抽象类，我们并不能直接使用它，而是让子类取实现它的`loadInBackground`方法去处理自己的业务逻辑。
```
public abstract class AsyncTaskLoader<D> extends Loader<D> {
    static final String TAG = "AsyncTaskLoader";
    static final boolean DEBUG = false;

    final class LoadTask extends ModernAsyncTask<Void, Void, D> implements Runnable {
        
        private final CountDownLatch mDone = new CountDownLatch(1);

        //表示该Task已经被post到了Handler当中，用来做延时操作。
        boolean waiting;

        /* Runs on a worker thread */
        @Override
        protected D doInBackground(Void... params) {
            try {
                //在后台获取数据。
                D data = AsyncTaskLoader.this.onLoadInBackground();
                return data;
            //onLoadInBackground表明要取消。
            } catch (OperationCanceledException ex) { 
                if (!isCancelled()) {
                    //LoaderManager仍然期望得到结果，因此继续抛出这个异常。
                    throw ex;
                }
                return null;
            }
        }

        //完成。
        @Override
        protected void onPostExecute(D data) {
            if (DEBUG) Log.v(TAG, this + " onPostExecute");
            try {
                AsyncTaskLoader.this.dispatchOnLoadComplete(this, data);
            } finally {
                mDone.countDown();
            }
        }

        //取消。
        @Override
        protected void onCancelled(D data) {
            if (DEBUG) Log.v(TAG, this + " onCancelled");
            try {
                AsyncTaskLoader.this.dispatchOnCancelled(this, data);
            } finally {
                mDone.countDown();
            }
        }

        //由于实现了Runnable接口，在这里执行executePendingTask。
        @Override
        public void run() {
            waiting = false;
            AsyncTaskLoader.this.executePendingTask();
        }

        //在计数达到0时，一直等待，也就是onPostExecute或onCancelled回调了。
        public void waitForLoader() {
            try {
                mDone.await();
            } catch (InterruptedException e) {
                // Ignore
            }
        }
    }

    private final Executor mExecutor;

    volatile LoadTask mTask;
    volatile LoadTask mCancellingTask;

    long mUpdateThrottle;
    long mLastLoadCompleteTime = -10000;
    Handler mHandler;

    public AsyncTaskLoader(Context context) {
        this(context, ModernAsyncTask.THREAD_POOL_EXECUTOR);
    }

    private AsyncTaskLoader(Context context, Executor executor) {
        super(context);
        mExecutor = executor;
    }

    //从上一次loadInBackground返回，到下次Task执行的时间。
    public void setUpdateThrottle(long delayMS) {
        mUpdateThrottle = delayMS;
        if (delayMS != 0) {
            mHandler = new Handler();
        }
    }

    @Override
    protected void onForceLoad() {
        super.onForceLoad();
        cancelLoad(); //这里会回调onCancelLoad。
        mTask = new LoadTask(); //新建一个Task。
        executePendingTask(); //执行这个Task。
    }

    //cancelLoad后回调。
    @Override
    protected boolean onCancelLoad() {
        //当前没有Task在执行。
        if (mTask != null) {
            //如果正在等待一个被取消的任务执行完毕，那么先取消最后的那个任务。
            if (mCancellingTask != null) {
                if (mTask.waiting) {
                    mTask.waiting = false;
                    mHandler.removeCallbacks(mTask);
                }
                mTask = null;
                return false;
            } else if (mTask.waiting) { //如果当前的任务正在等待被执行，那么直接取消它。
                mTask.waiting = false;
                mHandler.removeCallbacks(mTask);
                mTask = null;
                return false;
            } else {
                //如果当前的任务已经开始执行了，那么先记录下，赋值给mCancellingTask。
                boolean cancelled = mTask.cancel(false);
                if (cancelled) {
                    mCancellingTask = mTask;
                    cancelLoadInBackground();
                }
                mTask = null;
                return cancelled;
            }
        }
        return false;
    }
    
    //dispatchOnCancelled 或者在 dispatchOnLoadComplete 是判断是 abandon 了。
    public void onCanceled(D data) {}

    void executePendingTask() {
        if (mCancellingTask == null && mTask != null) {
            //如果mTask正在等待被执行。
            if (mTask.waiting) {
                mTask.waiting = false; //那么把它从队列中移除。
                mHandler.removeCallbacks(mTask);
            }
            if (mUpdateThrottle > 0) {
                long now = SystemClock.uptimeMillis();
                if (now < (mLastLoadCompleteTime+mUpdateThrottle)) {
                    //放入等待队列当中。
                    mTask.waiting = true;
                    mHandler.postAtTime(mTask, mLastLoadCompleteTime+mUpdateThrottle);
                    return;
                }
            }
            //执行这个任务。
            mTask.executeOnExecutor(mExecutor, (Void[]) null);
        }
    }

    void dispatchOnCancelled(LoadTask task, D data) {
        onCanceled(data); //回调onCanceled.
        if (mCancellingTask == task) { //如果被取消的task执行完了。
            rollbackContentChanged(); 
            mLastLoadCompleteTime = SystemClock.uptimeMillis();
            mCancellingTask = null;
            deliverCancellation(); //通过被取消的task执行完了。
            executePendingTask(); //执行当前的Task。
        }
    }

    void dispatchOnLoadComplete(LoadTask task, D data) {
        if (mTask != task) { //如果执行完的task不是最新的。
            dispatchOnCancelled(task, data);
        } else {
            if (isAbandoned()) { //如果被abandon了。
                onCanceled(data);
            } else {
                commitContentChanged();
                mLastLoadCompleteTime = SystemClock.uptimeMillis();
                mTask = null;
                deliverResult(data);
            }
        }
    }

    public abstract D loadInBackground();
    
    //mTask在后台执行时回调这个方法。
    protected D onLoadInBackground() {
        return loadInBackground();
    }
    
    //mTask在执行过程中被通过cancelLoad()取消了。
    public void cancelLoadInBackground() {}

    public boolean isLoadInBackgroundCanceled() {
        return mCancellingTask != null;
    }

    public void waitForLoader() {
        LoadTask task = mTask;
        if (task != null) {
            task.waitForLoader();
        }
    }

}
```
# 四、自定义`Loader`
关于如何实现`AsyncTaskLoader`，系统提供了一个很好的例子，那就是`CursorLoader`，它用来读取`ContentProvider`中的数据，并支持传入关于查询的`projection/selection/selectionArgs`等条件，最终返回一个`cursor`，在返回之后负责`cursor`的关闭。
为了更加灵活，我们参照它的写法，抽象出其中的关键代码，让使用者最多只需要负责两样东西：
- 数据的加载
- 资源的释放

`BaseLoader`的实现者只需要关注下面这三个方法：
```
    //异步加载数据，这个Bundle就是onCreateLoader时传入的Bundle，我们可以把查询的关键词放在其中，这个是子类必须实现的方法。
    protected abstract Result loadData(Bundle bundle);
    //下面两个方法是用于结果资源的回收，例如cursor的关闭，当我们的返回值只是一些基本数据类型时，并不需要实现它。
    protected void releaseResult(Result result) {}
    protected boolean isResultReleased(Result result) { return true; }
```
下面是全部的实现代码：
```
public abstract class BaseDataLoader<Result> extends AsyncTaskLoader<Result> {

    Result mResult;
    Bundle mBundles;
    CancellationSignal mCancellationSignal;

    //所有的子类最多只需要实现下面的三个方法，而不要回收资源的只用实现loadData就可以了。
    protected abstract Result loadData(Bundle bundle);
    protected void releaseResult(Result result) {}
    protected boolean isResultReleased(Result result) { return true; }

    public BaseDataLoader(Context context, Bundle bundle) {
        super(context);
        mBundles = bundle;
    }

    @Override
    public Result loadInBackground() {
        synchronized (this) {
            if (isLoadInBackgroundCanceled()) {
                throw new OperationCanceledException();
            }
            mCancellationSignal = new CancellationSignal();
        }
        try {
            return loadData(mBundles);
        } finally {
            synchronized (this) {
                mCancellationSignal = null;
            }
        }
    }

    @Override
    public void cancelLoadInBackground() {
        super.cancelLoadInBackground();
        synchronized (this) {
            if (mCancellationSignal != null) {
                mCancellationSignal.cancel();
            }
        }
    }

    @Override
    public void deliverResult(Result result) {
        if (isReset()) {
            if (result != null && !isResultReleased(result)) {
                releaseResult(result);
            }
            return;
        }
        Result oldResult = mResult;
        mResult = result;
        if (isStarted()) {
            super.deliverResult(result);
        }
        if (oldResult != null && oldResult != result && !isResultReleased(oldResult)) {
            releaseResult(oldResult);
        }
    }

    @Override
    protected void onStartLoading() {
        if (mResult != null) {
            deliverResult(mResult);
        }
        if (takeContentChanged() || mResult == null) {
            forceLoad();
        }
    }

    @Override
    protected void onStopLoading() {
        cancelLoad();
    }

    @Override
    public void onCanceled(Result result) {
        if (result != null && !isResultReleased(result)) {
            releaseResult(result);
        }
    }

    @Override
    protected void onReset() {
        super.onReset();
        onStopLoading();
        if (mResult != null && !isResultReleased(mResult)) {
            releaseResult(mResult);
        }
        mResult = null;
    }

}
```
