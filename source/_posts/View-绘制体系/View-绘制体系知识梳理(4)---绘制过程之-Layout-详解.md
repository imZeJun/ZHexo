---
title: 绘制过程 - Layout
date: 2017-02-25 14:10
categories : View 绘制体系知识梳理
---
# 一、布局的起点 - `performTraversals`
和前面分析测量过程类似，整个布局的起点也是在`ViewRootImpl`的`performTraversals`当中：
```
private void performTraversals() {
    ......
    mView.layout(0, 0, mView.getMeasuredWidth(), mView.getMeasuredHeight());
    ......
}
```
可以看到，布局过程会**参考**前面一步测量的结果，和测量过程的`measure`和`onMeasure`方法很像，布局过程也有两个方法`layout`和`onLayout`，参考前面的分析，我们先对这两个方法进行介绍：
## 1.1 `layout`和`onLayout`
- 对于`View`
 - `layout`方法是`public`的，和`measure`不同的是，它不是`final`的，也就是说，继承于`View`的控件可以重写`layout`方法，但是我们一般不这么做，因为在它的`layout`方法中又调用了`onLayout`，所以继承于`View`的控件一般是通过重写`onLayout`来实现一些逻辑。
   ```
    public void layout(int l, int t, int r, int b) {
        //....
        boolean changed = isLayoutModeOptical(mParent) ?
                setOpticalFrame(l, t, r, b) : setFrame(l, t, r, b);
        if (changed || (mPrivateFlags & PFLAG_LAYOUT_REQUIRED) == PFLAG_LAYOUT_REQUIRED) {
            onLayout(changed, l, t, r, b);
        }
        //...
    }
   ```
 - `onLayout`方法是一个空实现。
```
    protected void onLayout(boolean changed, int left, int top, int right, int bottom) {}
```
- 对于`ViewGroup`
 - 它重写了`View`中的`layout`，并把它设为`final`，也就是说**继承于`ViewGroup`的控件**，不能重写`layout`，在`layout`方法中，又会调用`super.layout`，也就是`View`的`layout`。
    ```
    @Override
    public final void layout(int l, int t, int r, int b) {
        if (!mSuppressLayout && (mTransition == null || !mTransition.isChangingLayout())) {
            if (mTransition != null) {
                mTransition.layoutChange(this);
            }
            super.layout(l, t, r, b);
        } else {
            // record the fact that we noop'd it; request layout when transition finishes
            mLayoutCalledWhileSuppressed = true;
        }
    }
    ```
 - `onLayout`方法重写`View`中的`onLayout`方法，并把它声明成了`abstract`，也就是说，**所有继承于`ViewGroup`的控件，都必须实现`onLayout`方法**。
```
    @Override
    protected abstract void onLayout(boolean changed, int l, int t, int r, int b);
```
- 对于继承于`View`的控件
 例如`TextView`，我们一般不会重写`layout`，而是在`onLayout`中进行简单的处理。
- 对于继承于`ViewGroup`的控件
 - 例如`LinearLayout`，由于`ViewGroup`的作用是为了包裹子`View`，而每个控件由于作用不同，布局的方法自然也不同，这也是为了安卓要求每个继承于`ViewGroup`的控件都必须实现`onLayout`方法的原因。
 - 因为`ViewGroup`的`layout`方法不可以重写，因此，当我们通过父容器调用一个继承于`ViewGroup`的控件的`layout`方法时，它最终会回调到该控件的`onLayout`方法。

## 1.2  布局`onLayout(boolean changed, int l, int t, int r, int b)`参数说明
当`onLayout`方法被回调时，传入了上面这四个参数，经过前面的分析，我们知道`onLayout`是通过`layout`方法调用过来，而`layout`方法父容器调用的，父容器在调用的时候是根据自己的坐标来计算出宽高，并把自己的位置的左上角当作是`(0,0)`点，重新决定它所属子`View`的坐标，因此**这个矩形的四个坐标是相对于父容器的坐标值**。
## 1.3 布局的遍历过程
虽然`layout`在某些方面和`measure`有所不同，但是它们有一点是共通的，那就是：**它们都是作为整个从根节点到叶节点传递的纽带，当从父容器到子`View`传递的过程中，我们不直接调用`onLayout`，而是调用`layout`**。
`onMeasure`在测量过程中负责两件事：它自己的测量和它的子`View`的测量，而`onLayout`不同：它并不负责自己的布局，这是由它的父容器决定的，它仅仅负责自己的下一级子`View`的布局。
再回到文章最开始的点，起点是通过`mView`也就是`DecorView`的`layout`方法触发的，而`DecorView`实际上是一个`FrameLayout`，经过前面的分析，我们应该直接去看`FrameLayout`的`onLayout`方法：
```
    @Override
    protected void onLayout(boolean changed, int left, int top, int right, int bottom) {
        layoutChildren(left, top, right, bottom, false /* no force left gravity */);
    }

    void layoutChildren(int left, int top, int right, int bottom, boolean forceLeftGravity) {
        final int count = getChildCount();
        for (int i = 0; i < count; i++) {
            final View child = getChildAt(i);
                child.layout(childLeft, childTop, childLeft + width, childTop + height);
            }
        }
    }
```
可以看到，在它的`onLayout`当中，又调用了它的子`View`的`layout`，那么这时候就分为两种情况，一种是该`child`是继承于`ViewGroup`的控件并且它有子节点，那么`child.layout`方法最终又会调用到`child.onLayout`，在里面，它同样会进行和`FrameLayout`所类似的操作，继续调用`child`的子节点的`layout`；另一种是该`child`是`View`或者是继承于`View`的控件或者是它是继承于`ViewGroup`的控件但是没有子节点，那么到该`child`节点的布局遍历过程就结束了。
## 1.4 小结
通过分析测量和布局的过程，它们基于一个思想，把**传递**和**实现**这两个逻辑分开在不同的函数中处理，在**实现**当中，再去决定是否要**传递**。
