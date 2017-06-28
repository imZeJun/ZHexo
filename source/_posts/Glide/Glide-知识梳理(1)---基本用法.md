---
title: Glide 知识梳理(1) - 基本用法
date: 2017-03-11 21:27
categories : Glide 知识梳理
---
# 一、概述
本文的内容大部分都是参考了下面这个链接中`Glide`分类的文章：
> [`https://futurestud.io/tutorials/glide-getting-started`](https://futurestud.io/tutorials/glide-getting-started)

为了区分，我们这篇只介绍一些基本用法，掌握这些基本就可以在项目中使用`Glide`了，而关于如何自定义`Target/Glide Module`等高级的用法，之后再进行讨论。
## 二、导入依赖包&加载网络静态图片
在使用前，首先需要在`build.gradle`中引入：
```
dependencies {
    //....
    compile 'com.github.bumptech.glide:glide:3.7.0'
    //...
}
```
下面是`Glide`加载一张网络静态图片最基本的用法：
```
Glide
.with(this)　//传入关联的Context，如果是Activity/Fragment，那么它会根据组件当前的状态来控制请求。
.load("http://i.imgur.com/DvpvklR.png") //需要加载的图片，大多数情况下就是网络图片的链接。
.into(getImageView()); //用来展现图片的ImageView.
```
# 三、`load`的其它用法
在第二章的例子当中，当我们调用完`.with(Context context)`之后，会返回一个`RequestManager`对象，之前我们就是调用它的`load(String string)`方法，除此之外，还提供了下面的这些`load`方法，`load`方法最终会返回一个`DrawableTypeRequest<xxxx>`，而`xxx`就是我们传入的参数的类型：
![](http://upload-images.jianshu.io/upload_images/1949836-bd1c502db8f6716b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
- `load(byte[] model)`，从`byte[]`中读取
- `load(File file)`，从`File`中读取
- `load(Integer resourceId)`，从`resourceId`中读取
- `load(String string)`，从`String`当中读取，这个一般对应于网络图片的链接，这个图片有可能是普通的图片，也可能是一个`Gif`，当我们需要展示一个`Gif`图片时，只需要像加载普通图片一样就可以了，在加载完之后，这个`Gif`会被自动播放。
- `load(T model)`，从任意类型`T`的中读取，这个后面讲到自定义`Model`时再介绍
- `load(Uri uri)`，从`Uri`类型中读取，这个`Uri`必须能够被`UriLoader`识别。
- `loadFromMediaStore(Uri uri)`，从媒体设备的`Uri`中读取，这个方法用来展示一个**本地**媒体视频的缩略图，也就是视频的第一帧，需要注意，这个链接只能是本地的，网络上视频链接地址是无效的。

下面是各个方法加载的例子：
```
    //从byte[]中加载.
    public void loadByteArray(View view) {
        Bitmap sourceBitmap = BitmapFactory.decodeResource(getResources(), R.drawable.book_local);
        ByteArrayOutputStream bArrayOS = new ByteArrayOutputStream();
        sourceBitmap.compress(Bitmap.CompressFormat.PNG, 100, bArrayOS);
        sourceBitmap.recycle();
        byte[] byteArray = bArrayOS.toByteArray();
        try {
            bArrayOS.close();
        } catch (Exception e) {
            e.printStackTrace();
        }
        Glide.with(this)
                .load(byteArray)
                .into(mImageView);
    }
    //从File中加载.
    public void loadFile(View view) {
        String storePath = "mnt/sdcard/book_local.jpg";
        File file = new File(storePath);
        Glide.with(this)
                .load(file)
                .into(mImageView);
    }
    //从resourceId中加载.
    public void loadResourceId(View view) {
        Glide.with(this)
                .load(R.drawable.book_local)
                .into(mImageView);
    }
    //从普通url中加载.
    public void loadNormalUrl(View view) {
        Glide.with(this)
                .load("http://i.imgur.com/DvpvklR.png")
                .into(mImageView);
    }
    //从gif加载.
    public void loadGif(View view) {
        Glide.with(this)
                .load("http://s1.dwstatic.com/group1/M00/66/4D/d52ff9b0727dfd0133a52de627e39d2a.gif")
                .diskCacheStrategy(DiskCacheStrategy.SOURCE) //要加上这句，否则有可能会出现加载很慢，或者加载不出来的情况.
                .into(mImageView);
    }
    //从本地媒体视频中加载.
    public void loadMedia(View view) {
        String storePath = "mnt/sdcard/media.mp4";
        File file = new File(storePath);
        Glide.with(this)
                .load(Uri.fromFile(file))
                .into(mImageView);
    }
```

## 四、占位图片和错误图片
## 4.1 显示占位图片
有时候由于网络原因，导致请求耗时，此时就需要在得到图片资源之前，采用一个占位图片，这样就不会显得界面太空，`placeHolder`就是做这个事的，当我们调用了`into`方法之后，如果需要从网络上获取图片，那么它会先展示`placeHolder`设置的图片，`placeHolder`除了支持传入`resourceId`，还支持直接传入一个`Drawable`对象。
```
    @Override
    public DrawableRequestBuilder<ModelType> placeholder(int resourceId) {
        super.placeholder(resourceId);
        return this;
    }
    @Override
    public DrawableRequestBuilder<ModelType> placeholder(Drawable drawable) {
        super.placeholder(drawable);
        return this;
    }
```
使用`placeHolder`的例子：
```
    public void loadHolder(View view) {
        Glide.with(this)
                .load("http://i.imgur.com/DvpvklR.png")
                .placeholder(R.drawable.book_placeholder)
                .into(mImageView);
    }
```
## 4.2 显示错误图片
有时候，当我们请求网络图片失败时，我们希望给用户一些提示，这时候给它设置一些回调，并在回调当中进行处理。但是一般情况下，我们显示一个表示错误的本地图片就可以了，为了和前面加载时的占位图片区分，它提供了另一个`error()`方法，和`placeHolder`类似，我们可以给它传入一个`resourceId`或者`Drawable`对象。
```
    public void loadHolderError(View view) {
        Glide.with(this)
                .load("http://i.imgur.com/DvpvklR.png") 
                .asGif()  //为了模拟加载失败的情况.
                .placeholder(R.drawable.book_placeholder)
                .error(R.drawable.book_error)
                .into(mImageView);
    }
```
上面为了模拟失败的情况，我们传入了一个`png`的链接，但是指定为加载`asGif()`资源。
我们发现它会先展示`placeHolder`的资源，再展示`error`的资源。
## 4.3 定义图片切换动画
无论是`placeHolder`还是`error`，都会涉及到切换`ImageView`的图片，这时我们可以通过设置一个动画来让这个切换的过程显得不那么突兀，默认情况下动画是开启的，`crossFade`有下面这三个重载方法：
- `crossFade()`：采用默认动画和默认时长。
- `crossFade(int duration)`：采用默认动画，自定义时长。
- `crossFade(int animationId, int duration)`：采用自定义动画，并自定义时长。

```
    public void loadCustomCrossFade(View view) {
        Glide.with(this)
                .load("http://i.imgur.com/DvpvklR.png")
                .placeholder(R.drawable.book_placeholder)
                .crossFade(5000) //改变的时长.
                .into(mImageView);
    }
```

# 4.4 关闭切换动画
当然我们也可以通过`dontAnimate`，来关闭动画，这样在切换的时候就不会出现动画。
```
    public void loadNoCrossFade(View view) {
        Glide.with(this)
                .load("http://i.imgur.com/DvpvklR.png")
                .placeholder(R.drawable.book_placeholder)
                .dontAnimate()
                .into(mImageView);
    }
```
## 五、获得图片资源后，进行裁剪
## 5.1 裁剪
和`Picasso`相比，`Glide`提供了一种更加高效的处理方式，它会把内存和缓存中的图片限制到所展示的`ImageView`的返回内，`Picasso`也可以实现这一功能，但是需要调用`fit()`方法。在使用`Glide`时，如果我们不希望它自动地去匹配`ImageView`的宽高，那么可以调用`override(width, height)`，这样图片资源在被展示之前就会裁剪为指定的大小。

这个方法还有另一种场景，就是当我们确切的知道需要加载多大的图片，但是此时`ImageView`的宽高并没有得到。
## 5.2 定义裁剪的方式
因为有时候采用`override`直接裁剪图片有可能导致只裁剪到了不必要的信息，因此`Glide`还提供了两个类似于`ImageView`中的`scaleType`属性：
- `centerCrop`：使原始的图片的宽高按同等比例放大到`override`所指定的大小，并裁剪到多余的部分，这时最终的图片资源的大小为`(width, height)`
- `fitCenter`：使得原始图片的宽高放大到小于等于`override`所指定的宽高，因此，我们最终得到图片的大小有可能不为`(width, height)`，也就是说不会撑满整个`ImageView`。
下面是相关的代码：

```
    public void loadOverride(View view) {
        Glide.with(this)
                .load(R.drawable.shader_pic)
                .override(20, 20)
                .into(mImageView);
    }

    public void loadOverrideCenterCrop(View view) {
        Glide.with(this)
                .load(R.drawable.shader_pic)
                .override(20, 20)
                .centerCrop()
                .into(mImageView);
    }

    public void loadOverrideFitCenter(View view) {
        Glide.with(this)
                .load(R.drawable.shader_pic)
                .override(20, 20)
                .fitCenter()
                .into(mImageView);
    }
```

## 5.3 和`ImageView`的`scaleType`的关系
由于`ImageView`的展示还需要受`android:scaleType`的影响，这里情况有很多，所以上面裁剪出来，并不是说在`ImageView`里面展示就是`20 * 20`，具体会出现的情况很多，这个之后再专门分析。


# 六、`Gif`图片
在第二节中，我们简单的介绍了如何用`Glide`展示`Gif`图片的展示，下面我们深入地讨论一下其它两点。
## 5.1 `asGif()`
有时候，我们获取的`Gif`图片链接是服务器配置的，因此我们无法知道这个链接到底是不是一个`Gif`图片。这时我们可以配置一个`asGif()`选项，这样`Glide`就会知道我们是需要加载一个`Gif`图片，当这个链接不是`Gif`时，加载就会失败，如果我们定义了`.error(xxx)`，就会展示这个失败的图片。
例如，下面这段代码就会失败：
```
    public void loadHolderError(View view) {
        Glide.with(this)
                .load("http://i.imgur.com/DvpvklR.png")  //传入的是一个静态图片的链接.
                .asGif()  //为了模拟加载失败的情况.
                .placeholder(R.drawable.book_placeholder)
                .error(R.drawable.book_error)
                .into(mImageView);
    }
```
## 5.2 `asBitmap()`
现在讨论另外一种情况，在列表当中，虽然我们获得的是`Gif`链接，但是我们希望这时候不让它播放，这时候就可以指定`asBitmap()`，那么就只会展示`Gif`图片的第一帧。
## 5.3 定义缓存
如果我们使用了`.diskCacheStrategy(DiskCacheStrategy.SOURCE)`，那么`Gif`资源的加载将会更快。

# 六、缓存策略
对于任何一个网络图片加载框架来说，缓存无疑是最关键的部分，`Glide`使用了**内存和磁盘**缓存来避免不必要的网络请求，同时它也提供了一系列地接口让使用者自定义缓存策略。
## 6.1 内存缓存策略
默认情况下，`Glide`会将图片资源缓存到内存当中。而如果使用了`skipMemoryCache(true)`，`Glide`不会将这张图片缓存到内存当中。
有一点需要注意：如果之前对某个指向`url`的图片使用了内存缓存，后面又用`skipMemoryCache(true)`声明想让同一个`url`不缓存到内存中，那么是不会生效的。
## 6.2 磁盘缓存策略
当某个图片变化很快时，我们有可能不需要将它缓存到磁盘当中，我们可以采用`diskCacheStrategy(int mode)`来定义缓存的策略，`Glide`默认情况下既会缓存原始的图片，也会缓存解析后的图片，举个例子，假如服务器上的图片是`1000 * 1000`，而`ImageView`的大小只有`500 * 500`，默认情况下`Glide`会缓存这两个版本的图片，我们可以设定的磁盘缓存类型有下面四种：
- `DiskCacheStrategy.NONE`：不缓存
- `DiskCacheStrategy.SOURCE`：只缓存原始大小的图片，也就是`1000 * 1000`。
- `DiskCacheStrategy.RESULT`：只缓存解析之后的图片，也就是上面`500 * 500`的图片，也就是说假如我们有两个不同大小的`ImageView`，用他们加载同一个`url`的图片，那么最终磁盘当中会有两份不同大小的图片资源。
- `DiskCacheStrategy.ALL`：缓存所有版本的图片。

现在我们介绍一下采用了`DiskCacheStrategy.RESULT`后的缓存文件名的命名策略，具体可以参考下面这篇文章：
> [`https://github.com/bumptech/glide/wiki/Caching-and-Cache-Invalidation`](https://github.com/bumptech/glide/wiki/Caching-and-Cache-Invalidation)

缓存文件的命名会依赖于四个部分：
- `DataFecher`的`getId()`方法的返回值，一般情况下就是`Data Model`的`toString`方法。对于传入`String`类型的网络链接而言，就是`url`，而如果是`File`，那么就是`File`的`path`。
- 目标的宽高，如果定义了`override(width, height)`，那就是指定的数值，默认情况下是`Target`的`getSize()`方法，也就是`ImageView`的宽高。
- 用来加载和缓存图片的`encoders`和`decoders`的`toString`方法。
- 在加载时的可选签名，这个方法在某些特殊的场景下很有用，例如加载的`url`没有变，但是服务器上这个`url`对应的图片资源更新了，我们就可以通过`.signature(xxx)`来刷新缓存。

## 6.3 两种策略的关系
上面我们讨论了两种缓存策略的定义，这两种策略是相互独立的，默认情况下内存的缓存为打开，而磁盘的缓存策略为`DiskCacheStrategy.ALL`。

# 七、请求优先级
有时候，在同一个界面上我们会展示多个图片，而为了用户体验，那么某个图片我们希望先加载出来，这时候就可以采用`.priority(int priority)`来定义请求的优先级，当然，这些优先级只是给`Glide`作为参考，因为还涉及到图片的大小，服务器的响应事件和网络环境等因素，最后图片展示的顺序并不一定是根据优先级来的，可选的优先级包括：
- `Priority.LOW`
- `Priority.NORMAL`
- `Priority.HIGH`
- `Priority.IMMEDIATE`

# 八、缩略图
前面，我们介绍了`placeHolder`，它可以指定一个本地资源，用来在网络资源加载完成之前进行展示，而`thumbnails`则可以认为是一个动态的`placeHolder`，与`placeHolder`不同，我们**可以给它指定一个网络图片链接**。如果这个缩略图请求在`load`请求之前返回那么，那么会先展示这个图片，等到`load`请求返回之后，这个图片就会消失。假如缩略图的请求在`load`请求之后返回，那么请求结果会被丢弃掉。
`Glide`提供了两种指定缩略图的方式：
## 8.1 `Simple Thumbnails`
```
    public void loadScaleThumbnail(View view) {
        Glide.with(this)
                .load("http://i.imgur.com/DvpvklR.png")
                .diskCacheStrategy(DiskCacheStrategy.NONE)
                .thumbnail(0.1f)
                .into(mImageView);
    }
```
## 8.2 `Complete Different Request`
`thumbnails`还支持传入一个`Glide Request`作为参数，这个请求和原本的请求是相互独立的，我们可以给它指定一个完全不同的`url`、缓存策略、大小等等。
```
    public void loadRequestThumbnail(View view) {
        DrawableRequestBuilder<String> thumbnailRequest = Glide
                .with(this)
                .load("http://i.imgur.com/DvpvklR.png");
        Glide.with(this)
                .load("http://s1.dwstatic.com/group1/M00/66/4D/d52ff9b0727dfd0133a52de627e39d2a.gif")
                .diskCacheStrategy(DiskCacheStrategy.NONE)
                .thumbnail(thumbnailRequest)
                .into(mImageView);
    }
```
