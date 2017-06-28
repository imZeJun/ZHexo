---
title: Framework 源码解析知识梳理(5) - startService 源码分析
date: 2017-06-16 23:01
categories : Framework 源码解析知识梳理
---
# 一、前言
最近在看关于插件化的知识，遇到了如何实现`Service`插件化的问题，因此，先学习一下`Service`内部的实现原理，这里面会涉及到应用进程和`ActivityManagerService`的通信，建议大家先阅读一下之前的这篇文章 [Framework 源码解析知识梳理(1) - 应用进程与 AMS 的通信实现](http://www.jianshu.com/p/f26a76e19a2d)，整个`Service`的启动过程离不开和`AMS`的通信，从整个宏观上来看，它的模型如下，是不是和我们之前分析的通信过程一模一样：
![](http://upload-images.jianshu.io/upload_images/1949836-32a15a52b33012d7.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
# 二、源码分析
当我们在`Activity/Service/Application`中，通过下面这个方法启动`Service`时：
```
public ComponentName startService(Intent service)
```
与之前分析过的很多源码类似，最终会调用到`ContextImpl`的同名方法当中，而该方法又会调用内部的`startServiceCommon(Intent, UserHandle)`：
![](http://upload-images.jianshu.io/upload_images/1949836-62cfa3cd3380a609.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
在该方法中，主要做了两件事：
- 校验`Intent`的合法性，对于`Android 5.0`以后，我们要求`Intent`中需要制定`Package`和`Component`中至少一个，否则会抛出异常。
- 通过`Binder`通信调用到`ActivityManagerService`的对应方法，关于调用的详细过程可以参考 [Framework 源码解析知识梳理(1) - 应用进程与 AMS 的通信实现](http://www.jianshu.com/p/f26a76e19a2d) 中对于应用进程到`ActivityManagerService`的通信部分分析。

`ActivityManagerService`中的实现为：
![](http://upload-images.jianshu.io/upload_images/1949836-18ff48b6a33fc9a7.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
这里面，通过`mServices`变量来去实现`Service`的启动，而`mServices`的类型为`ActiveServices`，它是`ActivityManagerService`中负责处理`Service`相关业务的类，无论我们是通过`start`还是`bind`方式启动的`Service`，都是通过它来实现的，它也被称为“应用服务”的管理类。

从通过`ActiveServices`调用`startServiceLocked`，到`Service`真正被启动的过程如下图所示，下面，我们就来一一分析每一个阶段究竟做了什么：
![](http://upload-images.jianshu.io/upload_images/1949836-786654e5dffcab63.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

我们首先来看入口函数`startServiceLocked`的内部实现：
```
    ComponentName startServiceLocked(IApplicationThread caller, Intent service, String resolvedType,
            int callingPid, int callingUid, String callingPackage, final int userId)
            throws TransactionTooLargeException {

        final boolean callerFg;
        //caller是调用者进程在AMS的远程代理对象，类型为ApplicationThreadProxy。
        if (caller != null) {
            //获得调用者的进程信息。
            final ProcessRecord callerApp = mAm.getRecordForAppLocked(caller);
            if (callerApp == null) {
                throw new SecurityException(
                        "Unable to find app for caller " + caller
                        + " (pid=" + Binder.getCallingPid()
                        + ") when starting service " + service);
            }
            callerFg = callerApp.setSchedGroup != ProcessList.SCHED_GROUP_BACKGROUND;
        } else {
            callerFg = true;
        }

        //通过传递的信息，解析出要启动的Service，封装在ServiceLookupResult中。
        ServiceLookupResult res =
            retrieveServiceLocked(service, resolvedType, callingPackage,
                    callingPid, callingUid, userId, true, callerFg, false);
        if (res == null) {
            return null;
        }
        if (res.record == null) {
            return new ComponentName("!", res.permission != null
                    ? res.permission : "private to package");
        }
        //对于每个被启动的Service，它们在AMS端的数据结构为ServiceRecord。
        ServiceRecord r = res.record;

        if (!mAm.mUserController.exists(r.userId)) {
            Slog.w(TAG, "Trying to start service with non-existent user! " + r.userId);
            return null;
        }
        //判断是否允许启动该服务。
        if (!r.startRequested) {
            final long token = Binder.clearCallingIdentity();
            try {
                final int allowed = mAm.checkAllowBackgroundLocked(
                        r.appInfo.uid, r.packageName, callingPid, true);
                if (allowed != ActivityManager.APP_START_MODE_NORMAL) {
                    Slog.w(TAG, "Background start not allowed: service "
                            + service + " to " + r.name.flattenToShortString()
                            + " from pid=" + callingPid + " uid=" + callingUid
                            + " pkg=" + callingPackage);
                    return null;
                }
            } finally {
                Binder.restoreCallingIdentity(token);
            }
        }
        //是否需要配置相应的权限。
        NeededUriGrants neededGrants = mAm.checkGrantUriPermissionFromIntentLocked(
                callingUid, r.packageName, service, service.getFlags(), null, r.userId);

        if (Build.PERMISSIONS_REVIEW_REQUIRED) {
            if (!requestStartTargetPermissionsReviewIfNeededLocked(r, callingPackage,
                    callingUid, service, callerFg, userId)) {
                return null;
            }
        }
        //如果该Service正在等待被重新启动，那么移除它。
        if (unscheduleServiceRestartLocked(r, callingUid, false)) {
            if (DEBUG_SERVICE) Slog.v(TAG_SERVICE, "START SERVICE WHILE RESTART PENDING: " + r);
        }
        //给ServiceRecord添加必要的信息。
        r.lastActivity = SystemClock.uptimeMillis();
        r.startRequested = true;
        r.delayedStop = false;
        r.pendingStarts.add(new ServiceRecord.StartItem(r, false, r.makeNextStartId(),
                service, neededGrants));
        //其它的一些逻辑，与第一次启动无关，就先不分析了。
        return startServiceInnerLocked(smap, service, r, callerFg, addToStarting);
    }
```
简单地来说，就是根据`Intent`查找需要启动的`Service`，封装成`ServiceRecord`对象，初始化其中的关键变量，在这一过程中，加入了一些必要的逻辑判断，最终调用了`startServiceInnerLocked`方法。
```
    ComponentName startServiceInnerLocked(ServiceMap smap, Intent service, ServiceRecord r,
            boolean callerFg, boolean addToStarting) throws TransactionTooLargeException {
        //...
        String error = bringUpServiceLocked(r, service.getFlags(), callerFg, false, false);
        //...
        return r.name;
    }
```
这里面的逻辑我们先忽略掉一部分，只需要知道在正常情况下它会去调用`bringUpServiceLocked`来启动`Service`：
```
    private final String bringUpServiceLocked(ServiceRecord r, int intentFlags, boolean execInFg,
            boolean whileRestarting) throws TransactionTooLargeException {
 
        //如果该Service已经启动。
        if (r.app != null && r.app.thread != null) {
            sendServiceArgsLocked(r, execInFg, false);
            return null;
        }
        //如果正在等待被重新启动，那么什么也不做。
        if (!whileRestarting && r.restartDelay > 0) {
            return null;
        }

        if (DEBUG_SERVICE) Slog.v(TAG_SERVICE, "Bringing up " + r + " " + r.intent);

        //清除等待被重新启动的状态。
        if (mRestartingServices.remove(r)) {
            r.resetRestartCounter();
            clearRestartingIfNeededLocked(r);
        }

        //因为我们马上就要启动该Service，因此去掉它的延时属性。
        if (r.delayed) {
            if (DEBUG_DELAYED_STARTS) Slog.v(TAG_SERVICE, "REM FR DELAY LIST (bring up): " + r);
            getServiceMap(r.userId).mDelayedStartList.remove(r);
            r.delayed = false;
        }

        //如果该Service所属的用户没有启动，那么调用 bringDownServiceLocked 方法。
        if (mAm.mStartedUsers.get(r.userId) == null) {
            String msg = "Unable to launch app "
                    + r.appInfo.packageName + "/"
                    + r.appInfo.uid + " for service "
                    + r.intent.getIntent() + ": user " + r.userId + " is stopped";
            Slog.w(TAG, msg);
            bringDownServiceLocked(r);
            return msg;
        }

        try {
            AppGlobals.getPackageManager().setPackageStoppedState(
                    r.packageName, false, r.userId);
        } catch (RemoteException e) {
        } catch (IllegalArgumentException e) {
            Slog.w(TAG, "Failed trying to unstop package "
                    + r.packageName + ": " + e);
        }

        final boolean isolated = (r.serviceInfo.flags&ServiceInfo.FLAG_ISOLATED_PROCESS) != 0;
        final String procName = r.processName;
        ProcessRecord app;
        //如果不是运行在独立的进程。
        if (!isolated) {
            app = mAm.getProcessRecordLocked(procName, r.appInfo.uid, false);
            if (DEBUG_MU) Slog.v(TAG_MU, "bringUpServiceLocked: appInfo.uid=" + r.appInfo.uid
                        + " app=" + app);
            //如果该进程已经启动，那么调用realStartServiceLocked方法。
            if (app != null && app.thread != null) {
                try {
                    app.addPackage(r.appInfo.packageName, r.appInfo.versionCode, mAm.mProcessStats);
                    //在Service所属进程已经启动的情况下调用的方法。
                    realStartServiceLocked(r, app, execInFg);
                    return null;
                } catch (TransactionTooLargeException e) {
                    throw e;
                } catch (RemoteException e) {
                    Slog.w(TAG, "Exception when starting service " + r.shortName, e);
                }
            }
        } else {
            app = r.isolatedProc;
        }

        //如果该Service所对应的进程没有启动，那么首先启动该进程。
        if (app == null) {
            if ((app=mAm.startProcessLocked(procName, r.appInfo, true, intentFlags,
                    "service", r.name, false, isolated, false)) == null) {
                String msg = "Unable to launch app "
                        + r.appInfo.packageName + "/"
                        + r.appInfo.uid + " for service "
                        + r.intent.getIntent() + ": process is bad";
                Slog.w(TAG, msg);
                bringDownServiceLocked(r);
                return msg;
            }
            if (isolated) {
                r.isolatedProc = app;
            }
        }
        //将该ServiceRecord加入到等待的集合当中，等到新的进程启动之后，再去启动它。
        if (!mPendingServices.contains(r)) {
            mPendingServices.add(r);
        }

        if (r.delayedStop) {
            r.delayedStop = false;
            if (r.startRequested) {
                if (DEBUG_DELAYED_STARTS) Slog.v(TAG_SERVICE,
                        "Applying delayed stop (in bring up): " + r);
                stopServiceLocked(r);
            }
        }

        return null;
    }
```
`bringUpServiceLocked`的逻辑就比较复杂了，它会根据目标`Service`及其所属进程的状态，走向不同的分支：
- 进程已经存在，并且目标`Service`已经启动：`sendServiceArgsLocked`
- 进程已经存在，但是目标`Service`没有启动：`realStartServiceLocked`
- 进程不存在：`startProcessLocked`，并且将`ServiceRecord`加入到`mPendingServices`中，等待进程启动之后再去启动该`Service`。

下面，我们就来一起分析一下这三种情况。
## 2.1 进程已经存在，并且目标 Service 已经启动
首先来当进程已经存在，且目标`Service`已经启动时所调用的`sendServiceArgsLocked`方法：
```
    private final void sendServiceArgsLocked(ServiceRecord r, boolean execInFg,
            boolean oomAdjusted) throws TransactionTooLargeException {
        final int N = r.pendingStarts.size();
        if (N == 0) {
            return;
        }

        while (r.pendingStarts.size() > 0) {
            Exception caughtException = null;
            ServiceRecord.StartItem si;
            try {
                si = r.pendingStarts.remove(0);
                if (DEBUG_SERVICE) Slog.v(TAG_SERVICE, "Sending arguments to: "
                        + r + " " + r.intent + " args=" + si.intent);
                if (si.intent == null && N > 1) {
                    // If somehow we got a dummy null intent in the middle,
                    // then skip it.  DO NOT skip a null intent when it is
                    // the only one in the list -- this is to support the
                    // onStartCommand(null) case.
                    continue;
                }
                si.deliveredTime = SystemClock.uptimeMillis();
                r.deliveredStarts.add(si);
                si.deliveryCount++;
                if (si.neededGrants != null) {
                    mAm.grantUriPermissionUncheckedFromIntentLocked(si.neededGrants,
                            si.getUriPermissionsLocked());
                }
                bumpServiceExecutingLocked(r, execInFg, "start");
                if (!oomAdjusted) {
                    oomAdjusted = true;
                    mAm.updateOomAdjLocked(r.app);
                }
                int flags = 0;
                if (si.deliveryCount > 1) {
                    flags |= Service.START_FLAG_RETRY;
                }
                if (si.doneExecutingCount > 0) {
                    flags |= Service.START_FLAG_REDELIVERY;
                }
                r.app.thread.scheduleServiceArgs(r, si.taskRemoved, si.id, flags, si.intent);
            } catch (TransactionTooLargeException e) {
                if (DEBUG_SERVICE) Slog.v(TAG_SERVICE, "Transaction too large: intent="
                        + si.intent);
                caughtException = e;
            } catch (RemoteException e) {
                // Remote process gone...  we'll let the normal cleanup take care of this.
                if (DEBUG_SERVICE) Slog.v(TAG_SERVICE, "Crashed while sending args: " + r);
                caughtException = e;
            } catch (Exception e) {
                Slog.w(TAG, "Unexpected exception", e);
                caughtException = e;
            }

            if (caughtException != null) {
                // Keep nesting count correct
                final boolean inDestroying = mDestroyingServices.contains(r);
                serviceDoneExecutingLocked(r, inDestroying, inDestroying);
                if (caughtException instanceof TransactionTooLargeException) {
                    throw (TransactionTooLargeException)caughtException;
                }
                break;
            }
        }
    }
```
这里面最关键的就是调用了`r.app.thread`的`scheduleServiceArgs`方法，它其实就是`Service`所属进程的`ApplicationThread`对象在`AMS`端的一个远程代理对象，也就是`ApplicationThreadProxy`，通过`binder`通信，它最终会回调到`Service`所属进程的`ApplicationThread`的`scheduleServiceArgs`中：
```
    private class ApplicationThread extends ApplicationThreadNative {

        public final void scheduleServiceArgs(IBinder token, boolean taskRemoved, int startId,
            int flags ,Intent args) {
            ServiceArgsData s = new ServiceArgsData();
            s.token = token;
            s.taskRemoved = taskRemoved;
            s.startId = startId;
            s.flags = flags;
            s.args = args;
            sendMessage(H.SERVICE_ARGS, s);
        }
   }
```
`sendMessage`函数会通过`ActivityThread`内部的`mH`对象发送消息到主线程当中，`mH`实际上是一个`Handler`的子类，在它的`handleMessage`回调中，处理`H.SERVICE_ARGS`这条消息：
![](http://upload-images.jianshu.io/upload_images/1949836-16ec590cd0223326.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
在`handleServiceArgs`方法，就会从`mServices`去查找对应的`Service`，调用我们熟悉的`onStartCommand`方法，至于`Service`对象是如何被加入到`mServices`中的，我们在`2.2`节中进行分析。
```
    private void handleServiceArgs(ServiceArgsData data) {
        Service s = mServices.get(data.token);
        if (s != null) {
            try {
                if (data.args != null) {
                    data.args.setExtrasClassLoader(s.getClassLoader());
                    data.args.prepareToEnterProcess();
                }
                int res;
                if (!data.taskRemoved) {
                    //调用 onStartCommand 方法。
                    res = s.onStartCommand(data.args, data.flags, data.startId);
                } else {
                    s.onTaskRemoved(data.args);
                    res = Service.START_TASK_REMOVED_COMPLETE;
                }

                QueuedWork.waitToFinish();

                try {
                    //通知AMS启动完成。
                    ActivityManagerNative.getDefault().serviceDoneExecuting(
                            data.token, SERVICE_DONE_EXECUTING_START, data.startId, res);
                } catch (RemoteException e) {
                    // nothing to do.
                }
                ensureJitEnabled();
            } catch (Exception e) {
                if (!mInstrumentation.onException(s, e)) {
                    throw new RuntimeException(
                            "Unable to start service " + s
                            + " with " + data.args + ": " + e.toString(), e);
                }
            }
        }
    }
```
这里面，我们会取得`onStartCommand`的返回值，再通过`Binder`将该返回值传回到`AMS`端，在`AMS`的`mServices`的`serviceDoneExecutingLocked`中，会根据该返回值来修改`ServiceRecord`的属性，这也就是我们常说的`onStartCommand`方法的返回值，会影响到该`Services`之后的一些行为的原因，关于这些返回值之间的差别，可以查看`Service.java`的注释，也可以查看网上的一些教程：
![](http://upload-images.jianshu.io/upload_images/1949836-f980431008ca981f.png)

## 2.2 进程已经存在，但是 Service 没有启动
下面，我们来看`Service`没有启动的情况，这里会调用`realStartServiceLocked`方法：
```
    private final void realStartServiceLocked(ServiceRecord r,
            ProcessRecord app, boolean execInFg) throws RemoteException {
        if (app.thread == null) {
            throw new RemoteException();
        }
        if (DEBUG_MU)
            Slog.v(TAG_MU, "realStartServiceLocked, ServiceRecord.uid = " + r.appInfo.uid
                    + ", ProcessRecord.uid = " + app.uid);
        r.app = app;
        r.restartTime = r.lastActivity = SystemClock.uptimeMillis();

        final boolean newService = app.services.add(r);
        bumpServiceExecutingLocked(r, execInFg, "create");
        //更新Service所在进程的oom_adj值。
        mAm.updateLruProcessLocked(app, false, null);
        mAm.updateOomAdjLocked();

        boolean created = false;
        try {
            if (LOG_SERVICE_START_STOP) {
                String nameTerm;
                int lastPeriod = r.shortName.lastIndexOf('.');
                nameTerm = lastPeriod >= 0 ? r.shortName.substring(lastPeriod) : r.shortName;
                EventLogTags.writeAmCreateService(
                        r.userId, System.identityHashCode(r), nameTerm, r.app.uid, r.app.pid);
            }
            synchronized (r.stats.getBatteryStats()) {
                r.stats.startLaunchedLocked();
            }
            mAm.ensurePackageDexOpt(r.serviceInfo.packageName);
            app.forceProcessStateUpTo(ActivityManager.PROCESS_STATE_SERVICE);
            //通知应用端创建Service对象。
            app.thread.scheduleCreateService(r, r.serviceInfo,
                    mAm.compatibilityInfoForPackageLocked(r.serviceInfo.applicationInfo),
                    app.repProcState);
            r.postNotification();
            created = true;
        } catch (DeadObjectException e) {
            Slog.w(TAG, "Application dead when creating service " + r);
            mAm.appDiedLocked(app);
            throw e;
        } finally {
            if (!created) {
                final boolean inDestroying = mDestroyingServices.contains(r);
                serviceDoneExecutingLocked(r, inDestroying, inDestroying);
                if (newService) {
                    app.services.remove(r);
                    r.app = null;
                }
                if (!inDestroying) {
                    scheduleServiceRestartLocked(r, false);
                }
            }
        }
        //这里面会遍历 ServiceRecord.bindings 列表，因为我们这里是用startService方式启动的，因此该列表为空，什么也不会调用。
        requestServiceBindingsLocked(r, execInFg);

        updateServiceClientActivitiesLocked(app, null, true);

        if (r.startRequested && r.callStart && r.pendingStarts.size() == 0) {
            r.pendingStarts.add(new ServiceRecord.StartItem(r, false, r.makeNextStartId(),
                    null, null));
        }
        //这个在第一种情况中分析过了，会调用 onStartCommand 方法。
        sendServiceArgsLocked(r, execInFg, true);

        if (r.delayed) {
            if (DEBUG_DELAYED_STARTS) Slog.v(TAG_SERVICE, "REM FR DELAY LIST (new proc): " + r);
            getServiceMap(r.userId).mDelayedStartList.remove(r);
            r.delayed = false;
        }

        if (r.delayedStop) {
            // Oh and hey we've already been asked to stop!
            r.delayedStop = false;
            if (r.startRequested) {
                if (DEBUG_DELAYED_STARTS) Slog.v(TAG_SERVICE,
                        "Applying delayed stop (from start): " + r);
                stopServiceLocked(r);
            }
        }
    }
```
这段逻辑，有三个地方需要注意：
- `app.thread.scheduleCreateService`：通过这里会在应用进程创建`Service`对象
- `requestServiceBindingsLocked `：如果是通过`bindService`方式启动的，那么会去调用它的`onBind`方法。
- `sendServiceArgsLocked`：正如`2.1`节中所分析，这里最终会触发应用进程的`Service`的`onStartCommand`方法被调用。

第二点由于我们今天分析的是`startService`的方式，所以不用在意，而第三点我们之前已经分析过了。所以我们主要关注`app.thread.scheduleCreateService`这一句，和`2.1`中的过程类似，它会回调到`H`中的下面这个消息处理分支当中：
![](http://upload-images.jianshu.io/upload_images/1949836-388eab689362fc69.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
具体的`handleCreateService`的处理逻辑如下图所示：
```
    private void handleCreateService(CreateServiceData data) {
        // If we are getting ready to gc after going to the background, well
        // we are back active so skip it.
        unscheduleGcIdler();

        LoadedApk packageInfo = getPackageInfoNoCheck(
                data.info.applicationInfo, data.compatInfo);
        Service service = null;
        try {
            //根据Service的名字，动态的加载该Service类。
            java.lang.ClassLoader cl = packageInfo.getClassLoader();
            service = (Service) cl.loadClass(data.info.name).newInstance();
        } catch (Exception e) {
            if (!mInstrumentation.onException(service, e)) {
                throw new RuntimeException(
                    "Unable to instantiate service " + data.info.name
                    + ": " + e.toString(), e);
            }
        }

        try {
            if (localLOGV) Slog.v(TAG, "Creating service " + data.info.name);
            //1.创建ContextImpl对象。
            ContextImpl context = ContextImpl.createAppContext(this, packageInfo);
            //2.将 Service 作为它的成员变量 mOuterContext 。
            context.setOuterContext(service);
            //3.创建Application对象，如果已经创建那么直接返回，否则先调用Application的onCreate方法。
            Application app = packageInfo.makeApplication(false, mInstrumentation);
            //4.调用attach方法，将ContextImpl作为它的成员变量mBase。
            service.attach(context, this, data.info.name, data.token, app,
                    ActivityManagerNative.getDefault());
            //5.调用Service的onCreate方法。
            service.onCreate();
            //6.将该Service对象缓存起来。
            mServices.put(data.token, service);
            try {
                ActivityManagerNative.getDefault().serviceDoneExecuting(
                        data.token, SERVICE_DONE_EXECUTING_ANON, 0, 0);
            } catch (RemoteException e) {
                // nothing to do.
            }
        } catch (Exception e) {
            if (!mInstrumentation.onException(service, e)) {
                throw new RuntimeException(
                    "Unable to create service " + data.info.name
                    + ": " + e.toString(), e);
            }
        }
    }
```
以上的逻辑分为以下几个部分：
- 根据`Service`的名字，通过`ClassLoader`加载该类，并生成一个`Service`实例。
- 创建一个新的`ContextImpl`对象，并将第一步中创建的`Service`实例作为它的`mOuterContext`变量。
- 创建`Application`对象，这里会先判断当前进程所对应的`Application`对象是否已经创建，如果已经创建，那么就直接返回，否则会先创建它，并依次调用它的`attachBaseContext(Context context)`和`onCreate()`方法。
- 调用`Service`的`attach`方法，初始化成员变量。
![](http://upload-images.jianshu.io/upload_images/1949836-563e7d8759949e9d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
- 调用`Service`的`onCreate()`方法。
- 将该`Service`通过`token`作为`key`，缓存在`mServices`列表当中，之后`Service`生命周期的回调，都依赖于该列表。

## 2.3 进程不存在的情况
在这种情况下，`Service`自然也不会存在，我们会走到`mAm.startProcessLocked`的逻辑，这里会去启动`Service`所在的进程，它究竟是怎么启动的我们先不去细看，只需要知道它是这个目的就可以了，此外，还需要注意的是它将该`ServiceRecord`加入到了`mPendingServices`中。

对于`Service`所在的进程，它的入口函数为`ActivityThread`的`main()`方法，在`main`方法中，会创建一个`ApplicationThread`对象，并调用它的`attach`方法：
![](http://upload-images.jianshu.io/upload_images/1949836-bde768cfc3c0996a.png)
而在`attach`方法中，会调用到`AMS`的`attachApplication`方法：
![](http://upload-images.jianshu.io/upload_images/1949836-07207273aeb686bc.png)
在`attachApplication`方法中，又会去调用内部的`attachApplicationLocked`：
![](http://upload-images.jianshu.io/upload_images/1949836-835648e9419a3d12.png)
这里面的逻辑比较长，我们只需要关注下面这句，我们又看到了熟悉的`mServices`对象：
![](http://upload-images.jianshu.io/upload_images/1949836-4fffdc1e72f26448.png)
在`ActiveServices`中的该方法中，就会去遍历前面谈到的`mPendingServices`列表，再依次调用`realStartServiceLocked`方法，至于这个方法做了什么，大家可以回到前面的`2.2`节去看，这里就不再重复分析了。
![](http://upload-images.jianshu.io/upload_images/1949836-28bf510901e9eb23.png)
# 三、小结
以上就是`startService`的整个流程，`bindService`也是类似一个调用过程，其过程并不复杂，本质上还是 [Framework 源码解析知识梳理(1) - 应用进程与 AMS 的通信实现](http://www.jianshu.com/p/f26a76e19a2d) 所谈到的通信过程，我们所需要学习的是`AMS`端和`Service`所在的应用进程对于`Service`是如何管理的。

系统当中的所有`Service`都是通过`AMS`的`mServices`变量，也就是`ActiveServices`类来进行管理的，并且每一个应用进程中的`Service`都会在`AMS`端会对应一个`ServiceRecord`对象，`ServiceRecord`中维护了应用进程中的`Service`对象所需要的状态信息。

并且，无论我们调用多少次`startService`方法，在应用进程侧都会只存在一个`Service`的实例，它被存储到`ActivityThread`的`ArrayMap`类型的`mServices`变量当中。
