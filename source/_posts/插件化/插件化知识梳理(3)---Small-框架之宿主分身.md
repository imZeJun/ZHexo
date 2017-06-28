---
title: 插件化知识梳理(3) - Small 框架之宿主分身
date: 2017-06-03 22:44
categories : 插件化知识梳理
---
# 一、前言
在 [插件化知识梳理(1) - Small 框架之如何引入应用插件](http://www.jianshu.com/p/07f88d4924db)，[插件化知识梳理(2) - Small 框架之如何引入公共库插件](http://www.jianshu.com/p/f882c1f6b68b) 前两篇文章中，我们介绍了如何通过`Small`框架来实现应用插件及公共库插件，今天，我再来介绍一个新的知识点 - 宿主分身。

与应用插件和公共库插件不同，宿主分身会作为宿主的一部分，被编译到宿主当中，因此它不能被独立更新，但是它提供了一些插件所不具备的功能：
- 插件模块可以自由访问其中的代码和资源
- 预留特定的资源、代码和`AndroidManifest.xml`声明，让系统可以正常识别
- 预留稳定的资源、代码、第三方库，减少插件体积

正如前面介绍的，应用插件通过`app.xxx`，公共库插件通过`lib.xxx`的命名方式，让`Small`进行识别。而宿主分身则需要以`app+xxxx`的方式来命名。

除了以上介绍的可选情况，有一些代码是必须要放入到宿主分身中的：
- `AndroidManifest.xml`的声明
 - 所有权限，例如引入网络权限、读取存储等
 - 包含了某些特殊权限的`Activity`，如`process`，`configChanges`等
 - 任何的`provider/receiver/service`声明
- 资源
 - 转场动画
 - 通知栏图标、自定义视图
 - 桌面快捷方式图标

# 二、具体示例
## 2.1 创建应用插件模块
首先，我们创建一个新的插件模块，创建的方式和 [插件化知识梳理(1) - Small 框架之如何引入应用插件](http://www.jianshu.com/p/07f88d4924db) 中介绍的相同：
![](http://upload-images.jianshu.io/upload_images/1949836-14096966b5b70081.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
其中，`DetailActivity`是插件的入口，而以`Stub`开头的四个类分别为`Activity/Service/ContentProvider/BroadcastReceiver`，我们需要在宿主分身模块中对其进行声明。
```
public class StubActivity extends AppCompatActivity {

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_stub);
    }
}
```
```
public class StubBroadcastReceiver extends BroadcastReceiver {

    @Override
    public void onReceive(Context context, Intent intent) {
        Toast.makeText(context, "StubBroadcastReceiver, onReceive", Toast.LENGTH_SHORT).show();
    }
}
```
```
public class StubContentProvider extends ContentProvider {

    public static final Uri CONTENT_URI = Uri.parse("content://com.demo.small.app.detail.stub.provider");

    @Override
    public boolean onCreate() {
        return false;
    }

    @Nullable
    @Override
    public Cursor query(@NonNull Uri uri, @Nullable String[] strings, @Nullable String s, @Nullable String[] strings1, @Nullable String s1) {
        return null;
    }

    @Nullable
    @Override
    public String getType(@NonNull Uri uri) {
        return null;
    }

    @Nullable
    @Override
    public Uri insert(@NonNull Uri uri, @Nullable ContentValues contentValues) {
        Toast.makeText(getContext(), "StubContentProvider, insert", Toast.LENGTH_SHORT).show();
        return null;
    }

    @Override
    public int delete(@NonNull Uri uri, @Nullable String s, @Nullable String[] strings) {
        return 0;
    }

    @Override
    public int update(@NonNull Uri uri, @Nullable ContentValues contentValues, @Nullable String s, @Nullable String[] strings) {
        return 0;
    }
}
```
```
public class StubService extends Service {

    @Override
    public void onCreate() {
        super.onCreate();
    }

    @Override
    public int onStartCommand(Intent intent, int flags, int startId) {
        Toast.makeText(getApplicationContext(), "StubService, onStartCommand", Toast.LENGTH_SHORT).show();
        return super.onStartCommand(intent, flags, startId);
    }

    @Nullable
    @Override
    public IBinder onBind(Intent intent) {
        return null;
    }
}
```
为了方便验证，我们在回调函数中弹出`Toast`。
```
public class DetailActivity extends AppCompatActivity {

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_detail);
    }

    public void activity(View view) {
        Intent intent = new Intent(this, StubActivity.class);
        startActivity(intent);
    }

    public void service(View view) {
        Intent intent = new Intent(this, StubService.class);
        startService(intent);
    }

    public void contentProvider(View view) {
        ContentValues values = new ContentValues();
        getContentResolver().insert(StubContentProvider.CONTENT_URI, values);
    }

    public void broadcastReceiver(View view) {
        Intent intent = new Intent();
        intent.setAction("com.demo.small.action.stub.broadcast");
        sendBroadcast(intent);
    }
}
```
## 2.2 创建宿主分身模块
创建宿主分身模块时，需要选择`Android Library`，并且它的模块名需要为`app+xxx`的结构，这里，我们命名为`app+detail`
![](http://upload-images.jianshu.io/upload_images/1949836-cad62c3e7241e53e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
对于宿主分身模块，这里我们需要在`AndroidManifest.xml`文件中，添加必须要的声明：
```
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
    package="com.demo.small.appdetail">

    <application
        android:allowBackup="true"
        android:label="@string/app_name">

        <!-- Stub Activity -->
        <activity android:name="com.demo.small.app.detail.StubActivity" android:theme="@style/Theme.AppCompat"/>

        <!-- Stub Service -->
        <service android:name="com.demo.small.app.detail.StubService"/>

        <!-- Stub ContentProvider -->
        <provider
            android:authorities="com.demo.small.app.detail.stub.provider"
            android:name="com.demo.small.app.detail.StubContentProvider"/>

        <!-- Stub BroadcastReceiver -->
        <receiver android:name="com.demo.small.app.detail.StubBroadcastReceiver">
            <intent-filter>
                <action android:name="com.demo.small.action.stub.broadcast"/>
            </intent-filter>
        </receiver>

    </application>

</manifest>
```
## 2.3 修改宿主模块的 bundle.json 文件
对于宿主分身模块，不需要添加声明，我们所需要的只是对新增的应用插件模块`app.detail`添加声明：
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
    },
    {
      "uri": "detail",
      "pkg": "com.demo.small.app.detail"
    }
  ]
}
```
## 2.4 结果
![](http://upload-images.jianshu.io/upload_images/1949836-180594212c3b5260.gif?imageMogr2/auto-orient/strip)
通过分析最后生成的`APK`文件，可以发现，只有应用插件和公共库插件模块作为`so`被打包进入了`APK`中，而宿主分身模块并没有生成`so`。
![](http://upload-images.jianshu.io/upload_images/1949836-4f3d8d7267e335f8.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
而我们在宿主分身中的声明则被加入到了最后的`AndroidManifest.xml`文件中：
![](http://upload-images.jianshu.io/upload_images/1949836-e05404f3f149d699.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
