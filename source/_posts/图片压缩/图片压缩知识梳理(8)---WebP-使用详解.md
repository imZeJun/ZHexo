---
title: 图片压缩知识梳理(8) - WebP 使用详解
date: 2017-04-21 22:01
categories : 图片压缩知识梳理
---
# 一、`WebP`是什么
`WebP`、`VectorDrawable`、`PNG`、`JPG`一起构成了`Android`中四种图片的存储格式，在学习这篇文章之前，最好能看一下下面的视频和文章，这样能对`WebP`有一个大概的了解：
- 图片存储格式的对比 [Image Compression For Android Developers](http://v.youku.com/v_show/id_XMTU3ODIyNjcxMg==.html?f=&spm=a2hzp.8250547.0.0&from=y1.7-1.4) 
- `WebP`的概念和优缺点  [WebP 探寻之路](http://isux.tencent.com/introduction-of-webp.html)

下面是上文对于`WebP`的概述，今天这篇文章，我们就一起来看如何在`App`中使用`WebP`。
> `WebP`是一种支持有损压缩和无损压缩的图片文件格式，派生自图像编码格式`VP8`。根据`Google`的测试，无损压缩后的`WebP`比`PNG`文件少了`45％`的文件大小，即使这些`PNG`文件经过其他压缩工具压缩之后，`WebP`还是可以减少`28％`的文件大小。

# 二、兼容性问题
和`VectorDrawable`类似，`WebP`也存在兼容性的问题：
- 在`Android 4.0`以前，默认是不支持`*.webp`，这时候如果图片源是`*.webp`，那么就需要导入外部的依赖，将`*.webp`转换成二进制流，再进行展示。
- 在`Android 4.0+`到`Android 4.2.1`之间，只支持完全不透明的`*.webp`图片。
- 在`Android 4.2.1`之后，对于`*.webp`已经完全支持，我们完全可以像使用`PNG`和`JPG`文件一样使用它。

# 三、`PNG/JPG`和`WebP`格式的互相转换
目前，对于`PNG/JPG`和`WebP`格式之间的互相转换，主要有下面两种方式：
## 3.1 使用`Google`提供的标准工具
对于官方的转换工具的介绍，可以查看下面这篇文档：[https://developers.google.cn/speed/webp/docs/cwebp](https://developers.google.cn/speed/webp/docs/cwebp)
![](http://upload-images.jianshu.io/upload_images/1949836-8108648f87839d21.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
下面，我们就来演示一下整个转换的过程：
- 下载转换的工具 
[https://storage.googleapis.com/downloads.webmproject.org/releases/webp/index.html](https://storage.googleapis.com/downloads.webmproject.org/releases/webp/index.html)
- 下载后解压，进入`/bin`目录下，执行转换的命令：
![](http://upload-images.jianshu.io/upload_images/1949836-9994f1195ee1674b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
## 3.2 使用开源的转换工具
此外，我们也可以使用开源的转换工具：[isparta](http://isparta.github.io/)
![](http://upload-images.jianshu.io/upload_images/1949836-e0523af55018d01e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
## 3.3 `Android Studio`自带的转换工具
在`Android Studio 2.3`之后，已经内置了对于`WebP`格式转换的功能，我们只需要在图片资源上点击右键，弹出菜单的最后一个选项：
![](http://upload-images.jianshu.io/upload_images/1949836-05c89ff02fe6dbc1.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
之后可以选择转换的质量，我们采用默认的配置：
![](http://upload-images.jianshu.io/upload_images/1949836-4d022a55d7208c12.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
在下一个界面，我们可以通过拖动底下的游标，来改变编码的质量，并实时查看转换后的图片展示效果：
![](http://upload-images.jianshu.io/upload_images/1949836-dedf8a1d3bf22e1a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

# 四、`WebP`常规方案
如果`Android`的版本为`4.2.1`以上，那么像使用普通图片一样就可以了：
```
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    tools:context="com.demo.lizejun.repotransition.WebPActivity">
    <ImageView
        android:id="@+id/web_p_1"
        android:layout_width="match_parent"
        android:layout_height="200dp"
        android:src="@drawable/ic_bg"/>
</LinearLayout>
```
展示效果如下：
![](http://upload-images.jianshu.io/upload_images/1949836-6753617f00f3dd16.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
# 五、`WebP`兼容低版本方案
假如当前的`Android`版本不支持`webP`，那么就需要导入外部依赖：[webp-android](https://github.com/EverythingMe/webp-android)，关于兼容性的问题，大家可以参考下面这篇文章： [Android Webp 完全解析 快来缩小apk的大小吧](http://blog.csdn.net/lmj623565791/article/details/53240600)

# 六、参考文献
[1.Image Compression For Android Developers](http://v.youku.com/v_show/id_XMTU3ODIyNjcxMg==.html?f=&spm=a2hzp.8250547.0.0&from=y1.7-1.4) 
[2.WebP 探寻之路](http://isux.tencent.com/introduction-of-webp.html)
[3.Android Webp 完全解析 快来缩小apk的大小吧](http://blog.csdn.net/lmj623565791/article/details/53240600)
[4.关于Android4.+(4.0~4.2.1)上无损、透明webp图像不显示问题分析](http://blog.csdn.net/jeffreyjingsi/article/details/50113185)
