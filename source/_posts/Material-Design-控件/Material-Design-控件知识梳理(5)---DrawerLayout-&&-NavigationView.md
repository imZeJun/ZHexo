---
title: Material Design 控件知识梳理(5) - DrawerLayout && NavigationView
date: 2017-04-14 20:38
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
今天，我们来介绍两个和侧滑菜单有关的`MD`控件：
- `DrawerLayout`：实现侧滑菜单的基础。
- `NavigationView`：作为侧滑菜单布局的一种实现方式。

# 二、`DrawerLayout`
## 2.1 基本原理
当我们需要使用到侧滑菜单时，可以通过`DrawerLayout`来实现，`DrawerLayout`、侧滑菜单布局、普通布局这三者的关系为：
![](http://upload-images.jianshu.io/upload_images/1949836-6fb023e17962d713.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
`layout_gravity`决定了将哪个菜单作为侧滑布局，`DrawerLayout`会根据是否声明了`layout_gravity`属性，把它内部的直接子`View`分成两类：
- 对于没有声明`layout_gravity`的布局，那么它会将它们当作普通布局，并按照`FrameLayout`的方式来排列它们。
- 对于声明了`layout_gravity="start"`或者`layout_gravity="left"`的布局，在普通情况下会将它们隐藏起来，当从屏幕的最左侧往右移动手指时，这个布局会渐渐展现出来，对于`DrawerLayout`的所有子`View`来说，只允许有一个子`View`的该属性为`start/left`，这就是我们的侧滑布局。
- `layout_gravity="end/right"`和上面类似，只不过它的调出是从屏幕的右侧向左侧移动。

## 2.2 简单事例
下面是一个使用`DrawerLayout`的最简单的例子：
```
<android.support.v4.widget.DrawerLayout
    xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    tools:context=".DrawerLayoutActivity">
    <!-- 普通布局 -->
    <FrameLayout
        android:id="@+id/fl_content"
        android:background="@android:color/holo_orange_dark"
        android:layout_width="match_parent"
        android:layout_height="match_parent">
        <TextView
            android:text="我是内容布局"
            android:layout_gravity="center"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"/>
    </FrameLayout>
    <!-- 侧滑布局 -->
    <include layout="@layout/layout_drawer_normal"/>
</android.support.v4.widget.DrawerLayout>
```
`layout_drawer_normal`就是侧滑菜单，我们将它的`layout_gravity`定义为`start`，按照前面的分析，它应当位于屏幕的左侧：
```
<LinearLayout
    xmlns:android="http://schemas.android.com/apk/res/android"
    android:orientation="vertical"
    android:layout_gravity="start"
    android:layout_width="200dp"
    android:background="@android:color/holo_green_dark"
    android:layout_height="match_parent">
    <TextView
        android:text="我是侧滑布局"
        android:layout_gravity="center"
        android:textColor="@android:color/white"
        android:gravity="center"
        android:layout_width="match_parent"
        android:layout_height="match_parent"/>
</LinearLayout>
```
下面是最终的效果：
![](http://upload-images.jianshu.io/upload_images/1949836-1144aac08d7c3c39.gif?imageMogr2/auto-orient/strip)
## 2.3 监听`DrawerLayout`的状态变化
如果我们希望监听`DrawerLayout`状态的变化，那么可以通过下面这个方法：
```
    private Toolbar mToolbar;
    private ActionBarDrawerToggle mActionBarDrawerToggle;
    private DrawerLayout mDrawerLayout;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_drawer_layout_simple);
        mDrawerLayout = (DrawerLayout) findViewById(R.id.drawer_layout);
        mDrawerLayout.addDrawerListener(new DrawerLayout.DrawerListener() {

            @Override
            public void onDrawerSlide(View drawerView, float slideOffset) {
                Log.d("mDrawerLayout", "onDrawerSlide, slideOffset=" + slideOffset);
            }

            @Override
            public void onDrawerOpened(View drawerView) {
                Log.d("mDrawerLayout", "onDrawerOpened");
            }

            @Override
            public void onDrawerClosed(View drawerView) {
                Log.d("mDrawerLayout", "onDrawerClosed");
            }

            @Override
            public void onDrawerStateChanged(int newState) {
                Log.d("mDrawerLayout", "onDrawerStateChanged, state=" + newState);

            }
        });

    }
```
其中`opened`和`closed`方法都很好理解，就是对应展开和收起，而`onDrawerSlide`方法中的`slideOffset`，则对应于`DrawerLayout`展开的偏移值，全部展开时为`1`，全部收起时为`0`，`onDrawerStateChanged`对应于所处的状态：`STATE_IDLE/STATE_DRAGGING/STATE_SETTLING`。

# 三、`DrawerLayout`和`Toolbar`结合使用
## 3.1 `Toolbar`的`NavigationIcon`跟随`DrawerLayout`变化
下面，我们看一下把`DrawerLayout`和`Toolbar`相结合，做出下面的效果，让`Toolbar`的`navigationIcon`跟随着`DrawerLayout`变化：
![](http://upload-images.jianshu.io/upload_images/1949836-1d3a0fb9a3b2b230.gif?imageMogr2/auto-orient/strip)
当拖动侧边栏的时候，`Toolbar`的按钮会跟随着进行状态的变化，我们也可以通过点击`Toolbar`的按钮来展开和收起侧边栏，首先看我们的布局
```
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:orientation="vertical"
    android:layout_width="match_parent"
    android:layout_height="match_parent">
    <!-- Toolbar -->
    <android.support.v7.widget.Toolbar
        android:id="@+id/toolbar"
        android:background="@android:color/holo_green_dark"
        android:layout_width="match_parent"
        android:layout_height="?attr/actionBarSize"/>
    <android.support.v4.widget.DrawerLayout
        android:id="@+id/drawer_layout"
        android:layout_width="match_parent"
        android:layout_height="match_parent">
        <!-- 普通布局 -->
        <FrameLayout
            android:id="@+id/fl_content"
            android:background="@android:color/holo_orange_dark"
            android:layout_width="match_parent"
            android:layout_height="match_parent">
            <TextView
                android:text="我是内容布局"
                android:layout_gravity="center"
                android:layout_width="wrap_content"
                android:layout_height="wrap_content"/>
        </FrameLayout>
        <!-- 侧滑布局 -->
        <include layout="@layout/layout_drawer_normal"/>
    </android.support.v4.widget.DrawerLayout>
</LinearLayout>
```
在`Activity`中，我们需要将`Toolbar`和`DrawerLayout`关联起来：
```
public class DrawerLayoutActivity extends AppCompatActivity {

    private Toolbar mToolbar;
    private ActionBarDrawerToggle mActionBarDrawerToggle;
    private DrawerLayout mDrawerLayout;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_drawer_layout_under_toolbar);
        initView();
    }

    private void initView() {
        mToolbar = (Toolbar) findViewById(R.id.toolbar);
        setSupportActionBar(mToolbar);
        getSupportActionBar().setDisplayHomeAsUpEnabled(true); //1.决定显示.
        mDrawerLayout = (DrawerLayout) findViewById(R.id.drawer_layout);
        mActionBarDrawerToggle = new ActionBarDrawerToggle(this, mDrawerLayout, mToolbar, R.string.drawer_open, R.string.drawer_close); //2.传入Toolbar可以点击.
        mDrawerLayout.addDrawerListener(mActionBarDrawerToggle); //3.监听变化.
    }

    @Override
    protected void onPostCreate(@Nullable Bundle savedInstanceState) {
        super.onPostCreate(savedInstanceState);
        //4.同步状态
        mActionBarDrawerToggle.syncState();
    }
}
```
这里需要做的有四步工作：
- 将`Toolbar`作为`ActionBar`，并调用`Toolbar`的`setDisplayHomeAsUpEnabled `，这样最左边的`Icon`才可以显示。
- 实例化`ActionBarDrawerToggle`，并传入`Toolbar`和`DrawerLayout`。
- 通过`DrawerLayout`的`addDrawerListener`方法，让`DrawerLayout`的状态能够回调到`ActionBarDrawerToggle`。
- 在`onPostCreate`中，调用`ActionBarDrawerToggle`的`syncState`方法。

## 3.2 `DrawerLayout`覆盖`Toolbar`
在上面的实现方式中，侧滑菜单位于`Toolbar`的下方，如果我们希望它覆盖`Toolbar`，那么可以像下面这样布局：
```
<android.support.v4.widget.DrawerLayout
    xmlns:android="http://schemas.android.com/apk/res/android"
    android:id="@+id/drawer_layout"
    android:layout_width="match_parent"
    android:layout_height="match_parent">
    <!-- 普通布局 -->
    <LinearLayout
        android:orientation="vertical"
        android:layout_width="match_parent"
        android:layout_height="match_parent">
        <android.support.v7.widget.Toolbar
            android:id="@+id/toolbar"
            android:background="@android:color/holo_green_dark"
            android:layout_width="match_parent"
            android:layout_height="?attr/actionBarSize"/>
        <FrameLayout
            android:id="@+id/fl_content"
            android:background="@android:color/holo_orange_dark"
            android:layout_width="match_parent"
            android:layout_height="match_parent">
            <TextView
                android:text="我是内容布局"
                android:layout_gravity="center"
                android:layout_width="wrap_content"
                android:layout_height="wrap_content"/>
        </FrameLayout>
    </LinearLayout>
    <!-- 侧滑布局 -->
    <include layout="@layout/layout_drawer_normal"/>
</android.support.v4.widget.DrawerLayout>
```
最终的效果为：
![](http://upload-images.jianshu.io/upload_images/1949836-a7ec4e677e155375.gif?imageMogr2/auto-orient/strip)

## 3.3 `DrawerLayout`和`Toolbar`延伸到状态栏上方
如果我们希望侧滑菜单的区域能够延伸到状态栏，那么可以进行以下三步的修改：
- 第一步：修改`Activity`的`style`，让状态栏透明，并修改状态栏的颜色：
```
<resources>
    <style name="AppTheme" parent="Theme.AppCompat.Light.NoActionBar">
        <!-- Customize your theme here. -->
        <item name="colorPrimary">@color/colorPrimary</item>
        <item name="colorAccent">@color/colorAccent</item>
        <!-- 修改状态栏颜色 -->
        <item name="colorPrimaryDark">@android:color/holo_green_dark</item>
        <!-- 状态栏透明 -->
        <item name="android:windowTranslucentStatus">true</item>
    </style>
</resources>
```
- 第二步：给`Activity`根布局设置`android:fitsSystemWindows="true"`属性
```
<android.support.v4.widget.DrawerLayout
    xmlns:android="http://schemas.android.com/apk/res/android"
    android:id="@+id/drawer_layout"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:fitsSystemWindows="true">
    <!-- 普通布局 -->
    <LinearLayout
        android:orientation="vertical"
        android:layout_width="match_parent"
        android:layout_height="match_parent">
        <android.support.v7.widget.Toolbar
            android:id="@+id/toolbar"
            android:background="@android:color/holo_green_dark"
            android:layout_width="match_parent"
            android:layout_height="?attr/actionBarSize"/>
        <FrameLayout
            android:id="@+id/fl_content"
            android:background="@android:color/holo_orange_dark"
            android:layout_width="match_parent"
            android:layout_height="match_parent">
            <TextView
                android:text="我是内容布局"
                android:layout_gravity="center"
                android:layout_width="wrap_content"
                android:layout_height="wrap_content"/>
        </FrameLayout>
    </LinearLayout>
    <!-- 侧滑布局 -->
    <include layout="@layout/layout_drawer_normal"/>
</android.support.v4.widget.DrawerLayout>
```
- 第三步：给侧滑布局设置`android:fitsSystemWindows="true"`属性
```
<LinearLayout
    xmlns:android="http://schemas.android.com/apk/res/android"
    android:orientation="vertical"
    android:layout_gravity="start"
    android:layout_width="200dp"
    android:background="@android:color/holo_green_dark"
    android:layout_height="match_parent"
    android:fitsSystemWindows="true">
    <TextView
        android:text="我是侧滑布局"
        android:layout_gravity="center"
        android:textColor="@android:color/white"
        android:gravity="center"
        android:layout_width="match_parent"
        android:layout_height="match_parent"/>
</LinearLayout>
```

通过以上三步，就可以达到沉浸式状态栏的效果：
![](http://upload-images.jianshu.io/upload_images/1949836-0e663bb42383a968.gif?imageMogr2/auto-orient/strip)

# 四、`NavigationView`
## 4.1 `Navigation`属性
在使用`DrawerLayout`的时候，我们可以随意定义侧滑菜单的布局，`NavigationView`其实就是`Google`推荐的侧滑布局应该有的样子，它的效果类似于下面这样：
![](http://upload-images.jianshu.io/upload_images/1949836-eab173f5f979d168.gif?imageMogr2/auto-orient/strip)
我们先来看一个使用`Navigation`的简单例子：
```
<android.support.design.widget.NavigationView
    xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    android:layout_width="300dp"
    android:layout_height="match_parent"
    android:layout_gravity="start"
    app:headerLayout="@layout/layout_drawer_navigation_header"
    app:menu="@menu/menu_navigation"
    app:itemIconTint="@android:color/white"
    app:itemBackground="@android:color/holo_green_dark"
    app:itemTextColor="@android:color/black">
</android.support.design.widget.NavigationView>
```
上面`app`各属性对应到下图中就是这样：
![](http://upload-images.jianshu.io/upload_images/1949836-ae594221e75338de.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
可以看到，`Navigation`分为两个部分：头部和列表。
- 头部是`app:headerLayout`所指定的布局：
```
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:orientation="vertical"
    android:layout_width="match_parent"
    android:layout_height="wrap_content">
    <ImageView
        android:src="@drawable/ic_bg"
        android:scaleType="centerCrop"
        android:layout_width="match_parent"
        android:layout_height="200dp"/>
</LinearLayout>
```
- 列表通过`app:menu`所指定：
```
<?xml version="1.0" encoding="utf-8"?>
<menu xmlns:android="http://schemas.android.com/apk/res/android">
    <item
        android:id="@+id/favorite"
        android:icon="@mipmap/ic_launcher"
        android:title="收藏"/>
    <item
        android:id="@+id/wallet"
        android:icon="@mipmap/ic_launcher"
        android:title="钱包"/>
    <item
        android:id="@+id/photo"
        android:icon="@mipmap/ic_launcher"
        android:title="相册"/>
    <item
        android:id="@+id/file"
        android:icon="@mipmap/ic_launcher"
        android:title="文件"/>
</menu>
```
`menu`当中的每个`item`就对应于列表当中的一项，而`item`的`icon`和`title`则分别对应列表项的图标和文字，如果希望对`item`进行分组，那么可以采用`group`的方式来组织`menu`：
```
<menu xmlns:android="http://schemas.android.com/apk/res/android">
    <group android:id="@+id/group1">
        <item
            android:id="@+id/favorite"
            android:icon="@mipmap/ic_launcher"
            android:title="group1 - item1"/>
        <item
            android:id="@+id/wallet"
            android:icon="@mipmap/ic_launcher"
            android:title="group1 - item2"/>
    </group>
    <group android:id="@+id/group2">
        <item
            android:id="@+id/photo"
            android:icon="@mipmap/ic_launcher"
            android:title="group2 - item1"/>
        <item
            android:id="@+id/file"
            android:icon="@mipmap/ic_launcher"
            android:title="group2 - item2"/>
    </group>
</menu>
```
不同组之间就会被分割线隔开：
![](http://upload-images.jianshu.io/upload_images/1949836-23fe41c79e745d68.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
如果想要给某个分组添加标题，那么可以采用`subMenu`的方式：
```
<menu xmlns:android="http://schemas.android.com/apk/res/android">
    <group android:id="@+id/group1">
        <item
            android:id="@+id/item1"
            android:icon="@mipmap/ic_launcher"
            android:title="group1 - item1"/>
        <item
            android:id="@+id/item2"
            android:icon="@mipmap/ic_launcher"
            android:title="group1 - item2"/>
    </group>
    <group android:id="@+id/group2">
        <item
            android:id="@+id/item3"
            android:icon="@mipmap/ic_launcher"
            android:title="group2 - item1"/>
        <item
            android:id="@+id/item4"
            android:icon="@mipmap/ic_launcher"
            android:title="group2 - item2"/>
    </group>
    <item
        android:id="@+id/item5"
        android:title="group3">
        <menu>
            <item
                android:id="@+id/item6"
                android:icon="@mipmap/ic_launcher"
                android:title="group3 - item1"/>
            <item
                android:id="@+id/item7"
                android:icon="@mipmap/ic_launcher"
                android:title="group3 - item2"/>
        </menu>
    </item>
</menu>
```
![](http://upload-images.jianshu.io/upload_images/1949836-b5c67dfc10e36bed.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 4.2 `Navigation`点击监听
### 4.2.1 列表点击监听
如果希望处理`Navigation`中列表的监听，那么可以使用现成的接口，根据`item`的`id`来判断是点击的是列表中的哪一项。
```
mNavigationView = (NavigationView) findViewById(R.id.navigation);
mNavigationView.setNavigationItemSelectedListener(new NavigationView.OnNavigationItemSelectedListener() {
    @Override
    public boolean onNavigationItemSelected(@NonNull MenuItem item) {
        Log.d("onSelected", "id=" + item.getItemId());
        return true;
    } 
});
```
### 4.2.2 头部点击监听
`NavigationView`并没有提供头部点击的监听回调，因此，我们只能够通过`findViewById`的方法找到对应的`View`，对其设置监听，下面是整个`headerLayout`在`NavigationView`中的层次：
![](http://upload-images.jianshu.io/upload_images/1949836-6d4963e204420402.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

# 五、参考文献
[Android 5.0 之 NavigationView 的使用](http://blog.csdn.net/u012702547/article/details/51253222)
