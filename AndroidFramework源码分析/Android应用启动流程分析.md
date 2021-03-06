---
title: Android应用启动流程分析
categories: 技术
---

本文主要介绍Android的App应用进程、Application和launch activity的初始化过程，希望看完后对应用启动过程有更清晰的认识，了解这部分内容对以后研究应用启动耗时的计算逻辑也有一定帮助。

<!--more-->

### Android启动进程概述

Android系统启动过程中，先由init进程通过解析init.rc文件fork生成Zygote进程，该进程也是Android系统的首个Java进程。之后Zygote进程负责孵化System Server进程和App进程。

<img src="https://github.com/hningoba/KnowledgeSummary/blob/master/img/android_framework_app_start.png?raw=true" style="zoom: 50%;" />

##### System Server进程：

* 由Zygote进程fork生成，System Server是Zygote孵化的第一个进程
* 负责启动、管理整个Java Framework，包含ActivityManager、WindowManager、PackageManager等服务

##### App进程：

* Zygote进程在App层中孵化出的第一个进程是Launcher进程，即手机的桌面App(桌面本身是一个App)
* Zygote还会孵化出Browser、Email、Phone等App进程，每个App至少运行在一个进程上
* 所有App进程都由Zygote进程fork生成



### 应用启动流程

当用户点击手机桌面某个应用的图标时，将由Launcher进程发起，通过binder发消息给System Server进程，然后System Server进程通过socket建立与zygote进程的连接，由zygote进程为目标应用创建进程。

各家厂商一般会定制Launcher，在Android 9.0中，我们以Android默认启动页为例，即launcher3/Launcher.java跟踪代码。当应用图标被点击时，由桌面程序Launcher响应，首先执行``ItemClickHandler.onClick()``：

#### Launcher响应桌面Icon点击

##### 方法调用栈

先看这一部分的方法调用，有个宏观认识后再对其中重要步骤展开：

```
- ItemClickHandler.onClick()
- ItemClickHandler.onClickAppShortcut()
- ItemClickHandler.startAppShortcutOrInfoActivity()
- Launcher.startActivitySafely()
- BaseDraggingActivity.startActivitySafely()
- BaseDraggingActivity.startShortcutIntentSafely()
- Activity.startActivity()
- Activity.startActivityForResult()
- Instrumentation.execStartActivity()
- ActivityManagerService.startActivity()	// 跨进程交由AMS处理启动目标进程
- ActivityManagerService.startActivityAsUser()
- ActivityStarter.execute()
- ActivityStarter.startActivityMayWait() 	// AMS.startActivityAsUser()中执行了setWait()
- ActivityStarter.startActivity() // 此处经历多个startActivity重载方法调用
- ActivityStarter.startActivityUnchecked() // 此处处理Activity启动模式(4种)
- ActivityStackSupervisor.resumeFocusedStackTopActivityLocked() //几种启动模式最终都执行该方法
- ActivityStack.resumeTopActivityUncheckedLocked()
- ActivityStack.resumeTopActivityInnerLocked()
- ActivityStackSupervisor.startSpecificActivityLocked() //冷启动时执行AMS.startProcessLocked()
- ActivityManagerService.startProcessLocked()
```

##### ItemClickHandler.onClick()

从Launcher响应用户点击应用的桌面Icon开始，看下``ItemClickHandler.onClick()``：

```
com.android.launcher3.touch.ItemClickHandler

/**
 * Class for handling clicks on workspace and all-apps items
 */
public class ItemClickHandler {

    ...
    // step1. 响应桌面Icon点击事件
    private static void onClick(View v) {
        ...
        Launcher launcher = Launcher.getLauncher(v.getContext());
        ...

        Object tag = v.getTag();
        if (tag instanceof ShortcutInfo) {
            // step2. 处理点击事件
            onClickAppShortcut(v, (ShortcutInfo) tag, launcher);
        }
        ...
    }
    
    /**
     * Event handler for an app shortcut click.
     *
     * @param v The view that was clicked. Must be a tagged with a {@link ShortcutInfo}.
     */
    private static void onClickAppShortcut(View v, ShortcutInfo shortcut, Launcher launcher) {
        ...

        // step3. 处理点击事件，准备启动应用
        // Start activities
        startAppShortcutOrInfoActivity(v, shortcut, launcher);
    }
    
    private static void startAppShortcutOrInfoActivity(View v, ItemInfo item, Launcher launcher) {
        ...
        // step4. 交由Launcher处理，启动应用
        launcher.startActivitySafely(v, intent, item);
    }
```

ItemClickHandler响应点击事件后，交给Launcher处理，最终通过启动目标应用。

##### ActivityStack.resumeTopActivityInnerLocked()

根据前面列的整体代码调用流程，从``Activity.startActivity()``开始，后续会调用到``ActivityStack.resumeTopActivityInnerLocked()``，看下该方法的具体实现逻辑：

