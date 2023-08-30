---
layout:     post
title:      "ANR治理-Binder调用统计"
subtitle:   ""
date:       2022-08-20 12:00:00
author:     "Yaku"
header-img: "img/post-bg-2015.jpg"
tags:
    - Android
    - ANR
    - Binder
---

# 1.基于adb命令统计binder请求

可以通过adb shell am trace-ipc可以检测binder的调用次数。

比如想知道下拉通知栏有哪些binder调用，先运行adb shell am trace-ipc start，下拉通知栏，在执行

adb shell am trace-ipc stop --dump-file /data/local/tmp/systemui.trace，查看dump出来的文件：

> Traces for process: com.android.systemui Count: 1 Trace: java.lang.Throwable at android.os.BinderProxy.transact(BinderProxy.java:486) at android.view.IWindowManager$Stub$Proxy.statusBarVisibilityChanged(IWindowManager.java:3795) at android.view.IWindowManagerCompat.statusBarVisibilityChanged(IWindowManagerCompat.java:50) at com.android.systemui.statusbar.phone.StatusBar.notifyUiVisibilityChanged(StatusBar.java:4867) at com.android.systemui.statusbar.phone.StatusBar.setSystemUiVisibility(StatusBar.java:4661) at com.android.systemui.statusbar.CommandQueue$H.handleMessage(CommandQueue.java:729) at android.os.Handler.dispatchMessage(Handler.java:107) at android.os.Looper.loop(Looper.java:221) at android.app.ActivityThread.main(ActivityThread.java:7520) at java.lang.reflect.Method.invoke(Native Method) at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:539) at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:950)

可以看到统计的所有进程的binder调用堆栈和次数，接下来从源码中看下是如何统计的。

1. 处理"trace-ipc"命令

```Java
// ActivityManagerShellCommand.java
int runTraceIpc(PrintWriter pw) throws RemoteException {
    String op = getNextArgRequired();
    if (op.equals("start")) {
        return runTraceIpcStart(pw);
    } else if (op.equals("stop")) {
        return runTraceIpcStop(pw);
    } else {
        getErrPrintWriter().println("Error: unknown trace ipc command '" + op + "'");
        return -1;
    }
}

int runTraceIpcStart(PrintWriter pw) throws RemoteException {
    pw.println("Starting IPC tracing.");
    pw.flush();
    mInterface.startBinderTracking();
    return 0;
}

int runTraceIpcStop(PrintWriter pw) throws RemoteException {
    final PrintWriter err = getErrPrintWriter();
    String opt;
    String filename = null;
    while ((opt=getNextOption()) != null) {
        if (opt.equals("--dump-file")) {
            filename = getNextArgRequired();
        } else {
            err.println("Error: Unknown option: " + opt);
            return -1;
        }
    }
    if (filename == null) {
        err.println("Error: Specify filename to dump logs to.");
        return -1;
    }

    // Writes an error message to stderr on failure
    ParcelFileDescriptor fd = openFileForSystem(filename, "w");
    if (fd == null) {
        return -1;
    }

    if (!mInterface.stopBinderTrackingAndDump(fd)) {
        err.println("STOP TRACE FAILED.");
        return -1;
    }

    pw.println("Stopped IPC tracing. Dumping logs to: " + filename);
    return 0;
}
```

从代码可以看出"trace-ipc"命令有两个参数，分别为"start"和"stop"，分别对应开始统计和结束统计写入到文件，这里的mInterface是ActivityManagerService。

1. AMS处理Binder统计

