---
title: Android 数据存储知识梳理(2) - Android存储目录
date: 2017-03-23 19:46
categories : Android 数据存储知识梳理
---
# 一、概述
对于`Android`系统中的存储，一般分为以下三个方面：
- 内部存储
- 外部存储
 - 外部缓存目录
 - 外部永久目录
- `SD`卡存储

# 二、内部存储
## 2.1 内部存储的特点
内部存储指的是下面这个路径下的文件夹或者文件：
```
/data/data/应用包名/
```
截图如下：
![](http://upload-images.jianshu.io/upload_images/1949836-e150b57fb12568df.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
对于内部存储来说，有这么几个特点：
- 应用不需要声明读写权限就能操作这个目录下的文件夹或者文件
- 一般情况下，只有应用本身具有操作该目录的权利
- 当应用卸载之后，该目录也会被删除
- 这部分目录，普通用户通过手机自带的文件管理器是看不到的，除非使用`Root Explorer`等工具才可以看到，并且要申请`Root`权限才能进行读写操作。

`Android`提供了一些方法来操作内部存储的路径，我们可以选择它作为我们提供的标准目录，也可以自己创建目录，下面我们开始介绍。
## 2.2 标准文件目录`/data/data/应用包名/files`
- 在`files`目录下创建、读写文件，可以用下面这一对方法
```
    public static void writeToPrivateFile(Context context, String str, String path) {
        FileOutputStream fileOutputStream = null;
        BufferedWriter bufferedWriter;
        try {
            fileOutputStream = context.openFileOutput(path, Context.MODE_APPEND);
            bufferedWriter = new BufferedWriter(new OutputStreamWriter(fileOutputStream));
            bufferedWriter.write(str);
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            try {
                if (fileOutputStream != null) {
                    fileOutputStream.close();
                }
            } catch (Exception e) {
                e.printStackTrace();
            }
        }
    }

    public static String readFromPrivateFile(Context context, String path) {
        FileInputStream fileInputStream = null;
        BufferedReader bufferedReader;
        String result = null;
        try {
            fileInputStream = context.openFileInput(path);
            bufferedReader = new BufferedReader(new InputStreamReader(fileInputStream));
            String line;
            while ((line = bufferedReader.readLine()) != null) {
                result += line;
            }
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            try {
                if (fileInputStream != null) {
                    fileInputStream.close();
                }
            } catch (Exception e) {
                e.printStackTrace();
            }
        }
        return result != null ? result : "";
    }
```
- 在`files`目录下创建新的目录
上面的方法是直接打开输入输出流，读写位于`files`根目录下的文件，如果我们希望进行更灵活的操作，那么通过下面方法可以获得`files`文件夹所指向的`File`对象，再通过这个返回的`File`对象操作：
```
File file = context.getFilesDir();
```
- 列出`files`目录下的所有文件名
```
String[] files = context.fileList();
```
- 删除`files`目录下的文件
```
context.deleteFile(path);
```

## 2.3 标准缓存目录`data/data/应用包名/cache`
和`files`目录相比，`cache`目录有一个特点：就是当系统存储空间不足时，会删除其中的文件夹：
```
File cacheFile = context.getCacheDir();
```
## 2.4 代码缓存目录`data/data/应用/code_cache`
它和上面`cache`目录类似，都是只能得到一个`File`对象，同样的，它也有一个特点，就是当`App`升级时，会删除该目录下的内容，这个`API`要求大于`21`：
```
File code = context.getCodeCacheDir()
```
## 2.5 `SharePreference`和数据库保存的目录
在平时的开发中，我们经常会用到`SharePreference`来保存数据，这些数据就位于：`/data/data/应用包名/shared_prefs`目录下，而数据库则保存在`/data/data/应用包名/databases`下。
## 2.6 根目录
除了使用上面四种根目录之外，我们还可以直接在`data/data/应用包名`目录下新建目录，调用下面的方法会我们在`/data/data/应用包名/`目录下新建一个名字为`app_<path>`的文件夹，并返回这个文件夹的`File`对象，之后，我们再通过这个对象进行相应的操作：
```
File dir = context.getDir(path, Context.MODE_APPEND);
```

# 三、外部存储
这里指的外部存储是我们平时常说的`32g/64g`，它是手机出厂时自带的，不需要再额外的插入`SD`卡，它主要解决上面内部存储的两个问题：
- 外部应用无法访问
- 数据随着应用数据被卸载而被删除了

也就是下面截图中部分的存储：
![](http://upload-images.jianshu.io/upload_images/1949836-281e0b6fca591ec7.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
## 3.1 独立于应用的外部存储
这部分存储的特点是：
- 需要申请读写权限才能访问
- 任何有权限的应用都可以访问它
- 不会随着应用的卸载而被删除

由于这部分的存储和应用无关，因此它的方法都不是通过`Context`来调用，而是通过`Environment`的静态方法来返回一个`File`对象，我们再通过这个`File`对象进行操作。
- `getRootDirectory`
返回是`/system`目录，它和`/sdcard`以及`/data`是同级的.
- `getExternalStorageDirectory`
返回的是外部存储的根目录，也就是我们平时从文件管理器中能看到的最顶级目录，它的`File`绝对路径为：`/storage/emulated/0`
- `getDownloadCacheDirectory`
返回是`/cache`目录，它和`/system`以及`/data`是同级的.
- `getExternalStoragePublicDirectory(String type)`
其中，`type`的选项有如下几种：
![](http://upload-images.jianshu.io/upload_images/1949836-ada660f77dffcf90.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
最终得到的文件夹管理器下面的几个一级目录，例如`DIRECTORY_DCIM`得到的就是下面这个目录：
![](http://upload-images.jianshu.io/upload_images/1949836-e0ce2ea133d4fd79.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 3.2 和应用相关的外部存储
这部分存储的特点是：
- `4.4`以后不需要申请读写权限也能访问
- 随着应用的卸载而被删除
- 该目录可以被其它应用访问

这部分存储的位置位于`/Android/data/应用包名/`下，例如下面这样，就是`com.android.browser`的应用相关外部存储：
![](http://upload-images.jianshu.io/upload_images/1949836-fb66e68876e111ca.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

- `getExternalCacheDir()`
返回的是`/Android/data/应用包名/cache`目录所对应的`File`对象
- `getExternalFilesDir(String type)`
`type`和`Environment`中定义的类型相同，根据`type`的不同返回不同的`File`对象，例如`DIRECTORY_DCIM`，那么得到的是`/Android/data/应用包名/files/DCIM`所对应的`File`对象。
