# AOSP 学习：startActivty（1）

<span style="color:#080808;">startActivty（1）只到Application.onCreate，后续的start首个Activity在 </span>[AOSP 学习：startActivty（2）]()<span style="color:#080808;"> </span>

## <span style="color:#080808;">顺序图</span>

![图片](./SequenceDiagram1.png)

## <span style="color:#080808;">流程概述</span>

### <span style="color:#080808;">Step1</span>

<span style="color:#080808;">startActivity经过层层封装、传递、校验，到达ATMS的startActivityAsUser，在这里构建出ActivityStarter，并调用execute处理。</span>

```java
private int startActivityAsUser(IApplicationThread caller, String callingPackage,
            @Nullable String callingFeatureId, Intent intent, String resolvedType,
            IBinder resultTo, String resultWho, int requestCode, int startFlags,
            ProfilerInfo profilerInfo, Bundle bOptions, int userId, boolean validateIncomingUser) {
        mAmInternal.addCreatorToken(intent, callingPackage);
        final int callingPid = Binder.getCallingPid();
        final int callingUid = Binder.getCallingUid();
        final SafeActivityOptions opts = SafeActivityOptions.fromBundle(
                bOptions, callingPid, callingUid);

        ...

        return getActivityStartController().obtainStarter(intent, "startActivityAsUser")
                .setCaller(caller)
                .setCallingPackage(callingPackage)
                .setCallingFeatureId(callingFeatureId)
                .setResolvedType(resolvedType)
                .setResultTo(resultTo)
                .setResultWho(resultWho)
                .setRequestCode(requestCode)
                .setStartFlags(startFlags)
                .setProfilerInfo(profilerInfo)
                .setActivityOptions(opts)
                .setUserId(userId)
                .execute();
    }
```

### Step2

<span style="color:#080808;">execute流转到executeRequest，构建ActivityRecord，此类将在后续作为主要的传参，然后传递给startActivityUnchecked，流转到TaskFragment并调用它的resumeTopActivity，首先检查是否进程已经创建，如果没创建那么就调用startProcessAsync创建进程，然后return true。</span>

```java
final boolean resumeTopActivity(ActivityRecord prev, ActivityOptions options,
            boolean skipPause) {
        ...
        boolean pausing = !skipPause && taskDisplayArea.pauseBackTasks(next);
        ...
        if (pausing) {
            ...
            if (next.attachedToProcess()) {
               ...
            } else if (!next.isProcessRunning()) {
                final boolean isTop = this == taskDisplayArea.getFocusedRootTask();
                // 创建进程
                mAtmService.startProcessAsync(next, false /* knownToBeDead */, isTop,
                        isTop ? HostingRecord.HOSTING_TYPE_NEXT_TOP_ACTIVITY
                                : HostingRecord.HOSTING_TYPE_NEXT_ACTIVITY);
            }
            ...
            return true;
        } else if (mResumedActivity == next && next.isState(RESUMED)
                && taskDisplayArea.allResumedActivitiesComplete()) {
            ...
        }
        ...
        if (next.attachedToProcess()) {
            ...
        } else {
            ...
            // 创建Activity，下一篇幅的入口点
            mTaskSupervisor.startSpecificActivity(next, true, true);
        }
        return true;
    }
```

### Step3

<span style="color:#080808;">调用startProcessAsync创建进程，通过Handler发送消息调用到AMS的startProcess方法，再流转到ProcessList的startProcessLocked方法，在这里开始传递了一个String变量entryPoint，为"android.app.ActivityThread"，这里记住，后面反射调用的时候会用。</span>

