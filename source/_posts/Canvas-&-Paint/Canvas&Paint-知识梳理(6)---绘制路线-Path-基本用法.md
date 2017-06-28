---
title: Canvas&Paint 知识梳理(6) - 绘制路线 Path 基本用法
date: 2017-03-05 22:58
categories : Canvas&Paint 知识梳理
---
# 一、概述
在实际的开发当中，我们经常会涉及到绘制路径，这里我们总结一下`Path`的常用`API`。

# 二、基本用法
对于一个`Path`来说，它其中有很多的”子路径“，对于每个”子路径“，它又会有两个变量，"源点"和"当前点"，也就是源码当中所描述的：
```
The Path class encapsulates compound (multiple contour) geometric paths
```
## 2.1 简单连线 - `xxxTo`
下面是`Path`当中和连线相关的方法：

![](http://upload-images.jianshu.io/upload_images/1949836-f47481e9098317be.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
```
public void moveTo(float x, float y)
public void rMoveTo(float dx, float dy)
```
每次调用完`moveTo/rMoveTo`方法，都会生成一条新的子路径，这两者的区别在于：
`1.`新建一条路径，`(x, y)`作为这个新路径的“源点"的值，在图上不会产生新的连线。
`2.`相对于当前路径的"当前点"的值，移动坐标`(dx, dy)`，作为新路径的"源点"，在图上不会产生新的连线。


```
public void lineTo(float x, float y)
public void rLineTo(float dx, float dy)
```
`1.`从当前点位置，直线连接到绝对坐标`(x, y)`，如果没有调用过`moveTo/rMoveTo`，那么会先调用`moveTo(0, 0)`来生成一条从源点开始的子路径。
`2.`相对于当前点坐标，移动`(dx, dy)`作为终点坐标，并连接起点和终点。
```
public void quadTo(float x1, float y1, float x2, float y2)
public void rQuadTo(float dx1, float dy1, float dx2, float dy2)
```
`1.`从当前点位置开始，以`(x1, y1)`为控制点，按一阶贝塞尔曲线计算方法连接到`(x2, y2)`，如果之前没有调用过`moveTo/rMoveTo`，也会先调用一次`moveTo(0, 0)`方法。
`2.`类似于上面，只不过终点坐标是相对于当前点坐标移动了`(dx2, dy2)`后的值。

```
public void cubicTo(float x1, float y1, float x2, float y2, float x3, float y3)
public void rCubicTo(float x1, float y1, float x2, float y2, float x3, float y3)
```
和一阶贝塞尔曲线类似，不过是多了一个控制点`(x2, y2)`。
```
public void arcTo(RectF oval, float startAngle, float sweepAngle, boolean forceMoveTo)
public void arcTo(RectF oval, float startAngle, float sweepAngle)
public void arcTo(float left, float top, float right, float bottom, float startAngle, float sweepAngle, boolean forceMoveTo)
```
上面这三个函数，最终都是调用了同一个方法：
```
native_arcTo(mNativePath, left, top, right, bottom, startAngle, sweepAngle, forceMoveTo)
```
它的原理就是：首先根据`RectF`或者`4`个点的坐标来确定一个矩形区域，之后得到这个矩形的内切椭圆，然后再根据`startAngle`和`sweepAngle`来确定这个内切源的一段弧，这段弧就是新绘制的路径。但是这段弧的起点很有可能和`Path`的当前点不是重合的，这时候就根据`forceMoveTo`来决定是否产生一条新的子路径，从最终的结果看，就是是否需要绘制一条从`Path`当前点到这段弧起点的连线，如果为`forceMoveTo=false`，那么就绘制这么一条直线，反之，则不绘制，`forceMoveTo`的默认值为`false`。
`forceMoveTo=false`
```
    private void drawArcTo(Canvas canvas) {
        setLayerType(LAYER_TYPE_SOFTWARE, null);
        Path path = new Path();
        path.moveTo(0, 300);
        path.lineTo(300, 300);
        path.arcTo(0, 0, 500, 500, 0, -90, false);
        path.rLineTo(0, 250);
        canvas.drawPath(path, mPaint);
    }
```

![](http://upload-images.jianshu.io/upload_images/1949836-b9a34e2eb6becd58.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

`forceMoveTo=true`
```
    private void drawArcTo(Canvas canvas) {
        setLayerType(LAYER_TYPE_SOFTWARE, null);
        Path path = new Path();
        path.moveTo(0, 300);
        path.lineTo(300, 300);
        path.arcTo(0, 0, 500, 500, 0, -90, true);
        path.rLineTo(0, 250);
        canvas.drawPath(path, mPaint);
    }
```
![](http://upload-images.jianshu.io/upload_images/1949836-776e94d2da5d2bd0.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

`forceMoveTo=true + path.close()`
```
    private void drawArcTo(Canvas canvas) {
        setLayerType(LAYER_TYPE_SOFTWARE, null);
        Path path = new Path();
        path.moveTo(0, 300);
        path.lineTo(300, 300);
        path.arcTo(0, 0, 500, 500, 0, -90, true);
        path.rLineTo(0, 250);
        path.close();
        canvas.drawPath(path, mPaint);
    }
```
对于这种情况，由于采用了`true`标志位，因此生成了一条新的”子路径“，所以在调用`close`方法之后，连接的是当前子路径的源点和当前点。
![](http://upload-images.jianshu.io/upload_images/1949836-3c49324fb4ddb3a7.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

```
public void close()
```
如果对于当前”子路径“来说，它的"当前点"和"源点"不重合，那么会绘制一条从当前点到源点的直线。
## 2.2 直接增加新的路径
除了采用连线的方法来确定一条路径，`Path`还提供了`addXXX`来直接地增加一整段的路径，下面是相关的方法：
![](http://upload-images.jianshu.io/upload_images/1949836-c7f1474265694a5a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
效果其实从函数名上就可以很清楚地看出来，当调用完`addXXX`方法之后，会增加一条子路径到`Path`当中，并把该子路径作为`Path`的当前子路径。
```
    private void drawAddArc(Canvas canvas) {
        setLayerType(LAYER_TYPE_SOFTWARE, null);
        Path path = new Path();
        path.moveTo(0, 250);
        path.lineTo(500, 250);
        path.addArc(0, 0, 500, 500, 0, -90);
        path.close();
        canvas.drawPath(path, mPaint);
    }
```
下面是运行的结果：
![](http://upload-images.jianshu.io/upload_images/1949836-883424da8329f23f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
## 2.3 填充类型`FillType`
关于填充类型的方法有如下这些：
![](http://upload-images.jianshu.io/upload_images/1949836-1d4450c3285b8718.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
我们先来看一下`FillType`的定义有哪些：
```
    public enum FillType {
        // these must match the values in SkPath.h
        /**
         * Specifies that "inside" is computed by a non-zero sum of signed
         * edge crossings.
         */
        WINDING         (0),
        /**
         * Specifies that "inside" is computed by an odd number of edge
         * crossings.
         */
        EVEN_ODD        (1),
        /**
         * Same as {@link #WINDING}, but draws outside of the path, rather than inside.
         */
        INVERSE_WINDING (2),
        /**
         * Same as {@link #EVEN_ODD}, but draws outside of the path, rather than inside.
         */
        INVERSE_EVEN_ODD(3);

        FillType(int ni) {
            nativeInt = ni;
        }

        final int nativeInt;
    }
```
关于这个`FillType`的理解，下面这篇文章的作者说的很好：
> [`http://blog.csdn.net/u013831257/article/details/51477575`](http://blog.csdn.net/u013831257/article/details/51477575)

我这里只是稍微地总结一下：
### 2.3.1 `FillType`的意义
我们都知道`Paint`有三种模式：`FILL/FILL_AND_STROKE/STROKE`，对于`STROKE`来说，是只绘制轮廓，而两种模式都涉及到”填充“，那么”填充“就涉及到怎么定义一个`Path`所组成的图形的内部，`FillType`就是用来确定这个所谓的”内部“的定义的，需要注意，只讨论封闭图形的情况。
### 2.3.2 `FillType`的类型的含义
- `EVEN_ODD`表示奇偶规则：奇数表示在图形内，偶数表示在图形外，并绘制内部。
从任意位置`p`作一条射线， 若与该射线相交的图形边的数目为奇数，则`p`是图形内部点，否则是外部点。
- `INVERSE_EVEN_ODD`：和`EVEN_ODD`对应，绘制外部。
- `WINDING`表示非零环绕数规则：若环绕数为`0`表示在图形外，非零表示在图形内，并绘制内部。
首先使图形的边变为矢量。将环绕数初始化为零。再从任意位置`p`作一条射线。当从`p`点沿射线方向移动时，对在每个方向上穿过射线的边计数，每当图形的边从右到左穿过射线时，环绕数加`1`，从左到右时，环绕数减`1`。处理完图形的所有相关边之后，若环绕数为非零，则`p`为内部点，否则，`p`是外部点。
- `INVERSE_WINDING`：和`WINDING`对应，绘制外部。

### 2.3.3 示例
```
    private void drawFillType(Canvas canvas) {
        setLayerType(LAYER_TYPE_SOFTWARE, null);
        Path fillTypePath = new Path();
        //两个圆圈都为顺时针的情况.
        fillTypePath.addCircle(250, 250, 250, Path.Direction.CW);
        fillTypePath.addCircle(500, 500, 250, Path.Direction.CW);
        //填充类型采用奇偶原则.
        fillTypePath.setFillType(Path.FillType.EVEN_ODD);
        canvas.drawPath(fillTypePath, mPaint);
    }
```
![](http://upload-images.jianshu.io/upload_images/1949836-c5a14b5a15b36e3e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
```
    private void drawFillType(Canvas canvas) {
        setLayerType(LAYER_TYPE_SOFTWARE, null);
        Path fillTypePath = new Path();
        //两个圆圈都为顺时针的情况.
        fillTypePath.addCircle(250, 250, 250, Path.Direction.CW);
        fillTypePath.addCircle(500, 500, 250, Path.Direction.CW);
        //填充类型采用非零环绕数规则.
        fillTypePath.setFillType(Path.FillType.WINDING);
        canvas.drawPath(fillTypePath, mPaint);
    }
```
![](http://upload-images.jianshu.io/upload_images/1949836-45409e8d802fe768.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
```
    private void drawFillType(Canvas canvas) {
        setLayerType(LAYER_TYPE_SOFTWARE, null);
        Path fillTypePath = new Path();
        //第一个圆圈为顺时针,第二个圆圈为逆时针的情况.
        fillTypePath.addCircle(250, 250, 250, Path.Direction.CW);
        fillTypePath.addCircle(500, 500, 250, Path.Direction.CCW);
        //填充类型采用奇偶原则.
        fillTypePath.setFillType(Path.FillType.EVEN_ODD);
        canvas.drawPath(fillTypePath, mPaint);
    }
```
![](http://upload-images.jianshu.io/upload_images/1949836-c5a14b5a15b36e3e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
```
    private void drawFillType(Canvas canvas) {
        setLayerType(LAYER_TYPE_SOFTWARE, null);
        Path fillTypePath = new Path();
        //第一个圆圈为顺时针,第二个圆圈为逆时针的情况.
        fillTypePath.addCircle(250, 250, 250, Path.Direction.CW);
        fillTypePath.addCircle(500, 500, 250, Path.Direction.CCW);
        //填充类型采用非零环绕数规则.
        fillTypePath.setFillType(Path.FillType.WINDING);
        canvas.drawPath(fillTypePath, mPaint);
    }
```
![](http://upload-images.jianshu.io/upload_images/1949836-c5a14b5a15b36e3e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
可以看到，对于`2`和`4`，由于`Path.FillType.WINDING`会涉及到路径的方向，因此路径方向不同会影响它的结果，但是对比`1`和`3`，由于`EVEN_ODD`的判断只涉及到边，不涉及到路径的方向，因此不会影响它的结果。
## 2.4 `Path`的计算
关于`Path`的计算，有下面这两个方法：
![](http://upload-images.jianshu.io/upload_images/1949836-7c3bbd58997d58da.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
其中，第一个表示对当前`Path`和参数中的`Path`进行计算，计算结果放入当前`Path`当中；第二个表示对参数内的两个`Path`进行计算，计算的结果放入到当前`Path`中，计算的类型有以下几种：
```
    public enum Op {
        /**
         * Subtract the second path from the first path.
         */
        DIFFERENCE,
        /**
         * Intersect the two paths.
         */
        INTERSECT,
        /**
         * Union (inclusive-or) the two paths.
         */
        UNION,
        /**
         * Exclusive-or the two paths.
         */
        XOR,
        /**
         * Subtract the first path from the second path.
         */
        REVERSE_DIFFERENCE
    }
```
- `DIFFERENCE`：从`Path1`中减去`Path2`。
- `INTERSECT`：取`Path1`和`Path2`的交集。
- `UNION`：取`Path1`和`Path2`的并集。
- `XOR`：从`Path1`和`Path2`的并集中，减去它们的交集。
- `REVERSE_DIFFERENCE`：从`Path2`中减去`Path1`。

我们以`XOR`为例子：
```
    private void drawOp(Canvas canvas) {
        setLayerType(LAYER_TYPE_SOFTWARE, null);
        Path path1 = new Path();
        //Path1为顺时针.
        path1.addCircle(250, 250, 250, Path.Direction.CW);
        Path path2 = new Path();
        //Path2为逆时针
        path2.addCircle(400, 250, 250, Path.Direction.CCW);
        path1.op(path2, Path.Op.XOR);
        canvas.drawPath(path1, mPaint);
    }
```
最后的结果为：
![](http://upload-images.jianshu.io/upload_images/1949836-8015ddb9fe3fb3b2.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
## 2.5 重置方法
`Path`提供了两种重置方法：
```
    /**
     * Clear any lines and curves from the path, making it empty.
     * This does NOT change the fill-type setting.
     */
    public void reset()

    /**
     * Rewinds the path: clears any lines and curves from the path but
     * keeps the internal data structure for faster reuse.
     */
    public void rewind()
```
- `reset()`：清除路径及其信息，但保留`FillType`。
- `rewind()`：清除路径，但保留信息。
