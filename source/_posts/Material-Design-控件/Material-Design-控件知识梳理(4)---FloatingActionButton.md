---
title: Material Design 控件知识梳理(4) - FloatingActionButton
date: 2017-04-13 20:16
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
今天，我们介绍一个比较简单的控件：`FloatingActionButton`，相信大家一定都听过也在网上见过类似的例子，我们就分为三个部分介绍一下`FloatingActionButton`的相关知识：
- `Fab`的基础使用
- `Fab`和其它`MD`控件的组合
- 通过自定义`FloatingActionButton.Behavior`，让`Fab`根据列表的状态显示和隐藏

# 二、`Fab`的基础使用
`Fab`本质上其实是一个`ImageButton`，只是它在`ImageButton`的基础上增加了一些属性，这里介绍几个常用的属性：

## 展现属性
- `android:src`：`Fab`的图片，这其实是`ImageView`的属性。
- `app:backgroundTint`：`Fab`的背景色，如果没有设置，那么会取`theme`中的`colorAccent`作为背景色。
- `app:fabSize` ：`Fab`的大小，可选的值包括：
 - `mini`：
 - `normal`
 - `auto`：`mini`和`normal`都预设了固定的大小，而`auto`属性则会根据屏幕的宽度来设置，在小屏幕上使用`mini`，而在大屏幕上使用`normal`，当然我们也可以直接通过`layout_width/layout_height`来指定。
