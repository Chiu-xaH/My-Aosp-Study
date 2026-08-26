# Android 学习记录：startActivty（2）——启动首个Activity

<span style="color:#080808;">书接上文 </span>[AOSP 学习：startActivty（1）——进程与Application](/startActivty（1）——进程创建与Application初始化.md)<span style="color:#080808;">，Application.onCreate完成之后的流程。</span>

## <span style="color:#080808;">顺序图：</span>

![图片](/AMS-2.png)

## 流程概述

### Step1

继AMS调用ApplicationThread.bindApplication完成后，调用attachApplication进而流转到ActivityTaskSupervisor的realStartActivityInner，构建两个ClientTransactionItem：LaunchActivityItem和ResumeActivityItem，然后发送出去并组装成List，通过Handler发送给ActivityThread。

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
      			if (com.android.server.am.Flags.expediteActivityLaunchOnColdStart()) {
                if (normalMode) {
             			 // 启动Activity
                    mAtmInternal.attachApplication(app.getWindowProcessController());
                }
            }
      			...
        } catch (Exception e) {
           ...
        }
    }

```

```java
@Nullable
    private RemoteException tryRealStartActivityInner(
            @NonNull Task task,
            @NonNull ActivityRecord r,
            @NonNull WindowProcessController proc,
            @Nullable IActivityClientController activityClientController,
            boolean andResume) {
        ...
        final LaunchActivityItem launchActivityItem = new LaunchActivityItem(r.token,
                r.intent, System.identityHashCode(r), r.info,
                procConfig, overrideConfig, deviceId,
                r.getFilteredReferrer(r.launchedFromPackage), task.voiceInteractor,
                proc.getReportedProcState(), r.getSavedState(), r.getPersistentSavedState(),
                results, newIntents, r.takeSceneTransitionInfo(), isTransitionForward,
                proc.createProfilerInfoIfNeeded(), r.assistToken, activityClientController,
                r.shareableActivityToken, r.getLaunchedFromBubble(), fragmentToken,
                r.initialCallerInfoAccessToken, activityWindowInfo, r.getDisplayId());

        
        final ActivityLifecycleItem lifecycleItem;
        if (andResume) {
            lifecycleItem = new ResumeActivityItem(r.token, isTransitionForward,
                    r.shouldSendCompatFakeFocus());
        } else if (r.isVisibleRequested()) {
            lifecycleItem = new PauseActivityItem(r.token);
        } else {
            lifecycleItem = new StopActivityItem(r.token);
        }
        ...
          // scheduleTransactionItems分发这两个ClientTransaction
        final boolean isSuccessful = mService.getLifecycleManager().scheduleTransactionItems(
                proc.getThread(),true /* shouldDispatchImmediately */,
                launchActivityItem, lifecycleItem);
        ...
        return null;
    }
```

```java
void scheduleTransaction(ClientTransaction transaction) {
        transaction.preExecute(this);
  			// 发送消息
        sendMessage(ActivityThread.H.EXECUTE_TRANSACTION, transaction);
    }
```

### Step2

ActivityThread接收到消息后，调用execute函数，遍历列表处理每个ClientTransactionItem。对于LaunchActivityItem，调用executeNonLifecycleItem进行处理；对于ResumeActivityItem，调用executeLifecycleItem进行处理。

```java
@VisibleForTesting
    public void executeTransactionItems(@NonNull ClientTransaction transaction) {
        final List<ClientTransactionItem> items = transaction.getTransactionItems();
      // [LaunchActivityItem,ResumeActivityItem]
        final int size = items.size();
        for (int i = 0; i < size; i++) {
            final ClientTransactionItem item = items.get(i);
            if (item.isActivityLifecycleItem()) {
              // LaunchActivityItem
                executeLifecycleItem(transaction, (ActivityLifecycleItem) item);
            } else {
              // ResumeActivityItem
                executeNonLifecycleItem(transaction, item,
                        shouldExcludeLastLifecycleState(items, i));
            }
        }
    }
```

### Step3

调用executeNonLifecycleItem处理LaunchActivityItem，调用它的execute函数，构建ActivityClientRecord，作为传输载体，进而调用到ActivityThread的handleLaunchActivity，在这里初始化WindowManagerGlobal，反射创建Activity，调用Activity的attath方法，attath方法attathBaseContext、新建WIndow的唯一实现类PhoneWindow，设置WindowManager等配置。最终完成一系列操作后调用onCreate，并设置ActivityClientRecord的state为ON\_CREATE。

```java
private void executeNonLifecycleItem(@NonNull ClientTransaction transaction,
            @NonNull ClientTransactionItem item, boolean shouldExcludeLastLifecycleState) {
        ...
        item.execute(mTransactionHandler, mPendingActions);
        item.postExecute(mTransactionHandler, mPendingActions);
  			...
    }
