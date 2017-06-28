---
title: Retrofit 知识梳理(1) - 流程分析
date: 2017-03-09 20:10
categories : Retrofit 知识梳理
---
# 一、概述
`Retrofit`之所以能做到如此简洁，最重要的一个原因就是它把网络请求当中复杂的参数设置都封装了起来，对于使用者而言，只需要定义一个`interface`，并在`interface`当中定义好请求的参数，`Retrofit`在构建请求的时候就会帮我们自动配置。
除此之外，它还提供了`Converter/CallAdapter`让使用者进行充分的定制，要理解整个`Retrofit`的架构，还是应当从一个简单的流程开始，一步步地`Debug，`这篇文章，我们就以一个最简单的例子，从创建到返回的流程，来看一下整个`Retrofit`的框架。
## 二、整体流程
下面是一个使用`Retrofit`的最简单的例子，我们将通过这个例子，一步步地`Debug`，看一下整个的过程：
```
    public void simpleExample(View view) {
        Retrofit retrofit = new Retrofit
                .Builder()
                .baseUrl("https://api.github.com/")
                .build();
        GitHubService service = retrofit.create(GitHubService.class);
        Call<ResponseBody> call = service.listRepos("octocat");
        try {
            call.enqueue(new Callback<ResponseBody>() {
                @Override
                public void onResponse(Call<ResponseBody> call, Response<ResponseBody> response) {}
                @Override
                public void onFailure(Call<ResponseBody> call, Throwable t) {}
            });
        } catch (Exception e) {
            e.printStackTrace();
        }

    }
```
# 三、构建`Retrofit`对象
## 3.1 `Retrofit.Builder`构造函数
`Retrofit`的构建，采用了建造者模式，在第`2`步中，我们传入了一个`url`，我们看一下`Retrofit.Builder`做了什么，这里，我们只截取例子中调用了的部分：
```
 public static final class Builder {

    Builder(Platform platform) {
      this.platform = platform;
      converterFactories.add(new BuiltInConverters());
    }

    public Builder() {
      this(Platform.get());
    }
    //....
  }
```
首先，当我们调用无参的构造函数之后，在它的内部会调用`Builder(Platform platform)`，这里面会确定当前的平台，接着给它的`converterFactories`列表当中添加一个默认的`BuiltInConverters`，下面我们看一下这个类的定义。
## 3.2 `Converter / Converter.Factory / BuiltInConverters `
`BuiltInConverters`继承了`Converter.Factory`这个抽象类：
```
final class BuiltInConverters extends Converter.Factory { }
```
`Converter`是一个带有`convert`的接口，它就是把一个类型转换成另一个类型，但是由于使用阶段的不同，所以它有三个工厂方法。
至于为什么要定义这个，这里先说结论：
- 因为`Retrofit`在处理返回的请求时，默认是把`Response`转换为`ResponseBody`，如果使用者希望得到别的类型，那么就需要定义自己的`Converter.Factory`，并将它设置给`Retrofit`。在网上很多例子中使用的`GsonConverterFactory`就是用来做这个事的，它可以把标准的`Call<ResponseBody>`转换成`Call<任意类型>`，而从`ResponseBody`到`任意类型`的`JSON`解析也是由它完成的。
- 而后两个方法，则是用来初始化`ServiceMethod`中的变量，这些变量又会决定发起请求的`Request`的参数，`Retrofit`抽象出了其中可定制的部分来给使用者。

