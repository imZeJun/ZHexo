---
title: 多线程知识梳理(4) - synchronized 三部曲之等待/通知模型
date: 2017-05-05 21:49
categories : 多线程知识梳理
---
# 一、概述
在前面两篇文章当中，我们介绍了`synchronized`的基本使用和原理，但是在使用`synchronized`保证数据一致性的同时，我们希望能够让线程之间进行一些交互逻辑，也是我们今天要介绍的等待/通知模型，那么就需要使用到`wait/notify`。

# 二、等待/通知相关方法
## 2.1 方法说明
下面，我们先介绍等待/通知机制的相关方法，首先要说明两点：
- 这些都是`Object`定义的方法
- 调用这些方法的前提条件是：该线程已经获得了`Object`对象所关联的锁，也就是说它们需要位于`synchronized`修饰的同步代码块中。

**(a) wait() **
调用该方法的线程进入等待状态，并释放它所获取的对象锁，只有出现这两种情况之一，它才会从`wait`方法中返回，否则将会一直处于等待状态：
- 其它线程通过`notify / notifyAll`方法通知该线程，并且该线程获取到了对象锁
- 线程被中断

**(b) wait(long) / wait(long, int)**
和`wait`方法相同，差别是增加一种从`wait`方法返回的情况：等待的时间已经到了，并且获取到了对象锁。

**(c) notify()**
通知位于等待队列中的第一个线程，使其从`wait()`方法返回，而被通知的线程的继续执行需要等到它获得对象所为止。
需要注意，调用`notify`方法后，并不会立刻释放它所持有的对象锁，这需要等到它执行完同步代码块为止。

**(d) notifyAll()**
与`notify()`类似，但是它是通知所有在对象上等待的线程。

## 2.2 实现原理
通过上面的介绍，我们可以看到，在整个等待/通知机制当中，线程被挂起时主要有以下三种状态：等待状态、超时等待状态、阻塞状态，这些状态都是通过`synchroized`所修饰的对象来实现的。

在前面我们介绍`synchronized`原理的时候，曾经说过每个对象都会和一个`Monitor`相关联，其实每个`Monitor`又包含有两个队列：等待队列和同步队列，其中等待队列中存放是进入等待状态的线程，而同步队列中存放的是等待获取锁的线程。

下面，我们通过一段简单的伪代码来立即两个线程的状态转换过程：
```
synchronized public void waitThread() {
    //执行a方法.
    wait();
    //执行b方法
}

synchronized public void notifyThread() {
    //执行c方法
    notify();
    //执行d方法
}
```
我们有`AB`两个线程，我们模拟以下的一系列行为：

**(1) A 线程执行 waitThread 方法**
此时由于对象锁没有被任何线程持有，因此，`A`线程成为对象锁的持有者：
![](http://upload-images.jianshu.io/upload_images/1949836-cd58d49fbbd236c5.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
**(2) B 线程执行 notifyThread 方法**
当`B`线程执行`notifyThread`方法时，由于此时对象锁已经被`A`线程持有，因此它被加入到同步队列中：
![](http://upload-images.jianshu.io/upload_images/1949836-3497701a1482b326.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
**(3) A 线程执行 a 方法**
![](http://upload-images.jianshu.io/upload_images/1949836-d5798a318b8c8281.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
**(4) A 线程执行 wait 方法 **
当`A`线程执行`wait`方法后，它会释放对象锁，并加入到同步队列当中，而`B`线程则成为对象锁新的持有者：
![](http://upload-images.jianshu.io/upload_images/1949836-febc0f2cbc884bc4.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
**(5) B 线程执行 c 方法**
![](http://upload-images.jianshu.io/upload_images/1949836-add0979bc7229931.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
**(6) B 线程执行 notify 方法**
此时会唤醒等待队列中`A`线程，但是此时`B`线程仍然持有对象锁，因此，`A`线程只能被加入到同步队列：
![](http://upload-images.jianshu.io/upload_images/1949836-bc42d9e8cc847dfd.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
**(7) B 线程执行 d 方法**
![](http://upload-images.jianshu.io/upload_images/1949836-bc42d9e8cc847dfd.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
**(8) B 线程从 notifyThread 方法返回**
此时`A`线程重新获取到对象锁，因此它被从同步队列中取出，继续执行接下来的逻辑：
![](http://upload-images.jianshu.io/upload_images/1949836-38456e34948bb108.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
**(9) A 线程执行 b 方法**
![](http://upload-images.jianshu.io/upload_images/1949836-4d0b527326a65a6b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
**(10) A 线程从 waitThread 方法中返回**
当`A`线程从同步方法返回之后，那么会释放它所持有的锁
![](http://upload-images.jianshu.io/upload_images/1949836-33e2ba405dffa721.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

# 三、等待/通知的经典范式
对于等待/通知模型，我们可以总结出它的经典范式，分别针对等待方和通知方。
## 3.1 等待方
等待方遵循如下的原则：
- 获取对象的锁
- 如果条件不满足，那么调用对象的`wait`方法，被通知后仍然需要检查条件
- 条件满足则继续执行对应的逻辑

对应的伪代码为：
```
synchronized( 对象 ) {
    while( 条件不满足 ) {
        对象.wait();
    }
    对应的处理逻辑
}
```
## 3.2 通知方
通知方遵循如下的原则：
- 获得对象的锁
- 改变条件
- 通知所有等待在对象上的线程

```
synchronized( 对象 ) {
    改变条件;
    对象.notifyAll();
}
```



