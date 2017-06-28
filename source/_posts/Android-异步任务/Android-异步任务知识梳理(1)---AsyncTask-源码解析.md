---
title: Android 异步任务知识梳理(1) - AsyncTask 解析
date: 2017-02-20 21:43
categories : Android 异步任务知识梳理
---
# 一、概述
这篇文章中，让我们从源码的角度看一下`AsyncTask`的原理，最后会根据原理总结一下使用`AsyncTask`中需要注意的点。

# 二、源码解析
在`AsyncTask`中，有一个线程池 `THREAD_POOL_EXECUTOR` 和与这个线程池相关联的 `Executor`，它负责执行我们的任务`Runnable`：
```
private static final int CPU_COUNT = Runtime.getRuntime().availableProcessors();
private static final int CORE_POOL_SIZE = CPU_COUNT + 1;
private static final int MAXIMUM_POOL_SIZE = CPU_COUNT * 2 + 1;
private static final int KEEP_ALIVE = 1;

private static final ThreadFactory sThreadFactory = new ThreadFactory() {    
    private final AtomicInteger mCount = new AtomicInteger(1);    
    public Thread newThread(Runnable r) {        
        return new Thread(r, "AsyncTask #" + mCount.getAndIncrement());    
    }
};

private static final BlockingQueue<Runnable> sPoolWorkQueue = new LinkedBlockingQueue<Runnable>(128);

public static final Executor THREAD_POOL_EXECUTOR 
    = new ThreadPoolExecutor(CORE_POOL_SIZE, MAXIMUM_POOL_SIZE, KEEP_ALIVE, TimeUnit.SECONDS, sPoolWorkQueue, sThreadFactory);

public static final Executor SERIAL_EXECUTOR = new SerialExecutor();

private static volatile Executor sDefaultExecutor = SERIAL_EXECUTOR;

//这是执行Runnable的地方，调用execute会执行它，如果当前已经有一个任务在执行，那么就是把它放到队列当中。
private static class SerialExecutor implements Executor {    
    final ArrayDeque<Runnable> mTasks = new ArrayDeque<Runnable>();    
    Runnable mActive;    
    public synchronized void execute(final Runnable r) {        
        mTasks.offer(new Runnable() {                
             public void run() {                
                 try {                    
                     r.run();                
                } finally {
                    //判断队列当中是否还有未执行的任务。                    
                    scheduleNext();                
                }            
             }        
        });
        //如果为null，那么立刻执行它；
        //如果不为null，说明当前队列中还有任务，那么等Runnable执行完之后，再由上面的scheduleNext()执行它。        
        if (mActive == null) {               
            scheduleNext();        
        }    
   }    

   protected synchronized void scheduleNext() {
        //第一次进来调用了offer方法，因此会走进去，执行上面Runnable的run()方法。        
        if ((mActive = mTasks.poll()) != null) {            
           THREAD_POOL_EXECUTOR.execute(mActive);        
        }    
    }
}
```
从上面可以看出，每次调用`sDefaultExecutor.execute`时就是执行一个任务，这个任务会被加入到`ArrayDeque`中串行执行，我们看一下当我们调用`AsyncTask`的`execute`方法时，任务是如何被创建并加入到这个队列当中的：
```
//这个静态方法，不会回调 loadInBackground 等方法。
public static void execute(Runnable runnable) {    
    sDefaultExecutor.execute(runnable);
}

//这是我们最常使用的方法，参数可以由调用者任意指定。
public final AsyncTask<Params, Progress, Result> execute(Params... params) {    
    return executeOnExecutor(sDefaultExecutor, params);
}

public final AsyncTask<Params, Progress, Result> executeOnExecutor(Executor exec, Params... params) {
    //判断当前是否有未完成的任务。    
    if (mStatus != Status.PENDING) {        
        switch (mStatus) {            
            case RUNNING:                
                throw new IllegalStateException("Cannot execute task:" + " the task is already running.");            
           case FINISHED:                
                throw new IllegalStateException("Cannot execute task:" + " the task has already been executed " + "(a task can be executed only once)");        
          }    
      }
      //表明正在运行。    
      mStatus = Status.RUNNING;
      //通知调用者它准备要执行任务了。    
      onPreExecute();    
      mWorker.mParams = params;    
      exec.execute(mFuture);    
      return this;
}
```
在调用了`executeOnExecutor`之后，我们把传入的参数赋值给了`mWorker`，并把`mFuture`传入给`Executor`去执行，而从下面我们可以看到`mFuture`的构造函数中传入的参数正是`mWorker`，这两个东西其实才是`AsyncTask`的核心，它们是在`AsyncTask`的构造函数中被初始化的：
```
private final WorkerRunnable<Params, Result> mWorker;
private final FutureTask<Result> mFuture;

mWorker = new WorkerRunnable<Params, Result>() {    
    public Result call() throws Exception {        
        mTaskInvoked.set(true);        
        Process.setThreadPriority(Process.THREAD_PRIORITY_BACKGROUND);        
        //noinspection unchecked        
        Result result = doInBackground(mParams);        
        Binder.flushPendingCommands();         
        return postResult(result);    
    }
};

mFuture = new FutureTask<Result>(mWorker) {    
    @Override    
    protected void done() {        
        try {
            //要求mWorker的call方法没有被调用，否则什么也不做。            
            postResultIfNotInvoked(get());        
        } catch (InterruptedException e) {            
            android.util.Log.w(LOG_TAG, e);         
       } catch (ExecutionException e) {            
            throw new RuntimeException("An error occurred while executing doInBackground()",                    e.getCause());        
       } catch (CancellationException e) {            
            postResultIfNotInvoked(null);        
       }    
   }
};
```
先看一下`WorkerRunnable`，实现了`Callable<V>`接口，增加了一个不定参数的成员变量用来传给 `doInBackground`，这个不定参数就是我们在`execute`时传入的，调用`call`时会执行器内部的函数，而`call` 时会调用`doInBackground`方法，这个方法执行完之后，调用`postResult`，注意到`call`和`done`都是在子线程当中执行的：
```
private static abstract class WorkerRunnable<Params, Result> implements Callable<Result> {    
    Params[] mParams;
}
```
我们主要看一下`FutureTask`，还记得最开始我们的`Executor`最终执行的是传入的`Runnable`的`run`方法，因此我们直接看一下它的`run`方法中做了什么：
```
public interface Runnable {
   public void run();
}

public interface Future<V> {
    boolean cancel(boolean mayInterruptIfRunning);
    boolean isCancelled();
    boolean isDone();
    V get() throws InterruptedException, ExecutionException;
    V get(long timeout, TimeUnit unit) throws InterruptedException, ExecutionException, TimeoutException;
}

public interface RunnableFuture<V> extends Runnable, Future<V> { 
    void run();
}

//我们只要关注run方法
public class FutureTask<V> implements RunnableFuture<V> {
    public void run() {
        Callable<V> c = callable;
        if (c != null && state == NEW) {
            result = c.call(); //mWorker.call()
            ran = true;
        }
        if (ran) {
            set(result);
        }
    }

    protected void set(V v) {
        finishCompletion();
    }

    private void finishCompletion() {
        done(); //mFuture.done()
        callable = null;
    }
}
```
那么结论就清晰了，整个的执行过程是下面这样的：
- 首先调用`excuter.execute(mFuture)`，把`mFuture`加入到队列当中
- 当`mFuture`得到执行时，会回调`mFuture`的`run`方法
- `mFuture#run`是运行在子线程当中的，它在它所在的线程中执行的`mWorker#call`方法
- `mWorkder#call`调用了`doInBackground`，用户通过实现这个抽象方法来进行耗时操作
- `mFuture#call`执行完后调用`mFuture#done`

