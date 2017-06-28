---
title: Material Design 控件知识梳理(7) - BottomNavigationBar
date: 2017-04-15 12:12
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
当某个主页面有多个子页面时，我们一般会采用`ViewPager`来承载这些子页面，并会提供一组选项卡让用户通过点击对应的选项的方式来进行页面之间的快速切换，而这一组选项卡根据摆放位置的不同，一般可以分为下面两种实现方式：
- 放在顶部，采用`TabLayout`
- 放在底部，采用`BottomNavigationBar`

今天，我们介绍第二种方式。
# 二、`BottomNavigationBar`详解
## 2.1 基本用法
引入依赖包：
```
compile 'com.ashokvarma.android:bottom-navigation-bar:1.2.0'
```
在布局当中引入控件，这里，我们将它放置在容器的底部：
```
<android.support.design.widget.CoordinatorLayout 
    xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    tools:context="com.demo.lizejun.repotransition.NavigationBarActivity">
    <com.ashokvarma.bottomnavigation.BottomNavigationBar
        android:id="@+id/bottom_navigation"
        android:layout_gravity="bottom"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"/>
</android.support.design.widget.CoordinatorLayout>
```
在代码中对数据进行初始化：
```
    private void initViews() {
        mBottomNavigationBar = (BottomNavigationBar) findViewById(R.id.bottom_navigation);
        //1.设置Mode
        mBottomNavigationBar.setMode(BottomNavigationBar.MODE_FIXED);
        //2.设置BackgroundStyle
        mBottomNavigationBar.setBackgroundStyle(BottomNavigationBar.BACKGROUND_STYLE_STATIC);
        //3.设置背景色
        mBottomNavigationBar.setBarBackgroundColor(android.R.color.white);
        //4.设置每个Item
        mBottomNavigationBar.addItem(new BottomNavigationItem(R.drawable.ic_1, "Item 1").setActiveColorResource(android.R.color.holo_blue_dark));
        mBottomNavigationBar.addItem(new BottomNavigationItem(R.drawable.ic_2, "Item 2").setActiveColorResource(android.R.color.holo_green_dark));
        mBottomNavigationBar.addItem(new BottomNavigationItem(R.drawable.ic_3, "Item 3").setActiveColorResource(android.R.color.holo_orange_dark));
        mBottomNavigationBar.addItem(new BottomNavigationItem(R.drawable.ic_4, "Item 4").setActiveColorResource(android.R.color.holo_green_dark));
        mBottomNavigationBar.addItem(new BottomNavigationItem(R.drawable.ic_5, "Item 5").setActiveColorResource(android.R.color.holo_orange_dark));
        //5.初始化
        mBottomNavigationBar.initialise();
    }
```
上面代码的运行效果如下图所示：
![](http://upload-images.jianshu.io/upload_images/1949836-6a28d73218df941d.gif?imageMogr2/auto-orient/strip)
可以看到，上面我们设置了很多的属性，下面我们就一一来讲解各个属性的含义。
## 2.2 `BottomNavigationBar`的`Mode`属性
`Mode`的设置对应于这句：
```
mBottomNavigationBar.setMode(BottomNavigationBar.MODE_FIXED);
```
这个属性有两种可选的值，`MODE_FIXED`和`MODE_SHIFTING`
- `MODE_FIXED`：选中的`Item`会稍大于未选中的`Item`，无论`Item`是否选中，都会显示文字和图标。
- `MODE_SHIFTING`：选中的`Item`明显大于未选中的`Item`，未选中的`Item`只显示图标，并且在选中项切换的时候，会有一定的偏移效果。

在`2.1`当中，我们演示的就是第一种方式，下面，我们看一下第二种方式的效果：
```
mBottomNavigationBar.setMode(BottomNavigationBar.MODE_SHIFTING);
```
![](http://upload-images.jianshu.io/upload_images/1949836-cd27685e06c2d86d.gif?imageMogr2/auto-orient/strip)
## 2.3 `BottomNavigationBar`的`BackgroundStyle`属性
`BackgroundStyle`的设置对应于这句：
```    
mBottomNavigationBar.setBackgroundStyle(BottomNavigationBar.BACKGROUND_STYLE_STATIC);
```
这个属性有两个可选的值：
- `BACKGROUND_STYLE_STATIC`
- `BACKGROUND_STYLE_RIPPLE`

这两种选项决定了两点：**整个`BottomNavigationBar`的颜色**和**被选中`Item`的颜色**，在解释这个之前，我们需要先了解一下三种颜色：
- `barBackgroudColor`：只能通过`BottomNavigationBar`来设置
```
mBottomNavigationBar.setBarBackgroundColor(android.R.color.white);
```
- `activeColor`：被激活颜色，可以通过`BottomNavigationBar`来进行全局的设置，也可以给每个`Item`单独设置
```
mBottomNavigationBar.addItem(new BottomNavigationItem(R.drawable.ic_1, "Item 1").setActiveColorResource(android.R.color.holo_blue_dark));
```
- `inActiveColor`：未被激活颜色，可以通过`BottomNavigationBar`来进行全局的设置，也可以给每个`Item`单独设置。

注意到，上面这三种颜色并不是说被选中的`Item`的文字和图标的颜色一定是被激活颜色，这需要根据`BackgroundStyle`来决定，在每种模式下，被选中`Item`的文字图片颜色、未被选中的`Item`的文字图标颜色、整个`BottomNavigationBar`的背景颜色的对应关系为：
![](http://upload-images.jianshu.io/upload_images/1949836-07a8b9965a998612.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
也就是说，`inActiveColor`在任何时候都是未被选中`Item`的文字和图片颜色，而其它两种则不然：
- 在`static`模式下，`activeColor`是被选中`Item`的文字图标颜色，`backgroundColor`为`BottomNavigationBar`的背景色
- 而在`ripple`模式下，恰巧是反过来。

对于`2.1`中例子的`Item 1`，它的`BackgroundStyle`为`BACKGROUND_STYLE_STATIC`，因此在它被选中的时候，文字和图片的颜色为给它设置的`ActiveColor`，而整个`BottomNavigationBar`的背景色为`BackgroundColor`，现在我们看一下`BACKGROUND_STYLE_RIPPLE`的情况：
```       
mBottomNavigationBar.setBackgroundStyle(BottomNavigationBar.BACKGROUND_STYLE_RIPPLE);
```
![](http://upload-images.jianshu.io/upload_images/1949836-807ec898e80dc375.gif?imageMogr2/auto-orient/strip)
## 2.4 给`Item`设置角标
通过`BottomNavigationItem`的`setBadgeItem`方法，可以给每个`Item`设置一个独立的角标，对于角标支持设置它的背景、文案、文案颜色以及在选中时是否隐藏角标：
```
BadgeItem badgeItem = new BadgeItem()
                .setBackgroundColorResource(android.R.color.holo_red_dark) //设置角标背景色
                .setText("5") //设置角标的文字
                .setTextColorResource(android.R.color.white) //设置角标文字颜色
                .setHideOnSelect(true); //在选中时是否隐藏角标
mBottomNavigationBar.addItem(new BottomNavigationItem(R.drawable.ic_5, "Item 5")
    .setActiveColorResource(android.R.color.holo_orange_dark)
    .setBadgeItem(badgeItem));
```
效果如下图所示：
![](http://upload-images.jianshu.io/upload_images/1949836-bbae3b6e91633665.gif?imageMogr2/auto-orient/strip)
## 2.5 监听`Item`的切换
可以通过下面的方法来监听`Item`之间的切换：
```
mBottomNavigationBar.addItem(new BottomNavigationItem(R.drawable.ic_5, "Item 5").setActiveColorResource(android.R.color.holo_orange_dark).setBadgeItem(badgeItem));
mBottomNavigationBar.setTabSelectedListener(new BottomNavigationBar.OnTabSelectedListener() {

            @Override
            public void onTabSelected(int position) {
                Log.d("onTabSelected", "position=" + position);
            }

            @Override
            public void onTabUnselected(int position) {
                Log.d("onTabUnselected", "position=" + position);

            }

            @Override
            public void onTabReselected(int position) {
                Log.d("onTabReselected", "position=" + position);

            }
});
```
- `onTabSelected`，某个`Item`从未选中状态变为选中状态时回调
- `onTabUnselected`，某个`Item`从选中变为未选中时回调
- `onTabReselected`，某个`Item`已经处于选中状态，但是它又被再次点击了，那么回调这个函数。

## 2.6 指定当前选中的位置
指定初始时刻的位置：
```
mBottomNavigationBar.setFirstSelectedPosition(3).initialise();
```
动态改变位置：
```
mBottomNavigationBar.selectTab(2);
```
## 2.7 初始化
在改变设置之后，需要在最后调用下面这句才会生效
```
mBottomNavigationBar.initialise();
```
# 三、`BottomNavigationBar`的显示和隐藏
## 3.1 手动隐藏和显示`BottomNavigationBar`
通过下面的两个方法可以手动显示和隐藏`BottomNavigationBar`：
```
    public void show(View view) {
        mBottomNavigationBar.unHide(true);
    }

    public void hide(View view) {
        mBottomNavigationBar.hide(true);
    }
```
![](http://upload-images.jianshu.io/upload_images/1949836-0fa723d4a230fefb.gif?imageMogr2/auto-orient/strip)
## 3.2 根据列表的滚动来显示和隐藏
如果我们的根布局使用的是`CoordinatorLayout`，那么可以通过给`BottomNavigationBar`设置内置的`Behavior`来实现动态地显示和隐藏，首先继承于这个内置的`Bahavior`，给它指定一个构造函数：
```
public class BottomBehavior extends BottomVerticalScrollBehavior<BottomNavigationBar> {

    public BottomBehavior(Context context, AttributeSet attributeSet) {
        super();
    }

}
```
把这个`Behavior`设置给`BottomNavigationBar`：
```
<android.support.design.widget.CoordinatorLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    tools:context="com.demo.lizejun.repotransition.NavigationBarActivity">
    <android.support.v7.widget.RecyclerView
        android:id="@+id/rv_content"
        android:layout_width="match_parent"
        android:layout_height="match_parent"/>
    <com.ashokvarma.bottomnavigation.BottomNavigationBar
        android:id="@+id/bottom_navigation"
        android:layout_gravity="bottom"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        app:layout_behavior="com.demo.lizejun.repotransition.behavior.BottomBehavior"/>
</android.support.design.widget.CoordinatorLayout>
```
只需要这两部操作，就可以实现动态地显示和隐藏了：
![](http://upload-images.jianshu.io/upload_images/1949836-c6eda5afa04ff926.gif?imageMogr2/auto-orient/strip)
