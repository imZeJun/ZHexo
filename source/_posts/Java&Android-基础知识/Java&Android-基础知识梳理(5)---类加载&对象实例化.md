---
title: Java&Android 基础知识梳理(5) - 类加载&对象实例化
date: 2017-03-20 21:40
categories : Java&Android 基础知识梳理
---
# 一、概述
虚拟机的**类加载机制定义**：把描述类的数据从`Class`文件（一串二进制的字节流）加载到内存，并对数据进行校验、转换解析和初始化，最终形成被虚拟机直接使用的`Java`类型。

在`Java`语言里，类型的**加载、连接和初始化**过程都是在程序运行期间完成的，`Java`里天生可以动态扩展的语言特性就是依赖运行期**动态加载和动态连接**这个特点实现的。

用户可以通过`Java`预定义的和自定义类加载器，让一个本地的应用程序可以在运行时从网络或其他地方加载一个二进制流作为程序代码的一部分。

# 二、类加载的时机
## 2.1 类加载包含那些阶段
类从被加载到虚拟机内存中开始，到卸载出内存，所经过的生命周期有：
- 1.加载
- 2.验证
- 3.准备
- 4.解析
- 5.初始化
- 6.使用
- 7.卸载

其中`2-4`统称为连接，上面的过程有几个需要注意的点：
- 加载、验证、准备、初始化、卸载这五个阶段按顺序按部就班地**开始**，在一个阶段执行的过程中有可能调用、激活另外一个阶段。
- 解析阶段有可能在初始化之后开始，这是为了支持`Java`语言的**运行时绑定**。

## 2.2 类加载触发的时机
**有且仅有**下面五种情况必须立即对类进行初始化：
- 第一种：遇到`new/getstatic/putstatic/invokestatic`这`4`条字节码指令时，如果类没有进行过初始化，则需要先触发其初始化，场景：
 - 使用`new`关键字实例化对象
 - 读取或设置一个类的静态字段（被`final`修饰，已在编译期把结果放入常量池的字段除外）
 - 调用一个类的静态方法
```
        //1.new关键字.
        LoadInvokeClass loadInvokeClass = new LoadInvokeClass();
        //2.访问静态变量
        int content = LoadInvokeClass.sContent;
        //3.调用静态方法.
        LoadInvokeClass.staticMethod();
```
- 第二种：使用`java.lang.reflect`包的方法对类进行反射调用的时候，如果类没有进行过初始化，则需要先触发其初始化。
```
        try {
            Class<?> mClass = Class.forName("com.example.lizejun.repojavalearn.load.LoadInvokeClass");
        } catch (Exception e) { e.printStackTrace(); }
```
- 第三种：当初始化一个类的时候，如果需要初始化其父类，但是发现父类没有初始化、那么需要先触发其父类的初始化。
```
        //其中LoadInvokeClass是LoadInvokeClassChild的父类.
        LoadInvokeClassChild classChild = new LoadInvokeClassChild();
```
- 第四种：当虚拟机启动时，用户需要指定一个要执行的主类（包含`main()`方法），虚拟机会先初始化这个主类。
- 第五种：使用`JDK 1.7`的动态语言支持时，如果一个`java.lang.invoke.MethodHandle`实例最后的解析结果`REF_getStatic/REF_putStatic/REF_invokeStatic`的句柄方法，并且这个方法句柄所对应的类没有进行过初始化，则需要先触发其初始化。

## 2.3 被动引用
在`2.2`中谈到的都是主动引用，除此之外，所有引用类的方法都称为被动引用，而被动引用不会触发**类的初始化**：
- 类初始化时，如果父类没有被初始化，那么会先初始化父类，这一过程将一直递归到`Object`为止，但是不会去初始化它所实现的接口，即当我们初始化`ClassChild`的时候，只会先初始化`ClassParent`，淡不会初始化`ClassInterface`。

```
public interface ClassInterface {}

public class ClassParent implements ClassInterface {
    static {
        System.out.println("load ClassParent");
    }
}

public class ClassChild extends ClassParent {
    static {
        System.out.println("load ClassChild");
    }
}
```
- 接口初始化时，不要求父接口全部初始化，只有真正用到了父接口的时候（如引用接口中定义的常量），那么才会初始化。
- 当访问某个类的静态域时，不会触发父类的初始化或者子类的初始化，即使静态域被子类或子接口或者它的实现类所引用，我们给`ClassChild`添加一个静态属性，访问这个静态属性不会初始化`ClassParent`。
```
public class ClassChild extends ClassParent {

    public static int sNumber;

    static {
        System.out.println("load ClassChild");
    }
}
```
- 如果一个静态变量是编译时常量，则对它的引用不会引起定义它的类的初始化，如下面访问`sNumber`，那么不会引起`ClassChild`的实例化。
```
public class ClassChild extends ClassParent {

    public static final int sNumber = 2;

    static {
        System.out.println("load ClassChild");
    }
}
```
- 通过数组定义来引用类，不会触发此类的初始化。
```
ClassChild[] children = new ClassChild[10];
```

# 三、类加载的过程
## 3.1 加载
在"加载"阶段，虚拟机需要完成以下三件事情：
- 通过一个类的全限定名来获取**定义此类的二进制字节流**。
- 将这个字节流所代表的静态存储结构转化为**方法区的运行时数据结构**。
- 在内存中生成一个代表这个类的`java.lang.Class`对象，作为**方法区这个类的各种数据的访问入口**。

