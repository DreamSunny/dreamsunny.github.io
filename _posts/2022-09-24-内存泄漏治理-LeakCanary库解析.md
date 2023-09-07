---
layout:     post
title:      "内存泄漏治理-LeakCanary库解析"
subtitle:   ""
date:       2022-09-24 12:00:00
author:     "Yaku"
header-img: "img/post-bg-2015.jpg"
tags:
    - Android
    - MemoryLeak
    - LeakCanary
---

# 1.认识 LeakCanary

## 1.1 什么是内存泄漏

内存泄漏（Memory Leaks）指不再使用的对象或数据没有被回收，随着内存泄漏的堆积，应用性能会逐渐变差，甚至发生 OOM 奔溃。在 Android 应用中的内存泄漏可以分为 2 类：

- Java 内存泄露： 不再使用的对象被生命周期更长的 GC Root 引用，无法被判定为垃圾对象而导致内存泄漏（LeakCanary 只能监控 Java 内存泄漏）。
- Native 内存泄露： Native 内存没有垃圾回收机制，未手动回收导致内存泄漏。


## 1.2 为什么要使用 LeakCanary?

LeakCanray 是 Square 开源的 Java 内存泄漏分析工具，用于在实验室阶段检测 Android 应用中常见中的内存泄漏。

LeakCanary 的特点或优势在于提前预判出 Android 应用中最常见且影响较大的内存泄漏场景，并对此做针对性的监测手段。 这使得 LeakCanary 相比于其他排查内存泄漏的方案（如分析 OOM 异常时的堆栈日志、MAT 分析工具）更加高效。因为当内存泄漏堆积而内存不足时，应用可能从任何一次无关紧要的内存分配中抛出 OOM，堆栈日志只能体现最后一次内存分配的堆栈信息，而无法体现出导致发生 OOM 的主要原因。

目前，LeakCanary 支持以下五种 Android 场景中的内存泄漏监测：

1. 已销毁的 Activity 对象（进入 DESTROYED 状态）

2. 已销毁的 Fragment 对象和 Fragment View 对象（进入 DESTROYED 状态）

3. 已清除的的 ViewModel 对象（进入 CLEARED 状态）

4. 已销毁的的 Service 对象（进入 DESTROYED 状态）

5. 已从 WindowManager 中移除的 RootView 对象

   

# 2. LeakCanary监控内存泄漏原理

## 2.1 ActivityWatcher

通过 Application.registerActivityLifecycleCallbacks(…) 接口监听 Activity.onDestroy() 事件，将当前 Activity 对象交给 ObjectWatcher 监控：

```Kotlin
private val lifecycleCallbacks =
  object : Application.ActivityLifecycleCallbacks by noOpDelegate() {
    override fun onActivityDestroyed(activity: Activity) {
      reachabilityWatcher.expectWeaklyReachable(
        activity, "${activity::class.java.name} received Activity#onDestroy() callback"
      )
    }
  }
```

## 2.2 FragmentAndViewModelWatcher

通过 Application.registerActivityLifecycleCallbacks(…) 接口监听 Activity.onCreate() 事件，再通过 FragmentManager.registerFragmentLifecycleCallbacks(…) 接口监听 Fragment 的生命周期：

```Kotlin
private val fragmentLifecycleCallbacks = object : FragmentManager.FragmentLifecycleCallbacks() {

  override fun onFragmentCreated(
    fm: FragmentManager,
    fragment: Fragment,
    savedInstanceState: Bundle?
  ) {
    ViewModelClearedWatcher.install(fragment, reachabilityWatcher)
  }

  override fun onFragmentViewDestroyed(
    fm: FragmentManager,
    fragment: Fragment
  ) {
    val view = fragment.view
    if (view != null) {
      reachabilityWatcher.expectWeaklyReachable(
        view, "${fragment::class.java.name} received Fragment#onDestroyView() callback " +
        "(references to its views should be cleared to prevent leaks)"
      )
    }
  }

  override fun onFragmentDestroyed(
    fm: FragmentManager,
    fragment: Fragment
  ) {
    reachabilityWatcher.expectWeaklyReachable(
      fragment, "${fragment::class.java.name} received Fragment#onDestroy() callback"
    )
  }
}
```



由于 Android Framework 未提供设置 ViewModel.onClear() 全局监听的方法，所以 LeakCanary 是通过 Hook 的方式实现。即：在 Activity.onCreate() 和 Fragment.onCreate() 事件中实例化一个自定义ViewModel，在进入 ViewModel.onClear() 方法时，通过反射获取当前作用域中所有的 ViewModel 对象交给 ObjectWatcher 监控：

```Kotlin
override fun invoke(activity: Activity) {
  if (activity is FragmentActivity) {
    val supportFragmentManager = activity.supportFragmentManager
    supportFragmentManager.registerFragmentLifecycleCallbacks(fragmentLifecycleCallbacks, true)
    ViewModelClearedWatcher.install(activity, reachabilityWatcher)
  }
}
```



