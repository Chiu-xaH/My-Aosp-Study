# Android 学习记录：系统启动（2）——Zygote

顺序图：

![图片](/Zygote.png)

Step1：Init进程加载init.zygote64.rc（入口点等上一篇幅[系统启动（1）——Init 进程](/系统启动（1）——Init%20进程.md)写完我再讲），启动service，名为zygote，可执行文件在<span style="color:#080808;">/system/bin/app\_process64，参数为 -Xzygote /system/bin --zygote --start-system-server --socket-name=zygote。这里有人会疑惑，/system/bin/app\_process64根本找不到，这是编译产物，源码里面对应的是frameworks/base/cmds/app\_process，里面是app\_main.cpp，也就是执行到的是app\_main.cpp的main函数。</span>

```bash
service zygote /system/bin/app_process64 -Xzygote /system/bin --zygote --start-system-server --socket-name=zygote
    class main
    priority -20
    user root
    group root readproc reserved_disk
    socket zygote stream 660 root system
    socket usap_pool_primary stream 660 root system
    onrestart exec_background - system system -- /system/bin/vdc volume abort_fuse
    onrestart write /sys/power/state on
    # NOTE: If the wakelock name here is changed, then also
    # update it in SystemSuspend.cpp
    onrestart write /sys/power/wake_lock zygote_kwl
    onrestart restart audioserver
    onrestart restart cameraserver
    onrestart restart media
    onrestart restart --only-if-running media.tuner
    onrestart restart netd
    onrestart restart wificond
    task_profiles ProcessCapacityHigh MaxPerformance
    critical window=${zygote.critical_window.minute:-off} target=zygote-fatal
```

