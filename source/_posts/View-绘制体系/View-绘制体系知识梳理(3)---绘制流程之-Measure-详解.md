---
title: View 绘制体系知识梳理(3) - 绘制流程之 Measure 详解
date: 2017-02-25 12:53
categories : View 绘制体系知识梳理
---
# 一、测量过程的信使 - MeasureSpec
因为测量是一个从上到下的过程，而在这个过程当中，父容器有必要告诉子`View`它的一些绘制要求，那么这时候就需要依赖一个信使，来传递这个要求，它就是`MeasureSpec`.
`MeasureSpec`是一个`32`位的`int`类型，我们把它分为高`2`位和低`30`位。
其中高`2`位表示`mode`，它的取值为：
- `UNSPECIFIED(0) : The parent has not imposed any constraint on the child. It can be whatever size it wants.`
- `EXACTLY(1) : The parent has determined an exact size for the child. The child is going to be given those bounds regardless of how big it wants to be.`
- `AT_MOST(2) : The child can be as large as it wants up to the specified size.`

低`30`位表示具体的`size`。
> `MeasureSpec`是父容器**传递给**子`View`的宽高要求，并不是说它传递的`size`是多大，子`View`最终就是多大，它是根据**父容器的`MeasureSpec`**和子**`View`的`LayoutParams`**共同计算出来的。

为了更好的理解上面这段话，我们需要借助`ViewGroup`中的两个函数：
- `measureChildWithMargins(View child, int parentWidthMeasureSpec, int widthUsed, int parentHeightMeasureSpec, int heightUsed)`
- `getChildMeasureSpec(int spec, int padding, int childDimension)`

```
    protected void measureChildWithMargins(View child,
            int parentWidthMeasureSpec, int widthUsed,
            int parentHeightMeasureSpec, int heightUsed) {
        final MarginLayoutParams lp = (MarginLayoutParams) child.getLayoutParams();
        final int childWidthMeasureSpec = getChildMeasureSpec(parentWidthMeasureSpec,
                mPaddingLeft + mPaddingRight + lp.leftMargin + lp.rightMargin
                        + widthUsed, lp.width);
        final int childHeightMeasureSpec = getChildMeasureSpec(parentHeightMeasureSpec,
                mPaddingTop + mPaddingBottom + lp.topMargin + lp.bottomMargin
                        + heightUsed, lp.height);
        child.measure(childWidthMeasureSpec, childHeightMeasureSpec);
    }

    public static int getChildMeasureSpec(int spec, int padding, int childDimension) {
        int specMode = MeasureSpec.getMode(spec);
        int specSize = MeasureSpec.getSize(spec);
        int size = Math.max(0, specSize - padding);
        int resultSize = 0;
        int resultMode = 0;
        switch (specMode) {
        case MeasureSpec.EXACTLY:
            if (childDimension >= 0) {
                resultSize = childDimension;
                resultMode = MeasureSpec.EXACTLY;
            } else if (childDimension == LayoutParams.MATCH_PARENT) {
                resultSize = size;
                resultMode = MeasureSpec.EXACTLY;
            } else if (childDimension == LayoutParams.WRAP_CONTENT) {
                resultSize = size;
                resultMode = MeasureSpec.AT_MOST;
            }
            break;
        case MeasureSpec.AT_MOST:
            if (childDimension >= 0) {
                resultSize = childDimension;
                resultMode = MeasureSpec.EXACTLY;
            } else if (childDimension == LayoutParams.MATCH_PARENT) {
                resultSize = size;
                resultMode = MeasureSpec.AT_MOST;
            } else if (childDimension == LayoutParams.WRAP_CONTENT) {
                resultSize = size;
                resultMode = MeasureSpec.AT_MOST;
            }
            break;
        case MeasureSpec.UNSPECIFIED:
            if (childDimension >= 0) {
                resultSize = childDimension;
                resultMode = MeasureSpec.EXACTLY;
            } else if (childDimension == LayoutParams.MATCH_PARENT) {
                resultSize = View.sUseZeroUnspecifiedMeasureSpec ? 0 : size;
                resultMode = MeasureSpec.UNSPECIFIED;
            } else if (childDimension == LayoutParams.WRAP_CONTENT) {
                resultSize = View.sUseZeroUnspecifiedMeasureSpec ? 0 : size;
                resultMode = MeasureSpec.UNSPECIFIED;
            }
            break;
        }
        return MeasureSpec.makeMeasureSpec(resultSize, resultMode);
    }
```
可以看到，在调用`getChildMeasureSpec`之前，需要考虑`parent`和`child`之间的间距，这包括`parent`的`padding`和`child`的`margin`，因此，参与传递给`child`的`MeasureSpec`的参数要考虑这么几方面：
- 父容器的`measureSpec`和`padding`
- 子`View`的`height`和`widht`以及`margin`。

