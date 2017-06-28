---
title: RecyclerView 知识梳理(2) - Adapter
date: 2017-04-02 14:29
categories : RecyclerView 知识梳理
---
# 一、概述
当我们使用`RecyclerView`时，第一件事就是要继承于`RecyclerView.Adapter`，实现其中的抽象方法，来处理数据的展示逻辑，今天，我们就来介绍一下`Adapter`中的相关方法。

# 二、基础用法
我们从一个简单的线性列表布局开始，介绍`RecyclerView.Adapter`的基础用法。
首先，需要导入远程依赖包：
```
 compile'com.android.support:recyclerview-v7:25.3.1'
```
接着，继承于`RecyclerView.Adapter`来实现自定义的`NormalAdapter`：
```
public class NormalAdapter extends RecyclerView.Adapter<NormalAdapter.NormalViewHolder> {

    private List<String> mTitles = new ArrayList<>();

    public NormalAdapter(List<String> titles) {
        mTitles = titles;
    }

    @Override
    public NormalViewHolder onCreateViewHolder(ViewGroup parent, int viewType) {
        View itemView = LayoutInflater.from(parent.getContext()).inflate(R.layout.layout_normal_item, parent, false);
        return new NormalViewHolder(itemView);
    }

    @Override
    public void onBindViewHolder(NormalViewHolder holder, int position) {
        holder.setTitle(mTitles.get(position));
    }

    @Override
    public int getItemCount() {
        return mTitles.size();
    }

    class NormalViewHolder extends RecyclerView.ViewHolder {

        private TextView mTextView;

        NormalViewHolder(View itemView) {
            super(itemView);
            mTextView = (TextView) itemView.findViewById(R.id.tv_title);
        }

        void setTitle(String title) {
            mTextView.setText(title);
        }

    }
}
```
当我们实现自己的`Adapter`时，至少要做四个工作：
- 第一：继承于`RecyclerView.ViewHolder`，编写自己的`ViewHolder`
 - 这个子类用来描述`RecyclerView`中每个`Item`的布局以及和它关联的数据，它同时也是`RecyclerView.Adapter<VH>`中需要指定的`VH`类型。
 - 在构造方法中，除了需要调用`super(View view)`方法来传入`Item`的跟布局来给基类中`itemView`变量赋值，还应当提前执行`findViewById`来获得其中的子`View`以便我们之后对它们进行更新。
- 第二：实现`onCreateViewHolder(ViewGroup parent, int viewType)`
 - 当`RecyclerView`需要我们提供类型为`viewType`的新`ViewHolder`时，会回调这个方法。
 - 在这里，我们实例化出了`Item`的根布局，并返回一个和它绑定的`ViewHolder`。
- 第三：实现`onBindViewHolder(VH viewHolder, int position)`
 - 当`RecyclerView`需要展示对应`position`位置的数据时会回调这个方法。
 - 通过`viewHolder`中持有的对应`position`上的`View`，我们可以更新视图。
- 第四：实现`getItemCount()`
 - 返回`Item`的总数。