```java
private Process.ProcessStartResult startProcess(HostingRecord hostingRecord, String entryPoint,
            ProcessRecord app, int uid, int[] gids, int runtimeFlags, int zygotePolicyFlags,
            int mountExternal, String seInfo, String requiredAbi, String instructionSet,
            String invokeWith, long startTime) {
        try {
            ...
            final Process.ProcessStartResult startResult;
            ...
            if (hostingRecord.usesWebviewZygote()) {
                ...
            } else if (hostingRecord.usesAppZygote()) {
                // 我们走这里 记得entryPoint是"android.app.ActivityThread"
                final AppZygote appZygote = createAppZygoteForProcessIfNeeded(app);
                startResult = appZygote.startProcess(entryPoint,
                        app.processName, uid, gids, runtimeFlags, mountExternal,
                        app.info.targetSdkVersion, seInfo, requiredAbi, instructionSet,
                        app.info.dataDir, app.info.packageName, isTopApp,
                        app.getDisabledCompatChanges(), pkgDataInfoMap,
                        allowlistedAppDataInfoMap, app.getStartSeq(),
                        new String[]{PROC_START_SEQ_IDENT + app.getStartSeq()});
            } else {
                ...
            }
            ...
            return startResult;
        } finally {
            ...
        }
    }

```

### <span style="color:#080808;">Step4</span>

<span style="color:#080808;">从startProcessLocked继续调用到Zygote的Java层，到ZygoteProcess的start，组装一些传递参数，最后调用到attemptZygoteSendArgs，在里面通过zygoteWriter的write和flush方法完成一次消息的发送，走socket通讯Zygote去fork进程。</span>

```java
private Process.ProcessStartResult startViaZygote(...) throws ZygoteStartFailedEx {
        ...

        // 组装参数列表
        ArrayList<String> argsForZygote = new ArrayList<>();

        argsForZygote.add("--runtime-args");
        argsForZygote.add("--setuid=" + uid);
        ...

        if (niceName != null) {
            argsForZygote.add("--nice-name=" + niceName);
        }

        if (seInfo != null) {
            argsForZygote.add("--seinfo=" + seInfo);
        }
        ...
        synchronized(mLock) {
            return zygoteSendArgsAndGetResult(openZygoteSocketIfNeeded(abi),
                                              zygotePolicyFlags,
                                              argsForZygote);
        }
    }
```

```java
private Process.ProcessStartResult attemptZygoteSendArgsAndGetResult(
            ZygoteState zygoteState, String msgStr) throws ZygoteStartFailedEx {
        try {
            final BufferedWriter zygoteWriter = zygoteState.mZygoteOutputWriter;
            final DataInputStream zygoteInputStream = zygoteState.mZygoteInputStream;

            // 发送上面组装的参数
            zygoteWriter.write(msgStr);
            zygoteWriter.flush();

            // 接收结果
            Process.ProcessStartResult result = new Process.ProcessStartResult();
            result.pid = zygoteInputStream.readInt();
            result.usingWrapper = zygoteInputStream.readBoolean();

            if (result.pid < 0) {
                throw new ZygoteStartFailedEx("fork() failed");
            }

            return result;
        } catch (IOException ex) {
            zygoteState.close();
            ...
            throw new ZygoteStartFailedEx(ex);
        }
    }
```

### Step5

Zygote这边，从Init进程到启动Zygote之后，就一直处于runSelectLoop，死循环轮询消息到达并处理（具体见Zygote篇幅 [AOSP 学习：Init进程到首次Zygote - 云文档]() ）。消息到达时调用acceptCommandPeer新建ZygoteConnction，然后调用processCommand处理，processCommand调用到native层的Zygote进行fork并返回pid到processCommand内，根据pid对子进程进行一些处理例如关闭Socket等，处理完成后继续调用childZygoteInit进而调用findStaticMain方法，按照传入的参数（还记得entryPoint吗，android.app.ActivityThread）反射创建ActivityThread并调用它的main方法。

```java
Runnable runSelectLoop(String abiList) {
        ...
        while (true) {
            ... 
            if (pollReturnValue == 0) {
                ...
            } else {
                ...
                while (--pollIndex >= 0) {
                    if ((pollFDs[pollIndex].revents & POLLIN) == 0) {
                        continue;
                    }

                    if (pollIndex == 0) {
                        // 消息到达，建立ZygoteConnection
                        ZygoteConnection newPeer = acceptCommandPeer(abiList);
                        // 添加到peers中
                        peers.add(newPeer);
                        socketFDs.add(newPeer.getFileDescriptor());
                    } else if (pollIndex < usapPoolEventFDIndex) {
                        try {
                            // 从peers得到ZygoteConnection，调用processCommand处理
                            ZygoteConnection connection = peers.get(pollIndex);
                            boolean multipleForksOK = !isUsapPoolEnabled()
                                    && ZygoteHooks.isIndefiniteThreadSuspensionSafe();
                            final Runnable command =
                                    connection.processCommand(this, multipleForksOK);

                            if (mIsForkChild) {
                                ...
                                return command;
                            } else {
                                ...
                               if (connection.isClosedByPeer()) {
                                    connection.closeSocket();
                                    peers.remove(pollIndex);
                                    socketFDs.remove(pollIndex);
                                }
                            }
                        } catch (Exception e) {
                            if (!mIsForkChild) {
                                ZygoteConnection conn = peers.remove(pollIndex);
                                conn.closeSocket();

                                socketFDs.remove(pollIndex);
                            } else {
                               ...
                            }
                        } finally {
                            mIsForkChild = false;
                        }
                    } else {
                       ...
                    }
                }
                ...
            }
          ...
        }
    }
```

