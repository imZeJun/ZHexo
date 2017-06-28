---
title: Fragment 知识梳理(4) - FragmentPagerAdapter 和 FragmentStatePagerAdapter 解析
date: 2017-03-30 21:19
categories : Fragment 知识梳理
---
# 一、概述
在平时的开发当中，用到`ViewPager`的场景主要是以下两种：
- 对于主页中的每个子页面，用`Fragment`包裹起来，然后通过`ViewPager`来实现页面之间的切换。
- 广告轮播图。

其中对于第一种情况，我们常常会使用到两个`PagerAdapter`的实现类，也就是`FragmentStatePagerAdapter`和`FragmentPagerAdapter`，今天，我们就来学习一下它们的使用方法，并进行对比。
# 二、`FragmentPagerAdapter`
## 2.1 使用
在我们的例子中，我们定义了一个`Acitivity`，它的布局中包含有一个`ViewPager`。初始时候我们会给`mFragments`列表中新建`4`个`Fragment`实例，然后把它传给继承于`FragmentPagerAdapter`的适配器，`LogcatFragment`就是用来打印`Fragment`的生命周期：
```
    private void initFPAFragments() {
        mFragments = new ArrayList<>();
        for (int i = 0; i < INCREASE; i++) {
            //初始时刻有4个Fragment，每个Fragment和一条数据相关联.
            mFragments.add(LogcatFragment.newInstance("index=" + i));
        }
        ViewPager viewPager = (ViewPager) findViewById(R.id.vp_content);
        mFPAdapter = new FPAdapter(getSupportFragmentManager(), mFragments);
        viewPager.setAdapter(mFPAdapter);
    }

    private class FPAdapter extends FragmentPagerAdapter {

        private List<Fragment> mFragments;

        public FPAdapter(FragmentManager fm, List<Fragment> fragments) {
            super(fm);
            mFragments = fragments;
        }

        @Override
        public Fragment getItem(int position) {
            Log.d("LogcatFragment", "get Item from FPAdapter, position=" + position);
            return mFragments.get(position);
        }

        @Override
        public int getCount() {
            return mFragments.size();
        }

    }
```
## 2.2 现象
使用过`ViewPager`的同学都知道，`ViewPager`有一个`setOffscreenPageLimit`，它表示对于当`ViewPager`处于`IDLE`状态时，它的左右两端最多会保留多少个页面，对于超出这个范围的页面有可能会需要从`PageAdapter`中进行重建，这里我们设置的是`1`，下面我们进行一系列的操作，并观察此时各个页面及其内部的`Fragment`的变化情况：
- 第一步：当我们第一次启动`Activity`的时候，默认会添加它的左右两个界面，由于我们位于第一个（`index=0`），因此会添加它及其右边的界面（`index=0`），此时这两个页面当中内部的`Fragment`的生命周期如下图所示：
![](http://upload-images.jianshu.io/upload_images/1949836-826853a0a510b726.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
从我们经常看到的`Fragment`生命周期的图来看，就是下面红色的部分：
![](http://upload-images.jianshu.io/upload_images/1949836-ccf11252d82fb266.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
- 第二步：下面，我们滑动到`index=1`的界面，此时`index=2`的页面会被添加，它内部的`Fragment`所走的生命周期和上面完全相同，由于`index=1`左右两边的界面个数都为`1`，因此不会有页面被移除。
- 第三步：继续往右滑动到`index=2`的界面，此时会添加`index=3`的页面，并移除`index=0`的页面，其内部包含的`Fragment`的生命周期打印为：
![](http://upload-images.jianshu.io/upload_images/1949836-9bd7741d704aedce.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
可以看到对于添加的`index=3`的页面而言，它内部的`Fragment`所走的生命周期和`index=0/1/2`相同，而被移除的`index=0`的页面内部的`Fragment`所走的生命周期为：
![](http://upload-images.jianshu.io/upload_images/1949836-b16d36e64de4a871.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
- 第四步：向右滑动到`index=1`的界面，此时`index=0`的界面需要被重新添加，而`index=3`的界面则需要被移除，此时的打印为：
![](http://upload-images.jianshu.io/upload_images/1949836-8dd6b994724ed80a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
这时候，对于**重新添加**的页面`index=0`，它和第一次添加的时候有两点不同：
 - 没有再去自定义的`FragmentPagerAdapter`中取`Fragment`
 - 其内部的`Fragment`所走的生命周期不同，此时为：
![](http://upload-images.jianshu.io/upload_images/1949836-aefd8329c17b8376.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

最后，我们总结一下，对于三种情况的页面内部的`Fragment`所走生命周期的区别如下图所示：
- 第一次添加的页面
- 重新添加的页面
- 移除的页面

![](http://upload-images.jianshu.io/upload_images/1949836-fc5799fd0642cf67.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
## 2.3 源码解析
现在，我们就开始解释一下，为什么**第一次添加**和**重新添加**的页面内部对应的`Fragment`会有所不同，我们只需要关注`FragmentPagerAdapter`内的两个函数：
- `public Object instantiateItem(ViewGroup container, int position)`，添加页面时回调。
- `public void destroyItem(ViewGroup container, int position, Object object)`，移除页面时回调。

```
    @Override
    public Object instantiateItem(ViewGroup container, int position) {
        if (mCurTransaction == null) {
            mCurTransaction = mFragmentManager.beginTransaction();
        }
        //这里的itemId返回的是对应position的页面的唯一标识符.
        final long itemId = getItemId(position);
        //1.先是通过FragmentManager来找.
        String name = makeFragmentName(container.getId(), itemId);
        Fragment fragment = mFragmentManager.findFragmentByTag(name);
        //2.如果找到了，那么调用attach方法.
        if (fragment != null) {
            if (DEBUG) Log.v(TAG, "Attaching item #" + itemId + ": f=" + fragment);
            mCurTransaction.attach(fragment);
        } else {
            //3.如果没找到，那么通过子类实现的getItem方法来获取.
            fragment = getItem(position);
            //这里调用的是add方法.
            mCurTransaction.add(container.getId(), fragment, makeFragmentName(container.getId(), itemId));
        }
        //根据需要，回调下面这两个方法.
        if (fragment != mCurrentPrimaryItem) {
            fragment.setMenuVisibility(false);
            fragment.setUserVisibleHint(false);
        }
        //返回给ViewPager.
        return fragment;
    }

    @Override
    public void destroyItem(ViewGroup container, int position, Object object) {
        if (mCurTransaction == null) {
            mCurTransaction = mFragmentManager.beginTransaction();
        }
        //4.移除界面时，调用的是detach方法.
        mCurTransaction.detach((Fragment)object);
    }
```
注意看上面的注释，就能解释上面我们看到的现象了：
- 第一次添加界面的时候，由于`FragmentManager`中没有这个`Fragment`，因此需要通过自定义的`FragmentPagerAdapter`获取，然后调用`add`方法，也就是上面代码中的第**(3)**步，所走的生命周期为`onAttach() -> onResume()`。
- 移除界面时，使用的是`detach`方法，也就是上面代码中的第**(4)**步，接触过`Fragment`的人都知道，这时候仅仅是`Fragment`的界面被从`View`树上移除了而已，它的实例仍然被保存在`FragmentManager`当中，所走的生命周期为`onPause() -> onDestroyView()`。
- 重新添加界面时，由于此时去`FragmentManager`中能找到那个`Fragment`，所以调用的是`attach`方法，也就是上面代码中的第**(2)**步，所走的生命周期为`onCreateView() -> onResume()`，并且不需要再从自定义的`FragmentPagerAdapter`中获取`Fragment`。

整个逻辑如下图所示：
![](http://upload-images.jianshu.io/upload_images/1949836-8e83f356dbba854d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

# 三、`FragmentStatePagerAdapter`
## 3.1 现象
我们的代码基本不用改动，只需要把原来继承于`FragmentPagerAdapter`的子类替换为继承`FragmentStatePagerAdapter`就可以了。在第二章当中，我们分析得很详细，相信大家对于整个分析的套路已经理解，因此，为了减少篇幅，我们直接说结论，当进行和上面相同的操作之后，把页面分为三种类型：
- 第一次添加
- 重新添加
- 移除

此时，它们内部的`Fragment`所走的生命周期为：
![](http://upload-images.jianshu.io/upload_images/1949836-8ca419641293da42.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

对于**重新添加**和**移除**的界面，其内部的`Fragment`所走的生命周期都和`FragmentPagerAdapter`不同，下面，我们就从源码的角度，来看一下导致这些区别的原因。

## 3.2 源码解析
和前面类似，我们只关注添加和移除时调用的那两个方法：
```
    @Override
    public Object instantiateItem(ViewGroup container, int position) {
        //如果mFragments中存在对应位置的fragment，那么直接返回.
        if (mFragments.size() > position) {
            Fragment f = mFragments.get(position);
            if (f != null) {
                return f;
            }
        }

        if (mCurTransaction == null) {
            mCurTransaction = mFragmentManager.beginTransaction();
        }

        Fragment fragment = getItem(position);
        //多了恢复状态的操作.
        if (mSavedState.size() > position) {
            Fragment.SavedState fss = mSavedState.get(position);
            if (fss != null) {
                fragment.setInitialSavedState(fss);
            }
        }
       //保证mFragments的大小和ViewPager往右滑动的最远的index相同.
        while (mFragments.size() <= position) {
            mFragments.add(null);
        }
        fragment.setMenuVisibility(false);
        fragment.setUserVisibleHint(false);
        //设置对应位置.
        mFragments.set(position, fragment);
        //这里很关键，调用的add方法.
        mCurTransaction.add(container.getId(), fragment);

        return fragment;
    }

    @Override
    public void destroyItem(ViewGroup container, int position, Object object) {
        Fragment fragment = (Fragment) object;
        if (mCurTransaction == null) {
            mCurTransaction = mFragmentManager.beginTransaction();
        }
        while (mSavedState.size() <= position) {
            mSavedState.add(null);
        }
        mSavedState.set(position, fragment.isAdded() ? mFragmentManager.saveFragmentInstanceState(fragment) : null);
        mFragments.set(position, null);
        //调用的是remove方法.
        mCurTransaction.remove(fragment);
    }
```
从上面的源码当中，总结出以下几点：
- `FragmentStatePagerAdapter`在**移除页面**的时候，调用的是`remove`方法，也就是说，`FragmentManager`中不再有这个`Fragment`的实例，所走的生命周期为`onPause() -> onDetach()`。
- 无论是**添加页面**还是**重新添加页面**，它是通过`add`方法，并且每次都会通过自定义的`FragmentStatePagerAdapter`子类的`getItem`方法来获取`Fragment`，所以它们内部的`Fragment`所走生命周期相同，都是从`onAttach() -> onResume()`。
- 对于`ViewPager`当前界面中所对应的`Fragment`，是通过一个`mFragments`列表来管理的，由于此时没有`FragmentManager`来帮我们实现`Fragment`集合的状态的保存和恢复，所以就需要我们自己实现`onSave/onRestore`方法来进行状态的保存和恢复。

整个流程如下图所示：
![](http://upload-images.jianshu.io/upload_images/1949836-9de931dfd372a5a8.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

# 四、总结
`FragmentPagerAdapter`和`FragmentStatePagerAdapter`最大的区别就在于前者会把所有`Fragment`的示例都缓存在内存当中，而后者仅仅保存了`ViewPager`当前存在的页面所对应的`Fragment`，当页面被移除之后，这个`Fragment`的示例它也就不再保存了。
当然，在我们前面的例子中，虽然使用了`FragmentStatePagerAdapter`，但是由于我们在`DemoActivity`中用一个列表保存了所有的`Fragment`实例，因此它没有被回收，如果希望让页面被移除的时候，其对应的`Fragment`实例也被回收，那么我们的`FragmentStatePagerAdapter`的子类应该写成这样：
```
    private void initFSPAFragments() {
        ViewPager viewPager = (ViewPager) findViewById(R.id.vp_content);
        mFSPAdapter = new FSPAdapter(getSupportFragmentManager());
        viewPager.setAdapter(mFSPAdapter);
    }

    private class FSPAdapter extends FragmentStatePagerAdapter {

        public FSPAdapter(FragmentManager fm) {
            super(fm);
        }

        @Override
        public Fragment getItem(int position) {
            Log.d("LogcatFragment", "get Item from FSPAdapter, position=" + position);
            return LogcatFragment.newInstance("index=" + position);
        }

        @Override
        public int getCount() {
            return INCREASE;
        }
    }
```