## 3.2 验证
"验证"阶段的目的是为了**确保`Class`文件的字节流中包含的信息符合当前虚拟机的要求**，并且不会危害自身的安全，大致会完成下面四个阶段的校验动作：
- 文件格式验证
- 元数据验证
- 字节码验证
- 符号引用验证

## 3.3 准备
"准备"阶段是正式**为类变量（被`static`修饰，而不是实例变量）分配内存并设置类变量初始值的阶段**，这些变量所使用的内存都将在方法区中进行分配。
- 对于`static`并且非`final`的类变量，将被初始化为数据类型的零值。
- 对于`static`且`final`的类变量，在这个阶段就会被初始化为`ConstantValue`属性所指定的值。

## 3.4 解析
“解析”阶段是虚拟机将常量池的符号引用替换为直接引用的过程，包括：
- 类或接口的解析
- 字段解析
- 类方法解析
- 接口方法解析

## 3.5 初始化
根据程序员通过程序指定的主观计划去初始化类变量和其它资源，也就是执行类构造器`<clinit>()`方法的过程：
- `<clinit>`方法是由编译器自动收集类中的所有类变量的赋值动作和静态语句块中的语句合并而成，顺序是由语句在源文件中出现的顺序决定的。静态语句块只能访问到定义在它之前的变量，对于定义在它后面的变量只能赋值不能访问。

- `<clinit>()`方法与类的构造函数不同，它不需要显示地调用父类构造器，虚拟机会保证在子类的`<clinit>()`方法执行前，父类的`<clinit>()`方法已经执行完毕，因此在虚拟机中第一个杯知行的`<clinit>()`方法的类肯定是`java.lang.Object`。

- 父类的静态语句块要优先于子类的变量赋值操作。

- 如果一个类中没有静态语句块，也没有对类变量的赋值操作，那么编译器可以不为这个类生成`<clinit>()`方法。

- 接口不能接口中仅有变量初始化的赋值操作，但执行接口的`<clinit>()`方法不需要先执行父接口的`<clinit>()`方法，只有当父接口中定义的变量使用时，父接口才会初始化，另外，接口的实现类在初始化时也一样不会执行接口的`<clinit>()`方法。

- 虚拟机会保证一个类的`<clinit>()`方法在多线程环境中被正确地加锁、同步。

# 四、类加载器
## 4.1 概念
类加载器用来“通过一个类的全限定名来获取描述此类的二进制字节流”。
## 4.2 类与类加载器
类加载器用于实现类的加载动作，除此之外，任意一个类，都需要由它加载它的类加载器和这个类本身一同确立其在`Java`虚拟机中的唯一性。

每一个类加载器，都拥有一个独立的类名称空间，比较两个类是否相等，只有在两个类由同一个类加载器加载的前提下才有意义。

**相等**代表类的`Class`对象的`equals`方法，`isAssignableFrom`方法，`isInstance`方法。

## 4.3 双亲委派模型
绝大部分`Java`程序都会用到以下三种系统提供的类加载器：
- 启动类加载器
- 扩展类加载器
- 应用类加载器

类加载器之间的层次关系，称为类加载器的双亲委派模型，这个模型要求除了顶层的启动类加载器外，其余的类都应当有自己的父类加载器，一般使用组合来复用父加载器的代码。

双亲委派模型的工作过程：如果一个类加载器收到了类加载的请求，它首先不会去尝试加载这个类，而是把这个请求委派给父类加载器去完成，只有当父类加载器反馈自己无法完成这个加载请求时，子加载器才会尝试自己加载。

# 五、对象实例化
在类加载过程完毕后，如果需要进行实例化对象就需要经过一下步骤，按优先加载父类，再到子类的顺序执行：
- 加载父类构造器
 - 为父类实例对象分配存储空间并赋值
 - 执行父类的初始化块
 - 执行父类构造函数
- 加载子类加载器
 - 为子类实例对象分配存储控件并赋值
 - 执行子类的初始化块
 - 执行子类构造函数

我们用一个简单的例子：
其中`ClassOther`是一个单独的类：
```
public class ClassOther {

    public int mNumber;

    public ClassOther() {
        System.out.println("ClassOther Constructor");
    }

    public void setNumber(int number) {
        this.mNumber = number;
    }

    public int getNumber() {
        return mNumber;
    }
}
```
`ClassChild`则继承于`ClassChild`：

```
public class ClassParent {

    {
        System.out.println("ClassParent before mClassParentContent");
    }

    private ClassOther mClassParentContent = new ClassOther(10);

    {
        System.out.println("ClassParent after mClassParentContent=" + mClassParentContent.mNumber);
    }

    public ClassParent(int number) {
        mClassParentContent.setNumber(number);
        System.out.println("ClassParent Constructor, mClassParentContent=" + mClassParentContent.mNumber);
    }


}

public class ClassChild extends ClassParent {

    {
        System.out.println("ClassChild before a");
    }

    private int mClassChildContent = 1;

    {
        System.out.println("ClassChild after mClassChildContent=" + mClassChildContent);
    }

    public ClassChild() {
        super(2);
        System.out.println("ClassChild Constructor");
    }
}
```
当我们实例化一个`ClassChild`对象时，调用的顺序如下：
![](http://upload-images.jianshu.io/upload_images/1949836-e5d21a198683e0a2.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