```java
Runnable processCommand(ZygoteServer zygoteServer, boolean multipleOK) {
        ZygoteArguments parsedArgs;

        try (ZygoteCommandBuffer argBuffer = new ZygoteCommandBuffer(mSocket)) {
            while (true) {
                ...
                if (parsedArgs.mInvokeWith != null || parsedArgs.mStartChildZygote
                        || !multipleOK || peer.getUid() != Process.SYSTEM_UID) {
                    ...
                    // 最终调用到native层的zygote.fork
                    pid = Zygote.forkAndSpecialize(parsedArgs.mUid, parsedArgs.mGid,
                            parsedArgs.mGids, parsedArgs.mRuntimeFlags, rlimits,
                            parsedArgs.mMountExternal, parsedArgs.mSeInfo, parsedArgs.mNiceName,
                            fdsToClose, fdsToIgnore, parsedArgs.mStartChildZygote,
                            parsedArgs.mInstructionSet, parsedArgs.mAppDataDir,
                            parsedArgs.mIsTopApp, parsedArgs.mPkgDataInfoList,
                            parsedArgs.mAllowlistedDataInfoList, parsedArgs.mBindMountAppDataDirs,
                            parsedArgs.mBindMountAppStorageDirs,
                            parsedArgs.mBindMountSyspropOverrides);

                    try {
                        if (pid == 0) {
                            // in child
                            zygoteServer.setForkChild();

                            zygoteServer.closeServerSocket();
                            IoUtils.closeQuietly(serverPipeFd);
                            serverPipeFd = null;

                            return handleChildProc(parsedArgs, childPipeFd,
                                    parsedArgs.mStartChildZygote);
                        } else {
                            // In the parent. A pid < 0 indicates a failure and will be handled in
                            // handleParentProc.
                            IoUtils.closeQuietly(childPipeFd);
                            childPipeFd = null;
                            handleParentProc(pid, serverPipeFd);
                            return null;
                        }
                    } finally {
                        IoUtils.closeQuietly(childPipeFd);
                        IoUtils.closeQuietly(serverPipeFd);
                    }
                } else {
                    ...
                }
            }
        }
        ...
    }

```

```java
private Runnable handleChildProc(ZygoteArguments parsedArgs,
            FileDescriptor pipeFd, boolean isZygote) {
        
        closeSocket();
  			...
        if (parsedArgs.mInvokeWith != null) {
            ...
        } else {
            if (!isZygote) {
               // 这里是首次启动时走的，可以看Zygote启动的篇幅
                return ZygoteInit.zygoteInit(parsedArgs.mTargetSdkVersion,
                        parsedArgs.mDisabledCompatChanges,
                        parsedArgs.mRemainingArgs, null /* classLoader */);
            } else {
              // 本次走这里
                return ZygoteInit.childZygoteInit(
                        parsedArgs.mRemainingArgs  /* classLoader */);
            }
        }
    }
```

```java
static Runnable childZygoteInit(String[] argv) {
  	RuntimeInit.Arguments args = new RuntimeInit.Arguments(argv);
  	return RuntimeInit.findStaticMain(args.startClass, args.startArgs, /* classLoader= */null);
}

protected static Runnable findStaticMain(String className, String[] argv,
            ClassLoader classLoader) {
        Class<?> cl;
  			...
  			// 反射
        cl = Class.forName(className, true, classLoader);
  			...
        Method m;
        // 寻找main函数
        m = cl.getMethod("main", new Class[] { String[].class });
        ...
        return new MethodAndArgsCaller(m, argv);
    }
```

