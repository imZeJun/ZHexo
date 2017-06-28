---
title: RecyclerView 知识梳理(4) - ItemDecoration
date: 2017-04-04 18:26
categories : RecyclerView 知识梳理
---
# 一、概述
通过`ItemDecoration`，可以给`RecyclerView`或者`RecyclerView`中的每个`Item`添加额外的装饰效果，最常用的就是用来为`Item`之间添加分割线，今天，我们就来一起学习有关的知识：
- `API`
- `DividerItemDecoration`解析
- 自定义`ItemDecoration`

# 二、`API`介绍
当我们实现自己的`ItemDecoration`时，需要继承于`ItemDecoration`，并根据需要实现以下三个方法：
## 2.1 `public void onDraw(Canvas c, RecyclerView parent, State state)`
- `canvas`：`RecyclerView`的`canvas`
- `parent`：`RecyclerView`实例
- `State`：`RecyclerView`当前的状态，值包括`START/LAYOUT/ANIMATION`。

所有在这个方法中的绘制操作，将会在`itemViews`被绘制之前执行，因此，它会显示在`itemView`之下。
## 2.2 `public void onDrawOver(Canvas c, RecyclerView parent, State state)`
和`2.1`方法类似，区别在于它绘制在`itemViews`之上。
## 2.3 `public void getItemOffsets(Rect outRect, View view, RecyclerView parent, State state)`
通过`outRect`，可以设置`item`之间的间隔，间隔区域的大小就是`outRect`所指定的范围，`view`就是对应位置的`itemView`，其它的参数解释和上面相同。
# 三、`DividerItemDecoration`解析
## 3.1 使用方法
上面我们解释了需要重写的方法以及其中参数的含义，下面，我们通过官方自带的`DividerItemDecoration`，来进一步加深对这些方法的认识。
`DividerItemDecoration`是为`LinearLayoutManager`提供的分割线，在创建它的时候，需要指定`ORIENTATION`，这个方向应当和`LinearLayoutManager`的方向相同。
```
    private void init() {
        RecyclerView recyclerView = (RecyclerView) findViewById(R.id.rv_content);
        mTitles = new ArrayList<>();
        for (int i = 0; i < 20; i++) {
            mTitles.add(String.valueOf(i));
        }
        BaseAdapter baseAdapter = new BaseAdapter(mTitles);
        recyclerView.setLayoutManager(new LinearLayoutManager(this));
        recyclerView.addItemDecoration(new DividerItemDecoration(this, DividerItemDecoration.VERTICAL));
        recyclerView.setAdapter(baseAdapter);
    }
```
最终展示的效果为：
![](http://upload-images.jianshu.io/upload_images/1949836-177632791e101088.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
## 3.2 源码解析
### 3.2.1 绘制
`DividerItemDecoration`重写了基类当中的`onDraw`方法，也就是说这个分割线是在`itemView`之前绘制的：
```
    @Override
    public void onDraw(Canvas c, RecyclerView parent, RecyclerView.State state) {
        if (parent.getLayoutManager() == null) {
            return;
        }
        if (mOrientation == VERTICAL) {
            drawVertical(c, parent);
        } else {
            drawHorizontal(c, parent);
        }
    }
```
我们先看纵向排列的`RecyclerView`分割线：
```
    @SuppressLint("NewApi")
    private void drawVertical(Canvas canvas, RecyclerView parent) {
        //首先保存画布
        canvas.save();
        final int left;
        final int right;
        //确定左右边界的范围，如果RecyclerView不允许子View绘制在Padding内，那么这个范围为去掉Padding后的范围
        if (parent.getClipToPadding()) {
            left = parent.getPaddingLeft();
            right = parent.getWidth() - parent.getPaddingRight();
            canvas.clipRect(left, parent.getPaddingTop(), right,
                    parent.getHeight() - parent.getPaddingBottom());
        } else {
            left = 0;
            right = parent.getWidth();
        }

        final int childCount = parent.getChildCount();
        for (int i = 0; i < childCount; i++) {
            final View child = parent.getChildAt(i);
            //获得itemView的范围，这个范围包括了margin和offset，它们被保存在mBounds当中
            parent.getDecoratedBoundsWithMargins(child, mBounds);
            //需要考虑translationY和translationY
            final int bottom = mBounds.bottom + Math.round(ViewCompat.getTranslationY(child));
            //由于是垂直排列的，因此上边界等于下边界减去分割线的高度.
            final int top = bottom - mDivider.getIntrinsicHeight();
            //设置divider和范围
            mDivider.setBounds(left, top, right, bottom);
            //绘制.
            mDivider.draw(canvas);
        }
        //回复画布.
        canvas.restore();
    }
```
整个过程分为三步：
- 确定子`View`在`RecyclerView`中的绘制范围
- 确定每个子`View`的范围
- 确定`mDivider`的绘制范围

下图就是最终计算的结果：
![](http://upload-images.jianshu.io/upload_images/1949836-cc11c628c5fb54d5.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
横向排列的`RecyclerView`列表和上面的原理是相同的，区别就在于计算`mDivider.setBounds`的计算：
```
//....
parent.getLayoutManager().getDecoratedBoundsWithMargins(child, mBounds);
final int right = mBounds.right + Math.round(ViewCompat.getTranslationX(child));
final int left = right - mDivider.getIntrinsicWidth();
mDivider.setBounds(left, top, right, bottom);
//..
```
### 3.2.2 边界处理
从上面的分析可以知道，如果将`divider`直接绘制在`itemView`的范围内，那么由于我们是先绘制`divider`，再绘制`itemView`的内容的，那么它就会被覆盖，因此，通过重写`getItemOffsets`，通过其中的`outRect`来指定留出的空隙：
```
    @Override
    public void getItemOffsets(Rect outRect, View view, RecyclerView parent,
            RecyclerView.State state) {
        if (mOrientation == VERTICAL) {
            //如果是纵向排列，那么要在itemView的下方留出一个下边界
            outRect.set(0, 0, 0, mDivider.getIntrinsicHeight());
        } else {
            //如果是横向排列，那么要在itemView的右方留出一个右边界
            outRect.set(0, 0, mDivider.getIntrinsicWidth(), 0);
        }
    }
```
# 四、自定义`ItemDecoration`
下面，我们参考上面的写法，写一个简单的`GridLayoutManager`的分割线：
```
public class GridDividerItemDecoration extends RecyclerView.ItemDecoration {

    private static final int[] ATTRS = new int[] { android.R.attr.listDivider };
    private Drawable mDivider;
    private final Rect mBounds = new Rect();

    public GridDividerItemDecoration(Context context) {
        final TypedArray a = context.obtainStyledAttributes(ATTRS);
        mDivider = a.getDrawable(0);
        a.recycle();
    }

    public void setDrawable(@NonNull Drawable drawable) {
        mDivider = drawable;
    }

    @Override
    public void onDraw(Canvas c, RecyclerView parent, RecyclerView.State state) {
        drawDivider(c, parent);
    }

    @Override
    public void getItemOffsets(Rect outRect, View view, RecyclerView parent, RecyclerView.State state) {
        outRect.set(0, 0, mDivider.getIntrinsicWidth(), mDivider.getIntrinsicHeight());
    }

    private void drawDivider(Canvas canvas, RecyclerView parent) {
        canvas.save();
        final int childCount = parent.getChildCount();
        for (int i = 0; i < childCount; i++) {
            final View view = parent.getChildAt(i);
            parent.getDecoratedBoundsWithMargins(view, mBounds);
            mDivider.setBounds(mBounds.right - mDivider.getIntrinsicWidth(), mBounds.top, mBounds.right, mBounds.bottom);
            mDivider.draw(canvas);
            mDivider.setBounds(mBounds.left, mBounds.bottom - mDivider.getIntrinsicHeight(), mBounds.right , mBounds.bottom);
            mDivider.draw(canvas);
        }
        canvas.restore();
   }
}
```
最终的效果为：
![](http://upload-images.jianshu.io/upload_images/1949836-2931df927aef7190.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
这里我们为了演示方便，没有考虑最后一列或者最后一行没有分割线的情况，这篇文章写的比较好：[Android RecyclerView 使用完全解析 体验艺术般的控件](http://blog.csdn.net/lmj623565791/article/details/45059587)。
# 五、总结
`ItemDecoration`的使用并不难，大多数情况下就只需要重写`onDraw`和`onDrawOver`中的一个；如果需要在`Item`之间添加间隔，那么要重写`getItemOffsets`并理解`outRect`的含义，假如不需要添加间隔，那么不需要重写该方法。