```
com.android.server.am.ActivityStack

private boolean resumeTopActivityInnerLocked(ActivityRecord prev, ActivityOptions options) {
	// 有栈顶Activity处于Resume状态，先将其pause
	boolean pausing = mStackSupervisor.pauseBackStacks(userLeaving, next, false);
	 if (mResumedActivity != null) {
            pausing |= startPausingLocked(userLeaving, false, next, false);
        }
	...
	// 启动目标Activity
	mStackSupervisor.startSpecificActivityLocked(next, true, true);
}
```

上面代码会去判断是否有栈顶Activity处于Resume状态，即``mResumedActivity != null``，如果有的话会，通过``startPausingLocked()``先让栈顶Activity执行Pause过程，然后再执行``ActivityStackSupervisor.startSpecificActivityLocked()``启动目标Activity。因为是冷启动，内部会先创建应用进程，再启动launch activity。

##### ActivityStackSupervisor.startSpecificActivityLocked()

看下``startSpecificActivityLocked()``逻辑：

```
com.android.server.am.ActivityStackSupervisor

void startSpecificActivityLocked(ActivityRecord r,
            boolean andResume, boolean checkConfig) {
        // Is this activity's application already running?
        ProcessRecord app = mService.getProcessRecordLocked(r.processName,
                r.info.applicationInfo.uid, true);

        getLaunchTimeTracker().setLaunchTime(r);
        
        // 分支1：如果已经开辟进程，则直接启动目标Activity
        if (app != null && app.thread != null) {
                ...
                realStartActivityLocked(r, app, andResume, checkConfig);
                return;
        }

        // 分支2：当前App还没启动过，先为APP创建进程
        mService.startProcessLocked(r.processName, r.info.applicationInfo, true, 0,
                "activity", r.intent.getComponent(), false, false, true);
    }
```

上面代码中会去根据进程和线程是否存在判断App是否已经启动，如果已经启动，就会调用``ActivityManagerService.realStartActivityLocked()``方法启动目标Activity。如果没有启动则调用``ActivityManagerService.startProcessLocked()``优先为待启动App创建进程。

下面我们看下AMS如何为APP创建进程。



#### 创建应用进程

创建应用进程主要分两块工作：

* system server进程中，从AMS.startProcess()开始，先配置新建进程参数，然后通过socket建立与zygote进程的连接，将参数列表写给zygote进程，等待zygote进程fork新的进程，返回pid
* zygote进程被system server进程的socket连接请求唤醒，在native层fork目标进程

##### system server发起请求

该部分方法调用栈如下：

```
- ActivityManagerService.startProcessLocked() // 后续会执行几个重载方法
- ActivityManagerService.startProcess()
- Process.start()
- ZygoteProcess.start()
- ZygoteProcess.startViaZygote()
- ZygoteProcess.openZygoteSocketIfNeeded()
- ZygoteProcess.zygoteSendArgsAndGetResult()
```



上面《应用启动流程》部分，最后代码跟到``mService.startProcessLocked()``，该方法内部会执行一系列AMS的``startProcessLocked``重载方法。最终会调用``ActivityManagerService.startProcess()``：

```
com.android.server.am.ActivityManagerService

private ProcessStartResult startProcess(String hostingType, String entryPoint,
            ProcessRecord app, int uid, int[] gids, int runtimeFlags, int mountExternal,
            String seInfo, String requiredAbi, String instructionSet, String invokeWith,
            long startTime) {
            ...
                startResult = Process.start(entryPoint,
                        app.processName, uid, uid, gids, runtimeFlags, mountExternal,
                        app.info.targetSdkVersion, seInfo, requiredAbi, instructionSet,
                        app.info.dataDir, invokeWith,
                        new String[] {PROC_START_SEQ_IDENT + app.startSeq});
            ...
    }
```

``Process.start()``后续会依次执行``ZygoteProcess.start()``、``ZygoteProcess.startViaZygote()``，在``ZygoteProcess.startViaZygote()``内部为创建的目标进程设置参数：

```
android.os.ZygoteProcess

private Process.ProcessStartResult startViaZygote(final String processClass,
                                                      final String niceName,
                                                      final int uid, final int gid,
                                                      final int[] gids,
                                                      int runtimeFlags, int mountExternal,
                                                      int targetSdkVersion,
                                                      String seInfo,
                                                      String abi,
                                                      String instructionSet,
                                                      String appDataDir,
                                                      String invokeWith,
                                                      boolean startChildZygote,
                                                      String[] extraArgs)
                                                      throws ZygoteStartFailedEx {
        ArrayList<String> argsForZygote = new ArrayList<String>();

        argsForZygote.add("--runtime-args");
        argsForZygote.add("--setuid=" + uid);
        argsForZygote.add("--setgid=" + gid);
        argsForZygote.add("--runtime-flags=" + runtimeFlags);
        ...
        argsForZygote.add("--target-sdk-version=" + targetSdkVersion);
        ...

        if (appDataDir != null) {
            argsForZygote.add("--app-data-dir=" + appDataDir);
        }

        if (invokeWith != null) {
            argsForZygote.add("--invoke-with");
            argsForZygote.add(invokeWith);
        }

        if (startChildZygote) {
            argsForZygote.add("--start-child-zygote");
        }

        argsForZygote.add(processClass);

        if (extraArgs != null) {
            for (String arg : extraArgs) {
                argsForZygote.add(arg);
            }
        }

        synchronized(mLock) {
            return zygoteSendArgsAndGetResult(openZygoteSocketIfNeeded(abi), argsForZygote);
        }
    }
```

