---
title: Java&Android 基础知识梳理(1) - 注解
date: 2017-02-20 18:17
categories : Java&Android 基础知识梳理
---
# 一、什么是注解
注解可以向编译器、虚拟机等解释说明一些事情。举一个最常见的例子，当我们在子类当中**覆写**父类的`aMethod`方法时，在子类的`aMethod`上会用`@Override`来修饰它，反之，如果我们给子类的`bMethod`用`@Override`注解修饰，但是在它的父类当中并没有这个`bMethod`，那么就会报错。这个`@Override`就是一种注解，它的作用是告诉编译器它所注解的方法是重写父类的方法，这样编译器就会去**检查父类是否存在这个方法**。
注解是用来描述`Java`代码的，它既能被编译器解析，也能在运行时被解析。


# 二、元注解
元注解是**描述注解的注解**，也是我们编写自定义注解的基础，比如以下代码中我们使用`@Target`元注解来说明`MethodInfo`这个注解只能应用于对方法进行注解：
```
@Target(ElementType.METHOD)
public @interface MethodInfo {
    //....
}
```
下面我们来介绍4种元注解，我们可以发现这四个元注解的定义又借助到了其它的元注解：
## 2.1 `Documented`
当一个注解类型被`@Documented`元注解所描述时，那么无论在哪里使用这个注解，都会被`Javadoc`工具文档化，我们来看以下它的定义：
```
@Documented 
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.ANNOTATION_TYPE)
public @interface Documented {
    //....
}
```
- 定义注解时使用`@interface`关键字：
- `@Document`表示它本身也会被文档化；
- `Retention`表示`@Documented`这个注解能保留到运行时；
- `@ElementType.ANNOTATION_TYPE`表示`@Documented`这个注解只能够被用来描述注解类型。

## 2.2 `Inherited`
表明被修饰的注解类型是自动继承的，若一个注解被`Inherited`元注解修饰，则当用户在一个类声明中查询该注解类型时，若发现这个类声明不包含这个注解类型，则会自动在这个类的父类中查询相应的注解类型。
我们需要注意的是，用`inherited`修饰的注解，它的这种自动继承功能，只能对**类**生效，对方法是不生效的。也就是说，如果父类有一个`aMethod`方法，并且该方法被注解`a`修饰，那么无论这个注解`a`是否被`Inherited`修饰，**只要我们在子类中覆写了`aMethod`**，子类的`aMethod`都不会继承父类`aMethod`的注解，反之，如果我们没有在子类中覆写`aMethod`，那么通过子类我们依然可以获得注解`a`。
```
@Documented
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.ANNOTATION_TYPE)
public @interface Inherited {
    //....
}
```

## 2.3 `Retention`
这个注解表示一个注解类型会被保留到什么时候，它的原型为：
```
@Documented
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.ANNOTATION_TYPE)
public @interface Retention { 
    RetentionPolicy value();
}
```
其中，`RetentionPolicy.xxx`的取值有：
- `SOURCE`：表示在编译时这个注解会被移除，不会包含在编译后产生的`class`文件中。
- `CLASS`：表示这个注解会被包含在`class`文件中，但在运行时会被移除。
- `RUNTIME`：表示这个注解会被保留到运行时，我们可以在运行时通过反射解析这个注解。

## 2.4 `Target`
这个注解说明了被修饰的注解的应用范围，其用法为：
```
@Documented
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.ANNOTATION_TYPE)
public @interface Target { 
    ElementType[] value();
}
```
`ElementType`是一个枚举类型，它包括：
- `TYPE`：类、接口、注解类型或枚举类型。
- `PACKAGE`：注解包。
- `PARAMETER`：注解参数。
- `ANNOTATION_TYPE`：注解 注解类型。
- `METHOD`：方法。
- `FIELD`：属性（包括枚举常量）
- `CONSTRUCTOR`：构造器。
- `LOCAL_VARIABLE`：局部变量。

