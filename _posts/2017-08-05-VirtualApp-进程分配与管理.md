---
layout: post
title: VirtualApp 进程分配与管理
categories: [Android, 插件化, VirtualApp]
description: 进程分配与管理
keywords: Android, 插件化, VirtualApp
---
# VirtualApp 进程分配与管理

## 一. 前景

> 在Android中每个App在启动前必须先创建一个进程，该进程是由Zygote fork出来的，进程具有独立的资源空间，用于承载App上运行的各种Activity/Service等组件。进程对于上层应用来说是完全透明的，这也是google有意为之，让App程序都是运行在Android Runtime。大多数情况一个App就运行在一个进程中，除非在AndroidManifest.xml中配置Android:process属性，或通过native代码fork进程。 -- 来自[Gityuan](http://gityuan.com/)

VA作为Android应用的容器框架，允许用户安装第三方应用到该容器中，需要为运行在其中的应用创建进程，然而VA也只是运行在Android系统中的一款普通的应用，不能像framework那样简单的通过Zygote fork进程来提供新启动的App的运行环境。所以需要有一套有效的机制来创建进程，并且分配给运行在VA中的应用使用。想要了解Android framework如何创建进程的同学可以查看[理解Android进程创建流程](http://gityuan.com/2016/03/26/app-process-create/)，在这里不做过多研究。

## 二. 四大组件与VAppClient进程

当VA启动容器中的应用时，如果还未分配进程给该应用，那么在`startProcessIfNeedLocked`方法中会为该应用启动并分配进程。

### 2.1. Activity

VA启动Activity时，调用`startActivity`，最终会调到ActivityStack.java中的`startActivityLocked`方法，根据不同的状态，分别执行`startActivityInNewTaskLocked`，`startActivityFromSourceTask`，在这两个方法中都会去调用`startActivityProcess`方法。

*deliverNewIntentLocked方法走的时Activity的onNewIntent逻辑，在这不做过多描述*

> ActivityStack.java

```java
int startActivityLocked(int userId, Intent intent, ActivityInfo info, IBinder resultTo, Bundle options,
                        String resultWho, int requestCode) {
    ...
    if (reuseTask == null) {
        startActivityInNewTaskLocked(userId, intent, info, options);
    } else {
        ...
        if (clearTarget.deliverIntent || singleTop) {
            ...
            if (topRecord != null && !topRecord.marked && topRecord.component.equals(intent.getComponent())) {
                deliverNewIntentLocked(sourceRecord, topRecord, intent);
                delivered = true;
            }
        }
        ...
        if (!startTaskToFront) {
            if (!delivered) {
                destIntent = startActivityProcess(userId, sourceRecord, intent, info);
                if (destIntent != null) {
                    startActivityFromSourceTask(reuseTask, destIntent, info, resultWho, requestCode, options);
                }
            }
        }
    }
    return 0;
}

private Intent startActivityInNewTaskLocked(int userId, Intent intent, ActivityInfo info, Bundle options) {
    Intent destIntent = startActivityProcess(userId, null, intent, info);
    ...
    return destIntent;
}

private boolean startActivityFromSourceTask(TaskRecord task, Intent intent, ActivityInfo info, String resultWho,
                                                int requestCode, Bundle options) {
    ActivityRecord top = task.activities.isEmpty() ? null : task.activities.get(task.activities.size() - 1);
    if (top != null) {
        if (startActivityProcess(task.userId, top, intent, info) != null) {
            realStartActivityLocked(top.token, intent, resultWho, requestCode, options);
            return true;
        }
    }
    return false;
}

private Intent startActivityProcess(int userId, ActivityRecord sourceRecord, Intent intent, ActivityInfo info) {
    ProcessRecord targetApp = mService.startProcessIfNeedLocked(info.processName, userId, info.packageName);
    ...
}
```

### 2.2. Service

VA启动Service时，调用`startService`或者`bindService`，最终会调到VActivityManagerService.java的`startServiceCommon`方法。

> VActivityManagerService.java

```java
private ComponentName startServiceCommon(Intent service,
                                         boolean scheduleServiceArgs, int userId) {
    ServiceInfo serviceInfo = resolveServiceInfo(service, userId);
    if (serviceInfo == null) {
        return null;
    }
    ProcessRecord targetApp = startProcessIfNeedLocked(ComponentUtils.getProcessName(serviceInfo),
            userId,
            serviceInfo.packageName);

    if (targetApp == null) {
        VLog.e(TAG, "Unable to start new Process for : " + ComponentUtils.toComponentName(serviceInfo));
        return null;
    }
    ...
    return ComponentUtils.toComponentName(serviceInfo);
}
```

### 2.3. ContentProvider

VA调用ContentProvider时，调用ContentResolver.query等方法，最终会调到VActivityManagerService.java的`acquireProviderClient`方法。

> VActivityManagerService.java

```java
public IBinder acquireProviderClient(int userId, ProviderInfo info) {
    ProcessRecord callerApp;
    synchronized (mPidsSelfLocked) {
        callerApp = findProcessLocked(VBinder.getCallingPid());
    }
    if (callerApp == null) {
        throw new SecurityException("Who are you?");
    }
    String processName = info.processName;
    ProcessRecord r;
    synchronized (this) {
        r = startProcessIfNeedLocked(processName, userId, info.packageName);
    }
    if (r != null && r.client.asBinder().isBinderAlive()) {
        try {
            return r.client.acquireProviderClient(info);
        } catch (RemoteException e) {
            e.printStackTrace();
        }
    }
    return null;
}
```

### 2.4. BroadcastReceiver

VA处理BroadcastReceiver时，调用sendBroadcast，最终会调到VActivityManagerService的`handleStaticBroadcastAsUser`方法。

```java
private void handleStaticBroadcastAsUser(int vuid, ActivityInfo info, Intent intent,
                                         PendingResultData result) {
    synchronized (this) {
        ProcessRecord r = findProcessLocked(info.processName, vuid);
        if (BROADCAST_NOT_STARTED_PKG && r == null) {
            r = startProcessIfNeedLocked(info.processName, getUserId(vuid), info.packageName);
        }
        if (r != null && r.appThread != null) {
            performScheduleReceiver(r.client, vuid, info, intent,
                    result);
        }
    }
}
```

## 三. VAppClient进程启动与分配

刚刚说了四大组件与进程创建的调用方法，接下来再说说VA中如何为容器中应用分配进程以及确定进程名后如何启动对应进程。

`startProcessIfNeedLocked`中第一个参数为manifest文件中每个组件(application, activity, service, provider, receiver)设置的进程名(`android:process`)，该方法的处理流程如下:

1. 判断当前宿主中可分配的进程数量是否少于3个
  * 是: 回收所有进程
2. 根据传入的进程名(processName)，判断该进程是否启动
  * 是: 直接返回
3. 查询可分配进程: `queryFreeStubProcessLocked`
4. 启动 3 中对应的进程: `performStartProcessLocked`

> VActivityManagerService.java

```java
ProcessRecord startProcessIfNeedLocked(String processName, int userId, String packageName) {
    if (VActivityManagerService.get().getFreeStubCount() < 3) {
        // run GC
        killAllApps();
    }
    PackageSetting ps = PackageCacheManager.getSetting(packageName);
    ApplicationInfo info = VPackageManagerService.get().getApplicationInfo(packageName, 0, userId);
    if (ps == null || info == null) {
        return null;
    }
    if (!ps.isLaunched(userId)) {
        sendFirstLaunchBroadcast(ps, userId);
        ps.setLaunched(userId, true);
        VAppManagerService.get().savePersistenceData();
    }
    int uid = VUserHandle.getUid(userId, ps.appId);
    ProcessRecord app = mProcessNames.get(processName, uid);
    if (app != null && app.client.asBinder().isBinderAlive()) {
        return app;
    }
    int vpid = queryFreeStubProcessLocked();
    if (vpid == -1) {
        return null;
    }
    app = performStartProcessLocked(uid, vpid, info, processName);
    if (app != null) {
        app.pkgList.add(info.packageName);
    }
    return app;
}
```

### 3.1. 分配进程

首先我们来看一下，VA lib下的manifest文件，里面声明了50个StubContentProvider$0～49以及50个StubActivity$C0～49，对应的进程名为p0～p49。

```xml
<provider
    android:name="com.lody.virtual.client.stub.StubContentProvider$C0"
    android:authorities="${applicationId}.virtual_stub_0"
    android:exported="false"
    android:process=":p0" />

...

<provider
    android:name="com.lody.virtual.client.stub.StubContentProvider$C49"
    android:authorities="${applicationId}.virtual_stub_49"
    android:exported="false"
    android:process=":p49" />

<activity
    android:name="com.lody.virtual.client.stub.StubActivity$C0"
    android:configChanges="mcc|mnc|locale|touchscreen|keyboard|keyboardHidden|navigation|orientation|screenLayout|uiMode|screenSize|smallestScreenSize|fontScale"
    android:process=":p0"
    android:taskAffinity="com.lody.virtual.vt"
    android:theme="@style/VATheme" />

...

<activity
    android:name="com.lody.virtual.client.stub.StubActivity$C49"
    android:configChanges="mcc|mnc|locale|touchscreen|keyboard|keyboardHidden|navigation|orientation|screenLayout|uiMode|screenSize|smallestScreenSize|fontScale"
    android:process=":p49"
    android:taskAffinity="com.lody.virtual.vt"
    android:theme="@style/VATheme" />
```

再来看`queryFreeStubProcessLocked`方法，从零开始遍历，查找尚未使用的进程号vpid并返回，返回的数值跟manifest中声明的p0～49中的数值一一对应，如果没有可分配进程号了，返回-1。

> VActivityManagerService.java

```java
// 最大VAppClient进程数，50刚好是manifest中声明的进程数。
public static int STUB_COUNT = 50;

// mPidsSelfLocked中存储了当前正在运行的进程
private final SparseArray<ProcessRecord> mPidsSelfLocked = new SparseArray<ProcessRecord>();

private int queryFreeStubProcessLocked() {
    for (int vpid = 0; vpid < StubManifest.STUB_COUNT; vpid++) {
        int N = mPidsSelfLocked.size();
        boolean using = false;
        while (N-- > 0) {
            ProcessRecord r = mPidsSelfLocked.valueAt(N);
            if (r.vpid == vpid) {
                using = true;
                break;
            }
        }
        if (using) {
            continue;
        }
        return vpid;
    }
    return -1;
}
```

### 3.2. 启动进程

在`performStartProcessLocked`方法中，封装ContentProvider的authority，通过ProviderCall来调用对应StubContentProvider的`call`方法，从而利用framework巧妙的为容器中的应用创建进程。关于framework如何通过ContentProvider创建进程，可查看[Android四大组件与进程启动的关系](http://gityuan.com/2016/10/09/app-process-create-2/)，这里不做深入研究。

> VActivityManagerService.java

```java
private ProcessRecord performStartProcessLocked(int vuid, int vpid, ApplicationInfo info, String processName) {
    ProcessRecord app = new ProcessRecord(info, processName, vuid, vpid);
    Bundle extras = new Bundle();
    BundleCompat.putBinder(extras, "_VA_|_binder_", app);
    extras.putInt("_VA_|_vuid_", vuid);
    extras.putString("_VA_|_process_", processName);
    extras.putString("_VA_|_pkg_", info.packageName);
    Bundle res = ProviderCall.call(StubManifest.getStubAuthority(vpid), "_VA_|_init_process_", null, extras);
    if (res == null) {
        return null;
    }
    int pid = res.getInt("_VA_|_pid_");
    IBinder clientBinder = BundleCompat.getBinder(res, "_VA_|_client_");
    attachClient(pid, clientBinder);
    return app;
}
```

> StubManifest.java

```java
public static String STUB_CP_AUTHORITY = "virtual_stub_";

public static String getStubAuthority(int index) {
    return String.format(Locale.ENGLISH, "%s%d", STUB_CP_AUTHORITY, index);
}
```

阅读源码的时候可能会发现，此处封装的`getStubAuthority`方法返回的是virtual_stub_0～49，并非manifest中注册的${applicationId}.virtual_stub_0～49。其实STUB_CP_AUTHORITY的值，已经在VA初始化的方法中被修改了。哈哈，刚开始看的时候还没注意，没想明白authority不一样是如何查到provider的。

> VirtualCore.java

```java
public void startup(Context context) throws Throwable {
    if (!isStartUp) {
        ...
        StubManifest.STUB_CP_AUTHORITY = context.getPackageName() + "." + StubManifest.STUB_DEF_AUTHORITY;
        ...
    }
}
```

> StubContentProvider.java

当call方法执行时，manifest中声明的对应进程就已经启动了，在该方法中初始化进程相关数据。

```java
@Override
public Bundle call(String method, String arg, Bundle extras) {
    if ("_VA_|_init_process_".equals(method)) {
        return initProcess(extras);
    }
    return null;
}

private Bundle initProcess(Bundle extras) {
    ConditionVariable lock = VirtualCore.get().getInitLock();
    if (lock != null) {
        lock.block();
    }
    IBinder token = BundleCompat.getBinder(extras,"_VA_|_binder_");
    int vuid = extras.getInt("_VA_|_vuid_");
    VClientImpl client = VClientImpl.get();
    client.initProcess(token, vuid);
    Bundle res = new Bundle();
    BundleCompat.putBinder(res, "_VA_|_client_", client.asBinder());
    res.putInt("_VA_|_pid_", Process.myPid());
    return res;
}
```

*注: ProviderCall.call方法的使用可查看[VirtualApp 中的进程]()*

## 四. 参考

* [理解Android进程创建流程](http://gityuan.com/2016/03/26/app-process-create/)
* [Android四大组件与进程启动的关系](http://gityuan.com/2016/10/09/app-process-create-2/)