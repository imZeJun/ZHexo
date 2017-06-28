---
title: NDK 知识梳理(1) - 使用 CMake 进行 NDK 开发之初体验
date: 2017-05-20 23:50
categories : NDK 知识梳理
---
# 一、前言
在`Eclipse`的时代，我们进行`NDK`的开发一般需要通过手动执行`NDK`脚本生成`*.so`文件，再将`.so`文件放到对应的目录之后，之后再进行打包。

而如果使用的是`Android Studio`进行`NDK`开发，在`2.2`的版本以后，我们可以不需要手动地运行`NDK`脚本来生成`*.so`文件，而是将这一过程作为`Gradle`构建过程的依赖项，事先编写好编译的脚本文件，然后在`build.gradle`中指定编译脚本文件的路径就可以一次性完成生成原生库并打包成`APK`的过程。

目前这种`AS + Gradle`的`NDK`开发方式又可以分为三种：`ndk-build`、`CMake`和`Experimental Gradle`：
- `ndk-build`：和上面谈到的传统方式相比，它们两个的目录结构相同，`Gradle`脚本其实最终还是依赖于`Android.mk`文件，对于使用传统方式的项目来说，比较容易过度。
- `CMake`：`Gradle`脚本依赖的是`CMakeLists.txt`文件。
- `Experimental Gradle`：需要引入实验性的`gradle`插件，全部的配置都可以通过`build.gradle`来完成，不再需要编写`Android.mk`或者`CMakeLists.txt`，可能坑比较多，对于旧的项目来说过度困难。

目前，`Android Studio`已经将`CMake`作为默认的`NDK`实现方式，并且官网上对于`NDK`的介绍也是基于`CMake`，声称要永久支持。按照官方的教程使用下来，感觉这种方式有几点好处：
- 不需要再去通过`javah`根据`java`文件生成头文件，并根据头文件生成的函数声明编写`cpp`文件
- 当在`Java`文件中定义完`native`接口，可以在`cpp`文件中自动生成对应的`native`函数，所需要做的只是补全函数体中的内容
- 不需要手动执行`ndk-build`命令得到`so`，再将`so`拷贝到对应的目录
- 在编写`cpp`文件的过程中，可以有提示了
- `CMakeLists.txt`要比`Android.mk`更加容易理解

下面，我们就来介绍一下如何使用`CMake`进行简单的`NDK`开发，整个内容主要包括两个方面：
- 创建支持`C/C++`的全新项目
- 在现有的项目中添加`C/C++`代码

# 二、创建支持`C/C++`的全新项目
# 2.1 安装组件
在新建项目之前，我们需要通过`SDK Manager`安装一些必要的组件：
- `NDK`
- `CMake`
- `LLDB`