```Java
// ActivityManagerService.java
public boolean startBinderTracking() throws RemoteException {
    ...
    synchronized (mProcLock) {
        mBinderTransactionTrackingEnabled = true;
        mProcessList.forEachLruProcessesLOSP(true, process -> {
            final IApplicationThread thread = process.getThread();
            if (!processSanityChecksLPr(process, thread)) {
                return;
            }
            try {
                thread.startBinderTracking();
            } catch (RemoteException e) {
                Log.v(TAG, "Process disappared");
            }
        });
    }
    return true;
}

public boolean stopBinderTrackingAndDump(final ParcelFileDescriptor fd) throws RemoteException {
    ...
    boolean closeFd = true;
    try {
        synchronized (mProcLock) {
            if (fd == null) {
                throw new IllegalArgumentException("null fd");
            }
            mBinderTransactionTrackingEnabled = false;

            PrintWriter pw = new FastPrintWriter(new FileOutputStream(fd.getFileDescriptor()));
            pw.println("Binder transaction traces for all processes.\n");
            mProcessList.forEachLruProcessesLOSP(true, process -> {
                final IApplicationThread thread = process.getThread();
                if (!processSanityChecksLPr(process, thread)) {
                    return;
                }

                pw.println("Traces for process: " + process.processName);
                pw.flush();
                try {
                    TransferPipe tp = new TransferPipe();
                    try {
                        thread.stopBinderTrackingAndDump(tp.getWriteFd());
                        tp.go(fd.getFileDescriptor());
                    } finally {
                        tp.kill();
                    }
                } catch (IOException e) {
                    pw.println("Failure while dumping IPC traces from " + process +
                            ".  Exception: " + e);
                    pw.flush();
                } catch (RemoteException e) {
                    pw.println("Got a RemoteException while dumping IPC traces from " +
                            process + ".  Exception: " + e);
                    pw.flush();
                }
            });
            closeFd = false;
            return true;
        }
    } finally {
        ...
    }
}

private boolean processSanityChecksLPr(ProcessRecord process, IApplicationThread thread) {
    if (process == null || thread == null) {
        return false;
    }

    return Build.IS_DEBUGGABLE || process.isDebuggable();
}
```

AMS中会遍历存活的进程，通过IApplicationThread通知应用执行startBinderTracking/stopBinderTrackingAndDump。此外，这个功能只在root版本或者应用是Debug版本上生效。

1. 应用处理binder统计

```Java
// ActivityThread.java
private void handleStartBinderTracking() {
    Binder.enableStackTracking();
}

private void handleStopBinderTrackingAndDump(ParcelFileDescriptor fd) {
    try {
        Binder.disableStackTracking();
        Binder.getTransactionTracker().writeTracesToFile(fd);
    } finally {
        IoUtils.closeQuietly(fd);
        Binder.getTransactionTracker().clearTraces();
    }
}
```

应用收到请求后会直接调用对应的Binder静态方法。

1. Binder类中的逻辑

```Java
// Binder.java
public static void enableStackTracking() {
    sStackTrackingEnabled = true;
}

public static void disableStackTracking() {
    sStackTrackingEnabled = false;
}

public synchronized static TransactionTracker getTransactionTracker() {
    if (sTransactionTracker == null)
        sTransactionTracker = new TransactionTracker();
    return sTransactionTracker;
}

public static boolean isStackTrackingEnabled() {
    return sStackTrackingEnabled;
}
```

当应用向服务端发起binder通信时，会执行到BinderProxy的transact方法：

```Java
// BinderProxy.java
public boolean transact(int code, Parcel data, Parcel reply, int flags) throws RemoteException {
    Binder.checkParcel(this, code, data, "Unreasonably large binder buffer");
    ...
    final boolean tracingEnabled = Binder.isStackTrackingEnabled();
    if (tracingEnabled) {
        final Throwable tr = new Throwable();
        Binder.getTransactionTracker().addTrace(tr);
        StackTraceElement stackTraceElement = tr.getStackTrace()[1];
        Trace.traceBegin(Trace.TRACE_TAG_ALWAYS,
                stackTraceElement.getClassName() + "." + stackTraceElement.getMethodName());
    }
    ...
}
```

当tracingEnabled为true时，会抓取堆栈并记录到TransactionTracker中：

```Java
// TransactionTracker.java
private Map<String, Long> mTraces;

private void resetTraces() {
    synchronized (this) {
        mTraces = new HashMap<String, Long>();
    }
}

public void addTrace(Throwable tr) {
    String trace = Log.getStackTraceString(tr);
    synchronized (this) {
        if (mTraces.containsKey(trace)) {
            mTraces.put(trace, mTraces.get(trace) + 1);
        } else {
            mTraces.put(trace, Long.valueOf(1));
        }
    }
}
```