从上面可以看到，先配置一系列进程参数，之后执行``openZygoteSocketIfNeeded()``建立与Zygote进程的通信连接，再调用``zygoteSendArgsAndGetResult()``向zygote进程传递参数。

看下如何建立和Zygote进程的通信连接：

```
android.os.ZygoteProcess

private ZygoteState openZygoteSocketIfNeeded(String abi) throws ZygoteStartFailedEx {

        if (primaryZygoteState == null || primaryZygoteState.isClosed()) {
            try {
            	// 利用Socket建立和Zygote进程的通信
                primaryZygoteState = ZygoteState.connect(mSocket);
            } catch (IOException ioe) {
                throw new ZygoteStartFailedEx("Error connecting to primary zygote", ioe);
            }
            maybeSetApiBlacklistExemptions(primaryZygoteState, false);
            maybeSetHiddenApiAccessLogSampleRate(primaryZygoteState);
        }
        //根据abi决定使用zygote还是zygote64进行通信
        if (primaryZygoteState.matches(abi)) {
            return primaryZygoteState;
        }
        ...
    }
```

从上面代码主要描述SystemServer进程利用socket建立和Zygote进程的通信，具体可以看一下``ZygoteState.connect()``的逻辑：

```
android.os.ZygoteProcess.ZygoteState

public static ZygoteState connect(LocalSocketAddress address) throws IOException {
            DataInputStream zygoteInputStream = null;
            BufferedWriter zygoteWriter = null;
            final LocalSocket zygoteSocket = new LocalSocket();

            try {
                // 内部使用LocalSocketImpl，建立与zygote进程的Socket连接
                zygoteSocket.connect(address);

                // 获取socket输入流，用来读取zygote进程数据
                zygoteInputStream = new DataInputStream(zygoteSocket.getInputStream());

                // 获取socket输出流，用来将新建进程的参数传递给zygote进程
                zygoteWriter = new BufferedWriter(new OutputStreamWriter(
                        zygoteSocket.getOutputStream()), 256);
            } catch (IOException ex) {
                ...
            }
            ...
            return new ZygoteState(zygoteSocket, zygoteInputStream, zygoteWriter,
                    Arrays.asList(abiListString.split(",")));
        }
```

connect()操作主要获取socket的输入流、输出流，封装到ZygoteState中，交给``zygoteSendArgsAndGetResult()``使用。看下``zygoteSendArgsAndGetResult()``：

```
android.os.ZygoteProcess

private static Process.ProcessStartResult zygoteSendArgsAndGetResult(
            ZygoteState zygoteState, ArrayList<String> args)
            throws ZygoteStartFailedEx {
        try {
            ...

            /**
             * See com.android.internal.os.SystemZygoteInit.readArgumentList()
             * Presently the wire format to the zygote process is:
             * a) a count of arguments (argc, in essence)
             * b) a number of newline-separated argument strings equal to count
             *
             * After the zygote process reads these it will write the pid of
             * the child or -1 on failure, followed by boolean to
             * indicate whether a wrapper process was used.
             */
             // 向zygote进程写数据
            final BufferedWriter writer = zygoteState.writer;
            final DataInputStream inputStream = zygoteState.inputStream;

            writer.write(Integer.toString(args.size()));
            writer.newLine();

            for (int i = 0; i < sz; i++) {
                String arg = args.get(i);
                writer.write(arg);
                writer.newLine();
            }

            writer.flush();

            // Should there be a timeout on this?
            Process.ProcessStartResult result = new Process.ProcessStartResult();

            // 此处阻塞，直到zygote进程fork出新的进程、返回pid
            result.pid = inputStream.readInt();
            result.usingWrapper = inputStream.readBoolean();

            if (result.pid < 0) {
                throw new ZygoteStartFailedEx("fork() failed");
            }
            return result;
        } ...
    }
```

上面代码将前面设置的创建进程的参数写给Zygote进程，然后阻塞，直到zygote进程fork出新的进程，并返回新进程的pid。

下面看看zygote进程如何fork新的进程。



##### zygote创建app进程

该部分方法调用栈如下：