- `app:elevation`：`Fab`在`Z`轴方向的距离，也就是深度。
- `app:borderWidth`：`Fab`边界的宽度，边界的颜色会比背景色稍淡，如下图所示
![](http://upload-images.jianshu.io/upload_images/1949836-da87f7c332a8b4f4.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 点击属性
- `app:pressedTranslationZ`：点击时`Fab`在`Z`轴的变化值。
- `app:rippleColor`：点击时水波纹扩散的颜色。

# 三、与其它`MD`控件结合使用
## 3.1 和`AppBarLayout`联动
通过给`Fab`设置`app:layout_anchor`和`layout_anchorGravity`两个属性，可以让`Fab`跟随`AppBarLayout`移动，并在合适的时候隐藏，下面是我们的布局：
```
<android.support.design.widget.CoordinatorLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    tools:context="com.demo.lizejun.repotransition.FABActivity">
    <android.support.design.widget.AppBarLayout
        android:id="@+id/al_title"
        android:layout_width="match_parent"
        android:layout_height="wrap_content">
        <ImageView
            android:id="@+id/iv_title"
            android:src="@drawable/ic_bg"
            android:layout_width="match_parent"
            android:scaleType="centerCrop"
            android:layout_height="150dp"
            app:layout_scrollFlags="scroll"/>
    </android.support.design.widget.AppBarLayout>
    <android.support.v7.widget.RecyclerView
        android:id="@+id/rv_content"
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        app:layout_behavior="@string/appbar_scrolling_view_behavior"/>
    <android.support.design.widget.FloatingActionButton
        android:id="@+id/fab"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:src="@android:drawable/ic_btn_speak_now"
        android:layout_margin="10dp"
        app:backgroundTint="@color/colorPrimary"
        app:layout_anchor="@id/al_title" 
        app:layout_anchorGravity="bottom|end"/>
</android.support.design.widget.CoordinatorLayout>
```
当我们滚动布局的时候，`Fab`会跟着`AppBarLayout`先上移，然后消失：
![](http://upload-images.jianshu.io/upload_images/1949836-27fc91bd32eee7c6.gif?imageMogr2/auto-orient/strip)
## 3.2 和`BottomSheet`联动
和`AppBarLayout`类似，我们也可以通过设置`layout_anchor`的方法让`Fab`和`BottomSheet`联动：
```
<android.support.design.widget.CoordinatorLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    tools:context="com.demo.lizejun.repotransition.FABActivity">
    <android.support.design.widget.AppBarLayout
        android:id="@+id/al_title"
        android:layout_width="match_parent"
        android:layout_height="wrap_content">
        <ImageView
            android:id="@+id/iv_title"
            android:src="@drawable/ic_bg"
            android:layout_width="match_parent"
            android:scaleType="centerCrop"
            android:layout_height="150dp"
            app:layout_scrollFlags="scroll"/>
    </android.support.design.widget.AppBarLayout>
    <android.support.v7.widget.RecyclerView
        android:id="@+id/rv_content"
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        app:layout_behavior="@string/appbar_scrolling_view_behavior"/>
    <include android:id="@+id/bottom_sheet" layout="@layout/layout_bottom_sheet_linear"/>
    <android.support.design.widget.FloatingActionButton
        android:id="@+id/fab"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:src="@android:drawable/ic_btn_speak_now"
        android:layout_margin="10dp"
        app:backgroundTint="@color/colorPrimary"
        app:layout_anchor="@id/bottom_sheet"
        app:layout_anchorGravity="end"/>
</android.support.design.widget.CoordinatorLayout>
```
这里，我们把`layout_anchor`设为`bottom_sheet`：
![](http://upload-images.jianshu.io/upload_images/1949836-5d8cbfbee63d7181.gif?imageMogr2/auto-orient/strip)
## 3.3 和`Snackbar`联动
需要和`Snackbar`联动时，不需要`Fab`设置`layout_anchor`，而是需要在`Snackbar`展现的时候，第一个参数传入的是`CoordinatorLayout`：
```
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_fab);
        initView();
        mRootView = (CoordinatorLayout) findViewById(R.id.cl_root);
        mFab = (FloatingActionButton) findViewById(R.id.fab);
        mFab.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                //这里需要传入CoordinatorLayout
                Snackbar.make(mRootView, "点击Fab", Snackbar.LENGTH_LONG).show();
            }
        });

    }
```
下面是展现的效果：
![](http://upload-images.jianshu.io/upload_images/1949836-d0b08b586a1d1f06.gif?imageMogr2/auto-orient/strip)

# 四、`Fab`根据列表的状态显示或隐藏
在`Fab`的内部，定义了一个`Behavior`，我们可以通过继承这个`Behavior`来监听`CoordinatorLayout`内布局的变化，以实现`Fab`的显示和隐藏，首先看我们的布局：
```
<android.support.design.widget.CoordinatorLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    xmlns:tools="http://schemas.android.com/tools"
    android:id="@+id/cl_root"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    tools:context="com.demo.lizejun.repotransition.FABActivity">
    <android.support.v7.widget.RecyclerView
        android:id="@+id/rv_content"
        android:tag="rv_content"
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        app:layout_behavior="@string/appbar_scrolling_view_behavior"/>
    <android.support.design.widget.FloatingActionButton
        android:id="@+id/fab"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:src="@android:drawable/ic_btn_speak_now"
        android:layout_margin="10dp"
        android:layout_gravity="bottom|end"
        app:backgroundTint="@color/colorPrimary"
        app:layout_behavior="com.demo.lizejun.repotransition.behavior.FabListBehavior"/>
</android.support.design.widget.CoordinatorLayout>
```
我们的根布局是一个`CoordinatorLayout`，`RecyclerView`和`Fab`是它的两个子`View`，`Fab`位于`CoordinatorLayout`的右下角，注意到，这里我们给`Fab`设置了一个自定义的`Behavior`，正是通过这个`behavior`，`Fab`可以监听到`CoordinatorLayout`内布局的滚动情况，下面是我们的`Behavior`：
```
public class FabListBehavior extends FloatingActionButton.Behavior {

    private static final int MIN_CHANGED_DISTANCE = 30;

    public FabListBehavior(Context context, AttributeSet attributeSet) {
        super(context, attributeSet);
    }

    @Override
    public boolean onStartNestedScroll(CoordinatorLayout coordinatorLayout, FloatingActionButton child, View directTargetChild, View target, int nestedScrollAxes) {
        return true;
    }

    @Override
    public void onNestedScroll(CoordinatorLayout coordinatorLayout, FloatingActionButton child, View target, int dxConsumed, int dyConsumed, int dxUnconsumed, int dyUnconsumed) {
        super.onNestedScroll(coordinatorLayout, child, target, dxConsumed, dyConsumed, dxUnconsumed, dyUnconsumed);
        if (dyConsumed > MIN_CHANGED_DISTANCE) {
            createValueAnimator(coordinatorLayout, child, false).start();
        } else if (dyConsumed < -MIN_CHANGED_DISTANCE) {
            createValueAnimator(coordinatorLayout, child, true).start();
        }
    }

    private Animator createValueAnimator(CoordinatorLayout coordinatorLayout, final View fab, boolean dismiss) {
        int distanceToDismiss = coordinatorLayout.getBottom() - fab.getBottom() + fab.getHeight();
        int end = dismiss ? 0 : distanceToDismiss;
        float start = fab.getTranslationY();
        ValueAnimator animator = ValueAnimator.ofFloat(start, end);
        animator.addUpdateListener(new ValueAnimator.AnimatorUpdateListener() {
            @Override
            public void onAnimationUpdate(ValueAnimator animation) {
                fab.setTranslationY((Float) animation.getAnimatedValue());
            }
        });
        return animator;
    }

}
```
这里，我们继承于`FloatingActionButton.Behavior`来实现自己的`Behavior`，注意，必须要声明构造函数为`FabListBehavior(Context, AttributeSet)`，并调用`super()`方法，否则会无法实例化：
- `onStartNestedScroll`决定了之后是否需要回调`onNestedScroll`，这里我们直接返回`true`。
- `onNestedScroll`：我们根据`dyConsumed`的正负值来判断列表滚动的方法，然后通过改变`Fab`的`translationY`来让它移入或者移出屏幕。
![](http://upload-images.jianshu.io/upload_images/1949836-34a8b9f4b82cea05.gif?imageMogr2/auto-orient/strip)
# 五、总结
这篇文章，介绍了`Fab`的基本用法、其它`MD`控件的联动以及如何自定义`FloatingActionButtonBehavior`。
