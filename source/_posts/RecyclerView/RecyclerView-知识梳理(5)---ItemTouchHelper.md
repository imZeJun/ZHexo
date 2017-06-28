---
title: RecyclerView 知识梳理(5) - ItemTouchHelper
date: 2017-04-05 23:17
categories : RecyclerView 知识梳理
---
# 一、概述
`ItemTouchHelper`在`RecyclerView`的整个体系中，负责监听`Item`的手势操作，我们通过给它设置一个继承于`ItemTouchHelper.Callback`的子类，在其中处理`Item`的`UI`变化，就可以完成侧滑删除、拖动排序等操作，下面，我们分以下几部介绍：
- `API`解析
- 实战
 - 采用默认动画
 - 自定义侧滑删除动画

# 二、`API`分析
对于`Item`的手势操作分为两种：侧滑和拖动，如果需要支持这两种，那么需要给`ItemTouchHelper`传入一个`ItemTouchHelper.Callback`的子类，并把`ItemTouchHelper`和`RecyclerView`关联起来，下面，我们先来介绍一下`ItemTouchHelper.Callback`个回调方法的含义：
## 控制相关
- `public boolean isLongPressDragEnabled()`
是否可以通过长按来触发拖动操作，默认返回`true`，如果返回`false`，那么可以通过`startDrag(ViewHolder)`方法来触发某个特定`Item`的拖动的机制。
- `public boolean isItemViewSwipeEnabled()`
是否可以对每个`Item`进行侧滑。
- `public int getMovementFlags(RecyclerView recyclerView, RecyclerView.ViewHolder viewHolder)`
返回对于某个`ViewHolder`可以移动的方向，可选的值有`UP/DOWN/LEFT/RIGHT/START/END`。对于纵向排列的线性布局而言，如果要支持上下拖动排序，那么就要标志位中就要包含`UP&DOWN`，而如果需要支持左滑删除，那么标志位中就要包含`LEFT`。

## 结果相关
- `public abstract boolean onMove(RecyclerView recyclerView, ViewHolder viewHolder, ViewHolder target)`
当某个被拖动的`Item`被从旧位置拖动到了新位置后回调，如果返回`true`，那么`ItemTouchHelper`就认为`viewHolder`已经被移动到了`target`在`Adapter`中的位置。
- `public abstract void onSwiped(ViewHolder viewHolder, int direction)`
当某个`Item`被滑动到消失时回调，`direction`表示滑动的方向。

## 状态相关
- `public void onSelectedChanged(RecyclerView.ViewHolder viewHolder, int actionState)`
当`Item`的状态发生改变时，回调该方法，`actionState`的取值有`ACTION_STATE_IDLE/ACTION_STATE_SWIPE/ACTION_STATE_DRAG`。
- `public void clearView(RecyclerView recyclerView, ViewHolder viewHolder)`
标志着用户对于某个`Item`的操作并且`Item`的动画结束，此时我们应该恢复它的状态，以保证它被重新使用的时候能正确地展现。
- `public void onChildDraw(Canvas c, RecyclerView recyclerView, RecyclerView.ViewHolder viewHolder, float dX, float dY, int actionState, boolean isCurrentlyActive)`
 - `Canvas`：绘制`RecyclerView`的`Canvas`
 - `dx, dy`：偏移。
 - `actionState`：拖拽还是侧滑，对应`ACTION_STATE_DRAG`和`ACTION_STATE_SWIPE`。
 - `isCurrentlyActive`为`true`表示这个`Item`正在被用户所控制，`false`则表示它仅仅是在回到原本状态的动画过程当中。

- `public void onChildDrawOver(Canvas c, RecyclerView recyclerView, ViewHolder viewHolder, float dX, float dY, int actionState, boolean isCurrentlyActive)`
和上面类似，只不过它是绘制在`Item`之上。