```
- ZygoteInit.main()
- ZygoteServer.runSelectLoop()
- ZygoteServer.acceptCommandPeer()
- ZygoteConnection.processOneCommand()
- Zygote.forkAndSpecialize()
- com_android_internal_os_Zygote_nativeForkAndSpecialize() // 后续执行native层逻辑
- ForkCommon.fork()
```



zygote进程由init进程通过解析init.rc文件fork生成，zygote进程启动后会执行``ZygoteInit.main()``。

```
com.android.internal.os.ZygoteInit

public static void main(String argv[]) {
        ZygoteServer zygoteServer = new ZygoteServer();

        ...
        try {
            ...

            // The select loop returns early in the child process after a fork and
            // loops forever in the zygote.
            caller = zygoteServer.runSelectLoop(abiList);
        } ...
    }
```

看下``zygoteServer.runSelectLoop()``：

```
com.android.internal.os.ZygoteServer

/**
     * Runs the zygote process's select loop. Accepts new connections as
     * they happen, and reads commands from connections one spawn-request's
     * worth at a time.
     */
    Runnable runSelectLoop(String abiList) {
        ArrayList<FileDescriptor> fds = new ArrayList<FileDescriptor>();
        ArrayList<ZygoteConnection> peers = new ArrayList<ZygoteConnection>();

        // mServerSocket是socket通信的server端，即zygote进程
        fds.add(mServerSocket.getFileDescriptor());
        peers.add(null);

        while (true) {
            StructPollfd[] pollFds = new StructPollfd[fds.size()];
            for (int i = 0; i < pollFds.length; ++i) {
                pollFds[i] = new StructPollfd();
                pollFds[i].fd = fds.get(i);
                pollFds[i].events = (short) POLLIN;
            }
            try {
                // 不断轮询，当有新的pollFds到来时，往下执行，否则在这里阻塞
                // poll()官方解释：wait for some event on a file descriptor
                Os.poll(pollFds, -1);
            } ...
            
            for (int i = pollFds.length - 1; i >= 0; --i) {
                if ((pollFds[i].revents & POLLIN) == 0) {
                    continue;
                }

                if (i == 0) {
                    // pollFds[0]就是mServerSocket，有客户端socket连接时执行下面方法
                    ZygoteConnection newPeer = acceptCommandPeer(abiList);
                    peers.add(newPeer);
                    fds.add(newPeer.getFileDesciptor());
                } else {
                    try {
                        ZygoteConnection connection = peers.get(i);
                        final Runnable command = connection.processOneCommand(this);

                        ...
                    } ...
                }
            }
        }
    }
```

上面方法的内容系统注释也说的比较清楚，zygote进程会进入loop循环等待来自客户端socket的连接，即前面讲到system server进程通过socket发起与zygote进程的连接。

zygote进程收到socket连接后，执行``acceptCommandPeer()``创建ZygoteConnection：

```
private ZygoteConnection acceptCommandPeer(String abiList) {
        try {
            return createNewConnection(mServerSocket.accept(), abiList);
        } catch (IOException ex) {
            throw new RuntimeException(
                    "IOException during accept()", ex);
        }
    }
    
 protected ZygoteConnection createNewConnection(LocalSocket socket, String abiList)
            throws IOException {
        return new ZygoteConnection(socket, abiList);
    }
```

ZygoteConnection创建后，接着执行``ZygoteConnection.processOneCommand()``：

```
com.android.internal.os.ZygoteConnection

Runnable processOneCommand(ZygoteServer zygoteServer) {

        ...
        try {
            // 读取新建进程的参数
            args = readArgumentList();
            descriptors = mSocket.getAncillaryFileDescriptors();
        } ...

        ...

        pid = Zygote.forkAndSpecialize(parsedArgs.uid, parsedArgs.gid, parsedArgs.gids,
                parsedArgs.runtimeFlags, rlimits, parsedArgs.mountExternal, parsedArgs.seInfo,
                parsedArgs.niceName, fdsToClose, fdsToIgnore, parsedArgs.startChildZygote,
                parsedArgs.instructionSet, parsedArgs.appDataDir);

        ...
    }
```

上面方法先读取待创建进程的参数列表，即前面``ZygoteProcess.startViaZygote()``中提到的，然后通过``Zygote.forkAndSpecialize()`` fork出app进程：

```
com.android.internal.os.Zygote

public static int forkAndSpecialize(int uid, int gid, int[] gids, int runtimeFlags,
          int[][] rlimits, int mountExternal, String seInfo, String niceName, int[] fdsToClose,
          int[] fdsToIgnore, boolean startChildZygote, String instructionSet, String appDataDir) {
        ...
        int pid = nativeForkAndSpecialize(
                  uid, gid, gids, runtimeFlags, rlimits, mountExternal, seInfo, niceName, fdsToClose,
                  fdsToIgnore, startChildZygote, instructionSet, appDataDir);
        ...
        return pid;
    }
```

后续走到native层``nativeForkAndSpecialize()``：

