---
title: 插件化知识梳理(5) - Small 框架之如何不将插件打包到宿主中
date: 2017-06-06 12:38
categories : 插件化知识梳理
---
# 一、前言
在前面的例子当中，我们都是把插件预置在`Apk`当中一起安装的，如 [插件化知识梳理(4) - Small 框架之如何实现插件更新](http://www.jianshu.com/p/dee27b35f72f) 所示，我们初始时候会将代表插件的`so`文件放置在`jniLibs/armeabi`目录下。
![](http://upload-images.jianshu.io/upload_images/1949836-3622a4c555908d52.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
那么如果我们将不是很重要的插件放在服务器上，当应用启动之后判断需要该插件，然后再从服务器上下载，将能够有效减小初始时候安装包的体积。

下面，我们就来演示一下如何从外部加载插件。

# 二、示例
加载插件部分的源码位于`Bundle.java`中：
![](http://upload-images.jianshu.io/upload_images/1949836-dc9f704c2df8292c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
可以看到，加载插件有两种方式，它们是根据`Small.isLoadFromAssets`来区分：
- 如果该标志位为真，那么读取`/data/data/{host_pkg_name}/small_base/`下的插件，并且包名和插件的对应关系为：`pkg_name -> {pkg_name}.apk` 
- 如果该标志位为假，那么读取`nativeLibraryDir`目录下的`so`，并且包名和插件的对应关系为：`pkg_name -> lib{pkg_name}.so`，也就是我们之前一直演示的方式。

通过这段源码，那么如何实现加载外部插件就有思路了：
- 删除调`jniLIbs/armeabi`下的`so`文件
- 通过`Small.setLoadFromAssets`方法将标志位设为`true`
- 从网络上获取插件，创建`/data/data/{host_pkg_name}/small_base/{pkg_name}.apk`，通过文件流的形式将下载下来的插件写入到`.apk`当中。

这里因为没有服务器，所以我们把预先编译好的插件`.so`文件拷贝到外部存储中，从外部存储读取的过程就相当于是网络下载的过程：
```
    private void initPlug() {
        //表明需要从外部加载插件。
        Small.setLoadFromAssets(true);
        try {
            File dstFile = new File(FileUtils.getInternalBundlePath(), "com.demo.small.update.app.upgrade.apk");
            if (!dstFile.exists()) {
                dstFile.createNewFile();
            }
            File srcFile = new File(Environment.getExternalStorageDirectory().toString() + "/Small/" + "libcom_demo_small_update_app_upgrade.so");
            FileInputStream inputStream = new FileInputStream(srcFile);
            OutputStream outputStream = new FileOutputStream(dstFile);
            byte[] buffer = new byte[1024];
            int length;
            //将.so的内容写入到.apk当中。
            while ((length = inputStream.read(buffer)) != -1) {
                outputStream.write(buffer, 0, length);
            }
            outputStream.flush();
            outputStream.close();
            inputStream.close();
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
```
