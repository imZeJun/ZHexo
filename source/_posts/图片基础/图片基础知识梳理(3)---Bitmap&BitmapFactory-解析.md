---
title: 图片基础知识梳理(3) - Bitmap&BitmapFactory 解析
date: 2017-03-22 12:21
categories : 图片基础知识梳理
---
# 一、概述
今天这篇文章我们来了解一下两个类：
- `Bitmap`
- `BitmapFactory`

# 二、`Bitmap`
## 2.1 创建`Bitmap`
通过`Bitmap`的源码，我们可以看到它内部提供了很多`.createBitmap(xxx)`的静态方法，我们可以通过这些方法来获得一个`Bitmap`：
![](http://upload-images.jianshu.io/upload_images/1949836-abdf20b136b22a7d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
上述的方法最终可以分为以下三类：
- 通过一个已有的`Bitmap`创建
- 创建一个空的`Bitmap`
- 创建一个新的`Bitmap`，该`Bitmap`每个像素点的颜色通过一个`colors[]`数组指定。

下面，我们来看一下这三类方法对于`Bitmap`的生产过程：

### 第一类
```
    public static Bitmap createBitmap(Bitmap source, int x, int y, int width, int height,
            Matrix m, boolean filter) {

        checkXYSign(x, y);
        checkWidthHeight(width, height);
        //新的bitmap范围不能大于原始的bitmap
        if (x + width > source.getWidth()) {
            throw new IllegalArgumentException("x + width must be <= bitmap.width()");
        }
        if (y + height > source.getHeight()) {
            throw new IllegalArgumentException("y + height must be <= bitmap.height()");
        }

        //如果满足下面这些条件，那么直接返回原始的bitmap
        if (!source.isMutable() && x == 0 && y == 0 && width == source.getWidth() &&
                height == source.getHeight() && (m == null || m.isIdentity())) {
            return source;
        }

        int neww = width;
        int newh = height;
        Canvas canvas = new Canvas();
        Bitmap bitmap;
        Paint paint;
        //生成bitmap对应区域
        Rect srcR = new Rect(x, y, x + width, y + height);
        //原始bitmap对应区域
        RectF dstR = new RectF(0, 0, width, height);

        Config newConfig = Config.ARGB_8888;
        //获得原始bitmap的config
        final Config config = source.getConfig();
        // GIF files generate null configs, assume ARGB_8888
        if (config != null) {
            switch (config) {
                case RGB_565:
                    newConfig = Config.RGB_565;
                    break;
                case ALPHA_8:
                    newConfig = Config.ALPHA_8;
                    break;
                //noinspection deprecation
                case ARGB_4444:
                case ARGB_8888:
                default:
                    newConfig = Config.ARGB_8888;
                    break;
            }
        }
        //如果不需要变换，那么创建一个空的bitmap.
        if (m == null || m.isIdentity()) {
            bitmap = createBitmap(neww, newh, newConfig, source.hasAlpha());
            paint = null;   // not needed
        } else {
            //根据Matrix，对原始的bitmap进行一些变换操作.
            final boolean transformed = !m.rectStaysRect();

            RectF deviceR = new RectF();
            m.mapRect(deviceR, dstR);

            neww = Math.round(deviceR.width());
            newh = Math.round(deviceR.height());

            bitmap = createBitmap(neww, newh, transformed ? Config.ARGB_8888 : newConfig,
                    transformed || source.hasAlpha());

            canvas.translate(-deviceR.left, -deviceR.top);
            canvas.concat(m);

            paint = new Paint();
            paint.setFilterBitmap(filter);
            if (transformed) {
                paint.setAntiAlias(true);
            }
        }
        //返回bitmap的这些属性和原始bitmap相同
        bitmap.mDensity = source.mDensity;
        bitmap.setHasAlpha(source.hasAlpha());
        bitmap.setPremultiplied(source.mRequestPremultiplied);

        //设置canvas对应的bitmap为返回的bitmap
        canvas.setBitmap(bitmap);

        //通过canvas把原始的bitmap绘制上去.
        canvas.drawBitmap(source, srcR, dstR, paint);

        //重新置为空.
        canvas.setBitmap(null);
        return bitmap;
    }
```
- 方法作用：返回原始的`Bitmap`中一个不可改变的子集，返回的`Bitmap`有可能是原始的`Bitmap`（原始的`Bitmap`不可改变，并且大小和请求的新的`Bitmap`大小和原来一样），也有可能是复制出来的，它和原始的`Bitmap`的`density`相同。
- 参数说明：
 - `source`：原始的`Bitmap`
 - `x, y`：在原始的`Bitmap`中的起始坐标。
 - `width, height`：返回的`Bitmap`的宽高，如果超过了原始`Bitmap`的范围，那么会抛出异常。
 - `m`：`Matrix`类型，表示需要的变换
 - `filter`：是否需要优化，只有当`m`不只有平移操作时才去进行。

### 第二类
```
    private static Bitmap createBitmap(DisplayMetrics display, int width, int height, Config config, boolean hasAlpha) {
        if (width <= 0 || height <= 0) {
            throw new IllegalArgumentException("width and height must be > 0");
        }
        Bitmap bm = nativeCreate(null, 0, width, width, height, config.nativeInt, true);
        if (display != null) {
            bm.mDensity = display.densityDpi;
        }
        bm.setHasAlpha(hasAlpha);
        if (config == Config.ARGB_8888 && !hasAlpha) {
            nativeErase(bm.mNativePtr, 0xff000000);
        }
        return bm;
    }
```
- 方法作用：返回一个可变的`bitmap`，它的`density`由传入的`DisplayMetrics`指定。
- 参数说明：
 - `display`：`Bitmap`将要被绘制的`Display metrics`
 - `width, height`：`bitmap`的宽高
 - `config`：配置信息，对应`ARGB_8888`那些。
 - `hasAlpha`：如果`bitmap`的属性是`ARGB_8888`，那么这个标志为可以用来把`bitmap`标志为透明，它会把`bitmap`中的黑色像素转换为透明。

### 第三类
```
    public static Bitmap createBitmap(DisplayMetrics display, int colors[],
            int offset, int stride, int width, int height, Config config) {

        checkWidthHeight(width, height);
        if (Math.abs(stride) < width) {
            throw new IllegalArgumentException("abs(stride) must be >= width");
        }
        int lastScanline = offset + (height - 1) * stride;
        int length = colors.length;
        if (offset < 0 || (offset + width > length) || lastScanline < 0 ||
                (lastScanline + width > length)) {
            throw new ArrayIndexOutOfBoundsException();
        }
        if (width <= 0 || height <= 0) {
            throw new IllegalArgumentException("width and height must be > 0");
        }
        Bitmap bm = nativeCreate(colors, offset, stride, width, height,
                            config.nativeInt, false);
        if (display != null) {
            bm.mDensity = display.densityDpi;
        }
        return bm;
    }
```
- 方法作用：返回一个不可变的`bitmap`对象，它的长宽由`width/height`指定，每个像素点的颜色通过`colos[]`数组得到，初始的`density`来自于`DisplayMetrics`。
- 方法参数：
 - `display`：`Bitmap`将要被绘制的`Display metrics`
 - `colors`：用来初始化像素点的颜色
 - `offset`：第一个像素点的颜色在数组当中跳过的个数。
 - `stride`：两行之间需要跳过的颜色个数。
 - `width/height`：宽高。
 - `config`：对应`ARGB_8888`那些。

## 2.2 压缩`bitmap`
```
    public boolean compress(CompressFormat format, int quality, OutputStream stream) {
        checkRecycled("Can't compress a recycled bitmap");
        // do explicit check before calling the native method
        if (stream == null) {
            throw new NullPointerException();
        }
        if (quality < 0 || quality > 100) {
            throw new IllegalArgumentException("quality must be 0..100");
        }
        Trace.traceBegin(Trace.TRACE_TAG_RESOURCES, "Bitmap.compress");
        boolean result = nativeCompress(mNativePtr, format.nativeInt,
                quality, stream, new byte[WORKING_COMPRESS_STORAGE]);
        Trace.traceEnd(Trace.TRACE_TAG_RESOURCES);
        return result;
    }
```
- 方法作用：把当前这个`bitmap`的压缩版本写入到某个输出流当中，如果返回`true`，那么这个`bitmap`可以被`BitmapFactory.decodeStream()`恢复。需要注意的是，并不是所有的`bitmap`都支持所有的格式，因此，通过`BitmapFactory`恢复回来的`bitmap`有可能和原来不同。
- 方法参数：
 - `CompressFormat`是一个枚举类型，它的值有`JPEG/PNG/WEBP`
 - `quality`对应`0-100`
 - `stream`则是压缩后结果的输出流。

## 2.3 回收`bitmap`
```
    public void recycle() {
        if (!mRecycled && mNativePtr != 0) {
            if (nativeRecycle(mNativePtr)) {
                // return value indicates whether native pixel object was actually recycled.
                // false indicates that it is still in use at the native level and these
                // objects should not be collected now. They will be collected later when the
                // Bitmap itself is collected.
                mBuffer = null;
                mNinePatchChunk = null;
            }
            mRecycled = true;
        }
    }
```
`recycle`方法主要做几件事：
- 释放和这个`bitmap`关联的`native`对象
- 清除像素数据`mBuffer`的引用，但是这一过程不是同步的，它只是将引用置为空，等待垃圾回收器将它回收。
- 在调用这个方法之后，`mRecycled`标志位就为`true`，之后如果再调用`bitmap`的方法，那么很有可能发生异常。
- 一般情况下，我们不需要手动调用这个方法，因为当这个`bitmap`不被引用时，垃圾回收器就会自动回收它所占用的内存。

## 2.4 获取`Bitmap`所占内存
- `getAllocationByteCount()`
返回存储这个`bitmap`对象所需要的内存，当我们对`bitmap`所占内存区域进行复用的时候，这个函数的返回结果可能要大于`getByteCount`的值，否则，它和`getByteCount`的值是相同的。
这个值，在`bitmap`整个生命周期之内都不会改变。
```
    public final int getAllocationByteCount() {
        if (mBuffer == null) {
            return getByteCount();
        }
        return mBuffer.length;
    }
```
- `getByteCount`
表示存储`bitmap`像素所需要的最小字节，自从`4.4`之后，这个就不能用来确定`bitmap`占用的内存了，需要用`getAllocationByteCount`。
```
    public final int getByteCount() {
        // int result permits bitmaps up to 46,340 x 46,340
        return getRowBytes() * getHeight();
    }
```

## 2.5 获取缩放后大小
![](http://upload-images.jianshu.io/upload_images/1949836-f766d631df86ece2.png)
我们可以通过上面这六个方法获得缩放后的宽高，它们的原理就是传入一个目标的`density`，然后和当前`bitmap`的`density`进行比较，然后算出一个缩放的倍数，在和原来的大小相乘。目标`density`的来源有以下三个：
- 直接传入
- `Canvas`的`density`
- `DisplayMetrics`的`density`

计算的规则为：
```
    static public int scaleFromDensity(int size, int sdensity, int tdensity) {
        if (sdensity == DENSITY_NONE || tdensity == DENSITY_NONE || sdensity == tdensity) {
            return size;
        }

        // Scale by tdensity / sdensity, rounding up.
        return ((size * tdensity) + (sdensity >> 1)) / sdensity;
    }
```

# 三、`BitmapFactory`
`BitmapFactory`用来从多种不同的来源获得`Bitmap`：
- 文件、文件描述符
- 资源文件`Resource`
- `byte[]`数组
- 输入流`InputStream`
![](http://upload-images.jianshu.io/upload_images/1949836-a1b39f27c4fcc38b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 3.1 `BitmapFactory.Options`类
- `Bitmap inBitmap`
如果给`Options`设置了这个`Bitmap`，那么在通过这个`Options`解码的时候，解码方法返回的`bitmap`会尝试复用这个`Options`中的`bitmap`，如果不能复用，那么解码方法会返回`null`，并抛出异常，**它要求复用的`bitmap`是可变的**。
在`4.4`以后，只要求新申请的`bitmap`的`getByteCount()`小于等于`Options`中的`bitmap`的`getAllocationByteCount()`就可以。
在`4.4`以前，格式必须是`jpeg/png`，并且要求两个`bitmap`相同并且`inSampleSize`为`1`。
- `boolean inJustDecodeBounds`
如果设为`true`，那么解码方法的返回值`null`，但是它会设置`outXXX`的值，这样调用者就可以在不用解码整张图片的前提下查询到这个`bitmap`的长宽。
- `int inSampleSize`
对原来的图片进行采样，如果`inSampleSize`为`4`，那么图片的长宽会缩短为原来的`1/4`，这样就可以减少`bitmap`占用的内存。
- `Bitmap.Config inPreferredConfig`
图片解码的格式要求。
- 缩放相关的标志：`inScaled`、`inDensity`、`inTargetDensity`、`inScreenDensity`
首先，只有在`inScaled`为`true`的时候，缩放的机制才会生效，这个值默认是`true`的。
 - `inDensity`
我们先讨论一下`inDensity`，当我们没有给`density`赋值的时候，系统会给我们初始化它：
```
        //如果没有设置density
        if (opts.inDensity == 0 && value != null) {
            final int density = value.density; //这里的TypeValue会根据存放文件夹的不同而不同.
            if (density == TypedValue.DENSITY_DEFAULT) {
                opts.inDensity = DisplayMetrics.DENSITY_DEFAULT; //如果density为0，那么把density设置为160.
            } else if (density != TypedValue.DENSITY_NONE) {
                opts.inDensity = density; //否则，设置为value中的density.
            }
        }
```
 - `inTargetDensity`
再来看一下`inTargetDensity`，它得到的就是屏幕的`density`.
```
        if (opts.inTargetDensity == 0 && res != null) {
            opts.inTargetDensity = res.getDisplayMetrics().densityDpi;
        }
```
 - `inScreenDensity`
最后`inScreenDensity`没有被赋予默认值，也就是说它为`0`，如果我们期望图片不要被缩放，那么就要给它设置为手机的`density`。

这三者的关系是：**当`inDensity`不为`0`并且`inTargetDensity`不为`0`，`inDensity`和`inScreenDensity`不相等时，会对图片进行缩放，缩放倍数为`inTargetDensity/inDensity`**。

这样说可能比较抽象，我们举一个实际的例子，假如我们的手机的`density`是`320dpi`的，那么`inTargetDensity`就等于`320`，这时候我们把某张图片资源放在了`drawable-xxxhpi`下，那么`inDensity`的值就为`640`，我们没有设置`inScreenDensity`，那么它的默认值是`0`，这时候满足：
```
inDensity != 0 && inTargetDensity != 0 && inDensity != inScreenDensity
```
图片就会进行缩放，缩放的倍数就为`320/640`，也就是说最终得到的`bitmap`的长宽是原来的一半。
- `outXXX`
这个返回的结果和`inJustDecodeBounds`有关，如果`inJustDecodeBounds`为`true`，那么返回的是没有经过缩放的大小，如果为`false`，那么就是缩放后的大小。

## 3.2 获取`bitmap`方法
下面是`BitmapFactory`提供的方法：
![](http://upload-images.jianshu.io/upload_images/1949836-7f21610c5d48520e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
所有的获取`bitmap`最终都是调用了一下四个方法`Native`方法其中之一，可以看到它可以从这些来源读取：
- `file`
- `byte[]`
- `InputStream`

![](http://upload-images.jianshu.io/upload_images/1949836-47be8e2da162dec1.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

其中有个需要注意的是`Rect`，这是一个传入的值，在读取资源完毕后，它会写入读取资源的`padding`，如果没有那么为`[-1, -1, -1,- 1]`，而如果返回的`bitmap`为空，那么传入的值不会改变。

# 四、`Bitmap`的转换方法
```
public class BitmapConvertUtils {

    public static Bitmap fromResourceIdAutoScale(Resources resources, int resourceId, BitmapFactory.Options options) {
        return BitmapFactory.decodeResource(resources, resourceId, options);
    }

    public static Bitmap fromResourceIdNotScale(Resources resources, int resourceId, Rect rect, BitmapFactory.Options options) {
        InputStream resourceStream = null;
        Bitmap bitmap = null;
        try {
            resourceStream = resources.openRawResource(resourceId);
            bitmap = BitmapFactory.decodeStream(resourceStream, rect, options);
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            try {
                if (resourceStream != null) {
                    resourceStream.close();
                }
            } catch (Exception e) {
                e.printStackTrace();
            }
        }
        return bitmap;
    }

    public static Bitmap fromAssert(Context context, String assertFilePath, Rect rect, BitmapFactory.Options options) {
        Bitmap bitmap = null;
        InputStream assertStream = null;
        AssetManager assetManager = context.getAssets();
        try {
            assertStream = assetManager.open(assertFilePath);
            bitmap = BitmapFactory.decodeStream(assertStream, rect, options);
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            try {
                if (assertStream != null) {
                    assertStream.close();
                }
            } catch (Exception e) {
                e.printStackTrace();
            }

        }
        return bitmap;
    }

    public static Bitmap fromByteArray(byte[] byteArray, int offset, int length, BitmapFactory.Options options) {
        return BitmapFactory.decodeByteArray(byteArray, offset, length, options);
    }

    public static Bitmap fromFile(String filePath, BitmapFactory.Options options) {
        return BitmapFactory.decodeFile(filePath, options);
    }

    public static Bitmap fromDrawable(Drawable drawable) {
        int width = drawable.getIntrinsicWidth();
        int height = drawable.getIntrinsicHeight();
        Bitmap bitmap = Bitmap.createBitmap(width, height, drawable.getOpacity() != PixelFormat.OPAQUE ? Bitmap.Config.ARGB_8888 : Bitmap.Config.RGB_565);
        if (bitmap != null) {
            Canvas canvas = new Canvas(bitmap);
            drawable.setBounds(0, 0, width, height);
            drawable.draw(canvas);
            return bitmap;
        }
        return null;
    }

    public static Bitmap fromView(View view) {
        view.clearFocus();
        view.setPressed(false);
        boolean willNotCache = view.willNotCacheDrawing();
        view.setWillNotCacheDrawing(false);
        int color = view.getDrawingCacheBackgroundColor();
        view.setDrawingCacheBackgroundColor(color);
        if (color != 0) {
            view.destroyDrawingCache();
        }
        view.buildDrawingCache();
        Bitmap cacheBitmap = view.getDrawingCache();
        if (cacheBitmap == null) {
            return null;
        }
        Bitmap bitmap = Bitmap.createBitmap(cacheBitmap);
        view.destroyDrawingCache();
        view.setWillNotCacheDrawing(willNotCache);
        view.setDrawingCacheBackgroundColor(color);
        return bitmap;
    }

    public static Bitmap fromInputStream(InputStream inputStream) {
        return BitmapFactory.decodeStream(inputStream);
    }

    public static byte[] toByteArray(Bitmap bitmap, Bitmap.CompressFormat format, int quality) {
        byte[] bytes = null;
        ByteArrayOutputStream outputStream = new ByteArrayOutputStream();
        bitmap.compress(format, quality, outputStream);
        bytes = outputStream.toByteArray();
        try {
            outputStream.close();
        } catch (Exception e) {
            e.printStackTrace();
        }
        return bytes;
    }

    public static Drawable toDrawable(Resources resources, Bitmap bitmap) {
        return new BitmapDrawable(resources, bitmap);
    }

    public static void toFile(Bitmap bitmap, Bitmap.CompressFormat format, int quality, String path) {
        FileOutputStream fileOutputStream = null;
        try {
            fileOutputStream = new FileOutputStream(path);
            bitmap.compress(format, quality, fileOutputStream);
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            try {
                if (fileOutputStream != null) {
                    fileOutputStream.close();
                }
            } catch (Exception e) {
                e.printStackTrace();
            }
        }
    }
    
}
```
