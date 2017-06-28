---
title: Canvas&Paint 知识梳理(5) - Paint#setShader
date: 2017-03-04 23:22
categories : Canvas&Paint 知识梳理
---
# 一、概述
`Shader`称为着色器，通过给`Paint`设置`Shader`，我们可以对图像进行渲染，在实际的使用当中，我们一般使用`Shader`的以下五个子类来实现不同的效果：
- `BitmapShader`
- `LinearGradient`
- `SweepGradient`
- `RadialGradient`
- `ComposeShader`

其中第`1`个用来设置`Bitmap`的变换，第`2~4`用来设置颜色的变换，第`5`个用来组合上面的几个`Shader`，下面我们一起来看以下各个子类的使用和应用场景。

# 二、使用示例
## 2.1 `BitmapShader`
`BitmapShader`是所有五个子类当中唯一一个对`Bitmap`进行操作的，我们看一下它的构造函数：
```
    /**
     * Call this to create a new shader that will draw with a bitmap.
     *
     * @param bitmap            The bitmap to use inside the shader
     * @param tileX             The tiling mode for x to draw the bitmap in.
     * @param tileY             The tiling mode for y to draw the bitmap in.
     */
    public BitmapShader(@NonNull Bitmap bitmap, TileMode tileX, TileMode tileY) {
        mBitmap = bitmap;
        mTileX = tileX;
        mTileY = tileY;
        init(nativeCreate(bitmap, tileX.nativeInt, tileY.nativeInt));
    }
```
第一个参数很好理解，就是需要绘制的`Bitmap`，我们看一下后面的两个参数，它的取值有：
```
    public enum TileMode {
        /**
         * replicate the edge color if the shader draws outside of its
         * original bounds
         */
        CLAMP   (0),
        /**
         * repeat the shader's image horizontally and vertically
         */
        REPEAT  (1),
        /**
         * repeat the shader's image horizontally and vertically, alternating
         * mirror images so that adjacent images always seam
         */
        MIRROR  (2);
    
        TileMode(int nativeInt) {
            this.nativeInt = nativeInt;
        }
        final int nativeInt;
    }
```
需要注意的是，下面几种模式都是建立在绘制的区域要比原来的`bimtap`大的情况下的。
```
the shader draws outside of its original bounds
```
- `CLAMP`：取`bitmap`边缘的最后一个像素进行扩展。
- `REPEAT`：水平地重复整张`bitmap`。
- `MIRROR`：和`REPEAT`类似，但是每次重复的时候，将`bitmap`进行翻转。

### 2.1.1 `CLAMP`
首先，我们取一张宽高为`200dp * 200dp`的图片，我们整个`View`的宽高为`300dp * 300dp`，


