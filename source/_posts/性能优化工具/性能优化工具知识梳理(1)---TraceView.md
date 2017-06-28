---
title: 插性能优化工具知识梳理(1) - TraceView
date: 2017-03-25 12:08
categories : 插性能优化工具知识梳理
---
> [性能优化工具知识梳理(1) - TraceView](http://www.jianshu.com/p/37c263f9886b)
[性能优化工具知识梳理(2) - Systrace](http://www.jianshu.com/p/41bb27235921)
[性能优化工具知识梳理(3) - 调试GPU过度绘制 & GPU呈现模式分析](http://www.jianshu.com/p/ac2d58666106)
[性能优化工具知识梳理(4) - Hierarchy Viewer](http://www.jianshu.com/p/7ac6a2b8d740)
[性能优化工具知识梳理(5) - MAT](http://www.jianshu.com/p/fa016c32360f)
[性能优化工具知识梳理(6) - Memory Monitor & Heap Viewer & Allocation Tracker](http://www.jianshu.com/p/29a539bca730)
[性能优化工具知识梳理(7) - LeakCanary](http://www.jianshu.com/p/3c055862f353)
[性能优化工具知识梳理(8) - Lint](http://www.jianshu.com/p/4ebe5d502842)

# 一、概述
使用`Systrace`需要开发者对于整个渲染的原理有较深的理解，而`TraceView`则更为直观，你可以通过它来分析在一段时间应用内各个线程的运行情况，它会帮你计算出每个方法的具体耗时，这样我们就可以了解到方法运行的效率，定位到当前性能的瓶颈在哪，从而考虑将一些耗时的操作放在子线程中进行，或者延后执行来优化应用的启动速度和解决卡顿问题。
和之前一样，我们分几个部分介绍它：
- 获取`TraceView`的分析报表
- 分析报表的格式
- 具体案例

# 二、获取`*.trace`报表
- 第一步：点击`Android Studio`中的`Tools/Android/Android Device Monitor`，打开调试界面。
- 第二步：如下面截图所示，选中要分析的应用包名，点击`Start Method Profiling`按钮，之后就会开始跟踪：
![](http://upload-images.jianshu.io/upload_images/1949836-3a6b1999c36b8210.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
- 第三步：进入应用内进行操作，操作完毕后，再次点击上面的按钮`Stop Method Profiling`，等待一会，就会在右边的窗口生成分析的报表，将鼠标上移到红框的文件名处，可以看到保存的位置，我们可以把它保存起来以便之后分析跟踪问题：
![](http://upload-images.jianshu.io/upload_images/1949836-40b2fac6954fec4d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

# 三、分析`*.trace`报表
获取完报表之后，我们就可以通过它来分析，这个区域分为三个部分：
![](http://upload-images.jianshu.io/upload_images/1949836-8e3c61e581d0cc84.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
- 红色区域：列出了运行的各个线程。
- 蓝色区域：线程在一段时间内的运行情况，我们点击有颜色的地方，就可以定位到具体做了哪些操作。
- 紫色区域：方法调用的具体情况。

前面两个区域都比较好理解，我们主要看一下紫色区域的每一列具体的含义：
![](http://upload-images.jianshu.io/upload_images/1949836-a38efe53a2dc5ae4.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

上面的表格中，一部分单位是百分比，另一部分是`ms`，要注意区别，除此之外，有两点需要解释一下：
- `Incl`和`Excl`的概念
关于`Incl`和`Excl`的概念可以用下面这段代码来表示：
```
    public static void inclExcl(Context context) { //这个函数的运行时间为 Incl Real Time
        //假如这里没有调用任何函数，那么这里的运行时间就是：Excl Real Time
        writeSomething(context, 1000); //这里的运行时间是：Incl Real Time - Excl Real Time
    }
```
- `Cpu Time`和`Real Time`的区别
`Real Time`表示的是一个函数从开始到运行时候所占用的时间，而`CPU Time`则表示`CPU`执行这段函数所耗费的时间。
举个例子，假如我们一个函数`doSomething()`，`CPU`执行它首先花了`time1`的时间，之后`CPU`被分配用去执行别的函数花了`time2`，执行完之后继续回来执行`doSomething()`，花了`time3`的时间把它执行完，那么`doSomething()`的`Real Time`就等于`time1 + time2 + time3`，而`Cpu Time`则等于`time1 + time3`。

我们可以通过点击具体的列进行排序，在平时的分析中，我们主要关注一下：
- `CPU Time / Call`较高
这一列表示函数的**单个执行时间较长**，这里往往是可以优化的点：
 - 分析该函数的实现方式，能够用其它的方法来实现，从而减少函数的运行时间
 - 分析该函数所执行的线程，如果可以那么把它放到子线程中执行
 - 分析该函数所调用的时机，避免在应用启动，或者做动画的过程中执行它，否则会导致启动速度变慢和界面卡顿。

- `CPU Time`较高，但`CPU Time / Call`不高
这一列表示函数的**单个执行时长不长，但调用的次数很多**，这同样是可以优化的点，需要看一下是否有必要在每个地方都调用它，能否进行延后操作，统计到一个地方执行。

- 和`Android`相关的某些函数的`Real Time`：
由于我们的界面需要等到`onCreate`、`onResume`等函数执行完之后，才会真正的显示出来，因此，我们需要保证它们能够尽快地执行完，也就是它的`Incl Real Time`尽可能地短：
例如下面两点是最基本的：
 - `onCreate`：
![](http://upload-images.jianshu.io/upload_images/1949836-347ecd8a7f0a0d7b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
 - `onResume`：
![](http://upload-images.jianshu.io/upload_images/1949836-d4f50ca97381560b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

# 四、具体案例
我们分析一下在启动过程中，由于布局复杂或者耗时操作导致的问题：
首先我们看一下正常的情况：
- `callActivityOnCreate`部分耗时为`6.180ms`
![](http://upload-images.jianshu.io/upload_images/1949836-56672cf6fc93b510.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
- `callActivityOnResume`耗时为`1.219ms`
![](http://upload-images.jianshu.io/upload_images/1949836-5f6fcfc2366b596d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 4.1 布局层次过于复杂导致的问题
我们修改`Activity`的根布局：
```
<FrameLayout
        android:background="@android:color/black"
        android:layout_width="match_parent"
        android:layout_height="match_parent">
        <FrameLayout
            android:background="@android:drawable/screen_background_dark"
            android:layout_width="match_parent"
            android:layout_height="500dp">
            <FrameLayout
                android:background="@android:drawable/star_big_off"
                android:layout_width="match_parent"
                android:layout_height="400dp">
                <FrameLayout
                    android:background="@android:drawable/bottom_bar"
                    android:layout_width="match_parent"
                    android:layout_height="300dp">
                    <FrameLayout
                        android:background="@android:color/holo_blue_bright"
                        android:layout_width="match_parent"
                        android:layout_height="200dp">
                    </FrameLayout>
                </FrameLayout>
            </FrameLayout>
        </FrameLayout>
</FrameLayout>
```
再次抓取之后，看一下`onCreate`的时间，发现耗时增加到`17ms`：
![](http://upload-images.jianshu.io/upload_images/1949836-a2ccf719b0cfeca0.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
之后再点击这个函数，分析里面耗时最长的子方法，可以看到就是由于布局引起的：
![](http://upload-images.jianshu.io/upload_images/1949836-9a7dd8e02563c2bb.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
## 4.2 在启动过程中进行耗时的操作
首先把布局恢复成没有问题的状态，然后在`onCreate`和`onResume`方法中增加不同的`IO`操作：
```
public class TraceViewActivity extends Activity {

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_trace_view);
        TraceViewOperation.writeOnActivityOnCreate(this, 1000);
    }

    @Override
    protected void onResume() {
        super.onResume();
        TraceViewOperation.writeOnActivityOnResume(this, 1000);
    }

}
```
其中`onCreate`在主线程中进行，而`onResume`里面则另起了一个线程：
```
public class TraceViewOperation {

    public static void writeOnActivityOnCreate(Context context, int count) {
        writeSomething(context, count);
    }
    
    public static void writeOnActivityOnResume(final Context context, final int count) {
        new Thread() {
            @Override
            public void run() {
                super.run();
                writeSomething(context, count);
            }
        }.start();
    }

}
```
- `onCreate`耗时：
![](http://upload-images.jianshu.io/upload_images/1949836-ec2851ac52e51535.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
和上面类似，点进去看耗时的方法，正是由于我们在前面进行`IO`操作引起的：
![](http://upload-images.jianshu.io/upload_images/1949836-8bd51807c19ca232.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
- `onResume`耗时：
![](http://upload-images.jianshu.io/upload_images/1949836-9d36dd128b39b9ee.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
可以看到，由于我们是另起了一个线程进行操作，因此，并不会占用它的运行时间。

# 五、小结
通过`TraceView`，我们可以了解到某些关键函数上的运行时间，正如前面第三点谈到的，根据不同的情况进行分析和优化，将会有效地提高应用的启动速度，避免出现卡顿现象。