下面我们来分析`getChildMeasureSpec`的具体流程，它对宽高的处理逻辑都是相同的，根据父容器`measureSpec`的`mode`，分成以下几种情况：
## 1.1 父容器的`mode`为`EXACTLY`
这种情况下说明父容器的大小已经确定了，就是固定的值。
- 子`View`指定了大小
那么子`View`的`mode`就是`EXACTLY`，`size`就是布局里面的值，这里就有疑问了，**子`View`所指定的宽高大于父容器的宽高怎么办呢？**，我们先留着这个疑问。
- 子`View`为`MATCH_PARENT`
子`View`希望和父容器一样大，因为父容器的大小是确定的，所以子`View`的大小也是确定的，`size`就是父容器`measureSpec`的`size` - 父容器的`padding` - 子`View``margin`。
- 子`View`为`WRAP_CONTENT`
子容器只要求能够包裹自己的内容，但是这时候它又不知道它所包裹的内容到底是多大，那么这时候它就指定自己的大小就不能超过父容器的大小，所以`mode`为`AT_MOST`，`size`和上面类似。

## 1.2 父容器的`mode`为`AT_MOST`
在这种情况下，父容器说明了自己最多不能超过多大，数值在`measureSpec`的`size`当中：
- 子`View`指定大小
同上分析。
- 子`View`为`MATCH_PARENT`
子`View`希望和父容器一样大，而此时父容器只知道自己不能超过多大，因此子`View`也就只能知道自己不能超过多大，所以它的`mode`为`AT_MOST`，`size`就是父容器`measureSpec`的`size` - 父容器的`padding` - 子`View``margin`。
- 子`View`为`WRAP_CONTENT`
子容器只要求能够包裹自己的内容，但是这时候它又不知道它所包裹的内容到底是多大，这时候虽然父容器没有指定大小，但是它指定了最多不能超过多少，这时候子`View`也不能超过这个值，所以`mode`为`AT_MOST`，`size`的计算和上面类似。

## 1.3 父容器的`mode`为`UNSPECIFIED`
- 子`View`指定大小
同上分析。
- 子`View`为`MATCH_PARENT`
子`View`希望和父容器一样大，但是这时候父容器并没有约束，所以子`View`也是没有约束的，所以它的`mode`也为`UNSPECIFIED`，`size`的计算和之前一致。
- 子`View`为`WRAP_CONTENT`
子`View`不知道它包裹的内容多大，并且父容器是没有约束的，那么也只能为`UNSPECIFIED`了，`size`的计算和之前一致。

