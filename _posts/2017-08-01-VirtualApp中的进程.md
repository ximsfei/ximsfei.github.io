---
layout: post
title: VirtualApp中的进程
categories: [Android, 插件化, VirtualApp]
description: VirtualApp进程
keywords: Android, 插件化, VirtualApp
---

# VirtualApp 中的进程

## 一. 什么是进程

> 进程（Process）是计算机中的程序关于某数据集合上的一次运行活动，是系统进行资源分配和调度的基本单位，是操作系统结构的基础。在早期面向进程设计的计算机结构中，进程是程序的基本执行实体；在当代面向线程设计的计算机结构中，进程是线程的容器。程序是指令、数据及其组织形式的描述，进程是程序的实体。 -- 来自[百度百科](http://baike.baidu.com/item/%E8%BF%9B%E7%A8%8B)

## 二. VA进程

VA在运行时，存在四种类型的进程:
* Server Process: 进程名hostPackageName:x，参照Android system_server进程实现，用于管理VPackageManagerService，VActivityManagerService等服务。
* VAppClient Process: 进程名hostPackageName:p[0~...]，VA容器中安装的第三方应用将运行在此类进程中。
* Main Process: 宿主应用主进程。
* Child Process: 宿主应用子进程。

> com/lody/virtual/client/core/VirtualCore.java
```java
/**
 * Process type
 */
private enum ProcessType {
    /**
     * Server process
     */
    Server,
    /**
     * Virtual app process
     */
    VAppClient,
    /**
     * Main process
     */
    Main,
    /**
     * Child process
     */
    CHILD
}
```

### 2.1 Server进程

Android通过system_server进程来管理整个Java framework层，包含ActivityManager，PackageManager等各种系统服务；在系统启动的时候就会启动该进程。

VA是一个开源的Android应用容器框架，允许用户在应用中创建虚拟空间，并且安装第三方应用到该虚拟空间中。为了保证应用可以无感知的运行在容器中，VA参照framework源码实现，构造了一个能够运行app的环境。
VA利用ContentProvider的同步特性构建了一套跨进程同步通信机制，把ContentProvider当作一个ServiceManager，所有的系统服务都放在这里管理，利用ContentProvider call的同步调用方式，把Binder对象透传到不同的进程；

> com/lody/virtual/server/BinderProvider.java

在BinderProvider的onCreate中会初始化VPackageManagerService, VActivityManagerService, VAppManagerService等服务，这些服务均能在Android framework中找到缩影(PackageManagerService, ActivityManangerService等)。并把实例化的Service存储到ServiceCache的sCache Map中。

```java
@Override
public boolean onCreate() {
    ...
    VPackageManagerService.systemReady();
    addService(ServiceManagerNative.PACKAGE, VPackageManagerService.get());
    VActivityManagerService.systemReady(context);
    addService(ServiceManagerNative.ACTIVITY, VActivityManagerService.get());
    addService(ServiceManagerNative.USER, VUserManagerService.get());
    VAppManagerService.systemReady();
    addService(ServiceManagerNative.APP, VAppManagerService.get());
    BroadcastSystem.attach(VActivityManagerService.get(), VAppManagerService.get());
    if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.LOLLIPOP) {
        addService(ServiceManagerNative.JOB, VJobSchedulerService.get());
    }
    VNotificationManagerService.systemReady(context);
    addService(ServiceManagerNative.NOTIFICATION, VNotificationManagerService.get());
    VAppManagerService.get().scanApps();
    VAccountManagerService.systemReady();
    addService(ServiceManagerNative.ACCOUNT, VAccountManagerService.get());
    addService(ServiceManagerNative.VS, VirtualStorageService.get());
    return true;
}


private void addService(String name, IBinder service) {
    ServiceCache.addService(name, service);
}
```

> com/lody/virtual/server/ServiceCache.java

```java
public class ServiceCache {

	private static final Map<String, IBinder> sCache = new ArrayMap<>(5);

	public static void addService(String name, IBinder service) {
		sCache.put(name, service);
	}

	public static IBinder removeService(String name) {
		return sCache.remove(name);
	}

	public static IBinder getService(String name) {
		return sCache.get(name);
	}

}
```

了解源码的同学都知道，ActivityManagerService、PackageManagerService等系统服务，在每个进程中都对应一个Client(ActivityManager, PackageManager)。VA参照源码的实现，在Server进程中启动VPackageManagerService，VActivityManagerService等系统服务，在每个进程中也有对应的Client(VActivityManager，VPackageManager)。

### 2.2 Client与Server通信

以VActivityManager与VActivityManagerService通信为例，其他Service流程类似。

> com/lody/virtual/client/ipc/VActivityManager.java

业务层调用VActivityManager的任意方法，会先调用getService()方法获取IActivityManager。

```java
public IActivityManager getService() {
    if (mRemote == null || !mRemote.asBinder().isBinderAlive()) {
        synchronized (VActivityManager.class) {
            final Object remote = getRemoteInterface();
            mRemote = LocalProxyUtils.genProxy(IActivityManager.class, remote);
        }
    }
    return mRemote;
}

private Object getRemoteInterface() {
    return IActivityManager.Stub
            .asInterface(ServiceManagerNative.getService(ServiceManagerNative.ACTIVITY));
}
```

> com/lody/virtual/client/ipc/ServiceManagerNative.java

ServiceManagerNative的getService方法中，先判断是否为Server进程:
* 是，直接从ServiceCache的Map中获取对应的Service。
* 不是，会通过authorities获取BinderProvider，并调用BinderProvider的call方法，参数method为"@"。

```java
public static String SERVICE_CP_AUTH = "virtual.service.BinderProvider";

public static IBinder getService(String name) {
    if (VirtualCore.get().isServerProcess()) {
        return ServiceCache.getService(name);
    }
    IServiceFetcher fetcher = getServiceFetcher();
    if (fetcher != null) {
        try {
            return fetcher.getService(name);
        } catch (RemoteException e) {
            e.printStackTrace();
        }
    }
    VLog.e(TAG, "GetService(%s) return null.", name);
    return null;
}

private static IServiceFetcher getServiceFetcher() {
    if (sFetcher == null || !sFetcher.asBinder().isBinderAlive()) {
        synchronized (ServiceManagerNative.class) {
            Context context = VirtualCore.get().getContext();
            Bundle response = new ProviderCall.Builder(context, SERVICE_CP_AUTH).methodName("@").call();
            if (response != null) {
                IBinder binder = BundleCompat.getBinder(response, "_VA_|_binder_");
                linkBinderDied(binder);
                sFetcher = IServiceFetcher.Stub.asInterface(binder);
            }
        }
    }
    return sFetcher;
}
```

```xml
<provider
    android:name="com.lody.virtual.server.BinderProvider"
    android:authorities="${applicationId}.virtual.service.BinderProvider"
    android:exported="false"
    android:process=":x" />
```

> com/lody/virtual/client/ipc/ProviderCall.java

通过传入的authority，查找对应的ContentProvider，并调用ContentProvider的call方法。

```java
public class ProviderCall {

	public static Bundle call(String authority, String methodName, String arg, Bundle bundle) {
		return call(authority, VirtualCore.get().getContext(), methodName, arg, bundle);
	}

	public static Bundle call(String authority, Context context, String method, String arg, Bundle bundle) {
		Uri uri = Uri.parse("content://" + authority);
		return ContentProviderCompat.call(context, uri, method, arg, bundle);
	}

	public static final class Builder {

		private Context context;

		private Bundle bundle = new Bundle();

		private String method;
		private String auth;
		private String arg;

		public Builder(Context context, String auth) {
			this.context = context;
			this.auth = auth;
		}

		public Builder methodName(String name) {
			this.method = name;
			return this;
		}

		public Builder arg(String arg) {
			this.arg = arg;
			return this;
		}

		public Builder addArg(String key, Object value) {
			if (value != null) {
				 if (value instanceof Boolean) {
					bundle.putBoolean(key, (Boolean) value);
				} else if (value instanceof Integer) {
					bundle.putInt(key, (Integer) value);
				} else if (value instanceof String) {
					bundle.putString(key, (String) value);
				} else if (value instanceof Serializable) {
					bundle.putSerializable(key, (Serializable) value);
				} else if (value instanceof Bundle) {
					bundle.putBundle(key, (Bundle) value);
				} else if (value instanceof Parcelable) {
					bundle.putParcelable(key, (Parcelable) value);
				} else {
					throw new IllegalArgumentException("Unknown type " + value.getClass() + " in Bundle.");
				}
			}
			return this;
		}

		public Bundle call() {
			return ProviderCall.call(auth, context, method, arg, bundle);
		}

	}

}
```

> com/lody/virtual/helper/compat/ContentProviderCompat.java

```java
public static Bundle call(Context context, Uri uri, String method, String arg, Bundle extras) {
    if (VERSION.SDK_INT < Build.VERSION_CODES.JELLY_BEAN_MR1) {
        return context.getContentResolver().call(uri, method, arg, extras);
    }
    ContentProviderClient client = crazyAcquireContentProvider(context, uri);
    Bundle res = null;
    try {
        res = client.call(method, arg, extras);
    } catch (RemoteException e) {
        e.printStackTrace();
    } finally {
        releaseQuietly(client);
    }
    return res;
}
```

> com/lody/virtual/server/BinderProvider.java

BinderProvider的call方法会将mServiceFetcher封装到Bundle中返回给Client。Client端可以通过ServiceFetcher的getService()方法获取Server端启动时存储在ServiceCache中的Service。至此，Client与Server可以通过getService进行方法调用，建立通信了。

```java
@Override
private final ServiceFetcher mServiceFetcher = new ServiceFetcher();

public Bundle call(String method, String arg, Bundle extras) {
    if ("@".equals(method)) {
        Bundle bundle = new Bundle();
        BundleCompat.putBinder(bundle, "_VA_|_binder_", mServiceFetcher);
        return bundle;
    }
    return null;
}

private class ServiceFetcher extends IServiceFetcher.Stub {
    @Override
    public IBinder getService(String name) throws RemoteException {
        if (name != null) {
            return ServiceCache.getService(name);
        }
        return null;
    }

    @Override
    public void addService(String name, IBinder service) throws RemoteException {
        if (name != null && service != null) {
            ServiceCache.addService(name, service);
        }
    }

    @Override
    public void removeService(String name) throws RemoteException {
        if (name != null) {
            ServiceCache.removeService(name);
        }
    }
}
```

## 三. 总结


至此，我们已经打通了VA构建在ContentProvider机制上的同步通信方式。利用ContentProvider来模拟framework层的ServiceManager，使整个框架的核心得以摆脱AIDL Service异步过程的苦恼，代码看起来比较整洁，用起来也非常方便；同时，这个机制也解决了插件之间跨进程通信的难题；类似DroidPlugin这样的多进程机制的插件系统，如果插件之间需要通信，必须使用IPC机制：广播，AIDL等等，这些通信都是异步的，写起来非常麻烦；而构建在ContentProvider机制上的同步AIDL通信使得插件的跨进程通信就像普通函数调用一样简单。

## 四. 参考

* [如何看待 Lody 新开源的 VirtualApp 插件框架？](https://www.zhihu.com/question/48269910)
