# Android 学习记录：系统启动（3）——SystemSever

顺序图：

![图片](/SystemServer.png)

书接上篇，从[Android启动（2）——Zygote](/系统启动（2）——Zygote.md) 的ZygoteInit的main函数中，在runSelectLoop之前，调用了forkSystemServer函数，这里作为启动ServerServer（system\_server）的起点。

Step1：进行参数组装，调用Zygote的forkSystemServer去Zygote的Native层fork子进程并pid，然后关闭子进程的socket，调用并返回handleSystemServerProcess

```java
private static Runnable forkSystemServer(String abiList, String socketName,
            ZygoteServer zygoteServer) {
        ...
        String[] args = {
                "--setuid=1000",
                "--setgid=1000",
                "--setgroups=1001,1002,1003,1004,1005,1006,1007,1008,1009,1010,1018,1021,1023,"
                        + "1024,1032,1065,3001,3002,3003,3005,3006,3007,3009,3010,3011,3012",
                "--capabilities=" + capabilities + "," + capabilities,
                "--nice-name=system_server",
                "--runtime-args",
                "--target-sdk-version=" + VMRuntime.SDK_VERSION_CUR_DEVELOPMENT,
         				 // 记住这里，后续用反射findStaticMain创建并调用main
                "com.android.server.SystemServer",
        };

        int pid;

        try {
            ...
            pid = Zygote.forkSystemServer(
                    parsedArgs.mUid, parsedArgs.mGid,
                    parsedArgs.mGids,
                    parsedArgs.mRuntimeFlags,
                    null,
                    parsedArgs.mPermittedCapabilities,
                    parsedArgs.mEffectiveCapabilities);
        } catch (IllegalArgumentException ex) {
           ...
        }

        /* For child process */
        if (pid == 0) {
            if (hasSecondZygote(abiList)) {
                waitForSecondaryZygote(socketName);
            }

            zygoteServer.closeServerSocket();
            return handleSystemServerProcess(parsedArgs);
        }

        return null;
    }

```

Step2：在handleSystemServerProcess中调用到zygoteInit，最终走到findStaticMain，上一篇幅[AOSP 学习：Android启动（2）——Zygote](/系统启动（2）——Zygote.md)有讲此函数根据传入路径反射创建类并调用它的main函数，即com.android.server.SystemServer的main。

Step3：在SystemServer的main函数中，只新建了SystemServer并调用它的run函数。在run函数中做了一系列初始化操作。

```java
public static void main(String[] args) {
        new SystemServer().run();
    }
```

Step4：run函数里做的事情很多，主要有prepareMainLooper、loadLibrary("android\_servers")、createSystemContext、新建SystemServiceManager，然后就是启动若干SystemService的几大方法：startBootstrapServices、startCoreServices、startOtherServices、startApexServices。即引导服务、核心服务、其他服务、Apex服务。最后开始Looper.loop()。

```java
private void run() {
        if (VMDebug.isDebuggingEnabled()
            ...
            Debug.waitForDebugger();
        }
            ...
        try {
            ...
            // Mmmmmm... more memory!
          	// gc
            VMRuntime.getRuntime().clearGrowthLimit();
         ...
           // mainLooper
            Looper.prepareMainLooper();
           ....
            // Prepare the thread pool for init tasks that can be parallelized
            SystemServerInitThreadPool.start();
         ...
            // Initialize native services.
            System.loadLibrary("android_servers");
           ....
            // Initialize the system context.
            createSystemContext();
          ....
            // Call per-process mainline module initialization.
            ActivityThread.initializeMainlineModules();
          ...
            // Create the system service manager.
            mSystemServiceManager = new SystemServiceManager(mSystemContext);
            mSystemServiceManager.setStartInfo(mRuntimeRestart,
                    mRuntimeStartElapsedTime, mRuntimeStartUptime);
            mDumper.addDumpable(mSystemServiceManager);
            LocalServices.addService(SystemServiceManager.class, mSystemServiceManager);
          ....
        } finally {
            ...
        }
     
        try {
            ...
              // 启动若干SystemServices
              // 引导服务
            startBootstrapServices(t);
          // 核心服务
            startCoreServices(t);
          // 其他服务
            startOtherServices(t);
          // Android Pony Express服务 可以自己搜一下Apex，新特性
            startApexServices(t);
          ....
        } catch (Throwable ex) {
          ...
        } finally {
           ...
        }
       ....
        // Loop forever.
        Looper.loop();
            ....
    }
```

