---
title: Glide 知识梳理(4) - 自定义animate
date: 2017-03-12 23:17
categories : Glide 知识梳理
---
# 一、概述
在之前基础用法的文章中，我们介绍了使用`crossFade`来进行`placeHolder`和要加载的图片之间渐进渐出的动画，今天，我们来介绍一个更加高级的用法 - `animate()`。

`animate`有以下两个重载方法，我们可以给它指定一个`R.anim`下的动画资源文件，或者传入一个实现了`ViewPropertyAnimation.Animator`接口的实例，来自定义自己的动画。
- `public DrawableRequestBuilder<ModelType> animate(int animationId)`
- `public DrawableRequestBuilder<ModelType> animate(ViewPropertyAnimation.Animator animator)`

# 二、示例
## 2.1 在`xml`中定义动画
在`xml`中使用自定的动画很简单，
首先，我们在`R.anim`文件夹下定义一个动画资源文件，`R.anim.glide_animate`
```
<?xml version="1.0" encoding="utf-8"?>
<set xmlns:android="http://schemas.android.com/apk/res/android"
    android:duration="500">
    <alpha
        android:fromAlpha="0.5"
        android:toAlpha="1.0" />
    <scale
        android:fromXScale="0.5"
        android:toXScale="1.0"
        android:fromYScale="0.5"
        android:toYScale="1.0"
        android:pivotY="50%"
        android:pivotX="50%"/>
</set>
```
然后，我们调用`animate`把这个资源文件的`id`传进去，这样，当图片加载完成之后，就会以一个慢慢放大且渐显的方式出现了。
```
    public void loadAnimate(View view) {
        Glide.with(this)
                .load("http://i.imgur.com/DvpvklR.png")
                .diskCacheStrategy(DiskCacheStrategy.NONE)
                .animate(R.anim.glide_animate)
                .into(mImageView);
    }
```
## 2.2 通过`ViewPropertyAnimation.Animator`定义动画
当我们的容器是一个`ImageView`时，用上面的方式是最方便的。然后回想一下，之前我们介绍过的自定义`Target`文章中，我们谈到了`ViewTarget`，也就是我们定义了一个自定义的`View`，那么这时候如果我们希望这个自定义`View`中的各个组件可以用不同的动画方式展现出来，那么上面这种用`xml`定义动画执行过程就不适用了，下面我们展示一下继承于`ViewPropertyAnimation.Animator`来进行动画。
```
    private class MyAnimator implements ViewPropertyAnimation.Animator {

        @Override
        public void animate(View view) {
            final View finalView = view;
            ValueAnimator valueAnimator = ValueAnimator.ofFloat(0, 1);
            valueAnimator.addUpdateListener(new ValueAnimator.AnimatorUpdateListener() {

                @Override
                public void onAnimationUpdate(ValueAnimator animation) {
                    float value = (float) animation.getAnimatedValue();
                    finalView.setScaleX((float) (0.5 + 0.5 * value));
                    finalView.setScaleY((float) (0.5 + 0.5 * value));
                    finalView.setAlpha(value);
                }
            });
            valueAnimator.start();
        }
    }
```
然后，我们实例化一个`MyAnimator`，通过`animate()`传入：
```
    public void loadAnimator(View view) {
        MyAnimator myAnimator = new MyAnimator();
        Glide.with(this)
                .load("http://i.imgur.com/DvpvklR.png")
                .diskCacheStrategy(DiskCacheStrategy.NONE)
                .animate(myAnimator)
                .into(mImageView);
    }
```