# 二、测量过程的起点 - `performTraversals()`
介绍完了基础的知识，我们来从起点来整个看一下从`View`树的根节点到叶节点的整个测量的过程。
我们先直接说明结论，**整个测量的起点是在`ViewRootImpl`的`performTraversals() `当中**：
```
private void performTraversals() {
        ......
        int childWidthMeasureSpec = getRootMeasureSpec(mWidth, lp.width);
        int childHeightMeasureSpec = getRootMeasureSpec(mHeight, lp.height);
        //...
        mView.measure(childWidthMeasureSpec, childHeightMeasureSpec);
}
```
上面的`mView`是通过`setView(View view, WindowManager.LayoutParams attrs, View panelParentView)`传进来的，那么这个`view`是什么时候传递进来的呢？
现在回忆一下，在`ActivityThread`的`handleResumeActivity`中，我们调用了`ViewManager.add(mDecorView, xxx)`，而这个方法最终会调用到`WindowManagerGlobal`的下面这个方法：
```
    public void addView(View view, ViewGroup.LayoutParams params, Display display, Window parentWindow) {
            root = new ViewRootImpl(view.getContext(), display);
        }
        //....
        root.setView(view, wparams, panelParentView);
    }
```
也就是说，上面的**`mView`也就是我们在`setContentView`当中渲染出来的`mDecorView`**，也就是说它是整个`View`树的根节点，因为`mDecorView`是一个`FrameLayout`，所以它调用的是`FrameLayout`的`measure`方法。
那么这整个从根节点遍历完整个`View`树的过程是怎么实现的呢？
它其实就是依赖于`measure`和`onMeasure`：
- 对于`View`，`measure`是在它里面定义的，而且它是一个`final`方法，因此它的所有子类都没有办法重写该方法，在该方法当中，会调用`onMeasure`来设置最终测量的结果，对于`View`来说，它只是简单的取出父容器传进来的要求来设置，并没有复杂的逻辑。
   ```
public final void measure(int widthMeasureSpec, int heightMeasureSpec) {
        boolean optical = isLayoutModeOptical(this);
        if (optical != isLayoutModeOptical(mParent)) {
            Insets insets = getOpticalInsets();
            int oWidth  = insets.left + insets.right;
            int oHeight = insets.top  + insets.bottom;
            widthMeasureSpec  = MeasureSpec.adjust(widthMeasureSpec,  optical ? -oWidth  : oWidth);
            heightMeasureSpec = MeasureSpec.adjust(heightMeasureSpec, optical ? -oHeight : oHeight);
        }

        // Suppress sign extension for the low bytes
        long key = (long) widthMeasureSpec << 32 | (long) heightMeasureSpec & 0xffffffffL;
        if (mMeasureCache == null) mMeasureCache = new LongSparseLongArray(2);

        if ((mPrivateFlags & PFLAG_FORCE_LAYOUT) == PFLAG_FORCE_LAYOUT ||
                widthMeasureSpec != mOldWidthMeasureSpec ||
                heightMeasureSpec != mOldHeightMeasureSpec) {

            // first clears the measured dimension flag
            mPrivateFlags &= ~PFLAG_MEASURED_DIMENSION_SET;

            resolveRtlPropertiesIfNeeded();

            int cacheIndex = (mPrivateFlags & PFLAG_FORCE_LAYOUT) == PFLAG_FORCE_LAYOUT ? -1 :
                    mMeasureCache.indexOfKey(key);
            if (cacheIndex < 0 || sIgnoreMeasureCache) {
                // measure ourselves, this should set the measured dimension flag back
                onMeasure(widthMeasureSpec, heightMeasureSpec);
                mPrivateFlags3 &= ~PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT;
            } else {
                long value = mMeasureCache.valueAt(cacheIndex);
                // Casting a long to int drops the high 32 bits, no mask needed
                setMeasuredDimensionRaw((int) (value >> 32), (int) value);
                mPrivateFlags3 |= PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT;
            }

            // flag not set, setMeasuredDimension() was not invoked, we raise
            // an exception to warn the developer
            if ((mPrivateFlags & PFLAG_MEASURED_DIMENSION_SET) != PFLAG_MEASURED_DIMENSION_SET) {
                throw new IllegalStateException("View with id " + getId() + ": "
                        + getClass().getName() + "#onMeasure() did not set the"
                        + " measured dimension by calling"
                        + " setMeasuredDimension()");
            }

            mPrivateFlags |= PFLAG_LAYOUT_REQUIRED;
        }

        mOldWidthMeasureSpec = widthMeasureSpec;
        mOldHeightMeasureSpec = heightMeasureSpec;

        mMeasureCache.put(key, ((long) mMeasuredWidth) << 32 |
                (long) mMeasuredHeight & 0xffffffffL); // suppress sign extension
    }
    ```
- 对于`ViewGroup`，由于它是`View`的子类，因此它不可能重写`measure`方法，并且它也没有重写`onMeasure`方法。
- 对于继承于`View`的控件，例如`TextView`，它会重写`onMeasure`，与`View#onMeasure`不同的是，它会考虑更多的情况来决定最终的测量结果。
- 对于继承于`ViewGroup`的控件，例如`FrameLayout`，它同样会重写`onMeasure`方法，与继承于`View`的控件不同的是，由于`ViewGroup`可能会有子`View`，因此它在设置自己最终的测量结果之前，还有一个重要的任务：**调用子`View`的`measure`方法，来对子`View`进行测量，并根据子`View`的结果来决定自己的大小**。

