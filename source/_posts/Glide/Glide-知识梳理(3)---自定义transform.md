---
title: Glide 知识梳理(3) - 自定义transform
date: 2017-03-12 17:37
categories : Glide 知识梳理
---
# 一、概述
有时候，当我们去服务器请求图片资源成功之后，希望先对它进行一些处理，例如缩放、旋转、蒙灰等，然后再把处理后的图片展示在控件上，这时候就可以用上`Glide`提供的`transform()`方法，简单来说，`transform()`的作用就是**改变原始资源在客户端上最终的展现结果**。

# 二、示例
## 2.1 采用`BitmapTransformation`进行变换
首先，`transform(xxx)`接收的参数类型有两种，一种是`BitmapTransformation`，另一种是`Transformtion<GifBitmapWrapper>`，下面是它们的定义，在平时的使用过程中，如果我们的资源为静态图片，而不是`Gif`或者`Media`，那么通过继承`BitmapTransformation`来实现自己的变换就可以了。
```
    //接收BitmapTransformation作为参数.
    public DrawableRequestBuilder<ModelType> transform(BitmapTransformation... transformations) {
        return bitmapTransform(transformations);
    }
    //接收Transformation作为参数.
    @Override
    public DrawableRequestBuilder<ModelType> transform(Transformation<GifBitmapWrapper>... transformation) {
        super.transform(transformation);
        return this;
    }
```
下面，我们就以一下简单的例子，来看一下如何使用`BitmapTransformation`来改变图片的展示结果。
这里的变换用到了前面介绍过的`setShader`相关的知识，可以参考下面这篇文章：
> [`http://www.jianshu.com/p/6ab058329ca8`](http://www.jianshu.com/p/6ab058329ca8)

首先，定义自己的`BitmapTransformation`实现：
```
    private class MyBitmapTransformation extends BitmapTransformation {

        public MyBitmapTransformation(Context context) {
            super(context);
        }

        @Override
        protected Bitmap transform(BitmapPool pool, Bitmap toTransform, int outWidth, int outHeight) {
            Canvas canvas = new Canvas(toTransform);
            BitmapShader bitmapShader = new BitmapShader(toTransform, Shader.TileMode.CLAMP, Shader.TileMode.CLAMP);
            int min = Math.min(toTransform.getWidth(), toTransform.getHeight());
            int radius = min / 2;
            RadialGradient radialGradient = new RadialGradient(toTransform.getWidth() / 2 , toTransform.getHeight() / 2, radius, Color.TRANSPARENT, Color.WHITE, Shader.TileMode.CLAMP);
            ComposeShader composeShader = new ComposeShader(bitmapShader, radialGradient, PorterDuff.Mode.SRC_OVER);
            Paint paint = new Paint();
            paint.setShader(composeShader);
            canvas.drawRect(0, 0, toTransform.getWidth(), toTransform.getHeight(), paint);
            return toTransform;
        }

        @Override
        public String getId() {
            return "MyBitmapTransformation";
        }
    }
```
得到了`BitmapTransform`之后，通过下面的方式来加载图片资源，并传入我们定义的`Transformation`
```
    public void loadTransform(View view) {
        MyBitmapTransformation myBitmapTransformation = new MyBitmapTransformation(this);
        Glide.with(this)
                .load("http://i.imgur.com/DvpvklR.png")
                .diskCacheStrategy(DiskCacheStrategy.NONE)
                .transform(myBitmapTransformation)
                .into(mImageView);
    }
```
最终得到的结果如下图所示：
![](http://upload-images.jianshu.io/upload_images/1949836-ae4ff0cd4bf87b8f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
## 2.2 使用多个`BitmapTransformation`
从上面的定义中，可以看到`transform`接收的参数个数是可变，也就是说，我们可以传入多个`Transformation`进行组合，它们会按顺序应用到最终的图像上。
# 三、方法说明
看完上面的应用之后，我们去源码中看一下各方法的说明。
## 3.1 `Transformation<T>`
`BitmapTransformation`是实现了`Transformation<T>`接口的抽象类，源码当中对于`Transformation`的描述是：
```
/**
 * A class for performing an arbitrary transformation on a resource.
 *
 * @param <T> The type of the resource being transformed.
 */
```
`BitmapTransform`的参数`T`就是`Bitmap`，也就是说它是用来给`Bitmap`进行变换，`Transformation`定义了两个接口：
- `Resource<T> transform(Resource<T> resource, int outWidth, int outHeight)`
`resource`的`get`方法会返回原始资源，它可能是从服务器上面下载的图片，也可能是上一个`transformation`变换后的资源，而返回值就是经过变化后的`resource`，而`outWidth/outHeight`就是我们所要加载到的目标`Target`的宽高，在上面的例子中，就是`ImageView`所指定的宽高。
- `String getId()`
前面在介绍基础使用的文章当中，我们说过最终缓存的资源文件的`Cache key`会依赖于一些因素，这个`getId`方法的返回值就是其中的一个因素。

## 3.2 `BitmapTransformation`
`BitmapTransformation`是实现了`Transformation`的抽象类，它实现了`transform(Resource<T> resource, int outWidth, int outHeight)`这个方法，并在里面调用了下面这个方法让子类去实现，也就是我们例子中实现的抽象方法，这主要是方便我们通过`BitmapPool`对`Bitmap`对象进行复用，同时使用者只用关心需要变换的`Bitmap`，`BitmapTransformation`会负责把它更新回原来的`Resource`对象中。
```
protected abstract Bitmap transform(BitmapPool pool, Bitmap toTransform, int outWidth, int outHeight);
```
对于这个方法的实现，有几个需要注意的点：
- 对于传入的`toTransform Bitmap`，不要去回收它或者把它放入到缓存池。如果实现者返回了一个不同的`Bitmap`实例，`Glide`会负责去回收或者复用`toTransform`这个`Bitmap`。
- 不要去回收作为这个方法返回值的`bitmap`。
- 如果在这个方法的执行过程中产生了临时的`Bitmap`实例，那么最好把它放入缓存池，或者回收它。
