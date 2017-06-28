---
title: Canvas&Paint 知识梳理(2) - Canvas 的保存和恢复
date: 2017-03-02 20:32
categories : Canvas&Paint 知识梳理
---
# 一、和`Canvas`保存相关的标志
在了解`Canvas`的保存之前，我们先看一下和保存相关的标志的定义，它们定义了保存的类型，这些标志定义在`Canvas.java`当中，一共有六个标志。
```
/**
     * Restore the current matrix when restore() is called.
     */
    public static final int MATRIX_SAVE_FLAG = 0x01;

    /**
     * Restore the current clip when restore() is called.
     */
    public static final int CLIP_SAVE_FLAG = 0x02;

    /**
     * The layer requires a per-pixel alpha channel.
     */
    public static final int HAS_ALPHA_LAYER_SAVE_FLAG = 0x04;

    /**
     * The layer requires full 8-bit precision for each color channel.
     */
    public static final int FULL_COLOR_LAYER_SAVE_FLAG = 0x08;

    /**
     * Clip drawing to the bounds of the offscreen layer, omit at your own peril.
     * <p class="note"><strong>Note:</strong> it is strongly recommended to not
     * omit this flag for any call to <code>saveLayer()</code> and
     * <code>saveLayerAlpha()</code> variants. Not passing this flag generally
     * triggers extremely poor performance with hardware accelerated rendering.
     */
    public static final int CLIP_TO_LAYER_SAVE_FLAG = 0x10;

    /**
     * Restore everything when restore() is called (standard save flags).
     * <p class="note"><strong>Note:</strong> for performance reasons, it is
     * strongly recommended to pass this - the complete set of flags - to any
     * call to <code>saveLayer()</code> and <code>saveLayerAlpha()</code>
     * variants.
     */
    public static final int ALL_SAVE_FLAG = 0x1F;
```
从上面的定义可以看出，`flag`是用一个`32`位的`int`型变量来定义的，它的低`5`位的每一位用来表示需要保存`Canvas`当前哪部分的信息，如果全部打开，那么就是全部保存，也就是最后定义的`ALL_SAVE_FLAG`，这`5`位分别对应：
- `xxxx1`：保存`Matrix`信息，例如平移、旋转、缩放、倾斜等。
- `xxx1x`：保存`Clip`信息，也就是裁剪。
- `xx1xx`：保存`Alpha`信息。
- `x1xxx`：保存`8`位的颜色信息。
- `1xxxx`：`Clip drawing to the bounds of the offscreen layer`，不太明白是什么意思。

如果需要多选以上的几个信息进行保存，那么对多个标志位执行或操作即可。
# 二、`save()`和`save(int saveFlags)`
下面是这两个方法的定义：
```
    /**
     * Saves the current matrix and clip onto a private stack.
     * <p>
     * Subsequent calls to translate,scale,rotate,skew,concat or clipRect,
     * clipPath will all operate as usual, but when the balancing call to
     * restore() is made, those calls will be forgotten, and the settings that
     * existed before the save() will be reinstated.
     *
     * @return The value to pass to restoreToCount() to balance this save()
     */
    public int save() {
        return native_save(mNativeCanvasWrapper, MATRIX_SAVE_FLAG | CLIP_SAVE_FLAG);
    }

    /**
     * Based on saveFlags, can save the current matrix and clip onto a private
     * stack.
     * <p class="note"><strong>Note:</strong> if possible, use the
     * parameter-less save(). It is simpler and faster than individually
     * disabling the saving of matrix or clip with this method.
     *
     * @param saveFlags flag bits that specify which parts of the Canvas state
     *                  to save/restore
     * @return The value to pass to restoreToCount() to balance this save()
     */
    public int save(@Saveflags int saveFlags) {
        return native_save(mNativeCanvasWrapper, saveFlags);
    }
```
注释已经很好地说明了`save()`和`save(int saveFlags)`的作用：当调用完`save`方法之后，例如平移、缩放、旋转、倾斜、拼接或者裁剪这些操作，都是和原来的一样，而当调用完`restore`方法之后，在`save()`到`restore()`之间的所有操作都会被遗忘，并且会恢复调用`save()`之前的所有设置。此外还可以获得以下信息：
- 这两个方法最终都调用`native_save`方法，而无参方法`save()`默认是保存`Matrix`和`Clip`这两个信息。
- 如果允许，那么尽量使用无参的`save()`方法，而不是使用有参的`save(int saveFlags)`方法传入别的`Flag`。
- 该方法的返回值，对应的是在堆栈中的`index`，之后可以在`restoreToCount(int saveCount)`中传入它来讲在它之上的所有保存图层都出栈。
- 所有的操作都是调用了`native_save`来对这个`mNativeCanvasWrapper`变量，我们会发现，所有对于`Canvas`的操作，其实最终都是操作了`mNativeCanvasWrapper`这个对象。
- 从`XXX_SAVE_FLAG`的命名来看，带有参数的`save(int saveFlags)`方法只允许保存`MATRIX_/CLIP_/ALL_`这三种状态，而`HAS_ALPHA_LAYER/FULL_COLOR_LAYER_/CLIP_TO_LAYER_`这三种状态，则是为后面的`saveLayer/saveLayerAlpha`提供的。