## 2.3 RootViewWatcher

由于 Android Framework 未提供设置全局监听 RootView 从 WindowManager 中移除的方法，所以 LeakCanary 是通过 Hook 的方式实现的，这一块是通过 squareup 另一个开源库 curtains 实现的。

RootView 监控这部分源码也比较复杂了，需要通过 2 步 Hook 来实现：

1. Hook WMS 服务内部的 WindowManagerGlobal.mViews RootView 列表，获取 RootView 新增和移除的时机
2. 检查 View 对应的 Window 类型，如果是 Dialog 或 DreamService 等类型，则在注册 View.addOnAttachStateChangeListener() 监听，在其中的 onViewDetachedFromWindow() 回调中将 View 对象交给 ObjectWatcher 监控

```Kotlin
private val listener = OnRootViewAddedListener { rootView ->
  val trackDetached = when(rootView.windowType) {
    PHONE_WINDOW -> {
      when (rootView.phoneWindow?.callback?.wrappedCallback) {
        // Activities are already tracked by ActivityWatcher
        is Activity -> false
        is Dialog -> {
          // Use app context resources to avoid NotFoundException
          // https://github.com/square/leakcanary/issues/2137
          val resources = rootView.context.applicationContext.resources
          resources.getBoolean(R.bool.leak_canary_watcher_watch_dismissed_dialogs)
        }
        // Probably a DreamService
        else -> true
      }
    }
    // Android widgets keep detached popup window instances around.
    POPUP_WINDOW -> false
    TOOLTIP, TOAST, UNKNOWN -> true
  }
  if (trackDetached) {
    rootView.addOnAttachStateChangeListener(object : OnAttachStateChangeListener {

      val watchDetachedView = Runnable {
        reachabilityWatcher.expectWeaklyReachable(
          rootView, "${rootView::class.java.name} received View#onDetachedFromWindow() callback"
        )
      }

      override fun onViewAttachedToWindow(v: View) {
        mainHandler.removeCallbacks(watchDetachedView)
      }

      override fun onViewDetachedFromWindow(v: View) {
        mainHandler.post(watchDetachedView)
      }
    })
  }
}
```

## 2.4 ServiceWatcher

由于 Android Framework 未提供设置 Service#onDestroy() 全局监听的方法，所以 LeakCanary 是通过 Hook 的方式实现的。

Service 监控这部分源码比较复杂了，需要通过 2 步 Hook 来实现：

1. Hook 主线程消息循环的 mH.mCallback 回调，监听其中的 STOP_SERVICE 消息，将即将 Destroy 的 Service 对象暂存起来（由于 ActivityThread.H 中没有 DESTROY_SERVICE 消息，所以不能直接监听到 onDestroy() 事件，需要第 2 步）
2. 使用动态代理 Hook AMS 与 App 通信的的 IActivityManager Binder 对象，代理其中的 serviceDoneExecuting() 方法，视为 Service.onDestroy() 的执行时机，拿到暂存的 Service 对象交给 ObjectWatcher 监控

```TypeScript
override fun install() {
  checkMainThread()
  check(uninstallActivityThreadHandlerCallback == null) {
    "ServiceWatcher already installed"
  }
  check(uninstallActivityManager == null) {
    "ServiceWatcher already installed"
  }
  try {
    swapActivityThreadHandlerCallback { mCallback ->
      uninstallActivityThreadHandlerCallback = {
        swapActivityThreadHandlerCallback {
          mCallback
        }
      }
      Handler.Callback { msg ->
        // https://github.com/square/leakcanary/issues/2114
        // On some Motorola devices (Moto E5 and G6), the msg.obj returns an ActivityClientRecord
        // instead of an IBinder. This crashes on a ClassCastException. Adding a type check
        // here to prevent the crash.
        if (msg.obj !is IBinder) {
          return@Callback false
        }

        if (msg.what == STOP_SERVICE) {
          val key = msg.obj as IBinder
          activityThreadServices[key]?.let {
            onServicePreDestroy(key, it)
          }
        }
        mCallback?.handleMessage(msg) ?: false
      }
    }
    swapActivityManager { activityManagerInterface, activityManagerInstance ->
      uninstallActivityManager = {
        swapActivityManager { _, _ ->
          activityManagerInstance
        }
      }
      Proxy.newProxyInstance(
        activityManagerInterface.classLoader, arrayOf(activityManagerInterface)
      ) { _, method, args ->
        if (METHOD_SERVICE_DONE_EXECUTING == method.name) {
          val token = args!![0] as IBinder
          if (servicesToBeDestroyed.containsKey(token)) {
            onServiceDestroyed(token)
          }
        }
        try {
          if (args == null) {
            method.invoke(activityManagerInstance)
          } else {
            method.invoke(activityManagerInstance, *args)
          }
        } catch (invocationException: InvocationTargetException) {
          throw invocationException.targetException
        }
      }
    }
  } catch (ignored: Throwable) {
    SharkLog.d(ignored) { "Could not watch destroyed services" }
  }
}
```