因此，整个从上到下的测量，其实就是一个`View`树节点的遍历过程，每个节点的`onMeasure`返回时，就标志它的测量结束了，而这整个的过程是以`View`中`measure`方法为纽带的：
- 整个过程的起点是`mDecorView`这个根节点的`measure`方法，也就是`performTraversals`中的那句话。
- 如果节点有子节点，也就是说它是继承于`ViewGroup`的控件，那么**在它的`onMeasure`方法中，它并不会直接调用子节点的`onMeasure`方法**，而是通过调用子节点`measure`方法，由于子节点不可能重写`View#measure`方法，因此它最终是**通过`View#measure`来调用子节点重写的`onMeasure`来进行测量**，子节点再在其中进行响应的逻辑处理。
- 如果节点没有子节点，那么当它的`onMeausre`方法被调用时，它需要设置好自己的测量结果就行了。

对于`measure`和`onMeasure`的区别，我们可以用一句简单的话来总结一下：**`measure`负责进行测量的传递，`onMeasure`负责测量的具体实现**。

# 三、测量过程的终点 - `onMeasure`当中的`setMeasuredDimension`
上面我们讲到设置的测量结果，其实测量过程的最终目的是：**通过调用`setMeasuredDimension`方法来给`mMeasureHeight`和`mMeasureWidth`赋值**。
只要上面这个过程完成了，那么该`ViewGroup/View/及其实现类`的测量也就结束了，而**`setMeasuredDimension`必须在`onMeasure`当中调用，否则会抛出异常**，所以我们观察所有继承于`ViewGroup/View`的控件，都会发现它们最后都是调用上面说的那个方法。
前面我们已经分析过，`measure`只是传递的纽带，因此它的逻辑是固定的，我们直接看各个类的`onMeasure`方法就好。
## 3.1 `View`的`onMeasure`
```
    protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
        setMeasuredDimension(getDefaultSize(getSuggestedMinimumWidth(), widthMeasureSpec),
                getDefaultSize(getSuggestedMinimumHeight(), heightMeasureSpec));
    }

    public static int getDefaultSize(int size, int measureSpec) {
        int result = size;
        int specMode = MeasureSpec.getMode(measureSpec);
        int specSize = MeasureSpec.getSize(measureSpec);

        switch (specMode) {
        case MeasureSpec.UNSPECIFIED:
            result = size;
            break;
        case MeasureSpec.AT_MOST:
        case MeasureSpec.EXACTLY:
            result = specSize;
            break;
        }
        return result;
    }

    protected int getSuggestedMinimumHeight() {
        return (mBackground == null) ? mMinHeight : max(mMinHeight, mBackground.getMinimumHeight());
    }

    protected int getSuggestedMinimumWidth() {
        return (mBackground == null) ? mMinWidth : max(mMinWidth, mBackground.getMinimumWidth());
    }
```
这里，我们会根据前面所说的，父容器传递进来`measureSpec`中的`mode`来给这两个变量赋值：
- 如果`mode`为`UNSPECIFIED`，那么说明父容器并不指望多个，因此子`View`根据自己的背景或者`minHeight/minWidth`属性来给自己赋值。
- 如果是`AT_MOST`或者`EXACTLY`，那么就把它设置为父容器指定的`size`。

