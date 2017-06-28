---
title: RecyclerView 知识梳理(1) - 综述
date: 2017-04-03 15:11
categories : RecyclerView 知识梳理
---
# 一、概述
对于`RecyclerView`的学习，主要是需要掌握以下几点：
- 数据：`Adapter`
 - 使用：[RecyclerView - Adapter](http://www.jianshu.com/p/ec6585e5220d)
 - 进阶：[BaseRecyclerViewAdapterHelper](https://github.com/CymChad/BaseRecyclerViewAdapterHelper)
- 布局：`LayoutManager`
 - 使用：[RecyclerView - LayoutManager](http://www.jianshu.com/p/1cfca4d34402)
 - 进阶：自定义
- 动画：`ItemAnimator` 
 - 使用
 - 进阶：[RecyclerViewItemAnimators](https://github.com/gabrielemariotti/RecyclerViewItemAnimators)
- 装饰：`ItemDecorator`
 - 使用：[RecyclerView - ItemDecoration](http://www.jianshu.com/p/b2ef4f8e859f)
- 手势：`ItemTouchHelper`
 - 使用：[RecyclerView - ItemTouchHelper](http://www.jianshu.com/p/0bbc44cc1582)

要理解整个`RecyclerView`的思想，有一个视频是一定要看的：[RecyclerView ins and outs - Google I/O 2016](http://v.youku.com/v_show/id_XMTU4MTQ1ODg2NA==.html?firsttime=1211&spm=a2hww.20023042.uerCenter.5!2~5~5!2~5~DL~DD~A)。今天，我们就通过这个视频，把上面所学到的东西串联起来。

# 二、为什么要使用`RecyclerView`
`RecyclerView`诞生的目的就是为了替代`ListView`，我们先总结一下在使用`ListView`过程当中所遇到的问题：
- 复用`Item`需要编写很多的代码
在使用`ListView`的时候，有经验的程序员一定会告诉你在`getView`中要这么写，如果忘了，那么会产生很严重的性能问题。
```
if (convertView == null) {
     //通过LayoutInflator生成convertView，并产生一个ViewHolder，通过setTag关联起来.       
} else {
    //通过getTag获取ViewHolder，进行更新操作.
}
```
- 焦点冲突问题
当`Item`有焦点时，`Item`的子控件就无法获取到焦点；而如果子控件抢夺了焦点，那么`Item`的点击事件又不能响应，这个相信大家都遇到过。
- 重复的`API`
`ListView`中提供了很多的`API`，但是这些`API`又和`View`的一些`API`重复了，例如我们可以给`ListView`设置`setOnItemClickListener`，也可以在`getView`中给某个`View`设置`setOnClickListener`，这就让人很疑惑，到底应当选用哪个。
- 动画
当我们需要在`ListView`中进行添加、删除、移动等操作的时候，如果希望加上动画，那么是很困难的，根本原因是我们是通过`Adapter`通知`ListView`进行更新，然而`ListView`根本就没法确定到底是哪些`View`发生了变化。
- 更加复杂的布局需求
`ListView`在布局是规整的列表的时候能满足大多数人的使用，然而如果想要实现像瀑布流这种复杂的布局，并且保证`View`能够复用，那么需要编写很多的代码。

如果之前有了解过`RecyclerView`的基本用法，那么你会发现，对于上述这些问题，它都给出了自己的解决方案：
- 强制使用开发者使用`ViewHolder`，提供了`onCreateViewHolder`和`onBindViewHolder`这两个方法，把创建`View`和绑定`View`的操作分离开。
- 把焦点交给系统处理。
- 去掉了`onItemClickListener`，以及一些重复的`API`。
- 在`Adapter`中增加了`notifyItemChanged()`等方法，让我们可以指定变化的类型和范围，并且提供了`setItemAnimator()`方法，让开发者能够方便地定义添加、删除、移动的动画。
- 把布局的工作抽象出来，放到了`LayoutManager`当中，并预制了瀑布流布局。

了解了这些，我们就能知道`RecyclerView`能帮我们解决什么问题，也就能更好地理解它为什么要这么设计，下面就开始进入真正的`RecyclerView`的学习。

# 三、`RecyclerView`架构
![](http://upload-images.jianshu.io/upload_images/1949836-be9474752abbf496.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
整个`RecyclerView`体系包含三大组件：
- `LayoutManager`：`position the view`
- `ItemAnimator`：`animate the view`
- `Adapter`：`provide the view`

这三大组件各司其职，而`RecyclerView`负责管理，就组成了整个`RecyclerView`的架构。
## 3.1 `LayoutManager`
`LayoutManager`需要负责以下几部分的工作：
- `Position`
它负责`View`的摆放，可以是线性、宫格、瀑布流式或者任意类型，而`RecyclerView`不知道也不关心这些，这是`LayoutManager`的职责。
- `Scroll`
对于滚动事件的处理，`RecyclerView`负责接收事件，但是最终还是由`LayoutManager`进行处理滚动后的逻辑，因为只有它在知道`View`具体摆放的位置。
- `Focus traversal`
当焦点转移导致需要一个新的`Item`出现在可视区域中时，也是由`LayoutManager`处理的。

## 3.2 `Adapter`
`Adapter`需要负责以下几部分的工作：
- 创建`View`和`ViewHolder`，后者作为整个复用机制的跟踪单元。
- 把具体位置的`Item`和`ViewHolder`进行绑定，并存储相关的信息。
- 通知`RecyclerView`数据变化，支持局部的更新，在提高效率的同时也有效地支持了动画。
- `Item`点击事件的处理。
- 多类型布局的支持。

# 四、`ViewHolder`的生命周期
## 4.1 `LayoutManager`请求`RecyclerView`提供指定`position`的`View`
`ViewHolder`是和`View`相绑定的，同时它也是整个复用框架的跟踪单元。在`RecyclerView`体系中，对`ViewHolder`采用了二级缓存，分为`Cache`和`Recycled Pool`，当`LayoutManager`向`RecyclerView`请求位于某个`Position`的`View`时，`Recycled View`会先去`Cache`中寻找，如果找到，那么直接返回；如果找不到，那么再去`Recycled Pool`中寻找，下面就是整个寻找过程的几种情况：
- 命中`Cache`
这种情况下，不会调用`Adapter`的`onCreateViewHolder`或者`onBindViewHolder`方法：
![](http://upload-images.jianshu.io/upload_images/1949836-a37f09d9e89688c8.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
- `Cache`不存在，`Recycled Pool`也不存在
这种情况下，会调用`Adapter`的`onCreateViewHolder`方法，让它提供一个对应`viewType`的`ViewHolder`，我们在其中建立`ViewHolder`和`View`之间的关联。
![](http://upload-images.jianshu.io/upload_images/1949836-15dbd6842926d475.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
- `Cache`不存在，`Recycled Pool`存在
这种情况下，会回调`Adapter`的`onBindViewHolder`方法，我们在其中使用当前的数据集合来更新`ViewHolder`所绑定的`itemView`的状态。
![](http://upload-images.jianshu.io/upload_images/1949836-7d2ccb23089cfc21.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 4.2 `LayoutManager`找到对应位置的`View`
`LayoutManager`通过`addView`方法把之前找到的`View`添加进`RecyclerView`，`RecyclerView`通过`onViewAttachToWindow(VH viewHolder)`方法，通知`Adapter`这个`viewHolder`所关联的`itemView`已经被添加到了布局当中，
![](http://upload-images.jianshu.io/upload_images/1949836-66c5387b73233253.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
## 4.3 `LayoutManager`请求`RecyclerView`移除某一个位置的`View`
### 4.3.1 普通情况
当`LayoutManager`发现不再需要某一个`position`的`View`时，它会通知`RecyclerView`，`RecyclerView`通过`onViewDetachFromWindow(VH viewHolder)`通知`Adapter`和它绑定的`itemView`被移出了。同时，`RecyclerView`判断它是否能够被缓存，假设能够被缓存，那么它会先被放到`Cache`当中，在`Cache`中又会判断它内部是否有需要转移到`Recycled Pool`中的`ViewHolder`，在放入之后回收池后，通过`onViewRecycled(VH viewHolder)`方法通知`Adapter`它被回收了。
![](http://upload-images.jianshu.io/upload_images/1949836-81e6ffb86f8175d2.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
### 4.3.2  特殊情况
在上面的普通的情况中，`onViewDetachFromWindow(VH viewHolder)`是立即被回调的。然而在实际当中，由于我们需要对`View`的添加、删除做一些过度动画，这时候，我们需要等待`ItemAnimator`进行完动画操作之后，才做`detach`和`recycle`的逻辑，这一过程对于`LayoutManager`是不可见的。
![](http://upload-images.jianshu.io/upload_images/1949836-5ba58d576f731088.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
## 4.4 `ViewHolder`的销毁
在一般情况下，我们不会去销毁`ViewHolder`，而是把它放入到缓存当中，除非出现以下两种情况。
### 4.4.1 `ViewHolder`所绑定的`itemView`当前状态异常
在放入`Recycled Pool`时，会去检查`itemView`的状态是否正常。这一操作的目的主要是为了避免出现诸如此类的情况：当前`itemView`正在执行动画，此时它可能呈现半透明的状态，如果此时把它放入到回收池中，那么当另一个位置的`position`需要复用它时就可能会出现问题。
当出现上面的情况后，`Recycled Pool`会先通过`Adapter`的`onFailedToRecycled(VH viewHolder)`告诉它我们现在出现了异常的情况，由`Adapter`的实现者通过返回值来决定是否仍然要把它放入到`Recycled Pool`，默认是返回`false`，也就是不放入，那么这个`ViewHolder`就会被销毁了。
![](http://upload-images.jianshu.io/upload_images/1949836-2c4c6488ac029e5d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
### 4.4.2 `Recycled Pool`中已经没有足够的空间
`Recycled Pool`的空间并不是无限大的，因此，如果没有足够的空间存放要被回收的`ViewHolder`，那么它也会被销毁。
![](http://upload-images.jianshu.io/upload_images/1949836-3d2601f5460e680b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
造成这种情况的一般是动画引起的，例如，我们调用了`notifyItemRangeChanged(0, getItemCount())`方法，这时候为了进行渐出渐进的动画，那么我们就需要创建两倍的`ViewHolder`，出现这种情况时一般有两种解决方法：
- 只通知具体发生变化的`Item`
- 通过`pool.setMaxRecycledViews(type, count)`改变回收池的大小。

# 五、`ItemAnimator`
对于`Item`的动画，主要有以下几种情况：
- 添加：`Fade In`
- 删除：`Fade Out`
- 移动：`Translate`
- 更新：`Cross Fade`

`RecyclerView`对于动画的处理采用了`Predictive`的方式，除了当前已经在`RecyclerView`布局中的`View`（实线框部分），它还需要知道在屏幕意外的信息（虚线框部分），这样在`H`被删除的时候，它才能够对`J-K`进行上移动画，并把原来不在屏幕内的`L`上移到可视范围之内。
![](http://upload-images.jianshu.io/upload_images/1949836-ba8d94a410a41429.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
# 六、`ChildHelper`和`AdapterHelper`
## 6.1 `ChildHelper` 
对于`ChildHelper`的作用是：`Provide a virtual children list to layoutmanager`，下面我们就首先看一下为什么需要它。
### 6.1.1 解决什么问题
我们看下面这种情况，假如`LayoutManager`想要移除一个`View`，而`ItemAnimator`又希望给这一移除的操作增加一个动画，那么这时候就会产生冲突，到底应该怎么办，为此，`RecyclerView`通过`ChildHelper`来把它们隔离开。
![](http://upload-images.jianshu.io/upload_images/1949836-1d313d48796ac165.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
### 6.1.2 解决问题的方法
当`RecyclerView`收到`LayoutManager`要求改变布局的请求时，它并不是直接去更改`ViewGroup`，而是让`ChildHelper`和`ItemAnimator`去协调，并由它来操作`ViewGroup`。
![](http://upload-images.jianshu.io/upload_images/1949836-bc3383f0a38a7323.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
最明显的例子是，假如我们当前列表中状态为`0,1,2,3`，此时我们移除了`position=0`的`Item`，这时候假如删除的动画还没有完成，那么`LayoutManager`和`RecyclerView`的`getChildAt(0)`返回值将会不同，因为在`LayoutManager`并不清楚`ChildHelper`的存在，在它看来，`position=0`的`Item`已经被移除了。
```
layoutManager.getChildAt(0); //return 1;
recyclerView.getChildAt(0); //return 0;
```
## 6.2 `AdapterHelper`
而`AdapterHelper`所解决的问题和`ChildHelper`类似，`ChildHelper`是处理`View`的，而`AdapterHelper`用来跟踪`ViewHolder`的，其作用为：
- `Tracks ViewHolder positions`
- `Virtual Adapter for LayoutManager`

说起来可能比较抽象，我们用下面这种图理解一下，当我们移动某个`Item`并且它的`onLayout`方法还没有完成，那么`Adapter`和`Layout`的`postion`是不相同的：
![](http://upload-images.jianshu.io/upload_images/1949836-3f7a9c8d10b64043.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
# 七、`ItemDecoration`
`ItemDecoration`用来在`RecyclerView`的`Canvas`上进行额外的绘制操作，我们不仅可以在单个`Item`（例如给每个`Item`添加分割线）的`Canvas`上进行绘制，也可以在整个`RecyclerView`的`Canvas`上进行绘制，此外，我们还可以指定`Item`之间的间隔：
- `Custom Drawing on RecyclerViews Canvas`
- `Add offset to View bounds`
- `Have multiple ItemDecoration`

需要注意的点：
- `Do not try to access to adapter`
- `Keep necessary information in viewHolder`
- `General onDraw rules apply`
- `recyclerView.getChildViewHolder(View view)`

参考文章：[Android RecyclerView 使用完全解析 体验艺术般的控件](http://blog.csdn.net/lmj623565791/article/details/45059587)。

# 八、`RecycledViewPool`
`RecyclerViewPool`用来缓存那些回收的`View`，这些缓存不仅可以提供给单个`RecyclerView`使用，还可以提供和别的自定义控件共享。
- `Sanctuary for reserve ViewHolders`
- `Can be shared between RecyclerViews or Custom ViewGroups`
- `PerActivity Context`

# 九、`ItemTouchHelper`
之前使用`ListView`的时候，如果需要支持侧滑删除、拖动排序这种操作，那么我们一般用引入一些开源库，现在`RecyclerView`已经帮我们提供了实现的接口，通过重写`ItemTouchHelper`的方法，就可以实现上面提到的那些操作。
- `Drag & Drop`
- `Swipe to dismiss`

参考文章：[RecyclerView 进阶：使用 ItemTouchHelper 实现拖拽和侧滑删除](http://www.jianshu.com/p/0c1984bc9383)
# 十、`Tips`
- `onBind Position != final`，`use holder.getAdapterPostion()`
如果我们像下面这样，在`onBindViewHolder`中绑定了监听：
```
public void onBindViewHolder(final ViewHolder, final int position) {
    holder.itemView.setOnClickListener(new View.onClickListener) {
        @Override
        public void onClick(View view) {
            removeAtPostion(position);
        }
    }
}
```
由于`Item`会被添加、删除、移动，因此，我们在`onBindViewHolder`中获得位置，并不一定是当前的位置，例如像下面这样：
```
onBindViewHolder(holder, 5);
notifyItemMoved(5, 15);
holder.itemView.callOnClick();
```
那么就会得到错误的位置，这时候应当使用`holder.getAdapterPostion()`来保证能够得到预期的结果。
- `Payloads`
通过`onBindViewHolder`中的`List payloads`，我们可以指定在`bind`的时候只更新某一部分的信息，而不是全部更新。
- `onCreate means create`
在`onCreateViewHolder`中，始终应当返回一个新的`ViewHolder`，而不是返回一个缓存的`ViewHolder`。
- `Adapter position and Layout position`
就像我们前面在`AdapterHelper`中讨论的那样，在某些时刻，`Adapter Position`和`Layout Position`并不相等，我们应当根据情况选择需要使用哪个，`Adapter Position`**数据所处的位置**，而`Layout Position`则对应当前**`View`的所处的位置**。
