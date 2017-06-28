---
title: Canvas&Paint 知识梳理(4) - 图像合成 Paint#setXfermode
date: 2017-03-03 21:15
categories : Canvas&Paint 知识梳理
---
# 一、概述
在**颜色合成**文章中的最后一个小结当中，我们已经见到了`PorterDuff.Mode`这个枚举类，在本次的**图像合成**中，我们也需要用到这个类，我们先看一下最终调用的方法为：
```
    /**
     * Set or clear the xfermode object.
     * <p />
     * Pass null to clear any previous xfermode.
     * As a convenience, the parameter passed is also returned.
     *
     * @param xfermode May be null. The xfermode to be installed in the paint
     * @return         xfermode
     */
    public Xfermode setXfermode(Xfermode xfermode) {
        long xfermodeNative = 0;
        if (xfermode != null)
            xfermodeNative = xfermode.native_instance;
        native_setXfermode(mNativePaint, xfermodeNative);
        mXfermode = xfermode;
        return xfermode;
    }
```
当一个`Paint`被设置了某个`Xfermode`时，那么会根据源图层、画笔和`Mode`，来决定画完之后的图像到底是什么，在使用的时候，我们一般采用`PorterDuffXfermode`作为`Xfermode`的实现类，它的构造函数的参数就是我们之前说到的`PoterDuff.Mode`中的某个类型。

# 二、混合方式
关于混合的方式，网上有张图是这么总结的，其中`DST`表示原本有的图像，而`SRC`表示即将绘制上去的图像：
![](http://upload-images.jianshu.io/upload_images/1949836-3f6eec90f2824a9a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
在某些我们以为的情况下，并不能得到对应的结果，感谢下面这篇博客的作者：
[`http://blog.csdn.net/u010335298/article/details/51983420`](http://blog.csdn.net/u010335298/article/details/51983420)
他总结了获得图中的结果，上面的图中还有几个隐含的条件：
- 关闭硬件加速。
- 两个进行叠加的图层的大小是相同的。
- 除了有颜色的部分，其它部分都是透明的。



# 三、示例
下面我们来看一下，`DST_ATOP`这种方式
## 3.1 开启硬件加速
```
    private void drawPorterDuffXferMode(Canvas canvas) {
        Paint paint = new Paint();
        //绘制DST图像.
        paint.setColor(Color.YELLOW);
        canvas.drawCircle(100, 100, 100, paint);
        //绘制SRC图像.
        paint.setColor(Color.BLUE);
        paint.setXfermode(new PorterDuffXfermode(PorterDuff.Mode.DST_ATOP));
        canvas.drawRect(100, 100, 300, 300, paint);
    }
```
对应的结果为：

![](http://upload-images.jianshu.io/upload_images/1949836-4c40083055d8cf5c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
和效果图完全不符，下面，我们试一下关闭硬件加速：
## 3.2 关闭硬件加速
```
    private void init() {
        setLayerType(LAYER_TYPE_SOFTWARE, null);
    }

    private void drawPorterDuffXferMode(Canvas canvas) {
        Paint paint = new Paint();
        //绘制DST图像.
        paint.setColor(Color.YELLOW);
        canvas.drawCircle(100, 100, 100, paint);
        //绘制SRC图像.
        paint.setColor(Color.BLUE);
        paint.setXfermode(new PorterDuffXfermode(PorterDuff.Mode.DST_ATOP));
        canvas.drawRect(100, 100, 300, 300, paint);
    }
```
结果为：
![](http://upload-images.jianshu.io/upload_images/1949836-95268923d7bc14a1.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
还是和理论结果不符合。
## 3.3 使用两个大小一样的`Bitmap`
```
    private Paint mDstPaint;
    private Paint mSrcPaint;
    private Canvas mDstCanvas;
    private Canvas mSrcCanvas;
    private Bitmap mSrcBitmap;
    private Bitmap mDstBitmap;
    
    private void init() {
        setLayerType(LAYER_TYPE_SOFTWARE, null);
        mDstPaint = new Paint();
        mSrcPaint = new Paint();
        mDstPaint.setColor(Color.YELLOW);
        mSrcPaint.setColor(Color.BLUE);
        mDstBitmap = Bitmap.createBitmap(300, 300, Bitmap.Config.ARGB_8888);
        mSrcBitmap = Bitmap.createBitmap(300, 300, Bitmap.Config.ARGB_8888);
        mDstCanvas = new Canvas(mDstBitmap);
        mSrcCanvas = new Canvas(mSrcBitmap);
    }

    private void drawPorterDuffXferMode(Canvas canvas) {
        //绘制DST图像.
        mDstCanvas.drawCircle(100, 100, 100, mDstPaint);
        canvas.drawBitmap(mDstBitmap, 0, 0, mDstPaint);
        //绘制SRC图像
        mSrcCanvas.drawRect(100, 100, 300, 300, mSrcPaint);
        mSrcPaint.setXfermode(new PorterDuffXfermode(PorterDuff.Mode.DST_ATOP));
        canvas.drawBitmap(mSrcBitmap, 0, 0, mSrcPaint);
    }
```
来看看这个的结果：
![](http://upload-images.jianshu.io/upload_images/1949836-b37a898886c261ef.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
为了得到这个结果，实例化了一堆的对象，并且需要在`Bitmap`的大小一样的时候才可以生效，其实这也不能说是坑，因为根据源码来看，计算的计算本来就是取各像素点的`ARGB`进行计算，如果图层的大小不一样，那么计算的结果自然就和上面不同，从图中来看，它也表明了`DST`和`SRC`的大小是相同的，并且在除了有颜色之外的部分都是透明的。
# 四、参考文献
[`1.http://blog.csdn.net/u010335298/article/details/51983420`](http://blog.csdn.net/u010335298/article/details/51983420)
