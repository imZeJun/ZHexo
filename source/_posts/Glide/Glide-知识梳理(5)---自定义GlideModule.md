---
title: Glide 知识梳理(5) - 自定义GlideModule
date: 2017-03-13 23:02
categories : Glide 知识梳理
---
# 一、概述
前面说的都是如何使用`Glide`提供的接口来展示图片资源，今天这篇，我们来讲一下如何改变`Glide`的配置。

# 二、定义`GlideModule`
## 2.1 步骤
首先，我们需要一个实现了`GlideModule`接口的类，重写其中的方法来改变`Glide`的配置，然后让`Glide`在构造实例的过程中，读取这个类中的配置信息。

- 第一步：实现`GlideModule`接口
```
public class CustomGlideModule implements GlideModule {

    @Override
    public void applyOptions(Context context, GlideBuilder builder) {
        //通过builder.setXXX进行配置.
    }

    @Override
    public void registerComponents(Context context, Glide glide) {
        //通过glide.register进行配置.
    }
}
```
- 第二步：在`AndroidManifest.xml`中的`<application>`标签下定义`<meta-data>`，这样`Glide`才能知道我们定义了这么一个类，其中`android:name`是我们自定义的`GlideModule`的完整路径，而`android:value`就固定写死`GlideModule`。
```
    <application
        android:allowBackup="true"
        android:icon="@mipmap/ic_launcher"
        android:label="@string/app_name"
        android:supportsRtl="true"
        android:theme="@style/AppTheme">
        <meta-data
            android:name="com.example.lizejun.repoglidelearn.CustomGlideModule"
            android:value="GlideModule"/>
    </application>
```
## 2.2 源码分析
上面`2.1`所做的工作都是为了在`Glide`创建时可以读取我们在两个回调中配置的信息，我们来看一下`Glide`是如何使用这个自定义的类的，它的整个流程在`Glide`的`get`方法中：
```
    public static Glide get(Context context) {
        if (glide == null) {
            synchronized (Glide.class) {
                if (glide == null) {
                    Context applicationContext = context.getApplicationContext();

                    //第一步
                    List<GlideModule> modules = new ManifestParser(applicationContext).parse();
                    
                     //第二步
                    GlideBuilder builder = new GlideBuilder(applicationContext);
                    for (GlideModule module : modules) {
                        //在builder构造出glide之前，读取使用者自定义的配置.
                        module.applyOptions(applicationContext, builder);
                    }
                    glide = builder.createGlide();

                    //第三步
                    for (GlideModule module : modules) {
                        module.registerComponents(applicationContext, glide);
                    }
                }
            }
        }

        return glide;
    }
```
可以看到，整个实例化`Glide`的过程分为三步：
- 第一步：去`AndroidManifest`中查找`meta-data`为`GlideModule`的类，然后通过反射实例化它。
- 第二步：之后`Glide`会新建一个`GlideBuilder`对象，它会先调用我们自定义的`GlideModule`的`applyOptions`方法，并把自己传进去，这样，自定义的`GlideModule`就可以通过`GlideBuilder`提供的接口来设置它内部的参数，在`builder.createGlide()`的过程中就会根据它内部的参数来构建`Glide`，假如我们没有设置相应的参数，那么在`createGlide`时，就会采取默认的实现，下面就是`memoryCache`的例子。
```
    //我们在applyOptions中，可以通过GlideBuilder的这个方法来设定自己的memoryCache.
    public GlideBuilder setMemoryCache(MemoryCache memoryCache) {
        this.memoryCache = memoryCache;
        return this;
    }

    Glide createGlide() {
        //如果builder中没有设定memoryCache，那么采用默认的实现.
        if (memoryCache == null) {
            memoryCache = new LruResourceCache(calculator.getMemoryCacheSize());
        }

        return new Glide(engine, memoryCache, bitmapPool, context, decodeFormat);
    }
```
- 第三步：在`Glide`实例化完毕之后，调用自定义`GlideModule`的`registerComponents`，并传入当前的`Glide`实例来让使用者注册自己的组件，其实在`Glide`实例化的过程中已经注册了默认的组件，如果用户定义了相同的组件，那么就会替换之前的。
```
    Glide(Engine engine, MemoryCache memoryCache, BitmapPool bitmapPool, Context context, DecodeFormat decodeFormat) {
        //Glide默认注册的组件.
        register(File.class, ParcelFileDescriptor.class, new FileDescriptorFileLoader.Factory());
        register(File.class, InputStream.class, new StreamFileLoader.Factory());
        register(int.class, ParcelFileDescriptor.class, new FileDescriptorResourceLoader.Factory());
        register(int.class, InputStream.class, new StreamResourceLoader.Factory());
        register(Integer.class, ParcelFileDescriptor.class, new FileDescriptorResourceLoader.Factory());
        register(Integer.class, InputStream.class, new StreamResourceLoader.Factory());
        register(String.class, ParcelFileDescriptor.class, new FileDescriptorStringLoader.Factory());
        register(String.class, InputStream.class, new StreamStringLoader.Factory());
        register(Uri.class, ParcelFileDescriptor.class, new FileDescriptorUriLoader.Factory());
        register(Uri.class, InputStream.class, new StreamUriLoader.Factory());
        register(URL.class, InputStream.class, new StreamUrlLoader.Factory());
        register(GlideUrl.class, InputStream.class, new HttpUrlGlideUrlLoader.Factory());
        register(byte[].class, InputStream.class, new StreamByteArrayLoader.Factory());
    }
```
通俗的来说，注册组件的目的就是告诉`Glide`，当我们调用`load(xxxx)`方法时，应该用什么方式来获取这个`xxxx`所指向的资源。因此，我们可以看到`register`的第一个参数就是我们`load(xxxx)`的类型，第二个参数是对应的输入流，而第三个参数就是定义获取资源的方式。
我们也就分两个部分，在第三、四节，我们分两部分讨论这两个回调函数的用法：`applyOptions/registerComponents`。

