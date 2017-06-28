---
title: Glide 知识梳理(2) - 自定义Target
date: 2017-03-12 00:12
categories : Glide 知识梳理
---
# 一、概述
首先，我们回顾一下`Glide`的基本用法，我们的最后一步都是调用`into(ImageView imageView)`，通过断点，我们可以看到正是这一步触发了图片的请求：
![](http://upload-images.jianshu.io/upload_images/1949836-2d78720ef708ead9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
这个`ImageView`最终会通过`buildTarget`方法，封装在`GlideDrawableImageViewTarget`当中，然后调用`GenericRequestBuilder#into`方法发起请求，`Glide`一直跟踪这个`Target`，并在获得图片资源之后，通知`Target`来更新它内部持有的`ImageView`的引用。
这个过程就好像是我们平时请求网络时，会传入一个`Callback`，等到异步的操作执行完毕后，通过`Callback`传回请求的资源来更新`UI`。
通过源码，可以看到和`Target`相关的类有：
- `Target`
 - `BaseTarget`
   - `SimpleTarget`：`AppWidgetTarget`、`PreloadTatget`、`NotificationTarget`
   - `ViewTarget`
     - `ImageViewTarget`：`GlideDrawableImageViewTarget`、`BitmapImageViewTarget`、`DrawableImageViewTarget`

今天这篇文章，我们就来学习一下`SimpleTarget`和`ViewTarget`的用法，主要参考了下面链接中的文章：
> [`https://futurestud.io/tutorials/glide-callbacks-simpletarget-and-viewtarget-for-custom-view-classes`](https://futurestud.io/tutorials/glide-callbacks-simpletarget-and-viewtarget-for-custom-view-classes)

# 二、`SimpleTarget`
`SimpleTarget`可以看作是请求的回调，我们在回调当中进行处理，而不是传入`ImageView`让`Glide`去负责更新。
## 2.1 基本用法
先看一下`SimpleTarget`的用法：
```
    public void loadSimpleTarget(View view) {
        MySimpleTarget mySimpleTarget = new MySimpleTarget();
        Glide.with(this)
                .load(R.drawable.book_url)
                .into(mySimpleTarget);
    }

    private class MySimpleTarget extends SimpleTarget<GlideDrawable> {
        
        @Override
        public void onResourceReady(GlideDrawable resource, GlideAnimation<? super GlideDrawable> glideAnimation) {
            mImageView.setImageDrawable(resource.getCurrent());
        }
    }
```
我们首先定义了一个`SimpleTarget`，然后把它通过`into`方法传入。这样当`Glide`去服务器请求图片成功之后，它会把请求到的图片资源作为`GlideDrawable `传递回来，你可以使用这个`GlideDrawable.getCurrent()`进行自己想要的操作。
关于上面`SimpleTarget`的使用需要知道两点：
- 由于我们把`SimpleTarget`定义成了局部变量，那么它很有可能会被回收，一旦它被回收，那么我们收不到任何的回调了。
- 我们在`with`中传入了`Activity`的`Context`，那么`Glide`就会监听`Activity`生命周期的变化，当`Activity`退到后台之后，停止该请求。如果你希望它独立于`Activity`的生命周期，那么需要传入一个`Application`的`Context`。

## 2.2 设置资源的大小
当我们传入`ImageView`时，`Glide`会根据`ImageView`的大小来自动调整缓存的图片资源大小，而当我们使用`Target`的时候，并没有提供这个条件来给`Glide`，因此，为了缩短处理时间和减少内存，我们可以按下面的方法来指定缓存的大小：
```
    public void loadSimpleTarget(View view) {
        MySimpleTarget mySimpleTarget = new MySimpleTarget(50, 50);
        Glide.with(this)
                .load(R.drawable.book_url)
                .into(mySimpleTarget);
    }

    private class MySimpleTarget extends SimpleTarget<GlideDrawable> {

        public MySimpleTarget(int width, int height) {
            super(width, height);
        }

        @Override
        public void onResourceReady(GlideDrawable resource, GlideAnimation<? super GlideDrawable> glideAnimation) {
            mImageView.setImageDrawable(resource.getCurrent());
        }
    }
```
# 三、`ViewTarget`
在上面一节中，我们展示了如何通过设置回调来获得最终的`Bitmap`，但是上面的方法就是有一个缺陷：回调中只能拿到最终请求到的资源。我们需要持有`View`的全局对象，这样才能在收到回调之后更新它，并且，`Glide`无法根据`View`的实际宽高来决定缓存图片的大小。
`ViewTarget`就提供了这样一种方案：我们在构造`Target`时就传入自定义的`View`，这样在回调时就可以通过它来更新`UI`。
它的原理其实和我们开头说到的传入`ImageView`的原理类似，就是通过传入`Target`的方式，但是`ViewTarget`会持有需要更新的`View`实例，这样在回调时候，我们就能执行自己需要的操作了，下面是使用`ViewTarget`的例子：
首先，定义一个自定义的`View`：
```
public class CustomView extends LinearLayout {

    private ImageView mImageView;
    private TextView mTextView;

    public CustomView(Context context) {
        super(context);
    }
    public CustomView(Context context, AttributeSet attrs) {
        super(context, attrs);
    }
    public CustomView(Context context, AttributeSet attrs, int defStyleAttr) {
        super(context, attrs, defStyleAttr);
    }
    
    @Override
    protected void onFinishInflate() {
        super.onFinishInflate();
        init();
    }

    private void init() {
        mTextView = (TextView) findViewById(R.id.tv_custom_result);
        mImageView = (ImageView) findViewById(R.id.iv_custom_result);
    }

    public void setResult(Drawable drawable) {
        mTextView.setText("load success");
        mImageView.setImageDrawable(drawable);
    }
}
```
在布局当中这么定义它：
```
    <com.example.lizejun.repoglidelearn.CustomView
        android:id="@+id/cv_result"
        android:orientation="vertical"
        android:layout_width="match_parent"
        android:layout_height="wrap_content">
        <TextView
            android:id="@+id/tv_custom_result"
            android:layout_width="match_parent"
            android:layout_height="30dp"
            style="@style/style_tv_normal"/>
        <ImageView
            android:id="@+id/iv_custom_result"
            android:layout_width="200dp"
            android:layout_height="200dp" />
    </com.example.lizejun.repoglidelearn.CustomView>
```
之后，我们定义一个`ViewTarget`，并加载它：
```
    public void loadViewTarget(View view) {
        CustomView customView = (CustomView) findViewById(R.id.cv_result);
        MyViewTarget myViewTarget = new MyViewTarget(customView);
        Glide.with(this)
                .load(R.drawable.book_url)
                .into(myViewTarget);
    }

    private class MyViewTarget extends ViewTarget<CustomView, GlideDrawable> {

        public MyViewTarget(CustomView customView) {
            super(customView);
        }

        @Override
        public void onResourceReady(GlideDrawable resource, GlideAnimation<? super GlideDrawable> glideAnimation) {
            view.setResult(resource.getCurrent());
        }
    }
```
它的整个步骤为：
- 首先获得这个自定义`View`实例。
- 之后把这个自定义`View`作为`ViewTarget`的构造函数的参数，新建一个`ViewTarget`实例。
- 把这个`ViewTarget`通过`into()`方法传给`Glide`。
- 等待`Glide`请求完毕，那么会回调`ViewTarget`中的`onResourceReady`方法，该`Target`中有我们传入的自定义`View`，这样，我们就可以调用这个自定义`View`内部的方法。

# 四、小结
- 从继承树上来看，`SimpleTarget`和`ViewTarget`是层次最低的可实现类，也是我们平时开发中比较常用的类。
- 这两者的区别就是`ViewTarget`内部的实现更加复杂，它会持有`View`的引用，并通过内部的`SizeDeterminer`计算`View`的宽高来提供给`Glide`作为参考，`SimpleTarget`则不会去处理这些逻辑，我们需要手动的指定一个宽高，所以，我们需要根据不同的使用场景来决定继承于哪个`Target`来实现自己的业务逻辑。
- 除了`SimpleTarget`和`ViewTarget`，`Glide`还提供了继承于它们的`Target`来简化我们的操作。例如，更新通知栏和桌面插件，从源码来看，它们是继承于`SimpleTarget`，其最基本的原理和我们自定义`SimpleTarget`都是相同的，只是在回调里面调用`AppWidgetManager/NotificationManager`增加了更新相应组件的操作。在这里就不多介绍了，有需要了解的可以看下面这篇文章：

> [`https://futurestud.io/tutorials/glide-loading-images-into-notifications-and-appwidgets`](https://futurestud.io/tutorials/glide-loading-images-into-notifications-and-appwidgets)
