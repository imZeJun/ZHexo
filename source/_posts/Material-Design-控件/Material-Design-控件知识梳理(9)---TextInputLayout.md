---
title: Material Design 控件知识梳理(9) - TextInputLayout
date: 2017-04-18 22:05
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

# 一、概述
`TextInputLayout`通过对`EditText`进行包装，扩展了`EditText`的功能，今天，我们就来介绍一下和`TextInputLayout`相关的知识：
- 输入检查
- 输入计数
- 密码隐藏

# 二、`TextInputLayout`
## 2.1 基本用法
- 首先，导入`TextInputLayout`的依赖包：
```
compile 'com.android.support:design:25.3.1'
```
- 之后，我们需要将`TextInputLayout`作为`EditText`的父容器以实现相关的功能，例如下面这样，我们的布局中有两个`EditText`，都使用`TextInputLayout`把它们包裹起来。
```
<LinearLayout
    xmlns:android="http://schemas.android.com/apk/res/android"
    android:orientation="vertical"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:layout_margin="20dp">
    <android.support.design.widget.TextInputLayout
        android:layout_width="match_parent"
        android:layout_height="wrap_content">
        <EditText
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:hint="Username"/>
    </android.support.design.widget.TextInputLayout>
    <android.support.design.widget.TextInputLayout
        android:layout_width="match_parent"
        android:layout_height="wrap_content">
        <EditText
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:hint="Password"/>
    </android.support.design.widget.TextInputLayout>
</LinearLayout>
```
当`EditText`获得焦点的时候，`android:hint`所指定的字符串会以高亮的颜色显示在`EditText`的上方，而当`EditText`失去焦点时，`hint`会以灰显的方式显示在`EditText`中，这就是`TextInputLayout`最基本的使用，它让我们在输入的过程当中仍然可以获得当前`EditText`所关联的提示。

![input_1.gif](http://upload-images.jianshu.io/upload_images/1949836-5c95d85490ccd4c8.gif?imageMogr2/auto-orient/strip)


## 2.2 输入检查
除了在`EditText`上面的提示之外，`TextInputLayout`还支持在`EditText`下方显示提示，这种一般用于用户在输入过程中，输入了不符合要求的文本，用来给予用户错误提示。
```
private void checkError() {
        mPasswordEditText.addTextChangedListener(new TextWatcher() {

            @Override public void beforeTextChanged(CharSequence s, int start, int count, int after) {}

            @Override public void onTextChanged(CharSequence s, int start, int before, int count) {}

            @Override public void afterTextChanged(Editable s) {
                int len = s.length();
                if (len > MAX_PASSWORD_LEN) {
                    mPasswordTextInput.setError("Max Password len is " + MAX_PASSWORD_LEN);
                } else {
                    mPasswordTextInput.setErrorEnabled(false);
                }
            }
        });
}
```
效果如下图所示：
![](http://upload-images.jianshu.io/upload_images/1949836-5c3e2c0f66917a0a.gif?imageMogr2/auto-orient/strip)
在上面的例子中，我们通过监听`EditText`的输入，然后在某个条件被触发时，调用`TextInputLayout`的`setError`方法，以提示用户，与`setError`相关的方法有：
- `public void setError(@Nullable final CharSequence error)`
设置错误提示的文字，它会被显示在`EditText`的下方
- `public void setErrorTextAppearance(@StyleRes int resId)`
设置错误提示的文字颜色和大小
- `public void setErrorEnabled(boolean enabled)`
设置错误提示是否可用

## 2.3 输入计数
`TextInputLayout`还支持对于输入框内字符的实时统计，并在字符数超过阈值之后，改变输入框及提示文字的颜色。
```
private void checkCount() {
        mPasswordTextInput.setCounterEnabled(true);
        mPasswordTextInput.setCounterMaxLength(MAX_PASSWORD_LEN);
}
```
![](http://upload-images.jianshu.io/upload_images/1949836-ddc278e4e27edb3a.gif?imageMogr2/auto-orient/strip)
与输入计数有关的方法有：
- `public void setCounterEnabled(boolean enabled)`
设置计数功能是否可用
- `public void setCounterMaxLength(int maxLength)`
设置计数功能的阈值

## 2.4 输入时隐藏密码
相信大家都有见到过这样的输入框，在输入框的右边有一个开关，可以让用户来决定输入过程中的字符是否隐藏（以`*`号显示），`TextInputLayout`也提供了这样的功能：
`EditText`的`inputType`为`textPassword/textWebPassword/numberPassword`时，通过下面的设置：
```
mPasswordTextInput.setPasswordVisibilityToggleEnabled(true);
```
![](http://upload-images.jianshu.io/upload_images/1949836-2d8436840329afc4.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
同时，我们也可以通过上面的方法来定义切换的图标以及它的着色。
