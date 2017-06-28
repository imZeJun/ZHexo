---
title: View 绘制体系知识梳理(6) - 绘制过程之 requestLayout 和 invalidate 详解
date: 2017-02-28 13:08
categories : View 绘制体系知识梳理
---
# 一、绘制的起点 - `performTraversals`
和测量、布局的过程类似，绘制的起点也是从`performTraversals`开始的：
```
private void performTraversals() {
    ......
    final Rect dirty = mDirty;
    ......
    canvas = mSurface.lockCanvas(dirty);
    ......
    mView.draw(canvas);
    ......
}
```
# 二、绘制的关键方法：`draw(Canvas canvas) onDraw(Canvas canvas) dispatchDraw()`
和绘制过程的前两个阶段不同的是，与`draw`过程相关的函数，不仅有`draw`、`onDraw`还有一个`dispatchDraw`，下面我们分析一下这几个函数在`View`和`ViewGroup`中的**定义**：
- `View`
 - 首先是`draw(Canvas)`方法，它是`public`的，我们通过注释可以看到它最多会执行6步操作，分别是：绘制背景、保存`canvas`，绘制`View`本身内容，绘制子`View`，恢复`canvas`，绘制装饰类（如滚动条）。其中第二、五步不是必要的。
对于整个绘制事件的分发，我们需要关注的是第三、四步：调用`onDraw(Canvas)`绘制`View`本身的的内容，调用`dispatchDraw(Canvas)`绘制子`View`，也就是我们上面说的其它两个方法。
```
@CallSuper
    public void draw(Canvas canvas) {
        final int privateFlags = mPrivateFlags;
        final boolean dirtyOpaque = (privateFlags & PFLAG_DIRTY_MASK) == PFLAG_DIRTY_OPAQUE &&
                (mAttachInfo == null || !mAttachInfo.mIgnoreDirtyState);
        mPrivateFlags = (privateFlags & ~PFLAG_DIRTY_MASK) | PFLAG_DRAWN;

        /*
         * Draw traversal performs several drawing steps which must be executed
         * in the appropriate order:
         *
         *      1. Draw the background
         *      2. If necessary, save the canvas' layers to prepare for fading
         *      3. Draw view's content
         *      4. Draw children
         *      5. If necessary, draw the fading edges and restore layers
         *      6. Draw decorations (scrollbars for instance)
         */

        // Step 1, draw the background, if needed
        int saveCount;

        if (!dirtyOpaque) {
            drawBackground(canvas);
        }

        // skip step 2 & 5 if possible (common case)
        final int viewFlags = mViewFlags;
        boolean horizontalEdges = (viewFlags & FADING_EDGE_HORIZONTAL) != 0;
        boolean verticalEdges = (viewFlags & FADING_EDGE_VERTICAL) != 0;
        if (!verticalEdges && !horizontalEdges) {
            // Step 3, draw the content
            if (!dirtyOpaque) onDraw(canvas);

            // Step 4, draw the children
            dispatchDraw(canvas);

            // Overlay is part of the content and draws beneath Foreground
            if (mOverlay != null && !mOverlay.isEmpty()) {
                mOverlay.getOverlayView().dispatchDraw(canvas);
            }

            // Step 6, draw decorations (foreground, scrollbars)
            onDrawForeground(canvas);

            // we're done...
            return;
        }

        /*
         * Here we do the full fledged routine...
         * (this is an uncommon case where speed matters less,
         * this is why we repeat some of the tests that have been
         * done above)
         */

        boolean drawTop = false;
        boolean drawBottom = false;
        boolean drawLeft = false;
        boolean drawRight = false;

        float topFadeStrength = 0.0f;
        float bottomFadeStrength = 0.0f;
        float leftFadeStrength = 0.0f;
        float rightFadeStrength = 0.0f;

        // Step 2, save the canvas' layers
        int paddingLeft = mPaddingLeft;

        final boolean offsetRequired = isPaddingOffsetRequired();
        if (offsetRequired) {
            paddingLeft += getLeftPaddingOffset();
        }

        int left = mScrollX + paddingLeft;
        int right = left + mRight - mLeft - mPaddingRight - paddingLeft;
        int top = mScrollY + getFadeTop(offsetRequired);
        int bottom = top + getFadeHeight(offsetRequired);

        if (offsetRequired) {
            right += getRightPaddingOffset();
            bottom += getBottomPaddingOffset();
        }

        final ScrollabilityCache scrollabilityCache = mScrollCache;
        final float fadeHeight = scrollabilityCache.fadingEdgeLength;
        int length = (int) fadeHeight;

        // clip the fade length if top and bottom fades overlap
        // overlapping fades produce odd-looking artifacts
        if (verticalEdges && (top + length > bottom - length)) {
            length = (bottom - top) / 2;
        }

        // also clip horizontal fades if necessary
        if (horizontalEdges && (left + length > right - length)) {
            length = (right - left) / 2;
        }

        if (verticalEdges) {
            topFadeStrength = Math.max(0.0f, Math.min(1.0f, getTopFadingEdgeStrength()));
            drawTop = topFadeStrength * fadeHeight > 1.0f;
            bottomFadeStrength = Math.max(0.0f, Math.min(1.0f, getBottomFadingEdgeStrength()));
            drawBottom = bottomFadeStrength * fadeHeight > 1.0f;
        }

        if (horizontalEdges) {
            leftFadeStrength = Math.max(0.0f, Math.min(1.0f, getLeftFadingEdgeStrength()));
            drawLeft = leftFadeStrength * fadeHeight > 1.0f;
            rightFadeStrength = Math.max(0.0f, Math.min(1.0f, getRightFadingEdgeStrength()));
            drawRight = rightFadeStrength * fadeHeight > 1.0f;
        }

        saveCount = canvas.getSaveCount();

        int solidColor = getSolidColor();
        if (solidColor == 0) {
            final int flags = Canvas.HAS_ALPHA_LAYER_SAVE_FLAG;

            if (drawTop) {
                canvas.saveLayer(left, top, right, top + length, null, flags);
            }

            if (drawBottom) {
                canvas.saveLayer(left, bottom - length, right, bottom, null, flags);
            }

            if (drawLeft) {
                canvas.saveLayer(left, top, left + length, bottom, null, flags);
            }

            if (drawRight) {
                canvas.saveLayer(right - length, top, right, bottom, null, flags);
            }
        } else {
            scrollabilityCache.setFadeColor(solidColor);
        }

        // Step 3, draw the content
        if (!dirtyOpaque) onDraw(canvas);

        // Step 4, draw the children
        dispatchDraw(canvas);

        // Step 5, draw the fade effect and restore layers
        final Paint p = scrollabilityCache.paint;
        final Matrix matrix = scrollabilityCache.matrix;
        final Shader fade = scrollabilityCache.shader;

        if (drawTop) {
            matrix.setScale(1, fadeHeight * topFadeStrength);
            matrix.postTranslate(left, top);
            fade.setLocalMatrix(matrix);
            p.setShader(fade);
            canvas.drawRect(left, top, right, top + length, p);
        }

        if (drawBottom) {
            matrix.setScale(1, fadeHeight * bottomFadeStrength);
            matrix.postRotate(180);
            matrix.postTranslate(left, bottom);
            fade.setLocalMatrix(matrix);
            p.setShader(fade);
            canvas.drawRect(left, bottom - length, right, bottom, p);
        }

        if (drawLeft) {
            matrix.setScale(1, fadeHeight * leftFadeStrength);
            matrix.postRotate(-90);
            matrix.postTranslate(left, top);
            fade.setLocalMatrix(matrix);
            p.setShader(fade);
            canvas.drawRect(left, top, left + length, bottom, p);
        }

        if (drawRight) {
            matrix.setScale(1, fadeHeight * rightFadeStrength);
            matrix.postRotate(90);
            matrix.postTranslate(right, top);
            fade.setLocalMatrix(matrix);
            p.setShader(fade);
            canvas.drawRect(right - length, top, right, bottom, p);
        }

        canvas.restoreToCount(saveCount);

        // Overlay is part of the content and draws beneath Foreground
        if (mOverlay != null && !mOverlay.isEmpty()) {
            mOverlay.getOverlayView().dispatchDraw(canvas);
        }

        // Step 6, draw decorations (foreground, scrollbars)
        onDrawForeground(canvas);
    }
```
 - `onDraw(Canvas canvas)`方法是一个空实现，因为绘制应当由具体的控件来实现。
 - 由于单纯的`View`来说，它没有子`View`，因此`dispatchDraw`方法也是一个空实现。