### <span style="color:#080808;">Step6</span>

<span style="color:#080808;">到了ActivityThread的main函数，主要就是初始化一些东西，例如prepareMainLooper，然后new一个ActivityThread并调用它的attath方法，进而调用AMS的attachApplication方法，流转到attachApplicationLocked方法，先调用ApplicationThread的bindApplication，完成Application的构建，再调用finishAttachApplicationInner开启第一个Activity（下一篇幅 </span>[AOSP 学习：startActivty（2） - 云文档]()<span style="color:#080808;">  讲）。</span>

```java
public static void main(String[] args) {
        ...
        Looper.prepareMainLooper();
  			...
        ActivityThread thread = new ActivityThread();
        thread.attach(false, startSeq);
  			...
        Looper.loop();
  			...
}
```

```java
@GuardedBy("this")
    private void attachApplicationLocked(@NonNull IApplicationThread thread,
            int pid, int callingUid, long startSeq) {
				...
            if (app.getIsolatedEntryPoint() != null) {
                ...
            } else {
               ...
                // 完成Application的创建
                thread.bindApplication(...);
            }
           ...
            if (!mConstants.mEnableWaitForFinishAttachApplication) {
              // 启动Activity
                finishAttachApplicationInner(startSeq, callingUid, pid);
            }
      			...
        } catch (Exception e) {
           ...
        }
    }

```

### <span style="color:#080808;">Step7</span>

<span style="color:#080808;">调用ApplicationThread的bindApplication，构建AppBindData并通过Handler发送给ActivityThread调用handleBindApplication，在这里注册UI线程、初始化Configuration、时区、字体、debuggable等配置，并调用ContextImpl创建applicationContext（具体可看后续是否有精力更新文章讲一下ContextImpl），然后调用LoadedApk的makeApplicationInner开始创建Application类，最终到了Instrumenation使用反射根据清单文件配置的name字段去实例化Application，并调用Application.attath方法挂载Context和LoadedApk引用，然后返回这个Application到LoadedApk的makeApplicationInner中，继续执行callApplicationOnCreate函数，去调用用户重写的Application.onCreate函数。</span>

```java
@Override
        @RavenwoodThrow(comment = "See ActivityThread_ravenwood for initialization on Ravenwood")
        public final void bindApplication(...) {
            ...
            AppBindData data = new AppBindData();
            data.processName = processName;
            data.appInfo = appInfo;
            data.sdkSandboxClientAppVolumeUuid = sdkSandboxClientAppVolumeUuid;
            data.sdkSandboxClientAppPackage = sdkSandboxClientAppPackage;
            data.isSdkInSandbox = isSdkInSandbox;
            data.providers = providerList.getList();
            ...
            // 发送消息
            sendMessage(H.BIND_APPLICATION, data);
        }
```

