---
title: 多线程知识梳理(5) - 线程池四部曲之 Executor 框架
date: 2017-05-07 00:12
categories : 多线程知识梳理
---
# 一、Executor 框架的调度模型
## 1.1 目的
在平时的开发中，我们经常需要将一些耗时的任务放到异步线程当中进行处理，而线程的创建和销毁都是需要耗费资源的，设计`Executor`框架的目的就是为了在上层能够对这些异步任务进行有效地管理和调度。

## 1.2 调度模型
我们可以把`Executor`框架想象成一个公司，开发者所需要完成的任务则是一个订单，那么任务的执行可以分解为下面这个过程：
- 开发者将订单提交给公司
- 公司把订单指派给对应的员工
- 公司员工将订单交给工厂
- 工厂员工执行订单

订单、公司、公司员工、工厂和工厂员工这几个概念的映射如下图所示：
![](http://upload-images.jianshu.io/upload_images/1949836-ec7a0386ae3d7e4a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

`Executor`框架对于内部线程的调度过程，就可以理解为公司对于员工的管理机制，而这一过程对于开发者是不可见的。

# 二、Executor 框架
整个`Executor`的框架主要由三个部分组成：
- `FutureTask`：任务
- `ThreadPoolExecutor`：常规任务管理者
- `ScheduledThreadPoolExecutor`：周期任务管理者

下面，我们就分这三个部分来简要介绍一下涉及到的相关类。
## 2.1 任务
与任务相关的类的继承关系如下图所示：
![](http://upload-images.jianshu.io/upload_images/1949836-0403d0ff079a2b6f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

当我们向`ThreadPoolExecutor`或者`ScheduledThreadPoolExecutor`提交任务时，有以下四种方式：
```
public void execute(Runnable command)

public Future<?> submit(Runnable task)
public <T> Future<T> submit(Runnable task, T result)
public <T> Future<T> submit(Callable<T> task)
```
- 当通过`execute`方法提交一个`Runnable`的实现类时，不会得到返回的结果
- 当通过`submit`方法提交一个`Runnable`或者`Callable`的实现类时，会返回一个`Future`的实现类，在目前`JDK`的实现当中：
 - 提交到`ThreadPoolExecutor`，返回`FutureTask`
 - 提交到`ScheduledThreadPoolExecutor`，返回`ScheduledFutureTask`

`Future`接口所提供的方法提供了下面几个功能：
- 通过`isCancelled()`和`isDone()`方法来获取任务当前的状态
- 通过`cancel()`来取消任务的执行
- 通过`get()`方法来阻塞地获取任务的执行结果

## 2.2 常规任务管理者
与常规任务执行者相关的类的继承关系如下：
![](http://upload-images.jianshu.io/upload_images/1949836-71c5ca83e454065f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
`ThreadPoolExecutor`通常作为常规任务的管理者，根据线程池的大小、工作队列的区别，可以实现不同的任务管理策略。

一般情况下，通常使用工厂类`Executors`来创建不同类型`ThreadPoolExecutor`：
- `FixedThreadPool`：为了满足限制当前线程数量的场景，适用于负载比较重的服务器。
- `SingleThreadPoolExecutor`：适用于需要保证顺序地执行各个任务，在任意时间点，不会有多个活动的应用场景。
- `CacheThreadPool`：大小无界的线程池，适用于执行很多的短期异步任务的小程序或者是负载较轻的服务器。

## 2.3 周期任务管理者
与周期任务有关的类的继承关系如下：
![](http://upload-images.jianshu.io/upload_images/1949836-3b90781390463d8e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

`ScheduledThreadPoolExecutor`相比于`ThreadPoolExecutor`，它主要用来在给定的延迟之后运行任务，或者定期执行任务。和`ThreadPoolExecutor`类似，我们也可以通过`Executors`类来获得它不同的实现：
- `ScheduledThreadPoolExecutor`：适用于需要多个后台线程执行周期任务，同时为了满足资源管理的需求而需要限制后台线程的数量的应用场景。
- `SingleThreadScheduledExecutor`：适用于需要单个后台线程执行周期任务，同时需要保证顺序地执行各个任务的应用场景。

# 三、小结
通过`Executor`框架，就可以把工作单元与执行机制进行分离，开发者只需要把需要执行的任务通过`Runnable`或者`Callable`封装成为执行单元，具体的执行机制则由`Executor`的实现类去处理，免去了开发者对于任务的管理成本。

这篇文章，主要是对`Executor`框架进行了一个简要的介绍，之后我们会深入到第二节讨论的三个部分当中，分析`Executor`内部对于线程管理的实现机制。