- `ViewGroup`
 - 没有重写`draw(Canvas canvas)`方法。
 - 没有重写`onDraw(Canvas canvas)`方法。
 - 由于`ViewGroup`要负责将绘制的消息通知子`View`，所以它重写了`dispatchDraw`方法，并在里面调用`drawChild`一级级把绘制的消息传递下去。

 ```
protected void dispatchDraw(Canvas canvas) {
        //....
        for (int i = 0; i < childrenCount; i++) {
            while (transientIndex >= 0 && mTransientIndices.get(transientIndex) == i) {
                final View transientChild = mTransientViews.get(transientIndex);
                if ((transientChild.mViewFlags & VISIBILITY_MASK) == VISIBLE ||
                        transientChild.getAnimation() != null) {
                    more |= drawChild(canvas, transientChild, drawingTime);
                }
                transientIndex++;
                if (transientIndex >= transientCount) {
                    transientIndex = -1;
                }
            }
            int childIndex = customOrder ? getChildDrawingOrder(childrenCount, i) : i;
            final View child = (preorderedList == null)
                    ? children[childIndex] : preorderedList.get(childIndex);
            if ((child.mViewFlags & VISIBILITY_MASK) == VISIBLE || child.getAnimation() != null) {
                more |= drawChild(canvas, child, drawingTime);
            }
        }
        //...
}
protected boolean drawChild(Canvas canvas, View child, long drawingTime) {
        return child.draw(canvas, this, drawingTime);
}
```
通过分析`View/ViewGroup`的这三个方法的定义，那么整个由`DecorView`到个叶节点`View`的绘制过程就很清楚了：
- 首先调用了`mView`的`draw`方法，之前我们分析过它其实就是类型为`FrameLayout`的`DecorView`，由于`FrameLayout`和`ViewGroup`都没有重写`draw`方法，它其实是调用了`View`当中的`draw`执行前面分析过的那些步骤，首先是调用自己的`onDraw`方法来绘制自己，然后调用`ViewGroup`的`dispatchDraw`来绘制子`View`，最终会调用到这些子`View`的`draw`方法。
- 当第一步的这些子`View`的`draw`方法被调用时，都会先执行它的`onDraw`方法，这个大家都是一样的，但是注意到对于每个具体的子`View`，由于`dispatchDraw`方法决定了事件是否会被分发，所以我们会有以下几种情况：
 - 继承于`ViewGroup`的控件，但是没有下一级子`View`，虽然`ViewGroup`重写了`dispatchDraw`方法，但是此时`childCount==0`，因此`drawChild`不会被调用，整个传递过程终止。
 - 继承于`View`的控件，此时`dispatchDraw`方法是一个空实现，因此不会继续传递下去。
 - 继承于`ViewGroup`的控件，并且有子`View`，由于`ViewGroup`重写了`dispatchDraw`方法，因此会继续调用子`View`的`draw`方法传递下去。

# 三、小结
对于这三个关键的方法，简单的总结一下它们的作用：
- `draw`：作为整个父容器和子`View`之间绘制事件传递的纽带。
- `onDraw`：负责`View`本身的具体绘制。
- `dispatchDraw`：负责将绘制分发给下一级的子`View`。