# 三、实战
## 3.1 使用系统默认效果
如果我们希望使用系统默认的效果，那么只需要做以下几步：
- 继承于`ItemTouchHelper.Callback`编写自己的回调类，并在拖动和侧滑操作完成之后更新数据：
```
public class SimpleItemTouchHelper extends ItemTouchHelper.Callback {

    private ItemTouchAdapter mAdapter;

    public SimpleItemTouchHelper(ItemTouchAdapter adapter) {
        mAdapter = adapter;
    }

    @Override
    public boolean onMove(RecyclerView recyclerView, RecyclerView.ViewHolder viewHolder, RecyclerView.ViewHolder target) {
        Log.d("SimpleItemTouchHelper", "onSwiped, onMove, source=" + viewHolder.getAdapterPosition() + ",target=" + target.getAdapterPosition());
        mAdapter.onItemDragged(viewHolder.getAdapterPosition(), target.getAdapterPosition());
        return true;
    }

    @Override
    public void onSwiped(RecyclerView.ViewHolder viewHolder, int direction) {
        Log.d("SimpleItemTouchHelper", "onSwiped, direction=" + direction);
        mAdapter.onItemSwiped(viewHolder.getAdapterPosition());
    }

    @Override
    public int getMovementFlags(RecyclerView recyclerView, RecyclerView.ViewHolder viewHolder) {
        return makeMovementFlags(ItemTouchHelper.UP | ItemTouchHelper.DOWN, ItemTouchHelper.LEFT | ItemTouchHelper.RIGHT);
    }
}
```
- 编写数据操作的代码：
```
public class NormalAdapter extends RecyclerView.Adapter<NormalAdapter.NormalViewHolder> implements ItemTouchAdapter {

    //......

    @Override
    public void onItemDragged(int from, int to) {
        Collections.swap(mTitles, from, to);
        notifyItemMoved(from, to);
    }

    @Override
    public void onItemSwiped(int position) {
        mTitles.remove(position);
        notifyItemRemoved(position);
    }
}
```
- 把`ItemTouchHelper.Callback`和`RecyclerView`关联起来，看注释中的`1,2,3`步：
```
    private void init() {
        List<String> titles = new ArrayList<>();
        for (int i = 0; i < 20; i++) {
            titles.add(String.valueOf(i));
        }
        LinearLayoutManager layoutManager = new LinearLayoutManager(this);
        RecyclerView recyclerView = (RecyclerView) findViewById(R.id.rv_content);
        recyclerView.setLayoutManager(layoutManager);
        NormalAdapter adapter = new NormalAdapter(titles);
        //1.自定义的ItemTouchHeloer.Callback
        SimpleItemTouchHelper simpleItemTouchHelper = new 
SimpleItemTouchHelper(adapter);
        //2.利用这个Callback构造ItemTouchHelper
        ItemTouchHelper itemTouchHelper = new ItemTouchHelper(simpleItemTouchHelper);
        //3.把ItemTouchHelper和RecyclerView关联起来.
        itemTouchHelper.attachToRecyclerView(recyclerView);
        recyclerView.setAdapter(adapter);
    }
```
下面就是最终的效果：
![](http://upload-images.jianshu.io/upload_images/1949836-5d944d797be166e7.gif?imageMogr2/auto-orient/strip)

## 3.2 自定义侧滑删除动画
当我们需要自定侧滑删除动画时，那么需要重写`onChildDraw`或者`onChildDrawOver`方法，在其中监听滑动距离的变化，并根据它来实时改变`viewHolder`中的`UI`，首先看效果：
![](http://upload-images.jianshu.io/upload_images/1949836-eb961e06528964af.gif?imageMogr2/auto-orient/strip)
- 首先，我们需要重写`Item`的布局，它包含两层，顶层是普通状态的标题文案，而底层则是蓝色底的删除提示：
```
<FrameLayout
    xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="66dp">
    <!-- 删除提示 -->
    <LinearLayout
        android:id="@+id/ll_delete"
        android:orientation="vertical"
        android:gravity="center"
        android:layout_gravity="end"
        android:background="@android:color/holo_blue_dark"
        android:paddingLeft="10dp"
        android:paddingRight="10dp"
        android:layout_width="wrap_content"
        android:layout_height="match_parent">
        <ImageView
            android:src="@android:drawable/ic_input_delete"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"/>
        <TextView
            android:text="delete"
            android:textColor="@android:color/white"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"/>
    </LinearLayout>
    <!-- 普通文案 -->
    <TextView
        android:id="@+id/tv_title"
        android:gravity="center"
        android:background="@android:color/white"
        android:layout_width="match_parent"
        android:layout_height="match_parent"/>
</FrameLayout>
```
接着，我们需要重写`ItemTouchHelper.Callback`：
```
public class AdvancedItemTouchHelper extends ItemTouchHelper.Callback {

    private ItemTouchAdapter mAdapter;

    public AdvancedItemTouchHelper(ItemTouchAdapter itemTouchAdapter) {
        mAdapter = itemTouchAdapter;
    }

    @Override
    public boolean isLongPressDragEnabled() {
        return false;
    }

    @Override
    public int getMovementFlags(RecyclerView recyclerView, RecyclerView.ViewHolder viewHolder) {
        return makeMovementFlags(0, ItemTouchHelper.LEFT);
    }

    @Override
    public void onSwiped(RecyclerView.ViewHolder viewHolder, int direction) {
        mAdapter.onItemSwiped(viewHolder.getAdapterPosition());
    }

    @Override
    public boolean onMove(RecyclerView recyclerView, RecyclerView.ViewHolder viewHolder, RecyclerView.ViewHolder target) {
        return false;
    }

    @Override
    public void clearView(RecyclerView recyclerView, RecyclerView.ViewHolder viewHolder) {
        super.clearView(recyclerView, viewHolder);
        ((NormalAdapter.NormalViewHolder) viewHolder).mTv.setTranslationX(0);
    }

    @Override
    public void onChildDraw(Canvas c, RecyclerView recyclerView, RecyclerView.ViewHolder viewHolder, float dX, float dY, int actionState, boolean isCurrentlyActive) {
        NormalAdapter.NormalViewHolder mViewHolder = (NormalAdapter.NormalViewHolder) viewHolder;
        int deleteWidth = mViewHolder.mDeleteLayout.getWidth();
        float fraction = deleteWidth / (float) mViewHolder.itemView.getWidth();
        mViewHolder.mTv.setTranslationX(dX * fraction);
    }
}
```

这里面有几点需要注意：
- 为了让`Item`支持左滑删除，我们需要在`getMovementFlags`中返回`ItemTouchHelper.LEFT`标志位。
- 在`onChildDraw`当中，通过传入的`dX`动态改变了普通文案的`translationX`，使得底层的删除提示能够漏出。
- 在侧滑操作完成之后，通过`Adapter`来删除数据。
- 在`clearView`中，需要把`mTv`重置为初始的状态。

最后，我们按照前面的方法，把它和`RecyclerView`关联起来：
```
    private void init() {
        List<String> titles = new ArrayList<>();
        for (int i = 0; i < 20; i++) {
            titles.add("Item " + String.valueOf(i));
        }
        LinearLayoutManager layoutManager = new LinearLayoutManager(this);
        RecyclerView recyclerView = (RecyclerView) findViewById(R.id.rv_content);
        recyclerView.setLayoutManager(layoutManager);
        NormalAdapter adapter = new NormalAdapter(titles);
        AdvancedItemTouchHelper advancedItemTouchHelper = new AdvancedItemTouchHelper(adapter);
        ItemTouchHelper itemTouchHelper = new ItemTouchHelper(advancedItemTouchHelper);
        itemTouchHelper.attachToRecyclerView(recyclerView);
        recyclerView.addItemDecoration(new DividerItemDecoration(this, DividerItemDecoration.VERTICAL));
        recyclerView.setAdapter(adapter);
    }
```
# 四、小结
自定义`RecyclerView`的手势动画，关键是要理解`ItemTouchHelper.Callback`中各回调函数的含义，再通过回调函数中传入的数值来动态改变`viewHolder`中保存的`itemView`以及其子`View`的展现形式，就可以做出各种绚丽的效果。

# 五、参考文献
[RecyclerView 进阶：使用 ItemTouchHelper 实现拖拽和侧滑删除](http://www.jianshu.com/p/0c1984bc9383)
