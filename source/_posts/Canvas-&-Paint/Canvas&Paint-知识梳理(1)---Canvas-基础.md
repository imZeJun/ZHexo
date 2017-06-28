---
title: Canvas&Paint 知识梳理(1) - Canvas 基础
date: 2017-02-21 00:13
categories : Canvas&Paint 知识梳理
---
# 一、概述
经过前面对绘制原理的学习，我们知道当`View`的`onDraw(Canvas canvas)`方法被调用时，会传入一个`canvas`，我们通过这个`canvas`进行绘制，即可得到对应的图像。
我们主要了解以下几个方面：
- `Canvas`的含义
-  如何获得`Canvas`
- `Canvas`的变换
- `Canvas`的绘图
- 恢复和保存

## 二、`Canvas`的含义
涉及到`Canvas`的主要有以下三个概念：画板、画布和图层，画板包含了多个画布，每个画布中间又可能会包含多个图层，同一层级的元素会叠加在一块，在上一层中间进行展示。
- 画板
画板用来承载画布。
- 画布
每一个画布就是一个`bitmap`，所有的图像都是画在`bitmap`上的。
画布有两种：
 - `View`的原始画布，也就是`onDraw(Canvas canvas)`当中传入的`canvas`实例。
 - 人造画布，即通过`saveLayer`或者`new Canvas(Bitmap)`这些方法来人为新建一个画布，`saveLayer`新建一个画布之后，以后所有的`draw`函数所画的图像都是在这个画布上的，只有在`restore、restoreToCount`函数以后，才会返回到原始画布上绘制。
- 图层
图层是绘制的最小单元，每次调用`canvas.drawXXX`函数，都会生成一个专门的透明图层来画这个图形，画完之后，就盖在了画布上。连续调用`draw`方法，那么就会生成多个透明图层，画完之后依次在**当前的画布**上显示。

# 三、获得`Canvas`
## 3.1 重写`View`中的函数 `onDraw(Canvas canvas)` `dispatchDraw(Canvas canvas)`
在`ViewGroup`中，当它有背景的时候调用`onDraw`方法，否则会跳过`onDraw`直接调用`dispatchDraw`方法，在`View`中，两个都会调用的。
## 3.2 通过`Bitmap`创建

```
Canvas c = new Canvas(bitmap);
//或者
Canvas c = new Canvas();
c.setBitmap(bitmap);
```
如果我们通过这个方法构造了一个`canvas`，那么这个`canvas`上绘制的图像会保存在这个`bitmap`上，而不是在`View`上，如果想画在`View`上，就必须通过`onDraw`函数传进来的`canvas`画一遍。
## 3.3 `SurfaceHolder.lockCanvas()`

# 四、`Canvas`变换
关于`Canvas`的变换可以分为两类：可逆变换和不可逆变换。
其中可逆变换包括：平移、旋转、缩放和倾斜四种。
不可逆变换包括：裁剪。
## 4.1 可逆变换
### 4.1.1  平移
`void translate(float dx, dy)`，向右和向下为正方向，此时移动的是`canvas`原点的位置。
### 4.1.2 旋转
`rotate(float degrees, float px, float py)`，正数表示顺时针旋转，此外还可以指定旋转的中心，默认是`(0, 0)`

### 4.1.3 缩放
`scale(float sx, float sy, float px, py)`，对`x`和`y`进行缩放，并且可以指定缩放的中心点。

### 4.1.4 倾斜
`skew(float sx, float sy)`，倾斜画布，分别表示倾斜角度的`tan`值。

## 4.2 不可逆变换
调用`clipXXX`函数进行裁剪，通过与`Rect、Path、Region`取交、并、差等集合运算来获得最新的画布状态，这个操作是不可逆的，一旦`canvas`被裁剪，那么就不能被恢复。

#五、`Canvas`绘制
## 5.1 绘制基本图形
### 线
 `drawLine(float startX, float startY, float stopX, float stopY, Paint paint)`
`drawLines(float[] pts, Paint paint)`
`drawLines(float[] pts, int offset, int count, Paint paint)`
### 点
`drawPoint(float x, float y, Paint paint)`
`drawPoints(float[] pts, Paint paint)`
`drawPoints(float[] pts, int offset, int count, Paint paint)`
### 矩形
`drawRect(float left, float top, float right, float bottom, Paint)`
`drawRect(RectF rect, Paint paint)`
`drawRect(Rect rect, Paint paint)`
### 圆角矩形
`drawRoundRect(RectF rect, float rx, float ry, Paint paint)`
### 圆
`drawCircle(float cx, float cy, float radius, Paint paint)`
### 椭圆
`drawOval(RectF oval, Paint paint)`
### 扇形
`drawArc(RectF oval, float startAngle, float sweepAngle, boolean useCenter, Paint paint)`

## 5.2 调用`Path`绘制
在`canvas`中，绘制路径时使用`drawPath(Path path, Paint paint)`方法
## 直线路径
调用`moveTo(float x, float y)`可以移动到某个绘制点，`lineTo(float x, float y)`连到直线的结束点，如果连续画了几条直线，但没有形成闭环，调用`close()`会将路径首尾连接起来。
## 矩形路径
`addRect(RectF rect, Path.Direction dir)`和`drawRect(float l, float t, float r, float b, Path.Direction dir)`，用来绘制一条封闭的矩形路径，第2个参数决定了该路径的方向。
## 圆角矩形路径
`addRoundRect(RectF rect, float[] radii, Path.Direction dir)`和`addRoundRect(RectF, float rx, float ry, Path.Direction dir)`
## 圆形路径
`addCircle(float x, float y, float radius, Path.Direction dir)`
## 椭圆路径
`addOval(RectF oval, Path.Direction dir)`
## 弧形路径
`addArc(RectF oval, float startAngle, float sweepAngle)`
## 5.3 绘制文字
`canvas`的文字绘制提供以下3种：
### 普通水平绘制
`drawText(String text, float x, float y, Paint paint)`
### 指定各个文字位置
`drawPosText(String text, float[] pos, Paint)`
### 沿路径绘制
`drawTextOnPath(String text, Path path, float hOffset, float vOffset, Paint paint)`，其中`hOffset`表示与路径起始点的水平偏移量，`vOffset`表示与路径中心的水平偏移量。
