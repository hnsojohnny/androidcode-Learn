﻿# Activity启动流程

---

#### android基于java编写，首先先从`main`方法中分析：
在`ActivityThread`中找到`main`方法：
```
public static void main(String[] args) {
    ActivityThread thread = new ActivityThread();
    thread.attach(false, startSeq);
}
```
进入`attach`方法：
```
private void attach(boolean system, long startSeq) {
    if (!system) {
        final IActivityManager mgr = ActivityManager.getService();
            try {
                mgr.attachApplication(mAppThread, startSeq);
            } catch (RemoteException ex) {
                throw ex.rethrowFromSystemServer();
            }
            // Watch for getting close to heap limit.
            BinderInternal.addGcWatcher(new Runnable() {
                @Override public void run() {
            });
        } 
    }
```
因为从上个`mian`方法传过来的`system`为`false`，所以进入到上面的方法中。
`IActivityManager mgr = ActivityManager.getService()`返回`ActivityManagerService`。

#### 进入到`ActivityManagerService`类：
找到`attachApplication`方法：
```
@Override
    public final void attachApplication(IApplicationThread thread, long startSeq) {
        synchronized (this) {
            int callingPid = Binder.getCallingPid();
            final int callingUid = Binder.getCallingUid();
            final long origId = Binder.clearCallingIdentity();
            attachApplicationLocked(thread, callingPid, callingUid, startSeq);
            Binder.restoreCallingIdentity(origId);
        }
    }
```
然后再看`attachApplicationLocked`方法
```
private final boolean attachApplicationLocked(IApplicationThread thread,
            int pid, int callingUid, long startSeq) {
    if (app.isolatedEntryPoint != null) {
                // This is an isolated process which should just call an entry point instead of
                // being bound to an application.
                thread.runIsolatedEntryPoint(app.isolatedEntryPoint, app.isolatedEntryPointArgs);
            } else if (app.instr != null) {
                thread.bindApplication(processName, appInfo, providers,
                        app.instr.mClass,
                        profilerInfo, app.instr.mArguments,
                        app.instr.mWatcher,
                        app.instr.mUiAutomationConnection, testMode,
                        mBinderTransactionTrackingEnabled, enableTrackAllocation,
                        isRestrictedBackupMode || !normalMode, app.persistent,
                        new Configuration(getGlobalConfiguration()), app.compat,
                        getCommonServicesLocked(app.isolated),
                        mCoreSettingsObserver.getCoreSettingsLocked(),
                        buildSerial, isAutofillCompatEnabled);
            } else {
                thread.bindApplication(processName, appInfo, providers, null, profilerInfo,
                        null, null, null, testMode,
                        mBinderTransactionTrackingEnabled, enableTrackAllocation,
                        isRestrictedBackupMode || !normalMode, app.persistent,
                        new Configuration(getGlobalConfiguration()), app.compat,
                        getCommonServicesLocked(app.isolated),
                        mCoreSettingsObserver.getCoreSettingsLocked(),
                        buildSerial, isAutofillCompatEnabled);
            }
}
```
执行`ActivityThread`的`bindApplication`方法：
```
public final void bindApplication(String processName, ApplicationInfo appInfo,
                List<ProviderInfo> providers, ComponentName instrumentationName,
                ProfilerInfo profilerInfo, Bundle instrumentationArgs,
                IInstrumentationWatcher instrumentationWatcher,
                IUiAutomationConnection instrumentationUiConnection, int debugMode,
                boolean enableBinderTracking, boolean trackAllocation,
                boolean isRestrictedBackupMode, boolean persistent, Configuration config,
                CompatibilityInfo compatInfo, Map services, Bundle coreSettings,
                String buildSerial, boolean autofillCompatibilityEnabled) {
    AppBindData data = new AppBindData();
            data.processName = processName;
            data.appInfo = appInfo;
            data.providers = providers;
            data.instrumentationName = instrumentationName;
            data.instrumentationArgs = instrumentationArgs;
            data.instrumentationWatcher = instrumentationWatcher;
            data.instrumentationUiAutomationConnection = instrumentationUiConnection;
            data.debugMode = debugMode;
            data.enableBinderTracking = enableBinderTracking;
            data.trackAllocation = trackAllocation;
            data.restrictedBackupMode = isRestrictedBackupMode;
            data.persistent = persistent;
            data.config = config;
            data.compatInfo = compatInfo;
            data.initProfilerInfo = profilerInfo;
            data.buildSerial = buildSerial;
            data.autofillCompatibilityEnabled = autofillCompatibilityEnabled;
            sendMessage(H.BIND_APPLICATION, data);
}
```
把一些参数放到`data`中，然后发送一个`BIND_APPLICATION`的消息：
```
public void handleMessage(Message msg) {
    switch (msg.what) {
                case BIND_APPLICATION:
                    Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "bindApplication");
                    AppBindData data = (AppBindData)msg.obj;
                    handleBindApplication(data);
                    Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
                    break;
}
```
然后执行`handleBindApplication`方法：
```
private void handleBindApplication(AppBindData data) {
    app = data.info.makeApplication(data.restrictedBackupMode, null);
}
```
通过`LoadedApk`类的`makeApplication`方法来返回一个`application`实例，
```
public Application makeApplication(boolean forceDefaultAppClass,
            Instrumentation instrumentation) {
    app = mActivityThread.mInstrumentation.newApplication(
                    cl, appClass, appContext);
    appContext.setOuterContext(app);
    if (instrumentation != null) {
        try {
            instrumentation.callApplicationOnCreate(app);
            } catch (Exception e) {
                if (!instrumentation.onException(app, e)) {
                    Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
                    throw new RuntimeException(
                        "Unable to create application " + app.getClass().getName()
                        + ": " + e.toString(), e);
                }
            }
        }
    return app;
}
```
通过反射得到`application`实例并返回。
`Instrumentation`执行`callApplicationOnCreate`方法：
```
public void callApplicationOnCreate(Application app) {
        app.onCreate();
}
```
执行`application`的`oncreate`方法。

