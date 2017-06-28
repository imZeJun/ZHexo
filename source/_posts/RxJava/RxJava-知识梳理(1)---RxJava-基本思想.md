---
title: RxJava 知识梳理(1) - RxJava 基本思想
date: 2017-02-21 00:16
categories : RxJava 知识梳理
---
# 一、基础概述
`RxJava`的关键是异步，即使随着程序的逻辑变得复杂，它依然能够保持简洁。
# 二、`API`介绍和原理剖析

观察者模式面向的需求是：`A`对象（观察者）对`B`对象（被观察者）的某种变化高度敏感，需要在`B`变化的一瞬间做出反应，观察者采用注册`Register`或者订阅`Subscribe`的方式，告诉观察者，我需要你的某某状态，并在它变化的时候通知我，在`RxJava`当中，`Observable`是被观察者，`Observer`就是观察者。

`RxJava`有四个基本概念：
- `Observable`：被观察者。
- `Observer`：观察者。
- `Subscribe`：订阅。
- `Event`：事件。

`Observable`和`Observer`通过`subscribe`方法实现订阅关系，`Observable`可以在需要的时候发出事件来通知`Observer`。

`RxJava`有以下三种事件：
- `onNext`：普通事件。
- `onCompleted`：`RxJava`不仅把每个事件单独处理，还会把它们看作一个队列，当不会再有新的`onNext`事件发出时，需要触发`onCompleted`事件作为标志。
- `onError`：`onCompleted`和有且仅有一个，并且是事件序列中的最后一个。

# 三、基本实现
`RxJava`的基本实现有以下三点：
1）创建观察者 - `Observer`
```
Observer<String> observer = new Observer<String>() {

    @Override
    public void onCompleted() {
        Log.d(TAG, "onCompleted");
    }

    @Override
    public void onError(Throwable e) {
        Log.d(TAG, "onError");
    }

    @Override
    public void onNext(String s) {
        Log.d(TAG, "onNext");
    }
};
```
除了`Observer`接口之外，`RxJava`还内置了一个实现了`Observer`的抽象类：`Subscriber`，它对`Observer`接口进行了一些扩展，实质上在`RxJava`的`subscribe`过程中，`Observer`也总是被转换成为一个`Subscriber`再使用，他们的区别在与：
- `onStart`：这是新增的方法，它会在`subscribe`刚开始，而事件还未发送之前被调用，它总是在`subscribe`所发生的线程被调用。
- `unsubscribe`：这是它实现的另一个接口`Subscription`的方法，用于取消订阅，在这个方法被调用后，`Subscriber`将不再接收事件，一般在调用这个方法前，可以使用`isUnsubscribed`判断一下状态，`Observable`在订阅之后会持有`Subscriber`的引用，因此不释放会有内存泄漏的危险。

2）创建被观察者 - `Observable`
`RxJava`用`create`方法来创建一个`observable`，
```
rx.Observable observable = rx.Observable.create(new rx.Observable.OnSubscribe<String>() {

    @Override
    public void call(Subscriber<? super String> subscriber) {
        subscriber.onNext("Hello World!");
        subscriber.onCompleted();
    }
});
```
这里传入了一个`Observable.OnSubscribe<T>`对象作为参数，它会被存储在返回的`Observable`对象当中，它的作用相当于一个计划表，当`Observable`被订阅的时候，`OnSubscribe`的`call`方法会自动被调用，事件序列被依次触发。
`create`是`RxJava`最基本的创造事件序列的方法，基于这个方法，还提供了一些快捷方法来创建事件队列：
- `just(T...)`
```
Observable observable = Observable.just("Hello", "Hi", "Aloha");
```
- `from(T[]) / from(Iterable<? extends T>)`
```
String[] words = {"Hello", "Hi", "Aloha"};
Observable observable = Observable.from(words);
```

3）订阅 - `subscribe`
```
observable.subscribe(observer);
observable.subscribe(subscriber);
```
其内部核心的代码类似于：
```
public Subscription subscribe(Subscriber subscriber) {
    //准备方法。
    subscriber.onStart();
    //事件发送的逻辑开始执行，这个onSubscribe就是创建Observable时新建的OnSubscribe对象。
    onSubscribe.call(subscriber);
    //把传入的Subscriber转换为Subscription并返回，方便unsubscribe。
    return subscriber;
}
```
`Observable.subscribe`方法除了支持传入`Observer`和`Subscriber`，还支持传入`Action0`、`Action1`这样不完整定义的回调，`RxJava`会自动根据定义创建出`Subscriber`。

