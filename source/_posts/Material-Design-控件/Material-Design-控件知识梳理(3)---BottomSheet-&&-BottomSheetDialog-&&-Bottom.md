---
title: Material Design 控件知识梳理(3) - BottomSheet && BottomSheetDialog && BottomSheetDialogFragment
date: 2017-04-12 17:06
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
今天，我们介绍三种底部菜单的实现方式，不同于之前的弹窗，它们支持通过手势对底部菜单的布局进行拖动，来显示或者隐藏，类似于下面的效果：
![](http://upload-images.jianshu.io/upload_images/1949836-af25cafbf35f7c60.gif?imageMogr2/auto-orient/strip)
这三个布局各有特点：
- `BottomSheet`
依赖于`CoordinatorLayout`和`BottomSheetBehavior`，需要将底部菜单布局作为`CoordinatorLayout`的子`View`，实现简单但不够灵活，适用于**底部菜单布局稳定**的情况。
- `BottomSheetDialog`
使用方式类似于`Dialog`，适用于需要**动态指定底部菜单布局**的情况。
- `BottomSheetDialogFragment`
通过继承于`BottomSheetFragment`来实现底部菜单布局，适用于需要**动态指定布局，并根据`Fragment`的生命周期做较多逻辑操作**的情况。

# 二、`BottomSheet`详解
`BottomSheet`需要依赖于`CoordinatorLayout`，采用`BottomSheet`的时候，我们的布局一般类似于下面这样：
![](http://upload-images.jianshu.io/upload_images/1949836-a97e435dc766d765.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
```

<android.support.design.widget.CoordinatorLayout 
    xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    tools:context="com.demo.lizejun.repotransition.BottomSheetActivity">
    <!-- 其它布局 -->
    <!-- 底部菜单布局 -->
    <include layout="@layout/layout_bottom_sheet_linear"/>
</android.support.design.widget.CoordinatorLayout>

```
下面，我们用两种方式来实现`BottomSheet`的底部菜单：
- `LinearLayout`
- `RecyclerView`

## 2.1 `LinearLayout`实现的`BottomSheet`
在进行实例演示之前，我们先介绍`BottomSheet`的五种状态：
- `STATE_DRAGGING`：手指在`BottomSheet`上下拖动从而使得布局跟着上下移动。
- `STATE_SETTLING`：当手指抬起之后，会根据当前的偏移量，决定是要将`BottomSheet`收起还是展开。

这两种属于中间态，类似于`ViewPager`的`SCROLL_STATE_DRAGGING `和`SCROLL_STATE_SETTLING `。

- `STATE_EXPANDED`：展开。
- `STATE_COLLAPSED`：收起。
- `STATE_HIDDEN`：隐藏。

这三种属于稳定态，当`BottomSheet`稳定下来，最终都会恢复到上面三种状态之一，展开很容易理解，需要区别的是收起和隐藏：
- 隐藏：意味着整个底部布局完全不可见，默认情况下没有这种状态，需要设置`app:behavior_hideable="true"`
- 收起：我们可以设置收起时的高度，让它仍然部分可见，并可以通过拖动这部分布局让它进入到展开状态，收起时的高度通过`app:behavior_peekHeight`设置。

下面是我们底部菜单的`LinearLayout`，也就是上面`include`的布局：
```
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout
    xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    android:id="@+id/bottom_sheet"
    android:layout_width="match_parent"
    android:layout_height="wrap_content"
    android:orientation="vertical"
    app:behavior_hideable="true"
    app:behavior_peekHeight="66dp"
    app:layout_behavior="@string/bottom_sheet_behavior">
    <TextView
        android:layout_width="match_parent"
        android:layout_height="66dp"
        android:background="@android:color/holo_green_dark" />
    <TextView
        android:background="@android:color/holo_orange_dark"
        android:layout_width="match_parent"
        android:layout_height="66dp" />
    <TextView
        android:background="@android:color/holo_green_dark"
        android:layout_width="match_parent"
        android:layout_height="66dp" />
    <TextView
        android:background="@android:color/holo_orange_dark"
        android:layout_width="match_parent"
        android:layout_height="66dp" />
    <TextView
        android:background="@android:color/holo_green_dark"
        android:layout_width="match_parent"
        android:layout_height="66dp" />
    <TextView
        android:background="@android:color/holo_orange_dark"
        android:layout_width="match_parent"
        android:layout_height="66dp" />
</LinearLayout>
```
这个底部菜单的根布局中有三个关键的属性：
- 必须设置：`layout_behavior`，只有设置了这个才能达到`BottomSheet`的效果，否则和放置一个普通布局没有区别：
```
app:layout_behavior="@string/bottom_sheet_behavior"
```
- 必须设置：收起高度，否则在监听偏移的时候会出现异常：
```
app:behavior_peekHeight="66dp"
```
- 可选设置：是否支持隐藏，如果设置为`false`，或者没有设置，那么不允许调用`setState(BottomSheetBehavior.STATE_HIDDEN)`
```
app:behavior_hideable="true"
```

这样，一个底部菜单布局的声明就完成了，接下来，我们通过`BottomSheetBehavior`来管理这个菜单：
```
    private View mBottomLayout;
    private BottomSheetBehavior mBottomSheetBehavior;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_bottom_sheet);
        //1.通过id获得底部菜单布局的实例
        mBottomLayout = findViewById(R.id.bottom_sheet);
        //2.把这个底部菜单和一个BottomSheetBehavior关联起来
        mBottomSheetBehavior = BottomSheetBehavior.from(mBottomLayout);
    }
```
通过这个`Behavior`，我们可以实现底部菜单的展开、隐藏和收起：
```
    public void expandBottomSheet(View view) {
        mBottomSheetBehavior.setState(BottomSheetBehavior.STATE_EXPANDED);
    }

    public void hideBottomSheet(View view) {
        mBottomSheetBehavior.setState(BottomSheetBehavior.STATE_HIDDEN);
    }

    public void collapseBottomSheet(View view) {
        mBottomSheetBehavior.setState(BottomSheetBehavior.STATE_COLLAPSED);
    }
```
![](http://upload-images.jianshu.io/upload_images/1949836-1c62bc30fb8ecbf1.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
除此之外，我们还可以通过`BottomSheetBehavior`监听底部菜单的滑动变化：
```
        mBottomSheetBehavior.setBottomSheetCallback(new BottomSheetBehavior.BottomSheetCallback() {

            @Override
            public void onStateChanged(@NonNull View bottomSheet, int newState) {
                Log.d("BottomSheet", "newState=" + newState);
            }

            @Override
            public void onSlide(@NonNull View bottomSheet, float slideOffset) {
                Log.d("BottomSheet", "onSlide=" + slideOffset);
            }
        });
```
第一个回调函数用来监听`BottomSheet`状态的改变，也就是我们上面所说到的五种状态，而`onSlide`回调当中的`slideOffset`则用来监听底部菜单的偏移量：
- 当处于展开状态时，偏移量为`1`
- 当处于收起状态时，偏移量为`0`
- 当处于隐藏状态时，偏移量为`-1`

## 2.2 `RecyclerView`实现的`BottomSheet`
如果我们列表内的`Items`较多，那么可以考虑使用`RecyclerView`来实现底部菜单，实现方式和前面类似，这里，我们最好固定`RecyclerView`的高度：
```
<android.support.v7.widget.RecyclerView
    xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    android:id="@+id/bottom_sheet"
    android:background="@android:color/darker_gray"
    android:layout_width="match_parent"
    android:layout_height="300dp"
    android:orientation="vertical"
    app:behavior_hideable="true"
    app:behavior_peekHeight="60dp"
    app:layout_behavior="@string/bottom_sheet_behavior"/>
```
当使用这种方式，如果初始时候处于收起状态，那么当手指上滑时，会优先让底部菜单慢慢进入展开状态，当完全进入展开状态之后，开始让列表向底部滚动。而当手指下滑时，优先让列表向顶部滚动，当滚动到顶部之后，让菜单从展开状态慢慢进入到收起状态。
![](http://upload-images.jianshu.io/upload_images/1949836-e995b8ea474e4812.gif?imageMogr2/auto-orient/strip)
这整个过程，`state`和`slideOffset`的变化趋势为：
![](http://upload-images.jianshu.io/upload_images/1949836-5f1fd51600ab51ca.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

# 三、`BottomSheetDialog`详解
`BottomSheet`的使用非常简单，但是也有它的局限性，它要求我们要实现预定根布局为`CoordinatorLayout`，同时它也没有平时我们使用弹框时的阴影效果，下面，我们介绍另一种实现方式：`BottomSheetDialog`，它的使用方式和我们平时使用`Dialog`时很类似，但是它增加了通过手势展开和收起对话框的操作。
```
public class BottomSheetDialogActivity extends AppCompatActivity {

    private BottomSheetDialog mDialog;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_bottom_sheet_dialog);
    }

    public void showDialog(View view) {
        mDialog = new BottomSheetDialog(this);
        mDialog.setContentView(R.layout.layout_bottom_sheet_dialog);
        mDialog.show();
    }

    public void hideDialog(View view) {
        if (mDialog != null && mDialog.isShowing()) {
            mDialog.hide();
        }
    }
}
```
`BottomSheetDialog`继承于`Dialog`，当我们通过`setContentView`方法传入自定义布局的时候，它会将这个布局使用`CoordinatorLayout`包裹起来，所以当使用`BottomSheetDialog`的时候，底部菜单和根布局并不属于同一个`window`：
![](http://upload-images.jianshu.io/upload_images/1949836-dfa352b9220f186a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
而我们的`Dialog`内部的布局其实是这样的：
![](http://upload-images.jianshu.io/upload_images/1949836-0fbecb8ca4abea55.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
由上图可以看出，**`Dialog`的根节点其实并不是通过`setContentView()`传入的`View`，它实际上是用`CoordinatorLayout`把它包装了起来**，这才实现了拖动展开和隐藏的行为。
# 四、`BottomSheetDialogFragment`
`BottomSheetDialogFragment`继承于`DialogFragment`，并重写了`onCreateDialog`返回我们上一节所说的`BottomSheetDialog`，它的使用方法和`DialogFragment`相同：
```
public class BottomSheetDialogFragment extends AppCompatDialogFragment {

    @Override
    public Dialog onCreateDialog(Bundle savedInstanceState) {
        return new BottomSheetDialog(getContext(), getTheme());
    }

}
```
下面，我们看一下如何使用：
- 首先，定义`Fragment`：

```
public class DemoBottomSheetDialogFragment extends BottomSheetDialogFragment {

    public static DemoBottomSheetDialogFragment newInstance() {
        return new DemoBottomSheetDialogFragment();
    }

    @Nullable
    @Override
    public View onCreateView(LayoutInflater inflater, @Nullable ViewGroup container, @Nullable Bundle savedInstanceState) {
        return inflater.inflate(R.layout.layout_bottom_sheet_dialog, container, false);
    }

}
```

- 弹出和收起的方式
```
public class BottomSheetDialogFragmentActivity extends AppCompatActivity {

    private DemoBottomSheetDialogFragment mDemoBottomSheetDialogFragment;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_bottom_sheet_dialog_fragment);
    }

    public void showDialog(View view) {
        mDemoBottomSheetDialogFragment = DemoBottomSheetDialogFragment.newInstance();
        mDemoBottomSheetDialogFragment.show(getSupportFragmentManager(), "demoBottom");
    }

    public void hideDialog(View view) {
        if (mDemoBottomSheetDialogFragment != null) {
            mDemoBottomSheetDialogFragment.dismiss();
        }
    }
}
```
最终它的布局和`BottomSheetDialog`一样，都是在一个新的`Window`当中：
![](http://upload-images.jianshu.io/upload_images/1949836-dd6229bb846b659c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
# 五、总结
通过上面的学习，我们发现，其实这三种方法最核心的就是使用了`CoordinatorLayout + bottom_sheet_behavior`：
- `BottomSheet`需要我们自己去声明`CoordinatorLayout`布局，并把底部菜单作为它的子`View`。
- `BottomSheetDialog`和`BottomSheetDialogFragment`，只需要我们提供一个底部菜单的布局，在它们内部的实现当中，它再把我们传入的布局放入到`CoordinatorLayout`当中，之后再把这一整个包装好的布局作为`Dialog`的布局。
