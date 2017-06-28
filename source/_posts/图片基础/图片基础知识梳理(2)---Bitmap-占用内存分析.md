---
title: 图片基础知识梳理(2) - Bitmap 占用内存分析
date: 2017-03-16 19:53
categories : 图片基础知识梳理
---
# 一、概述
今天介绍一些关于`Bitmap`的基础知识：
- `Bitmap`是什么
- 屏幕密度相关概念
- 工程中各`drawable`文件夹
- `Bitmap.Options`
- 简要分析`Bitmap`所占内存

# 二、什么是`Bitmap`
官方的说法是：`Bitmap`是对位图的抽象。
形象地来说，我们在手机屏幕上所看到的图片就是由一个个像素点拼接而成的，每个像素点的都是用不同数量的二进制位来表示，而`Bitmap`就是用来保存这些二进制位。

# 三、`Bitmap`占用内存分析
## 3.1 屏幕密度的相关概念
前面我们说到，`Bitmap`的最终目的是为了在屏幕上显示图片，所以首先我们需要了解关于屏幕密度的相关知识：
- 像素`px`：如果我们近距离地看手机屏幕，就可以发现它有一个个小点，每一个小点就表示一个像素，我们通常称它为`px`。
- 屏幕分辨率：指的是**屏幕的上的像素总和**，例如常说的某某手机屏幕分辨率为`1920 * 1080`，也就是说它的高有`1920px`，宽为`1080px`，像素点总和就是`1920 * 1080px`。
- 屏幕尺寸：指的是**屏幕的对象线长度**，这里的长度不是用`px`为单位，而是用英寸为单位的，它和我们平时说的米、厘米是一个概念。
- `ppi`：英文名为`pixel per inch`，中文全称为**每英寸屏幕上的像素数**，它决定了**屏幕的质量**，通常是按手机的对角线来计算的，也就是说，它等于**对象线的像素个数除以对角线的长度（单位为英寸）**。
- `dp/dip`：英文名为`density-independent pixel`，这是安卓特有的概念，它和`px`的作用相同，都是用来表示长度。但是它和硬件屏幕无关，如果需要转换为`px`，那么`1dp`在屏幕上最终会显示为`(ppi / 160)`个`px`。
- `dpi`：英文名为`dot per inch`，中文全称为**每英寸图片上点的个数**，它决定了**图片的质量**，比如`dpi`为`320`，那么最终在手机上一张`320 * 320`的图片就会用`320 * 320 * (dpi / 160)`个像素点来表示。