1. 写入到文件

```Java
 // TransactionTracker.java
 public void writeTracesToFile(ParcelFileDescriptor fd) {
    if (mTraces.isEmpty()) {
        return;
    }

    PrintWriter pw = new FastPrintWriter(new FileOutputStream(fd.getFileDescriptor()));
    synchronized (this) {
        for (String trace : mTraces.keySet()) {
            pw.println("Count: " + mTraces.get(trace));
            pw.println("Trace: " + trace);
            pw.println();
        }
    }
    pw.flush();
}
```

# 2.基于Binder接口统计binder请求

在查看BinderProxy源码时，发现Binder提供了Binder.ProxyTransactListener接口，可以供应用侧统计binder调用信息：

```Java
public boolean transact(int code, Parcel data, Parcel reply, int flags) throws RemoteException {
    Binder.checkParcel(this, code, data, "Unreasonably large binder buffer");
    ...
    // Make sure the listener won't change while processing a transaction.
    final Binder.ProxyTransactListener transactListener = sTransactListener;
    Object session = null;

    if (transactListener != null) {
        final int origWorkSourceUid = Binder.getCallingWorkSourceUid();
        session = transactListener.onTransactStarted(this, code, flags);

        // Allow the listener to update the work source uid. We need to update the request
        // header if the uid is updated.
        final int updatedWorkSourceUid = Binder.getCallingWorkSourceUid();
        if (origWorkSourceUid != updatedWorkSourceUid) {
            data.replaceCallingWorkSourceUid(updatedWorkSourceUid);
        }
    }

    final AppOpsManager.PausedNotedAppOpsCollection prevCollection =
            AppOpsManager.pauseNotedAppOpsCollection();

    if ((flags & FLAG_ONEWAY) == 0 && AppOpsManager.isListeningForOpNoted()) {
        flags |= FLAG_COLLECT_NOTED_APP_OPS;
    }

    try {
        final boolean result = transactNative(code, data, reply, flags);

        if (reply != null && !warnOnBlocking) {
            reply.addFlags(Parcel.FLAG_IS_REPLY_FROM_BLOCKING_ALLOWED_OBJECT);
        }

        return result;
    } finally {
        AppOpsManager.resumeNotedAppOpsCollection(prevCollection);

        if (transactListener != null) {
            transactListener.onTransactEnded(session);
        }

        if (tracingEnabled) {
            Trace.traceEnd(Trace.TRACE_TAG_ALWAYS);
        }
    }
}
```

ProxyTransactListener接口提供了binder请求开始和结束的回调：

```Java
public interface ProxyTransactListener {
    /**
     * Called before onTransact.
     *
     * @return an object that will be passed back to {@link #onTransactEnded} (or null).,
     *
     * @hide
     */
    @Nullable
    default Object onTransactStarted(@NonNull IBinder binder, int transactionCode, int flags) {
        return onTransactStarted(binder, transactionCode);
    }

    /**
     * Called before onTransact.
     *
     * @return an object that will be passed back to {@link #onTransactEnded} (or null).
     */
    @Nullable
    Object onTransactStarted(@NonNull IBinder binder, int transactionCode);

    /**
     * Called after onTransact (even when an exception is thrown).
     *
     * @param session The object return by {@link #onTransactStarted}.
     */
    void onTransactEnded(@Nullable Object session);
}
```

应用可以直接通过Binder的setProxyTransactListener方法来添加回调：

```Java
// Binder.java
public static void setProxyTransactListener(@Nullable ProxyTransactListener listener) {
    BinderProxy.setTransactListener(listener);
}

// BinderProxy.java
public static void setTransactListener(@Nullable Binder.ProxyTransactListener listener) {
    sTransactListener = listener;
}
```

# 3.基于Binder接口统计binder应答

以上两种方式统计的应用作为客户端向服务端发起的binder调用请求，如何统计应用作为服务端应答请求信息呢？

在Binder类中提供了一个接口：