# 四、线程控制
在不指定线程的情况下，`RxJava`遵循这样的原则，在哪个线程调用`subscribe`，就在哪个线程产生事件，在哪个线程产生事件，就在哪个线程消费事件，如果需要消费线程，那么就需要用到`Scheduler`， `RxJava`内置了几个`Scheduler`：
- `Schedulers.immediate`：直接在当前线程运行。
- `Schedulers.newThread`：总是启用新线程，并在线程执行操作。
- `Schedulers.io`：其内部实现是一个无数量上限的的线程池，可以重用空闲的线程，不要把计算工作放在`io`，可以避免创建不必要的线程。
- `Schedulers.computation`：使用固定的线程池，大小为`CPU`核数。
- `AndroidSchedulers.mainThread`：指定的操作将在`Android`主线程中运行。

对线程控制有以下两个方法：
- `subscribeOn`：指定`subscribe`发生的线程，即`Observable.OnSubscribe`被激活时所处的线程，也就是`call`方法执行时所处的线程。
- `observeOn`：指定`Subscriber`所运行在的线程。

`observeOn`指定的是`Subscriber`的线程，而这个`Subscriber`并不一定是`subscribe()`参数中的`Subscriber`，而是`observeOn`执行时的当前`Observable`所对应的`Subscriber`，即它的直接下级`Subscriber`，也就是它之后的操作所在的线程，因此，如果有多次切换线程的要求，只要在每个想要切换线程的位置调用依次`observeOn`即可。
和`observeOn`不同，`subscribeOn`只能调用一次，下面我们来分析一下它的内部实现，首先是`subscribeOn`的原理：
`subscribeOn`和`ObserveOn`都做了线程切换的工作：
- `subscribeOn`的线程切换发生在`OnSubscribe`中，即在它通知**上一级**的`OnSubscribe`时，这时事件还没有发送，因此`subscribeOn`的线程控制可以从事件发出的开端造成影响。