```
public interface Converter<F, T> {
  //和它的类名类似，只有一个接口，就是进行一个类型的转换。
  T convert(F value) throws IOException;

  //内部工厂类，调用不同的工厂方法来生成Converter
  abstract class Factory {

    //(1)将返回的ResponseBody转换成interface中Call<T>所指定的T.
    public Converter<ResponseBody, ?> responseBodyConverter(Type type, Annotation[] annotations, Retrofit retrofit) {
        return null;
    }

    //(2)利用参数构建Request，它依赖于@Body/@Part/@PartMap.
    public Converter<?, RequestBody> requestBodyConverter(Type type, Annotation[] parameterAnnotations, Annotation[] methodAnnotations, Retrofit retrofit) {
        return null;
    }

    //(3)利用参数构建String，它依赖于@Field/@FieldMap/@Header/@HeaderMap/@Path/@Query/@QueryMap.
    public Converter<?, String> stringConverter(Type type, Annotation[] annotations, Retrofit retrofit) {
        return null;
    }
  }
}
```
在上面的例子中，我们最终是会得到一个`BuiltInConverters `实例，它继承了`Convert.Factory`，并重写了`response/request`这两个方法，它会根据传入的`Type`来判断生成哪个`Convert`实例，因为我们现在不知道这两个方法是什么时候调用的，所以我们先大概了解一下它的逻辑，之后再来解释。
```
final class BuiltInConverters extends Converter.Factory {
  
  @Override
  public Converter<ResponseBody, ?> responseBodyConverter(Type type, Annotation[] annotations,
      Retrofit retrofit) {
    //如果Type是ResponseBody，那么当注解中包含了Streaming.class时，返回StreamingXXX，否则返回BufferingXXX
    //这两个其实都是Converter<ResponseBody, ResponseBody>
    if (type == ResponseBody.class) {
      return Utils.isAnnotationPresent(annotations, Streaming.class)
          ? StreamingResponseBodyConverter.INSTANCE
          : BufferingResponseBodyConverter.INSTANCE;
    }
    //如果Type为空，那么返回null。
    if (type == Void.class) {
      return VoidResponseBodyConverter.INSTANCE;
    }
    return null;
  }

  @Override
  public Converter<?, RequestBody> requestBodyConverter(Type type,
      Annotation[] parameterAnnotations, Annotation[] methodAnnotations, Retrofit retrofit) {
    //如果是RequestBody，或者是它的父类或者接口，那么返回Convert<RequestBody, RequestBody>，否则返回null。
    if (RequestBody.class.isAssignableFrom(Utils.getRawType(type))) {
      return RequestBodyConverter.INSTANCE;
    }
    return null;
  }
}
```
## 3.3 `baseUrl(String url)`方法
现在`Builder`的构造函数已经解释完了，那么我们再来看一下`baseUrl`方法，这个没什么好说的，就是把`String`转换成为`HttpUrl`：
```
    public Builder baseUrl(String baseUrl) {
      checkNotNull(baseUrl, "baseUrl == null");
      HttpUrl httpUrl = HttpUrl.parse(baseUrl);
      if (httpUrl == null) {
        throw new IllegalArgumentException("Illegal URL: " + baseUrl);
      }
      return baseUrl(httpUrl);
    }

    public Builder baseUrl(HttpUrl baseUrl) {
      checkNotNull(baseUrl, "baseUrl == null");
      List<String> pathSegments = baseUrl.pathSegments();
      if (!"".equals(pathSegments.get(pathSegments.size() - 1))) {
        throw new IllegalArgumentException("baseUrl must end in /: " + baseUrl);
      }
      this.baseUrl = baseUrl;
      return this;
    }
```
## 3.4 调用`build`方法，构建`Retrofit`对象
```
    public Retrofit build() {
      //1.这个就是传入的“https://api.github.com/”
      if (baseUrl == null) {
        throw new IllegalStateException("Base URL required.");
      }

      //2.默认采用OkHttpClient
      okhttp3.Call.Factory callFactory = this.callFactory;
      if (callFactory == null) {
        callFactory = new OkHttpClient();
      }

      //3.根据平台选择相应的执行者。
      Executor callbackExecutor = this.callbackExecutor;
      if (callbackExecutor == null) {
        callbackExecutor = platform.defaultCallbackExecutor();
      }

      //4.添加默认的CallAdapter.Factory，默认的Factory放在最后一个
      List<CallAdapter.Factory> adapterFactories = new ArrayList<>(this.adapterFactories);
      adapterFactories.add(platform.defaultCallAdapterFactory(callbackExecutor));

      //5.添加Converter.Factory，默认BuiltInConverters放在第一个.
      List<Converter.Factory> converterFactories = new ArrayList<>(this.converterFactories);

      //6.返回，内部其实就是给各个变量赋值，并把45生成的列表定义为不可更改的。
      return new Retrofit(callFactory, baseUrl, converterFactories, adapterFactories,
          callbackExecutor, validateEagerly);
    }
```
为了之后分析方便，我们看一下上面没有接触过的两个类：`Executor`和`CallAdapter.Factory`，因为我们是`Android`平台，因此我们直接看这两个平台中返回的实例：
```
static class Android extends Platform {
     //返回MainThreadExecutor
    @Override public Executor defaultCallbackExecutor() {
      return new MainThreadExecutor();
    }
    //返回ExecutorCallAdapterFactory
    @Override CallAdapter.Factory defaultCallAdapterFactory(Executor callbackExecutor) {
      return new ExecutorCallAdapterFactory(callbackExecutor);
    }
  }
```
下面，我们通过`3.4`和`3.5`来看一下这两个实例的内部实现。
## 3.5 `Executor / MainThreadExecutor`
对于`Android`平台，返回的是`MainThreadExecutor`，它其实就是把`Runnable`中的任务放到主线程中去执行，后面我们可以知道，它的作用就是**把异步执行的结果返回给主线程**。
```
    static class MainThreadExecutor implements Executor {
      private final Handler handler = new Handler(Looper.getMainLooper());
      @Override public void execute(Runnable r) {
        handler.post(r);
      }
    }
```
## 3.6 `CallAdapter / CallAdapter.Factory / ExecutorCallAdapterFactory`
`CallAdapter`的定义和我们上面看到的`Converter`很类似，都是定义了接口，然后内部有一个抽象工厂类：
```
public interface CallAdapter<R, T> {
  
  //(1)返回类型.
  Type responseType();

  //(2)将Call<R>转换为T.
  T adapt(Call<R> call);

  //内部抽象工厂.
  abstract class Factory {

    //(1)工厂方法，工厂类需要实现该方法来返回对应的CallAdapter.
    public abstract CallAdapter<?, ?> get(Type returnType, Annotation[] annotations, Retrofit retrofit);

    protected static Type getParameterUpperBound(int index, ParameterizedType type) {
      return Utils.getParameterUpperBound(index, type);
    }

    protected static Class<?> getRawType(Type type) {
      return Utils.getRawType(type);
    }
  }
}
```
看上面的描述不是很清楚，我们看一下`ExecutorCallAdapterFactory`是怎么实现它的：
```
final class ExecutorCallAdapterFactory extends CallAdapter.Factory {
  final Executor callbackExecutor;

  //这里传入的Executor，就是前面我们构造的MainThreadExecutor，即把任务放到主线程中执行.
  ExecutorCallAdapterFactory(Executor callbackExecutor) {
    this.callbackExecutor = callbackExecutor;
  }
   
   //只重写了get方法，没有重写另外两个方法.
  @Override
  public CallAdapter<?, ?> get(Type returnType, Annotation[] annotations, Retrofit retrofit) {
    //....
    //获得Call<?>中类型，对于例子来说，就是ResponseBody.
    final Type responseType = Utils.getCallResponseType(returnType);
    
    //get方法返回的是CallAdapter对象
    return new CallAdapter<Object, Call<?>>() {

       //(1)对于第一个接口而言，就是Call<T>中T的类型.
      @Override public Type responseType() {
        return responseType;
      }

      //(2)对于第二个接口，就是把传入的Call<Object>进行了包装，并没有改变它的类型.
      @Override public Call<Object> adapt(Call<Object> call) {
        return new ExecutorCallbackCall<>(callbackExecutor, call);
      }
    };
  }
  //..定义了ExecutorCallbackCall.
}
```
## 3.7 `Call / Callback / ExecutorCallbackCall`
`ExecutorCallbackCall`实现了`Call`接口，我们先看一下`Call`接口的定义：
```
//T表示的是返回的类型，也就是前面的List<Repo>.
public interface Call<T> extends Cloneable {
  //同步执行
  Response<T> execute() throws IOException;
  //异步执行
  void enqueue(Callback<T> callback);
  //正在执行.
  boolean isExecuted();
  //取消.
  void cancel();
  //已经取消.
  boolean isCanceled();
  //参数并且目标服务器相同的请求，那么调用clone方法.
  Call<T> clone();
  //最初始的请求.
  Request request();
}
```
接下来，看一下`Callback`的定义：
```
//T表示的是最终返回结果的类型，也就是List<Repo>
public interface Callback<T> {
  //返回成功.
  void onResponse(Call<T> call, Response<T> response);
  //返回失败.
  void onFailure(Call<T> call, Throwable t);
}
```
`ExecutorCallbackCall`的构造函数中传入了一个`Executor`和一个`Call`作为`delegate`，其实它对于`Call`接口的实现，都是调用了`delegate`的对应实现，**唯一不同是对于`enqueue`方法，它会把`callback`的回调放在`Executor`的`run()`方法中执行**。
```
    delegate.enqueue(new Callback<T>() {
        //在实际执行任务的Call完成之后，调用MainThreadExecutor，使得使用者收到的回调是运行在主线程当中的.
        @Override public void onResponse(Call<T> call, final Response<T> response) {
          //通过executor执行.
          callbackExecutor.execute(new Runnable() {
            @Override public void run() {
              //如果取消了，那么仍然算作失败.
              if (delegate.isCanceled()) {
                callback.onFailure(ExecutorCallbackCall.this, new IOException("Canceled"));
              } else {
                callback.onResponse(ExecutorCallbackCall.this, response);
              }
            }
          });
        }

        @Override public void onFailure(Call<T> call, final Throwable t) {
          callbackExecutor.execute(new Runnable() {
            @Override public void run() {
              callback.onFailure(ExecutorCallbackCall.this, t);
            }
          });
        }
      });
    }
```
## 3.8 构造`Retrofit`小结
从上面的分析来看，这个`Retrofit`最终它内部的成员变量包括：
```
  final okhttp3.Call.Factory callFactory; //OkHttpClient.
  final HttpUrl baseUrl; //baseUrl传入的值.
  final List<Converter.Factory> converterFactories; //列表中包括一个BuildInConverts.
  final List<CallAdapter.Factory> adapterFactories; //列表中包括一个ExecutorCallAdapterFactory.
  final Executor callbackExecutor; //MainThreadExecutor.
  final boolean validateEagerly; //没有传入，默认为false.
```
我们在`AS`当中断点，可以看到此时`Retrofit`内部的成员变量，和我们上面通过源码的分析的结果是相同的。

