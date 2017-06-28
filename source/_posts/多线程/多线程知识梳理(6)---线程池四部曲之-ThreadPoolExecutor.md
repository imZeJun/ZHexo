---
title: 多线程知识梳理(6) - 线程池四部曲之 ThreadPoolExecutor
date: 2017-05-09 21:59
categories : 多线程知识梳理
---
# 一、ThreadPoolExecutor 简介
## 1.1 优点
在 [多线程知识梳理(5) - 线程池四部曲之 Executor 框架](http://www.jianshu.com/p/80baeaf4eb32) 中，我们对`Executor`框架以及它的调度模型进行了简要的介绍，其中用于对线程进行调度和管理的线程池是整个框架的核心，通过线程池我们可以：
- 重复利用已经创建的线程降低线程创建和销毁造成的消耗。
- 当任务到达时，任务可以不需要等到线程创建就能够立即执行，提高响应速度。
- 利用线程池对线程进行统一分配、调优和监控，提供线程的可管理性。

## 1.2 处理流程
在`JDK`包中，`ThreadPoolExecutor`就是线程池的具体实现，在阅读源码之前，我们先对它的处理流程进行简要介绍，当我们通过`execute/submit`方法提交一个任务到线程池后，会经过以下的处理流程：
![](http://upload-images.jianshu.io/upload_images/1949836-aca0b0febc6d67f7.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
1) 如果当前运行的线程小于 `corePoolSize`，则创建新线程来执行任务。
2) 如果运行的线程等于或多于 `corePoolSize`，则将任务加入到等待队列中。
3) 如果无法将任务加入到等待队列，则继续创建新的线程来执行任务。
4) 如果创建新线程使得当前运行的线程超过`maximumPoolSize`，任务将被拒绝。

以上就是`ThreadPoolExecutor`对于任务的处理流程，其中有几点需要说明：
- 当创建一个新线程来执行任务时，需要获取全局锁，而如果仅仅是将任务加入到等待队列中则不需要，
- 当新线程执行完创建它时所指派的第一个任务之后，并不会马上退出，它会反复从等待队列中获取新的任务来执行。
- 如果一个线程在指定的时间内一直没有获取到新任务，那么我们会根据当前线程池当中活动的线程数量来决定是否销毁它：如果当前线程池数量大于`corePoolSize`，那么销毁该线程，否则只有当设置了`allowCoreThreadTimeOut`，才会销毁该线程。

# 二、ThreadPoolExecutor 实现
## 2.1 参数配置
从上面的处理流程可以看出，`ThreadPoolExecutor`对于任务的处理流程，会受到`corePoolSize`、等待队列、`maximumPoolSize`等参数的影响，而这些参数都是可以由`ThreadPoolExecutor`的创建者去指定的，正是鉴于这种灵活性，使得我们仅仅通过简单的配置就可以实现适用于不同的场景的`ThreadPoolExecutor`，下面，我们就来介绍一一介绍这些参数的含义：
```
public ThreadPoolExecutor(int corePoolSize,
                              int maximumPoolSize,
                              long keepAliveTime,
                              TimeUnit unit,
                              BlockingQueue<Runnable> workQueue,
                              ThreadFactory threadFactory,
                              RejectedExecutionHandler handler)
```
**(1) int corePoolSize**

指定核心线程池的大小，当线程的数量没有大于`corePoolSize`之前，始终会创建新线程来执行分配的任务。如果调用了`preStartAllCoreThreads()`方法，线程池会提前创建并启动所有核心线程。

**(2) int maximumPoolSize**

指定线程池的最大数量，当加入新任务时，如果发现等待队列已经满了，那么我们会尝试通过创建新线程的方式来执行该任务，而如果此时线程池内线程的数量已经等于`maximumPoolSize`，那么会采用指定的拒绝策略来处理该任务。

**(3) long keepAliveTime 和 TimeUnit unit**

当一个线程在执行完分配给它的任务之后，会尝试从等待队列中取出任务去执行，如果经过`keepAliveTime`之后仍然不能从队列中获取到任务，说明此时系统中可能并没有那么多的任务需要去处理，那么就会根据线程池此时的状态来决定是否销毁该线程，以保证在能够迅速响应任务的同时，又不至于有太多空闲的存活线程。

**(4) BlockingQueue<Runnable> workQueue**