## 3.2 动态获得上述的信息
在`Android`中，通过`DisplayMetrics`可以获得上述的信息：
```
    public static void logDensityInfo(Activity activity) {
        DisplayMetrics displayMetrics = new DisplayMetrics();
        //将信息保存到displayMetrics中.
        activity.getWindowManager().getDefaultDisplay().getMetrics(displayMetrics);
        //1.x轴和y轴的dpi.
        Log.d("logDensityInfo", "ydpi=" + displayMetrics.ydpi);
        Log.d("logDensityInfo", "xdpi=" + displayMetrics.xdpi);
        //2.x轴和y轴的像素个数.
        Log.d("logDensityInfo", "heightPixels=" + displayMetrics.heightPixels);
        Log.d("logDensityInfo", "widthPixels=" + displayMetrics.widthPixels);
        //3.dpi
        Log.d("logDensityInfo", "densityDpi=" + displayMetrics.densityDpi);
        //4.dpi/160.
        Log.d("logDensityInfo", "density=" + displayMetrics.density);
        //5.通常情况下和density相同.
        Log.d("logDensityInfo", "scaledDensity=" + displayMetrics.scaledDensity);
    }
```
在我的手机上，最终的结果是：
![](http://upload-images.jianshu.io/upload_images/1949836-b1414217cfc0b938.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
那么我们看一下`dpi`相关的值是怎么获得的：
```
    public void setToDefaults() {
        density =  DENSITY_DEVICE / (float) DENSITY_DEFAULT;
        densityDpi =  DENSITY_DEVICE;
        scaledDensity = density;
        xdpi = DENSITY_DEVICE;
        ydpi = DENSITY_DEVICE;
    }
```
其中`DENSITY_DEFAULT`为`160`，而`DENSITY_DEVICE`则是通过下面方式得到的：
```
private static int getDeviceDensity() {
    return SystemProperties.getInt("qemu.sf.lcd_density", SystemProperties.getInt("ro.sf.lcd_density", DENSITY_DEFAULT));
}
```
## 3.3 `drawable`文件夹内的图片
说起`dpi`，自然就会想到`res`文件夹下的`drawable-?`文件夹，我们在平时开发中会发现，如果将同一大小的图片，放在不同的`drawable`文件夹下，最终在屏幕上展现的大小是不一样的。
这其实是`Android`在读取资源的时候，会根据文件夹的不同，给每个文件夹定义一个叫做`density`的属性，根据最终找到的资源所在的文件夹位置，会有以下几种情况：
- 存在于和手机所匹配的`dpi`文件夹：不进行缩放。
- 存在于非`drawable-nodpi`其它的`dpi`文件夹：那么最终的长度变为`(原始长度 / 所在文件夹density) * 匹配文件夹density`
- 存在`drawable-nodpi`：不进行缩放。

而每个`ARBG`位的二进制个数相乘，就是图片所占内存的大小，对应的`density`如下：
- `drawable-nodpi`：不缩放
- `drawable-ldpi`：`0.75`
- `drawable`：`1`
- `drawable-mdpi`：`1`
- `drawable-hdpi`：`1.5`
- `drawable-xhdpi`：`2`
- `drawable-xxhdpi`：`3`
- `drawable-xxhdpi`：`4`

如果某个资源存在于上面的多个文件夹下，它的匹配优先级如下：
- 和手机`dpi`匹配的`dpi`文件夹
- 比匹配`dpi`高的`dpi`文件夹
- `drawable-nodpi`
- 比匹配`dpi`低，但大于等于`1`的`dpi`文件夹
- `drawable`
- `drawable-ldpi`

## 3.4 `Bitmap.Config`
这是一个枚举类型，它用来描述每个像素是如何被保存的，它会影响图片的质量并决定能否表示透明/半透明颜色。
- `ALPHA_8`：每个像素点仅表示`alpha`的值，它不会存储任何颜色信息，占`8位`。
- `RGB_565`：每个像素用`5位R/6位G/5位G`来表示，占`16位`。
- `ARGB_8888`：每个像素分别用`8位`存储`ARGB`，占`32位`。
- `ARGB_4444`：和`8888`类似，只不过对于每个通道是使用`4位`表示，因此它的图片质量比较低，已经不推荐使用了。

## 3.5 实例分析
已经介绍完所需要掌握的基础知识，总结下来，`Bitmap`所占内存大小其实由两个因素决定：
- 读入内存中的宽高
- `ARGB`所占位宽

第一点的影响因素有：原始图片的宽高、手机内置的`dpi`、图片所放位置。
第二点的影响因素有：`Bitmap`所对应的`Bitmap.Config`配置。

下面我们几个例子，验证前面说的三种情况：
## 3.5.1 放在匹配的文件夹中
- 所用的手机分辨率为`720 * 1280`，它所匹配到的文件夹为`drawable-xhdpi`，我们先将一个原始大小为`48 * 48`的图片方在`drawable-xhdpi`文件夹中，并配置`Option`为`Bitmap.Config.RGB_565`：
```
    private void logWrapperImageView() {
        BitmapFactory.Options options = new BitmapFactory.Options();
        options.inPreferredConfig = Bitmap.Config.RGB_565;
        Bitmap bitmap = BitmapFactory.decodeResource(getResources(), R.drawable.drawable_test, options);
        Log.d("logWrapperImageView", "width=" + bitmap.getWidth() + ",height=" + bitmap.getHeight() + ",size=" + bitmap.getByteCount());
    }
```
这种情况下最终的结果为，虽然我们指出了需要用`RGB_565`，但是系统每个像素还是只用了一位：
```
width=48,height=48,size=2304 //RGB_565
width=48,height=48,size=9216 //ARGB_8888
width=48,height=48,size=2304 //ALPHA_8
```
- 所用的手机分辨率为`1080 * 1920`，它所匹配到的文件夹为`drawable-xxhdpi`，我们先将一个原始大小为`48 * 48`的图片方在`drawable-xxhdpi`文件夹中，并配置`Option`为`Bitmap.Config.RGB_565`，运行和上面一样的程序，得到的结果为：
```
width=48,height=48,size=2304 //RGB_565
```
接着改变它的`Options`：
```
width=48,height=48,size=9216 //ARGB_8888
width=48,height=48,size=2304 //ALPHA_8
```

### 3.5.2 放在不匹配，且不是`drawable-nodpi`的文件夹中
- 所用的手机分辨率为`720 * 1280`，把图片放在`drawable-xxhdpi`文件夹中：
```
width=32,height=32,size=4096 //RGB_565
```
改变它的`Options`：
```
width=32,height=32,size=4096 //ALPHA_8
width=32,height=32,size=4096 //ARGB_8888
```
- 所用的手机分辨率为`1080 * 1960`，把图片放在`drawable-xhdpi`文件夹中：
```
width=72,height=72,size=20736
```
改变它的`Options`
```
width=72,height=72,size=20736 //ALPHA_8
width=72,height=72,size=20736 //ARGB_8888
```

### 3.5.3 放在`drawable-nodpi`文件夹中
- 所用的手机分辨率为`720 * 1280`：
```
width=48,height=48,size=2304 //RGB_565
width=48,height=48,size=9216 //ARGB_8888
width=48,height=48,size=2304 //ALPHA_8
```
- 所用的手机分辨率为`1080 * 1960`：
```
width=48,height=48,size=2304 //RGB_565
width=48,height=48,size=9216 //ARGB_8888
width=48,height=48,size=2304 //ALPHA_8
```

### 3.5.4 小结
从上面的例子中，总结出几点：
- 把资源放在和它对应的`dpi`文件夹，和放在`drawable-nodpi`文件夹是相同的。
- 我们使用`RGB_565`时，实际占用的是一个字节。
- 当将资源文件放在别的文件夹时，无论选择哪种`Options`，都是采用占用`4字节`的方式。
- 当资源放在对应的`dpi`文件夹下和`drawable-nodpi`文件夹下时，长宽不进行缩放。
- 当资源放在除以上两类的文件夹下时，缩放的倍数为 `匹配文件夹density/所在文件夹density`，以上面`3.5.2`的`720 * 1280`手机为例，原始的图片的长宽为`48`，其匹配文件夹的`density`为`2`，所在文件夹的`density`为`3`，所以缩放倍数为`2/3`，因此，最后的长宽为`32`。
- `Bitmap`所占内存大小就等于它的长宽乘以每个像素所占位数。
