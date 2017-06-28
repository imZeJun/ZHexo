---
title: View 绘制体系知识梳理(2) - setContentView 源码解析
date: 2017-02-23 11:11
categories : View 绘制体系知识梳理
---
# 一、概述
在`Activity`当中，我们一般都会调用`setContentView`方法来初始化布局。
# 二、与`ContentView`相关的方法
在`Activity`当中，与`ContentView`相关的函数有下面这几个，我们先看一下它们的注释说明：
```
    /**
     * Set the activity content from a layout resource.  The resource will be
     * inflated, adding all top-level views to the activity.
     *
     * @param layoutResID Resource ID to be inflated.
     *
     * @see #setContentView(android.view.View)
     * @see #setContentView(android.view.View, android.view.ViewGroup.LayoutParams)
     */
    public void setContentView(@LayoutRes int layoutResID) {
        getWindow().setContentView(layoutResID);
        initWindowDecorActionBar();
    }

    /**
     * Set the activity content to an explicit view.  This view is placed
     * directly into the activity's view hierarchy.  It can itself be a complex
     * view hierarchy.  When calling this method, the layout parameters of the
     * specified view are ignored.  Both the width and the height of the view are
     * set by default to {@link ViewGroup.LayoutParams#MATCH_PARENT}. To use
     * your own layout parameters, invoke
     * {@link #setContentView(android.view.View, android.view.ViewGroup.LayoutParams)}
     * instead.
     *
     * @param view The desired content to display.
     *
     * @see #setContentView(int)
     * @see #setContentView(android.view.View, android.view.ViewGroup.LayoutParams)
     */
    public void setContentView(View view) {
        getWindow().setContentView(view);
        initWindowDecorActionBar();
    }

    /**
     * Set the activity content to an explicit view.  This view is placed
     * directly into the activity's view hierarchy.  It can itself be a complex
     * view hierarchy.
     *
     * @param view The desired content to display.
     * @param params Layout parameters for the view.
     *
     * @see #setContentView(android.view.View)
     * @see #setContentView(int)
     */
    public void setContentView(View view, ViewGroup.LayoutParams params) {
        getWindow().setContentView(view, params);
        initWindowDecorActionBar();
    }

    /**
     * Add an additional content view to the activity.  Added after any existing
     * ones in the activity -- existing views are NOT removed.
     *
     * @param view The desired content to display.
     * @param params Layout parameters for the view.
     */
    public void addContentView(View view, ViewGroup.LayoutParams params) {
        getWindow().addContentView(view, params);
        initWindowDecorActionBar();
    }
```
通过上面的注释，可以看到这4个方法的用途：
- 第一种：渲染`layouResId`对应的布局，并将它添加到`activity`的顶级`View`中。
- 第二种：将`View`添加到`activity`的布局当中，它的默认宽高都是` ViewGroup.LayoutParams#MATCH_PARENT`。
- 第三种：和上面相同，但是指定了`LayoutParams`。
- 第四种：将内容添加进去，并且必须指定`LayoutParams`，已经存在的`View`不会被移除。

这四种方法其实都是调用了`PhoneWindow.java`中的方法，通过源码我们可以发现`setContentView(View view, ViewGroup.LayoutParams params)`和`setContentView(@LayoutRes int layoutResID)`的步骤基本上是一样的，只不过是在添加到布局的时候，前者因为已经获得了`View`的实例，因此用的是`addView`的方法，而后者因为需要先`inflate`，所以，使用的是`LayoutInflater`。

# 三、`setContentView`方法
下面我们以`setContentView(@LayoutRes int layoutResID)`为例，看一下具体的实现步骤：
## 3.1 `setContentView`
```
    @Override
    public void setContentView(int layoutResID) {
        // Note: FEATURE_CONTENT_TRANSITIONS may be set in the process of installing the window
        // decor, when theme attributes and the like are crystalized. Do not check the feature
        // before this happens.
        if (mContentParent == null) {
            installDecor();
        } else if (!hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
            mContentParent.removeAllViews();
        }

        if (hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
            final Scene newScene = Scene.getSceneForLayout(mContentParent, layoutResID,
                    getContext());
            transitionTo(newScene);
        } else {
            mLayoutInflater.inflate(layoutResID, mContentParent);
        }
        mContentParent.requestApplyInsets();
        final Callback cb = getCallback();
        if (cb != null && !isDestroyed()) {
            cb.onContentChanged();
        }
        mContentParentExplicitlySet = true;
    }
```
首先，我们会判断`mContentParent`是否为空，通过添加的代码我们可以知道，这个`mContentParent`其实就是`layoutResId`最后渲染出的布局所对应的父容器，当这个`ContentParent`为空时，调用了`installDecor`，`mContentParent`就是在里面初始化的。
## 3.2 `installDecor()`
```
    private void installDecor() {
        //如果DecorView不存在，那么先生成它，它其实是一个FrameLayout。
        if (mDecor == null) {
            mDecor = generateDecor();
        }
        //如果`ContentParent`不存在，那么也生成它，此时传入了前面的`DecorView`
        if (mContentParent == null) {
            mContentParent = generateLayout(mDecor);
            final DecorContentParent decorContentParent = (DecorContentParent) mDecor.findViewById(R.id.decor_content_parent);
            if (decorContentParent != null) {
                mDecorContentParent = decorContentParent;
            }
        }
    }
```
我们可以看到，`mDecor`是一个`FrameLayout`，它和`mContentParent`的关系是通过`            mContentParent = generateLayout(mDecor)`产生。
## 3.3 `generateLayout(DecorView decor)`
```
    protected ViewGroup generateLayout(DecorView decor) {
        //...首先根据不同的情况，给`layoutResource`赋予不同的值.
        View in = mLayoutInflater.inflate(layoutResource, null);
        decor.addView(in, new ViewGroup.LayoutParams(MATCH_PARENT, MATCH_PARENT));
        mContentRoot = (ViewGroup) in;
        ViewGroup contentParent = (ViewGroup) findViewById(ID_ANDROID_CONTENT);
        if (contentParent == null) {
            throw new RuntimeException("Window couldn't find content container view");
        }
        //...
        return contentParent;
    }
```
在上面赋值的过程中，我们主要关注以下几个变量，`mContentRoot/mContentParent/mDecorContent`：
- `mContentRoot`一定是`mDecor`的下一级子容器。
- `mContentParent `是`mDecor`当中`id`为`R.id.content`的`ViewGroup`，但是它**和`mDecor`的具体层级关系不确定**，这依赖于`mContentRoot`是通过哪个`xml`渲染出来。
- `mContentParent`**一定是传入的`layoutResId`进行 `inflate`完成之后的父容器**，它一定不会为空，否则会抛出异常，我们`setContentView(xxx)`方法传入的布局，就是它的子`View`。
```
    // This is the top-level view of the window, containing the window decor.
    private DecorView mDecor;

    // This is the view in which the window contents are placed. It is either
    // mDecor itself, or a child of mDecor where the contents go.
    private ViewGroup mContentParent;
```
- `mDecorContent`则是`mDecor`当中`id`为`decor_content_parent`的`ViewGroup`，但是也有可能`mDecor`
当中没有这个`id`的`View`，这需要依赖与我们的`mContentRoot`是使用了哪个`xml`来`inflate`的。