```java
@UnsupportedAppUsage
    @RavenwoodThrow(comment = "See ActivityThread_ravenwood for initialization on Ravenwood")
    private void handleBindApplication(AppBindData data) {
        mDdmSyncStageUpdater.next(Stage.Bind);

        // Register the UI Thread as a sensitive thread to the runtime.
        VMRuntime.registerSensitiveThread();
        ...
        initZipPathValidatorCallback();
        ...
        mConfigurationController.setConfiguration(data.config);
        mConfigurationController.setCompatConfiguration(data.config);
        mConfiguration = mConfigurationController.getConfiguration();
        ...
        android.graphics.Compatibility.setTargetSdkVersion(data.appInfo.targetSdkVersion);

        TimeZone.setDefault(null);

      LocaleList.setDefault(data.config.getLocales());

       ...
      
        if ((data.appInfo.flags&ApplicationInfo.FLAG_SUPPORTS_SCREEN_DENSITIES)
                == 0) {
            mDensityCompatMode = true;
            Bitmap.setDefaultDensity(DisplayMetrics.DENSITY_DEFAULT);
        }
        mConfigurationController.updateDefaultDensity(data.config.densityDpi);

        // mCoreSettings is only updated from the main thread, while this function is only called
        // from main thread as well, so no need to lock here.
        final String use24HourSetting = getCoreSettingsForDefaultDeviceLocked().getString(
                Settings.System.TIME_12_24);
        Boolean is24Hr = null;
        if (use24HourSetting != null) {
            is24Hr = "24".equals(use24HourSetting) ? Boolean.TRUE : Boolean.FALSE;
        }
        DateFormat.set24HourTimePref(is24Hr);
      
        ...

        boolean isAppDebuggable = (data.appInfo.flags & ApplicationInfo.FLAG_DEBUGGABLE) != 0;
        boolean isAppProfileable = isAppDebuggable || data.appInfo.isProfileable();
        Trace.setAppTracingAllowed(isAppProfileable);
        if ((isAppProfileable || Build.IS_DEBUGGABLE) && data.enableBinderTracking) {
            Binder.enableStackTracking();
        }
       ...

        final InstrumentationInfo ii;
        if (data.instrumentationName != null) {
            ii = prepareInstrumentation(data);
        } else {
            ii = null;
        }

       ...
        try {
           ...
            final IBinder b = ServiceManager.getService(Context.CONNECTIVITY_SERVICE);
            if (b != null) {
                final ConnectivityManager cm =
                        appContext.getSystemService(ConnectivityManager.class);
                Proxy.setHttpProxyConfiguration(cm.getDefaultProxy());
            }
        } finally {
            Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
        }

       ....
        NetworkSecurityConfigProvider.install(appContext);

        // Continue loading instrumentation.
        if (ii != null) {
            initInstrumentation(ii, data, appContext);
        } else {
            mInstrumentation = new Instrumentation();
            mInstrumentation.basicInit(this);
        }

        if ((data.appInfo.flags&ApplicationInfo.FLAG_LARGE_HEAP) != 0) {
            dalvik.system.VMRuntime.getRuntime().clearGrowthLimit();
        } else {
            // Small heap, clamp to the current growth limit and let the heap release
            // pages after the growth limit to the non growth limit capacity. b/18387825
            dalvik.system.VMRuntime.getRuntime().clampGrowthLimit();
        }

       ...
        Application app;
       ...
        try {
          // 这里创建Application
            app = data.info.makeApplicationInner(data.restrictedBackupMode, null);
           ...
        } finally {
           ...
        }

        // Preload fonts resources
        FontsContract.setApplicationContextForResources(appContext);
        ...
    }

```

```java
@RavenwoodKeep
    private Application makeApplicationInner(boolean forceDefaultAppClass,
            Instrumentation instrumentation, boolean allowDuplicateInstances) {
        ...
        try {
            ...
            Application app = null;

            final String myProcessName = Process.myProcessName();
            String appClass = mApplicationInfo.getCustomApplicationClassNameForProcess(
                    myProcessName);
            if (forceDefaultAppClass || (appClass == null)) {
                appClass = "android.app.Application";
            }

            try {
               ...
                ContextImpl appContext = ContextImpl.createAppContext(mActivityThread, this);
                callNetworkSecurityConfigProviderHandleNewApplication(appContext);
              // 新建Application
                app = mActivityThread.mInstrumentation.newApplication(
                        cl, appClass, appContext);
                appContext.setOuterContext(app);
            } catch (Exception e) {
                onNewApplicationFailed(e, app, appClass);
            }
            mActivityThread.addApplication(app);
            ...
            if (instrumentation != null) {
                try {
                  // 调用Application的onCreate
                    instrumentation.callApplicationOnCreate(app);
                } catch (Exception e) {
                    ...
                }
            }

            return app;
        } finally {
           ...
        }
    }

```

```java
@RavenwoodKeep
    public Application newApplication(ClassLoader cl, String className, Context context)
            throws InstantiationException, IllegalAccessException, 
            ClassNotFoundException {
        Application app = getFactory(context.getPackageName())
                .instantiateApplication(cl, className);
        app.attach(context);
        return app;
    }

public @NonNull Application instantiateApplication(@NonNull ClassLoader cl,
            @NonNull String className)
            throws InstantiationException, IllegalAccessException, ClassNotFoundException {
  // 反射      
  return (Application) cl.loadClass(className).newInstance();
    }
```