![Paste_Image.png](http://upload-images.jianshu.io/upload_images/1949836-179eb2ba8bdd14ea.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
- `observeOn`的线程切换则发生在它内建的`Subscriber`中，即发生在它即将给**下一级**`Subscriber`发送事件时，因此控制的是它后面的线程。

![Paste_Image.png](http://upload-images.jianshu.io/upload_images/1949836-9c1ce1881a0a41ec.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

# 五、变换
变换，就是将事件序列中的对象或整个序列进行加工处理，转换不同的事件或者序列。

## `5.1 map()`
通过`FuncX`，把参数中的`Integer`转换成为`String`，是最常用的变换，这个变换是发生在`subscribeOn`所指定的线程当中的。
```
Subscriber<String> subscriber = new Subscriber<String>() {

    @Override
    public void onCompleted() {

    }

    @Override
    public void onError(Throwable e) {

    }

    @Override
    public void onNext(String s) {
        long nextId = Thread.currentThread().getId();
        Log.d(TAG, "onNext:" + s + ", threadId=" + nextId);
    }
};
Observable<Integer> observable = Observable.create(new Observable.OnSubscribe<Integer>() {
    @Override
    public void call(Subscriber<? super Integer> subscriber) {
        long callId = Thread.currentThread().getId();
        subscriber.onNext(5);
        subscriber.onCompleted();
    }
});
observable.map(new Func1<Integer, String>() {

    @Override
    public String call(Integer integer) {
        long mapId = Thread.currentThread().getId();
        return "My Number is:" + integer;
    }
}).subscribeOn(Schedulers.io()).observeOn(AndroidSchedulers.mainThread()).subscribe(subscriber);
```
其示意图类似于：
![Paste_Image.png](http://upload-images.jianshu.io/upload_images/1949836-ae50c0cdd29e1725.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## `5.2 flatMap`
它和`map`有一个共同点，就是把传入的参数转化之后返回另一个对象，但是和`map`不同的是，`flatMap`返回的是一个`Observable`对象，而且它并不直接把这个对象传给`Subscriber`，而是通过这个新建的`Observable`来发送事件，其整个的调用过程：
- 使用传入的事件对象创建一个`Observable`。
- 激活这个`Observable`，通过它来发送事件。
- 每一个创建出来的`Observable`发送的事件，被汇入同一个`Observable`，它复杂将这些事件同一交给`Subscriber`的回调方法。
```
Subscriber<String> subscriber = new Subscriber<String>() {

    @Override
    public void onCompleted() {}

    @Override
    public void onError(Throwable e) {}

    @Override
    public void onNext(String s) {
        Log.d(TAG, "onNext, s=" + s);
    }
};
Observable<List<String>> observable = Observable.create(new Observable.OnSubscribe<List<String>>() {

    @Override
    public void call(Subscriber<? super List<String>> subscriber) {
        List<String> list = new ArrayList<>();
        list.add("First");
        list.add("Second");
        list.add("Third");
        subscriber.onNext(list);
    }
});
observable.flatMap(new Func1<List<String>, Observable<String>>() {
    @Override
    public Observable<String> call(List<String> strings) {
        return Observable.from(strings);
    }
}).subscribe(subscriber);
```
其示意图：
![Paste_Image.png](http://upload-images.jianshu.io/upload_images/1949836-25b18c3dda6fef6c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

# 六、变换的原理
变换的实质是针对事件序列的处理和再发送，在`RxJava`的内部，它们是基于同一个基础的变换方法`lift(operator)`
```
//生成了一个新的Observable并返回。
public <R> Observable<R> lift(Operator<? extends R, ? super T> operator) {
    //构造新的Observable时，同时新建了一个OnSubscribe对象。
    return Observable.create(new OnSubscribe<R>() {
        @Override
        public void call(Subscriber subscriber) {
            Subscriber newSubscriber = operator.call(subscriber);
            newSubscriber.onStart();
            //原始的onSubscribe。
            onSubscribe.call(newSubscriber);
        }
    });
}
```
示意图：

![Paste_Image.png](http://upload-images.jianshu.io/upload_images/1949836-77bc77be260454c4.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
- `lift`创建了一个`Observable`后，加上之前的原始`Observable`，有两个`Observable`。
- 新的`Observable`里的`OnSubscribe`加上原始的，共有两个`OnSubscribe`。
- 当用户通过调用`lift/map`创建的`Observable`对象的`subscribe`方法时，于是它触发了上面的`call`方法中的内容。
- 在这个新的`OnSubscribe`的`call`方法中，传入了目标的`Subscriber`，同时其外部类中还持有了原始的`OnSubscribe`。我们先通过`operator.call(oldSubscriber)`方法，生成了新的`Subscriber（new Subscriber）`，然后利用这个新的`Subscriber`向原始的`Observable`进行订阅。

下面我们以前面`map`实现的例子来分析一下源码，上面的例子通过`map`操作符把`Integer`类型的`Observable`和`String`类型的`Subscriber`生成了订阅关系。
- `map`方法，它通过`lift`方法返回了一个`String`类型的`Observable`。
```
//其中T=Integer，R=String。
public final <R> Observable<R> map(Func1<? super T, ? extends R> func) {
        return lift(new OperatorMap<T, R>(func));
}
```
- 下面看下`OperatorMap`这个对象，这个对象实现了`operator<R,T>`接口，而这个接口继承于`Func1<Subscriber<? super R>, Subscriber<? super T>>`，在它实现的`call`方法中传入了`String`类型的`Subscriber`（目标`Subscriber`），并返回了`Integer`类型的`Subscriber`（代理`Subscriber`），当它的方法被回调时，会调用目标`Subscriber`的对应方法，其中在调用`onNext`时，就用上了外部传入的`Func1`函数：
```
    @Override
    public Subscriber<? super T> call(final Subscriber<? super R> o) {
        return new Subscriber<T>(o) {

            @Override
            public void onCompleted() {
                o.onCompleted();
            }

            @Override
            public void onError(Throwable e) {
                o.onError(e);
            }

            @Override
            public void onNext(T t) {
                try {
                    o.onNext(transformer.call(t));
                } catch (Throwable e) {
                    Exceptions.throwIfFatal(e);
                    onError(OnErrorThrowable.addValueAsLastCause(e, t));
                }
            }

        };
    }
```
- 接着再回过头来看`lift`方法：
```
    public final <R> Observable<R> lift(final Operator<? extends R, ? super T> operator) {
        return new Observable<R>(new OnSubscribe<R>() {
            @Override
            public void call(Subscriber<? super R> o) {
                try {
                    //返回一个Integer类型的Subscriber。
                    Subscriber<? super T> st = hook.onLift(operator).call(o);
                    try {
                        st.onStart();
                        //关键方法：Integer类型的OnSubscribe调用对应的Subscribe，这个call方法里面写了我们的逻辑，当它调用onNext(Integer integer)时，实际上调用的是onNext(String str)。
                        onSubscribe.call(st);
                    } catch (Throwable e) {
                        if (e instanceof OnErrorNotImplementedException) {
                            throw (OnErrorNotImplementedException) e;
                        }
                        st.onError(e);
                    }
                } catch (Throwable e) {
                    if (e instanceof OnErrorNotImplementedException) {
                        throw (OnErrorNotImplementedException) e;
                    }
                    o.onError(e);
                }
            }
        });
    }
```
- 最后就是调用`subscribe`方法。
