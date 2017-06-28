---
title: Loader 知识梳理(2) - initLoader和restartLoader的区别
date: 2017-03-08 00:01
categories : Loader 知识梳理
---
# 一、概述
在前面的一篇文章中，我们分析了`LoaderManager`的实现，里面涉及到了很多的细节问题，我们并不需要刻意地记住每个流程，之所以需要分析，以后在使用的过程中，如果遇到问题了，再去查看相关的源代码就好了。
对于`Loader`框架的理解，我认为掌握以下四个方面的东西就可以了：
- 对`LoaderManager`的实现原理有一个大概的印象。
- `LoaderManager`的三个关键方法`initLoader/restartLoader/destroyLoader`的使用场景。
- 学会自定义`Loader`来实现数据的异步加载。
- 总结一些`App`开发中常用的场景。

第一点可以参考前面的这篇文章：
> [`Loader框架 - LoaderManager初探`](http://www.jianshu.com/p/cc03191ef40b)

今天这篇，我们将专注于分析第二点：**`initLoader/restartLoader`的区别**。

# 二、基本概念的区别
首先，我们回顾一下，对于`init/restart`的定义：
- `initLoader`
 - 通过调用这个方法来初始化一个特定`ID`的`Loader`，如果当前已经有一个和这个`ID`关联的`Loader`，那么并不会去回调`onCreateLoader`来通知使用者传入一个新的 `Loader`实例替代那个旧的实例，仅仅是替代`Callback`，也就是说，上面的`Bundle`参数被丢弃了；而假如不存在一个关联的实例，那么会进行初始化，并启动它。
 - 这个方法通常是在`Activity/Fragment`的初始化阶段调用，因为它会判断之前是否存在相同的`Loader`，这样我们就可以复用之前已经加载过的数据，当屏幕装转导致`Activity`重建的时候，我们就不需要再去重新创建一个新的`Loader`，也免去了重新读取数据的过程。

- `restartLoader`
 - 调用这个方法，将会重新创建一个指定`ID`的`Loader`，如果当前已经有一个和这个`ID`关联的`Loader`，那么会对它进行`canceled/stopped/destroyed`等操作，之后，使用新传入的`Bundle`参数来创建一个新的`Loader`，并在数据加载完毕后递交给调用者。
 - 并且，在调用完这个函数之后，所有之前和这个`ID`关联的`Loader`都会失效，我们将不会收到它们传递过来的任何数据。

总结下来就是：
- 当调用上面这两个方法时，如果不存在一个和`ID`关联的`Loader`，那么这两个方法是完全相同的。
- 如果已经存在相关联的`Loader`，那么`init`方法除了替代`Callback`，不会做任何其它的事情，包括取消/停止等。而`restart`方法将会创建一个新的`Loader`，并且重新开始查询。

# 三、代码的区别
为了方便大家更直观地理解，我们截取一部分的源码来看一下：
- `initLoader`
```
LoaderInfo info = mLoaders.get(id);
if (info == null) {
    //如果不存在关联的Loader，那么创建一个新的Loader
    info = createAndInstallLoader(id, args, LoaderManager.LoaderCallbacks<Object>)callback);
} else {
   //如果已经存在，那么仅仅替代Callback，传入的Bundle参数会被丢弃。
   info.mCallbacks = (LoaderManager.LoaderCallbacks<Object>)callback;
}
```
- `restartLoader`
```
LoaderInfo info = mLoaders.get(id);
//如果已经存在一个相关联的Loader，那么执行操作。
if (info != null) {
    //mInactiveLoaders列表就是用来跟踪那些已经被抛弃的Loader
    LoaderInfo inactive = mInactiveLoaders.get(id);
    if (inactive != null) {
        //对跟踪列表进行一系列的操作。
    } else {
        //取消被抛弃的Loader，并加入到跟踪列表当中，以便在新的Loader完成任务之后销毁它。
        info.mLoader.abandon();
        mInactiveLoaders.put(id, info);
    }
}
//通知调用者，创建一个新的Loader，这个Loader将会和新的Bundle和Callback相关联。
info = createAndInstallLoader(id, args,  (LoaderManager.LoaderCallbacks<Object>)callback);
```

通过上面的代码，就印证了前面第二节我们的说话，我们根据这些区别可以知道它们各自的职责：
- `initLoader`是用来确保`Loader`能够被初始化，如果已经存在相同`ID`的`Loader`，那么它会复用之前的。
- `restartLoader`的应用场景则是我们的查询条件发生了改变。因为`LoaderManager`是用`ID`关联的，当这个`Loader`已经获取到了数据，那么就不需要再启动它了。因此当我们的需求发生了改变，就需要重新创建一个`Loader`。

也就是说：
> - 查询条件一直不变时，使用`initLoader`
- 查询条件有可能发生改变时，采用`restartLoader`。

# 五、对于屏幕旋转的情况
## 5.1 重建
当我们在`Manifest.xml`没有给`Activity`配置`configChanged`的时候，旋转屏幕会导致的`Activity/Fragment`重建，这时候有两点需要注意的：
 - 由于此时我们的查询条件并不会发生改变，并且`LoaderManager`会帮我们恢复`Loader`的状态。因此，我们没有必要再去调用`restartLoader`来重新创建`Loader`来执行一次耗时的查询操作。

- `LoaderManager`虽然会恢复`Loader`，但是它不会保存`Callback`实例，因此，如果我们希望在旋转完之后获得数据，那么至少要调用一次`initLoader`来传入一个新的`Callback`进行监听。

在这种情况下，**假如我们在旋转之前`Loader`已经加载数据完毕了，那么`onLoadFinished `会立即被会调**。
## 5.2 没有重建
当没有重建时，不会走`onCreate`方法，因此需要在别的地方来初始化`Loader`。

## 5.3 `LoaderId`
针对上面的这两种情况，我们都需要自己去保存`LoaderId`，在组件恢复之后，通过这个保存的`id`去调用`init/restart`方法，一般情况下，是通过`savedInstanceState`来保存的。

# 六、示例
现在，我们通过一个很简单的例子，来看一下，`initLoader/restartLoader`的区别，我们的`Demo`中有一个`EditText`和一个`TextView`，当`EditText`发生改变时，我们将当前`EditText`的内容作为查询的`Key`，查询任务就是调用`Loader`，延时`2s`，并将这个`key`作为查询的结果展示在`TextView`上。

## 6.1 采用`initLoader`来查询：
```
public class MainActivity extends Activity {

    private static final String LOADER_TAG = "loader_tag";
    private static final String QUERY = "query";

    private MyLoaderCallback mMyLoaderCallback;
    private TextView mResultView;
    private EditText mEditText;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        init();
    }

    private void init() {
        mEditText = (EditText) findViewById(R.id.loader_input);
        mResultView = (TextView) findViewById(R.id.loader_result);
        mEditText.addTextChangedListener(new MyEditTextWatcher());
        mMyLoaderCallback = new MyLoaderCallback();
    }

    private void startQuery(String query) {
        if (query != null) {
            Bundle bundle = new Bundle();
            bundle.putString(QUERY, query);
            getLoaderManager().initLoader(0, bundle, mMyLoaderCallback);
        }
    }

    private void showResult(String result) {
        if (mResultView != null) {
            mResultView.setText(result);
        }
    }

    private static class MyLoader extends BaseDataLoader<String> {

        public MyLoader(Context context, Bundle bundle) {
            super(context, bundle);
        }

        @Override
        protected String loadData(Bundle bundle) {
            Log.d(LOADER_TAG, "loadData");
            try {
                Thread.sleep(2000);
            } catch (Exception e) {
                e.printStackTrace();
            }
            return bundle != null ? bundle.getString(QUERY) : "empty";
        }
    }

    private class MyLoaderCallback implements LoaderManager.LoaderCallbacks {

        @Override
        public Loader onCreateLoader(int id, Bundle args) {
            Log.d(LOADER_TAG, "onCreateLoader");
            return new MyLoader(getApplicationContext(), args);
        }

        @Override
        public void onLoadFinished(Loader loader, Object data) {
            Log.d(LOADER_TAG, "onLoadFinished");
            showResult((String) data);
        }

        @Override
        public void onLoaderReset(Loader loader) {
            Log.d(LOADER_TAG, "onLoaderReset");
            showResult("");
        }
    }

    private class MyEditTextWatcher implements TextWatcher {

        @Override
        public void beforeTextChanged(CharSequence s, int start, int count, int after) {}

        @Override
        public void onTextChanged(CharSequence s, int start, int before, int count) {
            Log.d(LOADER_TAG, "onTextChanged=" + s);
            startQuery(s != null ? s.toString() : "");
        }

        @Override
        public void afterTextChanged(Editable s) {}
    }

}
```
当我们输入`a`时，成功地获取到了数据：
![](http://upload-images.jianshu.io/upload_images/1949836-6c8c076c9f935b6b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
之后，我们继续输入`b`，发现`onLoadFinished`立即被回调了，但是结果还是之前地`a`
![](http://upload-images.jianshu.io/upload_images/1949836-d2fa0a4d212ea663.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
我们通过`AS`的断电发现，整个调用过程如下：
![](http://upload-images.jianshu.io/upload_images/1949836-1acd4e6c7002e976.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
也就是在`initLoader`之后，立即执行了：
```
    public <D> Loader<D> initLoader(int id, Bundle args, LoaderManager.LoaderCallbacks<D> callback) {
        //按照前面的分析，此时的info不为null。
        if (info.mHaveData && mStarted) {
            info.callOnLoadFinished(info.mLoader, info.mData);
        }
        return (Loader<D>)info.mLoader;
    }
```
而之后，`callOnLoadFinished`就会把之前的数据传回给调用者，因此没有执行`onCreateLoader`，也没有进行查询操作：
```
        void callOnLoadFinished(Loader<Object> loader, Object data) {
            //传递数据给调用者.
            if (mCallbacks != null) {
                mCallbacks.onLoadFinished(loader, data);
            }
        }
```
假如，我们在`a`触发的任务还没有执行时就输入`b`，我们看看会发生什么，可以看到，它并不会考虑后来的任务：
![](http://upload-images.jianshu.io/upload_images/1949836-ee7f857d0e40fcee.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 6.2 采用`restartLoader`查询
还是按照前面的方式，我们先输入`a`：
![](http://upload-images.jianshu.io/upload_images/1949836-c986f7d8d9da4418.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
这时候，和采用`initLoader`的结果完全相同，接下来再输入`b`：
![](http://upload-images.jianshu.io/upload_images/1949836-03efbbb32b559acd.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
可以清楚地看到，在执行完`restartLoader`之后，`LoaderManager`回调了`onCreateLoader`方法让我们传入新的`Loader`，并且之后重新进行了查询，并成功地返回了结果。
接下来，我们看一下连续输入的情况：
![](http://upload-images.jianshu.io/upload_images/1949836-b86b130529c7f4ad.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
可以看到，虽然`a`触发的任务已经开始了，但是当我们输入`b`的时候，最终得到的时`ab`这个结果，并且`a`所触发的任务的结果并没有返回给调用者，这也是我们所希望的，因为我们的结果一定是要以用户最后输入的为准。
## 6.3 加上保证正确初始化的代码
最后，我们再加上保证`Loader`能够正确初始化的代码，一个简单的联想输入 - 加载框架就搭建好了。
```
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        init();
        restore(savedInstanceState);
    }
    
    private void save(Bundle outState) {
        if (mEditText != null) {
            outState.putString(QUERY, mEditText.getText().toString());
        }
    }
    
    private void restore(Bundle savedInstanceState) {
        if (savedInstanceState != null) {
            Bundle bundle = new Bundle();
            String query = bundle.getString(QUERY);
            bundle.putString(QUERY, query);
            getLoaderManager().initLoader(0, bundle, mMyLoaderCallback);
        }
    }

    @Override
    protected void onSaveInstanceState(Bundle outState) {
        save(outState);
        super.onSaveInstanceState(outState);
    }
```