在`Activity`中，我们给`Adapter`传递数据，使用方法和`ListView`基本相同，只是多了一句在设置`LayoutManager`的操作，这个我们后面再分析。
```
    private void init() {
        RecyclerView recyclerView = (RecyclerView) findViewById(R.id.rv_content);
        mTitles = new ArrayList<>();
        for (int i = 0; i < 20; i++) {
            mTitles.add("My name is " + i);
        }
        NormalAdapter normalAdapter = new NormalAdapter(mTitles);
        recyclerView.setLayoutManager(new LinearLayoutManager(this));
        recyclerView.setAdapter(normalAdapter);
    }
```
这样，一个`RecyclerView`的例子就完成了：
![](http://upload-images.jianshu.io/upload_images/1949836-eb023196e2c5609d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

# 三、只有一种`ViewType`下的复用情况分析
下面，我们来分析一下两个关键方法的调用时机：
- `onCreateViewHolder`
- `onBindViewHolder`

通过这两个方法回调的时机，我们可以对`RecyclerView`复用的机制有一个大概的了解。
## 3.1 初始进入
刚开始进入界面的时候，我们只展示了`3`个`Item`，此时这两个方法的调用情况如下，可以看到，`RecyclerView`只实例化了屏幕内可见的`ViewHolder`，并且`onBindViewHolder`是在对应的`onCreateViewHolder`调用完后立即调用的：
![](http://upload-images.jianshu.io/upload_images/1949836-29642cd798d3724c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
## 3.2 开始滑动
当我们手指触摸到屏幕，并开始向下滑动，我们会发现，虽然`position=3`的`Item`还没有展示出来，但是这时候它的`onCreateViewHolder`和`onBindViewHolder`就被回调了，也就是说，我们会**预加载一个屏幕以外的`Item`**：
![](http://upload-images.jianshu.io/upload_images/1949836-0ccb49a4f8513a26.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
## 3.3 继续滑动
当我们继续往下滑动，`position=3`的`Item`一被展示，那么`position=4`的`Item`的两个方法就会被回调。
## 3.4 复用
当`postion=6`的`Item`被展示之后，按照前面的分析，这时候就应当回调`position=7`的`onCreateViewHolder`和`onBindViewHolder`方法了，但是我们发现，这时候只回调了`onBindViewHolder`方法，而传入的`ViewHolder`其实是`position=0`的`ViewHolder`，也就是我们所说的复用：
![](http://upload-images.jianshu.io/upload_images/1949836-19f1689db78c0f95.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
此时，屏幕中`Items`的展现情况为：
![](http://upload-images.jianshu.io/upload_images/1949836-1635ec093d45c60f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
目前不可见的`Item`为`position=0,1,2`，所以，我们可以得出结论：在单一布局的情况，`RecyclerView`在复用的时候，会取相反方向中超出显示范围的第`3`个`Item`来复用，而并不是超出显示范围的第一个`Item`进行复用。

# 四、多种类型的布局
## 4.1 基本使用
当我们需要在列表当中展示不同类型的`Item`时，我们一般需要重写下面的方法，告诉`RecyclerView`在对应的`position`上需要展示什么类型的`Item`。
- `public int getItemViewType(int position)`

`RecyclerView`在回调`onCreateViewHolder`的时候，同时也会把`viewType`传递进来，我们根据`viewType`来创建不同的布局。
下面，我们就来演示一下它的用法，这里我们返回三种不同类型的`item`：
```
public class NormalAdapter extends RecyclerView.Adapter<NormalAdapter.NormalViewHolder> {

    private List<String> mTitles = new ArrayList<>();

    public NormalAdapter(List<String> titles) {
        mTitles = titles;
    }

    @Override
    public NormalViewHolder onCreateViewHolder(ViewGroup parent, int viewType) {
        View itemView = null;
        switch (viewType) {
            case 0:
                itemView = LayoutInflater.from(parent.getContext()).inflate(R.layout.layout_normal_item_1, parent, false);
                break;
            case 1:
                itemView = LayoutInflater.from(parent.getContext()).inflate(R.layout.layout_normal_item_2, parent, false);
                break;
            case 2:
                itemView = LayoutInflater.from(parent.getContext()).inflate(R.layout.layout_normal_item_3, parent, false);
                break;

        }
        NormalViewHolder viewHolder = new NormalViewHolder(itemView);
        Log.d("NormalAdapter", "onCreateViewHolder, address=" + viewHolder.toString());
        return viewHolder;
    }

    @Override
    public void onBindViewHolder(NormalViewHolder holder, int position) {
        Log.d("NormalAdapter", "onBindViewHolder, address=" + holder.toString() + ",position=" + position);
        int viewType = getItemViewType(position);
        String title = mTitles.get(position);
        holder.setTitle1("title=" + title + ",viewType=" + viewType);
    }

    @Override
    public int getItemCount() {
        return mTitles.size();
    }

    @Override
    public int getItemViewType(int position) {
        return position % 3;
    }

    class NormalViewHolder extends RecyclerView.ViewHolder {

        private TextView mTv1;

        NormalViewHolder(View itemView) {
            super(itemView);
            mTv1 = (TextView) itemView.findViewById(R.id.tv_title_1);
        }

        void setTitle1(String title) {
            mTv1.setText(title);
        }

    }
}
```
最终，会得到下面的界面：
![](http://upload-images.jianshu.io/upload_images/1949836-28329e877d338536.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
## 4.2 多种`viewType`下的复用情况分析
前面，我们已经研究过一种`viewType`下的复用情况，现在，我们再来分析一下多种`viewType`时候的复用情况。
### 4.2.1 初始进入
此时，我们屏幕中展示了`postion=0~6`这七个`Item`，`onCreateViewHolder`和`onBindViewHolder`的回调和之前相同，只会生成屏幕内可见的`ViewHolder`
![](http://upload-images.jianshu.io/upload_images/1949836-a901eb0c4a19d88c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
### 4.2.2 开始滑动和继续滑动
这两种情况都和单个`viewType`时相同，会预加载屏幕以外的一个`Item`：
![](http://upload-images.jianshu.io/upload_images/1949836-c9cdeaced873ad40.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
### 4.2.3 复用
关键，我们看一下何时会复用`position=0/viewType=1`的`Item`：
![](http://upload-images.jianshu.io/upload_images/1949836-3c3dc9064e3d33c5.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
此时，屏幕内最上方的`Item`为`position=4/viewType=1`，最下方的`Item`为`position=11/viewType=2`，按照之前的分析，`RecyclerView`会保留相反方向的`2`个`ViewHolder`，也就是保留`postion=2,3`的`ViewHolder`，并复用`position=1`的`ViewHolder`，但是现在`position=0`的`ViewHolder`的`viewType=1`，不可以复用，因此，会继续往上寻找，这时候就找到了`position=0`的`ViewHolder`进行复用。

# 五、数据更新
## 5.1 更新方式
当数据源发生变化的时候，我们一般会通过`Adatper. notifyDataSetChanged()`来进行界面的刷新，`RecyclerView.Adapter`也提供了相同的方法：
```
public final void notifyDataSetChanged() 
```
除此之外，它还提供了下面几种方法，让我们进行局部的刷新：
```
//position的数据变化
notifyItemChanged(int postion)
//在position的下方插入了一条数据
notifyItemInserted(int position)
//移除了position的数据
notifyItemRemoved(int postion)
//从position开始，往下n条数据发生了改变
notifyItemRangeChanged(int postion, int n)
//从position开始，插入了n条数据
notifyItemRangeInserted(int position, int n)
//从position开始，移除了n条数据
notifyItemRangeRemoved(int postion, int n)
```
下面是一些简单的使用方法：
```
   //在头部添加多个数据.
   public void addItems() {
        mTitles.add(0, "add Items, name=0");
        mTitles.add(0, "add Items, name=1");
        mNormalAdapter.notifyItemRangeInserted(0, 2);
    }
    //移除头部的多个数据.
    public void removeItems() {
        mTitles.remove(0);
        mTitles.remove(0);
        mNormalAdapter.notifyItemRangeRemoved(0, 2);
    }
    //移动数据.
    public void moveItems() {
        mTitles.remove(1);
        mTitles.add(2, "move Items name=0");
        mNormalAdapter.notifyItemMoved(1, 2);
    }
```
## 5.2 比较
数据的更新分为两种：
- `Item changes`：除了`Item`所对应的数据被更新外，没有其它的变化，对应`notifyXXXChanged()`方法。
- `Structural changes`：`Items`在数据集中被插入、删除或者移动，对应`notifyXXXInsert/Removed/Moved`方法。

`notifyDataSetChanged`会把当前所有的`Item`和结构都视为已经失效的，因此它会让`LayoutManager`重新绑定`Items`，并对他们重新布局，这在我们知道已经需要更新某个`Item`的时候，其实是不必要的，这时候就可以选择进行局部更新来提高效率。

# 六、监听`ViewHolder`的状态
`RecyclerView.Adapter`中还提供了一些回调，让我们能够监听某个`ViewHolder`的变化：
```
    @Override
    public void onViewRecycled(NormalViewHolder holder) {
        Log.d("NormalAdapter", "onViewRecycled=" + holder);
        super.onViewRecycled(holder);
    }

    @Override
    public void onViewDetachedFromWindow(NormalViewHolder holder) {
        Log.d("NormalAdapter", "onViewDetachedFromWindow=" + holder);
        super.onViewDetachedFromWindow(holder);
    }

    @Override
    public void onViewAttachedToWindow(NormalViewHolder holder) {
        Log.d("NormalAdapter", "onViewAttachedToWindow=" + holder);
        super.onViewAttachedToWindow(holder);
    }
```
下面，我们就从实例来讲解这几个方法的调用时机，初始时刻，我们的界面为：
![](http://upload-images.jianshu.io/upload_images/1949836-d6e9e393ac2ce71f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
- 初始进入时，`position=0~6`的`onViewAttachedToWindow`被回调：
![](http://upload-images.jianshu.io/upload_images/1949836-e22d52c8962557f2.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
- 当滑动到`postion=7`可见时，它的`onViewAttachedToWindow`被回调：
![](http://upload-images.jianshu.io/upload_images/1949836-41b54bbecad8547b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
- 当`postion=0`被移出屏幕可视范围内，它的`onViewDetachedFromWindow`被回调：
![](http://upload-images.jianshu.io/upload_images/1949836-ed8314954647ab8b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
- 而当我们继续往下滑动，当`position=2`被移出屏幕之后，此时`position=0`的`onViewRecycled`被回调：
![](http://upload-images.jianshu.io/upload_images/1949836-c9da891b61471714.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
现在回忆一下之前我们对复用情况的分析，`RecyclerView`最多会保留相反方向上的两个`ViewHolder`，此时虽然`position=1,2`不可见，但是依然需要保留它们，这时候会回收`position=0`的`ViewHolder`以备之后被复用。

# 七、监听`RecyclerView`和`RecyclerView.Adapter`的关系
`RecyclerView`和`Adapter`是通过`setAdapter`方法来绑定的，因此在`Adapter`中也通过了绑定的监听：
```
public void onAttachedToRecyclerView(RecyclerView recyclerView) {}
public void onDetachedFromRecyclerView(RecyclerView recyclerView) {}
```
# 八、小结
这篇文章，主要总结了一些`RecyclerView.Adapter`中平时我们不常注意的细节问题，也通过实例了解到了关键方法的含义，最后，推荐一个`Adapter`的开源库：[`BaseRecyclerViewAdapterHelper`](https://github.com/CymChad/BaseRecyclerViewAdapterHelper)。