在上面的过程当中，有两个地方都调用了`postResult`，一个是`mWorkder#call`的最后，另一个是`mFuture#done`，但是区别在于后者在调用之前会判断`mTaskInvoked`为`false`时才会去执行，也就是在`mWorkder#call`没有执行的情况下，这是为了避免`call`方法没有被执行时（提前取消了任务），`postResult`没有被执行，导致使用者收不到任何回调。
`postResult`会通过`InternalHandler`把当前的`AsyncTask`和`FutureTask`的结果回调到主线程当中，之后调用`finish`方法，它会根据调用者是否执行过`cancel`方法来回调不同的函数：
```
private void finish(Result result) {    
    if (isCancelled()) {        
        onCancelled(result);     
    } else {        
        onPostExecute(result); 
    }    
    mStatus = Status.FINISHED;
}
```
调用者通过重写`onProgressUpdate`就可以得到当前的最新进度，`AsyncTask`最终会把结果回调到主线程当中：
```
protected final void publishProgress(Progress... values) {    
    if (!isCancelled()) {          
        getHandler().obtainMessage(MESSAGE_POST_PROGRESS, new AsyncTaskResult<Progress>(this, values)).sendToTarget();    
    }
}

private static class InternalHandler extends Handler {
    public void handleMessage(Message msg) {
        AsyncTaskResult<?> result = (AsyncTaskResult<?>) msg.obj;
        switch (msg.what) {    
             case MESSAGE_POST_RESULT:        // There is only one result            
                 result.mTask.finish(result.mData[0]);        
                 break;    
             case MESSAGE_POST_PROGRESS:        
                 result.mTask.onProgressUpdate(result.mData);        
                 break;
        }
    }
}
```
# 三、结论
经过上面的源码分析，我们有这么几个结论：
- 静态方法`execute(Runnable runnable)`和`AsyncTask`其实没什么太大关系，只是用到了里面一个静态的线程池而已，`AsyncTask`内部的状态都和它无关。
- 当我们对同一个`AsyncTask`实例调用`execute(..)`时，如果此时已经有任务正在执行，或者已经执行过任务，那么会抛出异常。
- 在`onPreExecute()`执行时，任务还没有被加入到队列当中。
- `doInBackground`是在子线程当中执行的，我们调用`cancel`后，并不一定会立即得到`onCancel`的回调，这是因为`cancel`只保证了`mFuture`的`done`方法被执行，有这么几种情况：
 + 如果`mWorker`的`call`函数没有执行，那么这时`mFuture`的`done`方法被调用时，`postResultIfNotInvoked`是满足条件的，调用者可以立即得到`onCancel`回调。
 + 如果`mWorker`的`call`调用了，虽然`mFuture`的`done`执行了，但是它不满足条件`(!mTaskInvoked.get())`，那么会一直等到`doInBackground`执行完所有的操作才通过`return postResult`返回，所以我们需要在`doInBackground`中通过`isCancel()`来判断是否需要提早返回，避免无用的等待。
- 在`postResult`完毕之后， `onCancel`和`onPostExecute`只会调用一个。
- 任务是和`AsyncTask`实例绑定的，而如果`AsyncTask`又和`Activity`绑定了，如果在执行过程中这个 `AsyncTask`实例被销毁了（例如`Activity`被重新创建），那么调用者在新的`Activity`中是无法收到任何回调的，因为那已经是另外一个`AsyncTask`了。
- 关于`AsyncTask`最大的问题其实是内存泄漏，因为把它作为`Activity`的内部类时，会默认持有`Activity`的引用，那么这时候如果有任务正在执行，那么 Activity 是无法被销毁的，这其实和`Handler`的泄漏类似，可以有以下这么一些用法：
 + 不让它持有`Activity`的引用或者持有弱引用。
 + 保证在`Activity`在需要销毁时`cancel`了所有的任务。 
 + 使`AsyncTask`的执行与`Activity`的生命周期无关，可以考虑通过建立一个没有`UI`的`fragment`来实现，因为在`Activity`重启时，会自动保存有之前`add`进去的`Fragment`的实例，`Fragment`持有的还是先前的 `AsyncTask`。
 + 使用`LoaderManager`，把管理的活交给系统来执行。
