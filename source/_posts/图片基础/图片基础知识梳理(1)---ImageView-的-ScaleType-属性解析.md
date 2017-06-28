---
title: 图片基础知识梳理(1) - ImageView 的 ScaleType 属性解析
date: 2017-03-14 20:03
categories : 图片基础知识梳理
---
# 一、概述
在使用`ImageView`的过程当中，经常需要通过`scaleType`来对原始的图像进行处理，使得它能在空间中合理地展示。
# 二、`scaleType`的分类
首先，我们简单介绍一下`scaleType`的分类：
## 2.1 通过`Matrix`设置
这种情况下，对应的模式只有一种：
- `ScaleType.MATRIX`

最终，在这种情况下，我们可以同`setImageMatrix(Matrix matrix)`来改变。
## 2.2 填充类型
这一类属性的特点就是**通过拉伸或者压缩图片，使得原图片中所有元素都能够展现，并且至少填满控件`x,y`轴的其中一个**。
一共有四类：
- `ScaleType.FIX_XY`：**不考虑原图的比例**，拉伸或者压缩使得它等于控件的宽高。

下面三种都会**维持原图的比例**，使得它们的`x,y`都小于等于控件的宽高，只是最终的图形放的位置不同。
- `ScaleType.FIT_START`：放置在左上角。
- `ScaleType.FIT_CENTER`：放置在中间。
- `ScaleType.FIT_END`：放置在右下角。

## 2.3 中心重合类型
下面的三种类型都会使得控件的中心和图片中心重合：
- `ScaleType.CENTER`
要求一点：
 - 原图的中心和控件的中心重合
- `ScaleType.CENTER_CROP`
要求三点：
 - 整个控件能够被填满
 - 原图的比例不变
 - 原图的中心和控件的中心重合。
 - 保证原图的`x,y`轴上的元素至少有一个在控件中能被完全展示，那么有一下两种情况，在下面的操作做完之后，裁剪掉多余的部分：
   - 如果原图没有填满控件，那么会慢慢按比例放大，直到填满控件；
   - 如果原图已经填满控件，那么它会慢慢缩小，直到某一边和控件重合。

- `ScaleType.CENTER_INSIDE`
要求三点：
 - 原图的所有像素位于控件内部
 - 原图的比例不变
 - 图片的中心和控件的中心重合。

它不要求原始图片填满`x,y`轴的任意一个，因此，**如果原图的长宽都小于等于控件的长宽，不会进行放大操作，这也是它和`ScaleType.FIT_CENTER`的区别**。

