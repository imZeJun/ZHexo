---
title: 插件化知识梳理(2) - Small 框架之如何引入公共库插件
date: 2017-06-03 18:28
categories : 插件化知识梳理
---
# 一、前言
在 [插件化知识梳理(1) - Small 框架之如何引入应用插件](http://www.jianshu.com/p/07f88d4924db) 中，我们简要地介绍了如何使用`Small`框架来通过插件实现一个`Activity`的跳转，也就是`app.main`，这里的`app.main`我们称为**应用插件**，除了应用插件外，还有一种称为**公共库插件**。

对于这两种插件的职责，`Small`官方建议的基本原则为
>- 公共库插件
 - 把各个 第三方库拆出来做成`lib`插件模块，包括统计、地图、网络、图片等库。
 - 把老项目积累的业务公共代码`utils`分离出来封装成一个`lib.utils`插件
 - 把基础的样式、主题分离出来封装成一个`lib.style`插件


>- 应用插件
 - 把业务模块拆成`app`模块，他们可以依赖`lib`模块，显示调用`lib`中的各个`API`
 - 相对独立的业务模块先拆，先放一个插件里
 - 如果都不好拆，先把全部业务做成一个`app.main`主插件

总结下来就是：
- 应用插件模块相互之间不直接引用，但是可以引用公共库插件模块
- 公共库插件模块不应当相互引用，也不应当引用应用插件模块
- 一个公共库插件模块可以被多个应用插件模块引用

在项目当中，许多个模块有可能用到一些共有的东西，例如图片加载、网络请求、公共控件等，那么我们就有必要将这些东西封装成共同库让各个模块去调用，一般来说，可以分为以下两种：
- 工具类，例如网络请求、图片加载、持久性存储等。
- 资源类，例如公共控件、图片资源、主题风格。

对于上面这两类，我们一般称为公共库插件，它的命名方式为`lib.xxx`。公共库插件模块分为两个阶段：
- 在开发阶段，我们可以通过`compile project(':lib_module_name')`让应用插件模块来引用它，以调用它模块中所定义的方法或使用主题样式。
- 在编译阶段，共同库插件会被打包成为一个可独立更新的插件。

下面，我们就分这两个方面，这介绍一下使用建立公共库插件。

# 二、公共库
## 2.1 新建 lib.utils 模块作为工具类共同库
**(a) 新建插件库 Android Library**
![](http://upload-images.jianshu.io/upload_images/1949836-30b33139ba738cc9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
**(b) 注意包名的定义，要分为两个部分**
![](http://upload-images.jianshu.io/upload_images/1949836-27130e21efcbe0a2.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
新建完毕之后，我们的项目结构变为下面这样：
![](http://upload-images.jianshu.io/upload_images/1949836-6137aadd2f35e96c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
**(c) 编写工具类代码**

在`lib.utils`模块中，引入第三方库`Glide`，用于图片的加载：
```
dependencies {
    //引入第三方库Glide。
    compile 'com.github.bumptech.glide:glide:3.7.0'
}
```
新建一个简单的图片加载类`ImageLoader`：
```
public class ImageLoader {

    public static void loadImage(Context context, String imgUrl, ImageView img) {
        Glide.with(context).load(imgUrl).into(img);
    }
}
```
## 2.2 新建 lib.style 作为资源类共同库
**(a) 新建插件库 Android Library**
![](http://upload-images.jianshu.io/upload_images/1949836-2bd231ce74f2c3b8.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
**(b) 在 res 目录下新建 styles.xml 文件**

在`styles.xml`文件中，我们定义一些公共的样式：
```
<?xml version="1.0" encoding="utf-8"?>
<resources>
    <style name="StyleTextViewTitle">
        <item name="android:textSize">45px</item>
        <item name="android:textColor">#333333</item>
        <item name="android:padding">12dp</item>
    </style>

    <style name="StyleTextViewSubTitle">
        <item name="android:textSize">27px</item>
        <item name="android:textColor">#888888</item>
    </style>

</resources>
```

## 2.3 修改插件模块 app.main
** (a) 引入 lib.utils 和 lib.style **
```
dependencies {
    //...
    compile project(':lib.utils')
    compile project(':lib.style')
}
```
**(b) 在布局文件中，使用 lib.style 的公共样式**
```
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout
    xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools"
    android:orientation="vertical"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:background="@android:color/white"
    tools:context="com.demo.small.app.main.PlugActivity">
    <ImageView
        android:id="@+id/iv_header"
        android:layout_gravity="center_horizontal"
        android:layout_width="200dp"
        android:layout_height="200dp"/>
    <TextView
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="Small Android"
        android:layout_gravity="center_horizontal"
        style="@style/StyleTextViewTitle"/>
    <TextView
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="Small 插件化方案适用于将一个APK拆分为多个公共库插件、业务模块插件的场景。"
        style="@style/StyleTextViewSubTitle"/>
</LinearLayout>
```
** (c) 在代码中，使用 lib.utils 定义的接口**
```
public class PlugActivity extends AppCompatActivity {
    
    private static final String IMG_URL = "http://i6.hexun.com/2017-06-02/189461191.jpg";

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_plug);
        ImageView imageView = (ImageView) findViewById(R.id.iv_header);
        //调用 lib.utils 中定义的接口。
        ImageLoader.loadImage(this, IMG_URL, imageView);
    }
}
```
## 2.4 修改宿主模块的 bundle.json 文件：
```
{
  "version": "1.0.0",
  "bundles": [
    {
      "uri": "lib.utils",
      "pkg": "com.demo.small.lib.utils"
    },
    {
      "uri": "lib.style",
      "pkg": "com.demo.small.lib.style"
    },
    {
      "uri": "main",
      "pkg": "com.demo.small.app.main"
    }
  ]
}
```
## 2.6 重新编译
![](http://upload-images.jianshu.io/upload_images/1949836-ffa28c8ac6357304.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
## 2.7 最终效果
![](http://upload-images.jianshu.io/upload_images/1949836-25c56d6e5e0777f2.gif?imageMogr2/auto-orient/strip)