```

```java
@Override
    public void execute(@NonNull ClientTransactionHandler client,
            @NonNull PendingTransactionActions pendingActions) {
       ...
        final ActivityClientRecord r = new ActivityClientRecord(mActivityToken, mIntent, mIdent,
                mInfo, mOverrideConfig, mReferrer, mVoiceInteractor, mState, mPersistentState,
                mPendingResults, mPendingNewIntents, mSceneTransitionInfo, mIsForward,
                mProfilerInfo, client, mAssistToken, mShareableActivityToken, mLaunchedFromBubble,
                mTaskFragmentToken, mInitialCallerInfoAccessToken, mActivityWindowInfo);
        client.handleLaunchActivity(r, pendingActions, mDeviceId, null /* customIntent */);
        ...
    }
```

```java
@Override
    public Activity handleLaunchActivity(ActivityClientRecord r,
            PendingTransactionActions pendingActions, int deviceId, Intent customIntent) {
        ....
        WindowManagerGlobal.initialize();
				....
        final Activity a = performLaunchActivity(r, customIntent);
        ...
        return a;
    }

 private Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent) {
   			...
        Activity activity = null;
        try {
            java.lang.ClassLoader cl;
            if (isSandboxedSdkContextUsed) {
                cl = activityBaseContext.getApplicationContext().getClassLoader();
            } else {
                cl = activityBaseContext.getClassLoader();
            }
          // 反射创建
            activity = mInstrumentation.newActivity(
                    cl, component.getClassName(), r.intent);
           ...
        } catch (Exception e) {
            ...
        }

        try {
          // 已经创建了，直接返回application
            Application app = r.packageInfo.makeApplicationInner(false, mInstrumentation);
            ...
            synchronized (mResourcesManager) {
                mActivities.put(r.token, r);
            }

            if (activity != null) {
              ...
                Window window = null;
                if (r.mPendingRemoveWindow != null && r.mPreserveWindow) {
                    window = r.mPendingRemoveWindow;
                    r.mPendingRemoveWindow = null;
                    r.mPendingRemoveWindowManager = null;
                }

                // 资源
                activityBaseContext.getResources().addLoaders(
                        app.getResources().getLoaders().toArray(new ResourcesLoader[0]));

              // context
                activityBaseContext.setOuterContext(activity);
              // attath方法
                activity.attach(activityBaseContext, this, getInstrumentation(), r.token,
                        r.ident, app, r.intent, r.activityInfo, title, r.parent,
                        r.embeddedID, r.lastNonConfigurationInstances, config,
                        r.referrer, r.voiceInteractor, window, r.activityConfigCallback,
                        r.assistToken, r.shareableActivityToken, r.initialCallerInfoAccessToken);

              ...
                int theme = r.activityInfo.getThemeResource();
                if (theme != 0) {
                  // 设置主题
                    activity.setTheme(theme);
                }

                // 调用onCreate
                r.activity = activity;
                if (r.isPersistable()) {
                    mInstrumentation.callActivityOnCreate(activity, r.state, r.persistentState);
                } else {
                    mInstrumentation.callActivityOnCreate(activity, r.state);
                }
                ...
            }
          // 设置状态ON_CREATE
            r.setState(ON_CREATE);

        } catch (Exception e) {
            ...
        }
        return activity;
    }

```

### Step4

调用executeLifecycleItem处理ResumeActivityItem，首先调用cycleToPath函数，传参getTargetState，即目标流转状态，即ResumeActivityItem重写的getTargetState，返回ON\_RESUME。cycleToPath内将当前状态ON\_CREATE作为start，目标状态ON\_RESUME作为finish，传参给getLifecyclePath，取两者的中间态即ON\_START返回给cycleToPath，然后根据状态调用到ActivityThread的handleStartActivity。

```java
private void executeLifecycleItem(@NonNull ClientTransaction transaction,
            @NonNull ActivityLifecycleItem lifecycleItem) {
        ...
         // 状态流转 lifecycleItem.getTargetState()为ON_RESUME
        cycleToPath(r, lifecycleItem.getTargetState(), true /* excludeLastState */, transaction);

  			// 执行execute
        lifecycleItem.execute(mTransactionHandler, mPendingActions);
        lifecycleItem.postExecute(mTransactionHandler, mPendingActions);
    }
```

```java
private void cycleToPath(ActivityClientRecord r, int finish, boolean excludeLastState,
            ClientTransaction transaction) {
  	// 当前为ON_CRETAE
        final int start = r.getLifecycleState();
       ...
        final IntArray path = mHelper.getLifecyclePath(start, finish, excludeLastState);
        performLifecycleSequence(r, path, transaction);
    }