再回到前面`setContentView`的地方，继续往下看，**当`mContentParent`不为空的时候，那么会移除它底下的所有子`View`**。
之后会调用`mLayoutInflater.inflate(layoutResID, mContentParent);`方法，把传入的`View`添加到`mContentParent`当中，最后回调一个监听。
```
final Callback cb = getCallback();
if (cb != null && !isDestroyed()) {
    cb.onContentChanged();
}
```
## 3.4 `mContentParentExplicitlySet`标志位
在`setContentView`的最后，将`mContentParentExplicitlySet`这个变量设置为`true`，这个变量其实是用在`requestFeature`当中，也就是说，我们必须在调用`setContentView`之前，调用`requestFeature`，否则就会抛出下面的异常：
```
    @Override
    public boolean requestFeature(int featureId) {
        if (mContentParentExplicitlySet) {
            throw new AndroidRuntimeException("requestFeature() must be called before adding content");
        }
        return super.requestFeature(featureId);
    }
```
因此：**`requestFeature(xxx)`必须要在调用`setContentView(xxx)`之前**。

## 三、`addContentView(View view, ViewGroup.LayoutParams params)`
下面我们再来看一下，`addContentView`方法：
```
    @Override
    public void addContentView(View view, ViewGroup.LayoutParams params) {
        if (mContentParent == null) {
            installDecor();
        }
        mContentParent.addView(view, params);
        mContentParent.requestApplyInsets();
        final Callback cb = getCallback();
        if (cb != null && !isDestroyed()) {
            cb.onContentChanged();
        }
    }
```
可以看到，它和`set`方法的区别就是，它在添加到`mContentParent`之前，并没有把`mContentParent`的所有子`View`都移除，而是将它直接添加进去，通过布局分析软件，可以看到`mContentParent`的类型为`ContentFrameLayout`，它其实是一个`FrameLayout`，因此，它会**覆盖在`mContentParent`已有子`View`之上**。
## 四、将添加的布局和`Activity`的`Window`关联起来
在上面的分析当中，我们仅仅是初始化了一个`DecorView`，并根据设置的`Style`属性，传入的`ContentView`来初始化它的子布局，但是这时候它还有真正和`Activity`的`Window`关联起来，关联的地方在`ActivityThread.java`中：
```
final void handleResumeActivity(IBinder token,
            boolean clearHide, boolean isForward, boolean reallyResume, int seq, String reason) {
        r = performResumeActivity(token, clearHide, reason);
        if (r != null) {
            if (r.window == null && !a.mFinished && willBeVisible) {
                r.window = r.activity.getWindow();
                View decor = r.window.getDecorView();
                decor.setVisibility(View.INVISIBLE);
                ViewManager wm = a.getWindowManager();
                WindowManager.LayoutParams l = r.window.getAttributes();
                a.mDecor = decor;
                l.type = WindowManager.LayoutParams.TYPE_BASE_APPLICATION;
                l.softInputMode |= forwardBit;
                if (r.mPreserveWindow) {
                    a.mWindowAdded = true;
                    r.mPreserveWindow = false;
                    // Normally the ViewRoot sets up callbacks with the Activity
                    // in addView->ViewRootImpl#setView. If we are instead reusing
                    // the decor view we have to notify the view root that the
                    // callbacks may have changed.
                    ViewRootImpl impl = decor.getViewRootImpl();
                    if (impl != null) {
                        impl.notifyChildRebuilt();
                    }
                }
                if (a.mVisibleFromClient && !a.mWindowAdded) {
                    a.mWindowAdded = true;
                    wm.addView(decor, l);
                }
            } else if (!willBeVisible) {
                if (localLOGV) Slog.v(
                    TAG, "Launch " + r + " mStartedActivity set");
                r.hideForNow = true;
            }
        } else {

        }
    }
```
从源码中可以看到，如果在执行`handleResumeActivity`时，之前`DecorView`没有被添加到`WindowManager`当中时，那么它的**第一次添加是在`onResume()`方法执行完之后添加的**。