```
com_android_internal_os_Zygote.cpp

static jint com_android_internal_os_Zygote_nativeForkAndSpecialize(
    JNIEnv *env, jclass, jint uid, jint gid, jintArray gids,
    jint runtime_flags, jobjectArray rlimits,
    jint mount_external, jstring se_info, jstring nice_name,
    jintArray managed_fds_to_close, jintArray managed_fds_to_ignore, jboolean is_child_zygote,
    jstring instruction_set, jstring app_data_dir)
{
  ...

  // 通用fork进程操作
  pid_t pid = ForkCommon(env, false, fds_to_close, fds_to_ignore);

  if (pid == 0)
  {
    SpecializeCommon(env, uid, gid, gids, runtime_flags, rlimits,
                     capabilities, capabilities,
                     mount_external, se_info, nice_name, false,
                     is_child_zygote == JNI_TRUE, instruction_set, app_data_dir);
  }
  return pid;
}
```

看下``ForkCommon``：

```
// Utility routine to fork a process from the zygote.
static pid_t ForkCommon(JNIEnv *env, bool is_system_server,
                        const std::vector<int> &fds_to_close,
                        const std::vector<int> &fds_to_ignore)
{
  ...

  // fork子进程
  pid_t pid = fork();

  ...
  return pid;
}
```

上面代码的逻辑主要是zygote进程通过ForkCommon，fork子进程，然后将pid返回给java层。

整个流程到这里就结束了，回顾一下：

* system server进程通过socket建立和zygote进程的连接，请求创建app进程
* zygote进程启动时进程进入loop状态，当收到客户端socket请求后便被唤醒，读取app进程参数，通过native层的fork机制创建app进程，并返回对应的pid。



#### 初始化主线程

应用进程创建完后会执行``ActivityThread.main()``，看下这部分内容：

```
android.app.ActivityThread

public static void main(String[] args) {
        ...

        // 初始化主线程Looper
        Looper.prepareMainLooper();

        ...
        // 初始化Application和启动页Activity
        ActivityThread thread = new ActivityThread();
        thread.attach(false, startSeq);

        if (sMainThreadHandler == null) {
            sMainThreadHandler = thread.getHandler();
        }
        ...
        // 启动消息轮询
        Looper.loop();
    }
```

从上面可以看到执行了Looper的prepare和loop方法，开启了主线程消息队列的轮询，做法和我们在子线程初始化Handler的逻辑是一样的。



#### 初始化Application

Application初始化流程内容也比较多，这里先列一下主要的方法调用栈，后续对部分细节不了解可以参考这个调用栈自行跟进学习。

##### 方法调用栈

```
- ActivityThread.main()
- ActivityThread.attach()
- ActivityManagerService.attachApplication()
- ActivityManagerService.attachApplicationLocked()
- ActivityStackSupervisor.attachApplicationLocked()
- ActivityStackSupervisor.realStartActivityLocked()
- ActivityThread.ApplicationThread.scheduleTransaction()
- ClientTransactionHandler.scheduleTransaction()
- ActivityThread.sendMessage()
- ActivityThread.H.handleMessage()
- TransactionExecutor.execute()
- LaunchActivityItem.execute()
- ActivityThread.handleLaunchActivity()
- ActivityThread.performLaunchActivity()
```



##### ActivityThread：

前面讲主线程初始化时，在代码``ActivityThread.attach()``的注释中提到内部会初始化Application，看下这部分逻辑：

```
android.app.ActivityThread

private void attach(boolean system, long startSeq) {
        sCurrentActivityThread = this;
        mSystemThread = system;
        if (!system) {
            ViewRootImpl.addFirstDrawHandler(new Runnable() {
                @Override
                public void run() {
                    ensureJitEnabled();
                }
            });
            android.ddm.DdmHandleAppName.setAppName("<pre-initialized>",
                                                    UserHandle.myUserId());
            RuntimeInit.setApplicationObject(mAppThread.asBinder());
            
            // 利用ActivityManagerService创建Application
            final IActivityManager mgr = ActivityManager.getService();
            try {
                // 注意，这里将ApplicationThread的实例传递进去作为ProcessRecord.thread
                mgr.attachApplication(mAppThread, startSeq);
            } catch (RemoteException ex) {
                throw ex.rethrowFromSystemServer();
            }
            
            // 添加GC监听器，系统触发GC时，占用内容量超过进程总量3/4时尝试进行内存释放
            // Watch for getting close to heap limit.
            BinderInternal.addGcWatcher(new Runnable() {
                @Override public void run() {
                    Runtime runtime = Runtime.getRuntime();
                    long dalvikMax = runtime.maxMemory();
                    long dalvikUsed = runtime.totalMemory() - runtime.freeMemory();
                    if (dalvikUsed > ((3*dalvikMax)/4)) {
                        mSomeActivitiesChanged = false;
                        try {
                            mgr.releaseSomeActivities(mAppThread);
                        } catch (RemoteException e) {
                            throw e.rethrowFromSystemServer();
                        }
                    }
                }
            });
        }
        ...

        // 为ViewRootImpl添加Config回调
        ViewRootImpl.ConfigChangedCallback configChangedCallback
                = (Configuration globalConfig) -> {
            synchronized (mResourcesManager) {
                // We need to apply this change to the resources immediately, because upon returning
                // the view hierarchy will be informed about it.
                if (mResourcesManager.applyConfigurationToResourcesLocked(globalConfig,
                        null /* compat */)) {
                    updateLocaleListFromAppContext(mInitialApplication.getApplicationContext(),
                            mResourcesManager.getConfiguration().getLocales());

                    // This actually changed the resources! Tell everyone about it.
                    if (mPendingConfiguration == null
                            || mPendingConfiguration.isOtherSeqNewer(globalConfig)) {
                        mPendingConfiguration = globalConfig;
                        sendMessage(H.CONFIGURATION_CHANGED, globalConfig);
                    }
                }
            }
        };
        ViewRootImpl.addConfigCallback(configChangedCallback);
    }
```

