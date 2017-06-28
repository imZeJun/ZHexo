---
title: RecyclerView 知识梳理(3) - LayoutManager
date: 2017-04-02 21:31
categories : RecyclerView 知识梳理
---
# 一、概述
在前面的学习中，我们已经对`Adapter`有了大概的了解，在整个`RecyclerView`的体系当中，`Adapter`负责提供`View`，而`LayoutManager`负责决定它们在`RecyclerView`中摆放的位置以及在窗口中不可见之后的回收策略。今天，我们来一起看一下`LayoutManager`的相关知识。
# 二、`LayoutManager`的使用
通过重写`LayoutManager`，我们可以得到各式各样的布局。官方提供了以下三种`LayoutManager`：
- `LinearLayoutManager`
- `GirdLayoutManager`
- `StaggeredGridLayoutManager`

下面，我们就来一起学习它们的使用方式。
## 2.1 线性布局：`LinearLayoutManager`
使用这个`LinearLayoutManager`时，所有的`Item`都是线性排列的，我们可以指定以下两点。
## 2.1.1 纵向/横向排列
- 纵向排列：`LinearLayoutManager.OrientationHelper.VERTICAL`：
![](http://upload-images.jianshu.io/upload_images/1949836-708fc5eea76ca4c5.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
- 横向排列：`LinearLayoutManager.OrientationHelper.HORIZONTAL`：
![](http://upload-images.jianshu.io/upload_images/1949836-5914a99a2f2d31a9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 2.1.2 `Item`排列的顺序和滑动方向
通过`reverse`指定`Items`排列的顺序：
- `true`：从右向左或从下到上排列，也就是`position=0`的`Item`位于最右边或最下面，往左或者往上滑动得到下一个`Item`。
![](http://upload-images.jianshu.io/upload_images/1949836-4eec73fd0a20f13b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
- `false`：和上面相反，也就是我们常见的模式。

## 2.2 宫格布局：`GirdLayoutManager`
### 2.2.1 指定某行或某一列的个数
通过`spanCount`参数指定，相当于把`RecyclerView`的每行或者每列均分为`spanCount`个格子，每个`Item`可以占据一个或者多个格子，默认情况下每个`Item`占据一个格子，也就是均分。
### 2.2.2 纵向/横向排列
- 纵向排列：先填满一行，再从下一行开始填充。
- 横向排列：先填满一列，在从下一列开始填充。

### 2.2.3 `reverse`参数
指定了`Items`排列的顺序：
- `reverse=true`：逆序排列所有的`Item`，和`2.1.2`的排列方式有关，如果是纵向排列，那么`position=0`的`Item`位于左下角，如果是横向排列，那么位于右上角。
- `reverse=false`：`position=0`的`Item`位于左上角。

### 2.2.4 指定分配的比例
上面我们说过，`spanCount`指定的是分配的格子数，默认情况下每个`Item`会占据一个格子，如果想要改变每一行或者每一列`Item`分配的比例，那么可以指定它们占据的格子数，如果该行或者该列剩余的格子不够分配了，那么就换行，但是一定不能够大于`spanCount`的值：
```
public void setSpanSizeLookup(SpanSizeLookup spanSizeLookup) {
    mSpanSizeLookup = spanSizeLookup;
}
```
下面是一个使用的例子：
```
   private void init() {
        mTitles = new ArrayList<>();
        for (int i = 0; i < 40; i++) {
            mTitles.add(String.valueOf(i));
        }
        RecyclerView recyclerView = (RecyclerView) findViewById(R.id.rv_content);
        GridLayoutManager layoutManager = new GridLayoutManager(this, 3, GridLayoutManager.VERTICAL, false);
        layoutManager.setSpanSizeLookup(new GridLayoutManager.SpanSizeLookup() {

            @Override
            public int getSpanSize(int position) {
                return position % 3 == 0 ? 1 : 2;
            }
        });
        recyclerView.setLayoutManager(layoutManager);
        LayoutManagerAdapter adapter = new LayoutManagerAdapter(mTitles);
        recyclerView.setAdapter(adapter);
    }
```
最终得到效果为：
![](http://upload-images.jianshu.io/upload_images/1949836-3ae97abb6c43da39.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 2.3 瀑布流：`StaggeredGridLayoutManager`
### 2.3.1 指定`spanCount`
和宫格布局类似，可以指定每行或者每列划分的格子数，但是它不支持让某个`Item`占据多个格子。
### 2.3.2 横向或者纵向排列
这个和上面两个`LayoutManager`的原理类似，就不解释了。
### 2.3.3 实战
默认情况下，如果我们只生成一个`StaggeredGridLayoutManager`，那么效果会是下面这样：
![](http://upload-images.jianshu.io/upload_images/1949836-3794cab6c1b37aa3.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
因此，我们要在`onBindViewHolder`中动态地改变每个`itemView`的高度，这样才可以达到瀑布流的效果：
```
    @Override
    public void onBindViewHolder(LayoutManagerViewHolder holder, int position) {
        holder.setTitle(mTitles.get(position));
        ViewGroup.LayoutParams layoutParams = holder.itemView.getLayoutParams();
        layoutParams.height = 200 + (position % 4) * 200;
        holder.itemView.setBackgroundColor(holder.itemView.getResources().getColor(COLOR[position % 5]));
        holder.itemView.setLayoutParams(layoutParams);
    }
```
之后，我们会得到下面的效果：
![](http://upload-images.jianshu.io/upload_images/1949836-0ced1cb90115114a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

# 三、小结
平时的开发当中，这几个布局已经基本能够满足我们的需求，如果需要了解自定义`LayoutManager`，那么需要对`RecyclerView`的整个机制就很好的了解，在分析完原理之后，我们在详细讲解自定义`LayoutManager`的方法。