![](http://upload-images.jianshu.io/upload_images/1949836-5a6fcf81c5aab4c4.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
## 2.2 创建工程
在安装完必要的组件之后，我们创建一个全新的工程，这里需要记得勾选`include C++ Support`选项：
![](http://upload-images.jianshu.io/upload_images/1949836-336344db1cddcc4b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
接下来一路`Next`，在最后一步我们会看见如下的几个选项，它们的含义为：
- `C++ Standard`：选择`C++`的标准，`Toolchain Default`表示使用默认的`CMake`配置，这里我们选择默认。
- `Excptions Support`：如果您希望启用对`C++`异常处理的支持，请选中此复选框。如果启用此复选框，`Android Studio`会将`-fexceptions`标志添加到模块级 `build.gradle`文件的`cppFlags`中，`Gradle`会将其传递到`CMake`。
- `Runtime Type information Support`：如果您希望支持`RTTI`，请选中此复选框。如果启用此复选框，`Android Studio`会将`-frtti`标志添加到模块级 `build.gradle`文件的`cppFlags`中，`Gradle`会将其传递到`CMake`。

![](http://upload-images.jianshu.io/upload_images/1949836-cc71ba53a09305e1.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
在新建工程完毕之后，我们得到的工程的结构如下图所示：
![](http://upload-images.jianshu.io/upload_images/1949836-044ae7a93fc5ab7f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
与传统的工程相比，它有如下几点区别：

**(1) cpp 文件夹**

用于存放`C/C++`的源文件，在磁盘上对应于`app/src/main/cpp`文件夹，当新建工程时，它会生成一个`native-lib.cpp`的事例文件，其内容如下：
```
#include <jni.h>
#include <string>

extern "C"
JNIEXPORT jstring JNICALL
Java_com_demo_lizejun_cmakenewdemo_MainActivity_stringFromJNI(
        JNIEnv* env,
        jobject /* this */) {
    std::string hello = "Hello from C++";
    return env->NewStringUTF(hello.c_str());
}
```

**(2) 增加 CMakeList.txt 脚本**

构建脚本，在磁盘上对应于`app/`目录下的`txt`文件，其内容为如下图所示，这里面涉及到的`CMake`语法包括下面四种，关于`CMake`的语法，可以查看 [官方的 API 说明](https://cmake.org/cmake/help/latest/manual/cmake-commands.7.html)
- `cmake_minimum_required`
- `add_library`
- `find_library`
- `target_link_libraries`

```
# For more information about using CMake with Android Studio, read the
# documentation: https://d.android.com/studio/projects/add-native-code.html

# Sets the minimum version of CMake required to build the native library.

cmake_minimum_required(VERSION 3.4.1)

# Creates and names a library, sets it as either STATIC
# or SHARED, and provides the relative paths to its source code.
# You can define multiple libraries, and CMake builds them for you.
# Gradle automatically packages shared libraries with your APK.

add_library( # Sets the name of the library.
             native-lib

             # Sets the library as a shared library.
             SHARED

             # Provides a relative path to your source file(s).
             src/main/cpp/native-lib.cpp )

# Searches for a specified prebuilt library and stores the path as a
# variable. Because CMake includes system libraries in the search path by
# default, you only need to specify the name of the public NDK library
# you want to add. CMake verifies that the library exists before
# completing its build.

find_library( # Sets the name of the path variable.
              log-lib

              # Specifies the name of the NDK library that
              # you want CMake to locate.
              log )

# Specifies libraries CMake should link to your target library. You
# can link multiple libraries, such as libraries you define in this
# build script, prebuilt third-party libraries, or system libraries.

target_link_libraries( # Specifies the target library.
                       native-lib

                       # Links the target library to the log library
                       # included in the NDK.
                       ${log-lib} )
```

**(3) build.gradle 脚本**

与传统的项目相比，该模块所对应的`build.gradle`需要在里面指定`CMakeList.txt`所在的路径，也就是下面`externalNativeBuild`对应的选项。
```
apply plugin: 'com.android.application'

android {
    compileSdkVersion 25
    buildToolsVersion "25.0.3"
    defaultConfig {
        applicationId "com.demo.lizejun.cmakenewdemo"
        minSdkVersion 15
        targetSdkVersion 25
        versionCode 1
        versionName "1.0"
        testInstrumentationRunner "android.support.test.runner.AndroidJUnitRunner"
        externalNativeBuild {
            cmake {
                cppFlags ""
            }
        }
    }
    buildTypes {
        release {
            minifyEnabled false
            proguardFiles getDefaultProguardFile('proguard-android.txt'), 'proguard-rules.pro'
        }
    }
    externalNativeBuild {
        cmake {
            path "CMakeLists.txt"
        }
    }
}

dependencies {
    compile fileTree(dir: 'libs', include: ['*.jar'])
    androidTestCompile('com.android.support.test.espresso:espresso-core:2.2.2', {
        exclude group: 'com.android.support', module: 'support-annotations'
    })
    compile 'com.android.support:appcompat-v7:25.3.1'
    testCompile 'junit:junit:4.12'
    compile 'com.android.support.constraint:constraint-layout:1.0.2'
}
```
**(4) 打印字符串**

在`MainActivity`中，我们加载原生库，并调用原生库中的方法获取了一个字符串展示在界面上：
```
public class MainActivity extends AppCompatActivity {

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        // Example of a call to a native method
        TextView tv = (TextView) findViewById(R.id.sample_text);
        tv.setText(stringFromJNI());
    }

    /**
     * A native method that is implemented by the 'native-lib' native library,
     * which is packaged with this application.
     */
    public native String stringFromJNI();

    // Used to load the 'native-lib' library on application startup.
    static {
        System.loadLibrary("native-lib");
    }
}
```
**(5) 运行结果**

最后，我们运行一下这个工程，会得到下面的结果：
![](http://upload-images.jianshu.io/upload_images/1949836-754ecfb7e45247f1.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
通过`APK Analyzer`工具，我们可以看到在`APK`当中，增加了`libnative-lib.so`文件：
![](http://upload-images.jianshu.io/upload_images/1949836-e4b0db6ce91dc30d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
## 2.3 原理
下面，我们来解释一下这一过程：
- 首先，在构建时，通过`build.gradle`中`path`所指定的路径，找到`CMakeList.txt`，解析其中的内容。
- 按照脚本中的命令，将`src/main/cpp/native-lib.cpp`编译到共享的对象库中，并将其命名为`libnative-lib.so`，随后打包到`APK`中。
- 当应用运行时，首先会执行`MainActivity`的`static`代码块的内容，使用`System.loadLibrary()`加载原生库。
- 在`onCreate()`函数中，调用原生库的函数得到字符串并展示。

## 2.4 小结
当通过`CMake`来对应用程序增加`C/C++`的支持时，对于应用程序的开发者，只需要关注以下三个方面：
- `C/C++`源文件
- `CMakeList.txt`脚本
- 在模块级别的`build.gradle`中通过`externalNativeBuild/cmake`进行配置

# 三、在现有的项目中添加`C/C++`代码
下面，我们演示一下如何在现有的项目中添加对于`C/C++`代码的支持。

**(1) 创建一个工程**

和第二步不同，这次创建的时候，我们不勾选`nclude C++ Support`选项，那么会得到下面这个普通的工程：
![](http://upload-images.jianshu.io/upload_images/1949836-e3b1aaf66b649c43.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

**(2) 定义接口**

这一次，我们将它定义在一个单独的文件当中：
```
public class NativeCalculator {

    private static final String SELF_LIB_NAME = "calculator";

    static {
        System.loadLibrary(SELF_LIB_NAME);
    }

    public native int addition(int a, int b);

    public native int subtraction(int a, int b);
}
```
这时候因为没有找到对应的本地方法，因此会有如下的错误提示：
![](http://upload-images.jianshu.io/upload_images/1949836-b251196f84cd3712.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

**(3) 定义 cpp 文件**

在模块根目录下的`src/main/`新建一个文件夹`cpp`，在其中新增一个`calculator.cpp`文件，这里，我们先只引入头文件：
```
#include <jni.h>
```

**(4) 定义 CMakeLists.txt**

在模块根目录下新建一个`CMakeLists.txt`文件：
```
cmake_minimum_required(VERSION 3.4.1)

add_library(calculator SHARED src/main/cpp/calculator.cpp)
```

**(5) 在 build.gradle 中进行配置**

之后，我们需要让`Gradle`脚本确定`CMakeLists.txt`所在的位置，我们可以在`CMakeLists.txt`上点击右键，之后选择`Link C++ Project with Gradle`：
![](http://upload-images.jianshu.io/upload_images/1949836-b627f0c196f4e34e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
那么在该模块下的`build.gradle`就会新增下面这句：
![](http://upload-images.jianshu.io/upload_images/1949836-c8ce7a3f84edfb32.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
在这步完成之后，我们选择`Build -> Clean Project`。

**(6) 实现 C++ **

在配置完上面的信息之后，会发现一个神奇的地方，之前我们定义`native`接口的地方，多出了一个选项，它可以帮助我们直接在对应的`C++`文件中生成函数的定义：
![](http://upload-images.jianshu.io/upload_images/1949836-6af56d1bb9ed4a97.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
点击`Create function`之后，我们的`calculator.cpp`文件变成了下面这样：
```
#include <jni.h>

extern "C"
JNIEXPORT jint JNICALL
Java_com_demo_lizejun_cmakeolddemo_NativeCalculator_addition(JNIEnv *env, jobject instance, jint a, jint b) {
    
}

extern "C"
JNIEXPORT jint JNICALL
Java_com_demo_lizejun_cmakeolddemo_NativeCalculator_subtraction(JNIEnv *env, jobject instance,  jint a, jint b) {
    
}
```
下面我们在函数体当中添加实现：
```
#include <jni.h>

extern "C"
JNIEXPORT jint JNICALL
Java_com_demo_lizejun_cmakeolddemo_NativeCalculator_addition(JNIEnv *env, jobject instance, jint a, jint b) {
    return a + b;
}

extern "C"
JNIEXPORT jint JNICALL
Java_com_demo_lizejun_cmakeolddemo_NativeCalculator_subtraction(JNIEnv *env, jobject instance,  jint a, jint b) {
    return a - b;
}
```

**(9) 调用本地函数**
```
public class MainActivity extends AppCompatActivity {

    private NativeCalculator mNativeCalculator;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        mNativeCalculator = new NativeCalculator();
        Log.d("Calculator", "11 + 12 =" + (mNativeCalculator.addition(11,12)));
        Log.d("Calculator", "11 - 12 =" + (mNativeCalculator.subtraction(11,12)));
    }
}
```
最终运行的结果为：
![](http://upload-images.jianshu.io/upload_images/1949836-cbf71590540dafb7.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

# 四、小结
以上就是使用`Android Studio 2.2`以上版本，通过`CMake`来进行`NDK`开发的一个简单例子，主要是学习一下开发的流程，下一篇文章，要学习一下`CMakeLists.txt`中的语法。

# 五、参考文献
[(1) 向您的项目添加 C 和 C++ 代码](https://developer.android.google.cn/studio/projects/add-native-code.html)
[(2) NDK笔记(二) - 在Android Studio中使用 ndk-build](http://www.cnblogs.com/tt2015-sz/p/6148723.html)
[(3) NDK开发 从入门到放弃(一：基本流程入门了解)](http://blog.csdn.net/xiaoyu_93/article/details/52870395)
[(4) NDK开发 从入门到放弃(七：Android Studio 2.2 CMAKE 高效 NDK 开发)](http://blog.csdn.net/xiaoyu_93/article/details/53082088)
[(5) The new NDK support in Android Studio](https://ph0b.com/new-android-studio-ndk-support/)
[(6) Google NDK 官方文档](https://developer.android.google.cn/ndk/index.html)
[(7) Android Studio 2.2 对 CMake 和 ndk-build 的支持](http://www.tuicool.com/articles/Evy2u2u)
[(8) Android开发学习之路--NDK、JNI之初体验](http://blog.csdn.net/eastmoon502136/article/details/50759209)
 [(9) NDK- JNI实战教程（一） 在Android Studio运行第一个NDK程序](http://blog.csdn.net/yanbober/article/details/45309049)
[(10) cmake-commands](https://cmake.org/cmake/help/latest/manual/cmake-commands.7.html#id2)
[(11) CMake](https://developer.android.google.cn/ndk/guides/cmake.html#variables)
[(12) 开发自己的 NDK 程序](http://wiki.jikexueyuan.com/project/jni-ndk-developer-guide/ndk.html)
[(13) NDK开发－Android Studio+gradle-experimental 开发 ndk](http://blog.csdn.net/a396901990/article/details/51922182)
[(14) Android NDK 开发入门指南](http://www.jianshu.com/p/fa613762f516)