上面``ActivityThread.attach()``主要做了三件事：

* 在AMS中创建Application，具体实现在``ActivityManagerService.attachApplication()``
* 添加GC监听器，系统触发GC时，占用内容量超过进程总量3/4时尝试进行内存释放
* 为ViewRootImpl，即跟View添加Config回调



##### AMS：

重点看下``ActivityThread.attach()``中第一件事``ActivityManagerService.attachApplication()``的实现逻辑：

```
com.android.server.am.ActivityManagerService

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

看下上面代码``attachApplicationLocked()``的实现：

```
com.android.server.am.ActivityManagerService

private final boolean attachApplicationLocked(IApplicationThread thread,
            int pid, int callingUid, long startSeq) {
        ...

        // See if the top visible activity is waiting to run in this process...
        if (normalMode) {
            try {
                if (mStackSupervisor.attachApplicationLocked(app)) {
                    didSomething = true;
                }
            } catch (Exception e) {
                Slog.wtf(TAG, "Exception thrown launching activities in " + app, e);
                badApp = true;
            }
        }
        ...
        return true;
    }
```

##### ActivityStackSupervisor：

上面代码执行了``mStackSupervisor.attachApplicationLocked()``：

```
com.android.server.am.ActivityStackSupervisor

boolean attachApplicationLocked(ProcessRecord app) throws RemoteException {
        final String processName = app.processName;
        boolean didSomething = false;
        for (int displayNdx = mActivityDisplays.size() - 1; displayNdx >= 0; --displayNdx) {
            final ActivityDisplay display = mActivityDisplays.valueAt(displayNdx);
            for (int stackNdx = display.getChildCount() - 1; stackNdx >= 0; --stackNdx) {
                final ActivityStack stack = display.getChildAt(stackNdx);
                if (!isFocusedStack(stack)) {
                    continue;
                }
                stack.getAllRunningVisibleActivitiesLocked(mTmpActivityList);
                final ActivityRecord top = stack.topRunningActivityLocked();
                final int size = mTmpActivityList.size();
                for (int i = 0; i < size; i++) {
                    final ActivityRecord activity = mTmpActivityList.get(i);
                    if (activity.app == null && app.uid == activity.info.applicationInfo.uid
                            && processName.equals(activity.processName)) {
                        try {
                            if (realStartActivityLocked(activity, app,
                                    top == activity /* andResume */, true /* checkConfig */)) {
                                didSomething = true;
                            }
                        }
                        ...
                    }
                }
            }
        }
        if (!didSomething) {
            ensureActivitiesVisibleLocked(null, 0, !PRESERVE_WINDOWS);
        }
        return didSomething;
    }
```

看下上面代码中``ActivityStackSupervisor.realStartActivityLocked()``的逻辑：

```
com.android.server.am.ActivityStackSupervisor

final boolean realStartActivityLocked(ActivityRecord r, ProcessRecord app,
            boolean andResume, boolean checkConfig) throws RemoteException {

        ...

                // 为ClientTransaction添加LaunchActivityItem回调
                // Create activity launch transaction.
                final ClientTransaction clientTransaction = ClientTransaction.obtain(app.thread,
                        r.appToken);
                        
                clientTransaction.addCallback(LaunchActivityItem.obtain(new Intent(r.intent),
                        System.identityHashCode(r), r.info,
                        mergedConfiguration.getGlobalConfiguration(),
                        mergedConfiguration.getOverrideConfiguration(), r.compat,
                        r.launchedFromPackage, task.voiceInteractor, app.repProcState, r.icicle,
                        r.persistentState, results, newIntents, mService.isNextTransitionForward(),
                        profilerInfo));

                // Set desired final state.
                final ActivityLifecycleItem lifecycleItem;
                if (andResume) {
                    lifecycleItem = ResumeActivityItem.obtain(mService.isNextTransitionForward());
                } else {
                    lifecycleItem = PauseActivityItem.obtain();
                }
                clientTransaction.setLifecycleStateRequest(lifecycleItem);

                // 执行transaction逻辑
                mService.getLifecycleManager().scheduleTransaction(clientTransaction);
                ...
            } catch (RemoteException e) {
                ...
            }
        } finally {
            endDeferResume();
        }
        ...

        return true;
    }