# 三、常见注解
# 3.1 `@Override`
```
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.SOURCE)
public @interface Override {}
```
告诉编译器被修饰的方法是重写的父类中的相同签名的方法，编译器会对此做出检查，若发现父类中不存在这个方法或是存在的方法签名不同，则会报错。
# 3.2 `@Deprecated`
```
@Documented
@Retention(RetentionPolicy.RUNTIME)
@Target(value={CONSTRUCTOR, FIELD, LOCAL_VARIABLE, METHOD, PACKAGE, PARAMETER, TYPE})
public @interface Deprecated {}
```
不建议使用这些被修饰的程序元素。
# 3.3 `@SuppressWarnings`
```
@Target({TYPE, FIELD, METHOD, PARAMETER, CONSTRUCTOR, LOCAL_VARIABLE})
@Retention(RetentionPolicy.SOURCE)
public @interface SuppressWarnings { 
  String[] value();
}
```
告诉编译器忽略指定的警告信息。

# 四、自定义注解
在自定义注解前，有一些基础知识：
- 注解类型是用`@interface`关键字定义的。
- 所有的方法均没有方法体，且只允许`public`和`abstract`这两种修饰符号，默认为`public`。
- 注解方法只能返回：原始数据类型，`String`，`Class`，枚举类型，注解，它们的一维数组。

下面是一个例子：
```
@Documented
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
@Inherited
public @interface MethodInfo { 
  String author() default "absfree"; 
  String date(); 
  int version() default 1;
}
```
# 五、注解的解析
## 5.1 编译时解析
`ButterKnife`是**解析编译时注解**很经典的例子，因为在`Activity/ViewGroup/Fragment`中，我们有很多的`findViewById/setOnClickListener`，这些代码具有一个特点，就是重复性很高，它们仅仅是`id`和返回值不同。
这时候，我们就可以给需要执行`findViewById`的`View`加上注解，然后在编译时根据规则生成特定的一些类，这些类中的方法会执行上面那些重复性的操作。

