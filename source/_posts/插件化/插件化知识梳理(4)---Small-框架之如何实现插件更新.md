---
title: 插件化知识梳理(4) - Small 框架之如何实现插件更新
date: 2017-06-05 22:50
categories : 插件化知识梳理
---
# 一、前言
通过前面三节：

[插件化知识梳理(1) - Small 框架之如何引入应用插件](http://www.jianshu.com/p/07f88d4924db)
[插件化知识梳理(2) - Small 框架之如何引入公共库插件](http://www.jianshu.com/p/f882c1f6b68b)
[插件化知识梳理(3) - Small 框架之宿主分身](http://www.jianshu.com/p/2162a4ce4560)

相信大家已经对`Small`的使用有了一个基本的认识，今天讲解另一个比较重要的知识点，插件更新。
# 二、示例
原始的工程包括两个部分：
- 宿主模块 - `app`
- 应用插件模块 - `app.upgrade`
- 原始路由文件 - `bundle.json`

如果要进行插件更新，那么我们需要：
- 新的路由文件 - `bundle.json`，通过该文件，我们获取需要更新的插件的包名以及获取插件的路径。
- 新的插件文件

`Github`地址为：**[SmallUpdate](https://github.com/imZeJun/SmallUpdate)**

# 2.1 准备原始的工程
这里的步骤和 [插件化知识梳理(1) - Small 框架之如何引入应用插件](http://www.jianshu.com/p/07f88d4924db) 相同，就是新建一个宿主模块`app`和一个应用插件模块`app.upgrade`，从宿主模块中可以跳转到应用插件模块的`Activity`，该`Activity`的布局如下：
```
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout
    xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    tools:context="com.demo.small.update.app.upgrade.UpgradeActivity">

    <TextView
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="Version 0!"/>

</LinearLayout>
```
整个工程结构如下：
![](http://upload-images.jianshu.io/upload_images/1949836-35ba36fc126b9a73.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
## 2.2 准备新的插件
**(1) 新的路由文件**
```
{
  "version": "1.0.0",
  "bundles": [
    {
      "uri": "upgrade",
      "pkg": "com.demo.small.update.app.upgrade"
    }
  ],
  "updates": [
    {
      "pkg": "com.demo.small.update.app.upgrade",
      "url": "libcom_demo_small_update_app_upgrade.so"
    }
  ]
}
```
这里的区别就是增加了`updates`字段，用于标志需要更新的插件包名以及新插件的路径，之后，把这个文件`push`到手机根目录下的`Small`文件夹中。

**(2) 新的插件模块**

我们修改`app.upgrade`的`build.gradle`的版本号为`2.0`：
```
android {
    defaultConfig {
        //....
        versionCode 2
        versionName "2.0"
    }
}
```
并修改`Activity`的布局文件，将显示的字符串从`Version 0!`改为`Version 2!`：
```
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout
    xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    tools:context="com.demo.small.update.app.upgrade.UpgradeActivity">

    <TextView
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="Version 2!"/> <!-- 区别，旧的插件版本为 Version 0! -->

</LinearLayout>
```
用于验证我们的插件是否更新。之后，通过命令编译出新的插件文件`so`，把它`push`到手机根目录下的`Small`文件夹中。

最终，在`sdcard/Small`目录保存了以上两部分的更新信息：
![](http://upload-images.jianshu.io/upload_images/1949836-b156133f96f9ee76.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 2.3 插件更新
以上准备工作做好之后，就开始真正的更新步骤了，我们把更新的操作都封装在`SmallManager`中，外部只需要调用`requestUpgrade()`方法，就可以实现插件的更新：
```
public class SmallManager {

    private static final String TAG = "SmallManager";
    private static final String PATCH_PATH = Environment.getExternalStorageDirectory().toString() + "/Small/bundle.json";

    private Handler mWorkerHandler;
    private Handler mUiHandler;

    private SmallManager() {
        HandlerThread workThread = new HandlerThread("workThread");
        workThread.start();
        mWorkerHandler = new MHandler(workThread.getLooper());
        mUiHandler = new MHandler(Looper.getMainLooper());
    }

    public void requestUpgrade() {
        mWorkerHandler.sendEmptyMessage(MHandler.MSG_REQ_UPGRADE);
    }

    private void onUpgraded(boolean success) {
        Log.d(TAG, "onUpgraded, success=" + success);
    }

    private void upgrade() {
        Log.d(TAG, "upgrade");
        try {
            //1.获取插件更新信息。
            UpgradeInfo upgradeInfo = getUpgradeInfo();
            if (upgradeInfo != null) {
                if (upgradeInfo.manifest != null) {
                    if (!Small.updateManifest(upgradeInfo.manifest, false)) {
                        mUiHandler.sendMessage(mUiHandler.obtainMessage(MHandler.MSG_UPGRADE_FINISHED, false));
                        return;
                    }
                }
                List<UpdateInfo> updates = upgradeInfo.updates;
                for (UpdateInfo update : updates) {
                    //2.获得对应包名的Bundle
                    Bundle bundle = Small.getBundle(update.packageName);
                    //3.利用更新信息来更新对应包名的Bundle。
                    boolean result = upgradeBundle(bundle, update);
                    Log.d(TAG, "pkg=" + update.packageName + ",result=" + result);
                }
                mUiHandler.sendMessage(mUiHandler.obtainMessage(MHandler.MSG_UPGRADE_FINISHED, true));
                return;
            }
        } catch (Exception e) {
            e.printStackTrace();
        }
        mUiHandler.sendMessage(mUiHandler.obtainMessage(MHandler.MSG_UPGRADE_FINISHED, false));
    }

    private UpgradeInfo getUpgradeInfo() {
        try {
            File patchFile = new File(PATCH_PATH);
            //获得新的bundle.json，这实际情况下可以通过访问网络进行下载的方式实现，这里我们用读取外部存储的方式代替，原理是一样的。
            FileInputStream fileInputStream = new FileInputStream(patchFile);
            StringBuilder sb = new StringBuilder();
            byte[] buffer = new byte[1024];
            int length;
            while ((length = fileInputStream.read(buffer)) != -1) {
                sb.append(new String(buffer, 0, length));
            }
            JSONObject jo = new JSONObject(sb.toString());
            //manifest部分的更新。
            JSONObject mf = jo.has("manifest") ? jo.getJSONObject("manifest") : null;
            //每个插件部分的更新，包含了包名以及新的插件so路径。
            JSONArray updateJson = jo.getJSONArray("updates");
            int N = updateJson.length();
            List<UpdateInfo> updates = new ArrayList<>(N);
            for (int i = 0; i < N; i++) {
                JSONObject o = updateJson.getJSONObject(i);
                UpdateInfo info = new UpdateInfo();
                info.packageName = o.getString("pkg");
                info.soPath = o.getString("url");
                updates.add(info);
            }
            //将更新的信息封装起来，用于下一步的更新操作。
            UpgradeInfo ui = new UpgradeInfo();
            ui.manifest = mf;
            ui.updates = updates;
            return ui;
        } catch (Exception e) {
            e.printStackTrace();
        }
        return null;
    }

    private boolean upgradeBundle(Bundle origin, UpdateInfo updateInfo) {
        try {
            //获得Bundle的PatchFile。
            File file = origin.getPatchFile();
            //获取新的so，也就是新的插件，实际情况下可以通过访问网络进行下载的方式实现，这里我们用读取外部存储的方式代替，原理是一样的。
            File updateFile = new File(Environment.getExternalStorageDirectory().toString() + "/Small/" + updateInfo.soPath);
            FileInputStream updateFileInput = new FileInputStream(updateFile);
            OutputStream outputStream = new FileOutputStream(file);
            byte[] buffer = new byte[1024];
            int length;
            //将更新的插件so，写入到Bundle的PatchFile。
            while ((length = updateFileInput.read(buffer)) != -1) {
                outputStream.write(buffer, 0, length);
            }
            outputStream.flush();
            outputStream.close();
            updateFileInput.close();
            //调用Bundle的upgrade方法。
            origin.upgrade();
            return true;
        } catch (Exception e) {
            e.printStackTrace();
        }
        return false;
    }

    private static class UpdateInfo {
        private String packageName;
        private String soPath;
    }

    private static class UpgradeInfo {
        private JSONObject manifest;
        private List<UpdateInfo> updates;
    }

    private class MHandler extends Handler {

        private static final int MSG_REQ_UPGRADE = 0;
        private static final int MSG_UPGRADE_FINISHED = 1;

        public MHandler(Looper looper) {
            super(looper);
        }

        @Override
        public void handleMessage(Message msg) {
            super.handleMessage(msg);
            switch (msg.what) {
                case MSG_REQ_UPGRADE:
                    upgrade();
                    break;
                case MSG_UPGRADE_FINISHED:
                    onUpgraded((boolean) msg.obj);
                    break;
                default:
                    break;
            }
        }
    }

    public static SmallManager getInstance() {
        return Holder.INSTANCE;
    }

    private static class Holder {
        private static SmallManager INSTANCE = new SmallManager();
    }
}
```
通过`Handler + HandlerThread`的方式，保证了耗时的更新操作`upgrade()`在子线程当中执行，整个更新操作分为以下几步，简要地解释一下：

**(1) 获取更新信息**

通过`getUpgradeInfo`方法，我们把所有需要更新的信息保存在`UpgradeInfo`结构中，这里我们从存储中读取了前面`push`到手机中的`bundle.json`文件，通过解析其中的`updates`阶段，得到更新的信息，也就是**包名**以及**新插件的路径**。

这里，我们是从存储中读取的，实际情况下，我们可以将这份文件放在服务器上，然后去下载。

**(2) 更新插件**

在`(1)`中，我们已经知道了要更新的插件的包名，现在，我们通过`Small`提供的这个方法，获得与该包名对应的`Bundle`，而`upgradeBundle`则进行真正的更新操作。
```
Bundle bundle = Small.getBundle(update.packageName);
```
在第`(1)`步中，我们获取到了新的插件所处的位置，然后现在我们通过该字段来获取插件，之后将该插件的内容通过输入输出流，写入到`Bundle`的`PatchFile`当中，写入完毕之后，再通过`Bundle`的`upgrade`方法进行更新。

与上面类似，这里也是从本地读取插件，实际情况下应当将该插件放在服务器上，需要更新的时候再去下载。

其实一路下来，最核心的部分就是先根据包名获取到对应的`Bundle`，然后将新的插件通过文件流的形式写入到`Bundle`的`PatchFile`中，最终调用该`Bundle`的`upgrade()`方法，至于内部是如何更新的，`Small`已经帮我们处理好了，我们完全不需要去操心。

## 2.4 演示效果
![](http://upload-images.jianshu.io/upload_images/1949836-4aad035018c530b1.gif?imageMogr2/auto-orient/strip)
