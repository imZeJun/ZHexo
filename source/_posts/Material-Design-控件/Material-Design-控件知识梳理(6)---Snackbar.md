---
title: Material Design 控件知识梳理(6) - Snackbar
date: 2017-04-14 23:52
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
`Snackbar`的作用和之前使用的`Toast`类似，都是作为一种轻量级的用户提示：
![](http://upload-images.jianshu.io/upload_images/1949836-f0c5abb0212ff4a5.gif?imageMogr2/auto-orient/strip)
但是和`Toast`相比，它又增加了一些额外的交互操作，今天我们就一起来学习一下有关`Snackbar`的知识。
# 二、`Snackbar`的基础使用
当我们需要使用`Snackbar`时，首先需要调用它的`make`静态方法来获得一个`Snackbar`对象，之后我们对于`Snackbar`的操作都是通过这个对象：
```
    public void showSnackBar(View view) {
        mSnackBarRootView = Snackbar.make(mCoordinatorLayout, "MessageView", Snackbar.LENGTH_INDEFINITE);
        mSnackBarRootView.setActionTextColor(getResources().getColor(android.R.color.holo_orange_dark));
        mSnackBarRootView.setAction("ActionView", new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                Log.d("mSnackBarRootView", "click ActionView");
            }
        });
        mSnackBarRootView.show();
    }
```
## 外观设置
对于`Snackbar`，可以分为两个区域，`MessageView`和`ActionView`，其中`MessageView`只支持设置文案，而`ActionView`不仅支持设置文案，还支持设置文案的颜色以及监听点击事件，具体的方法大家可以查阅`API`：
![](http://upload-images.jianshu.io/upload_images/1949836-4e43e4adcba019f6.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
## 操作方式
- 显示`Snackbar`时，需要像上面一样主动调用`show()`方法。
- 隐藏`Snackbar`时，有以下几种操作方式：
 - 主动调用`dismiss`方法
 - 点击`ActionView`
 - 从左向右滑动`Snackbar`
 - 通过`setDuration`方法，让`Snackbar`在经过指定的时间之后自动隐藏

# 三、`Snackbar`进阶
## 3.1 改变`Snackbar`的外观
从上面可以看出，`Snackbar`对于外观的支持不够充分，比如不能定义`MessageView`的颜色、以及整个`Snackbar`的背景等等，下面，我们就来看一下如何对它的外观进行进一步的定制。
源码当中对`Snackbar`对象的初始化分为了下面三步操作：
```
    public static Snackbar make(@NonNull View view, @NonNull CharSequence text,
            @Duration int duration) {
        //1.寻找Snackbar的父容器
        final ViewGroup parent = findSuitableParent(view);
        if (parent == null) {
            throw new IllegalArgumentException("No suitable parent found from the given view. "
                    + "Please provide a valid view.");
        }

        final LayoutInflater inflater = LayoutInflater.from(parent.getContext());
        //2.实例化出Snackbar的布局
        final SnackbarContentLayout content =
                (SnackbarContentLayout) inflater.inflate(
                        R.layout.design_layout_snackbar_include, parent, false);
        //3.利用父容器和Snackbar的布局，构造Snackbar对象
        final Snackbar snackbar = new Snackbar(parent, content, content);
        snackbar.setText(text);
        snackbar.setDuration(duration);
        return snackbar;
    }
```
通过查看布局，可以发现`SnackbarContentLayout`就是包含了`MessageView`和`ActionView`的父容器，也就是例子当中的黑色背景：
![](http://upload-images.jianshu.io/upload_images/1949836-cafc2a974ebc70dd.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
那么有没有什么办法能够获得这个`SnackbarContentLayout`呢，我们看一下`Snackbar`的构造函数：
```
protected BaseTransientBottomBar(@NonNull ViewGroup parent, 
    @NonNull View content, 
    @NonNull ContentViewCallback contentViewCallback) {
        //这个是我们传入的mCoordinatorLayout.
        mTargetParent = parent;
        //mView是mCoordinatorLayout的子View，同时又是SnackbarContentLayout的父容器
        mView = (SnackbarBaseLayout) inflater.inflate(
                R.layout.design_layout_snackbar, mTargetParent, false);
        mView.addView(content);
    }
```
而`Snackbar`提供了`getView`方法来得到`mView`对象，也就是布局中的`Snackbar$SnackbarLayout`：
```
public View getView() {
    return mView;
}
```
那么整个逻辑就很清楚了，我们可以通过通过`mView`获得`SnackbarContentLayout`，然后进行一系列的定制：
- 改变`Snackbar`的背景：
![](http://upload-images.jianshu.io/upload_images/1949836-3665d1c02fa8e997.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
```
private void changeSnackBarBackgroundColor(Snackbar snackbar) {
    View view = snackbar.getView();
    view.setBackgroundColor(getResources().getColor(android.R.color.holo_purple));
}
```
- 改变`MessageView`字体的颜色和大小：
![](http://upload-images.jianshu.io/upload_images/1949836-ba1c81d0c0adee14.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
```
    private void changeSnackBarMessageViewTextColor(Snackbar snackbar) {
        ViewGroup viewGroup = (ViewGroup) snackbar.getView();
        SnackbarContentLayout contentLayout = (SnackbarContentLayout) viewGroup.getChildAt(0);
        TextView textView = (TextView) contentLayout.getChildAt(0);
        textView.setTextColor(getResources().getColor(android.R.color.darker_gray));
    }
```

## 3.2 `Snackbar`弹出位置分析
`Snackbar`会弹出在父容器的底部，也就是上面`findSuitableParent`的过程，我们来分析一下这一寻找的过程，就可以知道`Snackbar`弹出的位置：
```
    private static ViewGroup findSuitableParent(View view) {
        ViewGroup fallback = null;
        do {
            if (view instanceof CoordinatorLayout) {
                // We've found a CoordinatorLayout, use it
                return (ViewGroup) view;
            } else if (view instanceof FrameLayout) {
                if (view.getId() == android.R.id.content) {
                    // If we've hit the decor content view, then we didn't find a CoL in the
                    // hierarchy, so use it.
                    return (ViewGroup) view;
                } else {
                    // It's not the content view but we'll use it as our fallback
                    fallback = (ViewGroup) view;
                }
            }

            if (view != null) {
                // Else, we will loop and crawl up the view hierarchy and try to find a parent
                final ViewParent parent = view.getParent();
                view = parent instanceof View ? (View) parent : null;
            }
        } while (view != null);

        // If we reach here then we didn't find a CoL or a suitable content view so we'll fallback
        return fallback;
    }
```
它其实是从我们在`make`方法中传入的`View`作为起点，沿着整个`View`树向上寻找，如果发现是`CoordinatorLayout`或者到达了`R.id.content`，那么就停止寻找，否则将一直到达`View`树的根节点为止，所以，如果我们的`CoordinatorLayout`不是全屏的话，那么`Snackbar`有可能不是弹出在整个屏幕的底部，例如下面这样，我们给`Snackbar`添加了一个`marginBottom`：
```
<?xml version="1.0" encoding="utf-8"?>
<android.support.design.widget.CoordinatorLayout
    xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools"
    android:id="@+id/cl_root"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:layout_marginBottom="200dp"
    tools:context="com.demo.lizejun.repotransition.SnackBarActivity">
    <TextView
        android:id="@+id/show_snack_bar"
        android:text="showSnackBar"
        android:layout_width="match_parent"
        android:layout_height="66dp"
        android:gravity="center"
        android:layout_margin="5dp"
        android:textColor="@android:color/white"
        android:background="@android:color/holo_orange_dark"
        android:onClick="showSnackBar"/>
</android.support.design.widget.CoordinatorLayout>
```
那么弹出的效果为：
![](http://upload-images.jianshu.io/upload_images/1949836-17e681b0b2d83de9.gif?imageMogr2/auto-orient/strip)
## 3.3 `Snackbar`和`FloatingActionButton`的结合
当`Snackbar`弹出的时候，有可能会遮挡底部的`FloatingActionButton`，此时就需要在`make`方法中传入`CoordinatorLayout`，让`Snackbar`弹出的时候，让`Fab`上移一定的高度，可以参考之前的这篇文章：[MD控件 - FloatingActionButton](http://www.jianshu.com/p/5a354e318019)
# 四、总结
`Snackbar`使用起来很简单，它比`Toast`增加了更多的操作性，也是官方推荐的替换`Toast`的控件。
