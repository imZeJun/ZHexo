---
title: 多线程知识梳理(2) - synchronized 三部曲之基本使用
date: 2017-04-29 23:54
categories : 多线程知识梳理
---
# 一、为什么要使用 synchronized 
使用`synchronized`的原因在于：它能够确保多个线程在同一时刻，只能有一个线程处于方法或者同步块中，它保证了线程对变量访问的可见性和排他性。
# 二、synchronized 原理
在`JDK 1.6`之前，`synchronized`的实现是基于对象上的监视器，这也被称为重量锁。默认情况下，每一个对象都有一个关联的`Monitor`，而每个`Monitor`包含了一个`EntryCount`计数器，它是`synchronized`实现可重入的关键。

在`JDK 1.6`之后，对锁进行了一系列优化的措施，通过引入自旋锁、适应性自旋锁、锁消除、锁粗化、偏向锁、轻量级锁等技术来减少锁操作的开销。

这些优化措施最终的目的是减少锁操作的开销，然而它所改变的只是锁的实现方式，但是加锁和解锁这一基本原则是没有改变的。这篇文章主要是介绍`synchronized`的使用，因此，在后面的介绍中，我们还是按照比较容易理解的重量锁的方式进行分析，在之后的文章中，我们再来谈一下优化后的实现策略。

## 2.1 进入同步方法或者代码块
当一个线程执行某个对象的同步方法或者代码块时，会先检查这个对象所关联的`Monitor's EntryCount`是否为`0`：
- 如果`EntryCount`为`0`，那么该线程就会将`Monitor’s EntryCount`设置为`1`，并成为该`Monitor`的所有者，接着执行该方法或者代码块中的语句。
- 如果`EntryCount`不为`0`，这时会去检查对象所关联的`Monitor`的持有者是哪一个线程：
 - 第一种情况：持有该`Monitor`的线程就是当前正在尝试获取`Monitor`的线程，那么将`EntryCount`的数值加`1`，继续执行方法或者代码块中的语句。
 - 第二种情况：持有该`Monitor`的是其它的线程，那么该线程进入阻塞状态，直到`EntryCount`的数值变为`0`。

## 2.2 退出同步方法或者代码块
当一个线程从同步方法或者代码块退出时，会将`EntryCount`减`1`，如果`EntryCount`变为`0`，那么该线程会释放它所持有的`Monitor`。之前那些阻塞在`synchronized`的线程会尝试去获取`Monitor`，成功获取`Monitor`的线程可以进入同步方法或者代码块。

# 三、synchronized 使用
对于`synchronized`的使用，我们有两种分类方法：
- 根据使用场景分类
- 根据`Monitor`关联的对象分类。

## 3.1 根据使用场景分类
很多介绍`synchronized`的文章，都是通过使用场景进行分类的，一般来说可以分为如下四种使用场景，而每种场景下根据`Monitor`所关联的对象不同，又会衍生出另外的用法：
- 静态方法
```
    //静态方法，使用的是Class类锁
    synchronized public static void staticMethod() {}
```
- 静态方法代码块
```
    private static final byte[] mStaticLockByte = new byte[1];

    //静态方法代码块1，使用的是Class类锁
    public static void staticBlock1() {
        synchronized (SynchronizedObject.class) {}
    }

    //静态方法代码块2，使用的是内部静态变量锁
    public static void staticBlock2() {
        synchronized (mStaticLockByte) {} 
    }
```
- 普通方法
```
    //普通方法，使用的是调用该方法的对象锁
    synchronized public void method() {}
```
- 普通方法代码块
```
    private static final byte[] mStaticLockByte = new byte[1];
    private final byte[] mLockByte = new byte[1];

    //普通方法代码块1，使用的是Class类锁
    public void block1() {
        synchronized (SynchronizedObject.class) {}
    }

    //普通方法代码块2，使用的是mLockByte的变量锁
    public void block2() {
        synchronized (mLockByte) {} //变量需要声明为final
    }
    
    //普通方法代码块3，使用的是mStaticLockByte的变量锁
    public void block3() {
        synchronized (mStaticLockByte) {} 
    }

    //普通方法代码块4，使用的是调用该方法的对象锁
    public void block4() {
        synchronized (this) {}
    }
```

## 3.2 根据 Monitor 关联的对象分类
根据使用场景进行分类，主要是为了让大家知道如何使用`synchronized`关键字，然而要真正地理解`synchronized`，就需要结合第二节谈到的`synchronized`原理，其实`3.1`中谈到的多种场景，都是和`Monitor`有关，那么从和`Monitor`关联的对象来看，我们重新对`3.1`中的`8`种场景重新进行分类：
- `Class`对象：
 - 静态方法
 - 静态方法代码块1 - `SynchronizedObject.class`
 - 普通方法代码块1 - `SynchronizedObject.class`
- 调用方法的对象
 - 普通方法
 - 普通方法代码块4 - `this`
- 静态对象
 - 静态方法代码块2 - `mStaticLockByte `
 - 普通方法代码块3 - `mStaticLockByte`
- 非静态对象
 - 普通方法代码块1 - `mLockByte`

如果使用场景属于上面的同一个分类当中，那么才有可能产生线程阻塞在`synchronized`关键字的情况，举一个例子，如果`A`线程通过静态方法访问（分类一）并且没有从该方法退出：
- 这时`B`线程是通过一个对象的普通方法来访问（分类二），那么是不会阻塞的，这是因为调用该方法的对象所关联的`Monitor`没有被持有。
- 如果`B`线程使用的是静态方法代码块来访问，而该静态方法代码块使用的是`SynchronizedObject.class`来修饰（分类一），由于这两种使用场景是属于同一个分类，那么就会`B`线程就会进入阻塞状态，这是因为`SynchronizedObject`类所关联的`Monitor`已经被`A`线程持有了。

# 四、小结
从表面上来看，`synchronized`的使用可以简单地分为同步方法和同步代码块，但是究竟在什么情况下会导致一个线程在`synchronized`上阻塞，则需要分析`synchronized`方法所尝试获取的`Monitor`的是否已经被其它线程持有了。
