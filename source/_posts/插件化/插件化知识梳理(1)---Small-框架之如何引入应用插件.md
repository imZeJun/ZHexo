---
title: 插件化知识梳理(1) - Small 框架之如何引入应用插件
date: 2017-06-03 10:10
categories : 插件化知识梳理
---
# 一、前言
上个星期，公司里有一个小的讲座，对插件化进行了简单的介绍，因此决定开始研究一下这方面的知识。

在网上查了一些相关的资料，发现了`Small`这个开源的插件化框架，因此打算从它入手，通过它的内部实现，学习一下插件化的相关原理。这篇文章是个开篇，先从一个简单的例子开始，把环境给搭建好。先给大家分享一些这几天查阅的资料，如果大家有比较好的文章也可以留言或者私信我：

[Android博客周刊专题之＃插件化开发＃](http://www.androidblog.cn/index.php/Index/detail/id/16#)

[Small Github 官网](https://github.com/wequick/Small)
[Small Issues](https://github.com/wequick/Small/issues) 
[Small 快速入门](https://github.com/wequick/Small/blob/master/Android/GETTING-STARTED.md)

[Android Small 源码分析 (一) 启动流程](http://www.jianshu.com/p/3724ee20ad99)
[Android Small 源码分析 (二) 插件加载过程](http://www.jianshu.com/p/3724ee20ad99)

 [Android Small 插件化框架源码分析](http://blog.csdn.net/yueqian_scut/article/details/51010078)

 [Android Small 插件化框架 -- 启动插件 Activity 源码解析(上)](http://blog.csdn.net/qq_21920435/article/details/52943370)
[Android Small 插件化框架 -- 启动插件 Activity 源码解析(下)](http://blog.csdn.net/qq_21920435/article/details/53158432)
[Android Small 插件化框架 -- Android 应用类加载机制](http://blog.csdn.net/qq_21920435/article/details/56015603)
[Android Small 插件化框架 -- 类加载实现解析](http://blog.csdn.net/qq_21920435/article/details/56674705)

# 二、基本示例
## 2.1 简要介绍
对于`Small`来说，一个最简单的框架分为三个部分：
- 宿主
- 插件
- `bundle.json`，用于宿主和插件之间的路由。

本文所用的是一个最简单的例子，因此在代码上基本不会有什么问题，主要是环境上的区别，遇到编译不过的问题可以多多百度，下面是我所采用的环境：
- `Android Studio` 版本：`Android Studio 3.0`
- `Gradle` 版本：`gradle-3.5-all.zip`
- `compileSdkVersion`：`24`
- `buildToolsVersion`：`24.0.2`

完整的例子可以查看 [https://github.com/imZeJun/SmallDemo](https://github.com/imZeJun/SmallDemo)

## 2.2 具体实现
整个具体的实现分为五步：
- 新建工程/宿主模块
- 修改项目根目录下的`build.gradle`文件，引入`Small`插件
- 新建插件模块
- 完善宿主模块
- 编译，安装

### 2.2.1 新建工程/宿主模块
这里比较关键的一点，是需要在新建工程/宿主模块的时候，将包名修正为`com.demo.small`，这是为了和以后的`lib/app`模块形成统一：
![](http://upload-images.jianshu.io/upload_images/1949836-08a577c308dd1aa7.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 2.2.2 修改项目根目录下的 build.gradle 文件

对于项目的`build.gradle`，修改包含以下三个部分：
- 必选，在`dependencies`节点中引入远程依赖。
- 必选，通过`apply plugin`应用插件。
- 可选，配置`Small`代码库版本。

```
buildscript {

    repositories {
        maven { url 'https://maven.google.com' }
        jcenter()
    }
    dependencies {
        classpath 'com.android.tools.build:gradle:2.2.3'
        //1.引入Small依赖，必选。
        classpath 'net.wequick.tools.build:gradle-small:1.2.0-alpha3'
    }
}

allprojects {
    repositories {
        jcenter()
        maven { url 'https://maven.google.com' }
    }
}

task clean(type: Delete) {
    delete rootProject.buildDir
}

//2.应用插件，必选。
apply plugin: 'net.wequick.small'

//3.配置Small的代码库版本，需要放在第2步的下面，否则会报错，可选。
small {
    aarVersion = '1.2.0-alpha3'
}
```

### 2.2.3 新建插件模块
这里用到的插件模块很简单，就是位于另一个模块中的`Activity`，选择`File -> New -> New Module`：
![](http://upload-images.jianshu.io/upload_images/1949836-62b3239f0eb82cbb.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
插件模块中最最关键的一点，就是插件模块的包名，它的包名分为两个部分
- 第一部分和宿主模块相同
- 第二部分要根据插件的类型来决定：
 - 如果是`Phone & Tablet Module`：那么要以`app.xxx`结尾
 - 如果是`Android Library`，那么要以`lib.xxx`结尾

这里，我们先演示第一种：
![](http://upload-images.jianshu.io/upload_images/1949836-4f3228278a38857e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
在插件模块中，我们声明一个新的`PlugActivity`，它的布局为：
```
<?xml version="1.0" encoding="utf-8"?>
<FrameLayout
    xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:background="@android:color/white"
    tools:context="com.demo.small.app.main.PlugActivity">

    <TextView
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:layout_gravity="center"
        android:textColor="@android:color/black"
        android:text="PlugActivity started success"/>

</FrameLayout>
```

### 2.2.4 完善宿主模块

**(a) 配置路由协议**

接下来要在宿主模块中进行路由配置，我们在宿主模块上单击右键，新建一个`assets`文件夹，之后在`assets`文件夹中，新建一个路由文件，`bundle.json`文件，注意`assets`文件夹从`Project`视图上所处的位置如下图所示，千万不要放错地方了：
![](http://upload-images.jianshu.io/upload_images/1949836-252613a9df81850b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
在`bundle.json`中，我们声明插件模块：
```
{
  "version": "1.0.0",
  "bundles": [
    {
      "uri": "main",
      "pkg": "com.demo.small.app.main"
    }
  ]
}
```
** (b) 在宿主模块的自定义 Application 中进行预加载**

```
public class SmallHostApp extends Application {

    public SmallHostApp() {
        //Small初始化。
        Small.preSetUp(this);
    }
}
```
** (c) 将自定义的 Application 配置到宿主模块的 AndroidManifest.xml 中**
```
<?xml version="1.0" encoding="utf-8"?>
<manifest xmlns:android="http://schemas.android.com/apk/res/android" package="com.demo.small">

    <application
        android:name=".app.SmallHostApp"
        android:allowBackup="true"
        android:icon="@mipmap/ic_launcher"
        android:label="@string/app_name"
        android:supportsRtl="true"
        android:theme="@style/AppTheme">
        <activity android:name=".LaunchActivity">
            <intent-filter>
                <action android:name="android.intent.action.MAIN" />
                <category android:name="android.intent.category.LAUNCHER" />
            </intent-filter>
        </activity>
    </application>

</manifest>
```
** (d) 在启动 Activity 的 onCreate() 方法中加载插件，点击按钮后跳转到插件的Activity**

```
public class LaunchActivity extends AppCompatActivity {

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_launch);
        setUp();
    }

    private void setUp() {
        Small.setUp(this, new net.wequick.small.Small.OnCompleteListener() {

            @Override
            public void onComplete() {
                Log.d("LaunchActivity", "onComplete");
            }
        });
    }

    public void startPlugActivity(View view) {
        Small.openUri("main", LaunchActivity.this);
    }
}
```
### 2.2.5 编译&安装
最后一步，就是进行编译和安装，编译时：
- 准备基础库 & 打包所有组件

```
./gradlew buildLib -q && ./gradlew buildBundle -q
```
- 安装：

```
 ./gradlew assembleDebug && adb install -r app/build/outputs/apk/app-debug.apk 
```
- 清除基础库 & 清除所有组件：

```
./gradlew cleanLib -q && ./gradlew cleanBundle -q
```




## 2.3 最终效果
![](http://upload-images.jianshu.io/upload_images/1949836-ce12ee66da455f70.gif?imageMogr2/auto-orient/strip)