## 2.3 注意事项
- 由于`Glide`是通过反射的方法来实例化`GlideModule`对象的，因此自定义的`GlideModule`只能有一个无参的构造方法。
- 可以看到，上面是支持配置多个`GlideModule`的，但是`GlideModule`的读取顺序并不能保证，因此，不要在多个`GlideModule`对同一个属性进行不同的配置。

# 三、`applyOptions(Context context, GlideBuilder builder)`方法详解
在第二节中，我们已经解释过，这个回调方法的目的就是为了让使用者能通过`builder`定义自己的配置，而所支持的配置也就是`GlideBuilder`的`setXXX`方法，它们包括：
- `setBitmapPool(BitmapPool bitmapPool)`
设置`Bitmap`的缓存池，用来重用`Bitmap`，需要实现`BitmapPool`接口，它的默认实现是`LruBitmapPool`
- `setMemoryCache(MemoryCache memoryCache)`
设置内存缓存，需要实现`MemoryCache`接口，默认实现是`LruResourceCache`。
- `setDiskCache(DiskCache.Factory diskCacheFactory)`
设置磁盘缓存，需要实现`DiskCache.Factory`，默认实现是`InternalCacheDiskCacheFactory`
- `setResizeService(ExecutorService service)`
当资源不在缓存中时，需要通过这个`Executor`发起请求，默认是实现是`FifoPriorityThreadPoolExecutor`。
- `setDiskCacheService(ExecutorService service)`
读取磁盘缓存的服务，默认实现是`FifoPriorityThreadPoolExecutor`。

- `setDecodeFormat(DecodeFormat decodeFormat)`
用于控制`Bitmap`解码的清晰度，`DecodeFormat`可选的值有`PREFER_ARGB_8888/PREFER_RGB_565`，默认为`PREFER_RGB_565`。

# 四、`registerComponents(Context context, Glide glide)`方法详解
`registerComponents`相对来说就复杂了很多，它主要和三个接口有关：
- `ModelLoaderFactory`
- `ModelLoader`
- `DataFetcher`

