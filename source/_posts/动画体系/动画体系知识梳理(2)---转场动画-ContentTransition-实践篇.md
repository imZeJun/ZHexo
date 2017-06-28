---
title: 动画体系知识梳理(2) - 转场动画 ContentTransition 实践篇
date: 2017-04-08 11:57
categories : 动画体系知识梳理
---
# 一、概述
在 [转场动画理论篇](http://www.jianshu.com/p/510e415c56f2) 中，我们介绍了`Content Transition`的基本理论，今天，我们来一起学习`Content Transition`使用当中的细节问题。

# 二、基本使用
## 2.1 启动`Activity`方式
如果我们希望在`Activity`切换的时候加上`Content Transition`动画，那么需要使用下面的启动方式：
```
    private void startTargetActivity(int position) {
        if (position == 0) {
            List<Pair> pairs = new ArrayList<>();
            //1.得到ActivityOptionsCompact对象
            ActivityOptionsCompat compat = ActivityOptionsCompat.makeSceneTransitionAnimation(mActivity, pairs.toArray(new android.support.v4.util.Pair[pairs.size()]));
            Intent intent = new Intent(mActivity, CTTargetActivity.class);
            //2.调用第1步生成的ActivityOptionsCompact的toBundle方法
            mActivity.startActivity(intent, compat.toBundle());
        }
    }
```
`makeSceneTransitionAnimation`有两个重载方法：
- `makeSceneTransitionAnimation(Activity activity, View sharedElement, String sharedElementName)`
- `makeSceneTransitionAnimation(Activity activity, Pair<View, String>... sharedElements)`

`Activity`就对应于当前所在的`Activity`，而后面的参数则用于`Shared Element Transition`，这一节我们介绍的是`Content Transition`，因此，直接传一个空的数组就可以了。

# 2.2 设置对应的`Transition`
在基础理论篇中，我们说过在`Activity`的切换过程中，每个`Acitivity`包括了四种`Transition`，我们可以通过`Acitivity`所在`Window`的`setxxxTransition`方法来设置：
- `getWindow().setExitTransition`
- `getWindow().setEnterTransition`
- `getWindow().setReturnTransition`
- `getWindow().setReenterTransition`

例如，我们**被启动的`Acitivity`**设置`Enter`和`Return`，那么需要像下面这样：
```
public class CTTargetActivity extends Activity {

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_ct_target);
        setUpNormalTransition();
    }

    private void setUpNormalTransition() {
        Window window = getWindow();
        LogFadeTransition transition = new LogFadeTransition();
        window.setEnterTransition(transition);
        window.setReturnTransition(transition);
    }
}
```
而如果我们希望对某个过程加上多个`Transition`，那么可以传入一个`TransitionSet`：
```
    private void setUpNormalTransition() {
        Window window = getWindow();
        LogFadeTransition transition = new LogFadeTransition();
        Slide slide = new Slide();
        TransitionSet set = new TransitionSet();
        set.addTransition(slide);
        set.addTransition(transition);
        window.setEnterTransition(set);
        window.setReturnTransition(set);
    }
```
## 2.3 `Transition`各方法回调验证
我们验证一下在上一篇文章中所谈到的`captureXXX`、`onAppear`、`onDisappear`的调用情况，下面是我们被启动`CTTargetActivity`的布局，为了在`Transition`的回调方法中，确定是哪个`View`，给每个`View`都加上了`Tag`：
```
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout
    xmlns:android="http://schemas.android.com/apk/res/android"
    android:tag="rootView"
    android:orientation="vertical"
    android:layout_width="match_parent"
    android:layout_height="match_parent">
    <RelativeLayout
        android:id="@+id/ll_header"
        android:tag="ll_header"
        android:gravity="center"
        android:layout_width="match_parent"
        android:layout_height="150dp">
        <ImageView
            android:id="@+id/iv_bg"
            android:tag="iv_bg"
            android:src="@drawable/ic_bg"
            android:layout_width="match_parent"
            android:layout_height="match_parent" />
        <ImageView
            android:id="@+id/iv_header"
            android:tag="iv_header"
            android:layout_centerInParent="true"
            android:src="@drawable/ic_1"
            android:layout_width="50dp"
            android:layout_height="50dp"
            android:layout_marginBottom="10dp"/>
        <TextView
            android:id="@+id/tv_header"
            android:tag="tv_header"
            android:layout_alignBottom="@id/iv_header"
            android:layout_centerHorizontal="true"
            android:layout_alignParentBottom="true"
            android:layout_marginBottom="30dp"
            android:text="我是标题"
            android:textSize="18sp"
            android:textColor="@android:color/white"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content" />
    </RelativeLayout>
    <TextView
        android:id="@+id/tv_content"
        android:tag="tv_content"
        android:layout_width="match_parent"
        android:padding="30dp"
        android:textColor="@android:color/white"
        android:background="#1F000000"
        android:text="我是内容;我是内容;我是内容;我是内容;我是内容;我是内容;我是内容;我是内容;我是内容;我是内容;我是内容;我是内容;我是内容;我是内容;我是内容;我是内容;我是内容;我是内容;我是内容;我是内容;我是内容;我是内容;我是内容;我是内容;"
        android:layout_height="match_parent"/>
</LinearLayout>
```
当我们从别的`Activity`通过`2.1`的方式启动它时，效果是这样的：
![](http://upload-images.jianshu.io/upload_images/1949836-b718c6894024aad4.gif?imageMogr2/auto-orient/strip)
当`CTTargetActivity`被启动的时候，打印的`Log`为，此时会调用每个`transitionView`的`onAppear`方法来获得一个渐显的`Animator`：
![](http://upload-images.jianshu.io/upload_images/1949836-7368faad34576583.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
而如果我们从`CTTargetActivity`返回到之前启动它的那个`Activity`，这时候打印的`Log`为，此时会调用每个`transitionView`的`onDisappear`方法来获得一个渐隐的`Animator`：
![](http://upload-images.jianshu.io/upload_images/1949836-71fd3a1c9b60ed01.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
## 2.4 指定`transitionView`
在默认情况下，需要执行`Transition`的`View`是通过系统遍历得到的，如果我们希望改变这一集合，那么可以通过`Transition`的`addXXX`和`excludeXXX`方法：
- `addTarget`，默认情况下是通过遍历`View`树的方式得到`transitionViews`，而如果我们使用了`addTarget`方法，那么会使得这一过程失效，最后变化的`transitionViews`只是`addTarget`所指定的`View`：
![](http://upload-images.jianshu.io/upload_images/1949836-30f718bdde34048b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
例如我们像下面这样操作：
```
    private void setUpNormalTransition() {
        Window window = getWindow();
        LogFadeTransition transition = new LogFadeTransition();
        transition.addTarget(R.id.iv_bg);
        window.setEnterTransition(transition);
        window.setReturnTransition(transition);
    }
```
那么此时就只会对`iv_bg`进行变换：
![](http://upload-images.jianshu.io/upload_images/1949836-7674350103ab09b3.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

- `excludeChildren`：
![](http://upload-images.jianshu.io/upload_images/1949836-328b02c3133d8cad.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
把这个`View`的所有子`View`从`transition`的列表中去除，例如，类似于`ListView`，如果我们希望他的每个`Item`不做变换，那么就可以使用这个标志位。
- `excludeTarget`：
![](http://upload-images.jianshu.io/upload_images/1949836-e188be4fa9fc40fa.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
排除某个具体的`View`。

## 2.5 `Transition`过程监听
某些时候，我们希望在`Transition`完成之后再进行某些操作，那么可以通过下面这个方法来监听整个过程：
```
contentTransition.addListener(new Transition.TransitionListener() {
            @Override
            public void onTransitionStart(Transition transition) {}
            @Override
            public void onTransitionEnd(Transition transition) {}
            @Override
            public void onTransitionCancel(Transition transition) {}
            @Override
            public void onTransitionPause(Transition transition) {}
            @Override
            public void onTransitionResume(Transition transition) {}
});
```
# 三、自定义`Transition`
经过这么长篇幅的介绍，相信大家一定对`Transition`有了一定的了解了，下面，我们就开始设置自己的`Transition`，这是我们最终的效果：
![](http://upload-images.jianshu.io/upload_images/1949836-d4bed815588bd736.gif?imageMogr2/auto-orient/strip)
在我们的`Transition`中，根据`View`的`Tag`来进行区分动画：
```
public class CustomContentTransition extends Visibility {

    public static final String TAG = "CustomContentTransition";

    @Override
    public void captureStartValues(TransitionValues transitionValues) {
        super.captureStartValues(transitionValues);
    }

    @Override
    public void captureEndValues(TransitionValues transitionValues) {
        super.captureEndValues(transitionValues);
    }

    @Override
    public Animator onAppear(ViewGroup sceneRoot, final View view, TransitionValues startValues, TransitionValues endValues) {
        Animator animator = null;
        String viewTag = (String) view.getTag();
        if ("transition_reveal".equals(viewTag)) {
            animator = createRevealAnimator(view, false);
        } else if ("transition_translationY".equals(viewTag)) {
            animator = createTranslateYAnimator(view, 200, 0, false);
        } else if ("transition_scale".equals(viewTag)) {
            animator = createScaleAnimator(view, .8f, 1f, false);
        }
        return animator;
    }

    @Override
    public Animator onDisappear(ViewGroup sceneRoot, View view, TransitionValues startValues, TransitionValues endValues) {
        Animator animator = null;
        String viewTag = (String) view.getTag();
        if ("transition_reveal".equals(viewTag)) {
            animator = createRevealAnimator(view, true);
        } else if ("transition_translationY".equals(viewTag)) {
            animator = createTranslateYAnimator(view, 0, 200, true);
        } else if ("transition_scale".equals(viewTag)) {
            animator = createScaleAnimator(view, 1f, .8f, true);
        }
        return animator;
    }

    private Animator createScaleAnimator(final View view, float startValue, final float endValues, final boolean dismiss) {
        ValueAnimator animator = ValueAnimator.ofFloat(startValue, endValues);
        animator.addUpdateListener(new ValueAnimator.AnimatorUpdateListener() {
            @Override
            public void onAnimationUpdate(ValueAnimator animation) {
                float scale = (Float) animation.getAnimatedValue();
                float faction = animation.getAnimatedFraction();
                view.setScaleX(scale);
                view.setScaleY(scale);
                view.setAlpha(dismiss ? (1 - faction) : faction);
            }
        });
        return animator;
    }

    private Animator createTranslateYAnimator(final View view, final int startValue, int endValue, final boolean dismiss) {
        ValueAnimator animator = ValueAnimator.ofInt(startValue, endValue);
        animator.addUpdateListener(new ValueAnimator.AnimatorUpdateListener() {

            @Override
            public void onAnimationUpdate(ValueAnimator animation) {
                int translationY = (Integer) animation.getAnimatedValue();
                float faction = animation.getAnimatedFraction();
                view.setTranslationY(translationY);
                view.setAlpha(dismiss ? (1 - faction) : faction);
            }
        });
        return animator;
    }

    private Animator createRevealAnimator(final View view, boolean dismiss) {
        int cx = (view.getLeft() + view.getRight()) / 2 - 270;
        int cy = (view.getTop() + view.getBottom()) / 2 - 120;
        float maxRadius = (float) Math.hypot(view.getWidth(), view.getHeight());
        float startRadius = dismiss ? maxRadius : 0;
        float finalRadius = dismiss ? 0 : maxRadius;
        return ViewAnimationUtils.createCircularReveal(view, cx, cy, startRadius, finalRadius);
    }

}
```
在布局当中，给`View`添加不同的`Tag`：
```
<LinearLayout
    xmlns:android="http://schemas.android.com/apk/res/android"
    android:tag="rootView"
    android:orientation="vertical"
    android:layout_width="match_parent"
    android:layout_height="match_parent">
    <RelativeLayout
        android:id="@+id/ll_header"
        android:tag="ll_header"
        android:gravity="center"
        android:layout_width="match_parent"
        android:layout_height="150dp">
        <ImageView
            android:id="@+id/iv_bg"
            //***
            android:tag="transition_reveal"
            android:src="@drawable/ic_bg"
            android:layout_width="match_parent"
            android:layout_height="match_parent" />
        <ImageView
            android:id="@+id/iv_header"
            //***
            android:tag="transition_scale"
            android:layout_centerInParent="true"
            android:src="@drawable/ic_1"
            android:layout_width="50dp"
            android:layout_height="50dp"
            android:layout_marginBottom="10dp"/>
        <TextView
            android:id="@+id/tv_header"
            //***
            android:tag="transition_scale"
            android:layout_alignBottom="@id/iv_header"
            android:layout_centerHorizontal="true"
            android:layout_alignParentBottom="true"
            android:layout_marginBottom="30dp"
            android:text="我是标题"
            android:textSize="18sp"
            android:textColor="@android:color/white"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content" />
    </RelativeLayout>
    <TextView
        android:id="@+id/tv_content"
        //***
        android:tag="transition_translationY"
        android:layout_width="match_parent"
        android:padding="30dp"
        android:textColor="@android:color/white"
        android:background="#1F000000"
        android:text="我是内容;我是内容;我是内容;我是内容;我是内容;我是内容;我是内容;我是内容;我是内容;我是内容;我是内容;我是内容;我是内容;我是内容;我是内容;我是内容;我是内容;我是内容;我是内容;我是内容;我是内容;我是内容;我是内容;我是内容;"
        android:layout_height="match_parent"/>
</LinearLayout>
```
最后，我们需要在`Activity`中把这个自定义的`Transition`设置给`Window`：
```
    private void setUpNormalTransition() {
        Window window = getWindow();
        CustomContentTransition contentTransition = new CustomContentTransition();
        contentTransition.addTarget(R.id.iv_bg);
        contentTransition.addTarget(R.id.tv_header);
        contentTransition.addTarget(R.id.iv_header);
        contentTransition.addTarget(R.id.tv_content);
        contentTransition.setDuration(500);
        window.setEnterTransition(contentTransition);
        window.setReturnTransition(contentTransition);
    }
```
# 五、总结
至此，我们对于`Content Transition`的学习就结束了，转场动画`Content Transition`其实并不难，主要就是要知道`VISIBILITY`中各回调的调用时机，以及个参数的含义，然后通过`onAppear`和`onDisappear`返回自定义的`Animator`。
