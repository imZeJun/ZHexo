---
title: NDK 知识梳理(2) - 使用 CMake 进行 NDK 开发之如何编写 CMakeLists.txt 脚本
date: 2017-05-21 21:34
categories : NDK 知识梳理
---
# 一、前言
在前一篇文章 [NDK 知识梳理(1) - 使用 CMake 进行 NDK 开发之初体验](http://www.jianshu.com/p/716641d15ee8) 中，我们一起学习了如何在`Android Studio`中使用`CMake`来进行`NDK`开发，而编写`CMakeLists.txt`构建脚本是其中一个重要的环节，今天我们就来一起学习`CMakeLists.txt`的一些应用，介绍它在下面三种场景的用法：
- 从原生代码构建一个原生库
- 添加`Android NDK`的`API`
- 引入第三方`so`

# 二、从原生代码构建一个原生库
## 2.1 指定 CMake 最低版本
`cmake_minimum_required`用于指定`CMake`的最低版本信息，不加入会收到警告。
```
cmake_minimum_required(VERSION 3.4.1)
```
## 2.2 从原生代码构建一个原生库
`add_library()`用于指示`CMake`从原生代码构建一个原生库，通俗地说，就是从`.cpp`经过编译得到`.so`文件。正如我们在 [NDK 知识梳理(1) - 使用 CMake 进行 NDK 开发之初体验](http://www.jianshu.com/p/716641d15ee8) 中看到的那样，我们通过`add_library`让`CMake`根据`native-lib.cpp`源文件构建一个名为`native-lib`的共享库：
```
# Specifies a library name, specifies whether the library is STATIC or
# SHARED, and provides relative paths to the source code. You can
# define multiple libraries by adding multiple add.library() commands,
# and CMake builds them for you. When you build your app, Gradle
# automatically packages shared libraries with your APK.

add_library( # Specifies the name of the library.
             native-lib

             # Sets the library as a shared library.
             SHARED

             # Provides a relative path to your source file(s).
             src/main/cpp/native-lib.cpp )
```
对于`add_library()`括号中的内容，可以分为三个部分：

**(1) 指定原生库的名字**

`add_library`的第一个参数，决定了最终生成的共享库的名字，例如我们将共享库的名字定义为`native-lib`，那么最终生成的`so`文件将在前面加上`lib`前缀，也就是`libnative-lib.so`，但是我们在代码中加载该共享库的时候，仍然应当使用`native-lib`，也就是像下面这样：
```
static {
    System.loadLibrary(“native-lib”);
}
```

**(2) 静态库 or 共享库**

通过第二个参数，我们可以指定根据源文件编译出来的是静态库还是共享库，分别对应`STATIC/SHARED`关键字，这里简单提一下两者的区别：
- 静态库：以`.a`结尾。静态库在程序链接的时候使用，链接器会将程序中使用到函数的代码从库文件中拷贝到应用程序中。一旦链接完成，在执行程序的时候就不需要静态库了。
- 共享库：以`.so`结尾。在程序的链接时候并不像静态库那样在拷贝使用函数的代码，而只是作些标记。然后在程序开始启动运行的时候，动态地加载所需模块。

**(3) 指定源文件**

指定编译的源文件，这里是一个和`CMakeLists.txt`相关的相对路径，如果我们有多个源文件，那么就在后面添加文件的路径即可。

## 2.3 关联多个源文件的例子
下面，我们对 [NDK 知识梳理(1) - 使用 CMake 进行 NDK 开发之初体验](http://www.jianshu.com/p/716641d15ee8) 中的计算器的例子进行优化，把加法和减法的操作放在另一个`.cpp`文件中实现，以演示关联多个`.cpp`文件的例子，整个目录的结构变为：
![](http://upload-images.jianshu.io/upload_images/1949836-ed248e5db30ad7d4.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

- `addition_subtraction.cpp`

```
int addition(int a, int b) {
    return a + b;
}

int subtraction(int a, int b) {
    return a - b;
}


```
- `addition_subtraction.h`

```
#ifndef CMAKEOLDDEMO_ADDITION_SUBTRACTION_H
#define CMAKEOLDDEMO_ADDITION_SUBTRACTION_H

//加法
int addition(int a, int b);

//减法
int subtraction(int a, int b);

#endif
```
- `calculator.cpp`

```
#include <jni.h>
#include "../include/addition_subtraction.h"

extern "C"
JNIEXPORT jint JNICALL
Java_com_demo_lizejun_cmakeolddemo_NativeCalculator_addition(JNIEnv *env, jobject instance, jint a, jint b) {
    return addition(a, b);
}


extern "C"
JNIEXPORT jint JNICALL
Java_com_demo_lizejun_cmakeolddemo_NativeCalculator_subtraction(JNIEnv *env, jobject instance,  jint a, jint b) {
    return subtraction(a, b);
}
```
那么我们需要将`add_library`改写为：
```
cmake_minimum_required(VERSION 3.4.1)

add_library(calculator SHARED src/main/cpp/calculator/calculator.cpp src/main/cpp/calculator/addition_subtraction.cpp)

include_directories(src/main/cpp/include/)
```

# 三、添加 NDK API
在`Android`系统当中，预制了一些标准的`NDK`库，这些库函数的目的就是让开发者能够在原生方法中实现之前在`Java`层开发的一些功能，我们可以通过 [NDK 库](https://developer.android.google.cn/ndk/guides/stable_apis.html) 查找所需要的`API`。
![](http://upload-images.jianshu.io/upload_images/1949836-1eb7f0d29713c385.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

因为这些库已经预制在系统当中了，所以如果我们要调用这些库中的函数，那么不需要将其打包到`APK`当中，所需要做的就是向`CMake`提供希望使用的库名称，并将其关联到自己的原生库，最后在原生代码中引入相应的头文件，调用方法就可以了。

下面，我们就介绍一个调用`Android Native API`的例子。我们给`Calculator`加上一个新的接口，而在其本地方法中调用`Android NDK`的方法来打印一串`Java`层传过来的字符。

**(1) 增加接口**

我们在`NativeCalculator.java`中增加一个接口`logByNative`：
```
public class NativeCalculator {

    private static final String SELF_LIB_NAME = "calculator";

    static {
        System.loadLibrary(SELF_LIB_NAME);
    }

    public native int addition(int a, int b);

    public native int subtraction(int a, int b);
    
    public native void logByNative(String tag, String log);

}
```
**(2) 在 CMakeLists.txt 引入 Android NDK 的 log 库并把它和 calculator 关联**
```
cmake_minimum_required(VERSION 3.4.1)

add_library(calculator SHARED src/main/cpp/calculator/calculator.cpp src/main/cpp/calculator/addition_subtraction.cpp)

include_directories(src/main/cpp/include/)

find_library(log-lib log)

target_link_libraries(calculator ${log-lib})
```
对比于之前，我们增加了下面这两句：
```
find_library(log-lib log)

target_link_libraries(calculator ${log-lib})
```
它们的作用分别是：
- `find_library`：将一个变量和`Android NDK`的某个库建立关联关系。该函数的第二个参数为`Android NDK`中对应的库名称，而调用该方法之后，它就被和第一个参数所指定的变量关联在一起。
在这种关联建立以后，我们就可以使用这个变量在构建脚本的其它部分引用该变量所关联的`NDK`库。
- `target_link_libraries `：把`NDK`库和我们自己的原生库`calculator`进行关联，这样，我们就可以调用该`NDK`库中的函数了。

**(3) 在 Calculator.cpp 引入头文件并调用打印 Log 的函数**
```
#include <jni.h>
#include "../include/addition_subtraction.h"
#include <android/log.h>

extern "C"
JNIEXPORT jint JNICALL
Java_com_demo_lizejun_cmakeolddemo_NativeCalculator_addition(JNIEnv *env, jobject instance, jint a, jint b) {
    return addition(a, b);
}


extern "C"
JNIEXPORT jint JNICALL
Java_com_demo_lizejun_cmakeolddemo_NativeCalculator_subtraction(JNIEnv *env, jobject instance,  jint a, jint b) {
    return subtraction(a, b);
}

extern "C"
JNIEXPORT void JNICALL
Java_com_demo_lizejun_cmakeolddemo_NativeCalculator_logByNative(JNIEnv *env, jobject instance, jstring tag_, jstring log_) {
    const char *tag = env->GetStringUTFChars(tag_, 0);
    const char *log = env->GetStringUTFChars(log_, 0);
    __android_log_write(ANDROID_LOG_DEBUG, tag, log);
    env->ReleaseStringUTFChars(tag_, tag);
    env->ReleaseStringUTFChars(log_, log);
}
```
这里需要做的就是两步：
- 引入头文件

```
#include <android/log.h>
```
- 调用`NDK`库中的方法

```
 __android_log_write(ANDROID_LOG_DEBUG, tag, log);

```
**(4) 在 Java 中调用，观察结果**

```
public class MainActivity extends AppCompatActivity {

    private NativeCalculator mNativeCalculator;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        mNativeCalculator = new NativeCalculator();
        Log.d("Calculator", "11 + 12 = " + (mNativeCalculator.addition(11,12)));
        Log.d("Calculator", "11 - 12 = " + (mNativeCalculator.subtraction(11,12)));
        mNativeCalculator.logByNative("Calculator", "Log By Native");
    }

}
```
最终的打印结果为：
![](http://upload-images.jianshu.io/upload_images/1949836-e40ec6409684dfe6.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

# 四、引入第三方的`.so`库
最后一部分，我们举一个通过第三方`.so`库来实现乘除法的例子，为了得到一个`.so`库，我们通过新建一个工程，然后将它编译出的`.apk`文件解压，取出其中的`.so`文件。

这里获得第三方`so`的原理其实和我们之前一直谈到的其实是一样的，我们只是借助了`Android Studio`来模拟了这个流程。

## 3.1 获得第三方 so 库
新建工程的目录结构为：
![](http://upload-images.jianshu.io/upload_images/1949836-46f9c05b228d06e7.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
- `multiplication_division.h`

```
#ifndef SOMAKER_MULTIPLICATION_DIVISION_H
#define SOMAKER_MULTIPLICATION_DIVISION_H

int multiplication(int a, int b);

int division(int a, int b);

#endif
```
- `multiplication_division.cpp`

```
int multiplication(int a, int b) {
    return a * b;
}

int division(int a, int b) {
    return a / b;
}


```
- `CMakeLists.txt`

```
cmake_minimum_required(VERSION 3.4.1)

add_library(multiplication_division SHARED src/main/cpp/multiplication_division.cpp)

include_directories(src/main/cpp/include/)

```
在`build.gradle`的`android`节点下，增加构建任务：
```
    externalNativeBuild {
        cmake {
            path 'CMakeLists.txt'
        }
    }
```
这个工程编译完毕之后，去`app/build/outputs/apk/`目录下将编译出来的`APK`文件解压，得到`libmultiplication_division.so`库，我们将它作为第三方的`so`库导入到计算器的例子当中。
![](http://upload-images.jianshu.io/upload_images/1949836-480efe7d3ca43abf.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
## 3.2 引入第三方 so 库

**(1) 将 so 库和头文件拷贝到对应的目录**
![](http://upload-images.jianshu.io/upload_images/1949836-323a50fe9f136c8e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

**(2) 修改 CMakeLists.txt 文件**
```
cmake_minimum_required(VERSION 3.4.1)

add_library(calculator SHARED src/main/cpp/calculator/calculator.cpp src/main/cpp/calculator/addition_subtraction.cpp)

include_directories(src/main/cpp/include/)

add_library(multiplication_division SHARED IMPORTED)

set_target_properties(multiplication_division PROPERTIES IMPORTED_LOCATION ${CMAKE_SOURCE_DIR}/src/main/jniLibs/${ANDROID_ABI}/libmultiplication_division.so )

find_library(log-lib log)

target_link_libraries(calculator multiplication_division ${log-lib})
```
这里相比于之前，修改了以下三句：
```
add_library(multiplication_division SHARED IMPORTED)

set_target_properties(multiplication_division PROPERTIES IMPORTED_LOCATION ${CMAKE_SOURCE_DIR}/src/main/jniLibs/${ANDROID_ABI}/libmultiplication_division.so )

target_link_libraries(calculator multiplication_division ${log-lib})
```
这三句话的作用分别为：
- 添加第三方`so`库
这里和之前在第二步中介绍的创建一个新的原生库类似，区别在于最后一个参数，我们通过`IMPORTANT`标志告知`CMake`只希望将库导入到项目中。

- 指定目标库的路径
这里有几点需要说明：
 - `${CMAKE_SOURCE_DIR}`表示的是`CMakeLists.txt`所在的路径，我们指定第三方`so`所在路径时，应当以这个常量为起点。
 - 按理来说，我们应当为每种`ABI`接口提供单独的软件包，那么，我们就可以在`jinLibs`下建立多个文件夹，每个文件夹对应一种`ABI`接口类型，之后再通过`${ANDROID_ABI}`来泛化这一层目录的结构，这样将有助于充分利用特定的`CPU`架构。

- 将第三方的库关联到原生库
这里和将`NDK`库关联到原生库的原理是一样的。

**(3) 声明新的接口，并在 Calculator.cpp 引入第三方的头文件，调用函数**
```
package com.demo.lizejun.cmakeolddemo;

public class NativeCalculator {

    private static final String SELF_LIB_NAME = "calculator";

    static {
        System.loadLibrary(SELF_LIB_NAME);
    }

    public native int addition(int a, int b);

    public native int subtraction(int a, int b);

    public native void logByNative(String tag, String log);

    public native int multiplication(int a, int b);

    public native int division(int a, int b);

}
```
```
#include <jni.h>
#include "../include/addition_subtraction.h"
#include <android/log.h>
#include "../include/multiplication_division.h"

extern "C"
JNIEXPORT jint JNICALL
Java_com_demo_lizejun_cmakeolddemo_NativeCalculator_addition(JNIEnv *env, jobject instance, jint a, jint b) {
    return addition(a, b);
}


extern "C"
JNIEXPORT jint JNICALL
Java_com_demo_lizejun_cmakeolddemo_NativeCalculator_subtraction(JNIEnv *env, jobject instance,  jint a, jint b) {
    return subtraction(a, b);
}

extern "C"
JNIEXPORT void JNICALL
Java_com_demo_lizejun_cmakeolddemo_NativeCalculator_logByNative(JNIEnv *env, jobject instance, jstring tag_, jstring log_) {
    const char *tag = env->GetStringUTFChars(tag_, 0);
    const char *log = env->GetStringUTFChars(log_, 0);
    __android_log_write(ANDROID_LOG_DEBUG, tag, log);
    env->ReleaseStringUTFChars(tag_, tag);
    env->ReleaseStringUTFChars(log_, log);
}

extern "C"
JNIEXPORT jint JNICALL
Java_com_demo_lizejun_cmakeolddemo_NativeCalculator_multiplication(JNIEnv *env, jobject instance, jint a, jint b) {
    return multiplication(a, b);
}

extern "C"
JNIEXPORT jint JNICALL
Java_com_demo_lizejun_cmakeolddemo_NativeCalculator_division(JNIEnv *env, jobject instance, jint a, jint b) {
    return division(a, b);
}
```
之前的步骤完成之后就很简单了，我们只需要引入该`so`库对应的头文件，再调用它提供的方法就可以了。

**(4) 在 Java 中调用本地方法**
```
public class MainActivity extends AppCompatActivity {

    private NativeCalculator mNativeCalculator;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        mNativeCalculator = new NativeCalculator();
        Log.d("Calculator", "11 + 12 = " + (mNativeCalculator.addition(11,12)));
        Log.d("Calculator", "11 - 12 = " + (mNativeCalculator.subtraction(11,12)));
        mNativeCalculator.logByNative("Calculator", "Log By Native");
        Log.d("Calculator", "11 * 12 = " + (mNativeCalculator.multiplication(11,12)));
        Log.d("Calculator", "11 / 12 = " + (mNativeCalculator.division(11,12)));
    }

}
```
运行程序，最终打印的结果为：
![](http://upload-images.jianshu.io/upload_images/1949836-9ac567d97d30d67e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

# 四、小结
这一篇文章，我们简要地总结了`CMakeLists.txt`在几种场景下应该如何编写。在学习的过程中，感觉之前学的`C/C++`都忘光了，头文件、静态库/动态库、`extern`关键字，都不记得了，打算先好好复习一下相关的知识再继续`NDK`的学习了。