```

上面代码中为ClientTransaction对象添加callback，即LaunchActivityItem。然后设置当前的生命周期状态，最后调用``ClientLifecycleManager.scheduleTransaction()``。``scheduleTransaction()``内部执行``ClientTransaction.schedule()``：

```
android.app.servertransaction.ClientTransaction

public void schedule() throws RemoteException {
        mClient.scheduleTransaction(this);
}
```

上面方法中的mClient对象，代码中可以只能看到声明类型是``IApplicationThread``，实际上其实现类是``ActivityThread.ApplicationThread``。这个点跟下下面代码流程就可以知道这个内容：

```
- ActivityThread.main()
- ActivityThread.attach()
- ActivityManagerService.attachApplication(mAppThread, startSeq) // mAppThread就是--ActivityThread.ApplicationThread的实例，也是我们要跟的mClient对象
- ActivityManagerService.attachApplicationLocked() // 这里构造ProcessRecord对象，并将其thread field赋值为mAppThread
- ActivityStackSupervisor.attachApplicationLocked(ProcessRecord app) // app.thread即mAppThread
- ActivityStackSupervisor.realStartActivityLocked()
- ClientTransaction.obtain(app.thread, r.appToken) // 这里创建ClientTransaction实例，并将- ClientTransaction.mClient赋值为入参app.thread，即ActivityThread.ApplicationThread
```

下面看看ClientTransaction的执行逻辑。



##### ClientTransaction：

Android9.0之后，系统引入了ClientTransaction来辅助管理应用及页面生命周期。

继上面接着讲``ClientTransaction.schedule()``，内部执行``mClient.scheduleTransaction(this)``。上面讲到``mClient``实现类是``ActivityThread.ApplicationThread``，那看下``ActivityThread.ApplicationThread.scheduleTransaction()``。该方法内部实现比较简单，主要是执行了下面的代码调用：

```
- ActivityThread.ApplicationThread.scheduleTransaction()
- ClientTransactionHandler.scheduleTransaction() //ActivityThread继承自ClientTransactionHandler
- ActivityThread.sendMessage()	//使用ActivityThread.mH（Handler实现）发送消息，即主线程消息
- ActivityThread.H.handleMessage()	//H接收消息，其中msg.what == EXECUTE_TRANSACTION
```

看下上面方法调用栈中的``ClientTransactionHandler.scheduleTransaction()``：

```
public abstract class ClientTransactionHandler {

    void scheduleTransaction(ClientTransaction transaction) {
        transaction.preExecute(this);
        // 向ActivityThread.mH发送个消息，msg.what为EXECUTE_TRANSACTION，object为ClientTransaction
        sendMessage(ActivityThread.H.EXECUTE_TRANSACTION, transaction);
    }
}
```

上面``sendMessage()``的实现在子类ActivityThread中：

```
android.app.ActivityThread

	void sendMessage(int what, Object obj) {
        sendMessage(what, obj, 0, 0, false);
    }

    private void sendMessage(int what, Object obj, int arg1, int arg2, boolean async) {
        Message msg = Message.obtain();
        msg.what = what;
        msg.obj = obj;
        msg.arg1 = arg1;
        msg.arg2 = arg2;
        if (async) {
            msg.setAsynchronous(true);
        }
        // 将消息发送给mH
        mH.sendMessage(msg);
    }
```

所以，从``ClientTransactionHandler.scheduleTransaction()``和``ActivityThread.sendMessage()``这两个方法的实现我们可以知道，其实是向``ActivityThread.mH``发送个消息，msg.what为EXECUTE_TRANSACTION，object为ClientTransaction。

关于ActivityThread.mH中如何处理消息及ClientTransaction的我们后续会讲到。

**总结**：

* 从ActivityThread.main()开始，经过ActivityManagerService、ActivityStackSupervisor，最后通过ClientTransaction把LaunchActivityItem组装为ClientTransaction的一个Callback
* ClientTransaction通过``ActivityThread.ApplicationThread``给ActivityThread.mH发送主线程消息，其中消息的what为EXECUTE_TRANSACTION，object为ClientTransaction
* 在执行消息任务时执行ClientTransaction的Callback，即LaunchActivityItem，其内部发起Application、launch activity相关初始化流程，这部分后面会讲到



##### ActivityThread.mH：

ActivityThread.mH接收到``msg.what == EXECUTE_TRANSACTION``的消息后执行``TransactionExecutor.execute()``，代码如下：

```
android.app.ActivityThread.H