@VisibleForTesting
    public IntArray getLifecyclePath(int start, int finish, boolean excludeLastState) {
       
        mLifecycleSequence.clear();
        if (finish >= start) {
            if (start == ON_START && finish == ON_STOP) {
                ...
            } else {
                // just go there
                for (int i = start + 1; i <= finish; i++) {
                  // 添加了ON_START和ON_RSUME
                    mLifecycleSequence.add(i);
                }
            }
        } else { // finish < start, can't just cycle down
            ...
        }

       if (excludeLastState && mLifecycleSequence.size() != 0) {
          // 移除了ON_RESUME
            mLifecycleSequence.remove(mLifecycleSequence.size() - 1);
        }
      // 返回[ON_START]
        return mLifecycleSequence;
    }
```

```java
private void performLifecycleSequence(ActivityClientRecord r, IntArray path,
            ClientTransaction transaction) {
        final int size = path.size();
        for (int i = 0, state; i < size; i++) {
            state = path.get(i);
           ...
             // 根据上面得到的path列表流转到指定方法
            switch (state) {
                case ON_CREATE:
                    mTransactionHandler.handleLaunchActivity(r, mPendingActions,
                            Context.DEVICE_ID_INVALID, null /* customIntent */);
                    break;
                case ON_START:
                    mTransactionHandler.handleStartActivity(r, mPendingActions,
                            null /* sceneTransitionInfo */);
                    break;
                case ON_RESUME:
                    mTransactionHandler.handleResumeActivity(r, false /* finalStateRequest */,
                            r.isForward, false /* shouldSendCompatFakeFocus */,
                            "LIFECYCLER_RESUME_ACTIVITY");
                    break;
                case ON_PAUSE:
                    mTransactionHandler.handlePauseActivity(r, false /* finished */,
                            false /* userLeaving */,
                            false /* autoEnteringPip */, mPendingActions,
                            "LIFECYCLER_PAUSE_ACTIVITY");
                    break;
                case ON_STOP:
                    mTransactionHandler.handleStopActivity(r,
                            mPendingActions, false /* finalStateRequest */,
                            "LIFECYCLER_STOP_ACTIVITY");
                    break;
                case ON_DESTROY:
                    mTransactionHandler.handleDestroyActivity(r, false /* finishing */,
                            false /* getNonConfigInstance */,
                            "performLifecycleSequence. cycling to:" + path.get(size - 1));
                    break;
                case ON_RESTART:
                    mTransactionHandler.performRestartActivity(r, false /* start */);
                    break;
                default:
                    throw new IllegalArgumentException("Unexpected lifecycle state: " + state);
            }
        }
    }
```

### Step5

在handleStartActivity这里调用到Activity的performStart，在这里调用onStart，向子Fragment分发状态，判断debug则显示Dialog等操作。最后设置ActivityClientRecord的state为ON\_START。

```java
@Override
    public void handleStartActivity(ActivityClientRecord r,
            PendingTransactionActions pendingActions, SceneTransitionInfo sceneTransitionInfo) {
        final Activity activity = r.activity;
        ...
        // Start
        activity.performStart("handleStartActivity");
      // 设置状态
        r.setState(ON_START);

      ...
        // Restore instance state
        if (pendingActions.shouldRestoreInstanceState()) {
            if (r.isPersistable()) {
                if (r.state != null || r.persistentState != null) {
                    mInstrumentation.callActivityOnRestoreInstanceState(activity, r.state,
                            r.persistentState);
                }
            } else if (r.state != null) {
                mInstrumentation.callActivityOnRestoreInstanceState(activity, r.state);
            }
        }

        // Call postOnCreate()
        if (pendingActions.shouldCallOnPostCreate()) {
            activity.mCalled = false;
            Trace.traceBegin(Trace.TRACE_TAG_WINDOW_MANAGER, "onPostCreate");
            if (r.isPersistable()) {
                mInstrumentation.callActivityOnPostCreate(activity, r.state,
                        r.persistentState);
            } else {
                mInstrumentation.callActivityOnPostCreate(activity, r.state);
            }
            ...
        }

        updateVisibility(r, true /* show */);
        mSomeActivitiesChanged = true;
    }

```

### Step6

executeLifecycleItem在完成cycleToPath之后，还需要调用ResumeActivityItem的execute函数，调用到ActivityThread的handleResumeActivity，在这里先调用到Activity的performResume，向子Fragment分发状态等操作。然后调用onResume，设置ActivityClientRecord的state为ON\_RESUME。接下来调用WIndowManager的addView传入DectorView，并调用Activity的makeVisible函数取设置DectorView为View.VISIBLE，UI就可见了，然后向Loop.myQueue发送一个Idler标志着结束。

```java
private void executeLifecycleItem(@NonNull ClientTransaction transaction,
            @NonNull ActivityLifecycleItem lifecycleItem) {
        ...
         // 状态流转 lifecycleItem.getTargetState()为ON_RESUME
        cycleToPath(r, lifecycleItem.getTargetState(), true /* excludeLastState */, transaction);

  			// 执行execute
        lifecycleItem.execute(mTransactionHandler, mPendingActions);
        lifecycleItem.postExecute(mTransactionHandler, mPendingActions);
    }
