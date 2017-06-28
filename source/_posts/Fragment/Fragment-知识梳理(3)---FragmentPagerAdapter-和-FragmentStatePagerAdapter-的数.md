---
title: Fragment 知识梳理(3) - FragmentPagerAdapter 和 FragmentStatePagerAdapter 的数据更新问题
date: 2017-03-31 21:05
categories : Fragment 知识梳理
---
# 一、概述
在上一篇文章中，我们通过源码的角度了解`FragmentPagerAdapter`和`FragmentStatePagerAdapter`的原理。这其实是为我们**分析数据更新问题**做一个铺垫。
在实际的开发当中，我们在`ViewPager`中嵌套的`Fragment`中并不是固定不变的，需要动态地添加和删除，下面我们就从几个大家经常会遇到的问题入手，然后分析问题的原因，最后我们尝试总结一种比较好的数据更新方式。

# 二、使用`FragmentPagerAdapter`
## 2.1 一段有问题的代码
```
public class DemoActivity extends AppCompatActivity {

    private static final int INCREASE = 4;
    private FPAdapter mFPAdapter;
    private List<String> mTitles;
    private int mGroup = 0;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_not_update);
        TextView updateTv = (TextView) findViewById(R.id.tv_update);
        updateTv.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                updateFragments();
            }
        });
        initFPAFragments();
    }

    private void initFPAFragments() {
        mTitles = new ArrayList<>();
        for (int i = 0; i < INCREASE; i++) {
            mTitles.add("index=" + i + ",group=0");
        }
        ViewPager viewPager = (ViewPager) findViewById(R.id.vp_content);
        mFPAdapter = new FPAdapter(getSupportFragmentManager(), mTitles);
        viewPager.setAdapter(mFPAdapter);
    }

    private void updateFragments() {
        mTitles.clear();
        mGroup++;
        for (int i = 0; i < INCREASE; i++) {
            mTitles.add("index=" + i + ",group=" + mGroup);
        }
        mFPAdapter.notifyDataSetChanged();
    }

    private class FPAdapter extends FragmentPagerAdapter {

        private List<String> mTitles;

        public FPAdapter(FragmentManager fm, List<String> titles) {
            super(fm);
            mTitles = titles;
        }

        @Override
        public Fragment getItem(int position) {
            Log.d("LogcatFragment", "get Item from FPAdapter, position=" + position);
            return LogcatFragment.newInstance(mTitles.get(position));
        }

        @Override
        public int getCount() {
            return mTitles.size();
        }

    }

}
```
之所以会写出这样的代码，很大一部分是受到我们平时写`ListView`中`BaseAdapter`的影响，因为我们浅意识地认为，调用了`notifyDataSetChanged()`方法之后，`ViewPager`就会去调用`getItem`方法来获取新的`Fragment`以替换旧的`Fragment`，就好像我们使用`ListVIew`的时候，它会去回调`BaseAdapter`的`getView`方法来获取新的`View`一样，运行上面的`Demo`，会有发现以下几个问题：
- 第一个问题：调用`notifyDataSetChanged()`之后，`ViewPager`当前存在的页面中的`Fragment`不会发生变化。
- 第二个问题：对于重新添加的界面，不会回调`getItem`来获取新的`Fragment`。

## 2.2 原因分析 - 问题一
我们首先分析问题一：调用`notifyDataSetChanged()`之后，`ViewPager`当前存在的页面中的`Fragment`不会发生变化。

