---
layout: post
title: VirtualApp Java层Hook基础
categories: [Android, 插件化, VirtualApp]
description: 反射注入
keywords: Android, 插件化, VirtualApp
---

# VirtualApp Java层Hook基础

Hook技术是VirtualApp(后续简称VA)的核心实现原理之一。

## 一. 什么是Hook

> Hook是Windows中提供的一种用以替换DOS下“中断”的系统机制，中文译为“挂钩”或“钩子”。在对特定的系统事件进行hook后，一旦发生已hook事件，对该事件进行hook的程序就会收到系统的通知，这时程序就能在第一时间对该事件做出响应。 -- 来自[百度百科](http://baike.baidu.com/item/hook/19512231)

在Android插件化中的Hook机制一般是通过反射注入和动态代理来实现的。

## 二. Hook实现

Android中的Hook本身依赖反射机制，VA的Hook框架基于注解的反射注入技术实现的极为优雅，初看源码可能会不知所云，深入研究就会发现VA虽然模拟了整个AndroidFramework，Hook了大量对象，代码却依然井井有条。

基于VA，研究注解的反射注入技术，相关源码:

```java
mirror/
 - RefClass.java
 - RefConstructor.java
 - RefStaticMethod.java
 - RefStaticObject.java
 - RefStaticInt.java
 - RefMethod.java
 - RefObject.java
 - RefBoolean.java
 - RefDouble.java
 - RefFloat.java
 - RefInt.java
 - RefLong.java
 - MethodParams.java
 - MethodReflectParams.java
 
mirror/android/app/
 - ActivityThread.java
 
// Android Framework
frameworks/base/core/java/android/app/
 - ActivityThread.java
```

### 1. ActivityThread的反射注入-普通实现

```java
public class PluginInstrumentation extends Instrumentation {
    private final Instrumentation mBase;

    public PluginInstrumentation(Instrumentation base) {
        mBase = base;
    }
}

public class HookHelper {
    public static void inject() throws Exception {
        Class<?> activityThreadClass = Class.forName("android.app.ActivityThread");
        Method currentActivityThreadMethod = activityThreadClass.getMethod("currentActivityThread");
        currentActivityThreadMethod.setAccessible(true);
        Object currentActivityThread = currentActivityThreadMethod.invoke(null);
        Field instrumentationField = activityThreadClass.getDeclaredField("mInstrumentation");
        instrumentationField.setAccessible(true);
        Instrumentation instrumentation = (Instrumentation) instrumentationField.get(currentActivityThread);
        // HookHelper: 反射注入前 instrumentation = android.app.Instrumentation@3520a6d
        Log.e("HookHelper", "反射注入前 instrumentation = " + instrumentationField.get(currentActivityThread));
        if(!(instrumentation instanceof PluginInstrumentation)) {
            instrumentationField.set(currentActivityThread, new PluginInstrumentation(instrumentation));
        }
        // HookHelper: 反射注入后 instrumentation = com.ximsfei.kotlin.hook.PluginInstrumentation@669d6a2
        Log.e("HookHelper", "反射注入后 instrumentation = " + instrumentationField.get(currentActivityThread));
    }
}
```

### 2. ActivityThread的反射注入-VA实现

#### 1. 原理介绍

> mirror/MethodParams.java

```java
@Target({ElementType.FIELD})
@Retention(RetentionPolicy.RUNTIME)
public @interface MethodParams {
    Class<?>[] value();
}
```

该注解MethodParams用于标注某个方法(包括构造方法)包含哪些参数。eg: `@MethodParams({IBinder.class, List.class})`, 参数为Class<?>对象。

> mirror/MethodReflectParams.java

```java
@Target({ElementType.FIELD})
@Retention(RetentionPolicy.RUNTIME)
public @interface MethodReflectParams {
    String[] value();
}
```

该注解MethodReflectParams用于标注某个方法(包括构造方法)包含哪些参数。eg: `@MethodReflectParams({"android.content.pm.PackageParser$Package", "int"})`, 参数为字符串对象(全类名)，最终通过`Class.forName(params);`获取对应的Class<?>对象。

VA还基于Java的反射技术封装了:
* 基本Field类(RefBoolean.java, RefDouble.java, RefFloat.java, RefInt.java, RefLong.java)
* 普通Field类(RefObject.java)
* 静态成Field类(RefStaticInt.java, RefStaticObject.java)
* 方法类(RefMethod.java)
* 静态方法类(RefStaticMethod.java)
* 构造方法类(RefConstructor.java)
* Class类(RefClass.java)

> mirror/RefClass.java

RefClass类使用HashMap存储了上述封装类Class<?>对象与其构造方法的映射关系。当调用`RefClass.load(mirror.android.app.ActivityThread.class, "android.app.ActivityThread")`方法时，RefClass会遍历mappingClass(`mirror.android.app.ActivityThread`)类中的成员变量，并从realClass(`android.app.ActivityThread`)中获取对应的Field封装成上述类赋给mappingClass。

```java
public final class RefClass {

    private static HashMap<Class<?>,Constructor<?>> REF_TYPES = new HashMap<Class<?>, Constructor<?>>();
    static {
        try {
            REF_TYPES.put(RefObject.class, RefObject.class.getConstructor(Class.class, Field.class));
            REF_TYPES.put(RefMethod.class, RefMethod.class.getConstructor(Class.class, Field.class));
            REF_TYPES.put(RefInt.class, RefInt.class.getConstructor(Class.class, Field.class));
            REF_TYPES.put(RefLong.class, RefLong.class.getConstructor(Class.class, Field.class));
            REF_TYPES.put(RefFloat.class, RefFloat.class.getConstructor(Class.class, Field.class));
            REF_TYPES.put(RefDouble.class, RefDouble.class.getConstructor(Class.class, Field.class));
            REF_TYPES.put(RefBoolean.class, RefBoolean.class.getConstructor(Class.class, Field.class));
            REF_TYPES.put(RefStaticObject.class, RefStaticObject.class.getConstructor(Class.class, Field.class));
            REF_TYPES.put(RefStaticInt.class, RefStaticInt.class.getConstructor(Class.class, Field.class));
            REF_TYPES.put(RefStaticMethod.class, RefStaticMethod.class.getConstructor(Class.class, Field.class));
            REF_TYPES.put(RefConstructor.class, RefConstructor.class.getConstructor(Class.class, Field.class));
        }
        catch (Exception e) {
            e.printStackTrace();
        }
    }

    public static Class<?> load(Class<?> mappingClass, String className) {
        try {
            return load(mappingClass, Class.forName(className));
        } catch (Exception e) {
            return null;
        }
    }

    public static Class load(Class mappingClass, Class<?> realClass) {
        Field[] fields = mappingClass.getDeclaredFields();
        for (Field field : fields) {
            try {
                if (Modifier.isStatic(field.getModifiers())) {
                    Constructor<?> constructor = REF_TYPES.get(field.getType());
                    if (constructor != null) {
                        field.set(null, constructor.newInstance(realClass, field));
                    }
                }
            }
            catch (Exception e) {
                // Ignore
            }
        }
        return realClass;
    }
}
```

> mirror/RefObject.java

Field类封装了Java反射中java.lang.reflect.Field的field.get()和field.set()方法；其他Field类类似。
```java
public class RefObject<T> {
    private Field field;

    public RefObject(Class<?> cls, Field field) throws NoSuchFieldException {
        this.field = cls.getDeclaredField(field.getName());
        this.field.setAccessible(true);
    }

    public T get(Object object) {
        try {
            return (T) this.field.get(object);
        } catch (Exception e) {
            return null;
        }
    }

    public void set(Object obj, T value) {
        try {
            this.field.set(obj, value);
        } catch (Exception e) {
            //Ignore
        }
    }
}
```

> mirror/RefMethod.java

方法类封装了Java反射中java.lang.reflect.Method的method.invoke()方法；在RefMethod构造方法中，会根据不同的注解MethodParams, MethodReflectParams获取对应的方法参数，从而获取对应的Method。如果没有注解，表明对应方法没有参数。

```java
public class RefMethod<T> {
    private Method method;

    public RefMethod(Class<?> cls, Field field) throws NoSuchMethodException {
        if (field.isAnnotationPresent(MethodParams.class)) {
            Class<?>[] types = field.getAnnotation(MethodParams.class).value();
            for (int i = 0; i < types.length; i++) {
                Class<?> clazz = types[i];
                if (clazz.getClassLoader() == getClass().getClassLoader()) {
                    try {
                        Class.forName(clazz.getName());
                        Class<?> realClass = (Class<?>) clazz.getField("TYPE").get(null);
                        types[i] = realClass;
                    } catch (Throwable e) {
                        throw new RuntimeException(e);
                    }
                }
            }
            this.method = cls.getDeclaredMethod(field.getName(), types);
            this.method.setAccessible(true);
        } else if (field.isAnnotationPresent(MethodReflectParams.class)) {
            String[] typeNames = field.getAnnotation(MethodReflectParams.class).value();
            Class<?>[] types = new Class<?>[typeNames.length];
            for (int i = 0; i < typeNames.length; i++) {
                Class<?> type = getProtoType(typeNames[i]);
                if (type == null) {
                    try {
                        type = Class.forName(typeNames[i]);
                    } catch (ClassNotFoundException e) {
                        e.printStackTrace();
                    }
                }
                types[i] = type;
            }
            this.method = cls.getDeclaredMethod(field.getName(), types);
            this.method.setAccessible(true);
        }
        else {
            for (Method method : cls.getDeclaredMethods()) {
                if (method.getName().equals(field.getName())) {
                    this.method = method;
                    this.method.setAccessible(true);
                    break;
                }
            }
        }
        if (this.method == null) {
            throw new NoSuchMethodException(field.getName());
        }
    }

    public T call(Object receiver, Object... args) {
        try {
            return (T) this.method.invoke(receiver, args);
        } catch (InvocationTargetException e) {
            if (e.getCause() != null) {
                e.getCause().printStackTrace();
            } else {
                e.printStackTrace();
            }
        } catch (Throwable e) {
            e.printStackTrace();
        }
        return null;
    }

    public T callWithException(Object receiver, Object... args) throws Throwable {
        try {
            return (T) this.method.invoke(receiver, args);
        } catch (InvocationTargetException e) {
            if (e.getCause() != null) {
                throw e.getCause();
            }
            throw e;
        }
    }

    public Class<?>[] paramList() {
        return method.getParameterTypes();
    }
}
```

> mirror/RefConstructor.java

构造方法类封装了Java反射中java.lang.reflect.Constructor的ctor.newInstance()方法。

```java
public class RefConstructor<T> {
    private Constructor<?> ctor;

    public RefConstructor(Class<?> cls, Field field) throws NoSuchMethodException {
        if (field.isAnnotationPresent(MethodParams.class)) {
            Class<?>[] types = field.getAnnotation(MethodParams.class).value();
            ctor = cls.getDeclaredConstructor(types);
        } else if (field.isAnnotationPresent(MethodReflectParams.class)) {
            String[] values = field.getAnnotation(MethodReflectParams.class).value();
            Class[] parameterTypes = new Class[values.length];
            int N = 0;
            while (N < values.length) {
                try {
                    parameterTypes[N] = Class.forName(values[N]);
                    N++;
                } catch (Exception e) {
                    e.printStackTrace();
                }
            }
            ctor = cls.getDeclaredConstructor(parameterTypes);
        } else {
            ctor = cls.getDeclaredConstructor();
        }
        if (ctor != null && !ctor.isAccessible()) {
            ctor.setAccessible(true);
        }
    }

    public T newInstance() {
        try {
            return (T) ctor.newInstance();
        } catch (Exception e) {
            return null;
        }
    }

    public T newInstance(Object... params) {
        try {
            return (T) ctor.newInstance(params);
        } catch (Exception e) {
            return null;
        }
    }
}
```

#### 2. VA实例

对应[ActivityThread的反射注入-普通实现](#ActivityThread的反射注入-普通实现)，在VA中的反射注入实现:

> mirror/android/app/ActivityThread.java

在`mirror.android.app.ActivityThread`类加载时，会调用`RefClass.load(ActivityThread.class, "android.app.ActivityThread");`方法，遍历`mirror.android.app.ActivityThread`的成员变量，并从`android.app.ActivityThread`中获取对应的Field封装成上述类赋给`mirror.android.app.ActivityThread`。

```java
public class ActivityThread {
    public static Class<?> TYPE = RefClass.load(ActivityThread.class, "android.app.ActivityThread");
    public static RefStaticMethod currentActivityThread;
    public static RefMethod<String> getProcessName;
    public static RefMethod<Handler> getHandler;
    public static RefMethod<Object> installProvider;
    public static RefObject<Object> mBoundApplication;
    public static RefObject<Handler> mH;
    public static RefObject<Application> mInitialApplication;
    public static RefObject<Instrumentation> mInstrumentation;
    public static RefObject<Map<String, WeakReference<?>>> mPackages;
    public static RefObject<Map> mProviderMap;
    @MethodParams({IBinder.class, List.class})
    public static RefMethod<Void> performNewIntents;
    public static RefStaticObject<IInterface> sPackageManager;
    @MethodParams({IBinder.class, String.class, int.class, int.class, Intent.class})
    public static RefMethod<Void> sendActivityResult;
    public static RefMethod<Binder> getApplicationThread;
}
```

*注: 在mirror.android.app.ActivityThread中的成员变量必须与framework中android.app.ActivityThread中对应的成员变量名相同，因为在RefClass.load()方法中是通过遍历mirror.android.app.ActivityThread中的Fields来获取对应的android.app.ActivityThread中的Fields的*

> com/lody/virtual/client/hook/delegate/AppInstrumentation.java

在业务层使用的时候，只需要调用`ActivityThread.mInstrumentation.get(VirtualCore.mainThread());`方法就可以获取当前`android.app.ActivityThread`中mInstrumentation的值，调用`ActivityThread.mInstrumentation.set(VirtualCore.mainThread(), this);`就可以通过反射注入业务层自己实现的AppInstrumentation了。

```java
public final class AppInstrumentation extends InstrumentationDelegate implements IInjector {
    ...

    private AppInstrumentation(Instrumentation base) {
        super(base);
    }


    @Override
    public void inject() throws Throwable {
        base = ActivityThread.mInstrumentation.get(VirtualCore.mainThread());
        ActivityThread.mInstrumentation.set(VirtualCore.mainThread(), this);
    }
    
    ...
}
```

## 三. 总结
 
对比[ActivityThread的反射注入-普通实现](#ActivityThread的反射注入-普通实现)和[ActivityThread的反射注入-VA实现](#ActivityThread的反射注入-VA实现)，初一看，可能觉得VA实现多了很多代码，还有些绕；但是普通方法实现，每对一个属性进行反射注入都需要写下方流程中的所有代码，或者部分代码；通过VA的实现方法，只需要在对应的类中(例如: `mirror.android.app.ActivityThread`)写上需要反射注入的属性，业务层代码也更简洁可读性更高。

### 1. 反射注入的一般流程

1. 获取需要注入的类的Class对象

`Class<?> activityThreadClass = Class.forName("android.app.ActivityThread");`

2. 获取Class对象中需要注入的Field

`Field instrumentationField = activityThreadClass.getDeclaredField("mInstrumentation");`

3. 通过Field的set方法注入业务层已实现的对象

`instrumentationField.set(currentActivityThread, new PluginInstrumentation(instrumentation));`

### 2. 反射调用方法的一般流程

1. 获取需要注入的类的Class对象

`Class<?> activityThreadClass = Class.forName("android.app.ActivityThread");`

2. 获取Class对象中需要调用的Method

`Method currentActivityThreadMethod = activityThreadClass.getMethod("currentActivityThread");`

3. 通过Method的invoke方法调用

`Object currentActivityThread = currentActivityThreadMethod.invoke(null);`

Github地址: https://github.com/ximsfei/RefInject