Step5：startBootstrapServices开启看门狗，通过SystemServiceManager.startService开启ATMS与AMS、PMS等重要服务。

```java
private void startBootstrapServices(@NonNull TimingsTraceAndSlog t) {
       ...
        final Watchdog watchdog = Watchdog.getInstance();
        watchdog.start();
        ...
        // Activity manager runs the show.
        t.traceBegin("StartActivityManager");
        // TODO: Might need to move after migration to WM.
        ActivityTaskManagerService atm = mSystemServiceManager.startService(
                ActivityTaskManagerService.Lifecycle.class).getService();
        mActivityManagerService = ActivityManagerService.Lifecycle.startService(
                mSystemServiceManager, atm);
        mActivityManagerService.setSystemServiceManager(mSystemServiceManager);
        mActivityManagerService.setInstaller(installer);
        mWindowManagerGlobalLock = atm.getGlobalLock();
        t.traceEnd();

       
      ...
        t.traceBegin("StartUserManagerService");
        mUserManagerService = mSystemServiceManager
                .startService(UserManagerService.LifeCycle.class).getService();
        t.traceEnd();
...
        // Manages Resources packages
        t.traceBegin("StartResourcesManagerService");
        ResourcesManagerService resourcesService = new ResourcesManagerService(mSystemContext);
        resourcesService.setActivityManagerService(mActivityManagerService);
        mSystemServiceManager.startService(resourcesService);
        t.traceEnd();

       ...
    }

```

Step6：startCoreServices通过SystemServiceManager.startService开启电池、Gpu等重要服务。

Step7：startOtherServices通过SystemServiceManager.startService开启IMS、WMS（并调用main函数，再调用AMS的setWindowManager设置自身，最后调用onInitReady函数），后面调用若干服务的systemReady函数通知完成初始化，其中AMS的onSystemReady在最后.

```java
private void startOtherServices(@NonNull TimingsTraceAndSlog t) {
       ...
        try {
       
...
            t.traceBegin("StartWindowManagerService");
            // WMS needs sensor service ready
            mSystemServiceManager.startBootPhase(t, SystemService.PHASE_WAIT_FOR_SENSOR_SERVICE);
            wm = WindowManagerService.main(context, inputManager, !mFirstBoot,
                    new PhoneWindowManager(), mActivityManagerService.mActivityTaskManager);
            ServiceManager.addService(Context.WINDOW_SERVICE, wm, /* allowIsolated= */ false,
                    DUMP_FLAG_PRIORITY_CRITICAL | DUMP_FLAG_PRIORITY_HIGH
                            | DUMP_FLAG_PROTO);
            t.traceEnd();

            t.traceBegin("SetWindowManagerService");
            mActivityManagerService.setWindowManager(wm);
            t.traceEnd();

            t.traceBegin("WindowManagerServiceOnInitReady");
            wm.onInitReady();
            t.traceEnd();
            ...
            t.traceBegin("StartInputManager");
            inputManager.setWindowManagerCallbacks(wm.getInputManagerCallback());
            inputManager.start();
            t.traceEnd();
        ...
        } catch (Throwable e) {
           ...
        }
      ....
        // It is now time to start up the app processes...

        t.traceBegin("MakeLockSettingsServiceReady");
        if (lockSettings != null) {
            try {
                lockSettings.systemReady();
            } catch (Throwable e) {
                reportWtf("making Lock Settings Service ready", e);
            }
        }
        t.traceEnd();
       ...
        t.traceBegin("MakeWindowManagerServiceReady");
        try {
            wm.systemReady();
        } catch (Throwable e) {
            reportWtf("making Window Manager Service ready", e);
        }
        t.traceEnd();

        t.traceBegin("RegisterLogMteState");
        try {
            LogMteState.register(context);
        } catch (Throwable e) {
            reportWtf("RegisterLogMteState", e);
        }
        t.traceEnd();
      ...
        t.traceBegin("MakePackageManagerServiceReady");
        mPackageManagerService.systemReady();
        t.traceEnd();

        t.traceBegin("MakeDisplayManagerServiceReady");
        try {
            // TODO: use boot phase and communicate this flag some other way
            mDisplayManagerService.systemReady(safeMode);
        } catch (Throwable e) {
            reportWtf("making Display Manager Service ready", e);
        }
        t.traceEnd();

       ...
        // We now tell the activity manager it is okay to run third party
        // code.  It will call back into us once it has gotten to the state
        // where third party code can really run (but before it has actually
        // started launching the initial applications), for us to complete our
        // initialization.
        mActivityManagerService.systemReady(() -> {
           ...
            t.traceBegin("MakeVpnManagerServiceReady");
            try {
                if (vpnManagerF != null) {
                    vpnManagerF.systemReady();
                }
            } catch (Throwable e) {
                reportWtf("making VpnManagerService ready", e);
            }
            t.traceEnd();
          ...
            t.traceBegin("MakeTelephonyRegistryReady");
            try {
                if (telephonyRegistryF != null) {
                    telephonyRegistryF.systemRunning();
                }
            } catch (Throwable e) {
                reportWtf("Notifying TelephonyRegistry running", e);
            }
            t.traceEnd();
          ...
            if (mPackageManager.hasSystemFeature(PackageManager.FEATURE_TELEPHONY)) {
                t.traceBegin("MakeMmsServiceReady");
                try {
                    if (mmsServiceF != null) mmsServiceF.systemRunning();
                } catch (Throwable e) {
                    reportWtf("Notifying MmsService running", e);
                }
                t.traceEnd();
            }

....
        }, t);

     ...

        t.traceBegin("StartSystemUI");
        try {
          // 启动SystemUI 作为后续篇幅的入口点
            startSystemUi(context, windowManagerF);
        } catch (Throwable e) {
            reportWtf("starting System UI", e);
        }
  ...
    }

```