为了便于理解，我们先通过它内部一个默认`Module`的实现来看一下源码是如何实现的。

我们选取是通用的加载普通图片的`url`的例子，它对应的注册方法是下面这句：
```
Glide(Engine engine, MemoryCache memoryCache, BitmapPool bitmapPool, Context context, DecodeFormat decodeFormat) {
        //注册加载网络url的组件.
        register(GlideUrl.class, InputStream.class, new HttpUrlGlideUrlLoader.Factory());
    }
```
## 4.1 源码分析
### 4.1.1 `HttpUrlGlideUrlLoader.Factory`
首先看一下`HttpUrlGlideUrlLoader`的内部工厂类，它实现了`ModelLoaderFactory<T, Y>`接口
```
public interface ModelLoaderFactory<T, Y> {
    ModelLoader<T, Y> build(Context context, GenericLoaderFactory factories);
    void teardown();
}
```
它要求我们返回一个`ModelLoader`，我们看一下`HttpUrlGlideUrlLoader.Factory`是怎么做的，可以看到，它返回了`HttpUrlGlideUrlLoader`，而它的两个泛型参数就是我们`register`中指定的前两个参数类型。
```
    public static class Factory implements ModelLoaderFactory<GlideUrl, InputStream> {
        private final ModelCache<GlideUrl, GlideUrl> modelCache = new ModelCache<GlideUrl, GlideUrl>(500);

        @Override
        public ModelLoader<GlideUrl, InputStream> build(Context context, GenericLoaderFactory factories) {
            return new HttpUrlGlideUrlLoader(modelCache);
        }

        @Override
        public void teardown() {}
    }
```
### 4.1.2 `HttpUrlGlideUrlLoader`
`HttpUrlGlideUrlLoader`实现了`ModelLoader`接口：
```


public interface ModelLoader<T, Y> {
    DataFetcher<Y> getResourceFetcher(T model, int width, int height);
}
```
`ModelLoader`提供了一个`DataFetcher`，它会去请求这个抽象模型所表示的数据：
- `T`：模型的类型。
- `Y`：一个可以被`ResourceDecoder`解析出数据的表示。

