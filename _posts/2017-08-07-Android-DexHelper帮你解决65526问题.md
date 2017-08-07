---
layout: post
title: Android DexHelper帮你解决65536问题
categories: [Android, 优化]
description: 进程分配与管理
keywords: Android, 优化
---
# Android DexHelper帮你解决65536问题

## 一. 前景

随着Android平台的持续成长，Android应用的大小也在增加。当应用及其引用的库达到特定大小时，会遇到构建错误，指明你的应用已达到Android应用构建架构的极限。一般情况下，你会获得类似的错误信息：

```
Dex: The number of method references in a .dex file cannot exceed 64K.
com.android.dex.DexIndexOverflowException: method ID not in [0, 0xffff]: 65536
```

## 二. MultiDex

### 2.1. 官方方案

Dex文件64K方法数的限制已经被提及过很多次，Android官方也提供了[MultiDex](https://developer.android.com/studio/build/multidex.html)来解决这一问题。

### 2.2. 局限性

官方文档中也指出了MultiDex的局限性：

* 启动期间在设备数据分区中安装 DEX 文件的过程相当复杂，如果辅助 DEX 文件较大，可能会导致应用无响应 (ANR) 错误。在此情况下，您应该通过 ProGuard 应用代码压缩以尽量减小 DEX 文件的大小，并移除未使用的那部分代码。
* 使用 Dalvik 可执行文件分包的应用可能无法在运行的平台版本早于 Android 4.0（API 级别 14）的设备上启动。
* 使用 Dalvik 可执行文件分包配置的应用发出非常庞大的内存分配请求，则可能会在运行期间发生崩溃。尽管 Android 4.0（API 级别 14）提高了分配限制，但在 Android 5.0（API 级别 21）之前的 Android 版本上，应用仍有可能遭遇这一限制。

在将应用配置为支持使用64K或更多方法引用之前，开发者应该采取措施减少应用代码调用的引用总数，包括由应用代码或包含的库定义的方法。下列策略可帮助您避免达到 DEX 引用限制：

* 检查您的应用的直接和传递依赖项 - 确保您在应用中使用任何庞大依赖库所带来的好处大于为应用添加大量代码所带来的弊端。一种常见的反面模式是，仅仅为了使用几个实用方法就在应用中加入非常庞大的库。减少您的应用代码依赖项往往能够帮助您规避 dex 引用限制。
* 通过 ProGuard 移除未使用的代码 - 为您的版本构建启用代码压缩以运行 ProGuard。启用压缩可确保您交付的 APK 不含有未使用的代码。

## 三. 工具

由于MultiDex存在的局限性，官方也给出了上述的两个策略来辅助减少应用代码调用的引用总数。

在实施这两个策略之前，我们需要一个好用的工具来统计当前应用代码的方法数，在添加一个依赖库之后，需要评估该依赖库会引入多少方法，是否值得引用。

### 3.1. DexHelper

一款分析Dex文件的Gradle插件，在APK打包的同时，输出APK包中的dex文件相关信息: 引用方法数等。实现原理参考的官网[Dex 格式](https://source.android.com/devices/tech/dalvik/dex-format)，根据Dex文件格式，解析APK包中的.dex二进制文件，获取该dex文件中方法总数。

1. 修改父项目build.gradle，添加依赖`com.ximsfei:dexhelper:version`

```gradle
buildscript {
    repositories {
        jcenter()
    }
    dependencies {
        classpath 'com.android.tools.build:gradle:2.3.3'
        // 添加依赖
        classpath 'com.ximsfei:dexhelper:version'
    }
}
```

*注: 最新version，以[官方地址](https://github.com/ximsfei/DexHelper#dependencies)为准*

2. 修改子项目build.gradle，应用插件`com.ximsfei.dexhelper`

```
apply plugin: 'com.android.application'
// 应用插件
apply plugin: 'com.ximsfei.dexhelper'
```

3. 运行，通过assemble*命令打包apk，自动输出结果

```
$ ./gradlew assemble -q
  DexHelper: apk file -> app-debug.apk
  DexHelper: dex file -> classes.dex
  DexHelper: method size -> 27022
  DexHelper: apk file -> app-release-unsigned.apk
  DexHelper: dex file -> classes.dex
  DexHelper: method size -> 27021
```

## 四. 总结

在选择解决方案时，利用好的工具来评估实施方案的价值，提高代码运行效率，降低应用本身的体积，是每个开发者需要努力去做的。使用MultiDex来解决项目日益增长带来的问题，不如在开发的过程中，将点滴细节做到极致。小工具大用途，[DexHelper](https://github.com/ximsfei/DexHelper)可以帮你将应用做到小而美。

项目地址: [https://github.com/ximsfei/DexHelper](https://github.com/ximsfei/DexHelper)