#### 再看`ActivityThread`的`handleLaunchActivity`方法：
```
public Activity handleLaunchActivity(ActivityClientRecord r,
            PendingTransactionActions pendingActions, Intent customIntent) {
        final Activity a = performLaunchActivity(r, customIntent);
}
```
进入到`performLaunchActivity`方法：
```
private Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent) {
     Activity activity = null;
        try {
            java.lang.ClassLoader cl = appContext.getClassLoader();
            activity = mInstrumentation.newActivity(
                    cl, component.getClassName(), r.intent);
            StrictMode.incrementExpectedActivityCount(activity.getClass());
            r.intent.setExtrasClassLoader(cl);
            r.intent.prepareToEnterProcess();
            if (r.state != null) {
                r.state.setClassLoader(cl);
            }
        }
        if (r.isPersistable()) {
                    mInstrumentation.callActivityOnCreate(activity, r.state, r.persistentState);
                } else {
                    mInstrumentation.callActivityOnCreate(activity, r.state);
                }
}
```
通过反射获取到`activity`的实例，然后进入`Instrumentation的callActivityOnCreate`方法：
```
public void callActivityOnCreate(Activity activity, Bundle icicle,
            PersistableBundle persistentState) {
        prePerformCreate(activity);
        activity.performCreate(icicle, persistentState);
        postPerformCreate(activity);
    }
```
查看`Activity`的`performCreate`方法：
```
final void performCreate(Bundle icicle, PersistableBundle persistentState) {
        mCanEnterPictureInPicture = true;
        restoreHasCurrentPermissionRequest(icicle);
        if (persistentState != null) {
            onCreate(icicle, persistentState);
        } else {
            onCreate(icicle);
        }
    
    }
```
最后执行`Activity`的`oncreate`方法。


