---
title: Material Design 控件知识梳理(8) - TabLayout
date: 2017-04-15 23:56
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
前面我们介绍了`BottomNavigationBar`，`TabLayout`和它的作用类似，都是方便用户进行页面之间的切换。
# 二、`TabLayout`的基本用法
下面，我们先演示一遍`TabLayout`的基本用法
第一步：引入依赖：
```
compile 'com.android.support:design:25.3.1'
```
第二步：定义布局，这里使用了最常用的`TabLayout+ViewPager`的结构：
```
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    android:orientation="vertical"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    tools:context="com.demo.lizejun.repotransition.TabLayoutActivity">
    <android.support.design.widget.TabLayout
        android:id="@+id/tab_layout"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        app:tabMode="scrollable"
        app:tabGravity="center"/>
    <android.support.v4.view.ViewPager
        android:id="@+id/vp_content"
        android:layout_width="match_parent"
        android:layout_height="match_parent"/>
</LinearLayout>
```
第三步：定义`ViewPager`的`Adapter`，这里需要重写`getPageTitle`方法，它就对应的是`TabLayout`当中的文案：
```
public class ViewPagerAdapter extends PagerAdapter {

    private List<String> mTitles;

    public ViewPagerAdapter(List<String> title) {
        mTitles = title;
    }

    @Override
    public Object instantiateItem(ViewGroup container, int position) {
        ViewGroup itemView = (ViewGroup) LayoutInflater.from(container.getContext()).inflate(R.layout.item_view_pager, container, false);
        container.addView(itemView);
        TextView tv = (TextView) itemView.findViewById(R.id.tv_title);
        tv.setText(mTitles.get(position));
        return itemView;
    }

    @Override
    public void destroyItem(ViewGroup container, int position, Object object) {
        container.removeView((View) object);
    }

    @Override
    public boolean isViewFromObject(View view, Object object) {
        return view == object;
    }

    @Override
    public int getCount() {
        return mTitles.size();
    }

    @Override
    public CharSequence getPageTitle(int position) {
        return mTitles.get(position);
    }
}
```
第四步：给`ViewPager`设置数据，并把`ViewPager`和`TabLayout`关联起来：
```
public class TabLayoutActivity extends AppCompatActivity {

    private static final String[] TITLE_SHORT = new String[] {
            "深圳","南京","内蒙古"
    };

    private static final String[] TITLE_LONG = new String[] {
            "深圳","南京","内蒙古呼和浩特","广西壮族自治区","上海","北京","天津"
    };

    private ViewPager mViewPager;
    private TabLayout mTabLayout;
    private ViewPagerAdapter mViewPagerAdapter;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_tab_layout);
        initViewPager();
        initTabLayout();
    }

    private void initViewPager() {
        mViewPager = (ViewPager) findViewById(R.id.vp_content);
        List<String> titles = new ArrayList<>();
        Collections.addAll(titles, TITLE_LONG);
        mViewPagerAdapter = new ViewPagerAdapter(titles);
        mViewPager.setAdapter(mViewPagerAdapter);
    }

    private void initTabLayout() {
        mTabLayout = (TabLayout) findViewById(R.id.tab_layout);
        mTabLayout.setupWithViewPager(mViewPager);
    }

}
```
最终的效果为：
![](http://upload-images.jianshu.io/upload_images/1949836-61e13ee078370c35.gif?imageMogr2/auto-orient/strip)
# 三、`TabLayout`相关设置
## 3.1 `TabItem`的外观设置
对于每个`TabItem`的外观，`TabLayout`我们比较常用的属性有：
- 未选中`TabItem`的文本颜色：`app:tabTextColor`
- 选中`TabItem`的文本颜色`app:tabSelectedTextColor`
- 底部滑动条的颜色：`app:tabIndicatorColor`
- 底部滑动条的高度：`app:tabIndicatorHeight`
- `TabItem`的背景：`app:tabBackground`

我们在第二节的基础上进行修改：
```
<android.support.design.widget.TabLayout
        android:id="@+id/tab_layout"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        app:tabMode="scrollable"
        app:tabGravity="center"
        app:tabTextColor="@android:color/darker_gray"
        app:tabSelectedTextColor="@android:color/holo_orange_dark"
        app:tabIndicatorColor="@android:color/holo_orange_dark"
        app:tabIndicatorHeight="2dp"
        app:tabBackground="@android:color/transparent" />
```
最终展现的效果为：
![](http://upload-images.jianshu.io/upload_images/1949836-d8fc4d61d7f37ec3.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
## 3.2 `TabItem`的宽高相关设置
有两个关于宽度的属性比较重要：`tabMinWidth`和`tabMaxWidth`，它允许我们这是每个`TabItem`的最小和最大宽度。
此外，我们还可以设置`TabItem`四个维度的`Padding`。
## 3.3 `tabMode`和`tabGravity`
上面只是决定了单个`TabItem`的外观，而`TabItem`在`TabLayout`当中的摆放规则需要由这两个属性来决定，它们可选的值分别为：
- `tabMode`：`fixed`或者`scrollable`
- `tabGravity`：`fill`或者`center`

这两个属性分别通过：`app:tabMode`和`app:tabGravity`来设置，下面，我们就来解释这几个值的含义。
### 3.3.1 `tabMode=fixed`
在这种模式下，每个`TabItem`的宽度都是相等的，并且所有的`TabItem`加起来的宽度不会大于`TabLayout`的宽度，最终展现的结果还需要依赖于`tabGravity`属性：
- `tabGravity=fill`
要求所有`TabItem`加起来的宽度等于`TabLayout`的宽度，也就是下面的效果
![](http://upload-images.jianshu.io/upload_images/1949836-9fec62cb873e2b3b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
- `tabGravity=center`
不要求`TabItem`加起来的宽度等于`TabLayout`的宽度，会根据每个`TabItem`的属性，计算出`TabItem`的宽度，然后取它们中的最大值乘上`TabItem`的个数，当这一数值小于`TabLayout`的宽度时，会将每个`TabItem`的宽度设置为这个最大值，并将它们放置在整个`TabLayout`的中间，而如果这一数值大于`TabLayout`的宽度，那么会将它们在`TabLayout`中均匀排列。
![](http://upload-images.jianshu.io/upload_images/1949836-b408c5f776c74f4f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 3.3.2 `tabMode=scrollable`
在`tabMode=fixed`模式下，`TabItem`的位置是固定的，并且它们加起来的宽度不会大于`TabLayout`的宽度，而在`scrollable`模式下则没有这一限制，只需要计算好每个`TabItem`的宽度，然后把它们排列在`TabLayout`中就好了，可以通过拖动的方式查看更多的`TabItem`，并且当`tabMode=scrollable`时，`tabGravity`其实并没有作用，下面是`tabMode=scrollable`的事例：
![](http://upload-images.jianshu.io/upload_images/1949836-eda4663839da944c.gif?imageMogr2/auto-orient/strip)
在这种模式下，每个`TabItem`的宽度默认情况下是根据文字的宽度计算出来的，如果我们希望进行相应的限制，那么可以修改`tabMinWidth`或者`tabMaxWidth`属性。
## 3.4 设置监听
```
mTabLayout.addOnTabSelectedListener(new TabLayout.OnTabSelectedListener() {

            @Override
            public void onTabSelected(TabLayout.Tab tab) {
                Log.d("onTabSelected", "position=" + tab.getPosition());
            }

            @Override
            public void onTabUnselected(TabLayout.Tab tab) {
                Log.d("onTabUnselected", "position=" + tab.getPosition());

            }

            @Override
            public void onTabReselected(TabLayout.Tab tab) {
                Log.d("onTabReselected", "position=" + tab.getPosition());

            }
});
```
- `onTabSelected`，某个`TabItem`从未选中状态变为选中状态时回调
- `onTabUnselected`，某个`TabItem`从选中变为未选中时回调
- `onTabReselected`，某个`TabItem`已经处于选中状态，但是它又被再次点击了，那么回调这个函数。

# 四、`TabLayout + ViewPager`
使用`TabLayout`时，一般都是采用`ViewPager`来管理多个子界面，而`TabLayout`也十分贴心地提供了方法让它和`ViewPager`的滚动关联起来，当点击`TabLayout`中的`Item`时，会切换到`ViewPager`对应的界面，而如果滑动了`ViewPager`，那么`TabLayout`也会切换到对应的`TabItem`。
我们所需要的只是下面这句设置：
```
mTabLayout.setupWithViewPager(mViewPager);
```
为了能让`TabLayout`知道`TabItem`的标题，需要重写`ViewPager`所对应的`PagerAdapter`中`getPageTitle`的方法：
```
@Override
public CharSequence getPageTitle(int position) {
    return mTitles.get(position);
}
```
虽然`TabLayout`并没有提供切换到某个具体位置`TabItem`的方法，但是其实这一过程这完全可以通过`mViewPager.setCurrentItem(int item)`来实现。
