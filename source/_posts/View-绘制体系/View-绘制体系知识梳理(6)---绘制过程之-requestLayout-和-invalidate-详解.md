---
title: View 绘制体系知识梳理(6) - 绘制过程之 requestLayout 和 invalidate 详解
date: 2017-02-28 21:06
categories : View 绘制体系知识梳理
---
# 一、概述
经过前面三篇文章的分析：
- [`绘制流程 - Measure`](http://www.jianshu.com/p/8bfdeaf01661)
- [`绘制过程 - Layout`](http://www.jianshu.com/p/6b8c99dd2dd6)
- [`绘制过程 - Draw`](http://www.jianshu.com/p/531804c77247)

对于绘制的整个分发过程已经有了一个大致的了解，我们可以发现一个规律，无论是测量、布局还是绘制，对于任何一个`View/Group`来说，它都是一个至上而下的递归事件调用，直到到达整个`View`树的叶节点为止。
下面，我们来分析几个平时常用的方法：
- `requestLayout`
- `invalidate`
- `postInvalidate`

# 二、`requestLayout`
`requestLayout`是在`View`中定义的，并且在`ViewGroup`中没有重写该方法，它的注释是这样解释的：在需要刷新`View`的布局时调用这个函数，它会安排一个布局的传递。我们不应该在布局的过程中（`isInLayout()`）调用这个函数，如果当前正在布局，那么这一请求有可能在以下时刻被执行：当前布局结束、当前帧被绘制完或者下次布局发生时。
```
    /**
     * Call this when something has changed which has invalidated the
     * layout of this view. This will schedule a layout pass of the view
     * tree. This should not be called while the view hierarchy is currently in a layout
     * pass ({@link #isInLayout()}. If layout is happening, the request may be honored at the
     * end of the current layout pass (and then layout will run again) or after the current
     * frame is drawn and the next layout occurs.
     *
     * <p>Subclasses which override this method should call the superclass method to
     * handle possible request-during-layout errors correctly.</p>
     */
    @CallSuper
    public void requestLayout() {
        if (mMeasureCache != null) mMeasureCache.clear();

        if (mAttachInfo != null && mAttachInfo.mViewRequestingLayout == null) {
            // Only trigger request-during-layout logic if this is the view requesting it,
            // not the views in its parent hierarchy
            ViewRootImpl viewRoot = getViewRootImpl();
            if (viewRoot != null && viewRoot.isInLayout()) {
                if (!viewRoot.requestLayoutDuringLayout(this)) {
                    return;
                }
            }
            mAttachInfo.mViewRequestingLayout = this;
        }

        mPrivateFlags |= PFLAG_FORCE_LAYOUT;
        mPrivateFlags |= PFLAG_INVALIDATED;

        if (mParent != null && !mParent.isLayoutRequested()) {
            mParent.requestLayout();
        }
        if (mAttachInfo != null && mAttachInfo.mViewRequestingLayout == this) {
            mAttachInfo.mViewRequestingLayout = null;
        }
    }
```
在上面的代码当中，设置了两个标志位：`PFLAG_FORCE_LAYOUT/PFLAG_INVALIDATED`，除此之外最关键的一句话是：
```
protected ViewParent mParent;
//....
mParent.requestLayout();
```
这个`mParent`存储的时候该`View`所对应的父节点，而当调用父节点的`requestLayout()`时，它又会调用它的父节点的`requestLayout`，就这样，以调用`requestLayout`的`View`为起始节点，一步步沿着`View`树传递上去，那么这个过程什么时候会终止呢？
根据前面的分析，我们知道整个`View`树的根节点是`DecorView`，那么我们需要看一下`DecorView`的`mParent`变量是什么，回到`ViewRootImpl`的`setView`方法当中，有这么一句：
```
view.assignParent(this);
```
因此，`DecorView`中的`mParent`就是`ViewRootImpl`，而`ViewRootImpl`中的`mView`就是`DecorView`，所以，这一传递过程的终点就是`ViewRootImpl`的`requestLayout`方法：
```
    //ViewRootImpl中的requestLayout方法.
    @Override
    public void requestLayout() {
        if (!mHandlingLayoutInLayoutRequest) {
            checkThread();
            mLayoutRequested = true;
            scheduleTraversals();
        }
    }

    void scheduleTraversals() {
        if (!mTraversalScheduled) {
            mTraversalScheduled = true;
            mTraversalBarrier = mHandler.getLooper().getQueue().postSyncBarrier();
            //该Runnable进行操作doTraversal.
            mChoreographer.postCallback(Choreographer.CALLBACK_TRAVERSAL, mTraversalRunnable, null);
            if (!mUnbufferedInputDispatch) {
                scheduleConsumeBatchedInput();
            }
            notifyRendererOfFramePending();
            pokeDrawLockIfNeeded();
        }
    }

    final class TraversalRunnable implements Runnable {
        @Override
        public void run() {
            doTraversal();
        }
    }

    void doTraversal() {
        if (mTraversalScheduled) {
            mTraversalScheduled = false;
            mHandler.getLooper().getQueue().removeSyncBarrier(mTraversalBarrier);

            if (mProfile) {
                Debug.startMethodTracing("ViewAncestor");
            }
            //这里最终会进行布局.
            performTraversals();

            if (mProfile) {
                Debug.stopMethodTracing();
                mProfile = false;
            }
        }
    }
```
其中`scheduleTraversals()`中会执行一个`mTraversalRunnable`，该`Runnable`中最终会调用`doTraversal`，而`doTraversal`中执行的就是我们前面一直在谈到的`performTraversals`。
那么，前面我们分析过，`performTraversals`的`measure`方法会从根节点调用子节点的测量操作，并依次传递下去，那么是否所有的子`View`都有必要重新测量呢，这就需要我们在调用`View`的`requestLayout`是设置的标志位`PFLAG_FORCE_LAYOUT`来判断，在`measure`当中，调用`onMeasure`之前，会有这么一个判断条件：
```
if ((mPrivateFlags & PFLAG_FORCE_LAYOUT) == PFLAG_FORCE_LAYOUT ||
                widthMeasureSpec != mOldWidthMeasureSpec ||
                heightMeasureSpec != mOldHeightMeasureSpec) {
    onMeasure(widthMeasureSpec, heightMeasureSpec);
}
```
这个标志位会在`layout`完成之后被恢复：
```
    public void layout(int l, int t, int r, int b) {
        if (changed || (mPrivateFlags & PFLAG_LAYOUT_REQUIRED) == PFLAG_LAYOUT_REQUIRED) {
            onLayout(changed, l, t, r, b);
        }
        mPrivateFlags &= ~PFLAG_FORCE_LAYOUT;
        mPrivateFlags3 |= PFLAG3_IS_LAID_OUT;
    }
```
在进行完`layout`之后，`requestLayout()`所引发的过程就此终止了，它不会调用`draw`，不会重新绘制任何视图包括该调用者本身。
# 三、`invalidate`
`invalidate`最终会调用到下面这个方法：
```
    void invalidateInternal(int l, int t, int r, int b, boolean invalidateCache,
            boolean fullInvalidate) {
        if (mGhostView != null) {
            mGhostView.invalidate(true);
            return;
        }

        if (skipInvalidate()) {
            return;
        }

        if ((mPrivateFlags & (PFLAG_DRAWN | PFLAG_HAS_BOUNDS)) == (PFLAG_DRAWN | PFLAG_HAS_BOUNDS)
                || (invalidateCache && (mPrivateFlags & PFLAG_DRAWING_CACHE_VALID) == PFLAG_DRAWING_CACHE_VALID)
                || (mPrivateFlags & PFLAG_INVALIDATED) != PFLAG_INVALIDATED
                || (fullInvalidate && isOpaque() != mLastIsOpaque)) {
            if (fullInvalidate) {
                mLastIsOpaque = isOpaque();
                mPrivateFlags &= ~PFLAG_DRAWN;
            }

            mPrivateFlags |= PFLAG_DIRTY;

            if (invalidateCache) {
                mPrivateFlags |= PFLAG_INVALIDATED;
                mPrivateFlags &= ~PFLAG_DRAWING_CACHE_VALID;
            }

            // Propagate the damage rectangle to the parent view.
            final AttachInfo ai = mAttachInfo;
            final ViewParent p = mParent;
            if (p != null && ai != null && l < r && t < b) {
                final Rect damage = ai.mTmpInvalRect;
                damage.set(l, t, r, b);
                p.invalidateChild(this, damage);
            }

            // Damage the entire projection receiver, if necessary.
            if (mBackground != null && mBackground.isProjected()) {
                final View receiver = getProjectionReceiver();
                if (receiver != null) {
                    receiver.damageInParent();
                }
            }

            // Damage the entire IsolatedZVolume receiving this view's shadow.
            if (isHardwareAccelerated() && getZ() != 0) {
                damageShadowReceiver();
            }
        }
    }
```
其中，关键的一句是：
```
p.invalidateChild(this, damage);
```
在这里，`p`一定不为空并且它一定是一个`ViewGroup`，那么我们来看一下`ViewGroup`的这个方法：
```
public final void invalidateChild(View child, final Rect dirty) {
    do {
        parent = parent.invalidateChildInParent(location, dirty);
    } while (parent != null);
}
```
而`ViewGroup`当中的`invalidateChildInParent`会根据传入的区域来决定自己的绘制区域，和`requestLayout`类似，最终会调用`ViewRootImpl`的该方法：
```
    @Override
    public ViewParent invalidateChildInParent(int[] location, Rect dirty) {
        checkThread();
        if (DEBUG_DRAW) Log.v(TAG, "Invalidate child: " + dirty);

        if (dirty == null) {
            invalidate();
            return null;
        } else if (dirty.isEmpty() && !mIsAnimating) {
            return null;
        }

        if (mCurScrollY != 0 || mTranslator != null) {
            mTempRect.set(dirty);
            dirty = mTempRect;
            if (mCurScrollY != 0) {
                dirty.offset(0, -mCurScrollY);
            }
            if (mTranslator != null) {
                mTranslator.translateRectInAppWindowToScreen(dirty);
            }
            if (mAttachInfo.mScalingRequired) {
                dirty.inset(-1, -1);
            }
        }
        invalidateRectOnScreen(dirty);
        return null;
    }
```
这其中又会调用`invalidate`：
```
    void invalidate() {
        mDirty.set(0, 0, mWidth, mHeight);
        if (!mWillDrawSoon) {
            scheduleTraversals();
        }
    }
```
这里，最终又会走到前面说的`performTraversals()`方法，请求重绘`View`树，即`draw()`过程，假如视图发生大小没有变化就不会调用`layout()`过程，并且只绘制那些需要重绘的视图。
# 三、其它知识点
- `invalidate`，请求重新`draw`，只会绘制调用者本身。
- `setSelection`，同上。
- `setVisibility`：当`View`从`INVISIBLE`变为`VISIBILE`，会间接调用`invalidate`方法，继而绘制该`View`，而从`INVISIBLE/VISIBLE`变为`GONE`之后，由于`View`树的大小发生了变化，会进行`measure/layout/draw`，同样，他只会绘制需要重绘的视图。
- `setEnable`：请求重新`draw`，只会绘制调用者本身。
- `requestFocus`：请求重新`draw`，只会绘制需要重绘的视图。

# 四、参考文献
[`1.http://blog.csdn.net/yanbober/article/details/46128379/`](http://blog.csdn.net/yanbober/article/details/46128379/)
[`2.http://blog.csdn.net/a553181867/article/details/51583060`](http://blog.csdn.net/a553181867/article/details/51583060)