## 3.2 `ViewGroup`的`onMeasure`
由于`ViewGroup`的目的是为了容纳各子`View`，但是它并不确定子`View`应当如何排列，也就不知道该如何测量自己，因此它的`onMeasure`是没有任何意义的，所以并没有重写，而是应当由继承于它的控件来重写该方法。
## 3.3 继承于`ViewGroup`控件的`onMeasure`
为了方面，我们以`DecorView`为例，经过前面的分析，我们知道当我们在`performTraversals`中调用它的`measure`方法时，最终会回调到它对应的控件类型，也就是`FrameLayout`的`onMeasure`方法：
```
@Override
    protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
        int count = getChildCount();

        final boolean measureMatchParentChildren =
                MeasureSpec.getMode(widthMeasureSpec) != MeasureSpec.EXACTLY ||
                MeasureSpec.getMode(heightMeasureSpec) != MeasureSpec.EXACTLY;
        mMatchParentChildren.clear();

        int maxHeight = 0;
        int maxWidth = 0;
        int childState = 0;

        for (int i = 0; i < count; i++) {
            final View child = getChildAt(i);
            if (mMeasureAllChildren || child.getVisibility() != GONE) {
                measureChildWithMargins(child, widthMeasureSpec, 0, heightMeasureSpec, 0);
                final LayoutParams lp = (LayoutParams) child.getLayoutParams();
                maxWidth = Math.max(maxWidth,
                        child.getMeasuredWidth() + lp.leftMargin + lp.rightMargin);
                maxHeight = Math.max(maxHeight,
                        child.getMeasuredHeight() + lp.topMargin + lp.bottomMargin);
                childState = combineMeasuredStates(childState, child.getMeasuredState());
                if (measureMatchParentChildren) {
                    if (lp.width == LayoutParams.MATCH_PARENT ||
                            lp.height == LayoutParams.MATCH_PARENT) {
                        mMatchParentChildren.add(child);
                    }
                }
            }
        }

        // Account for padding too
        maxWidth += getPaddingLeftWithForeground() + getPaddingRightWithForeground();
        maxHeight += getPaddingTopWithForeground() + getPaddingBottomWithForeground();

        // Check against our minimum height and width
        maxHeight = Math.max(maxHeight, getSuggestedMinimumHeight());
        maxWidth = Math.max(maxWidth, getSuggestedMinimumWidth());

        // Check against our foreground's minimum height and width
        final Drawable drawable = getForeground();
        if (drawable != null) {
            maxHeight = Math.max(maxHeight, drawable.getMinimumHeight());
            maxWidth = Math.max(maxWidth, drawable.getMinimumWidth());
        }

        setMeasuredDimension(resolveSizeAndState(maxWidth, widthMeasureSpec, childState),
                resolveSizeAndState(maxHeight, heightMeasureSpec,
                        childState << MEASURED_HEIGHT_STATE_SHIFT));

        count = mMatchParentChildren.size();
        if (count > 1) {
            for (int i = 0; i < count; i++) {
                final View child = mMatchParentChildren.get(i);
                final MarginLayoutParams lp = (MarginLayoutParams) child.getLayoutParams();

                final int childWidthMeasureSpec;
                if (lp.width == LayoutParams.MATCH_PARENT) {
                    final int width = Math.max(0, getMeasuredWidth()
                            - getPaddingLeftWithForeground() - getPaddingRightWithForeground()
                            - lp.leftMargin - lp.rightMargin);
                    childWidthMeasureSpec = MeasureSpec.makeMeasureSpec(
                            width, MeasureSpec.EXACTLY);
                } else {
                    childWidthMeasureSpec = getChildMeasureSpec(widthMeasureSpec,
                            getPaddingLeftWithForeground() + getPaddingRightWithForeground() +
                            lp.leftMargin + lp.rightMargin,
                            lp.width);
                }

                final int childHeightMeasureSpec;
                if (lp.height == LayoutParams.MATCH_PARENT) {
                    final int height = Math.max(0, getMeasuredHeight()
                            - getPaddingTopWithForeground() - getPaddingBottomWithForeground()
                            - lp.topMargin - lp.bottomMargin);
                    childHeightMeasureSpec = MeasureSpec.makeMeasureSpec(
                            height, MeasureSpec.EXACTLY);
                } else {
                    childHeightMeasureSpec = getChildMeasureSpec(heightMeasureSpec,
                            getPaddingTopWithForeground() + getPaddingBottomWithForeground() +
                            lp.topMargin + lp.bottomMargin,
                            lp.height);
                }

                child.measure(childWidthMeasureSpec, childHeightMeasureSpec);
            }
        }
    }
```
我们可以看到，整个的`onMeasure`其实分为三步：
- 遍历所有子`View`，调用`measureChildWithMargins`进行第一次子`View`的测量，在第一节中，我们也分析了这个方法，它最终也是调用子`View`的`measure`方法。
- 根据第一步的结果，调用`setMeasuredDimension`来设置自己的测量结果。
- 遍历所有子`View`，根据第二步的结果，调用`child.measure`进行第二次的测量。

这也验证了第二节中的结论：**父容器和子`View`的关联是通过`measure`进行关联的**。
同时我们也可以有一个新的结论，对于`View`树的某个节点，它的测量结果有可能并不是一次决定的，这是由于父容器可能需要依赖于子`View`的测量结果，而父容器的结果又可能会影响子`View`，但是，我们需要保证这个过程不是无限调用的。
