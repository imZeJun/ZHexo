---
title: 插件化知识梳理(6) - Small 源码分析之 Hook 原理
date: 2017-06-07 21:12
categories : 插件化知识梳理
---
# 一、前言
至此，花了四天时间、五篇文章，学习了如何使用`Small`框架来实现插件化。但是，对于我来说，一开始的目标就不是满足于仅仅知道如何用，而是希望通过这一框架作为平台，学习插件化中所用到的知识。

对于许多插件化的开源框架而言，一个比较核心的部分就是`Hook`的实现，所谓`Hook`，简单地来说就是在应用侧启动`A.Activity`，但是在`AMS`看来却是启动的`B.Activity`，之后`AMS`通知应用侧后，我们再重新替换成`A.Activity`。

在阅读这篇文章之前，大家可以先看一下之前的这篇文章 [Framework 源码解析知识梳理(1) - 应用进程与 AMS 的通信实现](http://www.jianshu.com/p/f26a76e19a2d) ，`Small`其实就是通过替换这一双向通信过程中的关键类，对调用方法中传递的参数进行替换，来实现`Hook`机制。

# 二、源码分析
`Hook`的过程是`Small`预初始化的第一步，就是我们前面在自定义的`Application`构造方法中所进行的操作：
```
public class SmallApp extends Application {

    public SmallApp() {
        Small.preSetUp(this);
    }

}
```
在`Small`的`preSetUp(Application context)`函数中，做了下面的两件事：
- 实例化三个`BundleLauncher`的实现类，添加到`Bundle`类中的静态变量`sBundleLaunchers`中，这三个类的继承关系为：
![](http://upload-images.jianshu.io/upload_images/1949836-b4e009b90f35b851.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

- 依次调用这个三个实现类的`onCreate()`方法。

```
    public static void preSetUp(Application context) {
        //1.添加关键的 BundleLauncher。
        registerLauncher(new ActivityLauncher());
        registerLauncher(new ApkBundleLauncher());
        registerLauncher(new WebBundleLauncher());
        //2.调用 BundleLauncher 的 onCreate() 方法。
        Bundle.onCreateLaunchers(context);
    }
```
对于之前添加进入的三个实现类，只有`ApkBundleLauncher()`实现了`onCreate()`方法，其它两个都是空实现。
```
    protected static void onCreateLaunchers(Application app) {
        //调用之前添加进入的 BundleLauncher 的 onCreate() 方法。
        for (BundleLauncher launcher : sBundleLaunchers) {
            launcher.onCreate(app);
        }
    }
```
我们看一下`ApkBundleLauncher`的内部实现，这里就是`Hook`的实现代码：
```
    @Override
    public void onCreate(Application app) {
        super.onCreate(app);

        Object/*ActivityThread*/ thread;
        List<ProviderInfo> providers;
        Instrumentation base;
        ApkBundleLauncher.InstrumentationWrapper wrapper;
        Field f;

        // Get activity thread
        thread = ReflectAccelerator.getActivityThread(app);

        // Replace instrumentation
        try {
            f = thread.getClass().getDeclaredField("mInstrumentation");
            f.setAccessible(true);
            base = (Instrumentation) f.get(thread);
            wrapper = new ApkBundleLauncher.InstrumentationWrapper(base);
            f.set(thread, wrapper);
        } catch (Exception e) {
            throw new RuntimeException("Failed to replace instrumentation for thread: " + thread);
        }

        // Inject message handler
        ensureInjectMessageHandler(thread);

        // Get providers
        try {
            f = thread.getClass().getDeclaredField("mBoundApplication");
            f.setAccessible(true);
            Object/*AppBindData*/ data = f.get(thread);
            f = data.getClass().getDeclaredField("providers");
            f.setAccessible(true);
            providers = (List<ProviderInfo>) f.get(data);
        } catch (Exception e) {
            throw new RuntimeException("Failed to get providers from thread: " + thread);
        }

        sActivityThread = thread;
        sProviders = providers;
        sHostInstrumentation = base;
        sBundleInstrumentation = wrapper;
    }
```
**(1) 获得当前应用进程的 ActivityThread 实例**

首先，我们通过反射获得当前应用进程的`ActivityThread`实例
```
thread = ReflectAccelerator.getActivityThread(app)
```
具体的逻辑为：
```
    public static Object getActivityThread(Context context) {
        try {
            //1.首先尝试通过 ActivityThread 内部的静态变量获取。
            Class activityThread = Class.forName("android.app.ActivityThread");
            // ActivityThread.currentActivityThread()
            Method m = activityThread.getMethod("currentActivityThread", new Class[0]);
            m.setAccessible(true);
            Object thread = m.invoke(null, new Object[0]);
            if (thread != null) return thread;

            //2.静态变量获取失败，那么再通过 Application 的 mLoadedApk 中的 mActivityThread 获取。
            Field mLoadedApk = context.getClass().getField("mLoadedApk");
            mLoadedApk.setAccessible(true);
            Object apk = mLoadedApk.get(context);
            Field mActivityThreadField = apk.getClass().getDeclaredField("mActivityThread");
            mActivityThreadField.setAccessible(true);
            return mActivityThreadField.get(apk);
        } catch (Throwable ignore) {
            throw new RuntimeException("Failed to get mActivityThread from context: " + context);
        }
    }
```
这里面的逻辑为：
- 通过`ActivityThread`中的静态方法`currentActivityThread`来获取：
```
    public static ActivityThread currentActivityThread() {
        return sCurrentActivityThread;
    }
```
`sCurrentActivityThread`是在`ActivityThread#attach(boolean)`方法中被赋值的，而`attach`方法则是在入口函数`main`中调用的：
```
   public static void main(String[] args) {
        //创建应用进程的 ActivityThread 实例。
        ActivityThread thread = new ActivityThread();
        thread.attach(false);
    }
```
- 如果上面的方法获取失败，那么我们再尝试获取`Application`中的`LoadedApk#mActivityThread`。

**(2) 替换 ActivityThread 中的 mInstrumentation**

通过`(1)`拿到`ActivityThread`实例之后，接下来就是替换其中`mInstrumentation`成员变量为`Small`自己的实现类`ApkBundleLauncher.InstrumentationWrapper`，并将原始的`mInstrumentation`传入作为其成员变量。

正如 [Framework 源码解析知识梳理(1) - 应用进程与 AMS 的通信实现](http://www.jianshu.com/p/f26a76e19a2d) 中所介绍的，当我们调用`startActivity`之后，那么会调用到它内部的`mInstrumentation`的`execStartActivity`方法，经过替换之后，就会调用`ApkBundleLauncher.InstrumentationWrapper`的对应方法，下面截图中的`mBase`就是原始的`mInstrumentation`：
![](http://upload-images.jianshu.io/upload_images/1949836-2395ff82e5ca9d07.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
`ReflectAccelerator`又通过反射调用了`mBase`的对应方法：
![](http://upload-images.jianshu.io/upload_images/1949836-43b6a1008bb6f42a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
由此可见，`hook`的目的就在于替代者的方法被调用，到调用原始对象的对应方法之间所进行的操作，也就是下面红色框中的这两句：
![](http://upload-images.jianshu.io/upload_images/1949836-fc4c515dfe67e63d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

首先看一下`wrap(Intent intent)`方法，它的作用为：当我们在应用侧启动一个插件`Activity`时，需要将它替换成为`AndroidManifest.xml`预先注册好的占坑`Activity`。
```
        private void wrapIntent(Intent intent) {
            ComponentName component = intent.getComponent();
            String realClazz;
            //判断是否显示地设置了目标组件的类名。
            if (component == null) {
                //如果没有显示设置 Component，那么通过 resolveActivity 来解析出目标组件。
                component = intent.resolveActivity(Small.getContext().getPackageManager());
                if (component != null) {
                    return;
                }

                //获得目标组件全路径名。
                realClazz = resolveActivity(intent);
                if (realClazz == null) {
                    return;
                }
            } else {
                //如果设置了类名，那么直接取出。
                realClazz = component.getClassName();
                if (realClazz.startsWith(STUB_ACTIVITY_PREFIX)) {
                    realClazz = unwrapIntent(intent);
                }
            }
            if (sLoadedActivities == null) return;
            //根据类名，确定它是否是插件当中的 Activity 
            ActivityInfo ai = sLoadedActivities.get(realClazz);
            if (ai == null) return;
            //将真实的 Activity 保存在 Category 中，并加上 > 标识符。
            intent.addCategory(REDIRECT_FLAG + realClazz);
            //选取占坑的 Activity 
            String stubClazz = dequeueStubActivity(ai, realClazz);
            //重新设置 intent，用占坑的 Activity 来替代目标 Activity 
            intent.setComponent(new ComponentName(Small.getContext(), stubClazz));
        }
```
其中，`dequeueStubActivity`就是取出占坑的`Activity`，它是预先在`AndroidManifest.xml`中注册的一些占坑`Activity`，同时，我们也会把真实的目标`Activity`放在`Category`字段当中。

接下来，再看一下`ensureInjectMessageHandler(Object thread)`函数，代码的逻辑很简单，就是替换`AcitivtyThread`的`mH`中的`mCallback`变量为`sActivityThreadHandlerCallback`，它的类型为`ActivityThreadHandlerCallback`，是我们自定的一个内部类。
```
    private static void ensureInjectMessageHandler(Object thread) {
        try {
            Field f = thread.getClass().getDeclaredField("mH");
            f.setAccessible(true);
            Handler ah = (Handler) f.get(thread);
            f = Handler.class.getDeclaredField("mCallback");
            f.setAccessible(true);

            boolean needsInject = false;
            if (sActivityThreadHandlerCallback == null) {
                needsInject = true;
            } else {
                Object callback = f.get(ah);
                if (callback != sActivityThreadHandlerCallback) {
                    needsInject = true;
                }
            }

            if (needsInject) {
                // Inject message handler
                sActivityThreadHandlerCallback = new ActivityThreadHandlerCallback();
                f.set(ah, sActivityThreadHandlerCallback);
            }
        } catch (Exception e) {
            throw new RuntimeException("Failed to replace message handler for thread: " + thread);
        }
    }
```

在  [Framework 源码解析知识梳理(1) - 应用进程与 AMS 的通信实现](http://www.jianshu.com/p/f26a76e19a2d) 我们分析过，`Activity`的生命周期是由`AMS`使用运行在系统进程的代理对象`ApplicationThreadProxy`，通过`Binder`通信发送消息，在应用进程中的`ActivityThread#ApplicationThread`在`onTransact()`收到消息后，再通过`mH`（一个自定的`Handler`，类型为`H`），发送消息到主线程，`H`在`handleMessage`中处理消息，回调`Activity`对应的生命周期方法。

而`ensureInjectMessageHandler`所做就是让`H`的`handleMessage`方法被调用之前，进行一些额外的操作，例如在占坑的`Activity`启动完成之后，将它在应用测的记录替换成为`Activity`，而这一过程是通过替换`Handler`当中的`mCallback`对象，因为在调用`handleMessage`之前，会先去调用`mCallback`的`handleMessage`，并且在其不返回`true`的情况下，会继续调用`Handler`本身的`handleMessage`方法：
![](http://upload-images.jianshu.io/upload_images/1949836-ea85c3879b3c1a43.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

对于`Small`来说，它会对以下四种类型的消息进行拦截：
![](http://upload-images.jianshu.io/upload_images/1949836-09be99bfa6ee35c2.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

我们以`redirectActivity`为例，看一下将占坑的`Activity`重新替换为真实的`Activity`的过程。
```
        private void redirectActivity(Message msg) {
            Object/*ActivityClientRecord*/ r = msg.obj;
            //通过反射获得启动该 Activity 的 intent。
            Intent intent = ReflectAccelerator.getIntent(r);
            //就是通过前面放在 Category 中的字段，来取得真实的 Activity 名字。
            String targetClass = unwrapIntent(intent);
            boolean hasSetUp = Small.hasSetUp();
            if (targetClass == null) {
                if (hasSetUp) return; // nothing to do
                if (intent.hasCategory(Intent.CATEGORY_LAUNCHER)) {
                    return;
                }
                Small.setUpOnDemand();
                return;
            }
            if (!hasSetUp) {
                //确保初始化了。
                Small.setUp();
            }
            //重新替换为真实的 Activity
            ActivityInfo targetInfo = sLoadedActivities.get(targetClass);
            ReflectAccelerator.setActivityInfo(r, targetInfo);
        }
```
由于`handleMessage`的返回值为`false`，按照前面的分析，`mH`的`handleMessage`方法也会得到执行。

以上就是整个`Hook`的过程，简单的总结下来就是在应用进程与`AMS`进程的通信过程的某个节点，通过替换类的方式，插入一些逻辑，以绕过系统的检查：
- 从应用进程到`AMS`所在进程的通信，是通过替换应用进程中的`ActivityThread`的`mInstrumentation`为自定义的`ApkBundleLauncher.InstrumentationWrapper`，在其中将`Intent`当中真实的`Activity`替换成为占坑的`Activity`，然后再调用原始的`mInstrumentation`通知`AMS`。
- 从`AMS`所在进行到应用进程的通信，是通过替换应用进程中的`H`中的`mCallback`，在其中将占坑`Activity`替换成为真实的`Activity`，再执行原本的操作。

**(3) ActivityThread 内部的 mBoundApplication 变量**

这一步没有进行`Hook`操作，而是先获得`ActivityThread`内部的`mBoundApplication`实例，然后获得该实例内部的`providers`变量，它的类型为`List<ProviderInfo>`。

**(4) 备份**

最后一步，就是备份一些关键变量，用于之后的操作：
```
        //ActivityThread 实例
        sActivityThread = thread;
        //List<ProviderInfo> 实例
        sProviders = providers;
        //原始的 Instrumentation 实例
        sHostInstrumentation = base;
        //执行 Hook 操作的 Instrumentation 实例
        sBundleInstrumentation = wrapper;
```

# 三、实例分析
以上就是源码分析部分，下面，我们通过一个启动插件`Activity`的过程，来验证一下前面的分析：
## 3.1 从应用进程到 AMS 进程
通过下面的方法启动一个插件`Activity`：
```
    public void startStubActivity(View view) {
        Small.openUri("upgrade", this);
    }
```
按照前面的分析，此时应当会调用经过`Hook`之后的`Instrumentation`实例的`execStartActivity`方法，可以看到在`wrapIntent`方法调用之前，我们的目标`Activity`仍然是真实的`UpgradeActivity`：
![](http://upload-images.jianshu.io/upload_images/1949836-802c323f6119120d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
让断点继续往下走，经过`wrapIntent`之后，`Intent`的目标对象替换成为了占坑的`Activity`：
![](http://upload-images.jianshu.io/upload_images/1949836-620ccd508091189e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
## 3.2 从 AMS 进程到应用进程
而当`AMS`需要通知应用进程时，它第一次回调的是占坑的`Activity`，也就是如下所示：
![](http://upload-images.jianshu.io/upload_images/1949836-eb7dab3e00699460.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
通过反射，我们修改`ActivityClientRecord`中的内容，让其还原成为真实的`Activity`：
![](http://upload-images.jianshu.io/upload_images/1949836-d558b256a2d832bd.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

# 四、小结
以上就是`Small`预初始化所做的一些事情，也就是其`Hook`实现的原理，很多第三方的插件化都是基于该原理来实现启动不在`AndroidManifest.xml`中注册的组件的，开始的时候，理解起来可能会有点困难，关键是要弄清楚应用程序和`AMS`进程的交互原理，欢迎阅读 [Framework 源码解析知识梳理(1) - 应用进程与 AMS 的通信实现](http://www.jianshu.com/p/f26a76e19a2d) 。