Step2：看app\_main.cpp的main函数，记得传入的参数是“<span style="color:#080808;">-Xzygote /system/bin --zygote --start-system-server --socket-name=zygote</span>”。首先创建了AppRuntime，然后对参数进行检查，接下来调用<span style="color:#080808;">maybeCreateDalvikCache，组装参数args列表，调用runtime.start(</span><span style="color:#067d17;">"</span>[com.android.int](http://com.android.int)<span style="color:#067d17;">ernal.os.ZygoteInit"</span><span style="color:#080808;">, args, zygote);</span>

```c
int main(int argc, char* const argv[])
{
  	...
    AppRuntime runtime(argv[0], computeArgBlockSize(argc, argv));
   ...
    bool zygote = false;
    bool startSystemServer = false;
    bool application = false;
    String8 niceName;
    String8 className;

  	// 参数解析
    ++i;  // Skip unused "parent dir" argument.
    while (i < argc) {
        const char* arg = argv[i++];
        if (strcmp(arg, "--zygote") == 0) {
            zygote = true;
            niceName = ZYGOTE_NICE_NAME;
        } else if (strcmp(arg, "--start-system-server") == 0) {
            startSystemServer = true;
        } else if (strcmp(arg, "--application") == 0) {
            application = true;
        } else if (strncmp(arg, "--nice-name=", 12) == 0) {
            niceName = (arg + 12);
        } else if (strncmp(arg, "--", 2) != 0) {
            className = arg;
            break;
        } else {
            --i;
            break;
        }
    }

    Vector<String8> args;
    if (!className.empty()) {
       ...
    } else {
        // We're in zygote mode.
        maybeCreateDalvikCache();

        if (startSystemServer) {
            args.add(String8("start-system-server"));
        }
       ...
        // In zygote mode, pass all remaining arguments to the zygote
        // main() method.
        for (; i < argc; ++i) {
            args.add(String8(argv[i]));
        }
    }
		...
    if (zygote) {
        runtime.start("com.android.internal.os.ZygoteInit", args, zygote);
    } else if (!className.empty()) {
       ...
    } else {
       ...
    }
}

```

<span style="color:#080808;">Step3:runtime是AppRuntime，查看AppRuntime发现没有start函数，在父类AndroidRuntime</span>

<span style="color:#080808;">中，首先启动了JVM（Java虚拟机），注册了JNI环境，很多JNI函数（nativeXX开头的Java函数）都是在这里注册的，然后通过反射上面传入的</span>[com.android.int](http://com.android.int)<span style="color:#080808;">ernal.os.ZygoteInit，并调用main函数。</span>

```c
void AndroidRuntime::start(const char* className, const Vector<String8>& options, bool zygote)
{
    // 启动JVM
    JniInvocation jni_invocation;
    jni_invocation.Init(NULL);
    JNIEnv* env;
    if (startVm(&mJavaVM, &env, zygote, primary_zygote) != 0) {
        return;
    }
    onVmCreated(env);
  
  	// 注册JNI环境
    if (startReg(env) < 0) {
        ALOGE("Unable to register all android natives\n");
        return;
    }

    ...
    // 反射创建传入的com.android.internal.os.ZygoteInit，并调用main函数
    char* slashClassName = toSlashClassName(className != NULL ? className : "");
    jclass startClass = env->FindClass(slashClassName);
    if (startClass == NULL) {
        ....
    } else {
        jmethodID startMeth = env->GetStaticMethodID(startClass, "main",
            "([Ljava/lang/String;)V");
        if (startMeth == NULL) {
           ....
        } else {
            env->CallStaticVoidMethod(startClass, startMeth, strArray);
        ....
        }
    }
    ....
}
```

Step4：【待写】

```java
    public static void main(String[] argv) {
        ZygoteServer zygoteServer = null;
        // 开启单线程限制
        ZygoteHooks.startZygoteNoThreadCreation();

        // 设置0号进程
        try {
            Os.setpgid(0, 0);
        } catch (ErrnoException ex) {
            ...
        }

        Runnable caller;
        try {
            ...
            RuntimeInit.preForkInit();

            boolean startSystemServer = false;
            String zygoteSocketName = "zygote";
            String abiList = null;
            boolean enableLazyPreload = false;
          // 参数读取
            for (int i = 1; i < argv.length; i++) {
                if ("start-system-server".equals(argv[i])) {
                    startSystemServer = true;
                } else if ("--enable-lazy-preload".equals(argv[i])) {
                    enableLazyPreload = true;
                } else if (argv[i].startsWith(ABI_LIST_ARG)) {
                    abiList = argv[i].substring(ABI_LIST_ARG.length());
                } else if (argv[i].startsWith(SOCKET_NAME_ARG)) {
                    zygoteSocketName = argv[i].substring(SOCKET_NAME_ARG.length());
                } else {
                    throw new RuntimeException("Unknown command line argument: " + argv[i]);
                }
            }

            ...
            
            if (!enableLazyPreload) {
                ...
                  // 预加载一些类、系统资源等，这样每次zygote fork的时候就不用再次初始化这些了
                preload(bootTimingsTraceLog);
               ...
            }
          ...
						// 手动触发GC            
            gcAndFinalize();
            // 初始化Native层的Zygote
            Zygote.initNativeState(isPrimaryZygote);
						// 解除单线程限制
            ZygoteHooks.stopZygoteNoThreadCreation();
						// 新建ZygoteServer
            zygoteServer = new ZygoteServer(isPrimaryZygote);

            if (startSystemServer) {
              // 在这里开启SystemServer，见下一篇幅
                Runnable r = forkSystemServer(abiList, zygoteSocketName, zygoteServer);
								...
                if (r != null) {
                    r.run();
                    return;
                }
            }

           // 开启死循环
            caller = zygoteServer.runSelectLoop(abiList);
        } catch (Throwable ex) {
							...
        } finally {
            if (zygoteServer != null) {
              // 关闭zygote
                zygoteServer.closeServerSocket();
            }
        }
        ...
        if (caller != null) {
            caller.run();
        }
    }

```

```java
 static void preload(TimingsTraceLog bootTimingsTraceLog) {
        // 以反射按照配置清单预加载类
        preloadClasses();
        ...
        cacheNonBootClasspathClassLoaders();
        ...
        // 加载资源
        Resources.preloadResources();
        ...
        nativePreloadAppProcessHALs();
        ...
        maybePreloadGraphicsDriver();
        ...
        preloadSharedLibraries();
        preloadTextResources();
        ...
        WebViewFactory.prepareWebViewInZygote();
   			...
    }
```

```java

 private static final String PRELOADED_CLASSES = "/system/etc/preloaded-classes";

 private static void preloadClasses() {
        final VMRuntime runtime = VMRuntime.getRuntime();

        InputStream is;
        try {
         // 加载文件
            is = new FileInputStream(PRELOADED_CLASSES);
        } catch (FileNotFoundException e) {
           ...
            return;
        }
        ...
        try {
            BufferedReader br =
                    new BufferedReader(new InputStreamReader(is), Zygote.SOCKET_BUFFER_SIZE);

            int count = 0;
            int missingLambdaCount = 0;
            String line;
            while ((line = br.readLine()) != null) {
                // Skip comments and blank lines.
                line = line.trim();
                if (line.startsWith("#") || line.equals("")) {
                    continue;
                }
              ...
                try {
                  // 反射创建类
                    Class.forName(line, true, null);
                    count++;
                } catch(...) {...}
            }
						...
        } catch (IOException e) {
            ...
        } finally {
            IoUtils.closeQuietly(is);

            // Fill in dex caches with classes, fields, and methods brought in by preloading.
            runtime.preloadDexCaches();

            // If we are profiling the boot image, reset the Jit counters after preloading the
            // classes. We want to preload for performance, and we can use method counters to
            // infer what clases are used after calling resetJitCounters, for profile purposes.
            if (shouldProfileBootClasspath()) {
                VMRuntime.resetJitCounters();
            }
           ...
        }
    }
```

Step5：

> 这里涉及新加的UsapPool，感兴趣可以去搜一下，大概意思就是系统会在空闲时fork好一些进程，需要用的时候直接特化即可使用。
```java
    Runnable runSelectLoop(String abiList) {
        ArrayList<FileDescriptor> socketFDs = new ArrayList<>();
        ArrayList<ZygoteConnection> peers = new ArrayList<>();

        socketFDs.add(mZygoteSocket.getFileDescriptor());
        peers.add(null);

        mUsapPoolRefillTriggerTimestamp = INVALID_TIMESTAMP;

        while (true) {
            fetchUsapPoolPolicyPropsWithMinInterval();
            mUsapPoolRefillAction = UsapPoolRefillAction.NONE;

            int[] usapPipeFDs = null;
            StructPollfd[] pollFDs;

            // Allocate enough space for the poll structs, taking into account
            // the state of the USAP pool for this Zygote (could be a
            // regular Zygote, a WebView Zygote, or an AppZygote).
            if (mUsapPoolEnabled) {
                usapPipeFDs = Zygote.getUsapPipeFDs();
                pollFDs = new StructPollfd[socketFDs.size() + 1 + usapPipeFDs.length];
            } else {
                pollFDs = new StructPollfd[socketFDs.size()];
            }

            /*
             * For reasons of correctness the USAP pool pipe and event FDs
             * must be processed before the session and server sockets.  This
             * is to ensure that the USAP pool accounting information is
             * accurate when handling other requests like API deny list
             * exemptions.
             */

            int pollIndex = 0;
            for (FileDescriptor socketFD : socketFDs) {
                pollFDs[pollIndex] = new StructPollfd();
                pollFDs[pollIndex].fd = socketFD;
                pollFDs[pollIndex].events = (short) POLLIN;
                ++pollIndex;
            }

            final int usapPoolEventFDIndex = pollIndex;

            if (mUsapPoolEnabled) {
                pollFDs[pollIndex] = new StructPollfd();
                pollFDs[pollIndex].fd = mUsapPoolEventFD;
                pollFDs[pollIndex].events = (short) POLLIN;
                ++pollIndex;

                // The usapPipeFDs array will always be filled in if the USAP Pool is enabled.
                assert usapPipeFDs != null;
                for (int usapPipeFD : usapPipeFDs) {
                    FileDescriptor managedFd = new FileDescriptor();
                    managedFd.setInt$(usapPipeFD);

                    pollFDs[pollIndex] = new StructPollfd();
                    pollFDs[pollIndex].fd = managedFd;
                    pollFDs[pollIndex].events = (short) POLLIN;
                    ++pollIndex;
                }
            }

            int pollTimeoutMs;

            if (mUsapPoolRefillTriggerTimestamp == INVALID_TIMESTAMP) {
                pollTimeoutMs = -1;
            } else {
                long elapsedTimeMs = System.currentTimeMillis() - mUsapPoolRefillTriggerTimestamp;

                if (elapsedTimeMs >= mUsapPoolRefillDelayMs) {
                    // The refill delay has elapsed during the period between poll invocations.
                    // We will now check for any currently ready file descriptors before refilling
                    // the USAP pool.
                    pollTimeoutMs = 0;
                    mUsapPoolRefillTriggerTimestamp = INVALID_TIMESTAMP;
                    mUsapPoolRefillAction = UsapPoolRefillAction.DELAYED;

                } else if (elapsedTimeMs <= 0) {
                    // This can occur if the clock used by currentTimeMillis is reset, which is
                    // possible because it is not guaranteed to be monotonic.  Because we can't tell
                    // how far back the clock was set the best way to recover is to simply re-start
                    // the respawn delay countdown.
                    pollTimeoutMs = mUsapPoolRefillDelayMs;

                } else {
                    pollTimeoutMs = (int) (mUsapPoolRefillDelayMs - elapsedTimeMs);
                }
            }

            int pollReturnValue;
            try {
                pollReturnValue = Os.poll(pollFDs, pollTimeoutMs);
            } catch (ErrnoException ex) {
                throw new RuntimeException("poll failed", ex);
            }

            if (pollReturnValue == 0) {
                // The poll returned zero results either when the timeout value has been exceeded
                // or when a non-blocking poll is issued and no FDs are ready.  In either case it
                // is time to refill the pool.  This will result in a duplicate assignment when
                // the non-blocking poll returns zero results, but it avoids an additional
                // conditional in the else branch.
                mUsapPoolRefillTriggerTimestamp = INVALID_TIMESTAMP;
                mUsapPoolRefillAction = UsapPoolRefillAction.DELAYED;

            } else {
                boolean usapPoolFDRead = false;

                while (--pollIndex >= 0) {
                    if ((pollFDs[pollIndex].revents & POLLIN) == 0) {
                        continue;
                    }

                    if (pollIndex == 0) {
                        // Zygote server socket
                        ZygoteConnection newPeer = acceptCommandPeer(abiList);
                        peers.add(newPeer);
                        socketFDs.add(newPeer.getFileDescriptor());
                    } else if (pollIndex < usapPoolEventFDIndex) {
                        // Session socket accepted from the Zygote server socket

                        try {
                            ZygoteConnection connection = peers.get(pollIndex);
                            boolean multipleForksOK = !isUsapPoolEnabled()
                                    && ZygoteHooks.isIndefiniteThreadSuspensionSafe();
                            final Runnable command =
                                    connection.processCommand(this, multipleForksOK);

                            // TODO (chriswailes): Is this extra check necessary?
                            if (mIsForkChild) {
                                // We're in the child. We should always have a command to run at
                                // this stage if processCommand hasn't called "exec".
                                if (command == null) {
                                    throw new IllegalStateException("command == null");
                                }

                                return command;
                            } else {
                                // We're in the server - we should never have any commands to run.
                                if (command != null) {
                                    throw new IllegalStateException("command != null");
                                }

                                // We don't know whether the remote side of the socket was closed or
                                // not until we attempt to read from it from processCommand. This
                                // shows up as a regular POLLIN event in our regular processing
                                // loop.
                                if (connection.isClosedByPeer()) {
                                    connection.closeSocket();
                                    peers.remove(pollIndex);
                                    socketFDs.remove(pollIndex);
                                }
                            }
                        } catch (Exception e) {
                            if (!mIsForkChild) {
                                // We're in the server so any exception here is one that has taken
                                // place pre-fork while processing commands or reading / writing
                                // from the control socket. Make a loud noise about any such
                                // exceptions so that we know exactly what failed and why.

                                Slog.e(TAG, "Exception executing zygote command: ", e);

                                // Make sure the socket is closed so that the other end knows
                                // immediately that something has gone wrong and doesn't time out
                                // waiting for a response.
                                ZygoteConnection conn = peers.remove(pollIndex);
                                conn.closeSocket();

                                socketFDs.remove(pollIndex);
                            } else {
                                // We're in the child so any exception caught here has happened post
                                // fork and before we execute ActivityThread.main (or any other
                                // main() method). Log the details of the exception and bring down
                                // the process.
                                Log.e(TAG, "Caught post-fork exception in child process.", e);
                                throw e;
                            }
                        } finally {
                            // Reset the child flag, in the event that the child process is a child-
                            // zygote. The flag will not be consulted this loop pass after the
                            // Runnable is returned.
                            mIsForkChild = false;
                        }

                    } else {
                        // Either the USAP pool event FD or a USAP reporting pipe.

                        // If this is the event FD the payload will be the number of USAPs removed.
                        // If this is a reporting pipe FD the payload will be the PID of the USAP
                        // that was just specialized.  The `continue` statements below ensure that
                        // the messagePayload will always be valid if we complete the try block
                        // without an exception.
                        long messagePayload;

                        try {
                            byte[] buffer = new byte[Zygote.USAP_MANAGEMENT_MESSAGE_BYTES];
                            int readBytes =
                                    Os.read(pollFDs[pollIndex].fd, buffer, 0, buffer.length);

                            if (readBytes == Zygote.USAP_MANAGEMENT_MESSAGE_BYTES) {
                                DataInputStream inputStream =
                                        new DataInputStream(new ByteArrayInputStream(buffer));

                                messagePayload = inputStream.readLong();
                            } else {
                                Log.e(TAG, "Incomplete read from USAP management FD of size "
                                        + readBytes);
                                continue;
                            }
                        } catch (Exception ex) {
                            if (pollIndex == usapPoolEventFDIndex) {
                                Log.e(TAG, "Failed to read from USAP pool event FD: "
                                        + ex.getMessage());
                            } else {
                                Log.e(TAG, "Failed to read from USAP reporting pipe: "
                                        + ex.getMessage());
                            }

                            continue;
                        }

                        if (pollIndex > usapPoolEventFDIndex) {
                            Zygote.removeUsapTableEntry((int) messagePayload);
                        }

                        usapPoolFDRead = true;
                    }
                }

                if (usapPoolFDRead) {
                    int usapPoolCount = Zygote.getUsapPoolCount();

                    if (usapPoolCount < mUsapPoolSizeMin) {
                        // Immediate refill
                        mUsapPoolRefillAction = UsapPoolRefillAction.IMMEDIATE;
                    } else if (mUsapPoolSizeMax - usapPoolCount >= mUsapPoolRefillThreshold) {
                        // Delayed refill
                        mUsapPoolRefillTriggerTimestamp = System.currentTimeMillis();
                    }
                }
            }

            if (mUsapPoolRefillAction != UsapPoolRefillAction.NONE) {
                int[] sessionSocketRawFDs =
                        socketFDs.subList(1, socketFDs.size())
                                .stream()
                                .mapToInt(FileDescriptor::getInt$)
                                .toArray();

                final boolean isPriorityRefill =
                        mUsapPoolRefillAction == UsapPoolRefillAction.IMMEDIATE;

                final Runnable command =
                        fillUsapPool(sessionSocketRawFDs, isPriorityRefill);

                if (command != null) {
                    return command;
                } else if (isPriorityRefill) {
                    // Schedule a delayed refill to finish refilling the pool.
                    mUsapPoolRefillTriggerTimestamp = System.currentTimeMillis();
                }
            }
        }
    }
```