首先，我们确定分析的场景：启动`DemoActivity`，按照之前的分析，现在会给`ViewPager`添加两个页面，分别是`index=0`和`index=1`，接着我们调用`PagerAdapter#notifyDataSetChanged()`，最终会走到`ViewPager#dataSetChanged`方法，我们看一下里面做了什么：
```
    void dataSetChanged() {
        //在我们的例子中，返回的是4
        final int adapterCount = mAdapter.getCount();
        mExpectedAdapterCount = adapterCount;
        //此时为true.
        boolean needPopulate = mItems.size() < mOffscreenPageLimit * 2 + 1
                && mItems.size() < adapterCount;
        int newCurrItem = mCurItem;
        boolean isUpdating = false;
        //遍历列表，这个
        for (int i = 0; i < mItems.size(); i++) {
            final ItemInfo ii = mItems.get(i);
            //这里是关键，默认都是返回POSITION_UNCHANGED
            final int newPos = mAdapter.getItemPosition(ii.object);
            //第一种情况：如果返回的是POSITION_UNCHANGED，那么表示这个界面在ViewPager中的位置没有变，那么不需要更新.
            if (newPos == PagerAdapter.POSITION_UNCHANGED) {
                continue;
            }
            //第二种情况：如果返回的是POSITION_NONE，就表示这个界面在ViewPager中不存在了，那么就把它移除.
            if (newPos == PagerAdapter.POSITION_NONE) {
                mItems.remove(i);
                i--;

                if (!isUpdating) {
                    mAdapter.startUpdate(this);
                    isUpdating = true;
                }

                mAdapter.destroyItem(this, ii.position, ii.object);
                needPopulate = true;

                if (mCurItem == ii.position) {
                    // Keep the current item in the valid range
                    newCurrItem = Math.max(0, Math.min(mCurItem, adapterCount - 1));
                    needPopulate = true;
                }
                continue;
            }
           //第三种情况：界面仍然存在，但是其在ViewPager中的位置发生了改变.
            if (ii.position != newPos) {
                if (ii.position == mCurItem) {
                    // Our current item changed position. Follow it.
                    newCurrItem = newPos;
                }

                ii.position = newPos;
                needPopulate = true;
            }
        }

        if (isUpdating) {
            mAdapter.finishUpdate(this);
        }

        Collections.sort(mItems, COMPARATOR);

        if (needPopulate) {
            // Reset our known page widths; populate will recompute them.
            final int childCount = getChildCount();
            for (int i = 0; i < childCount; i++) {
                final View child = getChildAt(i);
                final LayoutParams lp = (LayoutParams) child.getLayoutParams();
                if (!lp.isDecor) {
                    lp.widthFactor = 0.f;
                }
            }

            setCurrentItemInternal(newCurrItem, false, true);
            requestLayout();
        }
    }
```
要理解上面的这段代码，首先要明白`mItems`是什么，以及`mItems`的`ItemInfo`中各个成员变量的含义：
- `mItems`中的每一个`ItemInfo`和存在与`ViewPager`中的界面一一关联。
- `ItemInfo`中的含义：
```
    static class ItemInfo {
        Object object; //通过PagerAdapter#instantiateItem所返回的Object.
        int position; //这个Item所处的位置，也就是上面我们所说的index.
        boolean scrolling;
        float widthFactor;
        float offset;
    }
```
此时，也就是我们位于`index=0`的页面，`mItems`的内容为：
![](http://upload-images.jianshu.io/upload_images/1949836-cca0eb7af7c7d795.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
而如果我们滑动到`index=2`的页面，那么`mItems`的内容变为：
![](http://upload-images.jianshu.io/upload_images/1949836-00954e6773f5ffc6.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

我们注意上面有一句关键的话：
```
final int newPos = mAdapter.getItemPosition(ii.object);
```
对于它的返回值，有三种处理方式：
- `PagerAdapter.POSITION_UNCHANGED`：这个`ItemInfo`在整个`ViewPager`的位置没有发生改变。
- `PagerAdapter.POSITION_NONE`：这个`ItemInfo`在整个`ViewPager`中已经不存在了。
- `ii.position != newPos`，也就是说`ItemInfo`在`ViewPager`仍然需要存在，但是它的位置发生了改变。

也就是说，`notifyDataSetChanged()`只处理`ViewPager`当前已经存在的界面，而对于这些界面如何处理，则要依赖于`getItemPosition`的返回值，但是**`FragmentPagerAdapter`的返回值默认是`POSITION_UNCHANGED`**，因此，当前已经存在的界面不会发生任何改变。

## 2.3 原因分析 - 问题二
问题二：对于重新添加的界面，不会回调`getItem`来获取新的`Fragment`。
这个其实和`notifyDataSetChanged()`没有关系，而是和`FragmentPagerAdapter`寻找`Fragment`的方式有关，它会优先从`FragmentManager`中寻找，找不到了才会回调`getItem`来取新的`Fragment`。
但是我们在移除界面的是调用的是`detach`方法，因此`FragmentManager`中仍然保存了`Fragment`的实例，在重新添加的时候就不会再回调`getItem`来取了，这个我们在前一篇文章中已经分析过，就不贴具体的代码了。

# 三、`FragmentStatePagerAdapter`
我们把上面例子中的`FragmentPagerAdapter`替换成为`FragmentStatePagerAdapter`，采用一样的更新方式，此时调用`notifyDataSetChanged()`之后，`ViewPager`当前存在的页面中的`Fragment`依然不会发生变化，不刷新的原因和`FragmentPagerAdapter`是相同的。
与`FragmentPagerAdapter`不同的是，**对于重新添加的界面，会回调`getItem`来获取新的`Fragment`**，这个原因在之前的文章中也分析过了。

# 四、实现一个高效的动态`FragmentPagerAdapter`
我们的需求和下面的这个界面类似：
![](http://upload-images.jianshu.io/upload_images/1949836-054f183233d24cff.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
需求包括：
- 支持动态地添加和移除界面，界面的个数和频道的个数相同，并且是可变的。
- 当频道发生变化时，界面也要根据频道的顺序进行相应的改变，但是，如果上次存在的频道，在编辑之后仍然存在，那么应当复用之前的界面。

因此，我们继承于`FramentStatePagerAdapter`：
```
public abstract class FixedPagerAdapter<T> extends FragmentStatePagerAdapter {

    private List<ItemObject> mCurrentItems = new ArrayList<>();

    public FixedPagerAdapter(FragmentManager fragmentManager) {
        super(fragmentManager);
    }

    @Override
    public Object instantiateItem(ViewGroup container, int position) {
        while (mCurrentItems.size() <= position) {
            mCurrentItems.add(null);
        }
        Fragment fragment = (Fragment) super.instantiateItem(container, position);
        ItemObject object = new ItemObject(fragment, getItemData(position));
        mCurrentItems.set(position, object);
        return object;
    }

    @Override
    public void destroyItem(ViewGroup container, int position, Object object) {
        mCurrentItems.set(position, null);
        super.destroyItem(container, position, ((ItemObject) object).fragment);
    }

    @Override
    public int getItemPosition(Object object) {
        ItemObject itemObject = (ItemObject) object;
        if (mCurrentItems.contains(itemObject)) {
            T oldData = itemObject.t;
            int oldPosition = mCurrentItems.indexOf(itemObject);
            T newData = getItemData(oldPosition);
            if (equals(oldData, newData)) {
                return POSITION_UNCHANGED;
            } else {
                int newPosition = getDataPosition(oldData);
                return newPosition >= 0 ? newPosition : POSITION_NONE;
            }
        }
        return POSITION_UNCHANGED;
    }

    @Override
    public void setPrimaryItem(ViewGroup container, int position, Object object) {
        super.setPrimaryItem(container, position, ((ItemObject) object).fragment);
    }

    @Override
    public boolean isViewFromObject(View view, Object object) {
        return super.isViewFromObject(view, ((ItemObject) object).fragment);
    }

    public abstract T getItemData(int position);

    public abstract int getDataPosition(T t);

    public abstract boolean equals(T oldD, T newD);

    public class ItemObject {

        public Fragment fragment;
        public T t;

        public ItemObject(Fragment fragment, T t) {
            this.fragment = fragment;
            this.t = t;
        }
    }

}
```
这里：
- 我们通过一个`mCurrentItems`保存了当前页面中对应的`Fragment`和其所包含的数据。
- 最重要的是我们重写了`getItemPosition`方法，根据不同的情况返回位置，这里需要子类提供三个方面的信息：
 - 新数据在某个位置的数据。
 - 某个数据在新数据中的位置。
 - 判断两个数据是否相等的标准。

现在，我们的`Adapter`只需要重写很少的代码，就能实现数据的更新：
```
public class DemoActivity extends AppCompatActivity {

    private static final int INCREASE = 4;
    private FixedPagerAdapter mFixedPagerAdapter;
    private List<String> mTitles;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_not_update);
        TextView updateTv = (TextView) findViewById(R.id.tv_update);
        updateTv.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                updateFragments();
            }
        });
        initFragments();
    }

    private void initFragments() {
        mTitles = new ArrayList<>();
        for (int i = 0; i < INCREASE; i++) {
            mTitles.add(String.valueOf(i));
        }
        ViewPager viewPager = (ViewPager) findViewById(R.id.vp_content);
        mFixedPagerAdapter = new MyFixedPagerAdapter(getSupportFragmentManager(), mTitles);
        viewPager.setAdapter(mFixedPagerAdapter);
    }

    private void updateFragments() {
        mTitles.clear();
        mTitles.add("3");
        mTitles.add("2");
        mFixedPagerAdapter.notifyDataSetChanged();
    }

    private class MyFixedPagerAdapter extends FixedPagerAdapter<String> {

        private List<String> mTitles;

        public MyFixedPagerAdapter(FragmentManager fragmentManager, List<String> titles) {
            super(fragmentManager);
            mTitles = titles;
        }

        @Override
        public String getItemData(int position) {
            return mTitles.size() > position ? mTitles.get(position) : null;
        }

        @Override
        public int getDataPosition(String s) {
            return mTitles.indexOf(s);
        }

        @Override
        public boolean equals(String oldD, String newD) {
            return TextUtils.equals(oldD, newD);
        }

        @Override
        public Fragment getItem(int position) {
            return LogcatFragment.newInstance(mTitles.get(position));
        }

        @Override
        public int getCount() {
            return mTitles.size();
        }
    }

}
```
