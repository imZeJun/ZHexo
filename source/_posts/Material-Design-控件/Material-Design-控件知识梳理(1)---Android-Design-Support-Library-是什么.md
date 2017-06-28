---
title: Material Design 控件知识梳理(1) - Android Design Support Library 是什么
date: 2017-02-20 22:14
categories : Material Design 控件知识梳理
---
>[Material Design 控件知识梳理(1) - Android Design Support Library 是什么](http://www.jianshu.com/p/32b2638a1785)
[Material Design 控件知识梳理(2) - AppBarLayout & CollapsingToolbarLayout](http://www.jianshu.com/p/d4fd636d7c44)
[Material Design 控件知识梳理(3) - BottomSheet && BottomSheetDialog && BottomSheetDialogFragment](http://www.jianshu.com/p/2a5be29123e5)
[Material Design 控件知识梳理(4) - FloatingActionButton](http://www.jianshu.com/p/5a354e318019)
[Material Design 控件知识梳理(5) - DrawerLayout && NavigationView](http://www.jianshu.com/p/d70cfd724c7f)
[Material Design 控件知识梳理(6) - Snackbar](http://www.jianshu.com/p/6aea94f9ae2f)
[Material Design 控件知识梳理(7) - BottomNavigationBar](http://www.jianshu.com/p/22ec4fc1cb71)
[Material Design 控件知识梳理(8) - TabLayout](http://www.jianshu.com/p/5dd04fda13ab)
[Material Design 控件知识梳理(9) - TextInputLayout](http://www.jianshu.com/p/ed9642fc8634)

# 一、为什么需要`Support`包
由于应用除了会依赖`library`和`jar`包外，还需要依赖安卓系统本身的代码，也就是我们在`SDK`每个版本中看到的`android.jar`，这里面集成了`Android`的所有`API`，随着`SDK`版本的升级，高版本的`SDK`中会增加新的`API`，如果在低版本中要使用这些新增的`API`，那么只能将新增的`API`以依赖包的形式集成到需要使用高版本`API`的应用当中，也就是`support`包。

# 二、`Support`包的结构

![Paste_Image.png](http://upload-images.jianshu.io/upload_images/1949836-079345fe835d80b5.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
## 2.1 `V4`
在`Android Support Library 24.2.0`以前，`v4`包支持的最小`API`为4，而在之后的版本，移除了8及以下版本的支持，同时，将`v4`包拆分成了独立的5个包。
- `com.android.support:support-compat:24.2.1`
说明：兼容一些`framework API`，例如`Context.getDrawable`和`View.performAccessibilityAction`。
- `com.android.support:support-core-utils:24.2.1`
说明：提供一些核心的工具类，如`AsyncTaskLoader`和`PermissionChecker`。
- `com.android.support:support-core-ui:24.2.1`
说明：提供一系列核心的`UI`，例如`ViewPager`、`NestedScrollView`和`DrawerLayout `。
- `com.android.support:support-media-compat:24.2.1`
说明：媒体`android.media`兼容库，包括`MediaBrowser`和`MediaSession`。
- `com.android.support:support-fragment:24.2.1`
说明：依赖了其它4个子库，一旦导入这个包就会导入其余的库。

依赖关系：


![Paste_Image.png](http://upload-images.jianshu.io/upload_images/1949836-ab9804972696c062.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 2.2 `V7`
`V7`也包含多个独立包，从`24.2.0`开始，将`V7`支持的最低版本升为9。
- `com.android.support:appcompat-v7:24.2.1`
说明：这个支持对`ActionBar`接口的设计模式，`Material Design`接口的实现等，核心类包括`ActionBar`、`AppCompactActivity`、`AppCompactDialog`、`ShareActionProvider`等。
- `com.android.support:cardview-v7:24.2.1`
说明：`CardView`控件
- `com.android.support:gridlayout-v7:24.2.1`
说明：`GridLayout`布局
- `com.android.support:mediarouter-v7:24.2.1`
说明：用于设备间音频、视频交换显示的`support`包。
- `com.android.support:palette-v7:24.2.1`
说明：提取图片中的主题色
- `com.android.support:recyclerview-v7:24.2.1`
说明：`RecyclerView`
- `com.android.support:preference-v7:24.2.1`
说明：支持控件存储配置数据的，例如`CheckBoxPreference`和`ListPreference`。

## 2.3 `V8`
用于渲染脚本的`support`包

## 2.4 `V13`
为`API`为13或以上的`Fragment`提供更多特性的支持。

##2.5 `com.android.support:multidex:1.0.0`
用于使用多`Dex`技术编译`APP`，当一个应用的方法数大于65536时，需要使用`multidex`配置。

## 2.6 `com.android.support:support-annotations:24.2.1`
支持注解。

## 2.7 `com.android.support:design:24.2.1`
用于支持`Design Patterns`的`Support`包，它提供了`Material Design`设计风格的控件：
- `FloatingActionButton`
- `Snackbar`
- `TextInputLayout`
- `TabLayout`
- `AppBarLayout`
- `CollapsingToolbarLayout`
- `CoordinatorLayout`
- `NavigationView`

## 2.8 `com.android.support:customtabs:24.2.1`
在应用中添加和管理`Custom Tabs`的`support`包，提供了一种新的打开网页的方式。

## 2.9 `com.android.support:percent:24.2.1`
支持百分比布局的`support`包。