public void handleMessage(Message msg) {
	switch (msg.what) {
     case EXECUTE_TRANSACTION:
       final ClientTransaction transaction = (ClientTransaction) msg.obj;
       mTransactionExecutor.execute(transaction);
     }
}
```

在``mTransactionExecutor.execute()``内部会执行ClientTransaction中的Callback回调方法``execute()``。这里的Callback就是LaunchActivityItem。



##### LaunchActivityItem：

看下LaunchActivityItem的``execute()``的逻辑：

```
android.app.servertransaction.LaunchActivityItem

@Override
    public void execute(ClientTransactionHandler client, IBinder token,
            PendingTransactionActions pendingActions) {

        ActivityClientRecord r = new ActivityClientRecord(token, mIntent, mIdent, mInfo,
                mOverrideConfig, mCompatInfo, mReferrer, mVoiceInteractor, mState, mPersistentState,
                mPendingResults, mPendingNewIntents, mIsForward,
                mProfilerInfo, client);
        // 启动Launch Activity，其中client实现类是ActivityThread
        client.handleLaunchActivity(r, pendingActions, null /* customIntent */);
    }
```



##### ActivityThread.handleLaunchActivity()

上面代码中，client的实现类是``ActivityThread``，``ActivityThread.handleLaunchActivity()``的主要实现逻辑在``ActivityThread.performLaunchActivity()``：

```
android.app.ActivityThread

private Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent) {
        ActivityInfo aInfo = r.activityInfo;
        if (r.packageInfo == null) {
            r.packageInfo = getPackageInfo(aInfo.applicationInfo, r.compatInfo,
                    Context.CONTEXT_INCLUDE_CODE);
        }

        // 通过Intent解析Component
        ComponentName component = r.intent.getComponent();
        if (component == null) {
            component = r.intent.resolveActivity(
                mInitialApplication.getPackageManager());
            r.intent.setComponent(component);
        }

        if (r.activityInfo.targetActivity != null) {
            component = new ComponentName(r.activityInfo.packageName,
                    r.activityInfo.targetActivity);
        }

        // 创建Context
        ContextImpl appContext = createBaseContextForActivity(r);
        Activity activity = null;
        try {
        	// 利用Instrument实例化目标Activity
            java.lang.ClassLoader cl = appContext.getClassLoader();
            activity = mInstrumentation.newActivity(
                    cl, component.getClassName(), r.intent);
            StrictMode.incrementExpectedActivityCount(activity.getClass());
            r.intent.setExtrasClassLoader(cl);
            r.intent.prepareToEnterProcess();
            if (r.state != null) {
                r.state.setClassLoader(cl);
            }
        } ...

        try {
            // 初始化Application和Context，并将Context attach到Application。内部会执行我们非常熟悉的Application的attachBaseContext()、onCreate()方法
            Application app = r.packageInfo.makeApplication(false, mInstrumentation);

            ...
            if (activity != null) {
                ...

                // Activity初始化，包括创建Window、绑定mApplication/mIntent等
                activity.attach(appContext, this, getInstrumentation(), r.token,
                        r.ident, app, r.intent, r.activityInfo, title, r.parent,
                        r.embeddedID, r.lastNonConfigurationInstances, config,
                        r.referrer, r.voiceInteractor, window, r.configCallback);

                ...
                
                // 给Activity设置主题
                int theme = r.activityInfo.getThemeResource();
                if (theme != 0) {
                    activity.setTheme(theme);
                }

                // 调用Activity.onCreate()、Fragments.onCreate()生命周期方法
                if (r.isPersistable()) {
                    mInstrumentation.callActivityOnCreate(activity, r.state, r.persistentState);
                } else {
                    mInstrumentation.callActivityOnCreate(activity, r.state);
                }
                ...
            }
            
        }
        ...

        return activity;
    }
```

上面这段代码主要做Application、Activity、Context的初始化，建立三者之间的关系，执行Activity、Fragment的onCreate生命周期方法：

* 通过Intent解析Component，为实例化目标Activity做准备
* 利用Instrument实例化目标Activity，启动阶段将实例化luanch activity
* 初始化Application和Context，并将Context attach到Application。内部会执行Application的attachBaseContext()、onCreate()
* 将前面实例化的Activity对象进行相关初始化工作，包括创建Window、绑定mApplication/mIntent等
* 给Activity设置主题
* 调用Activity.onCreate()，如果Activity中有Fragment，则执行Fragment.onCreate()



#### 总结

App整体启动流程内容比较多，可以通过Launcher响应屏幕点击、应用进程的创建、Application初始化这三部分来理解。

* 由Launcher响应用户点击屏幕应用Icon事件

* system server进程通过socket建立与zygote的通信，从AMS.startProcess()开始为目标APP申请创建进程

* zygote进程被system server进程的socket连接请求唤醒，在native层fork目标进程

* 应用进程创建后执行ActivityTread.main()，先初始化主线程，再实例化Application对象，并启动launch Activity

  