![`Retrofit`的成员变量值](http://upload-images.jianshu.io/upload_images/1949836-6f9f1d701fdf3aa0.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
# 四、`GitHubService service = retrofit.create(GitHubService.class);`
这一步其实就是构建一个动态代理，真正执行的是要等到下面这句：
```
Call<ResponseBody> call = service.listRepos("octocat");
```
我们直接来看执行的部分，它可以分为三个步骤，对于这三个步骤，我们分为五、六、七这三个章节说明。
```
  @SuppressWarnings("unchecked") // Single-interface proxy creation guarded by parameter safety.
  public <T> T create(final Class<T> service) {
    //...忽略
    return (T) Proxy.newProxyInstance(service.getClassLoader(), new Class<?>[] { service },
        new InvocationHandler() {
          private final Platform platform = Platform.get();

          @Override public Object invoke(Object proxy, Method method, Object[] args)
              throws Throwable {
            //...

            //1.得到ServiceMethod对象.
            ServiceMethod<Object, Object> serviceMethod = (ServiceMethod<Object, Object>) loadServiceMethod(method);

            //2.通过serviceMethod和args来构建OkHttpCall对象，args就是“octocat”.
            OkHttpCall<Object> okHttpCall = new OkHttpCall<>(serviceMethod, args);

            //3.其实就是调用ExecutorCallbackCall.adapt方法.
            return serviceMethod.callAdapter.adapt(okHttpCall);
          }
        });
  }
```
#五、实例化`ServiceMethod<Object, Obejct>`对象。
## 5.1 `loadServiceMethod`
```

  ServiceMethod<?, ?> loadServiceMethod(Method method) {
      //如果之前加载过，那么直接返回,
      if (result == null) {
        //创建新的.
        result = new ServiceMethod.Builder<>(this, method).build();
        //加入缓存.
        serviceMethodCache.put(method, result);
      }
    }
    return result;
  }
```
在`ServiceMethod.Builder`当中，通过`method`首先初始化了下面这几个变量，可以看到它持有了`retrofit`实例以及我们定义的`GitHubService`这个接口内的方法，也就是说，它知道了所有和请求相关的信息：
```
    Builder(Retrofit retrofit, Method method) {
      this.retrofit = retrofit; //就是前面构建的retrofit实例.
      this.method = method; //方法，对应于listRepo
      this.methodAnnotations = method.getAnnotations(); //方法的注解，对应于@GET("users/{user}/repos")
      this.parameterTypes = method.getGenericParameterTypes(); //方法的形参的类型，对应于String
      this.parameterAnnotationsArray = method.getParameterAnnotations(); //方法的形参前的注解，对应于@Path("user")
    }
```
我们通过断点看一下目前`ServiceMethod`中这几个成员变量的值：

![`ServiceMethod.Builder`初始化的值](http://upload-images.jianshu.io/upload_images/1949836-5bd3a7046c4f70ad.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
## 5.2 `ServiceMethod.Builder`的`build()`过程
上面只是进行简单的初始化，而真正构建的过程会依赖这几个参数来初始化`ServiceMethod`中的其它成员变量，这个方法比较长，也是**`Retrofit`的核心部分**，我们去掉一些参数检查的代码，直接来看关键的五个步骤：
```
    public ServiceMethod build() {
      //1.初始化CallAdapter<T, R>，调用了retrofit.callAdapter.
      callAdapter = createCallAdapter();

      //2.得到callAdapter的返回类型.
      responseType = callAdapter.responseType();
    
      //3.初始化Converter<ResponseBody, T>，调用了retrofit.responseBodyConverter
      responseConverter = createResponseConverter();

      //4.处理方法的注解,
      for (Annotation annotation : methodAnnotations) {
        parseMethodAnnotation(annotation);
      }

      //5.处理形参的注解.
      int parameterCount = parameterAnnotationsArray.length;
      parameterHandlers = new ParameterHandler<?>[parameterCount];
      for (int p = 0; p < parameterCount; p++) {
        //首先得到形参的类型.
        Type parameterType = parameterTypes[p];
        //再得到形参的注解.
        Annotation[] parameterAnnotations = parameterAnnotationsArray[p];
        //(1)根据这两个值来初始化和该形参关联的ParameterHandler.
        //(2)其内部会调用parseParameterAnnotation
        //(3)而该方法最终会调用Retrofit.requestBodyConverter/stringConverter
        parameterHandlers[p] = parseParameter(p, parameterType, parameterAnnotations);
      }
      return new ServiceMethod<>(this);
    }
```
下面我们一步步来分析这些关键语句的作用：
### 5.2.1 第一步：构建`CallAdapter`
```
关键语句：callAdapter = createCallAdapter();
```
看一下`createCallAdapter`：
```
    private CallAdapter<T, R> createCallAdapter() {

      //方法的返回值，也就是Call<ResponseBody>
      Type returnType = method.getGenericReturnType();

      //方法的注解，就是@GET(xxxx)
      Annotation[] annotations = method.getAnnotations();

      //把返回类型和注解传给Retrofit.
      return (CallAdapter<T, R>) retrofit.callAdapter(returnType, annotations);  
    }
```
这个方法执行时，`returnType`和`annotation`的值为：

![`createCallAdapter`的`returnType/annotations`的值](http://upload-images.jianshu.io/upload_images/1949836-1490e74d5138afe7.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
下面，我们看一下`Retrofit`中的对应方法：
```
  public CallAdapter<?, ?> callAdapter(Type returnType, Annotation[] annotations) {
    return nextCallAdapter(null, returnType, annotations);
  }

  public CallAdapter<?, ?> nextCallAdapter(CallAdapter.Factory skipPast, Type returnType,
      Annotation[] annotations) {

    //遍历之前初始化时的adapterFactories.
    int start = adapterFactories.indexOf(skipPast) + 1;

    //1.此时为null,所以start从0开始遍历所有配置的CallAdapterFactory
    //2.然后调用它们的工厂方法，找到第一个满足returnType和annotation的CallAdapter返回.
    for (int i = start, count = adapterFactories.size(); i < count; i++) {
      CallAdapter<?, ?> adapter = adapterFactories.get(i).get(returnType, annotations, this);
      if (adapter != null) {
        return adapter;
      }
    }
  }
```
上面这步返回的`CallAdapter`其实就是我们之前分析过的，通过`ExecutorCallAdapterFactory`的`get`方法所返回的`CallAdaper`，它的`responseType()`得到就是`Call<ResponseBody>`中的`ResponseBody`。
```
  @Override
  public CallAdapter<?, ?> get(Type returnType, Annotation[] annotations, Retrofit retrofit) {

    //1.先判断返回值是不是Call<T>
    if (getRawType(returnType) != Call.class) {
      return null;
    }

    //2.得到T的类型.
    final Type responseType = Utils.getCallResponseType(returnType);

    //3.利用T来决定CallAdapter的returnType返回什么.
    return new CallAdapter<Object, Call<?>>() {
      @Override public Type responseType() {
        return responseType;
      }

      @Override public Call<Object> adapt(Call<Object> call) {
        return new ExecutorCallbackCall<>(callbackExecutor, call);
      }
    };
  }
```
所以，当第一步结束后，我们得到的就是`callAdapter`其实就是下面这个：
![](http://upload-images.jianshu.io/upload_images/1949836-3e6339796b1f9711.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
**小结：这一步就是得到`CallAdapter`，决定它的是`interface`的返回类型和注解**。
### 5.2.2 第二步：`responseType = callAdapter.responseType()`
这一步很好理解，就是调用第一步中生成的`callAdapter`的方法，对于示例来说就是`ResponseBody`：
![](http://upload-images.jianshu.io/upload_images/1949836-47ace9d45d1d286a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
**小结：这一步就是调用第一步中生成的`CallAdapter#responseType`来赋值给`responseType`**。
### 5.2.3 第三步：`responseConverter = createResponseConverter()`
```
    private Converter<ResponseBody, T> createResponseConverter() {
      Annotation[] annotations = method.getAnnotations();
      //这里和第一步不同，不是传入接口方法的返回类型，而是传入第二步中通过CallAdapter＃responseType返回的类型，对于例子来说，就是ResponseBody.
      return retrofit.responseBodyConverter(responseType, annotations);
    }
```
下面的代码和获得`CallAdapter`的代码很类似，都是遍历工厂，来找到可以生成符合需求的产品.
```
  public <T> Converter<ResponseBody, T> responseBodyConverter(Type type, Annotation[] annotations) {
    return nextResponseBodyConverter(null, type, annotations);
  }

  public <T> Converter<ResponseBody, T> nextResponseBodyConverter(Converter.Factory skipPast,
      Type type, Annotation[] annotations) {

    int start = converterFactories.indexOf(skipPast) + 1;
    for (int i = start, count = converterFactories.size(); i < count; i++) {
      Converter<ResponseBody, ?> converter = converterFactories.get(i).responseBodyConverter(type, annotations, this);
      return (Converter<ResponseBody, T>) converter;
    }

  }
```
根据我们之前的分析，它就是调用了`BuiltInConverters`这个工厂类，最终生产出下面的这个`Converter`：
```
  static final class BufferingResponseBodyConverter implements Converter<ResponseBody, ResponseBody> {
    static final BufferingResponseBodyConverter INSTANCE = new BufferingResponseBodyConverter();

    @Override public ResponseBody convert(ResponseBody value) throws IOException {
      try {
        // Buffer the entire body to avoid future I/O.
        return Utils.buffer(value);
      } finally {
        value.close();
      }
    }
  }
```
断点的值：
![](http://upload-images.jianshu.io/upload_images/1949836-d2814cbe440309af.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
经过第三步的分析，我们要明白两点：
- **`responseBodyConverter`这个`Converter`的作用就很清楚了，就是如果我们要把`Call<ResponseBody>`中的`ResponseBody`转换为其它的类型，那么就需要通过这个`Converter`类实现**。
- **`responseBodyConverter`方法所传入的`type`，其实是`CallAdapter#responseType`的类型，这也是它们相关联的地方**。

**小结：这一步就是得到`Converter`，它是由`CallAdapter#responseType`和`interface`的注解来决定的**
### 5.2.4 第四步：处理方法的注解 
```
for (Annotation annotation : methodAnnotations) {
    parseMethodAnnotation(annotation);
}
```
```
private void parseMethodAnnotation(Annotation annotation) {
      //针对注解不同，有不同的处理方式.
      if (annotation instanceof DELETE) {
        parseHttpMethodAndPath("DELETE", ((DELETE) annotation).value(), false);
      } else if (annotation instanceof GET) {
        //我们的例子调用了这个方法.
        parseHttpMethodAndPath("GET", ((GET) annotation).value(), false);
      } else {
            //....
      }
}
```
这个步骤完毕之后，之后再调用`parseHttpMethodAndPath()`方法，来给下面几个变量赋值：
```
String httpMethod; //"GET"
boolean hasBody; //false
String relativeUrl; // users/{user}/repos
Set<String> relativeUrlParamNamesl; // 大小为1，内容为user.
```
### 5.2.5 第五步：处理形参的注解
处理形参，依赖于该**形参上的注解和形参的类型**：
```
      //只处理有注解的形参.
      int parameterCount = parameterAnnotationsArray.length;
      //每个有注解的形参对应一个ParameterHandler.
      parameterHandlers = new ParameterHandler<?>[parameterCount];
      for (int p = 0; p < parameterCount; p++) {
        //1.获得形参的类型.
        Type parameterType = parameterTypes[p];
        //2.获得该形参对应的注解.
        Annotation[] parameterAnnotations = parameterAnnotationsArray[p];
        //3.传入这两个变量，来构建ParameterHandler.
        parameterHandlers[p] = parseParameter(p, parameterType, parameterAnnotations);
      }
```
我们再来看一下`parseParameter`这个函数：
```
    private ParameterHandler<?> parseParameter(int p, Type parameterType, Annotation[] annotations) {
      ParameterHandler<?> result = null;
      //遍历该形参上的所有注解.
      for (Annotation annotation : annotations) {
        //这里处理，所以应该是每个有注解的形参上的每个注解都会对应一个ParameterHandler.
        ParameterHandler<?> annotationAction = parseParameterAnnotation(p, parameterType, annotations, annotation);
        if (annotationAction == null) {
            continue;
        }
        //每个形参上只允许有一个注解.
        if (result != null) {
            throw parameterError(p, "Multiple Retrofit annotations found, only one allowed.");
        }

        result = annotationAction;
      }

      if (result == null) {
          throw parameterError(p, "No Retrofit annotation found.");
      }

      return result;
   }
```
这个过程之后，`ParameterHandler`的值变为：
![](http://upload-images.jianshu.io/upload_images/1949836-24a13e635ea02ec3.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
## 5.3 小结
整个过程结束之后，`ServiceMethod`内的变量变为：
![](http://upload-images.jianshu.io/upload_images/1949836-4687180fc6a71d59.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
# 六、构造`OkHttpCall`
这一步只是简单的把这两个变量传入，并返回`OkHttpCall`，`OkHttpCall`是实现了`Call`接口的，关于它的具体实现，我们在后面发起请求的部分再讨论。
```
    OkHttpCall(ServiceMethod<T, ?> serviceMethod, Object[] args) {
        this.serviceMethod = serviceMethod;
        this.args = args;
    }
```
# 七、将`OkHttpCall<Object>`转换为`interface`声明的类型
```
return serviceMethod.callAdapter.adapt(okHttpCall);
```
这里面调用的就是`ServiceMethod`中的`callAdapter`，经过我们前面的分析，这个`CallAdapter`就是`ExecutorCallAdapterFactory`，它的`adapt`方法，其实就是把`OkHttpCall<Object>`和`MainExecutor`包装在一起，然后返回那个对它的包装类。
**`CallAdapter`一共有两个接口，前面我们已经看到`CallAdapter#responseType`是用来辅助`responseConvert`的生成，那是它的第一个作用；现在我们可以知道，它的第二个接口`adapt`的作用就是把`OkHttpCall<Object>`转换成为自己希望的类型**。
# 八、发起请求
经过上面几章的分析，我们的准备工作已经做好了，现在我们调用下面的`enqueue`方法发起请求：
```
call.enqueue(new Callback<ResponseBody>() {
                @Override
                public void onResponse(Call<ResponseBody> call, Response<ResponseBody> response) {}
                @Override
                public void onFailure(Call<ResponseBody> call, Throwable t) {}
});
```
经过前面的分析，我们知道这个`call`其实就是`ExecutorCallbackCall`类型的，它的`enqueue`方法，其实是调用了构建它时所传入的`Call<T> delegate`的方法，而此时`delegate`就是`OkHttpCall<ResponseBody>`，我们取看一下它的`enqueue`方法。
```
@Override 
 public void enqueue(final Callback<T> callback) {
    //1.构建请求
    call = rawCall = createRawCall();
    //2.发起请求.
    call.enqueue(new okhttp3.Callback() {
      @Override public void onResponse(okhttp3.Call call, okhttp3.Response rawResponse)
          throws IOException {
        Response<T> response;
        try {
          //3.解析请求
          response = parseResponse(rawResponse);
        } catch (Throwable e) {
          callFailure(e);
          return;
        }
        //4.返回请求.
        callSuccess(response);
      }

      @Override public void onFailure(okhttp3.Call call, IOException e) {
        try {
          callback.onFailure(OkHttpCall.this, e);
        } catch (Throwable t) {
          t.printStackTrace();
        }
      }
    });
  }
```
## 8.1 构建请求

```
  private okhttp3.Call createRawCall() throws IOException {
    //通过ServiceMethod来构造OkHttpClient的Request实例，并传入方法的实参.
    Request request = serviceMethod.toRequest(args);
    okhttp3.Call call = serviceMethod.callFactory.newCall(request);
    if (call == null) {
      throw new NullPointerException("Call.Factory returned null.");
    }
    return call;
  }
```
这里会根据前面`loadServiceMethod`过程当中所生成的参数，来最终初始化一个`Request`.
```
  /** Builds an HTTP request from method arguments. */
  Request toRequest(Object... args) throws IOException {
    RequestBuilder requestBuilder = new RequestBuilder(httpMethod, baseUrl, relativeUrl, headers, contentType, hasBody, isFormEncoded, isMultipart);
    ParameterHandler<Object>[] handlers = (ParameterHandler<Object>[]) parameterHandlers;
    
    for (int p = 0; p < argumentCount; p++) {
      handlers[p].apply(requestBuilder, args[p]);
    }

    return requestBuilder.build();
  }
```
下面是断点之后的值：
![](http://upload-images.jianshu.io/upload_images/1949836-bd41fa8a4a74fa8c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
在构建完`Request`之后，再调用`serviceMethod.callFactory`来生成`okHttp3.Call`，这里的`callFactory`就是`Retrofit`中的`callFactory`，也就是`OkHttpClient`。
## 8.2 解析请求
从上面的一节可以看到，最终发起请求的使用调用`OkHttp`标准的流程，那么返回的时候也是用的标准的`OkHttp`的`Response`，接下来，就会调用下面的方法来把它转换成`Response<T>`：
```
  Response<T> parseResponse(okhttp3.Response rawResponse) throws IOException {
    //1.获得Body
    ResponseBody rawBody = rawResponse.body();
    //2.
    rawResponse = rawResponse.newBuilder()
        .body(new NoContentResponseBody(rawBody.contentType(), rawBody.contentLength()))
        .build();

    ExceptionCatchingRequestBody catchingBody = new ExceptionCatchingRequestBody(rawBody);
    try {
      //构建Body.
      T body = serviceMethod.toResponse(catchingBody);
      return Response.success(body, rawResponse);
    } catch (RuntimeException e) {
      // If the underlying source threw an exception, propagate that rather than indicating it was
      // a runtime exception.
      catchingBody.throwIfCaught();
      throw e;
    }
  }
```
这里面比较关键的是调用`serviceMethod`的`toResponse`方法，它会将对于`Retrofit`标准的`ResponseBody`转换为`T`：
```
  /** Builds a method return value from an HTTP response body. */
  R toResponse(ResponseBody body) throws IOException {
    return responseConverter.convert(body);
  }
```
这里的`responseConverter`就是我们之前通过`BuildInConvert`生成的`Converter`，回想一下，它什么也没有做，只是把`ResponseBody`原封不动地返回：
```
  @Override
  public Converter<ResponseBody, ?> responseBodyConverter(Type type, Annotation[] annotations, Retrofit retrofit) {
    if (type == ResponseBody.class) {
      return Utils.isAnnotationPresent(annotations, Streaming.class)
          ? StreamingResponseBodyConverter.INSTANCE
          : BufferingResponseBodyConverter.INSTANCE;
    }
    if (type == Void.class) {
      return VoidResponseBodyConverter.INSTANCE;
    }
    return null;
  }

  static final class BufferingResponseBodyConverter implements Converter<ResponseBody, ResponseBody> {

    static final BufferingResponseBodyConverter INSTANCE = new BufferingResponseBodyConverter();

    @Override public ResponseBody convert(ResponseBody value) throws IOException {
      try {
        // Buffer the entire body to avoid future I/O.
        return Utils.buffer(value);
      } finally {
        value.close();
      }
    }
  }
```
因此这一步最终会返回一个`Response<T>`对象，对于例子来说，这个`T`就是`ResponseBody`。
## 8.3 返回请求
加入请求成功，那么会用回调来通知使用者。
```
      private void callSuccess(Response<T> response) {
        try {
          callback.onResponse(OkHttpCall.this, response);
        } catch (Throwable t) {
          t.printStackTrace();
        }
      }
```
因此，最终使用者通过该回调，会收到一个`Reponse<T>`的响应，以及对于该请求的包装类`Call<T>`，也就是调用了`.enqueue`方法的那个对象。