指定等待队列的实现方式，我们可以根据需要选择以下几种等待队列：
- `ArrayBlockingQueue`：基于数组结构的有界等待队列，按先进先出原则排序任务
- `LinkedBlockingQueue`：基于链表结构的阻塞队列，同样按照先进先出原则排序任务，吞吐量要高于`ArrayBlockingQueue`
- `SynchronousQueue`：对于这种阻塞队列而言，每个插入操作必须要等到另一个线程调用移除操作，否则插入操作一直处于阻塞状态。
- `PriorityBlockingQueue`：一个具有优先级的无限阻塞队列。

**(5) ThreadFactory threadFactory**

用于创建线程的工厂。

**(6) RejectedExecutionHandler handler**

系统内置了以下几种策略，用于队列和线程池都满了的情况：
- `AbortPolicy`：抛出异常，这也是默认的策略
- `CallerRunsPolicy`：使用调用者所在线程来执行任务
- `DiscardOldestPolicy`：先丢弃队列中最末尾的任务，再重新通过`execute`方法执行该任务。
- `DiscardPolicy`：不做任何处理，直接丢弃

## 2.2 内置 ThreadPoolExecutor 
在 [多线程知识梳理(5) - 线程池四部曲之 Executor 框架](http://www.jianshu.com/p/80baeaf4eb32) 中，我们介绍了几种`ThreadPoolExecutor`的实现：
![](http://upload-images.jianshu.io/upload_images/1949836-0be76c6302873a50.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
它们其实都是`ThreadPoolExecutor`，只是在构造时传入了不同的参数，如下表所示：
![](http://upload-images.jianshu.io/upload_images/1949836-3eb034651bb1551b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
结合之前对于处理流程和核心参数的分析，对它们进行进一步的介绍：

**(1) FixedThreadPool**
- 传入的`nThread`参数将被作为核心线程数和最大线程数，当线程池的数量达到`nThread`后，之后的任务将会被加入到无界的等待队列当中
- 除非某个线程因为异常而结束，否则当线程池的数量达到`nThread`之后将会一直保持不变
- 由于使用的是无界队列，因此线程池不会拒绝任务

**(2) SingleThreadPoolExecutor**
- 如果当前线程池中无运行的线程时，将创建一个新线程来执行任务
- 由于最大线程数被设置为`1`，因此之后的任务都被加入到无界队列当中，并且由线程池中这个唯一的线程从等待队列中，按照添加的顺序依次执行任务

**(3) CachedThreadPool**
- 由于等待队列使用的是`SynchonousQueue`，它的每个插入操作都必须等待另一个线程的移除操作，对于线程池而言，也就是说：在添加任务到等待队列时，必须要有一个空闲线程正在尝试从等待队列获取任务，才有可能添加成功。
- 因此，当一个任务被添加进入线程池时，会有以下两种情况：
 - 如果当前有空闲线程正在尝试从等待队列中获取任务，那么这个任务将会被交给这个空闲线程进行处理
 - 如果当前没有空闲线程尝试从等待队列中获取任务，那么将会创建一个新线程来执行任务
- 由于设置了等待超时时间，因此某个线程在`60s`内都无法获取到新的任务，那么它将会被销毁。

# 三、ThreadPoolExecutor 源码走读
## 3.1 ctl 
在`ThreadPoolExector`中，有一个关键变量 - `ctl`，理解它是我们进行源码走读的基础。
```
private final AtomicInteger ctl = new AtomicInteger(ctlOf(RUNNING, 0))
```
这个原子操作类包含了两部分信息：**线程池的状态**、**线程池中的存活线程数目**，它们用`32`位的整型数来表示，其中高`3`位表示线程池的状态，低位表示当前线程池中存活的线程数。

在某一时刻，线程池会处于以下五种状态之一：
- `RUNNING`：`Accept new tasks and process queued tasks`
- `SHUTDOWN`：`Don't accept new tasks, but process queued tasks`
- `STOP`：`Don't accept new tasks, don't process queued tasks, and interrupt in-progress tasks`
- `TIDYING`：`All tasks have terminated, workerCount is zero, the thread transitioning to state TIDYING will run the terminated() hook method`
- `TERMINATED`：`terminated() has completed`

这五种状态之间转换转换图为：
![](http://upload-images.jianshu.io/upload_images/1949836-715e0672bd284b2c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

对于`ctl`变量，以下三个函数可以用来拆解和组装：
```
//获取线程池的状态信息
private static int runStateOf(int c)     { return c & ~CAPACITY; }
//获取线程池的存活线程数
private static int workerCountOf(int c)  { return c & CAPACITY; }
//将状态信息和存活线程数进行组合
private static int ctlOf(int rs, int wc) { return rs | wc; }
```

## 3.2 任务执行过程
下面，我们通过模拟一个任务的执行来对`ThreadPoolExecutor`的源码进行简单的走读，整个流程如下图所示，红色字部分为我们所要关注的关键方法：
![](http://upload-images.jianshu.io/upload_images/1949836-699a13da01269de3.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
**(1) public void execute(Runnable command)**
```
    public void execute(Runnable command) {
        if (command == null)
            throw new NullPointerException();
        int c = ctl.get();
        if (workerCountOf(c) < corePoolSize) {
            if (addWorker(command, true))
                return;
            c = ctl.get();
        }
        if (isRunning(c) && workQueue.offer(command)) {
            int recheck = ctl.get();
            if (! isRunning(recheck) && remove(command))
                reject(command);
            else if (workerCountOf(recheck) == 0)
                addWorker(null, false);
        } else if (!addWorker(command, false)) {
            reject(command);
        }
    }
```
前面我们说过，向一个线程池中提交任务有两种方法，`execute/submit`，它们最终都会调用到上面的这个`execute`当中，这一函数的逻辑分为三步：
- 当线程池中的存活线程数小于指定的核心线程数时，尝试通过`addWorker(firstTask, core)`创建一个新的线程来执行任务，这里将传入的`runnable`作为该线程的第一个任务，并且`core`参数为`true`，如果创建成功，那么直接返回，否则重新获取一次`ctl`变量，跳转到步骤`2`
- 接着通过`ctl`变量判断如果当前线程池处于`running`状态，那么将`runnable`添加到等待队列`workQueue`当中，如果添加失败跳转到步骤`3`，添加成功则进行二次检查，当发现了下面这两种情况之一，那么还需要进行额外的处理：
 - 如果发现线程池变为了非`running`状态，那么会将该任务从等待队列中移除；
 - 如果当前线程池已经没有存活的线程，那么为了让等待队列中的任务可以运行，我们需要通过`addWorker`方法启动一个新线程，与第一步不同的是，该线程的第一个任务为空。
- 通过`addWorker`方法创建新线程来执行该任务，和第一步的唯一区别就是`core`参数为`false`，如果创建失败，那么执行拒绝策略。

**(2) private boolean addWorker(Runnable firstTask, boolean core)**
下面，我们再来看一下这个核心的函数`addWorker`，它的最终目的就是创建一个新的线程来执行任务：
```
   private boolean addWorker(Runnable firstTask, boolean core) {
        retry:
        for (;;) {
            int c = ctl.get();
            int rs = runStateOf(c);

            //第一部分：当前线程池的状态是否满足加入的条件
            if (rs >= SHUTDOWN &&
                ! (rs == SHUTDOWN &&
                   firstTask == null &&
                   ! workQueue.isEmpty()))
                return false;
            //第二部分：当前线程池的容量是否满足加入的条件
            for (;;) {
                int wc = workerCountOf(c);
                if (wc >= CAPACITY ||
                    wc >= (core ? corePoolSize : maximumPoolSize))
                    return false;
                if (compareAndIncrementWorkerCount(c))
                    break retry;
                c = ctl.get();  // Re-read ctl
                if (runStateOf(c) != rs)
                    continue retry;
                // else CAS failed due to workerCount change; retry inner loop
            }
        }

        boolean workerStarted = false;
        boolean workerAdded = false;
        Worker w = null;
        try {
            //第三部分：创建工作类Worker
            w = new Worker(firstTask);
            final Thread t = w.thread;
            if (t != null) {
                final ReentrantLock mainLock = this.mainLock;
                mainLock.lock();
                try {
                    // Recheck while holding lock.
                    // Back out on ThreadFactory failure or if
                    // shut down before lock acquired.
                    int rs = runStateOf(ctl.get());

                    if (rs < SHUTDOWN ||
                        (rs == SHUTDOWN && firstTask == null)) {
                        if (t.isAlive()) 
                            throw new IllegalThreadStateException();
                        //第四部分：将Worker加入到线程池中
                        workers.add(w);
                        int s = workers.size();
                        if (s > largestPoolSize)
                            largestPoolSize = s;
                        workerAdded = true;
                    }
                } finally {
                    mainLock.unlock();
                }
                if (workerAdded) {
                    //第五部分：启动线程
                    t.start();
                    workerStarted = true;
                }
            }
        } finally {
            if (! workerStarted)
                addWorkerFailed(w);
        }
        return workerStarted;
    }
```
整个`addWorker`进行了以下几步操作：
- 根据当前线程池的状态，判断是否允许新建线程
- 根据当前线程池的工作线程数，判断是否允许新建线程
- 创建一个`Worker`对象，这个`Worker`类中包含了一个线程
```
        Worker(Runnable firstTask) {
            setState(-1); 
            this.firstTask = firstTask;
            this.thread = getThreadFactory().newThread(this);
        }
```
- 将新建的`Worker`对象加入到线程池中
- 启动`Worker`中的线程

**(3) final void runWorker(Worker w)**

在第`(2)`步中，我们启动了`Worker`对象中的线程`t`，它会调用`Worker`对象的`run()`函数，接着会执行`runWorker`方法：
```
final void runWorker(Worker w) {
        Thread wt = Thread.currentThread();
        Runnable task = w.firstTask;
        w.firstTask = null;
        w.unlock(); 
        boolean completedAbruptly = true;
        try {
            while (task != null || (task = getTask()) != null) {
                w.lock();
                if ((runStateAtLeast(ctl.get(), STOP) ||
                     (Thread.interrupted() &&
                      runStateAtLeast(ctl.get(), STOP))) &&
                    !wt.isInterrupted())
                    wt.interrupt();
                try {
                    beforeExecute(wt, task);
                    Throwable thrown = null;
                    try {
                        task.run();
                    } catch (RuntimeException x) {
                        thrown = x; throw x;
                    } catch (Error x) {
                        thrown = x; throw x;
                    } catch (Throwable x) {
                        thrown = x; throw new Error(x);
                    } finally {
                        afterExecute(task, thrown);
                    }
                } finally {
                    task = null;
                    w.completedTasks++;
                    w.unlock();
                }
            }
            completedAbruptly = false;
        } finally {
            processWorkerExit(w, completedAbruptly);
        }
    }
```
这里面的`task`就是我们通过`execute`方法传入的`Runnable`，如果`Worker`的第一个任务不为空，那么会首先执行该任务，如果第一个任务执行完毕，那么会调用`getTask()`方法来尝试去获取下一个任务，当`getTask()`方法不返回（等待队列为空）时，会一直阻塞在这里，而当这个`while`循环退出的时候，那么`Worker`所对应的线程就会被销毁。

**(4) private Runnable getTask()**

```
private Runnable getTask() {
        boolean timedOut = false; // Did the last poll() time out?

        for (;;) {
            int c = ctl.get();
            int rs = runStateOf(c);

            // Check if queue empty only if necessary.
            if (rs >= SHUTDOWN && (rs >= STOP || workQueue.isEmpty())) {
                decrementWorkerCount();
                return null;
            }

            int wc = workerCountOf(c);

            // Are workers subject to culling?
            boolean timed = allowCoreThreadTimeOut || wc > corePoolSize;

            if ((wc > maximumPoolSize || (timed && timedOut))
                && (wc > 1 || workQueue.isEmpty())) {
                if (compareAndDecrementWorkerCount(c))
                    return null;
                continue;
            }

            try {
                Runnable r = timed ?
                    workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS) :
                    workQueue.take();
                if (r != null)
                    return r;
                timedOut = true;
            } catch (InterruptedException retry) {
                timedOut = false;
            }
        }
    }
```
`getTask()`方法会去在第`(1)`步中的等待队列`workerQueue`取任务，在获取任务的时候会考虑超时时间`keepAliveTime`，如果超时时间到了仍然没有获取到任务，那么`getTask()`方法就会返回`null`，从而`runWorker()`中的`while`循环就会结束，之后在`finally`代码块中通过`processWorkerExit(w, completedAbruptly)`销毁该线程。

# 四、关闭线程池
关闭线程池有两种方法：`shutdown`和`shutdownNow`。
- `shutdown`：将线程池的状态设置成`SHUTDOWN`状态，然后中断所有没有正在执行任务的线程。
- `shutdownNow`：将线程池的状态设置为`STOP`，尝试停止所有正在执行或暂停任务的线程，并返回等待执行任务的列表。
