---
title: Activity 知识梳理(2) - Activity 栈
date: 2017-02-20 22:59
categories : Activity 知识梳理
---
# 一、`AndroidManifest.xml`中指定`launchMode`
# 1.1`standard`
标准模式，每次启动`Activity`都会创建一个新的`Activity`实例，并且将其压入任务栈栈顶，而不管这个 Activity 是否已经存在，都会执行`onCreate() ->onStart() -> onResume`。
## 1.2 `singleTop`
栈顶复用模式，如果新`Activity`已经位于栈顶，那么此`Activity`不会被重新创建，同时`Activity`的 `onNewIntent`方法会被回调，如果`Activity`已经存在但是不再栈顶，那么和`standard`模式一样。
如果`Activity`当前是`onResume`状态，那么调用后会执行`onPause() -> onNewIntent() -> onResume()`。

## 1.3 `singleTask`
栈内复用模式，创建这样的`Activity`，系统会确认它所需任务栈是否已经创建，否则先创建任务栈，然后放入`Activity`，**如果栈中已经有一个`Activity`实例**，那么会做两件事：
- 这个`Activity`会回到栈顶执行`onNewIntent`
- 清理在当前`Activity`上面的所有`Activity`

上面的**如果栈中已经有一个`Activity`实例**，这个判断条件的**标准是由`android:taskAffinity`**决定的，下面我们做一个简单的对比：
- 第一种情况，不给`singleTask`的`Activity`设置`taskAffinity`，这时默认情况下属于同一个`Application`的所有`Activity`具有的`taskAffinity`是相同的，就是我们在`AndroidManifest`中指定的`packageName`：

