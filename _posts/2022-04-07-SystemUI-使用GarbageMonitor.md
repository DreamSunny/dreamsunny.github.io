---
layout:     post
title:      "SystemUI-使用GarbageMonitor"
subtitle:   ""
date:       2022-04-07 12:00:00
author:     "Yaku"
header-img: "img/post-bg-2015.jpg"
tags:
    - Android
    - SystemUI
---

# 1.简介

GarbageMonitor 用于定期监测 SystemUI 堆内存，提供了一个自定义快捷开关方便生成 hprof 文件。

目前仅支持 UserDebug 版本中使用，其核心功能是在内部 Service 中启动的，且有一定的条件限制，需要手动写入 Settings 值或 SystemProperties 值：

```Java
public static final boolean LEAK_REPORTING_ENABLED = Build.IS_DEBUGGABLE
        && SystemProperties.getBoolean("debug.enable_leak_reporting", false);
public static final boolean HEAP_TRACKING_ENABLED = Build.IS_DEBUGGABLE;

public void start() {
    boolean forceEnable =
            Settings.Secure.getInt(
                            mContext.getContentResolver(), FORCE_ENABLE_LEAK_REPORTING, 0) //sysui_force_enable_leak_reporting
                    != 0;
    if (LEAK_REPORTING_ENABLED || forceEnable) {
        // 内存泄漏检测
        mGarbageMonitor.startLeakMonitor();
    }
    if (HEAP_TRACKING_ENABLED || forceEnable) {
        // 每隔1分钟记录rss值
        mGarbageMonitor.startHeapTracking();
    }
}
```

 

1.使用 UserDebug 版本，在命令行中执行：adb shell settings put secure sysui_force_enable_leak_reporting 1 ，重启 SystemUI ，开启 GarbageMonitor 的所有功能。

2.在自定义快捷开关中找到“转储 SysUI 堆”，点击后会在 /data/user_de/0/com.android.systemui/cache/leak/ 目录下生成对应进程id的 ahprof 文件，拷贝出来后重命名为 hprof 后可拖拽到 AndroidStudio 查看。



# 2.如何检测内存泄漏-LeakDetector

LeakDetector 主要提供了3种监测场景：

 

**1.监测类创建实例的数量，主要监控占用较大的对象，从而监测内存占用大小**

对应方法：public <T> void trackInstance(T object) {}

注意事项：在创建对象实例时需调用此方法。

 

**2.监测集合内元素的数量，从而监控内存占用大小**

对应方法：public <T> void trackCollection(Collection<T> collection, String tag) {}

注意事项：每当向集合内添加新的元素时需调用此方法。

 

**3.监测快要被****JVM****回收的实例，如果1分钟内实例没有被回收，则可能存在内存泄漏**

对应方法：public void trackGarbage(Object o) {}

注意事项：删除实例的最后一个强引用时需调用此方法。

 

小结：

1.代码中可通过依赖注入的方式，也可以通过 Dependency 获取 LeakDetector 的实例。

2.场景1,2用于主动监测内存占用，场景3可检测可能存在的内存泄漏。

3.内存泄漏发生后会自动生成 leak.hprof 文件，并提示获取此文件，也可通过 dump 查看统计信息。

 

# 3.检测内存泄漏原理-TrackedGarbage

TrackedGarbage 类中包含两个关键成员变量：mGarbage和mRefQueue：

```Java
private final HashSet<LeakReference> mGarbage = new HashSet<>();
private final ReferenceQueue<Object> mRefQueue = new ReferenceQueue<>();
 
public synchronized void track(Object o) {
    mGarbage.add(new LeakReference(o, mRefQueue));
}
```

当调用 track() 方法传入一个需要被监测的对象时，会首先创建改对象的弱引用并与 mRefQueue 关联，其作用是：

该对象被虚拟机回收后，会将该对象 Reference 添加到 ReferenceQueue 中，也就是 mRefQueue 中。通过遍历 mRefQueue 即可确认被追踪的对象是否被正常回收。

mGarbage 用来保存所有被追踪对象的弱引用，通过遍历 mRefQueue 获取到被追踪对象的引用，从而确认有哪些对象没有被回收，具体参考 cleanUp() 方法：

```Java
private void cleanUp() {
    Reference<?> ref;
    while ((ref = mRefQueue.poll()) != null) {
        mGarbage.remove(ref);
    }
}
```

 

检测内存泄漏方法是：每间隔15分钟统计一次1分钟之内没有被回收的对象个数，如果>=5个，则上报内存泄漏，需要开发者分析。

 

# 4.使用示例

原生示例：在通知移除时监测是否有内存泄漏的发生，具体可参考 NotificationEntryManager.removeNotificationInternal() 方法。

具体实践：使用测试应用发送通知，然后删除通知，间隔超过一分钟后，执行：adb shell dumpsys activity service SystemUIService | grep TrackedGarbage

TrackedGarbage:com.android.systemui.statusbar.notification.collection.NotificationEntry: 5 total, 5 old

发现有5个old对象（当old数>=5时会自动生成leak.hprof文件），点击“转储 SysUI 堆”快捷开关生成hprof文件，

导出后拖拽到AndroidStudio中，搜索“NotificationEnrty”，点击存疑的 instance，点击 References ，勾选 Show nearest GC root only ：

![img](/img/sysui/garbage.png)

可以分析出是 NotificationAlertController 的类成员遍历 mEntries 持有 NotificationEnrty 。

 

# 5.总结

GarbageMonitor 只能辅助开发人员主动监控内存大小和检测内存泄漏，适用于开发阶段。

从以往的经验来看，SystemUI相关的内存泄漏主要发生在：横竖屏转换，主题切换，屏幕刘海切换，新旧板子切换等场景，可以基于以上问题场景定制智能化监测方案。