# 三、`restore()` `restoreToCount(int count)` `getSaveCount()`
这三个方法用来恢复图层信息，也就是将之前保存到栈中的元素出栈，我们看一下这几个方法的定义：
```
    /**
     * This call balances a previous call to save(), and is used to remove all
     * modifications to the matrix/clip state since the last save call. It is
     * an error to call restore() more times than save() was called.
     */
    public void restore() {
        boolean throwOnUnderflow = !sCompatibilityRestore || !isHardwareAccelerated();
        native_restore(mNativeCanvasWrapper, throwOnUnderflow);
    }

    /**
     * Returns the number of matrix/clip states on the Canvas' private stack.
     * This will equal # save() calls - # restore() calls.
     */
    public int getSaveCount() {
        return native_getSaveCount(mNativeCanvasWrapper);
    }

    /**
     * Efficient way to pop any calls to save() that happened after the save
     * count reached saveCount. It is an error for saveCount to be less than 1.
     *
     * Example:
     *    int count = canvas.save();
     *    ... // more calls potentially to save()
     *    canvas.restoreToCount(count);
     *    // now the canvas is back in the same state it was before the initial
     *    // call to save().
     *
     * @param saveCount The save level to restore to.
     */
    public void restoreToCount(int saveCount) {
        boolean throwOnUnderflow = !sCompatibilityRestore || !isHardwareAccelerated();
        native_restoreToCount(mNativeCanvasWrapper, saveCount, throwOnUnderflow);
    }
```
这个几个方法很好理解：
- `restore()`方法用来恢复，**最近一次调用`save()`之前**的`Matrix/Clip`信息，如果调用`restore()`的次数大于`save()`的一次，也就是栈中已经没有元素，那么会抛出异常。
- `getSaveCount()`：返回的是当前栈中元素的数量。
- `restoreToCount(int count)`会将`saveCount()`之上对应的所有元素都出栈，如果`count < 1`，那么会抛出异常。
- 它们最终都是调用了`native`的方法。