# 三、示例
下面，我们通过一个简单的`Demo`来展示一下各种类型的具体表现，我们有两个大小一样的`ImageView`和两个大小不同的原图，其中左边的`ImageView`要比原图小，右边的`ImageView`要比原图大。
- `ScaleType.FIX_XY`：
![](http://upload-images.jianshu.io/upload_images/1949836-533142642e255233.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
- `ScaleType.FIT_START`：
![](http://upload-images.jianshu.io/upload_images/1949836-5f205b5a6f3d542b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
- `ScaleType.FIT_CENTER`
![](http://upload-images.jianshu.io/upload_images/1949836-3ddf5ea76bfec809.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
- `ScaleType.FIT_END`
![](http://upload-images.jianshu.io/upload_images/1949836-af2fa4c47d3d77c4.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
- `ScaleType.CENTER`
![](http://upload-images.jianshu.io/upload_images/1949836-ebde8261d02d6642.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
- `ScaleType.CENTER_CROP`
![](http://upload-images.jianshu.io/upload_images/1949836-0511928f95fb59e9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
- `ScaleType.CENTER_INSIDE`
![](http://upload-images.jianshu.io/upload_images/1949836-b01b50fdc39ece39.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

# 四、源码分析
## 4.1 给`ImageVIew`设置`src`的接口
在`ImageView`当中，设置图片的接口主要有下面几个函数：
```
public void setImageBitmap(Bitmap bm)
public void setImageResource(@DrawableRes int resId)
public void setImageURI(@Nullable Uri uri)
public void setImageDrawable(@Nullable Drawable drawable)
```
## 4.2 `setImageBitmap`的流程
我们就以平时常用的`setImageBitmap`为例，分析一下它整个的流程：
- 第一步：当我们调用`setImageBitmap`之后，它会把`Bitmap`封装在`BitmapDrawable`当中，之后调用了`setImageDrawable(Drawable drawable)`方法：
```
    public void setImageBitmap(Bitmap bm) {
        mDrawable = null;
        if (mRecycleableBitmapDrawable == null) {
            mRecycleableBitmapDrawable = new BitmapDrawable(mContext.getResources(), bm);
        } else {
            mRecycleableBitmapDrawable.setBitmap(bm);
        }
        setImageDrawable(mRecycleableBitmapDrawable);
    }
```
- 第二步：调用`setImageDrawable`
```
    public void setImageDrawable(@Nullable Drawable drawable) {
        //如果不是同一个资源.
        if (mDrawable != drawable) {
            mResource = 0;
            mUri = null;
            //旧的宽高.
            final int oldWidth = mDrawableWidth;
            final int oldHeight = mDrawableHeight;
            //关键方法
            updateDrawable(drawable);
            //如果宽高不同，才请求重新布局.
            if (oldWidth != mDrawableWidth || oldHeight != mDrawableHeight) {
                requestLayout();
            }
            //只要替换了资源就需要重新绘制.
            invalidate();
        }
    }
```
上面的关键方法在`updateDrawable`当中：
```
    private void updateDrawable(Drawable d) {
        //....
        if (d != null) {
            //...
            //这里面根据scaleType配置mDrawMatrix.
            configureBounds();
        } else {
            mDrawableWidth = mDrawableHeight = -1;
        }
    }
```
## 4.3 `configureBounds`改变`Matrix`
在`configureBounds`里就会根据我们所配置的`scaleType`来决定`mDrawable`如何显示，在这里面有一个重要的变量`mDrawMatrix`，我们前面说到的所有变换都是通过它来实现的，当然，我们除了可以让系统自己根据`scaleType`来生成`matrix`，也可以通过`setImageMatrix`手动的指定自己的变换：
```
    private void configureBounds() {
        if (mDrawable == null || !mHaveFrame) {
            return;
        }
　     //1.得到原始资源的宽高.
        final int dwidth = mDrawableWidth;
        final int dheight = mDrawableHeight;
        //2.得到控件的宽高，这里去掉了控件的padding.
        final int vwidth = getWidth() - mPaddingLeft - mPaddingRight;
        final int vheight = getHeight() - mPaddingTop - mPaddingBottom;

        //3.表示原始资源已经能够填满控件.
        final boolean fits = (dwidth < 0 || vwidth == dwidth)
                && (dheight < 0 || vheight == dheight);

        //4.假如有一边是wrap_content，或者是FIX_XY，那么填满整个控件.
        if (dwidth <= 0 || dheight <= 0 || ScaleType.FIT_XY == mScaleType) {
            mDrawable.setBounds(0, 0, vwidth, vheight);
            mDrawMatrix = null;
        } else {
            mDrawable.setBounds(0, 0, dwidth, dheight);
            //如果scaleType是matrix.
            if (ScaleType.MATRIX == mScaleType) {
                //单位矩阵的情况，设为null.
                if (mMatrix.isIdentity()) {
                    mDrawMatrix = null;
                } else {
                    //否则最后的DrawMatrix就是我们传入的Matrix.
                    mDrawMatrix = mMatrix;
                }
            } else if (fits) {
                //如果原始资源已经填满控件，那么不需要考虑其它的变换了.
                mDrawMatrix = null;
            } else if (ScaleType.CENTER == mScaleType) {
                //当scaleType为center的时候.
                mDrawMatrix = mMatrix;
                //移动到中心，初始时候，控件和原始资源的(0,0)点是重合的.
                mDrawMatrix.setTranslate(Math.round((vwidth - dwidth) * 0.5f), Math.round((vheight - dheight) * 0.5f));
            } else if (ScaleType.CENTER_CROP == mScaleType) {
                mDrawMatrix = mMatrix;
                //当scaleType是centerCrop的时候.
                float scale;
                float dx = 0, dy = 0;
                //取需要变换最小的轴，进行等比缩放.
                if (dwidth * vheight > vwidth * dheight) {
                    scale = (float) vheight / (float) dheight;
                    dx = (vwidth - dwidth * scale) * 0.5f;
                } else {
                    scale = (float) vwidth / (float) dwidth;
                    dy = (vheight - dheight * scale) * 0.5f;
                }
                //先缩放，再移动到中心点.
                mDrawMatrix.setScale(scale, scale);
                mDrawMatrix.postTranslate(Math.round(dx), Math.round(dy));
            } else if (ScaleType.CENTER_INSIDE == mScaleType) {
                //当scaleType是centetInside时
                mDrawMatrix = mMatrix;
                float scale;
                float dx;
                float dy;
                //如果原始资源的宽高都小于控件的宽高，那么不做缩放.
                if (dwidth <= vwidth && dheight <= vheight) {
                    scale = 1.0f;
                } else {
                   //否则等比缩放.
                    scale = Math.min((float) vwidth / (float) dwidth,
                            (float) vheight / (float) dheight);
                }

                dx = Math.round((vwidth - dwidth * scale) * 0.5f);
                dy = Math.round((vheight - dheight * scale) * 0.5f);
                //和上面类似，也是先缩放后平移.
                mDrawMatrix.setScale(scale, scale);
                mDrawMatrix.postTranslate(dx, dy);
            } else {
                //设置两个区域的大小.
                mTempSrc.set(0, 0, dwidth, dheight);
                mTempDst.set(0, 0, vwidth, vheight);
                mDrawMatrix = mMatrix;
                //这里处理FIX_START,FIX_END,FIX_CENTER的情况.
                mDrawMatrix.setRectToRect(mTempSrc, mTempDst, scaleTypeToScaleToFit(mScaleType));
            }
        }
    }
```
## 4.4 `onDraw`中进行绘制
那么这个`mDrawMatrix`是在什么时候使用的呢，我们看一下`onDraw`方法：
```
    protected void onDraw(Canvas canvas) {
        super.onDraw(canvas);

        if (mDrawMatrix == null && mPaddingTop == 0 && mPaddingLeft == 0) {
            mDrawable.draw(canvas);
        } else {
            final int saveCount = canvas.getSaveCount();
            //创建一个新的图层.
            canvas.save();
            if (mCropToPadding) {
                final int scrollX = mScrollX;
                final int scrollY = mScrollY;
                canvas.clipRect(scrollX + mPaddingLeft, scrollY + mPaddingTop,
                        scrollX + mRight - mLeft - mPaddingRight,
                        scrollY + mBottom - mTop - mPaddingBottom);
            }
            //移动到去掉padding的左上角.
            canvas.translate(mPaddingLeft, mPaddingTop);
            //根据mDrawMatrix进行变换.
            if (mDrawMatrix != null) {
                canvas.concat(mDrawMatrix);
            }
            //再在这个变换上面绘制我们的资源.
            mDrawable.draw(canvas);
            //把合成完成的图片绘制上去.
            canvas.restoreToCount(saveCount);
        }
    }
```
## 4.5 小结
我们总结一下，整个`scaleType`的原理就是在`configureBounds`中配置了`mDrawMatrix`，而在`onDraw`当中会根据`mDrawMatrix`来对图层进行变换，在这个变换之后的图层上进行绘制`mDrawable`，之后再恢复图层。
## 五、`ImageView`的`src`和`background`的区别
上面，我们看到的都是`src`设置的效果，我们回忆一下，通过设置`android:background`也可以设置一个图片给它，其实`background`是`View`的属性，在我们之前分析`View`的绘制流程的时候，`draw(canvas)`中有一步就是绘制背景：
```
    private void drawBackground(Canvas canvas) {
        //1.设置背景的边界.
        setBackgroundBounds();
       //2.如果有滚动，那么背景需要相应的滚动.
        final int scrollX = mScrollX;
        final int scrollY = mScrollY;
        if ((scrollX | scrollY) == 0) {
            background.draw(canvas);
        } else {
            canvas.translate(scrollX, scrollY);
            //3.绘制背景.
            background.draw(canvas);
            canvas.translate(-scrollX, -scrollY);
        }
    }
```
我们来看一下设置背景的边界的函数，可以看到，这里没有考虑`padding`值，也就是说我们通过`background`设置的图片是填满整个控件，并且不考虑`padding`的：
```
    void setBackgroundBounds() {
        if (mBackgroundSizeChanged && mBackground != null) {
            //没有考虑padding部分.
            mBackground.setBounds(0, 0, mRight - mLeft, mBottom - mTop);
            mBackgroundSizeChanged = false;
            rebuildOutline();
        }
    }
```
最后再结合一下第四节的知识，我们是先绘制背景，然后才在`ImageView`的`onDraw`函数当中在`canvas`上绘制的，因此，`src`的图片一定会绘制在`backgroud`之上。
