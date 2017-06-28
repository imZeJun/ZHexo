---
title: 动画体系知识梳理(1) - 转场动画 ContentTransition 理论篇
date: 2017-04-07 14:52
categories : 动画体系知识梳理
---
# 一、概述
在`Android 5.0`当中，`Google`基于`Android 4.4`中的`Transition`框架引入了转场动画，**设计转场动画的目的，在于让`Activity`之间或者`Fragment`之间的切换更加自然**，其根本原因在于界面间切换时的动画不再是以`Activity`或者`Fragment`的整个布局作为切换时动画的执行单元，而是将动画的执行单元细分到了`View`。目前提供的转场动画分为两种：
- `Content Transition`：用于两个界面之间非共享的`View`。
- `Shared Element Transition`：用于两个界面之间需要共享的`View`。

# 二、什么是`Transition`
## 2.1 `Transition`的基本概念
在学习`Content Transition`之前，我们先对转场动画所依赖的`Transition`框架做一个简要的介绍，这个框架是围绕着两个概念**`Scene`（场景）和`Transition`（变换）**来建立的，在后面我们会多次提到它：
![](http://upload-images.jianshu.io/upload_images/1949836-5f45b86758e1ca9e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
- 场景（`Scene`）：表示`UI`所对应的状态，一般来说，会有两个场景：起点场景和终点场景，在这两个场景当中，`UI`有可能会有不同的状态。在上图当中，`SceneA`和`SceneB`就是两个不同的场景，`ViewA`在两个场景中对应的状态分别为`VISIBLE`和`INVISIBLE`。
- 变换（`Transition`）：用来定义两个场景之间切换的规则，当场景发生发生变换时，`Transition`需要做的有两点：
 - 确定`View`在起点场景和终点场景的状态。
 - 创建`View`从终点场景切换到终点场景所需的`Animator`。

## 2.2 `Transition`的简单例子
下面，我们通过一个简单的例子，对上面的概念有一个直观的感受：
```
public class ExampleActivity extends Activity implements View.OnClickListener {
    private ViewGroup mRootView;
    private View mRedBox, mGreenBox, mBlueBox, mBlackBox;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        mRootView = (ViewGroup) findViewById(R.id.layout_root_view);
        mRootView.setOnClickListener(this);

        mRedBox = findViewById(R.id.red_box);
        mGreenBox = findViewById(R.id.green_box);
        mBlueBox = findViewById(R.id.blue_box);
        mBlackBox = findViewById(R.id.black_box);
    }

    @Override
    public void onClick(View v) {
        TransitionManager.beginDelayedTransition(mRootView, new Fade());
        toggleVisibility(mRedBox, mGreenBox, mBlueBox, mBlackBox);
    }

    private static void toggleVisibility(View... views) {
        for (View view : views) {
            boolean isVisible = view.getVisibility() == View.VISIBLE;
            view.setVisibility(isVisible ? View.INVISIBLE : View.VISIBLE);
        }
    }
}
```
- 第一步：通过`beginDelayedTranstion`传入场景对应布局的根节点（`mRootView`）以及场景变换的规则（`Fade`），此时系统理解调用`Transition`的`captureStartValues`方法，来确定场景当中所有子`View`的`visibility`。
- 第二步：当`beginDeleyedTransition`返回后，我们将子`View`设置为不可见。
- 第三步：在下一帧，系统调用`Transtion`的`captureEndValues()`方法获取场景当中所有子`View`的可见性。
- 第四步：这时候系统发现在起始场景中`View`是`VISIBLE`的，而在终点场景中它变为了`INVISIBLE`，那么`Fade Transition`就会根据这些信息创建并返回`AnimatorSet`，用它来将那些发生变化的`View`的`alpha`值渐变为`0`，而不是直接设为不可见。
- 第五步：系统启动这个`Animator`，使得这些`View`慢慢隐藏。


## 2.3 `Transition`小结
我们可以总结出`Transition`的两个特点：
- `Animator`对于开发者而言是抽象的，开发者设置`View`的起始值和最终值，`Transition`会根据这两者的差异，自动地创建切换的`Animator`。
- 可以随时通过替换`Transition`来改变切换的规则。

# 三、`Content Transition`基本概念
## 3.1 旧的界面切换动画
回忆一下，在`5.0`之前：
- 给`Activity`之间的切换添加动画，在启动`Activity`的地方加上`overridePendingTransition`
- 给`Fragment`之间的切换添加动画，通过`FragmentTransation`的`setCustomAnimation`。

这两种方式都有一个共同的特点，那就是它们**都是将`Activity`所在的窗口或`Fragment`所对应的布局作为切换动画的执行单元**。

## 3.2 新的界面切换动画
在新的切换方式当中，可以将布局中的每个`View`作为切换的执行单元，我们以`Activity`之间的切换为例。
### 3.2.1 启动`BActivity`
在`AActivity`启动中`BActivity`，这时候就会涉及到四种`Scene`和两种`Transition`：
![](http://upload-images.jianshu.io/upload_images/1949836-0c30ff98e872ae63.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
- `AActivity's Exit Transition`：它定义了`AActivity`中的元素如何从`VISIBLE`（起点场景）变为`INVISIBLE`（终点场景）。
- `BActivity's Enter Transition`：它定义了`BActivity`中的元素如果从`INVISIBLE`（起点场景）变为`VISIBLE`（终点场景）。

#### 3.2.1.1 确定需要执行`Transition`的`View`
整个`Transition`的第一步，就是先要**确定当前界面中需要执行`Transition`的动画切换单元**，这一过程是通过对整个`View`树进行递归调用得到的，而递归的逻辑在`ViewGroup`当中：
```
public void captureTransitioningViews(List<View> transitioningViews) {
        if (getVisibility() != View.VISIBLE) {
            return;
        }
        if (isTransitionGroup()) {
            transitioningViews.add(this);
        } else {
            int count = getChildCount();
            for (int i = 0; i < count; i++) {
                View child = getChildAt(i);
                child.captureTransitioningViews(transitioningViews);
            }
        }
}
```
而在`View`中，该方法为：
```
public void captureTransitioningViews(List<View> transitioningViews) {
        if (getVisibility() == View.VISIBLE) {
            transitioningViews.add(this);
        }
}
```
由此可见，所有需要变换的`ViewGroup/View`都保存在`transitioningViews`当中，关于这个集合的构成依据以下三点：
- 节点不可见，那么它以及它的所有子节点都不加入集合。
- 节点的`isTransitionGroup()`标志位为`true`，那么把它和它的所有子节点当成一个变换单元加入到集合当中。
- 除了以上两种情况，那么`View`树的所有叶子节点都加入到集合当中。

其中`isTransitionGroup()`的值我们可以通过`setTransitionGroup(boolean flag)`来改变，如果在场景当中用到了`WebView`，而我们希望将它作为一个整体进行变换，那么应当加上这个标志位。
除了系统默认的遍历，我们还可以通过`Transition`的`added`和`excluded`来改变这个集合。
#### 3.2.1.2 `Exit Transition`的执行过程
下面，我们以`AActivity`的`Exit Transition`为例，描述一下它整个的执行过程：
 - 第一步：系统遍历`AActivity`的`View`树，并决定在`exit transition`运行时需要变换的`View`，把它们放在集合当中，也就是我们在`3.2.1.1`中所说的`transitionViews`。
 - 第二步：`AActivity`的`Exit Transition`获取集合中`View`的起始状态，调用的是`captureStartValues`方法。
 - 第三步：将集合中的`View`设为`INVISIBLE`。
 - 第四步：在下一帧时，`Exit Transition`获取集合中`View`的终点状态，调用的是`captureEndValues`方法。
 - 第五步：`Exit Transition`根据第二步中的起始状态和终点状态，创建一个`Animator`，并执行这个`Animator`，由于是从`VISIBLE`变为`INVISIBLE`，因此，是通过`onDisappear`方法得到`Animator`。

#### 3.2.1.3 `Enter Transition`的执行过程。
`BActivity`的`Enter Transition`和`AActvity`的`Exit Transition`类似，只不过第三步操作是从`INVISIBLE`到`VISIBLE`。

### 3.2.2 从`BActivity`返回
而当我们从`BActivity`返回到`AActivity`，那么就会涉及到下面四种`Scene`和两种`Transition`：
![](http://upload-images.jianshu.io/upload_images/1949836-e321846ec6f9a8c7.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
- `BActivity's Return Transition`
- `AActivity's Reenter Transition`

其原理和上面是相同的，就不多介绍了。

### 3.2.3 小结
无论是`AActivity`启动`BActivity`，还是`BActivity`返回到`AActivity`，当`View`的可见性不断切换的时候，系统能保证根据状态信息来创建所需的动画。很显然，所有的`Content transition`对象都需要能够捕捉并记录`View`的起始状态和终点状态，幸运的是，抽象类`Visiblity`已经帮我们做了，我们只需要实现`onAppear`和`onDisappear`方法，在里面创建一个`Animator`来定义进入和退出场景的`View`的动画，系统默认提供了三种`Transition` - `Fade、Slide、Explode`，下面我们在分析`Fade`源码的时候，会详细解释这一过程。

## 3.3 `Content Transition`和`Shared Element Transition`
在上面的讨论当中，我们是从切换的角度来考虑的，而如果我们从`Transition`的角度来看，那么每个`Transition`又可以细分为两类：
- `content transitions`：定义了`Activity`非共享`View`进入和退出场景的方式。
- `shared element transitions`：定义了`Acitivity`共享`View`进入和退出场景的方法。

## 3.4 例子
下面，我们以一个视频来解释一下上面谈到的四个`Transition`：
![](http://upload-images.jianshu.io/upload_images/1949836-7b392f09a9cb6e21.gif?imageMogr2/auto-orient/strip)
在这个视频当中，我们将列表页称为`AActivity`，详情页称为`BActivity`，此时，对应于上面提到的四种`Transition`：
- `AActivity's Exit Transition`为`null`
- `AActivity's Reenter Transition`为`null`
- `BActivity's Enter Transition`则分为三个部分：
 - 封面从小的圆形渐渐变成大的方形
 - 播放图标的半径渐渐变大
 - 底下的列表采用了自定义的`Slide-in`动画。
- `BActivity's Exit Transition`：
 - 上半部分采用了`Slide(TOP)`的方式，而下半部分采用`Slide(BOTTOM)`的方式。

# 四、源码分析
系统默认自带了三种`Transition`，`Fade、Slide、Explode`，这一节，我们一起来分析一下它们的实现方式：
## 4.1 `Fade`
### 4.1.1 `captureXXX`函数
首先，我们看一下它获取起点和终点属性的函数：
- `public void captureStartValues(TransitionValues transitionValues)`
- `public void captureEndValues(TransitionValues transitionValues)`

`Fade`只重写了`captureStartValues`，在这里面，它把`View`当前的`translationAlpha`值保存起来，这个值表示的是在`Transition`开始之前`View`的`translationAlpha`的值：
```
    @Override
    public void captureStartValues(TransitionValues transitionValues) {
        super.captureStartValues(transitionValues);
        transitionValues.values.put(PROPNAME_TRANSITION_ALPHA, transitionValues.view.getTransitionAlpha());
    }
```
### 4.1.2 `onAppear`和`onDisappear`
在上面的分析当中，我们提到过，当`View`的可见性从`INVISIBLE`变为`VISIBLE`时会调用`Transition`中的`Animator`来执行这一变换的过程，例如从`AActivity`跳转到`BActivity`，那么`BActivity`中的`View`就会调用`onAppear`所返回的`Animator`：
```
    @Override
    public Animator onAppear(ViewGroup sceneRoot, View view, TransitionValues startValues, TransitionValues endValues) {
        float startAlpha = getStartAlpha(startValues, 0);
        if (startAlpha == 1) {
            startAlpha = 0;
        }
        return createAnimation(view, startAlpha, 1);
    }
```
这里首先会通过`getStartAlpha`去获取起始的`transitionAlpha`值，它是把之前保存在`PROPNAME_TRANSITION_ALPHA`中的值取出来：
```
    private static float getStartAlpha(TransitionValues startValues, float fallbackValue) {
        float startAlpha = fallbackValue;
        if (startValues != null) {
            Float startAlphaFloat = (Float) startValues.values.get(PROPNAME_TRANSITION_ALPHA);
            if (startAlphaFloat != null) {
                startAlpha = startAlphaFloat;
            }
        }
        return startAlpha;
    }
```
下面，我们再回到`onAppear`函数当中，看一下`Animator`的创建过程：
```
    private Animator createAnimation(final View view, float startAlpha, final float endAlpha) {
        if (startAlpha == endAlpha) {
            return null;
        }
        view.setTransitionAlpha(startAlpha);
        final ObjectAnimator anim = ObjectAnimator.ofFloat(view, "transitionAlpha", endAlpha);
        final FadeAnimatorListener listener = new FadeAnimatorListener(view);
        anim.addListener(listener);
        addListener(new TransitionListenerAdapter() {
            @Override
            public void onTransitionEnd(Transition transition) {
                view.setTransitionAlpha(1);
            }
        });
        return anim;
    }
```
从上面可以看出，它返回的是一个`ObjectAnimator`，这个`Animator`会把`View`的`translationAlpha`从`startAlpha`变为`1`，这也就是一个渐渐显示的过程。
再看一下`onDisappear`函数，它就是`onAppear`的反向过程：
```
    @Override
    public Animator onDisappear(ViewGroup sceneRoot, final View view, TransitionValues startValues,
            TransitionValues endValues) {
        float startAlpha = getStartAlpha(startValues, 1);
        return createAnimation(view, startAlpha, 0);
    }
```
## 4.2 `Slide`
下面，我们来看一下另一种`Transition` - `Slide`的实现原理，和上面类似，我们先看一下`captureXXX`方都做了什么：
### 4.2.1 `captureXXX`
```
    @Override
    public void captureStartValues(TransitionValues transitionValues) {
        super.captureStartValues(transitionValues);
        captureValues(transitionValues);
    }

    @Override
    public void captureEndValues(TransitionValues transitionValues) {
        super.captureEndValues(transitionValues);
        captureValues(transitionValues);
    }
```
对于起点和终点值的获取都是调用了下面这个函数，它保存的是`View`在窗口中的位置：
```
private void captureValues(TransitionValues transitionValues) {
    View view = transitionValues.view;
    int[] position = new int[2];
    view.getLocationOnScreen(position);
    transitionValues.values.put(PROPNAME_SCREEN_POSITION, position);
}
```
### 4.2.2 `onAppear`和`onDisappear`
```
    @Override
    public Animator onAppear(ViewGroup sceneRoot, View view, TransitionValues startValues, TransitionValues endValues) {
        if (endValues == null) {
            return null;
        }
        int[] position = (int[]) endValues.values.get(PROPNAME_SCREEN_POSITION);
        //终点值是确定的
        float endX = view.getTranslationX();
        float endY = view.getTranslationY();
        //起点值则需要根据所选的模式来确定
        float startX = mSlideCalculator.getGoneX(sceneRoot, view, mSlideFraction);
        float startY = mSlideCalculator.getGoneY(sceneRoot, view, mSlideFraction);
        //根据起点值、终点值、View所处窗口的位置，来得到一个`Animator`
        return TranslationAnimationCreator.createAnimation(view, endValues, position[0], position[1], startX, startY, endX, endY, sDecelerate, this);
    }
```
这里面，最关键的是`mSlideCalculator`，默认情况下为：
```
    private static final CalculateSlide sCalculateBottom = new CalculateSlideVertical() {
        @Override
        public float getGoneY(ViewGroup sceneRoot, View view, float fraction) {
            return view.getTranslationY() + sceneRoot.getHeight() * fraction;
        }
    };
```
用一张图解解释一下上面的坐标：
![](http://upload-images.jianshu.io/upload_images/1949836-84ac8c8c3da765be.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
所以当我们采用这个`Transition`的时候，就可以看到它从屏幕的底端滑上来。
而`onDisappear`则也是一个反向的过程：
```
    @Override
    public Animator onDisappear(ViewGroup sceneRoot, View view, TransitionValues startValues, TransitionValues endValues) {
        if (startValues == null) {
            return null;
        }
        int[] position = (int[]) startValues.values.get(PROPNAME_SCREEN_POSITION);
        //这里的起始值和终点值正好是和onAppear相反的.
        float startX = view.getTranslationX();
        float startY = view.getTranslationY();
        float endX = mSlideCalculator.getGoneX(sceneRoot, view, mSlideFraction);
        float endY = mSlideCalculator.getGoneY(sceneRoot, view, mSlideFraction);
        return TranslationAnimationCreator.createAnimation(view, startValues, position[0], position[1], startX, startY, endX, endY, sAccelerate, this);
    }
```
## 4.3 小结
通过分析`Fade`和`Slide`的源码，它们的主要思想就是：
- 在`capturexxx`方法中，把属性保存在`TranslationValues`中，这里，一定要记得调用对应的`super`方法让系统保存一些默认的状态。
- 在`onAppear`和`onDisappear`中，根据起点和终点和终点的`TranslationValues`，构造一个改变`View`属性的`Animator`，同时在动画结束之后，还原它的属性。

# 五、总结
这一篇我们分析了`Content Transition`的设计思想和原理，下一篇文章我们将着重讨论如何通过代码来实现上面的效果。

# 六、参考文献
[1.Getting Started with Activity & Fragment Transitions (part 1)](http://www.androiddesignpatterns.com/2014/12/activity-fragment-transitions-in-android-lollipop-part1.html)
[2.Content Transitions In-Depth (part 2)](http://www.androiddesignpatterns.com/2014/12/activity-fragment-content-transitions-in-depth-part2.html)
[3.Material-Animations](https://github.com/lgvalle/Material-Animations)