```Java
// Binder.java
public static void setObserver(@Nullable BinderInternal.Observer observer) {
    sObserver = observer;
}

// BinderInternal.java
public interface Observer {
    /**
     * Called when a binder call starts.
     *
     * @return a CallSession to pass to the callEnded method.
     */
    CallSession callStarted(Binder binder, int code, int workSourceUid);

    /**
     * Called when a binder call stops.
     *
     * <li>This method will be called even when an exception is thrown by the binder stub
     * implementation.
     */
    void callEnded(CallSession s, int parcelRequestSize, int parcelReplySize,
            int workSourceUid);

    /**
     * Called if an exception is thrown while executing the binder transaction.
     *
     * <li>BinderCallsStats#callEnded will be called afterwards.
     * <li>Do not throw an exception in this method, it will swallow the original exception
     * thrown by the binder transaction.
     */
    public void callThrewException(CallSession s, Exception exception);
}
```

当服务端处理Binder请求时会调用到Binder类的execTransact方法：

```Java
private boolean execTransact(int code, long dataObj, long replyObj,
        int flags) {
    final int callingUid = Binder.getCallingUid();
    final long origWorkSource = ThreadLocalWorkSource.setUid(callingUid);
    try {
        return execTransactInternal(code, dataObj, replyObj, flags, callingUid);
    } finally {
        ThreadLocalWorkSource.restore(origWorkSource);
    }
}

private boolean execTransactInternal(int code, long dataObj, long replyObj, int flags,
        int callingUid) {
    // Make sure the observer won't change while processing a transaction.
    final BinderInternal.Observer observer = sObserver;
    final CallSession callSession =
            observer != null ? observer.callStarted(this, code, UNSET_WORKSOURCE) : null;
    Parcel data = Parcel.obtain(dataObj);
    Parcel reply = Parcel.obtain(replyObj);

    boolean res;
    final boolean tracingEnabled = Trace.isTagEnabled(Trace.TRACE_TAG_AIDL) &&
            (Binder.isStackTrackingEnabled() || Binder.isTracingEnabled(callingUid));
    try {
        final BinderCallHeavyHitterWatcher heavyHitterWatcher = sHeavyHitterWatcher;
        if (heavyHitterWatcher != null) {
            // Notify the heavy hitter watcher, if it's enabled.
            heavyHitterWatcher.onTransaction(callingUid, getClass(), code);
        }
        if (tracingEnabled) {
            Trace.traceBegin(Trace.TRACE_TAG_AIDL, getTransactionTraceName(code));
        }

        if ((flags & FLAG_COLLECT_NOTED_APP_OPS) != 0) {
            AppOpsManager.startNotedAppOpsCollection(callingUid);
            try {
                res = onTransact(code, data, reply, flags);
            } finally {
                AppOpsManager.finishNotedAppOpsCollection();
            }
        } else {
            res = onTransact(code, data, reply, flags);
        }
    } catch (RemoteException|RuntimeException e) {
        if (observer != null) {
            observer.callThrewException(callSession, e);
        }
        if (LOG_RUNTIME_EXCEPTION) {
            Log.w(TAG, "Caught a RuntimeException from the binder stub implementation.", e);
        }
        if ((flags & FLAG_ONEWAY) != 0) {
            if (e instanceof RemoteException) {
                Log.w(TAG, "Binder call failed.", e);
            } else {
                Log.w(TAG, "Caught a RuntimeException from the binder stub implementation.", e);
            }
        } else {
            // Clear the parcel before writing the exception.
            reply.setDataSize(0);
            reply.setDataPosition(0);
            reply.writeException(e);
        }
        res = true;
    } finally {
        if (tracingEnabled) {
            Trace.traceEnd(Trace.TRACE_TAG_AIDL);
        }
        if (observer != null) {
            // The parcel RPC headers have been called during onTransact so we can now access
            // the worksource UID from the parcel.
            final int workSourceUid = sWorkSourceProvider.resolveWorkSourceUid(
                    data.readCallingWorkSourceUid());
            observer.callEnded(callSession, data.dataSize(), reply.dataSize(), workSourceUid);
        }

        checkParcel(this, code, reply, "Unreasonably large binder reply buffer");
        reply.recycle();
        data.recycle();
    }

    StrictMode.clearGatheredViolations();
    return res;
}
```

 

 