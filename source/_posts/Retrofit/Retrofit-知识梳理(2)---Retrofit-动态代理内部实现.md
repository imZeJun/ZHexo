---
title: Retrofit 知识梳理(2) - Retrofit 动态代理内部实现
date: 2017-06-01 21:13
categories : Retrofit 知识梳理
---
# 一、前言
在 [Retrofit 知识梳理(1) - 流程分析](http://www.jianshu.com/p/721fb07a1cff) 中，我们对于`Retrofit`的流程进行了简单的分析，大家谈到`Retrofit`的时候，往往也会提到动态代理，今天这篇文章，我们就来一起研究一下这一过程的内部实现。

# 二、内部实现
涉及到动态代理的部分为下面这两句：
```
//1.返回代理对象
GitHubService service = retrofit.create(GitHubService.class);
//2.调用该代理对象的接口方法
Call<ResponseBody> call = service.listRepos("octocat");
```
## 2.1 生成代理对象
第一步的目的就是通过`Retrofit`的`create`方法，根据`GithubService`这个接口所声明的方法来得到一个代理对象：
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
可以看到，这里面最核心的方法就是：
```
Proxy.newProxyInstance(ClassLoader loader, Class<?>[] interfaces, InvocationHandler h)
```
以上三个形参的含义为：
- `ClassLoader`：表示由哪一个`ClassLoader`来加载这个代理类。
- `Class<?>[]`：一个`interface`对象的数组，表明将要给被代理的对象提供一组什么样的接口来访问，这个可能比较抽象。从上面的例子来说，我们传入的是`GitHubService.class`，而它是一个接口，其方法包含有`listRepos`，那么`Proxy.newProxyInstance`所生成的代理对象也会有这个接口`listRepos`，所以我们才能在第二步当中调用它的方法。
- `InvocationHandler`：当调用代理对象的接口方法时，最终会回调到这个`InvocationHandler`对象的`invoke`方法中。

在对形参进行了简要的介绍之后，我们再来看一下其内部的实现：
```
    public static Object newProxyInstance(ClassLoader loader, Class<?>[] interfaces, InvocationHandler h) throws IllegalArgumentException {
        if (h == null) {
            throw new NullPointerException();
        }
        //1.根据前两个参数创建一个代理类.
        Class<?> cl = getProxyClass0(loader, interfaces);
        try {
            //2.实例化该代理类，并将第三个参数作为成员变量.
            final Constructor<?> cons = cl.getConstructor(constructorParams);
            return newInstance(cons, h);
        } catch (NoSuchMethodException e) {
            throw new InternalError(e.toString());
        }
    }
```
可以看到，生成代理对象又可以细分为两步：
- 根据类加载器和声明的接口，首先创建一个代理类，该代理类继承于`Proxy`，并且它实现了`interfaces`数组中所声明的接口
- 实例化第一步中创建的代理类，并将第三个参数所对象的`InvocationHandler`传入作为其成员变量`h`。

### 2.1.1 创建代理类
创建代理类的代码比较多，也就是对应于`getProxyClass0`方法：
```
private static Class<?> getProxyClass0(ClassLoader loader, Class<?>... interfaces) {
        if (interfaces.length > 65535) {
            throw new IllegalArgumentException("interface limit exceeded");
        }
        //最终要创建的代理类
        Class<?> proxyClass = null;
        String[] interfaceNames = new String[interfaces.length];
        Set<Class<?>> interfaceSet = new HashSet<>();
        //1.遍历传入的接口数组
        for (int i = 0; i < interfaces.length; i++) {
            String interfaceName = interfaces[i].getName();
            Class<?> interfaceClass = null;
            try {
                interfaceClass = Class.forName(interfaceName, false, loader);
            } catch (ClassNotFoundException e) {
            }
            if (interfaceClass != interfaces[i]) {
                throw new IllegalArgumentException(
                    interfaces[i] + " is not visible from class loader");
            }
            //必须是接口
            if (!interfaceClass.isInterface()) {
                throw new IllegalArgumentException(
                    interfaceClass.getName() + " is not an interface");
            }
            //不允许重复
            if (interfaceSet.contains(interfaceClass)) {
                throw new IllegalArgumentException(
                    "repeated interface: " + interfaceClass.getName());
            }
            interfaceSet.add(interfaceClass);

            interfaceNames[i] = interfaceName;
        }
        //2.如果之前已经生成过，那么就不需要再去生成.
        List<String> key = Arrays.asList(interfaceNames);
        Map<List<String>, Object> cache;
        synchronized (loaderToCache) {
            cache = loaderToCache.get(loader);
            if (cache == null) {
                cache = new HashMap<>();
                loaderToCache.put(loader, cache);
            }
        }
        synchronized (cache) {
            do {
                Object value = cache.get(key);
                if (value instanceof Reference) {
                    proxyClass = (Class<?>) ((Reference) value).get();
                }
                if (proxyClass != null) {
                    return proxyClass;
                } else if (value == pendingGenerationMarker) {
                    try {
                        cache.wait();
                    } catch (InterruptedException e) {
                    }
                    continue;
                } else {
                    cache.put(key, pendingGenerationMarker);
                    break;
                }
            } while (true);
        }
        //3.真正的产生操作.
        try {
            String proxyPkg = null;
            for (int i = 0; i < interfaces.length; i++) {
                int flags = interfaces[i].getModifiers();
                if (!Modifier.isPublic(flags)) {
                    String name = interfaces[i].getName();
                    int n = name.lastIndexOf('.');
                    String pkg = ((n == -1) ? "" : name.substring(0, n + 1));
                    if (proxyPkg == null) {
                        proxyPkg = pkg;
                    } else if (!pkg.equals(proxyPkg)) {
                        throw new IllegalArgumentException(
                            "non-public interfaces from different packages");
                    }
                }
            }

            if (proxyPkg == null) {
                proxyPkg = "";
            }

            {
                List<Method> methods = getMethods(interfaces);
                Collections.sort(methods, ORDER_BY_SIGNATURE_AND_SUBTYPE);
                validateReturnTypes(methods);
                List<Class<?>[]> exceptions = deduplicateAndGetExceptions(methods);

                Method[] methodsArray = methods.toArray(new Method[methods.size()]);
                Class<?>[][] exceptionsArray = exceptions.toArray(new Class<?>[exceptions.size()][]);

                final long num;
                synchronized (nextUniqueNumberLock) {
                    num = nextUniqueNumber++;
                }
                String proxyName = proxyPkg + proxyClassNamePrefix + num;

                proxyClass = generateProxy(proxyName, interfaces, loader, methodsArray,
                        exceptionsArray);
            }

            proxyClasses.put(proxyClass, null);

        } finally {
            synchronized (cache) {
                if (proxyClass != null) {
                    cache.put(key, new WeakReference<Class<?>>(proxyClass));
                } else {
                    cache.remove(key);
                }
                cache.notifyAll();
            }
        }
        return proxyClass;
    }
```
## 2.1.2 实例化代理类
当创建完代理类之后，接下来就是进行实例化，这里实例化调用的是参数为`InvocationHandler`的构造函数：
```
protected Proxy(InvocationHandler h) {
     this.h = h;
}
```
也就是说，最后通过`Proxy.newProxyInstance`返回的代理类实例，其内部的`InvocationHandler`对象，就是通过第三个参数所传入的对象，而当我们调用代理类所声明的方法时，最终会调用到该`InvocationHandler`的`invoke`方法，通过`invoke`方法的参数，我们又可以区分具体调用的方法。
## 2.2 调用代理对象的接口方法
通过第一步，我们就可以得到一个代理对象，而当我们调用这个代理对象的接口方法之后，该代理对象又会回调它的基类`Proxy`中的这个方法：
```
private static Object invoke(Proxy proxy, Method method, Object[] args) throws Throwable {
    InvocationHandler h = proxy.h;
    return h.invoke(proxy, method, args);
}
```
`invoke`的三个参数解释为：
- `Proxy`：代理对象实例，也就是上面在`2.1`中所返回的实例对象。
- `Method`：代理对象所被调用的对应接口方法。
- `Object[]`：接口方法所传入的参数。

# 三、示例
在第二节当中，我们对于源码进行了简单的走读，下面，我们以一个具体的示例来强化一下认识：
## 3.1 实例代码
** (1) 接口类**

对应于我们在上面所定义的`GitHubService.java`
```
public interface Subject {
    public CallObject getCallObject(String args);
}
```
**(2) 请求类**

对应于最终发起网络请求的类：
```
public class CallObject {

    private String args;

    public CallObject(String args) {
        this.args = args;
    }

    public void call() {
        Log.d("simulate", "call args=" + args);
    }
}
```
**(3) 模拟代理过程**
```
public class Simulator {

    public static void simulate() {
        Class mClass = Subject.class;
        Subject proxySubject = (Subject) Proxy.newProxyInstance(
                mClass.getClassLoader(),
                new Class[]{ mClass },
                new InvocationHandler() {

                    @Override
                    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
                        String modifiedArgs = (String) args[0] + " after modified";
                        Log.d("simulate", "invoke");
                        return new CallObject(modifiedArgs);
                    }

                });
        Log.d("simulate", "createProxy finished");
        CallObject callObject = proxySubject.getCallObject("args");
        Log.d("simulate", "getCallObject finished");
        callObject.call();
    }
}
```
## 3.2 示例详解
按照我们之前的分析，在上面的模拟过程中的第三步，会返回一个代理类，我们来看一下这个代理类具体是什么：
![](http://upload-images.jianshu.io/upload_images/1949836-72e23e146d4581a2.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
通过断点，我们可以看到它不再是我们之前声明的接口，而是一个代理类`$Proxy`，它的内部有一个`InvocationHandler`的实现类`h`，也就是我们通过`newProxyInstance`传入的实例。

之后，我们调用该代理类的接口方法，那么上面的`h`对象的`invoke`方法的就被回调，我们可以从中获取到传入的参数、注解等信息，通过该信息，我们构造出一个`CallObject`对象。

最后，再通过这个`CallObject`发起请求：
![](http://upload-images.jianshu.io/upload_images/1949836-2faa0fa800e7fd85.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
# 四、总结
普通的代理模式，往往会在`InvocationHandler`的实现类中包含一个接口的实现类，当该`InvocationHandler`的`invoke`方法被回调时，再调用他所包含的接口实现类进行操作，并在之前和之后进行一些处理的操作。对于标准的动态代理，可以参考 [这篇文章](http://paddy-w.iteye.com/blog/841798)。

`Retrofit`用到的动态代理，并不能算是严格的代理模式。它只是利用了代理模式中`invoke`这一中转过程，来解析接口中的注解声明，然后通过这些注解声明来创建一个请求类，最终再通过该请求类来发起请求。

也就是说，`Retrofit`所关注的重点在于如何创建`invoke`方法所返回的实例，而普通的代理模式则在于控制接口实现类的访问。