```

```java
@Override
    public void execute(@NonNull ClientTransactionHandler client, @NonNull ActivityClientRecord r,
            @NonNull PendingTransactionActions pendingActions) {
        ...
        client.handleResumeActivity(r, true /* finalStateRequest */, mIsForward,
                mShouldSendCompatFakeFocus, "RESUME_ACTIVITY");
        ...
    }
```

```java
@Override
    public void handleResumeActivity(ActivityClientRecord r, boolean finalStateRequest,
            boolean isForward, boolean shouldSendCompatFakeFocus, String reason) {
        ...
        if (!performResumeActivity(r, finalStateRequest, reason)) {
            return;
        }
        ...
        if (r.window == null && !a.mFinished && willBeVisible) {
            r.window = r.activity.getWindow();
            View decor = r.window.getDecorView();
            decor.setVisibility(View.INVISIBLE);
            ViewManager wm = a.getWindowManager();
            WindowManager.LayoutParams l = r.window.getAttributes();
            ...
            if (r.mPreserveWindow) {
                a.mWindowAdded = true;
                r.mPreserveWindow = false;
                ViewRootImpl impl = decor.getViewRootImpl();
                if (impl != null) {
                    impl.notifyChildRebuilt();
                }
            }
            if (a.mVisibleFromClient) {
                if (!a.mWindowAdded) {
                    a.mWindowAdded = true;
                    wm.addView(decor, l);
                } else {
                    a.onWindowAttributesChanged(l);
                }
            }
        } else if (!willBeVisible) {
            ...
        }

      
        if (!r.activity.mFinished && willBeVisible && r.activity.mDecor != null && !r.hideForNow) {
            if (localLOGV) Slog.v(TAG, "Resuming " + r + " with isForward=" + isForward);
            ...

            r.activity.mVisibleFromServer = true;
            mNumVisibleActivities++;
            if (r.activity.mVisibleFromClient) {
              // makeVisible
                r.activity.makeVisible();
            }
           ...
        }

        mNewActivities.add(r);
        // 结束
        Looper.myQueue().addIdleHandler(new Idler());
    }

```

```java
void makeVisible() {
        if (!mWindowAdded) {
            ViewManager wm = getWindowManager();
            wm.addView(mDecor, getWindow().getAttributes());
            mWindowAdded = true;
        }
        mDecor.setVisibility(View.VISIBLE);
    }

```

```java
final void performResume(boolean followedByPause, String reason) {
       ...
        mFragments.dispatchResume();
        mFragments.execPendingActions();

  			...
          // onResume
          mInstrumentation.callActivityOnResume(this);

       ...
        onPostResume();
      	...
        dispatchActivityPostResumed();
				...
    }

```

## 压缩

继AMS调用ApplicationThread.bindApplication完成后，调用attachApplication，构建两个ClientTransactionItem：LaunchActivityItem和ResumeActivityItem，组装列表通过Handler发送给ActivityThread，ActivityThread遍历列表处理每个ClientTransactionItem。

对于LaunchActivityItem，调用它的execute函数，构建ActivityClientRecord，作为传输载体，调用handleLaunchActivity初始化WindowManagerGlobal、反射创建Activity、调用Activity的attath方法、新建PhoneWindow，设置WindowManager等。完成一系列操作后调用onCreate，然后设置ActivityClientRecord的state为ON\_CREATE。

对于ResumeActivityItem，首先调用cycleToPath函数，根据目标流转状态ON\_RESUME和当然状态ON\_CREATE，找到中间态ON\_STRAT，根据状态调用到ActivityThread的handleStartActivity，进而调用到Activity的performStart，在这里调用onStart，向子Fragment分发状态，判断debug则显示Dialog等操作。完成一系列操作后调用onStart，然后设置ActivityClientRecord的state为ON\_START。

完成cycleToPath之后，还需要继续调用ResumeActivityItem的execute函数，调用到ActivityThread的handleResumeActivity，先执行Activity的performResume，进行向子Fragment分发状态等操作。完成一系列操作后调用onResume，然后设置ActivityClientRecord的state为ON\_RESUME。

接下来在handleResumeActivity中调用WIndowManager的addView传入DectorView，并调用Activity的makeVisible函数取设置DectorView为View.VISIBLE，UI就可见了，然后向Loop.myQueue发送一个Idler标志着结束。