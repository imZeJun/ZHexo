---
title: Canvas&Paint 知识梳理(3) - 颜色合成 Paint#setColorFilter
date: 2017-03-03 19:58
categories : Canvas&Paint 知识梳理
---
# 一、概述
有时候，我们希望对一个图片或一个复杂图形的**颜色**，进行处理，那么这时候可以采用`Paint`的`setColorFilter`方法，一个最常见的例子，就是图片的滤镜，当然，那里面的算法可能更加复杂。
# 二、`ColorFilter`的分类
关于`ColorFilter`，源码中是这么解释的，它可以**对`Paint`所绘制区域的每个像素进行颜色的改变**。
```
/**
 * A color filter can be used with a {@link Paint} to modify the color of
 * each pixel drawn with that paint. This is an abstract class that should
 * never be used directly.
 */
```
当我们使用`ColorFilter`的时候，不一样直接使用它，而是使用它的子类：
- `ColorMatrixColorFilter`
- `LightingColorFilter`
- `PoterDuffColorFilter`

## 2.1 `ColorMatrixColorFilter`
`ColorMatrixColorFilter`的通过`ColorMatrix`构造，而`ColorMatrix`则由一个长度为`20`的`float`数组构造，传入该数组后把该数组先从左到右，再从上到下排列，形成一个`4*5`的矩阵。
```
[ a, b, c, d, e,
  f, g, h, i, j,
  k, l, m, n, o,
  p, q, r, s, t ]
```
之后，再用矩阵和目标的`RGBA`进行计算，最后得到新的`RGBA`，它的计算方法为：
```
R’ = a*R + b*G + c*B + d*A + e;
G’ = f*R + g*G + h*B + i*A + j;
B’ = k*R + l*G + m*B + n*A + o;
A’ = p*R + q*G + r*B + s*A + t;
```
注意，新的`RGBA`会被限制在`0-255`的范围内。在实际使用的时候，我们先通过一个`ColorMatrix`辅助类来确定需要相乘的这个颜色矩阵，之后再把它作为`ColorMatrixColorFilter`构造函数的参数来构造它。
我们一般有两种用法：改变画笔的颜色，或者改变整个`Bitmap`的像素点的颜色。下面我们用第二种方式来举例。
```
    private void drawColorMatrixFilter(Canvas canvas) {
        Bitmap bitmap = BitmapFactory.decodeResource(getResources(), R.mipmap.ic_launcher);
        //1,得到一个颜色矩阵
        ColorMatrix colorMatrix = new ColorMatrix();
        colorMatrix.setSaturation(0.5f);
        //2.通过颜色矩阵构建ColorMatrixColorFilter对象
        ColorMatrixColorFilter colorMatrixColorFilter = new ColorMatrixColorFilter(colorMatrix);
        Paint matrixPaint = new Paint();
        //3.把构建的对象设置给Paint
        matrixPaint.setColorFilter(colorMatrixColorFilter);
        canvas.drawBitmap(bitmap, 0, 0, matrixPaint);
    }
```
最终我们会得到一个蒙了灰的图片：
![](http://upload-images.jianshu.io/upload_images/1949836-0f4d46ced8a05590.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
当然我们也可以先设置画笔的颜色，然后给它设置一个颜色矩阵，这样最后花上去的图形的就是就是等于画笔颜色和矩阵一起计算的结果。
## 2.2 `LightingColorFilter`
用来模拟光照的效果，它定义了两个参数来和原`color`相乘，第一个`colorMultiply`原来相乘，而第二个参数`colorAdd`用来相加，并且会忽略其中的`Aplha`参数，这个`32`位的表示`0xAARRGGBB`，计算的公式为：
```
R' = R * colorMultiply.R + colorAdd.R
G' = G * colorMultiply.G + colorAdd.G
B' = B * colorMultiply.B + colorAdd.B
```
需要注意的是这个倍数为整形，也就是我们只能增大，不能减小。
```
    private void drawLightingColorFilter(Canvas canvas) {
        Bitmap bitmap = BitmapFactory.decodeResource(getResources(), R.mipmap.ic_launcher);
        //1.构建一个LightingColorFilter
        LightingColorFilter lightingColorFilter = new LightingColorFilter(3, 0);
        Paint matrixPaint = new Paint();
        //2.设置给画笔
        matrixPaint.setColorFilter(lightingColorFilter);
        //3.绘制
        canvas.drawBitmap(bitmap, 0, 0, matrixPaint);
    }
```
最终会得到下面的结果：
![](http://upload-images.jianshu.io/upload_images/1949836-34cfafcdf5f697a1.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
可以看到，对于原本`RGB`为`0`的像素点，它不会做任何改变，它只会改变那些原本有颜色的像素点。
## 2.3 `PorterDuffColorFilter`
混合模式，其构造函数为`PorterDuffColorFilter(int dstColor, PorterDuff.Mode mode)`，其中`color`为`16`进制的终点颜色，`mode`为混合策略，它会根据起点颜色`Sc`，起点透明度`Sa`，终点颜色`Dc`和终点透明度`Da`最终计算得出要显示的`RGBA`。
```
 * A color filter that can be used to tint the source pixels using a single
 * color and a specific {@link PorterDuff Porter-Duff composite mode}.
```
其中`mode`为`PorterDuff.Mode`，其对应的模式和计算公式如下：
```
public enum Mode {
        /** [0, 0] */
        CLEAR       (0),
        /** [Sa, Sc] */
        SRC         (1),
        /** [Da, Dc] */
        DST         (2),
        /** [Sa + (1 - Sa)*Da, Rc = Sc + (1 - Sa)*Dc] */
        SRC_OVER    (3),
        /** [Sa + (1 - Sa)*Da, Rc = Dc + (1 - Da)*Sc] */
        DST_OVER    (4),
        /** [Sa * Da, Sc * Da] */
        SRC_IN      (5),
        /** [Sa * Da, Sa * Dc] */
        DST_IN      (6),
        /** [Sa * (1 - Da), Sc * (1 - Da)] */
        SRC_OUT     (7),
        /** [Da * (1 - Sa), Dc * (1 - Sa)] */
        DST_OUT     (8),
        /** [Da, Sc * Da + (1 - Sa) * Dc] */
        SRC_ATOP    (9),
        /** [Sa, Sa * Dc + Sc * (1 - Da)] */
        DST_ATOP    (10),
        /** [Sa + Da - 2 * Sa * Da, Sc * (1 - Da) + (1 - Sa) * Dc] */
        XOR         (11),
        /** [Sa + Da - Sa*Da,
             Sc*(1 - Da) + Dc*(1 - Sa) + min(Sc, Dc)] */
        DARKEN      (16),
        /** [Sa + Da - Sa*Da,
             Sc*(1 - Da) + Dc*(1 - Sa) + max(Sc, Dc)] */
        LIGHTEN     (17),
        /** [Sa * Da, Sc * Dc] */
        MULTIPLY    (13),
        /** [Sa + Da - Sa * Da, Sc + Dc - Sc * Dc] */
        SCREEN      (14),
        /** Saturate(S + D) */
        ADD         (12),
        OVERLAY     (15);

        Mode(int nativeInt) {
            this.nativeInt = nativeInt;
        }

        /**
         * @hide
         */
        public final int nativeInt;
    }
```
其中，`Sa`表示起点的`Alpha`值，而`Sc`表示起点的`color`值，对于`Dx`也是同理，最终会得到`[Ra, Rc]`，这样组成之后就是终点对应像素点的颜色。
```
    private void drawPorterDuffColorFilter(Canvas canvas) {
        Bitmap bitmap = BitmapFactory.decodeResource(getResources(), R.mipmap.ic_launcher);
        Paint paint = new Paint();
        paint.setColorFilter(new PorterDuffColorFilter(Color.RED, PorterDuff.Mode.DARKEN));
        canvas.drawBitmap(bitmap, 0, 0, paint);
    }
```
最终会得到下面的结果：
![](http://upload-images.jianshu.io/upload_images/1949836-2e90396d9fbaf8cd.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
后面我们会发现`PorterDuff.Mode`中定义的这些`Mode`不仅仅用于颜色合成，还用于图像合成。
