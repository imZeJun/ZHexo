---
title: Material Design 控件知识梳理(2) - AppBarLayout & CollapsingToolbarLayout
date: 2017-04-10 22:14
categories : Material Design 控件知识梳理
---
>[Material Design 控件知识梳理(1) - Android Design Support Library 是什么](http://www.jianshu.com/p/32b2638a1785)
[Material Design 控件知识梳理(2) - AppBarLayout & CollapsingToolbarLayout](http://www.jianshu.com/p/d4fd636d7c44)
[Material Design 控件知识梳理(3) - BottomSheet && BottomSheetDialog && BottomSheetDialogFragment](http://www.jianshu.com/p/2a5be29123e5)
[Material Design 控件知识梳理(4) - FloatingActionButton](http://www.jianshu.com/p/5a354e318019)
[Material Design 控件知识梳理(5) - DrawerLayout && NavigationView](http://www.jianshu.com/p/d70cfd724c7f)
[Material Design 控件知识梳理(6) - Snackbar](http://www.jianshu.com/p/6aea94f9ae2f)
[Material Design 控件知识梳理(7) - BottomNavigationBar](http://www.jianshu.com/p/22ec4fc1cb71)
[Material Design 控件知识梳理(8) - TabLayout](http://www.jianshu.com/p/5dd04fda13ab)
[Material Design 控件知识梳理(9) - TextInputLayout](http://www.jianshu.com/p/ed9642fc8634)

# 一、概述
在某些`App`当中，我们经常会见到类似于下面的这种设计：
![](http://upload-images.jianshu.io/upload_images/1949836-0ff0b7ba0b40806f.gif?imageMogr2/auto-orient/strip)
这种设计的目的是为了在初始时刻展示重要的信息，但是当用户需要查看列表的时候，不至于被封面占据过多的可视的区域，`Google`为这种设计提供了几个类，让我们只用实现很少的代码就能够实现这种效果，避免了我们通过去监听列表的滚动状态来去改变头部区域的显示，这篇文章，我们就一步步来学习如何实现这种效果。
要实现上面这种效果，需要对下面几方面的知识有所了解：
- `CoordinatorLayout`
- `AppBarLayout`
- `CollapsingToolbarLayout`
- 实现了`NestedScrollingChild`接口的滚动布局，例如`RecyclerView`、`NestedScrollView`等

这四者的关系可以用下面这张图来表示：
![](http://upload-images.jianshu.io/upload_images/1949836-537ba9a66e5b59a9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

其中`CollapsingToolbarLayout`需要依赖于`AppBarLayout`，因此我打算把文章的讨论分为两部分：
- 只采用`AppBarLayout`实现
- 采用`AppBarLayout + CollapsingToolbarLayout`实现

# 二、只采用`AppBarLayout`实现
当只采用`AppBarLayout`时，我们的布局层次一般是下面这样：
![](http://upload-images.jianshu.io/upload_images/1949836-23b90898e3033d08.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
## 2.1 `CoordinatorLayout`
`CoordinatorLayout`是一个重写的`ViewGroup`，它负责`AppBarLayout`以及可滚动布局之间的关系，因此，它是作为这两者的直接父控件。
## 2.2 `AppBarLayout`下的列表
一般是`RecyclerView`或者`ViewPager`，为了能让它和`AppBarLayout`协同工作，需要**给列表控件添加下面这个属性，这样列表就会显示在`AppBarLayout`的下方**：
```
app:layout_behavior="@string/appbar_scrolling_view_behavior"
```
## 2.3 `AppBarLayout`的标志位
需要**将`AppBarLayout`作为`CoordinatorLayout`的直接子`View`**，同时它也是一个`LinearLayout`，因此它的所有子`View`都是线性排列的，而它对子`View`的滚动显示管理是通过子`View`的`app:layout_scrollFlags`属性，注意，**这个标志位是在`AppBarLayout`的子`View`中声明的，而不是`AppBarLayout`中**。

`app:layout_scrollFlags`的值有下面五种可选：
- `scroll`
- `exitUntilCollapsed`
- `enterAlways`
- `enterAlwaysCollapsed `
- `snap`

## 2.3.1 `scroll`
使得`AppBarLayout`的子`View`伴随着滚动而收起或者展开，有两点需要注意：
- 这个标志位是其它四个标志位生效的前提
- 带有`scroll`标志位的子`View`必须放在`AppBarLayout`中的最前面

现在我们给`RecyclerView`设置了`app:layout_behavior`，而`AppBarLayout`中的两个子`View`都没有设置`scroll`标志位：
```
<?xml version="1.0" encoding="utf-8"?>
<android.support.design.widget.CoordinatorLayout
    xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    android:layout_width="match_parent"
    android:layout_height="match_parent">
    <android.support.design.widget.AppBarLayout
        android:id="@+id/al_title"
        android:layout_width="match_parent"
        android:layout_height="wrap_content">
        <TextView
            android:gravity="center"
            android:text="layout_scrollFlags=scroll"
            android:textColor="@android:color/white"
            android:background="@android:color/holo_green_dark"
            android:layout_width="match_parent"
            android:layout_height="100dp"/>
        <TextView
            android:gravity="center"
            android:text="没有设置layout_scrollFlags"
            android:textColor="@android:color/white"
            android:background="@android:color/holo_orange_dark"
            android:layout_width="match_parent"
            android:layout_height="100dp"/>
    </android.support.design.widget.AppBarLayout>
    <android.support.v7.widget.RecyclerView
        android:id="@+id/rv_content"
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        app:layout_behavior="@string/appbar_scrolling_view_behavior"/>
</android.support.design.widget.CoordinatorLayout>
```
此时`AppBarLayout`中的子`View`都一直固定在`CoordinatorLayout`的顶部：
![](http://upload-images.jianshu.io/upload_images/1949836-47305f37a40a046f.gif?imageMogr2/auto-orient/strip)
下面，我们给`AppBarLayout`中的第一个子`View`加上`app:layout_scrollFlags="scroll|exitUntilCollapsed"`标志位：
```
 <android.support.design.widget.AppBarLayout
        android:id="@+id/al_title"
        android:layout_width="match_parent"
        android:layout_height="wrap_content">
        <TextView
            android:gravity="center"
            android:text="layout_scrollFlags=scroll"
            android:textColor="@android:color/white"
            android:background="@android:color/holo_green_dark"
            android:layout_width="match_parent"
            android:layout_height="100dp"
            app:layout_scrollFlags="scroll"/>  //注意看这里!
        <TextView
            android:gravity="center"
            android:text="没有设置layout_scrollFlags"
            android:textColor="@android:color/white"
            android:background="@android:color/holo_orange_dark"
            android:layout_width="match_parent"
            android:layout_height="100dp"/>
</android.support.design.widget.AppBarLayout>
```
那么此时滑动情况为：
- 设置了`scroll`的子`View`可以在滚动后收起，而没有设置的则不可以。
- 在手指向上移动的时候，优先收起`AppBarLayout`中的可收起`View`，当它处于收起状态时，下面的列表内容才开始向尾部滚动。
- 在手指往下移动的时候，优先让下面的列表内容向顶部滚动，当列表滚动到顶端时，`AppBarLayout`的可收起`View`才展开。
![](http://upload-images.jianshu.io/upload_images/1949836-4460ecdd1e7b0d30.gif?imageMogr2/auto-orient/strip)

## 2.3.2 `exitUntilCollapsed `
对于可收起的子`View`来说有两种状态：`Enter`和`Collapsed`。**手指向上移动的时候，会优先收起`AppBarLayout`中的子`View`，而收起的最小高度为`0`**，假如希望它在收起时仍然保留部分可见，那么就需要使用`exitUntilCollapsed + minHeight`属性，**`minHeight`就决定了收起时的最小高度是多少**，也就是`Collapsed`状态。
```
<android.support.design.widget.AppBarLayout
        android:id="@+id/al_title"
        android:layout_width="match_parent"
        android:layout_height="wrap_content">
        <TextView
            android:gravity="center"
            android:text="layout_scrollFlags=scroll"
            android:textColor="@android:color/white"
            android:background="@android:color/holo_green_dark"
            android:layout_width="match_parent"
            android:layout_height="100dp"
            android:minHeight="50dp"
            app:layout_scrollFlags="scroll|exitUntilCollapsed"/>
        <TextView
            android:gravity="center"
            android:text="没有设置layout_scrollFlags"
            android:textColor="@android:color/white"
            android:background="@android:color/holo_orange_dark"
            android:layout_width="match_parent"
            android:layout_height="100dp"/>
</android.support.design.widget.AppBarLayout>
```
![](http://upload-images.jianshu.io/upload_images/1949836-d4a497862458435a.gif?imageMogr2/auto-orient/strip)
### 2.3.3 `enterAlways`
`exitUntilCollapsed`决定了**手指向上移动时**`AppBarLayout`怎么收起它的子`View`，而`enterAlways`则决定了**手指向下移动时**的行为。默认情况下，在手指向下移动时，会优先让列表滚动到顶部，而如果设置了`enterAlways`，那么会优先让`AppBarLayout`中的子`View`滚动到展开。
```
<android.support.design.widget.AppBarLayout
        android:id="@+id/al_title"
        android:layout_width="match_parent"
        android:layout_height="wrap_content">
        <!-- 需要设置属性 -->
        <TextView
            android:gravity="center"
            android:text="layout_scrollFlags=scroll|enterAlways"
            android:textColor="@android:color/white"
            android:background="@android:color/holo_green_dark"
            android:layout_width="match_parent"
            android:layout_height="100dp"
            app:layout_scrollFlags="scroll|enterAlways"/>
        <TextView
            android:gravity="center"
            android:text="没有设置layout_scrollFlags"
            android:textColor="@android:color/white"
            android:background="@android:color/holo_orange_dark"
            android:layout_width="match_parent"
            android:layout_height="100dp"/>
</android.support.design.widget.AppBarLayout>
```
![](http://upload-images.jianshu.io/upload_images/1949836-342e04d7a33ae7b8.gif?imageMogr2/auto-orient/strip)
### 2.3.4 `enterAlwaysCollapsed`
`enterAlwaysCollapsed`，`enterAlways`以及`minHeight`属性一起配合使用，采用这个标志位，主要是为了使得手指向下移动时出现下面这个效果：
- 优先滚动`AppBarLayout`中的子`View`，但是并不是滚动到全部展开，而是只滚动到`minHeight`
- `AppBarLayout`中的子`View`滚动到`minHeight`之后，开始滚动列表，直到列表滚动到头部
- 列表滚动到头部之后，开始滚动`AppBarLayout`中的子`View`，直到它完全展开

注意和`exitUntilCollapsed `的`minHeight`区分开来；
- `enterAlways|enterAlwaysCollapsed`的`minHeight`，决定的是**手指向下移动且列表没有滚动到顶端**的**最大高度**
- `exitUntilCollapsed`的`minHeight`，决定的是**手指向上移动**时的**最小高度**

这两个标志位和`minHeight`是相互依赖的关系，如果声明对应的标志位，那么`minHeight`是没有任何意义的；而如果只声明了标志位，没有声明`minHeight`，那么默认`minHeight`为`0`。

```
<android.support.design.widget.AppBarLayout
        android:id="@+id/al_title"
        android:layout_width="match_parent"
        android:layout_height="wrap_content">
        <!-- 需要设置属性 -->
        <TextView
            android:gravity="center"
            android:text="layout_scrollFlags=scroll|enterAlways|enterAlwaysCollapsed, minHeight=50dp"
            android:textColor="@android:color/white"
            android:background="@android:color/holo_green_dark"
            android:layout_width="match_parent"
            android:layout_height="100dp"
            android:minHeight="50dp"
            app:layout_scrollFlags="scroll|enterAlways|enterAlwaysCollapsed"/>
        <TextView
            android:gravity="center"
            android:text="没有设置layout_scrollFlags"
            android:textColor="@android:color/white"
            android:background="@android:color/holo_orange_dark"
            android:layout_width="match_parent"
            android:layout_height="100dp"/>
</android.support.design.widget.AppBarLayout>
```
![](http://upload-images.jianshu.io/upload_images/1949836-2c63914954616250.gif?imageMogr2/auto-orient/strip)
### 2.3.5 `snap`
默认情况下，在手指从屏幕上抬起之后，`AppBarLayout`中子`View`的状态就不会变化了，而如果我们设置了`snap`标志位，那么在手指抬起之后，会根据子`View`当前的偏移量，决定是让它变为收起还是展开状态。
```
<android.support.design.widget.AppBarLayout
        android:id="@+id/al_title"
        android:layout_width="match_parent"
        android:layout_height="wrap_content">
        <!-- 需要设置属性 -->
        <TextView
            android:gravity="center"
            android:text="layout_scrollFlags=scroll|snap"
            android:textColor="@android:color/white"
            android:background="@android:color/holo_green_dark"
            android:layout_width="match_parent"
            android:layout_height="100dp"
            app:layout_scrollFlags="scroll|snap"/>
        <TextView
            android:gravity="center"
            android:text="没有设置layout_scrollFlags"
            android:textColor="@android:color/white"
            android:background="@android:color/holo_orange_dark"
            android:layout_width="match_parent"
            android:layout_height="100dp"/>
</android.support.design.widget.AppBarLayout>
```
![](http://upload-images.jianshu.io/upload_images/1949836-21e872414217f7fd.gif?imageMogr2/auto-orient/strip)

## 2.4 监听`AppBarLayout`的滚动状态
`AppBarLayout`提供了监听滚动状态的接口，我们可以根据这个偏移值来改变界面的状态：
```
private void setAppBar() {
        final TextView moveView = (TextView) findViewById(R.id.iv_move_title);
        AppBarLayout appBarLayout = (AppBarLayout) findViewById(R.id.al_title);
        appBarLayout.addOnOffsetChangedListener(new AppBarLayout.OnOffsetChangedListener() {

            @Override
            public void onOffsetChanged(AppBarLayout appBarLayout, int verticalOffset) {
                Log.d("AppBarLayout", "offset=" + verticalOffset);
                int height = moveView.getHeight();
                int minHeight = moveView.getMinHeight();
                float fraction = verticalOffset / (float) (minHeight - height);
                moveView.setAlpha(1 - fraction);
            }

        });
}
```
这里，我们根据`offset`的值来改变`alpha`，最终达到下面的效果：
![scroll_7.gif](http://upload-images.jianshu.io/upload_images/1949836-18cecd7767b4b8b0.gif?imageMogr2/auto-orient/strip)
在手指往上移动的过程当中，`verticalOffset`从`0`变为负值：
![](http://upload-images.jianshu.io/upload_images/1949836-3e36e33a7ebb06c9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 2.6 `AppBarLayout`小结
`AppBarLayout`主要是掌握两点：
- `app:layout_scrollFlags`几种标志位之前的区别
- 如何监听`AppBarLayout`的滚动

掌握完这两点之后，我们就可以自由发挥，做出自己想要的效果了。
# 三、`AppBarLayout`和`CollapsingToolbarLayout`结合
`CollapsingToolbarLayout`用于对`Toolbar`进行包装，因此，它是`AppBarLayout`的直接子`View`，同时也是`Toolbar`的直接父`View`，当使用`CollapsingToolbarLayout`时我们的界面布局一般是这样的：
![](http://upload-images.jianshu.io/upload_images/1949836-1234b39e6e2a0f61.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
`CollapsingToolbarLayout`的标志比较多，而且需要和`Toolbar`相结合，下面，我们就以一种比较常见的例子，来讲解几个比较重要的标志位，先看效果：
![](http://upload-images.jianshu.io/upload_images/1949836-881e7fa4a273a18a.gif?imageMogr2/auto-orient/strip)
`AppBarLayout`的布局如下：
```
<android.support.design.widget.AppBarLayout
        android:id="@+id/al_title"
        android:layout_width="match_parent"
        android:layout_height="wrap_content">
        <android.support.design.widget.CollapsingToolbarLayout
            android:id="@+id/ctl_title"
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            app:contentScrim="@color/colorPrimaryDark"
            app:layout_scrollFlags="scroll|exitUntilCollapsed">
            <ImageView
                android:id="@+id/iv_title"
                android:src="@drawable/ic_bg"
                android:layout_width="match_parent"
                android:scaleType="centerCrop"
                android:layout_height="150dp"
                app:layout_collapseParallaxMultiplier="0.5"
                app:layout_collapseMode="parallax"/>
            <android.support.v7.widget.Toolbar
                android:id="@+id/toolbar"
                android:title="@string/app_name"
                android:layout_width="match_parent"
                android:layout_height="50dp"
                app:title="Collapse"
                app:navigationIcon="@android:drawable/ic_media_play"
                app:layout_collapseMode="pin"/>
        </android.support.design.widget.CollapsingToolbarLayout>
</android.support.design.widget.AppBarLayout>
```
下面，我们就来介绍上面这个布局中比较重要的标志位：
- `app:contentScrim`
设置收起之后`CollapsingToolbarLayout`的颜色，正如例子中的那样，在收起之后，`ImageView`的背景被我们设置的`contentScrim`所覆盖。
- `app:layout_scrollFlags="scroll|exitUntilCollapsed"`
`scroll`标志位是必须设置的，`exitUntilCollapsed`保证了我们在手指上移的时候，`CollapsingToolbarLayout`最多收起到`Toolbar`的高度，使得它始终保持可见。
- `app:layout_collapseMode="parallax"`和`app:layout_collapseParallaxMultiplier="0.5"`
`layout_collapseMode`有两种模式，`parallax`表示视差效果，简单地说，就是当`CollapsingToolbarLayout`滑动的距离不等于背景滑动的距离，从而产生一种视差效果，而视差效果的大小由`app:layout_collapseParallaxMultiplier `决定。
- `app:layout_collapseMode="pin"`
`layout_collapseMode`的另一种模式，它使得`Toolbar`一直固定在顶端。

除了上面这几个重要的标志位之外，相信大家还发现了`CollapsingToolbar`在全部展开的时候会把标题设置为比较大，而当上移时，标题慢慢缩小为`Toolbar`的标题大小，并移动到`Toolbar`的标题所在位置。

# 四、总结
以上就是对于使用`CoordinatorLayout`、`AppBarLayout`、`CollapsingToolbarLayout`实现标题跟随列表滚动的简要介绍，最主要的是掌握`layout_scrollFlags`几种标志之间的区别。

# 五、参考文献
[Material Design之 AppbarLayout 开发实践总结](http://www.jianshu.com/p/ac56f11e7ce1)
