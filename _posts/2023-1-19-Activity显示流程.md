---
layout:     post
title:      "Activity显示流程"
subtitle:   ""
date:       2023-1-19 12:00:00
author:     "Yaku"
header-img: "img/post-bg-2015.jpg"
tags:
    - Android
    - Activity
---


在 [Activity启动流程](https://dreamsunny.github.io/2023/01/09/Activity%E5%90%AF%E5%8A%A8%E6%B5%81%E7%A8%8B/) 中，我们知道了系统侧响应 Activity 启动请求会创建 Task ；

在 [DisplayArea树层级结构](https://dreamsunny.github.io/2022/12/24/DisplayArea%E6%A0%91%E5%B1%82%E7%BA%A7%E7%BB%93%E6%9E%84/)中，我们了解了 Task 最终会添加到 DefaultTaskDisplayArea ；

本文主要梳理下Activity显示流程。

# 1.Task添加到DefaultTaskDisplayArea

调用堆栈如下：

![trace1](/img/Activity/trace1.png)

```Java
// TaskDisplayArea.java
Task getOrCreateRootTask(..) {
    ...
    return new Task.Builder(mAtmService)
            ...
            .setParent(this)
            ...
            .build();
}
```

这里 setParent(this) 传入的是 TaskDisplayArea 。

```Java
// Task.java
Task build() {
    ...
    final Task task = buildInner();
    ...
    if (mParent != null) {
        if (mParent instanceof Task) {
            final Task parentTask = (Task) mParent;
            parentTask.addChild(task, mOnTop ? POSITION_TOP : POSITION_BOTTOM,
                    (mActivityInfo.flags & FLAG_SHOW_FOR_ALL_USERS) != 0);
        } else {
            mParent.addChild(task, mOnTop ? POSITION_TOP : POSITION_BOTTOM);
        }
    }
    ...
    return task;
}
```

在 Task.Builder的build() 方法中，将创建的task添加到 TaskDisplayArea。

# 2.ActivityRecord添加到Task

调用堆栈如下：

![trace2](/img/Activity/trace2.png)

1. 创建ActivityRecord：

```Java
// ActivityStarter.java
private int executeRequest(Request request) {
    ...
    final ActivityRecord r = new ActivityRecord.Builder(mService)
            ...
            .build();
    ...
}
// ActivityRecord.java
private ActivityRecord(..) {
    super(_service.mWindowManager, new Token(), TYPE_APPLICATION, true,
            null /* displayContent */, false /* ownerCanManageAppTokens */);
    ((Token) token).mActivityRef = new WeakReference<>(this);
}

private static class Token extends Binder {
    @NonNull WeakReference<ActivityRecord> mActivityRef;

    @Override
    public String toString() {
        return "Token{" + Integer.toHexString(System.identityHashCode(this)) + " "
                + mActivityRef.get() + "}";
    }
}
```

1. 添加到 Task 中：

```Java
// ActivityStarter.java
private void addOrReparentStartingActivity(@NonNull Task task, String reason) {
    ..
    TaskFragment newParent = task;
    ...
    if (mStartActivity.getTaskFragment() == null
            || mStartActivity.getTaskFragment() == newParent) {
        newParent.addChild(mStartActivity, POSITION_TOP);
    } else {
        mStartActivity.reparent(newParent, newParent.getChildCount() /* top */, reason);
    }
}
```

1. 添加到 DisplayContent 的 mTokenMap 中：

```Java
// WindowToken.java
void onDisplayChanged(DisplayContent dc) {
    dc.reParentWindowToken(this);
    super.onDisplayChanged(dc);
}
// DisplayContent.java
void reParentWindowToken(WindowToken token) {
    ...
    addWindowToken(token.token, token);
    ...
}

void addWindowToken(IBinder binder, WindowToken token) {
    ...
    mTokenMap.put(binder, token);
    ...
}
```

# 3.Activity窗口添加到WMS

![act_show](/img/Activity/act_show.png)

## 3.1 Activity.setContentView(..)

```Java
// Activity.java
public void setContentView(@LayoutRes int layoutResID) {
    getWindow().setContentView(layoutResID);
    initWindowDecorActionBar();
}
```

Activity.setContentView(..) 中直接调用 PhoneWindow.setContentView(..) 方法。

```Java
// PhoneWindow.java
public void setContentView(int layoutResID) {
    if (mContentParent == null) {
        installDecor();
    } else if (!hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
        ...
    }

    if (hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
        ...
    } else {
        mLayoutInflater.inflate(layoutResID, mContentParent);
    }
    ...
}

private void installDecor() {
    mForceDecorInstall = false;
    if (mDecor == null) {
        mDecor = generateDecor(-1);
        ...
    } else {
        mDecor.setWindow(this);
    }
    if (mContentParent == null) {
        mContentParent = generateLayout(mDecor);
        ...
    }
}

protected DecorView generateDecor(int featureId) {
    ...
    return new DecorView(context, featureId, this, getAttributes());
}

protected ViewGroup generateLayout(DecorView decor) {
    ...
    int layoutResource;
    int features = getLocalFeatures();
    if ((features & ((1 << FEATURE_LEFT_ICON) | (1 << FEATURE_RIGHT_ICON))) != 0) {
        ..
    } else if ((features & ((1 << FEATURE_PROGRESS) | (1 << FEATURE_INDETERMINATE_PROGRESS))) != 0
        && (features & (1 << FEATURE_ACTION_BAR)) == 0) {
        ..
    } else if ((features & (1 << FEATURE_CUSTOM_TITLE)) != 0) {
        ..
    } else if ((features & (1 << FEATURE_NO_TITLE)) == 0) {
        ..
    } else if ((features & (1 << FEATURE_ACTION_MODE_OVERLAY)) != 0) {
        ..
    } else {
        layoutResource = R.layout.screen_simple;
    }
    mDecor.onResourcesLoaded(mLayoutInflater, layoutResource);
    ViewGroup contentParent = (ViewGroup)findViewById(ID_ANDROID_CONTENT);
    ...
    return contentParent;
}
```

PhoneWindow.setContentView(..) 方法主要做3件事：

1. 创建DecorView。
2. inflate Activity 根布局。
3. inflate Activity 视图布局（setContentView()传入的layoutResID）。

其中系统预置 Activity 根布局有：

| feature                                                      | 布局                                  |
| ------------------------------------------------------------ | ------------------------------------- |
| *FEATURE_LEFT_ICON* <br> *FEATURE_RIGHT_ICON*                | screen_title_icons.xml                |
| *FEATURE_PROGRESS* <br/> *FEATURE_INDETERMINATE_PROGRESS* <br/> *FEATURE_ACTION_BAR* | screen_progress.xml                   |
| *FEATURE_CUSTOM_TITLE*                                       | screen_custom_title.xml               |
| *FEATURE_NO_TITLE*                                           | screen_title.xml                      |
| *FEATURE_ACTION_MODE_OVERLAY*                                | screen_simple_overlay_action_mode.xml |
| else                                                         | screen_simple.xml                     |

以 screen_simple.xml 为例查看其布局：

```Java
// screen_simple.xml
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:fitsSystemWindows="true"
    android:orientation="vertical">
    <ViewStub android:id="@+id/action_mode_bar_stub"
              android:inflatedId="@+id/action_mode_bar"
              android:layout="@layout/action_mode_bar"
              android:layout_width="match_parent"
              android:layout_height="wrap_content"
              android:theme="?attr/actionBarTheme" />
    <FrameLayout
         android:id="@android:id/content"
         android:layout_width="match_parent"
         android:layout_height="match_parent"
         android:foregroundInsidePadding="false"
         android:foregroundGravity="fill_horizontal|top"
         android:foreground="?android:attr/windowContentOverlay" />
</LinearLayout>
```

对应整个 Activity 视图为：

![decor](/img/Activity/decor.png)

最终，Activity.setContentView(..) 传入的布局会添加到 FrameLayout(id=ID_ANDROID_CONTENT) 。

## 3.2 ViewRootImpl.setView(..)

```Java
// ViewRootImpl.java
public ViewRootImpl(Context context, Display display) {
    this(context, display, WindowManagerGlobal.getWindowSession(),
            false /* useSfChoreographer */);
}

public ViewRootImpl(@UiContext Context context, Display display, IWindowSession session) {
    this(context, display, session, false /* useSfChoreographer */);
}

public ViewRootImpl(@UiContext Context context, Display display, IWindowSession session,
        boolean useSfChoreographer) {
    mWindowSession = session;
    mWindow = new W(this);
    ...
}

public void setView(View view, WindowManager.LayoutParams attrs, View panelParentView,
        int userId, Bundle bundle) {
    synchronized (this) {
        if (mView == null) {
            ...
            try {
                res = mWindowSession.addToDisplayAsUser(mWindow, mWindowAttributes,
                        getHostVisibility(), mDisplay.getDisplayId(), userId,
                        mInsetsController.getRequestedVisibilities(), inputChannel, mTempInsets,
                        mTempControls);
            } catch (RemoteException e) {
            } finally {
            }
            ...
        }
    }
}
// WindowManagerGlobal.java
public static IWindowSession getWindowSession() {
    synchronized (WindowManagerGlobal.class) {
        if (sWindowSession == null) {
            try {
                IWindowManager windowManager = getWindowManagerService();
                sWindowSession = windowManager.openSession(
                        new IWindowSessionCallback.Stub() {
                            @Override
                            public void onAnimatorScaleChanged(float scale) {
                                ValueAnimator.setDurationScale(scale);
                            }
                        });
            } catch (RemoteException e) {
                throw e.rethrowFromSystemServer();
            }
        }
        return sWindowSession;
    }
}
```

ViewRootImpl 是 View视图 与 WindowManager 的纽带，mIWindowSession、mWindow 为 Binder 对象，用于 APP 端与 WMS 之间的相互通信。

![win_binder](/img/Activity/win_binder.png)

这里通过 IWindowSession.addToDisplayAsUser(..) 接口，请求WMS添加新窗口。

此外，ViewRootImpl 管理整个 View 视图的绘制和 Input 事件分发。

# 4.WMS添加窗口

```Java
// WindowManagerService.java
public int addWindow(Session session, IWindow client, LayoutParams attrs, ..) {
    WindowState parentWindow = null;
    final int type = attrs.type;
    synchronized (mGlobalLock) {
        final DisplayContent displayContent = getDisplayContentOrCreate(displayId, attrs.token);
        
        if (type >= FIRST_SUB_WINDOW && type <= LAST_SUB_WINDOW) {
            parentWindow = windowForClientLocked(null, attrs.token, false);
            ...
        }

        ActivityRecord activity = null;
        final boolean hasParent = parentWindow != null;
        WindowToken token = displayContent.getWindowToken(
                hasParent ? parentWindow.mAttrs.token : attrs.token);
        final int rootType = hasParent ? parentWindow.mAttrs.type : type;

        if (token == null) {
            ...
        } else if (rootType >= FIRST_APPLICATION_WINDOW
                && rootType <= LAST_APPLICATION_WINDOW) {
            activity = token.asActivityRecord();
            ...
        } else if (rootType == TYPE_INPUT_METHOD) {
            ...
        } else if (rootType == TYPE_VOICE_INTERACTION) {
            ...
        } else if (rootType == TYPE_WALLPAPER) {
            ...
        } else if (rootType == TYPE_ACCESSIBILITY_OVERLAY) {
            ...
        } else if (type == TYPE_TOAST) {
            ...
        } else if (type == TYPE_QS_DIALOG) {
            ...
        } else if (token.asActivityRecord() != null) {
            ...
        }

        final WindowState win = new WindowState(this, session, client, token, parentWindow,
                appOp[0], attrs, viewVisibility, session.mUid, userId,
                session.mCanAddInternalSystemWindow);
        win.attach();
        mWindowMap.put(client.asBinder(), win);
        win.mToken.addWindow(win);

        if (type == TYPE_APPLICATION_STARTING && activity != null) {
            activity.attachStartingWindow(win);
        } else if (type == TYPE_INPUT_METHOD
            ...
        } else if (type == TYPE_INPUT_METHOD_DIALOG) {
            ...
        } else {
            ...
        }
    }
    return res;
}
```

## 4.1 创建WindowState

```Java
// WindowManagerService.java
public int addWindow(...) {
    ...
    WindowToken token = displayContent.getWindowToken(
            hasParent ? parentWindow.mAttrs.token : attrs.token);
    ...
}
```

这里传入的 token 是在创建 ActivityRecord 时创建的 Binder 对象（见上文2），获取到的 WindowToken 为 ActivityRecord 对象。

WMS每一个新窗口都会对应创建一个 WindowState 对象：

```Java
// WindowManagerService.java
public int addWindow(...) {
    ...
    final WindowState win = new WindowState(this, session, client, token, parentWindow,
            appOp[0], attrs, viewVisibility, session.mUid, userId,
            session.mCanAddInternalSystemWindow);
    ...
}
```

以 Activity 页面弹出一个Dialog对话框为例，WindowToken 与 WindowState 的关系为：

![win_token](/img/Activity/win_token.png)

## 4.2 创建SurfaceSession

```Java
// WindowManagerService.java
public int addWindow(...) {
    ...
    synchronized (mGlobalLock) {
        ...
        final WindowState win = new WindowState(...);
        win.attach();
        ...
    }
    return res;
}

// WindowState.java
void attach() {
    mSession.windowAddedLocked();
}

// Session.java
void windowAddedLocked() {
    ...
    if (mSurfaceSession == null) {
        mSurfaceSession = new SurfaceSession();
        ...
    }
}

// SurfaceSession.java
/** Create a new connection with the surface flinger. */
@UnsupportedAppUsage
public SurfaceSession() {
    mNativeClient = nativeCreate();
}

// android_view_SurfaceSession.cpp
static jlong nativeCreate(JNIEnv* env, jclass clazz) {
    SurfaceComposerClient* client = new SurfaceComposerClient();
    client->incStrong((void*)nativeCreate);
    return reinterpret_cast<jlong>(client);
}
```

SurfaceSession 构造方法中通过 nativeCreate() 方法返回了一个 SurfaceComposerClient 指针，它表示一个跟 SurfaceFlinger 的连接，当其第一次被使用时会调用 onFirstRef() 方法，创建一个实现 ISurfaceComposerClient 接口的 Client 对象。

```Java
// SurfaceComposerClient.cpp
void SurfaceComposerClient::onFirstRef() {
    sp<ISurfaceComposer> sf(ComposerService::getComposerService());
    if (sf != nullptr && mStatus == NO_INIT) {
        sp<ISurfaceComposerClient> conn;
        conn = sf->createConnection();
        if (conn != nullptr) {
            mClient = conn;
            mStatus = NO_ERROR;
        }
    }
}

// SurfaceFlinger.cpp
sp<ISurfaceComposerClient> SurfaceFlinger::createConnection() {
    const sp<Client> client = new Client(this);
    return client->initCheck() == NO_ERROR ? client : nullptr;
}
```

WMS 创建了一个 WindowState 对象表示客户端的一个 Window，接着调用 WindowState.attach() 方法创建了一个 SurfaceSession 对象，SurfaceSession 表示一个跟 SurfaceFlinger 的连接，它创建了一个 SurfaceComposerClient 对象，然后 SurfaceFlinger 又创建了一个 Client 对象。

## 4.3 WindowState添加到ActivityRecord

```Java
// WindowManagerService.java
public int addWindow(...) {
    ...
    final WindowState win = new WindowState(.., token, ..);
    ...
    win.mToken.addWindow(win);
    ...
}
```

# 5.应用绘制

## 5.1 创建Surface

```Java
// ViewRootImpl.java
public final Surface mSurface = new Surface();
private final SurfaceControl mSurfaceControl = new SurfaceControl();
private BLASTBufferQueue mBlastBufferQueue;
```

创建 ViewRootImp 实例时，会创建一个空的 Surface 和 SurfaceControl 对象，其初始化流程如下：

1. ViewRootImpl$TraversalRunnable.run(..)
2. ViewRootImpl.doTraversal(..)
3. ViewRootImpl.performTraversals(..)
4. ViewRootImpl.relayoutWindow(..)
5. Session.relayout(.., mSurfaceControl, ..)
6. WindowManagerService.relayoutWindow(.., outSurfaceControl, ..)
7. WindowManagerService.createSurfaceControl(outSurfaceControl, ..)
8. WindowStateAnimator.createSurfaceLocked()
9. new WindowSurfaceController(..)
10. SurfaceControl.Builder.build()
11. new SurfaceControl(..)
12. SurfaceControl.nativeCreate(..)

```Java
// android_view_SurfaceControl.cpp
static jlong nativeCreate(..) {
    sp<SurfaceComposerClient> client;
    if (sessionObj != NULL) {
        client = android_view_SurfaceSession_getClient(env, sessionObj);
    } else {
        client = SurfaceComposerClient::getDefault();
    }

    sp<SurfaceControl> surface;
    status_t err = client->createSurfaceChecked(String8(name.c_str()), w, h, format, &surface,
                                                flags, parentHandle, std::move(metadata));
    surface->incStrong((void *)nativeCreate);
    return reinterpret_cast<jlong>(surface.get());
}

// SurfaceComposerClient.cpp 
status_t SurfaceComposerClient::createSurfaceChecked(..) {
    status_t err = mStatus;

    if (mStatus == NO_ERROR) {
        sp<IBinder> handle;
        sp<IGraphicBufferProducer> gbp;

        err = mClient->createSurface(name, w, h, format, flags, parentHandle, std::move(metadata),
                                     &handle, &gbp, &id, &transformHint);

        if (err == NO_ERROR) {
            *outSurface =
                    new SurfaceControl(this, handle, gbp, id, w, h, format, transformHint, flags);
        }
    }
    return err;
}

// surfaceflinger/Client.cpp
status_t Client::createSurface(..) {
    LayerCreationArgs args(mFlinger.get(), this, name.c_str(), flags, std::move(metadata));
    return mFlinger->createLayer(args, outHandle, parentHandle, outLayerId, nullptr,
                                 outTransformHint);
}
```

1. WindowSurfaceController.getSurfaceControl(outSurfaceControl)
2. SurfaceControl.copyFrom(..)

```Java
outSurfaceControl.copyFrom(mSurfaceControl, "WindowSurfaceController.getSurfaceControl");
```

1. SurfaceControl.nativeCopyFromSurfaceControl(..)
2. ViewRootImpl.updateBlastSurfaceIfNeeded()
3. new BLASTBufferQueue(mTag, mSurfaceControl, ..)
4. BLASTBufferQueue.createSurface()
5. BLASTBufferQueue.nativeGetSurface(..)

```Java
// android_graphics_BLASTBufferQueue.cpp
static jobject nativeGetSurface(JNIEnv* env, jclass clazz, jlong ptr,
                                jboolean includeSurfaceControlHandle) {
    sp<BLASTBufferQueue> queue = reinterpret_cast<BLASTBufferQueue*>(ptr);
    return android_view_Surface_createFromSurface(env,
                                                  queue->getSurface(includeSurfaceControlHandle));
}

// BLASTBufferQueue.cpp
sp<Surface> BLASTBufferQueue::getSurface(bool includeSurfaceControlHandle) {
    std::unique_lock _lock{mMutex};
    sp<IBinder> scHandle = nullptr;
    if (includeSurfaceControlHandle && mSurfaceControl) {
        scHandle = mSurfaceControl->getHandle();
    }
    return new BBQSurface(mProducer, true, scHandle, this);
}
```

1. Surface.transferFrom(..)

```Java
mSurface.transferFrom(blastSurface);
```

1. Surface.setNativeObjectLocked(..)

在 Java 层中 ViewRootImpl 实例中持有一个 Surface 对象，该 Surface 对象中的 mNativeObject 属性指向 native 层中创建的 Surface 对象，native 层的 Surface 对应 SurfaceFlinger 中的 Layer 对象，它持有 Layer 中的 BufferQueueProducer 生产者指针，在 Surface 上绘制的内容最终会交由 SurfaceFlinger 来合成渲染送到显示器显示。

## 5.2 View绘制：构建阶段

![view_draw1](/img/Activity/view_draw1.png)

```Java
// Choreographer.java

private final class FrameDisplayEventReceiver extends DisplayEventReceiver
        implements Runnable {

    @Override
    public void onVsync(..) {
        try {
            ...
            Message msg = Message.obtain(mHandler, this);
            msg.setAsynchronous(true);
            mHandler.sendMessageAtTime(msg, timestampNanos / TimeUtils.NANOS_PER_MS);
        } finally {
        }
    }

    @Override
    public void run() {
        doFrame(mTimestampNanos, mFrame, mLastVsyncEventData);
    }
}

void doFrame(..) {
    try {
        ...
        doCallbacks(Choreographer.CALLBACK_INPUT, frameData, frameIntervalNanos);
        doCallbacks(Choreographer.CALLBACK_ANIMATION, frameData, frameIntervalNanos);
        doCallbacks(Choreographer.CALLBACK_INSETS_ANIMATION, frameData, frameIntervalNanos);
        doCallbacks(Choreographer.CALLBACK_TRAVERSAL, frameData, frameIntervalNanos);
        doCallbacks(Choreographer.CALLBACK_COMMIT, frameData, frameIntervalNanos);
    } finally {
    }
}
```

Choreographer中收到Vsync信号后，向主线程MessageQueue发送一条异步Message，当异步Message执行后会调用其doFrame(..)方法，依次执行 INPUT、ANIMATION、INSETS_ANIMATION、TRAVERSAL、COMMIT 回调。

在 TRAVERSAL 回调中会执行 mTraversalRunnable ，其 run() 方法中调用 doTraversal() 方法，执行 performTraversals() 方法，接着依次执行 View 的 measure、layout、draw 流程的代码。

## 5.3 View绘制：渲染阶段

RenderThread线程：

![view_draw2](/img/Activity/view_draw2.png)

渲染方式：软件绘制（CPU） VS 硬件绘制（GPU）

渲染引擎：OpenGL VS Vulkan

## 5.4 SurfaceFlinger：合成显示

![view_draw3](/img/Activity/view_draw3.png)

# 6.小结

应用侧 Window 是一个抽象概念，用来描述顶层视图的外观和行为，唯一的实现类是 PhoneWindow。

在创建 Activity / Dialog 时，会创建 PhoneWindow ，同时会创建 DecorView 。

应用侧向WMS请求添加视图时，会创建ViewRootImpl，同时会创建 Surface，视图绘制的数据会写入  Surface，由 SurfaceFlinger 合成显示。

WMS添加应用侧视图时会创建 WindowState 用来对应一个 Window，同时维护 Window 的 Z-Order 。

# 7.参考文档

[BLASTBufferQueue 详解](https://blog.simowce.com/all-about-blastbbq/)

[BBQ 机制介绍](https://www.jianshu.com/p/50a30fa6952e)

[BBQ 原理解读](https://www.jianshu.com/p/cdc60627df90)

[BBQ 运用场景](https://www.jianshu.com/p/384a5cd2e304)