sTEP8:AMS的onSystemReady单独拿出来讲，比较重要，在回调中启动了很多偏应用层的服务例如电话、短信、Vpn等，并且onSystemReady本身也会做一些操作比如拉起保活应用、启动Launcher（见下一篇幅[AOSP 学习：Android启动（4）——启动首个应用（Launcher）](/系统启动（4）——Launcher.md) ）最后在AMS的onSystemReady执行完之后调用startSystemUi启动SystemUI。

```java
    public void systemReady(final Runnable goingCallback, @NonNull TimingsTraceAndSlog t) {
        t.traceBegin("PhaseActivityManagerReady");
        mSystemServiceManager.preSystemReady();
        synchronized(this) {
            if (mSystemReady) {
                if (goingCallback != null) {
                  // 允许传入的回调
                    goingCallback.run();
                }
             ...
                return;
            }
          ...
            mLocalDeviceIdleController =
                    LocalServices.getService(DeviceIdleInternal.class);
            mActivityTaskManager.onSystemReady();
            mUserController.onSystemReady();
            mAppOpsService.systemReady();
            mProcessList.onSystemReady();
            mAppRestrictionController.onSystemReady();
            mSystemReady = true;
          ...
        }
       ...
        synchronized (this) {
         ...
         // 启动配置了persistent=true的应用，即保活的
            startPersistentApps(PackageManager.MATCH_DIRECT_BOOT_AWARE);

            boolean isBootingSystemUser = currentUserId == UserHandle.USER_SYSTEM;
            ...
            if (isBootingSystemUser && !UserManager.isHeadlessSystemUserMode()) {
              // 启动Launcher
                mAtmInternal.startHomeOnAllDisplays(currentUserId, "systemReady");
            }
         ...
        }
    }

```

Step7：startApexServices从ApexManager获取声明Apex的SystemService列表并遍历执行SystemServiceManager.startService开启服务。

由此可见，想增加一个SystemService，需要继承基类，并按规矩重写这些函数，最后放在指定的位置。

```java
private void startApexServices(@NonNull TimingsTraceAndSlog t) {
        ...
        List<ApexSystemServiceInfo> services = ApexManager.getInstance().getApexSystemServices();
        for (ApexSystemServiceInfo info : services) {
            String name = info.getName();
            String jarPath = info.getJarPath();
          	...
            if (TextUtils.isEmpty(jarPath)) {
                mSystemServiceManager.startService(name);
            } else {
                mSystemServiceManager.startServiceFromJar(name, jarPath);
            }
            ...
        }
        // make sure no other services are started after this point
        mSystemServiceManager.sealStartedServices();
        ...
    }

```