`GlideUrl`的实现如下：
```
public class HttpUrlGlideUrlLoader implements ModelLoader<GlideUrl, InputStream> {

    private final ModelCache<GlideUrl, GlideUrl> modelCache;

    public HttpUrlGlideUrlLoader() {
        this(null);
    }

    public HttpUrlGlideUrlLoader(ModelCache<GlideUrl, GlideUrl> modelCache) {
        this.modelCache = modelCache;
    }

     //最主要的方法，它决定了我们获取数据的方式.
    @Override
    public DataFetcher<InputStream> getResourceFetcher(GlideUrl model, int width, int height) {
        GlideUrl url = model;
        if (modelCache != null) {
            url = modelCache.get(model, 0, 0);
            if (url == null) {
                modelCache.put(model, 0, 0, model);
                url = model;
            }
        }
        return new HttpUrlFetcher(url);
    }
}
```
### 4.1.3 `HttpUrlFetcher`
`DataFetcher`就是我们读取数据的方式，它的关键方法是`loadData`，该`loadData`的返回值就是我们`register`的第二个参数：
```
public interface DataFetcher<T> {
    T loadData(Priority priority) throws Exception;
    void cleanup();
    String getId();
    void cancel();
}
```
`HttpUrlFetcher`实现了`DataFetcher`接口，在它的`loadData`方法中，通过传入的`url`发起请求，最终返回一个`InputStream`。
```
public class HttpUrlFetcher implements DataFetcher<InputStream> {
    private static final String TAG = "HttpUrlFetcher";
    private static final int MAXIMUM_REDIRECTS = 5;
    private static final HttpUrlConnectionFactory DEFAULT_CONNECTION_FACTORY = new DefaultHttpUrlConnectionFactory();

    private final GlideUrl glideUrl;
    private final HttpUrlConnectionFactory connectionFactory;

    private HttpURLConnection urlConnection;
    private InputStream stream;
    private volatile boolean isCancelled;

    public HttpUrlFetcher(GlideUrl glideUrl) {
        this(glideUrl, DEFAULT_CONNECTION_FACTORY);
    }

    HttpUrlFetcher(GlideUrl glideUrl, HttpUrlConnectionFactory connectionFactory) {
        this.glideUrl = glideUrl;
        this.connectionFactory = connectionFactory;
    }

    @Override
    public InputStream loadData(Priority priority) throws Exception {
        return loadDataWithRedirects(glideUrl.toURL(), 0 /*redirects*/, null /*lastUrl*/, glideUrl.getHeaders());
    }

    private InputStream loadDataWithRedirects(URL url, int redirects, URL lastUrl, Map<String, String> headers) throws IOException {
        //就是调用HttpUrlConnection请求.
    }

    @Override
    public void cleanup() {
        if (stream != null) {
            try {
                stream.close();
            } catch (IOException e) {
                // Ignore
            }
        }
        if (urlConnection != null) {
            urlConnection.disconnect();
        }
    }

    @Override
    public String getId() {
        return glideUrl.getCacheKey();
    }

    @Override
    public void cancel() {
        isCancelled = true;
    }
}
```
### 4.1.4 小结
对上面做个总结，整个流程如下：通过`register`传入一个`ModelLoaderFactory<T, Y>`工厂类，该工厂生产的是`ModelLoader<T, Y>`，而这个`ModelLoader`会根据`T`返回一个`DataFetcher<Y>`，在`DataFetcher<Y>`中，我们去获取数据。（在上面的例子中`T`就是`GlideUrl`，`Y`就是`InputStream`）
## 4.2 自定义`ModuleLoader`示例：用`OkHttpClient`替换`HttpURLConnection`
下面的例子来自于这篇文章：
> [`https://futurestud.io/tutorials/glide-module-example-accepting-self-signed-https-certificates`](https://futurestud.io/tutorials/glide-module-example-accepting-self-signed-https-certificates)

- 第一步：定义`ModelLoader`和`ModelLoader.Factory`

```
public class OkHttpGlideUrlLoader implements ModelLoader<GlideUrl, InputStream> {

        @Override
        public ModelLoader<GlideUrl, InputStream> build(Context context, GenericLoaderFactory factories) {
            return new OkHttpGlideUrlLoader(getOkHttpClient());
        }

        @Override
        public void teardown() {}
    }

    @Override
    public DataFetcher<InputStream> getResourceFetcher(GlideUrl model, int width, int height) {
        return new OkHttpGlideUrlFetcher(mOkHttpClient, model);
    }
}
```
- 第二步：`ModelLoader`的`getResourceFetcher`返回一个`DataFetcher`，我们给它传入一个`OkHttpClient`实例，让它通过`OkHttpClient`发起请求。

```
public class OkHttpGlideUrlFetcher implements DataFetcher<InputStream> {

    public OkHttpGlideUrlFetcher(OkHttpClient client, GlideUrl url) {
        this.client = client;
        this.url = url;
    }

    @Override
    public InputStream loadData(Priority priority) throws Exception {
        Request.Builder requestBuilder = new Request.Builder().url(url.toStringUrl());
        for (Map.Entry<String, String> headerEntry : url.getHeaders().entrySet()) {
            String key = headerEntry.getKey();
            requestBuilder.addHeader(key, headerEntry.getValue());
        }
        Request request = requestBuilder.build();
        Response response = client.newCall(request).execute();
        responseBody = response.body();
        if (!response.isSuccessful()) {
            throw new IOException("Request failed with code: " + response.code());
        }
        long contentLength = responseBody.contentLength();
        stream = ContentLengthInputStream.obtain(responseBody.byteStream(), contentLength);
        return stream;
    }

}
```
第三步：在`CustomGlideModule`中注册这个组件：
```
public class CustomGlideModule implements GlideModule {

    @Override
    public void applyOptions(Context context, GlideBuilder builder) {
        //通过builder.setXXX进行配置.
    }

    @Override
    public void registerComponents(Context context, Glide glide) {
        //通过glide.register进行配置.
        glide.register(GlideUrl.class, InputStream.class, new OkHttpGlideUrlLoader.Factory());
    }
}
```
接着我们发起一次请求，通过断点可以发现，调用的是`OkHttpClient`来进行数据的拉取：
![](http://upload-images.jianshu.io/upload_images/1949836-6d07f9ef1c0b3e17.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
## 4.3 自定义处理的`Module`
上面我们分析了如何定义`ModuleLoader`来关联已有的`Module`和最终的数据类型，下面我们介绍一些如何定义自己的`Model`，也就是前面在基础介绍中，所说的`load(Module)`方法。
- 第一步：定义`Module`的接口
```
public interface AutoSizeModel {
    String requestSizeUrl(int width, int height);
}
```
- 第二步：实现`Module`接口
```
public class AutoSizeModelImpl implements AutoSizeModel {

    String mUrl;

    public AutoSizeModelImpl(String url) {
        mUrl = url;
    }

    @Override
    public String requestSizeUrl(int width, int height) {
        return mUrl;
    }
}
```
- 第三步：定义`ModuleLoader`和`ModuleLoader.Factory`
```
public class AutoSizeModelLoader extends BaseGlideUrlLoader<AutoSizeModel> {

    public static class Factory implements ModelLoaderFactory<AutoSizeModel, InputStream> {

        @Override
        public ModelLoader<AutoSizeModel, InputStream> build(Context context, GenericLoaderFactory factories) {
            return new AutoSizeModelLoader(context);
        }

        @Override
        public void teardown() {}
    }

    public AutoSizeModelLoader(Context context) {
        super(context);
    }

    @Override
    protected String getUrl(AutoSizeModel model, int width, int height) {
        return model.requestSizeUrl(width, height);
    }
}
```
- 第四步：在`CustomGlideModule`中进行关联：
```
public class CustomGlideModule implements GlideModule {

    @Override
    public void applyOptions(Context context, GlideBuilder builder) {
        //通过builder.setXXX进行配置.
    }

    @Override
    public void registerComponents(Context context, Glide glide) {
        //通过glide.register进行配置.
        glide.register(AutoSizeModel.class, InputStream.class, new AutoSizeModelLoader.Factory());
    }
}
```
- 第五步：调用
```
    public void loadCustomModule(View view) {
        AutoSizeModelImpl autoSizeModel = new AutoSizeModelImpl("http://i.imgur.com/DvpvklR.png");
        Glide.with(this)
                .load(autoSizeModel)
                .diskCacheStrategy(DiskCacheStrategy.NONE)
                .skipMemoryCache(true)
                .into(mImageView);
    }
```

## 4.4 动态指定`ModelLoader`
在上面的例子中，我们是在自定义的`CustomGlideModule`中指定了`Model`和`ModuleLoader`的关联，当然，我们也可以采用动态指定`ModelLoader`的方法，也就是说，我们去掉`4.3`中的第四步，并把第五步改成下面这样：
```
    public void loadDynamicModule(View view) {
        AutoSizeModelImpl autoSizeModel = new AutoSizeModelImpl("http://i.imgur.com/DvpvklR.png");
        AutoSizeModelLoader autoSizeModelLoader = new AutoSizeModelLoader(this);
        Glide.with(this)
                .using(autoSizeModelLoader)
                .load(autoSizeModel)
                .diskCacheStrategy(DiskCacheStrategy.NONE)
                .skipMemoryCache(true)
                .into(mImageView);
    }
```
使用`using`方法，我们就可以在运行时根据情况为同一个`Module`选择不同类型的`ModuleLoader`了。

# 五、小结
这也是我们`Glide`学习的最后一章，所有的源码都可以从下面的链接中找到：
> [`https://github.com/imZeJun/RepoGlideLearn`](https://github.com/imZeJun/RepoGlideLearn)