# 四、示例
下面，我们用一个简单的例子，来更加直观的了解一下`save()`和`restore()`，我们重写了一个`View`当中的`onDraw()`方法：
## 4.1 恢复`Matrix`信息
```
    private void saveMatrix(Canvas canvas) {
        //绘制蓝色矩形
        mPaint.setColor(getResources().getColor(android.R.color.holo_blue_light));
        canvas.drawRect(0, 0, 200, 200, mPaint);
        //保存
        canvas.save();
        //裁剪画布,并绘制红色矩形
        mPaint.setColor(getResources().getColor(android.R.color.holo_red_light));
        //平移.
        //canvas.translate(100, 0);
        //缩放.
        //canvas.scale(0.5f, 0.5f);
        //旋转
        //canvas.rotate(-45);
        //倾斜
        canvas.skew(0, 0.5f);
        canvas.drawRect(0, 0, 200, 200, mPaint);
        //恢复画布
        canvas.restore();
        //绘制绿色矩形
        mPaint.setColor(getResources().getColor(android.R.color.holo_green_light));
        canvas.drawRect(0, 0, 50, 200, mPaint);
    }
```
我们对画布分别进行了平移、缩放、旋转、倾斜，得到的结果为：
![Translate.png](http://upload-images.jianshu.io/upload_images/1949836-1ddcdd379c9f4b32.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![Scale.png](http://upload-images.jianshu.io/upload_images/1949836-ff423112dd74c956.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![Rotate.png](http://upload-images.jianshu.io/upload_images/1949836-c454013087584528.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![Skew.png](http://upload-images.jianshu.io/upload_images/1949836-5cc76b1323ee3c92.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

可以看到，当我们调用上面这些方法时，其实就是对`Canvas`先进行了一些移动，旋转，缩放的操作，然后再在它这个新的状态上进行绘制，之后调用`restore()`之后，又恢复回了调用`save()`之前的状态。

## 4.2 恢复`Clip`信息
```
    private void saveClip(Canvas canvas) {
        //绘制蓝色矩形
        mPaint.setColor(getResources().getColor(android.R.color.holo_blue_light));
        canvas.drawRect(0, 0, 200, 200, mPaint);
        //保存.
        canvas.save();
        //裁剪画布,并绘制红色矩形
        mPaint.setColor(getResources().getColor(android.R.color.holo_red_light));
        canvas.clipRect(150, 0, 200, 200);
        canvas.drawRect(0, 0, 200, 200, mPaint);
        //恢复画布
        canvas.restore();
        //绘制绿色矩形
        mPaint.setColor(getResources().getColor(android.R.color.holo_green_light));
        canvas.drawRect(0, 0, 50, 200, mPaint);
    }
```
最终的结果如下所示：
![](http://upload-images.jianshu.io/upload_images/1949836-79ffc093c7ee2997.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
- 初始的时候，`canvas`的大小和`View`的大小是一样的，我们以`(0,0)`为原点坐标，绘制了一个大小为`200px * 200px`的蓝色矩形。
- 我们保存画布的当前信息，之后以`(150, 0)`为原点坐标，裁剪成了一个大小为`50px * 200px`的新画布，对于这个裁剪的过程可以这么理解：就是我们以前上学时候用的那些带镂空的板子，上面有各种的形状，而这一裁剪，其实就是在原来的`canvas`上盖了这么一个镂空的板子，镂空部分就是我们定义的裁剪区域，当我们进行绘制时，就是在这个板子上面进行绘制，而最终**在`canvas`上展现的部分就是这些镂空部分和之后绘制部分的交集**。
- 之后我们尝试以`(0, 0)`为原点绘制一个大小为`200px * 200px`的红色矩形，但是此时由于`(0, 0) - (150, 200)`这部分被盖住了，所以不我们画不上去，实际画上去的只有`(50, 0) - (200, 200)`这一部分。
- 之后调用了`restore()`方法，就相当于把板子拿掉，那么这时候就能够像之前那样正常的绘制了。

## 五、`saveLayer` `saveLayerAlpha`
## 5.1 方法定义
除了`save()`方法之外，`canvas`还提供了`saveLayer`方法

![](http://upload-images.jianshu.io/upload_images/1949836-2ff9aa1b88fc4cd2.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
以上的这八个方法可以分为两个大类：`saveLayer`和`saveLayerAlpha`，它们最终都是调用了两个`native`方法：
对于`saveLayer`，如果不传入`saveFlag`，那么默认是采用`ALL_SAVE_FLAG`：
```
native_saveLayer(mNativeCanvasWrapper, left, top, right, bottom, paint != null ? paint.getNativeInstance() : 0, saveFlags);
```
对于`saveLayerAlpha`，如果不传入`saveFlag`，那么默认是采用`ALL_SAVE_FLAG`，如果不传入`alpha`，那么最终调用的`alpha = 0`。
```
native_saveLayerAlpha(mNativeCanvasWrapper, left, top, right, bottom, alpha, saveFlags);
```
## 5.2 和`save()`方法的区别
关于`save()`和`saveLayer()`的区别，源码当中是这么解释的，也就是说它**会新建一个不在屏幕之内的`bitmap`**，之后的所有绘制都是在这个`bitmap`上操作的。
>  `This behaves the same as save(), but in addition it allocates and redirects drawing to an offscreen bitmap.`

并且这个方法是相当消耗资源的，因为它会导致**内容的二次渲染**，特别是当`canvas`的边界很大或者使用了`CLIP_TO_LAYER`这个标志时，更推荐使用`LAYER_TYPE_HARDWARE`，也就是硬件渲染来进行`Xfermode`或者`ColorFilter`的操作，它会更加高效。
```
     * this method is very expensive,
     *
     * incurring more than double rendering cost for contained content. Avoid
     * using this method, especially if the bounds provided are large, or if
     * the {@link #CLIP_TO_LAYER_SAVE_FLAG} is omitted from the
     * {@code saveFlags} parameter. It is recommended to use a
     * {@link android.view.View#LAYER_TYPE_HARDWARE hardware layer} on a View
     * to apply an xfermode, color filter, or alpha, as it will perform much
     * better than this method.
```
当我们在之后调用`restore()`方法之后，那么这个新建的`bitmap`会绘制回`Canvas`的当前目标，如果当前就位于`canvas`的最底层图层，那么就是目标屏幕，否则就是之前的图层。
```
     * All drawing calls are directed to a newly allocated offscreen bitmap.
     * Only when the balancing call to restore() is made, is that offscreen
     * buffer drawn back to the current target of the Canvas (either the
     * screen, it's target Bitmap, or the previous layer).
```
再回头看下方法的参数，这两大类方法分别传入了`Paint`和`Alpha`这两个变量，对于`saveLayer`来说，`Paint`的下面这三个属性会在新生成的`bitmap`被重新绘制到当前画布时，也就是调用了`restore()`方法之后，被采用：
```
     * Attributes of the Paint - {@link Paint#getAlpha() alpha},
     * {@link Paint#getXfermode() Xfermode}, and
     * {@link Paint#getColorFilter() ColorFilter} are applied when the
     * offscreen bitmap is drawn back when restore() is called.
```
而对于`saveLayerAlpha`来说，它的`Alpha`则会在被重新绘制回来时被采用：
```
     * The {@code alpha} parameter is applied when the offscreen bitmap is
     * drawn back when restore() is called.
```
对于这两个方法，都推荐传入`ALL_SAVE_FLAG`来提高性能，它们的返回值和`save()`方法的含义是相同的，都是用来提供给`restoreToCount(int count)`使用。
总结一下：就是调用`saveLayer`之后，创建了一个透明的图层，之后在调用`restore()`方法之前，我们都是在这个图层上面进行操作，而`save`方法则是直接在原先的图层上面操作，那么对于某些操作，我们不希望原来图层的状态影响到它，那么我们应该使用`saveLayer`。

# 六、`saveLayer`示例
和前面类似，我们创建一个继承于`View`的`SaveLayerView`，并重写`onDraw(Canvas canvas)`方法：
## 6.1 验证创建新的图层理论
首先，因为我们前面整个一整节得到的结论是`saveLayer`会创建一个新的图层，而验证是否产生新图层的方式就是采用`Paint#setXfermode()`方法，通过它和下面图层的结合关系，我们就能知道是否生成了一个新的图层了。当使用`saveLayer`时：
```
    @Override
    protected void onDraw(Canvas canvas) {
        useSaveLayer(canvas);
    }

    private void useSaveLayer(Canvas canvas) {
        //1.先画一个蓝色圆形.
        canvas.drawCircle(mRadius, mRadius, mRadius, mBlueP);
        //canvas.save();
        //2.这里产生一个新的图层
        canvas.saveLayer(0, 0, mRadius + mRadius, mRadius + mRadius, null);
        //3.现先在该图层上画一个绿色矩形
        canvas.drawRect(mRadius, mRadius, mRadius + mRadius, mRadius + mRadius, mGreenP);
        //4.设为取下面的部分
        mRedP.setXfermode(mDstOverXfermode);
        //5.再画一个红色圆形,如果和下面的图层有交集,那么取下面部分
        canvas.drawCircle(mRadius, mRadius, mRadius/2, mRedP);
    }
```
当我们使用`saveLayer()`方法时，得到的是：
![](http://upload-images.jianshu.io/upload_images/1949836-e8180fb136eab4f2.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
而当我们使用`save()`方法，得到的则是：
![](http://upload-images.jianshu.io/upload_images/1949836-6a326876c5ff96af.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
只所以产生不同的结果，是因为第4步的时候，我们给红色画笔设置了`DST_OVER`模式，也就是底下部分和即将画上去的部分有重合的时候，取底下部分。当我们在第2步当中使用`saveLayer`的时候，按我们的假设，会产生一个新的图层，那么第3步的绿色矩形就是画在这个新的透明图层上的，因此第5步画红色圆形的时候，`DST`是按绿色矩形部分来算的，重叠部分只占了红色圆形的`1/4`，因此最后画出来的结果跟第一张图一样。
而不使用`saveLayer`时，由于没有产生新的图层，因此在第5步绘制的时候，`DST`其实是由蓝色圆形和绿色矩形组成的，这时候和红色圆形的重叠部分占了整个红色圆形，所以最后画上去的时候就看不到了。
这就很好地验证了`saveLayer`会创建一个新的图层。

## 6.2 `saveLayerAlpha`
下面，我们再来看一下`saveLayerAlpha`，这个方法可以用来产生一个带有透明度的图层：
```
    private void useSaveLayerAlpha(Canvas canvas) {
        //先划一个蓝色的圆形.
        canvas.drawCircle(mRadius, mRadius, mRadius, mBlueP);
        //canvas.save();
        //这里产生一个新的图层
        canvas.saveLayerAlpha(0, 0, mRadius + mRadius, mRadius + mRadius, 128);
        //现先在该图层上画一个矩形
        canvas.drawRect(mRadius, mRadius, mRadius + mRadius, mRadius + mRadius, mGreenP);
    }
```
最终，我们就得到了下面的带有透明度的绿色矩形覆盖在上面：
![](http://upload-images.jianshu.io/upload_images/1949836-ae63207ec39eaa0f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
## 6.3 `HAS_ALPHA_LAYER_XXX`和`FULL_COLOR_LAYER_XXX`
`HAS_ALPHA_LAYER`表示图层结合的时候，没有绘制的地方会是透明的，而对于`FULL_COLOR_LAYER_XXX`，则会强制覆盖掉。
首先，我们先看一下整个布局为一个黑色背景的`Activity`，里面有一个背景为绿色的自定义`View`
```
<?xml version="1.0" encoding="utf-8"?>
<RelativeLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:background="@android:color/black"
    tools:context="com.example.lizejun.repocanvaslearn.MainActivity">
    <com.example.lizejun.repocanvaslearn.SaveLayerView
        android:background="@android:color/holo_green_light"
        android:layout_width="200dp"
        android:layout_height="200dp" />
</RelativeLayout>
```
下面，我们重写`onDraw(Canvas canvas)`方法：
```
    private void useSaveLayerHasAlphaOrFullColor(Canvas canvas) {
        //先划一个蓝色的圆形.
        canvas.drawRect(0, 0, mRadius * 2, mRadius * 2, mBlueP);
        //这里产生一个新的图层
        canvas.saveLayer(0, 0, mRadius, mRadius, null, Canvas.FULL_COLOR_LAYER_SAVE_FLAG);
        //canvas.saveLayer(0, 0, mRadius, mRadius, null, Canvas.HAS_ALPHA_LAYER_SAVE_FLAG);
        //绘制一个红色矩形.
        canvas.drawRect(0, 0, mRadius / 2, mRadius / 2, mRedP);
    }
```
当采用`FULL_COLOR_LAYER_SAVE_FLAG`时，对应的结果为下图：
![](http://upload-images.jianshu.io/upload_images/1949836-d8bf1c8d6d0bcb70.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
而采用`HAS_ALPHA_LAYER_SAVE_FLAG`时，对应的结果为：
![](http://upload-images.jianshu.io/upload_images/1949836-45b6c85af25c3937.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
可以看到，当使用`FULL_COLOR_LAYER_SAVE_FLAG`，不仅下一层原本绘制的蓝色没有用，连`View`本身的绿色背景也没有了，最后透上来的是`Activity`的黑色背景。
关于这个方法，有几个需要注意的地方：
- 需要在`View`中禁用硬件加速。
```
setLayerType(LAYER_TYPE_SOFTWARE, null);
```
- 当两个共用时，以`FULL_COLOR_LAYER_SAVE_FLAG`为准。
- 当调用`saveLayer`并且只指定`MATRIX_SAVE_FLAG`或者`CLIP_SAVE_FLAG`时，默认的合成方式是`FULL_COLOR_LAYER_SAVE_FLAG`。

## 6.3 `CLIP_TO_LAYER_SAVE_FLAG`
它在新建`bitmap`前，先把`canvas`给裁剪，一旦画板被裁剪，那么其中的各个画布就会被受到影响，并且它是无法恢复的。当其和`CLIP_SAVE_FLAG`共用时，是可以被恢复的。

# 6.4 `ALL_SAVE_FLAG`
对于`save()`来说，它相当于`MATRIX_SAVE_FLAG | CLIP_SAVE_FLAG`。
对于`saveLayer()`来说，它相当于`MATRIX_SAVE_FLAG | CLIP_SAVE_FLAG|HAS_ALPHA_LAYER_SAVE_FLAG`。

# 七、参考文献
[`1.http://blog.csdn.net/cquwentao/article/details/51423371`](http://blog.csdn.net/cquwentao/article/details/51423371)