![](http://upload-images.jianshu.io/upload_images/1949836-b6b9d26bc7d91a84.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
我们首先采用`CLAMP`的模式：
```
    private Bitmap mOriginalBitmap;
    private Paint mPaint;

    private void init() {
        mOriginalBitmap = BitmapFactory.decodeResource(getResources(), R.drawable.shader_pic);
        mPaint = new Paint();
    }

    private void drawBitmapShader(Canvas canvas) {
        BitmapShader shader = new BitmapShader(mOriginalBitmap, Shader.TileMode.CLAMP, Shader.TileMode.CLAMP);
        mPaint.setShader(shader);
        canvas.drawRect(0, 0, canvas.getWidth(), canvas.getHeight(), mPaint);
    }
```
最终得到的结果为下图，可以看到，由于`Paint`绘制的宽高要比`Bitmap`原本的宽高大，因此对于多出的部分，取了边缘最后一个像素的颜色进行重复：
![](http://upload-images.jianshu.io/upload_images/1949836-2468a7d97f5ba29c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
现在有个疑问，因为整个图片的大小为`600 * 600`，而我们绘制的大小为`900 * 900`，按前面的说法，对于`(600,0) - (899, 600)`的区域，取的是`(599, 0) - (599, 599)`这一列的颜色，而对于`(0, 600) - (600, 899)`取的是`(0, 599) - (599, 599)`这一行的颜色，那么`(600, 600) - (899, 899)`这一区域是怎么取的呢？
现在，我们试一下，把最后`drawRect`的起始点改为`(100, 100)`：
```
canvas.drawRect(100, 100, canvas.getWidth(), canvas.getHeight(), mPaint);
```
得到的效果如下图，可以看到，边缘部分被切割掉了。
![](http://upload-images.jianshu.io/upload_images/1949836-7fc9a3b00762df18.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 2.1.2 `REPEAT/MIRROR`
对于这两种模式，实现方式和上面类似，我们就不再重复描述了，只给出下面运行的结果：
```
BitmapShader shader = new BitmapShader(mOriginalBitmap, Shader.TileMode.REPEAT, Shader.TileMode.REPEAT);
```
![](http://upload-images.jianshu.io/upload_images/1949836-19e29e3dc9c97cfa.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
```
BitmapShader shader = new BitmapShader(mOriginalBitmap, Shader.TileMode.MIRROR, Shader.TileMode.MIRROR);
```
![](http://upload-images.jianshu.io/upload_images/1949836-3382940597e720ae.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
得到的结果都是和描述相符的。
### 2.1.3 当`X`轴和`Y`轴的`TileMode`不同时
上面讨论的情况，都是`x`轴和`y`轴的`TileMode`相同的情况，现在，我们来看一下，当两者不同时，会发生什么情况：
```
BitmapShader shader = new BitmapShader(mOriginalBitmap, Shader.TileMode.CLAMP, Shader.TileMode.MIRROR);
```
最终的结果如下，可以看到，我们是**先按`x`轴的模式进行处理，然后将`x`轴处理完毕后的图像再按`y`轴的模式进行处理**，这也解释了我们前面在`2.1.1`中留下的疑问。
![](http://upload-images.jianshu.io/upload_images/1949836-f9ccd83bd3ca112b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 2.2 `LinearGradient`
`LinearGradient`用来处理线性渐变，同理我们先来看它的构造函数说明，和前面不同，它有两个构造函数，其中一种是另一种的简化版，我们直接来看复杂的一种：
```
    /** Create a shader that draws a linear gradient along a line.
        @param x0           The x-coordinate for the start of the gradient line
        @param y0           The y-coordinate for the start of the gradient line
        @param x1           The x-coordinate for the end of the gradient line
        @param y1           The y-coordinate for the end of the gradient line
        @param  colors      The colors to be distributed along the gradient line
        @param  positions   May be null. The relative positions [0..1] of
                            each corresponding color in the colors array. If this is null,
                            the the colors are distributed evenly along the gradient line.
        @param  tile        The Shader tiling mode
    */
    public LinearGradient(float x0, float y0, float x1, float y1, int colors[], float positions[], TileMode tile) {
```
下面，我们从几个方面来分析一下这个构造函数中的参数。
### 2.2.1 起点坐标和终点坐标
对于这两个点的坐标我们可以这么理解，起点的颜色就是`color[]`数组的第一个元素，终点的颜色就是`color[]`数组的最后一个元素，这两个点的连线决定了线性变化的方向，如果两点连线和`x`轴的正方向是重合的时候，那么就是水平地变化，当和`x`轴正方向有度数时，那么这个连线相对于`x`轴旋转了多少，最后线性变化的图像也就会相对于水平变化的图像旋转了多少，下面我们用两个例子来说明。
首先是水平方向的：
```
    private void drawLinearGradient(Canvas canvas) {
        LinearGradient gradient = new LinearGradient(0, 0, 100, 0, new int[]{ Color.WHITE, Color.BLACK }, null, Shader.TileMode.REPEAT);
        mPaint.setShader(gradient);
        canvas.drawRect(0, 0, 900, 900, mPaint);
    }
```
这时候的图像为：
![](http://upload-images.jianshu.io/upload_images/1949836-1ca5a5b00e424cd1.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
下面，我们将终点的`y`轴坐标下移一点，让起点坐标和终点坐标的连线，与`x`轴形成一定的角度：
```
    private void drawLinearGradient(Canvas canvas) {
        LinearGradient gradient = new LinearGradient(0, 0, 100, 10, new int[]{ Color.WHITE, Color.BLACK }, null, Shader.TileMode.REPEAT);
        mPaint.setShader(gradient);
        canvas.drawRect(0, 0, 900, 900, mPaint);
    }
```
这时候的图像为，可以看到，由于此时连线相对于`x`轴，顺时针旋转了一定的度数，那么最终的图像也相对于上面那种情况顺时针旋转了相应的度数。
![](http://upload-images.jianshu.io/upload_images/1949836-10152525b3f342b1.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 2.2.2 `colors`和`positions`
这两个参数很好理解，因为在颜色由起点颜色变到终点颜色的过程中，我们可能还希望中间会经过别的颜色，那么这时候，我们就可以在数组的第一个和最后一个元素当中插入别的元素，这些元素就是中间会经过的颜色，并且当`positions`不为`null`的时候，`colors`的大小要和`positions`相同。
```
    private void drawLinearGradient(Canvas canvas) {
        LinearGradient gradient = new LinearGradient(0, 0, 100, 10, new int[]{ Color.WHITE, Color.BLUE, Color.BLACK }, new float[]{0, 0.5f, 1f}, Shader.TileMode.REPEAT);
        mPaint.setShader(gradient);
        canvas.drawRect(0, 0, 900, 900, mPaint);
    }
```
结果为：
![](http://upload-images.jianshu.io/upload_images/1949836-f312035f9b19e651.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
### 2.2.3 `TileMode`
和`BitmapShader`不同，此时我们只用指定一个方向的变化，这个方向就是颜色线性变化对应的方向。
### 2.2.4 另一个构造函数
```
    /** Create a shader that draws a linear gradient along a line.
        @param x0       The x-coordinate for the start of the gradient line
        @param y0       The y-coordinate for the start of the gradient line
        @param x1       The x-coordinate for the end of the gradient line
        @param y1       The y-coordinate for the end of the gradient line
        @param  color0  The color at the start of the gradient line.
        @param  color1  The color at the end of the gradient line.
        @param  tile    The Shader tiling mode
    */
    public LinearGradient(float x0, float y0, float x1, float y1, int color0, int color1,
            TileMode tile) {
        mType = TYPE_COLOR_START_AND_COLOR_END;
        mX0 = x0;
        mY0 = y0;
        mX1 = x1;
        mY1 = y1;
        mColor0 = color0;
        mColor1 = color1;
        mTileMode = tile;
        init(nativeCreate2(x0, y0, x1, y1, color0, color1, tile.nativeInt));
    }
```
唯一不同的就是去掉了`colors`和`position`数组，变成了`color0`和`color1`，那么我们就只能指定起点和终点的颜色了，其它的原理和上面那个构造函数是相同的。
## 2.3 `SweepGradient`
它用来提供类似雷达的效果，同理，我们看一下构造函数：
```
    /**
     * A subclass of Shader that draws a sweep gradient around a center point.
     *
     * @param cx       The x-coordinate of the center
     * @param cy       The y-coordinate of the center
     * @param colors   The colors to be distributed between around the center.
     *                 There must be at least 2 colors in the array.
     * @param positions May be NULL. The relative position of
     *                 each corresponding color in the colors array, beginning
     *                 with 0 and ending with 1.0. If the values are not
     *                 monotonic, the drawing may produce unexpected results.
     *                 If positions is NULL, then the colors are automatically
     *                 spaced evenly.
     */
    public SweepGradient(float cx, float cy, int colors[], float positions[]) 
```
### 2.3.1 中心点坐标`(cx, cy)`
对于`(cx, cy)`中心点的坐标，我们可以把它想象成一个时钟的指针，这个指针开始时指向`3`点钟方向，它初始的颜色就是起点颜色，那么它会以此为起点，顺时针旋转`360`度，在旋转的过程中，这个指针的颜色不断变化，当旋转到`360`度后，指针就变成了终点颜色，在旋转过程中，指针所形成的轨迹就是最终的图像。
### 2.3.2 `TileMode`
需要注意到，它和`LinearGradient`不同的是，由于指针是无限长的，所以形成的图像在`x`轴和`y`轴所拼接成的区域是无限大的，因此也就不存在了`TileMode`这个参数的必要了。
### 2.3.3 `colors[]`和`positions[]`
这两个数组的作用和上面`LinearGradient`的两个数组的作用是相同的，这里就不重复说明了。
### 2.3.4 举例
下面举个简单的例子：
```
    private void drawSweepGradient(Canvas canvas) {
        SweepGradient gradient = new SweepGradient(450, 450, Color.WHITE, Color.BLACK);
        mPaint.setShader(gradient);
        canvas.drawRect(0, 0, 900, 900, mPaint);
    }
```
最后的结果为：
![](http://upload-images.jianshu.io/upload_images/1949836-c24904d3e8aae172.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
### 2.3.5 另一个构造函数
```
    /**
     * A subclass of Shader that draws a sweep gradient around a center point.
     *
     * @param cx       The x-coordinate of the center
     * @param cy       The y-coordinate of the center
     * @param color0   The color to use at the start of the sweep
     * @param color1   The color to use at the end of the sweep
     */
    public SweepGradient(float cx, float cy, int color0, int color1) {
        mType = TYPE_COLOR_START_AND_COLOR_END;
        mCx = cx;
        mCy = cy;
        mColor0 = color0;
        mColor1 = color1;
        init(nativeCreate2(cx, cy, color0, color1));
    }
```
和前面`LinearGradient`中讨论的一样，`color0`和`color1`就是`colors[]`和`positions[]`的简化版本。

## 2.4 `RadialGradient`
它被称为圆形渐变，构造函数如下：
```
    /** Create a shader that draws a radial gradient given the center and radius.
        @param centerX  The x-coordinate of the center of the radius
        @param centerY  The y-coordinate of the center of the radius
        @param radius   Must be positive. The radius of the circle for this gradient.
        @param colors   The colors to be distributed between the center and edge of the circle
        @param stops    May be <code>null</code>. Valid values are between <code>0.0f</code> and
                        <code>1.0f</code>. The relative position of each corresponding color in
                        the colors array. If <code>null</code>, colors are distributed evenly
                        between the center and edge of the circle.
        @param tileMode The Shader tiling mode
    */
    public RadialGradient(float centerX, float centerY, float radius, @NonNull int colors[], @Nullable float stops[], @NonNull TileMode tileMode) 
```
### 2.4.1 原点坐标`(centerX, centerY)`和半径`radius`
对于圆形渐变，我们可以这么理解，开始的时候，有一个半径无限小的圆环位于`(centerX, centerY)`，它的颜色就是起点颜色，之后它开始慢慢变大，直到变为半径是`radius`为止，在此期间，圆环的颜色慢慢变为终点颜色，在整个变化的过程中，圆环所形成的轨迹就是最终的图像。
### 2.4.2 `TileMode`
由于在这种情况下，图像的大小是有限的，最大就是`radius`指定的范围，因此对于超出范围的图像，我们需要定义它的行为，但是原理还是和前面讨论的`TileMode`的三种情况一样的。
### 2.4.3 `colors[]`和`stops[]`
原理和上面讨论的`colors[]`和`positions[]`一样。
### 2.4.4 示例
```
    private void drawRadialGradient(Canvas canvas) {
        RadialGradient gradient = new RadialGradient(200, 200, 50, Color.BLUE, Color.RED, Shader.TileMode.REPEAT);
        mPaint.setShader(gradient);
        canvas.drawRect(0, 0, 900, 900, mPaint);
    }
```
最终的结果为：
![](http://upload-images.jianshu.io/upload_images/1949836-aeda159c7cf3e4c9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
## 2.5 `ComposeShader`
上面，我们已经学习了四种`Shader`的实现方式，但是有时候，我们希望能够将它组合起来，`ComposeShader`就为我们提供了这种途径，可以组合两种`Shader`的实现。
```
    /** Create a new compose shader, given shaders A, B, and a combining mode.
        When the mode is applied, it will be given the result from shader A as its
        "dst", and the result from shader B as its "src".
        @param shaderA  The colors from this shader are seen as the "dst" by the mode
        @param shaderB  The colors from this shader are seen as the "src" by the mode
        @param mode     The mode that combines the colors from the two shaders. If mode
                        is null, then SRC_OVER is assumed.
    */
    public ComposeShader(Shader shaderA, Shader shaderB, Xfermode mode)
```
这就涉及到之前我们学过的`PorterDuff.Mode`，第一个`Shader`作为`DST`，而第二个`Shader`作为`SRC`，两个组合的结果会根据`Mode`的不同而发生改变，下面我们用一个简单的例子，来看一下`BitmapShader`和`RadialGradient`的组合：
```
    private void drawComposeShader(Canvas canvas) {
        BitmapShader bitmapShader = new BitmapShader(mOriginalBitmap, Shader.TileMode.CLAMP, Shader.TileMode.MIRROR);
        RadialGradient radialGradient = new RadialGradient(300, 300, 300, Color.TRANSPARENT, Color.WHITE, Shader.TileMode.CLAMP);
        ComposeShader composeShader = new ComposeShader(bitmapShader, radialGradient, PorterDuff.Mode.SRC_OVER);
        mPaint.setShader(composeShader);
        canvas.drawCircle(300, 300, 300, mPaint);
    }
```
最终的结果为下图，可以看到，由于我们采用了`SRC_OVER`，因此就会出现朦胧的效果。
![](http://upload-images.jianshu.io/upload_images/1949836-767d7a5b6aa1adcc.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