```
<activity android:name=".SingleTaskActivity" android:launchMode="singleTask"/>
```
这时候我们从`MainActivity`启动`SingleTaskActivity`后，任务栈的情况是，`MainActivity`和 `SingleTaskActivity`处于同一个`Task`当中：
![](http://upload-images.jianshu.io/upload_images/1949836-a05689be2ee8ff2f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
我们看到`SingleTaskActivity`和`MainActivity`位于同一个栈中，因此`singleTask`并不是让这个`Activity`独占一个`Task`。
- 第二种情况，给`singleTask`的`Activity`设置`affinity`：

```
<activity android:name=".SingleTaskActivity" android:launchMode="singleTask" android:taskAffinity="com.android.singleTask"/>
```
此时进行同样的操作，任务栈的情况变为：
![](http://upload-images.jianshu.io/upload_images/1949836-9a83c8086c319341.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
我们在`SingleTaskActivity`的界面按下 `Home`键，再点击图标进入`MainActivity`，可以看到当前应用有两个栈：


![](http://upload-images.jianshu.io/upload_images/1949836-7327154632889136.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
此时，我们再点击按钮启动`SingleTaskActivity`，那么会执行 
```
MainActivity#onPause
SingleTaskActivity#onNewIntent 
SingleTaskActivity#onRestart 
SingleTaskActivity#onStart 
SingleTaskActivity#onResume
MainActivity#onStop
```
这是由于当启动`SingleTaskActivity`，系统去寻找该`SingleTaskActivity`所对应的栈是否存在，而这时候是存在的，也就上面看到`TaskRecord[cd9ca5d]`，所以它不会创建新的`SingleTaskActivity`，而是复用这个栈中的`Activity`，而由于这个`Activity`又位于栈顶，因此它的表现和`SingleTop`相同。
- 第三种情况，我们试着在第二种的基础上再加大一些难度， 在`SingleTaskActivity`所在的`Task`上再加一个 `SingleTaskAboveActivity`，首先我们从`MainAcitivity -> SingleTaskActivity -> SingleTaskAboveActvity`

```
<activity android:name=".SingleTaskAboveActivity"/>
```
这一流程过后，栈的结构为，可以看到`SingleTaskActivity`和`SingleTaskAboveActvity`位于同一栈中：

![](http://upload-images.jianshu.io/upload_images/1949836-41077a32981958fb.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
在`SingleTaskAboveActvity`界面，按`Home`退到后台之后重新进入，栈的结构不变，只不过当前可见的是 `MainActivity`，这时我们再次尝试启动`SingleTaskActivity`，那么会依次调用：
```
MainActivity#onPause
SingleTaskAboveActivity#onDestroy
SingleTaskActivity#onNewIntent
SingleTaskActivity#onRestart
SingleTaskActivity#onStart
SingleTaskActivity#onResume
MainActivity#onStop
```
而栈的结构变为如下：
![](http://upload-images.jianshu.io/upload_images/1949836-ee2e7aa7ddf2b3f4.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
和第二种情况类似，当启动`SingleTaskActivity`时，系统去寻找该`SingleTaskActivity`所对应的栈是否存在，而这时候是存在的，也就上面看到`TaskRecord[a3771ca]`，所以它不会创建新的`SingleTaskActivity`，而是复用这个栈中的`SingleTaskActivity`，但此时`SingleTaskActivity`并不位于栈顶，在它上面还有一个`SingleTaskAboveActivity`，因此会把`SingleTaskAboveActivity`先出栈，再复用原先位于这个栈中的`SingleTaskActivity`实例。
## 1.4 `singleInstance`
这种模式的`Activity`只能单独位于一个任务栈内，由于栈内的复用特性，后续请求均不会创建新的`Activity`，除非这个独特的任务栈被系统销毁了。
- 第一种情况，先看最简单的，我们新建一个`SingleInstanceActivity` 

```
<activity android:name=".SingleInstanceActivity" android:launchMode="singleInstance"/>
```
我们从`MainAcitivity`启动它，此时任务栈的情况是，他们位于不同的`Task`中，符合我们的预期：

![](http://upload-images.jianshu.io/upload_images/1949836-43ea89fd9e81bfdb.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
- 第二种情况，此时按`Home`回到桌面，再重新点图标进入`MainActivity`，任务栈依然是两个，我们此时再启动`SingleInstanceActivity`：
```
MainActivity#onPause
SingleInstanceActivity#onNewIntent 
SingleInstanceActivity#onRestart 
SingleInstanceActivity#onStart 
SingleInstanceActivity#onResume
MainActivity#onStop
```
和启动在另一个栈中已存在的`singleTaskAcitivity`的情况是类似的。

- 第三种情况，我们再看一下，`affinity`对于`singleInstance`会不会有影响呢，我们定义两个`affinity`相同的 `singleInstance`：

```
 <activity android:name=".SingleInstanceActivity" android:launchMode="singleInstance" android:taskAffinity="com.android.singleInstance"/>
<activity android:name=".SingleInstanceActivityAnother" android:launchMode="singleInstance" android:taskAffinity="com.android.singleInstance"/>
```
我们先从`MainActivity`启动`SingleInstanceActivity`，按`Home`回到桌面再进入，此时`Task`的情况和上面相同的，那么这时候我们启动`SingleInstanceActivityAnother`：

![singleInstance_another.png](http://upload-images.jianshu.io/upload_images/1949836-b4aa0c5e4d7e5110.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
可以看到，它并没有受到`affinity`的影响，而是重新起了一个新的栈。

## 二、在`Intent`当中指定启动模式

## 2.1 `FLAG_ACTIVITY_NEW_TASK`
和`singleTask`行为相同，前面已经详细分析过了，这里需要注意 affinity 的声明。

## 2.2 `FLAG_ACTIVITY_SINGLE_TOP`
和`singleTop`行为相同，比较简单，就不举例子了。

## 2.3 `FLAG_ACTIVITY_CLEAR_TASK`
和 `FLAG_ACTIVITY_NEW_TASK` 何用，这个`Activity`会新起一个栈，原来栈被清空，栈中的`Activity`也被销毁。

## 2.5 `FLAG_ACTIVITY_CLEAR_TOP`
会清除这个`Activity`之上所有的`Activity`，我们来试一下，新建两个新的`SecondActivity`和`ThirdActivity`，从`Main -> Second -> Third`，此时栈的结构是：
![clearTop_1.png](http://upload-images.jianshu.io/upload_images/1949836-834b2531b5873016.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
此时，我们按如下方式启动`SecondActivity`：
```
    public void third(View view) {
        Intent intent = new Intent(this, SecondActivity.class);
        intent.addFlags(Intent.FLAG_ACTIVITY_CLEAR_TOP);
        startActivity(intent);
    }
```
这之后，栈的结构变为，`ThirdAcitivity`被出栈了：

![clearTop_2.png](http://upload-images.jianshu.io/upload_images/1949836-9f99fb8f26072fa6.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 2.6 ` FLAG_ACTIVITY_REORDER_TO_FRONT`
上面的`FLAG_ACTIVITY_CLEAR_TOP`是把位于目标`Activity`之上的`Activity`都销毁，而则个`FLAG`则是对栈重新排序，把目标`Activity`移到最前台，其它的位置不变，我们在前一种的基础上，在`ThirdActivity`中换一种方式来启动`SecondActivity`：
```
    public void third(View view) {
        Intent intent = new Intent(this, SecondActivity.class);
        intent.addFlags(Intent.FLAG_ACTIVITY_REORDER_TO_FRONT);
        startActivity(intent);
    }
```
这回，最终栈的结构变为了，可以看到`ThirdActivity`并没有被出栈：

![Paste_Image.png](http://upload-images.jianshu.io/upload_images/1949836-9a643a4acadaa2d3.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 三、`AndroidManifest `中的属性

## 3.1 `alwaysRetainTaskState`
这个标志只对根`Activity`有用，默认情况下，当我们的应用在后台一段时间，它会销毁该`Task`除了根以外的所有`Activity`，如果我们希望保持这个`Task`的原有状态，那么给这个`Task`的根`Activity`设置这个属性，默认值是`false`。

## 3.2 `clearTaskOnLaunch`
从桌面启动该`Activity`的时候会清空该`Task`除了根`Activity`外的所有`Activity`，我们从`Main -> Second -> Third`，此时栈内有3个`Activity`，按`Home`回到桌面后，点图标重新进入，此时`Task`只剩下根`Activity` 了：

![Paste_Image.png](http://upload-images.jianshu.io/upload_images/1949836-5479b6fd8f7f2d28.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 3.3 `finishOnTaskLaunch`
这个和上面类似，但是它对根`Activity`无效，我们给`SecondActivity`设置这个属性，先启动到`ThirdActivity`，这时候栈的结构为：

![Paste_Image.png](http://upload-images.jianshu.io/upload_images/1949836-b6a9005ba3caee7b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

接着，我们按`Home`回到桌面，点图标重新进入，栈的结构变为下面这样，可以看到`SecondActivity`没有了：

![Paste_Image.png](http://upload-images.jianshu.io/upload_images/1949836-05638cdb357aa318.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 3.4 `noHistory`
`Activity`在不可见之后，不保存记录

- 第一种情况，我们给`SecondActivity`设置这个属性，接着从`Main -> Second -> Third`，然后按`Back`返回，此时的生命周期为：
```
ThirdActivity#onPause
MainActivity#onRestart
MainActivity#onStart
MainActivity#onResume
SecondActivity#onDestroy
ThirdActivity#onStop
ThirdActivity#onDestroy
```
- 第二种情况，如果我们在`ThirdActivity`时，不是按`Back`，而是按`Home`到桌面，会调用：
```
ThirdActivity#onPause
SecondActivity#onDestroy
ThirdActivity#onStop
```
- 第三种情况，我们给根`MainAcitivity`设置这个属性，启动它后退出：
```
MainActivity#onCreate
MainActivity#onStart
MainActivity#onResume
MainActivity#onDestory
```