下面是网上一个大神写的模仿`ButterKnife`的例子，我们来看一下编译时解析是如果运用的。
- [项目地址](https://github.com/brucezz/ViewFinder)
- [项目作者博客](http://blog.csdn.net/hb707934728/article/details/52213086)

整个项目的结构如下：
![](http://upload-images.jianshu.io/upload_images/1949836-bc311a76c83908ec.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

- `app`：示例模块，它和其它3个模块的关系为：
![](http://upload-images.jianshu.io/upload_images/1949836-619a8ae8d701b09f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
- `viewfinder`：`android-library`，它声明了`API`的接口。
- `viewfinder-annotation`：`Java-library`，包含了需要使用到的注解。
- `viewfinder-compiler`：`Java-library`，包含了注解处理器。

### 5.1.1 创建注解
新建一个`viewfinder-annotation`的`java-library`，它包含了所需要用到的注解，注意到这个注解是保留到编译时：
```
@Retention(RetentionPolicy.CLASS)
@Target(ElementType.FIELD)
public @interface BindView {
    int id();
}
```
### 5.1.2 声明`API`接口
新建一个`viewfinder`的`android-library`，用来提供给外部调用的接口。
首先新建一个`Provider`接口和它的两个实现类：
```
public interface Provider {
    Context getContext(Object source);
    View findView(Object source, int id);
}

public class ActivityProvider implements Provider{

    @Override
    public Context getContext(Object source) {
        return ((Activity) source);
    }

    @Override
    public View findView(Object source, int id) {
        return ((Activity) source).findViewById(id);
    }
}

public class ViewProvider implements Provider {

    @Override
    public Context getContext(Object source) {
        return ((View) source).getContext();
    }

    @Override
    public View findView(Object source, int id) {
        return ((View) source).findViewById(id);
    }
}
```
定义接口`Finder`，后面我们会根据被`@BindView`注解所修饰的变量所在类（`host`）来生成不同的`Finder`实现类，而这个判断的过程并不需要使用者去关心，而是由框架的实现者在编译器时就处理好的了。
```
public interface Finder<T> {

    /**
     * @param host 持有注解的类
     * @param source 调用方法的所在的类
     * @param provider 执行方法的类
     */
    void inject(T host, Object source, Provider provider);

}
```
`ViewFinder`是`ViewFinder`框架的使用者唯一需要关心的类，当在`Activity/Fragment/View`中调用了`inject`方法时，会经过一下几个过程：
- 获得调用`inject`方法所在类的类名`xxx`，也就是注解类。
- 获得属于该类的`xxx$$Finder`，调用`xxx$$Finder`的`inject`方法。

```
public class ViewFinder {

    private static final ActivityProvider PROVIDER_ACTIVITY = new ActivityProvider();
    private static final ViewProvider PROVIDER_VIEW = new ViewProvider();

    private static final Map<String, Finder> FINDER_MAP = new HashMap<>(); //由于使用了反射,因此缓存起来.

    public static void inject(Activity activity) {
        inject(activity, activity, PROVIDER_ACTIVITY);
    }

    public static void inject(View view) {
        inject(view, view);
    }

    public static void inject(Object host, View view) {
        inject(host, view, PROVIDER_VIEW);
    }

    public static void inject(Object host, Object source, Provider provider) {
        String className = host.getClass().getName(); //获得注解所在类的类名.
        try {
            Finder finder = FINDER_MAP.get(className); //每个Host类,都会有一个和它关联的Host$$Finder类,它实现了Finder接口.
            if (finder == null) {
                Class<?> finderClass = Class.forName(className + "$$Finder");
                finder = (Finder) finderClass.newInstance();
                FINDER_MAP.put(className, finder);
            }
            //执行这个关联类的inject方法.
            finder.inject(host, source, provider);
        } catch (Exception e) {
            throw new RuntimeException("Unable to inject for " + className, e);
        }
    }
}
```
那么这上面所有的`xxx$$Finder`类，到底是什么时候产生的呢，它们的`inject`方法里面又做了什么呢，这就需要涉及到下面注解处理器的创建。
### 5.1.3 创建注解处理器
创建`viewfinder-compiler`（`java-library`），在`build.gradle`中导入下面需要的类：
```
apply plugin: 'java'

dependencies {
    compile fileTree(dir: 'libs', include: ['*.jar'])
    compile project(':viewfinder-annotation')
    compile 'com.squareup:javapoet:1.7.0'
    compile 'com.google.auto.service:auto-service:1.0-rc2'
}
targetCompatibility = '1.7'
sourceCompatibility = '1.7'
```
`TypeUtil`定义了需要用到的类的包名和类名：
```
public class TypeUtil {
    public static final ClassName ANDROID_VIEW = ClassName.get("android.view", "View");
    public static final ClassName ANDROID_ON_LONGCLICK_LISTENER = ClassName.get("android.view", "View", "OnLongClickListener");
    public static final ClassName FINDER = ClassName.get("com.example.lizejun.viewfinder", "Finder");
    public static final ClassName PROVIDER = ClassName.get("com.example.lizejun.viewfinder.provider", "Provider");
}
```
每个`BindViewField`和注解类中使用了`@BindView`修饰的`View`是一一对应的关系。
```
public class BindViewField {

    private VariableElement mFieldElement;
    private int mResId;
    private String mInitValue;

    public BindViewField(Element element) throws IllegalArgumentException {
        if (element.getKind() != ElementKind.FIELD) { //判断被注解修饰的是否是变量.
            throw new IllegalArgumentException(String.format("Only fields can be annotated with @%s", BindView.class.getSimpleName()));
        }
        mFieldElement = (VariableElement) element; //获得被修饰变量.
        BindView bindView = mFieldElement.getAnnotation(BindView.class); //获得被修饰变量的注解.
        mResId = bindView.id(); //获得注解的值.
    }

    /**
     * @return 被修饰变量的名字.
     */
    public Name getFieldName() {
        return mFieldElement.getSimpleName();
    }

    /**
     * @return 被修饰变量的注解的值,也就是它的id.
     */
    public int getResId() {
        return mResId;
    }

    /**
     * @return 被修饰变量的注解的值.
     */
    public String getInitValue() {
        return mInitValue;
    }

    /**
     * @return 被修饰变量的类型.
     */
    public TypeMirror getFieldType() {
        return mFieldElement.asType();
    }
}
```
`AnnotatedClass`封装了添加被修饰注解`element`，通过`element`列表生成`JavaFile`这两个过程，`AnnotatedClass`和注解类是一一对应的关系：
```
public class AnnotatedClass {
    public TypeElement mClassElement;
    public List<BindViewField> mFields;
    public Elements mElementUtils;

    public AnnotatedClass(TypeElement classElement, Elements elementUtils) {
        this.mClassElement = classElement;
        mFields = new ArrayList<>();
        this.mElementUtils = elementUtils;
    }

    public String getFullClassName() {
        return mClassElement.getQualifiedName().toString();
    }

    public void addField(BindViewField bindViewField) {
        mFields.add(bindViewField);
    }

    public JavaFile generateFinder() {
        //生成inject方法的参数.
        MethodSpec.Builder methodBuilder = MethodSpec
                .methodBuilder("inject") //方法名.
                .addModifiers(Modifier.PUBLIC) //访问权限.
                .addAnnotation(Override.class) //注解.
                .addParameter(TypeName.get(mClassElement.asType()), "host", Modifier.FINAL) //参数.
                .addParameter(TypeName.OBJECT, "source")
                .addParameter(TypeUtil.PROVIDER, "provider");
        //在inject方法中,生成重复的findViewById(R.id.xxx)的语句.
        for (BindViewField field : mFields) {
            methodBuilder.addStatement(
                    "host.$N = ($T)(provider.findView(source, $L))",
                    field.getFieldName(),
                    ClassName.get(field.getFieldType()),
                    field.getResId());
        }
        //生成Host$$Finder类.
        TypeSpec finderClass = TypeSpec
                .classBuilder(mClassElement.getSimpleName() + "$$Finder")
                .addModifiers(Modifier.PUBLIC)
                .addSuperinterface(ParameterizedTypeName.get(TypeUtil.FINDER, TypeName.get(mClassElement.asType())))
                .addMethod(methodBuilder.build())
                .build();
        //获得包名.
        String packageName = mElementUtils.getPackageOf(mClassElement).getQualifiedName().toString();
        return JavaFile.builder(packageName, finderClass).build();

    }
}
```
在做完前面所有的准备工作之后，后面的事情就很清楚了：
- 编译时，系统会调用所有`AbstractProcessor`子类的`process`方法，也就是调用我们的`ViewFinderProcess`的类。
- 在`ViewFinderProcess`中，我们获得工程下所有被`@BindView`注解所修饰的`View`。
- 遍历这些被`@BindView`修饰的`View`变量，获得它们被声明时所在的类，首先判断是否已经为所在的类生成了对应的`AnnotatedClass`，如果没有，那么生成一个，并将`View`封装成`BindViewField`添加进入`AnnotatedClass`的列表，反之添加即可，所有的`AnnotatedClass`被保存在一个`map`当中。
- 当遍历完所有被注解修饰的`View`后，开始遍历之前生成的`AnnotatedClass`，每个`AnnotatedClass`会生成一个对应的`$$Finder`类。
- 如果我们在`n`个类中使用了`@BindView`来修饰里面的`View`，那么我们最终会得到`n`个`$$Finder`类，并且无论我们最终有没有在这`n`个类中调用`ViewFinder.inject`方法，都会生成这`n`个类；而如果我们调用了`ViewFinder.inject`，那么最终就会通过反射来实例化它对应的`$$Finder`类，通过调用`inject`方法来给被它里面被`@BindView`所修饰的`View`执行`findViewById`操作。

```
@AutoService(Processor.class)
public class ViewFinderProcess extends AbstractProcessor{

    private Filer mFiler;
    private Elements mElementUtils;
    private Messager mMessager;

    private Map<String, AnnotatedClass> mAnnotatedClassMap = new HashMap<>();

    @Override
    public synchronized void init(ProcessingEnvironment processingEnv) {
        super.init(processingEnv);
        mFiler = processingEnv.getFiler();
        mElementUtils = processingEnv.getElementUtils();
        mMessager = processingEnv.getMessager();
    }

    @Override
    public Set<String> getSupportedAnnotationTypes() {
        Set<String> types = new LinkedHashSet<>();
        types.add(BindView.class.getCanonicalName());
        return types;
    }

    @Override
    public SourceVersion getSupportedSourceVersion() {
        return SourceVersion.latestSupported();
    }

    @Override
    public boolean process(Set<? extends TypeElement> annotations, RoundEnvironment roundEnv) {
        mAnnotatedClassMap.clear();
        try {
            processBindView(roundEnv);
        } catch (IllegalArgumentException e) {
            return true;
        }
        for (AnnotatedClass annotatedClass : mAnnotatedClassMap.values()) { //遍历所有要生成$$Finder的类.
            try {
                annotatedClass.generateFinder().writeTo(mFiler); //一次性生成.
            } catch (IOException e) {
                return true;
            }
        }
        return true;
    }

    private void processBindView(RoundEnvironment roundEnv) throws IllegalArgumentException {
        for (Element element : roundEnv.getElementsAnnotatedWith(BindView.class)) {
            AnnotatedClass annotatedClass = getAnnotatedClass(element);
            BindViewField field = new BindViewField(element);
            annotatedClass.addField(field);
        }
    }

    private AnnotatedClass getAnnotatedClass(Element element) {
        TypeElement classElement = (TypeElement) element.getEnclosingElement();
        String fullClassName = classElement.getQualifiedName().toString();
        AnnotatedClass annotatedClass = mAnnotatedClassMap.get(fullClassName);
        if (annotatedClass == null) {
            annotatedClass = new AnnotatedClass(classElement, mElementUtils);
            mAnnotatedClassMap.put(fullClassName, annotatedClass);
        }
        return annotatedClass;
    }
}
```

## 5.2 运行时解析
首先我们需要定义注解类型，`RuntimeMethodInfo`：
```
@Documented
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
@Inherited
public @interface RuntimeMethodInfo {
    String author() default  "tony";
    String data();
    int version() default 1;
}
```
之后，我们再定义一个类`RuntimeMethodInfoTest`，它其中的`testRuntimeMethodInfo`方法使用了这个注解，并给它其中的两个成员变量传入了值：
```
public class RuntimeMethodInfoTest {
    @RuntimeMethodInfo(data = "1111", version = 2)
    public void testRuntimeMethodInfo() {}
}
```
最后，在程序运行时，我们动态获取注解中传入的信息：
```
private void getMethodInfoAnnotation() {
        Class cls = RuntimeMethodInfoTest.class;
        for (Method method : cls.getMethods()) {
            RuntimeMethodInfo runtimeMethodInfo = method.getAnnotation(RuntimeMethodInfo.class);
            if (runtimeMethodInfo != null) {
                System.out.println("RuntimeMethodInfo author=" + runtimeMethodInfo.author());
                System.out.println("RuntimeMethodInfo data=" + runtimeMethodInfo.data());
                System.out.println("RuntimeMethodInfo version=" + runtimeMethodInfo.version());
            }
        }
}
```
最后得到打印出的结果为：

![Paste_Image.png](http://upload-images.jianshu.io/upload_images/1949836-c56ff7781a6f53a7.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

# 参考文档：
[`1.http://blog.csdn.net/lemon89/article/details/47836783`](`http://blog.csdn.net/lemon89/article/details/47836783`)
[`2.http://blog.csdn.net/hb707934728/article/details/52213086`](http://blog.csdn.net/hb707934728/article/details/52213086)
[`3.https://github.com/brucezz/ViewFinder`](https://github.com/brucezz/ViewFinder)
[`4.http://www.cnblogs.com/peida/archive/2013/04/24/3036689.html`](http://www.cnblogs.com/peida/archive/2013/04/24/3036689.html)
