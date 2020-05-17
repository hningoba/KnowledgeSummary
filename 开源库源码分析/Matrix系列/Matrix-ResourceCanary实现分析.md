---
title: Matrix - ResourceCanary源码分析
categories: 技术
---

本文主要介绍Matrix的Resource部分，涉及Activity泄漏、Bitmap冗余对象的检测逻辑。

<!--more-->

再回顾下Matrix概览中提到的ResourceCanary模块的特点：基于 WeakReference 的特性和 [Square Haha](https://github.com/square/haha) 库开发的 Activity 泄漏和 Bitmap 重复创建检测工具。

主要特性如下：

- 分离了检测和分析部分，便于在不打断自动化测试的前提下持续输出分析后的检测结果
- 对检测部分生成的 Hprof 文件进行了裁剪，移除了大部分无用数据，降低了传输 Hprof 文件的开销
- 增加了重复 Bitmap 对象检测，方便通过减少冗余 Bitmap 数量，降低内存消耗



开始前的疑问：

* 如何判断Activity是否泄漏
* GCRoot
* 泄漏的Activity到GCRoot的最短强引用链（套用经典图搜索算法）
* Hprof文件介绍
* HAHA库如何将Hprof文件按照文档描述的格式解析成结构化的引用关系图
* LeakCanary的误报原因
* 如何裁剪Hprof文件
* 如何判断哪些bitmap对象是冗余的



### 如何判断Activity发生泄漏

判断Activity是否泄漏需要确定两个问题：

* 如何在一个恰当的时机得知一个Activity已经结束了生命周期
* 如何判断一个Activity无法被GC机制回收

这部分内容，Matrix借鉴了[LeakCanary](https://github.com/square/leakcanary)的做法，对LeakCanary比较熟悉的同学可以跳过。

Activity泄漏测试页面是``TestLeakActivity``。通过操作demo，进入该页面再退出，可以看到如下log：

```
V/Matrix.ActivityRefWatcher: triggering gc...
V/Matrix.ActivityRefWatcher: gc was triggered.
I/Matrix.ActivityRefWatcher: activity with key [MATRIX_RESCANARY_REFKEY_sample.tencent.matrix.resource.TestLeakActivity_2bd863cd6b8d4b78ae371fcc660c3b66] should be recycled but actually still exists in N times, wait for next detection to confirm.
```

通过log可以看出`TestLeakActivity`应该被回收但是仍然存在，即发生了泄漏。

针对第一个问题，我们看看Matrix怎么做的：

##### 获取已销毁Activity的信息

Matrix wiki中也提到的了解决方法：

- 让所有Activity继承一个BaseActivity，然后在`BaseActivity.onDestroy()`方法中进行记录。
- 通过某种机制得知``Activity.onDestroy()``方法被调用，然后进行记录
  - 4.0以前可以通过反射替换`ActivityThread.mInstrumentation`对象为自己的代理，然后在代理中的`callActivityOnDestroy()`方法中记录。
  - 4.0以后可以通过`Application.registerActivityLifecycleCallbacks()`方法注册一个回调对象，在回调对象的`onActivityDestroyed()`方法中记录。

显然第一种方法对业务侧入侵过重，不合适。第二种方法，考虑到4.0以前机器分布已经比较少了，Matrix直接使用`Application.registerActivityLifecycleCallbacks()`方式。

ResouceCanary中处理Activity泄漏问题的接口类是ResourcePlugin，看下代码：

```
com.tencent.matrix.resource.ResourcePlugin

public class ResourcePlugin extends Plugin {
    private final ResourceConfig mConfig;
    private ActivityRefWatcher mWatcher = null;

    //加载配置
    public ResourcePlugin(ResourceConfig config) {
        mConfig = config;
    }

    //处理泄漏误报
    public static void activityLeakFixer(Application application) {...}

    public ActivityRefWatcher getWatcher() {
        return mWatcher;
    }

    @Override
    public void init(Application app, PluginListener listener) {
        super.init(app, listener);
        ...
        mWatcher = new ActivityRefWatcher(app, this);
    }

    @Override
    public void start() {
        super.start();
        ...
        //启动ActivityRefWatcher，监听Activity生命周期
        mWatcher.start();
    }
}
```

上面代码主要在组件启动时，启动了`ActivityRefWatcher`，看下`mWatcher.start()`：

```
com.tencent.matrix.resource.watcher.ActivityRefWatcher

@Override
    public void start() {
        stopDetect();
        final Application app = mResourcePlugin.getApplication();
        if (app != null) {
            // 注册Application.ActivityLifecycleCallbacks监听
            app.registerActivityLifecycleCallbacks(mRemovedActivityMonitor);
            // Activity泄漏检测任务调度，并根据DumpMode（可配置）选项对外输出泄漏提示
            scheduleDetectProcedure();
        }
    }
```

上面代码主要做了两件事，注释中做了描述，其中Activity泄漏检测任务调度的逻辑后面会讲到。看下`mRemovedActivityMonitor`的逻辑：

```
com.tencent.matrix.resource.watcher.ActivityRefWatcher

private final Application.ActivityLifecycleCallbacks mRemovedActivityMonitor = new ActivityLifeCycleCallbacksAdapter() {

        @Override
        public void onActivityDestroyed(Activity activity) {
            pushDestroyedActivityInfo(activity);
        }
    };
```

当一个Activity销毁时，即`Activity.onDestroy()`执行时，`Application.ActivityLifecycleCallbacks.onActivityDestroyed()`就会被调用，看下`pushDestroyedActivityInfo()`：

```
com.tencent.matrix.resource.watcher.ActivityRefWatcher

private void pushDestroyedActivityInfo(Activity activity) {
        final String activityName = activity.getClass().getName();
        //泄漏Activity的上报排重
        if (!mResourcePlugin.getConfig().getDetectDebugger() && isPublished(activityName)) {
            MatrixLog.i(TAG, "activity leak with name %s had published, just ignore", activityName);
            return;
        }
        //主要根据UUID和Activity信息组装key
        final UUID uuid = UUID.randomUUID();
        final StringBuilder keyBuilder = new StringBuilder();
        keyBuilder.append(ACTIVITY_REFKEY_PREFIX).append(activityName)
            .append('_').append(Long.toHexString(uuid.getMostSignificantBits())).append(Long.toHexString(uuid.getLeastSignificantBits()));
        final String key = keyBuilder.toString();
        final DestroyedActivityInfo destroyedActivityInfo
            = new DestroyedActivityInfo(key, activity, activityName);
        mDestroyedActivityInfos.add(destroyedActivityInfo);
    }
```

上面代码主要做了两件事：

* Matrix通过`isPublished()`的逻辑上报改进点，已判断为泄漏的Activity，记录其类名，放到`FilePublisher.mPublishedMap`，后续再检测到该Activity，则排重，避免重复提示该Activity已泄漏
* 根据UUID和Activity信息组装key，将泄漏的Activity信息存放到`mDestroyedActivityInfos`，其中`DestroyedActivityInfo`内部使用一个WeakReference对象持有该Activity

**总结：**

针对“如何在一个恰当的时机得知一个Activity已经结束了生命周期”，Matrix和LeakCanary的逻辑一样，通过`Application.registerActivityLifecycleCallbacks()`方法注册一个回调对象，在回调对象的`onActivityDestroyed()`方法中记录泄漏页面信息。



##### 如何判断Activity无法被GC回收

针对前面提到的第二个问题，即“如何判断一个Activity无法被GC机制回收”，Matrix的做法是这样的：

首先通过上面《获取已销毁Activity的信息》部分获取到已经destroy的Activity，该Activity由WeakReference持有，然后主动触发一次“有效的GC”，如果该Activity能够被回收，则持有它的WeakReference会被置空，反之，如果持有它的WeakReference不为空，即GC无法回收这个已经销毁的Activity，认为该Activity发生了泄漏。

上面说的“有效的GC”，是因为JVM没有提供强制触发GC的API，像`System.gc()`或者`Runtime.getRuntime().gc()`都是建议系统进行GC，系统并不一定真正的触发GC。

针对这个问题，Matrix使用了“哨兵机制”，即增加了一个“哨兵对象”，该对象由WeakReference持有，执行`Runtime.getRuntime().gc()`后，如果该哨兵WeakReference被置空，则说明刚才的gc()调用，系统确实触发了一次GC操作。

看下这部分逻辑的代码，实现细节在`ActivityRefWatcher.scheduleDetectProcedure()`：

```
com.tencent.matrix.resource.watcher.ActivityRefWatcher

private void scheduleDetectProcedure() {
        mDetectExecutor.executeInBackground(mScanDestroyedActivitiesTask);
    }
```

看下`mDetectExecutor.executeInBackground()`：

```
com.tencent.matrix.resource.watcher.RetryableTaskExecutor

public void executeInBackground(final RetryableTask task) {
        postToBackgroundWithDelay(task, 0);
    }
    
private void postToBackgroundWithDelay(final RetryableTask task, final int failedAttempts) {
        mBackgroundHandler.postDelayed(new Runnable() {
            @Override
            public void run() {
                RetryableTask.Status status = task.execute();
                //如果task.execute()返回RETRY，则进行延时重试
                if (status == RetryableTask.Status.RETRY) {
                    postToBackgroundWithDelay(task, failedAttempts + 1);
                }
            }
        }, mDelayMillis);
    }
```

上面代码就是在异步线程执行`mScanDestroyedActivitiesTask`，当该task返回值为`RetryableTask.Status.RETRY`时，则该task进行延时重试。看下该task内容：

```
com.tencent.matrix.resource.watcher.ActivityRefWatcher

private final RetryableTask mScanDestroyedActivitiesTask = new RetryableTask() {

        @Override
        public Status execute() {
            //step1.没有destroy的Activity，则重试
            // If destroyed activity list is empty, just wait to save power.
            if (mDestroyedActivityInfos.isEmpty()) {
                MatrixLog.i(TAG, "DestroyedActivityInfo isEmpty!");
                return Status.RETRY;
            }
            ...

            //step2.构建“哨兵”
            final WeakReference<Object> sentinelRef = new WeakReference<>(new Object());
            //step3.触发gc
            triggerGc();
            //“哨兵”没有被释放，说明step3并没有真正触发内存回收
            if (sentinelRef.get() != null) {
                // System ignored our gc request, we will retry later.
                MatrixLog.d(TAG, "system ignore our gc request, wait for next detection.");
                return Status.RETRY;
            }

            final Iterator<DestroyedActivityInfo> infoIt = mDestroyedActivityInfos.iterator();

            while (infoIt.hasNext()) {
                final DestroyedActivityInfo destroyedActivityInfo = infoIt.next();
                
                //step4.排重，避免重复提示Activity泄漏
                if (!mResourcePlugin.getConfig().getDetectDebugger() && isPublished(destroyedActivityInfo.mActivityName) && mDumpHprofMode != ResourceConfig.DumpMode.SILENCE_DUMP) {
                    MatrixLog.v(TAG, "activity with key [%s] was already published.", destroyedActivityInfo.mActivityName);
                    infoIt.remove();
                    continue;
                }
                //step5.再次检测destroy的Activity是否被释放
                if (destroyedActivityInfo.mActivityRef.get() == null) {
                    // The activity was recycled by a gc triggered outside.
                    MatrixLog.v(TAG, "activity with key [%s] was already recycled.", destroyedActivityInfo.mKey);
                    infoIt.remove();
                    continue;
                }

                //step6.检测次数加1
                ++destroyedActivityInfo.mDetectedCount;

                //step7.检测次数小于最大重复检测次数，等下一次重试再安全check是否确实泄漏
                if (destroyedActivityInfo.mDetectedCount < mMaxRedetectTimes
                    || !mResourcePlugin.getConfig().getDetectDebugger()) {
                    // Although the sentinel tell us the activity should have been recycled,
                    // system may still ignore it, so try again until we reach max retry times.
                    MatrixLog.i(TAG, "activity with key [%s] should be recycled but actually still \n"
                            + "exists in %s times, wait for next detection to confirm.",
                        destroyedActivityInfo.mKey, destroyedActivityInfo.mDetectedCount);
                    continue;
                }

                MatrixLog.i(TAG, "activity with key [%s] was suspected to be a leaked instance. mode[%s]", destroyedActivityInfo.mKey, mDumpHprofMode);

                //step8.根据DumpMode执行不同的泄漏提示策略
                //step8.1 SILENCE_DUMP模式下，以IssueActivity形式展示泄漏Activity信息
                if (mDumpHprofMode == ResourceConfig.DumpMode.SILENCE_DUMP) {
                    if (mResourcePlugin != null && !isPublished(destroyedActivityInfo.mActivityName)) {
                        final JSONObject resultJson = new JSONObject();
                        try {
                            resultJson.put(SharePluginInfo.ISSUE_ACTIVITY_NAME, destroyedActivityInfo.mActivityName);
                        } catch (JSONException e) {
                            MatrixLog.printErrStackTrace(TAG, e, "unexpected exception.");
                        }
                        mResourcePlugin.onDetectIssue(new Issue(resultJson));
                    }
                    if (null != activityLeakCallback) {
                        activityLeakCallback.onLeak(destroyedActivityInfo.mActivityName, destroyedActivityInfo.mKey);
                    }
                } 
                //step8.2 AUTO_DUMP模式下，Toast提示并保存裁剪后hprof信息文件
                else if (mDumpHprofMode == ResourceConfig.DumpMode.AUTO_DUMP) {
                    final File hprofFile = mHeapDumper.dumpHeap();
                    if (hprofFile != null) {
                        markPublished(destroyedActivityInfo.mActivityName);
                        final HeapDump heapDump = new HeapDump(hprofFile, destroyedActivityInfo.mKey, destroyedActivityInfo.mActivityName);
                        mHeapDumpHandler.process(heapDump);
                        infoIt.remove();
                    } else {
                        MatrixLog.i(TAG, "heap dump for further analyzing activity with key [%s] was failed, just ignore.",
                                destroyedActivityInfo.mKey);
                        infoIt.remove();
                    }
                } 
                //step8.3 MANUAL_DUMP模式下，以通知形式提示泄漏信息
                else if (mDumpHprofMode == ResourceConfig.DumpMode.MANUAL_DUMP) {
                    NotificationManager notificationManager = (NotificationManager) context.getSystemService(Context.NOTIFICATION_SERVICE);
                    String dumpingHeapContent = context.getString(R.string.resource_canary_leak_tip);
                    String dumpingHeapTitle = destroyedActivityInfo.mActivityName;
                    mContentIntent.putExtra(SharePluginInfo.ISSUE_ACTIVITY_NAME, destroyedActivityInfo.mActivityName);
                    mContentIntent.putExtra(SharePluginInfo.ISSUE_REF_KEY, destroyedActivityInfo.mKey);
                    PendingIntent pIntent = PendingIntent.getActivity(context, 0, mContentIntent,
                            PendingIntent.FLAG_UPDATE_CURRENT);
                    NotificationCompat.Builder builder = new NotificationCompat.Builder(context)
                            .setContentTitle(dumpingHeapTitle)
                            .setContentIntent(pIntent)
                            .setContentText(dumpingHeapContent);
                    Notification notification = buildNotification(context, builder);
                    notificationManager.notify(NOTIFICATION_ID, notification);

                    infoIt.remove();
                    markPublished(destroyedActivityInfo.mActivityName);
                    MatrixLog.i(TAG, "show notification for notify activity leak. %s", destroyedActivityInfo.mActivityName);
                } 
                //step8.4 NO_DUMP模式下，以IssueActivity形式展示泄漏Activity信息
                else {
                    // Lightweight mode, just report leaked activity name.
                    MatrixLog.i(TAG, "lightweight mode, just report leaked activity name.");
                    markPublished(destroyedActivityInfo.mActivityName);
                    if (mResourcePlugin != null) {
                        final JSONObject resultJson = new JSONObject();
                        try {
                            resultJson.put(SharePluginInfo.ISSUE_ACTIVITY_NAME, destroyedActivityInfo.mActivityName);
                        } catch (JSONException e) {
                            MatrixLog.printErrStackTrace(TAG, e, "unexpected exception.");
                        }
                        mResourcePlugin.onDetectIssue(new Issue(resultJson));
                    }
                }
            }

            return Status.RETRY;
        }
    };
```

上面这部分代码主要做了8件事，注释中都一一标明，本质上是利用“哨兵”机制和排重策略，准确的找到无法被GC回收的已销毁Activity，然后再根据外部可配置的DumpMode输出泄漏Activity的信息。

测试内存泄漏时，需要在`MatrixApplication`中ResourceCanary配置初始化过程中，将`ResourceConfig.Builder.setDetectDebuger()`传true才能看到具体的泄漏信息。比如我们把`DumpMode`改成`MANUAL_DUMP`，测试`TestLeakActivity`时，可以在通知栏看到如下信息，即`TestLeakActivity`发生了泄漏。

<img src="https://raw.githubusercontent.com/hningoba/KnowledgeSummary/master/img/matrix_testactivity_leak_notification.png" style="zoom:65%;" />

### Hprof文件介绍

hprof文件通常也叫堆转储文件，包含了Dump时刻内存中的所有对象的信息，包括类的描述，实例的数据和引用关系，线程的栈信息等。



##### 如何获取Hprof文件

代码中可以使用`Debug.dumpHprofData(file)`方法获取Hprof文件。平时开发定位问题，可以直接用AndroidStudio中的Profiler获取hprof文件。官方文档[使用 Memory Profiler 查看 Java 堆和内存分配](https://developer.android.com/studio/profile/memory-profiler#save-hprof)比较详细的介绍了hprof的使用方法，具体可看官网。如果AS底部控制台没有Profiler选项，也可以通过AndroidStudio顶部Run菜单里面的profiler选项启动profiler。

Profiler目前已经是非常强大且易用的性能分析工具，可以分析CPU使用率、各种内存占用、网络使用、方法耗时等各种指标。笔者平视看代码，获取方法调用栈、分析方法耗时等，也经常使用该工具，比如之前分析Matrix-Trace源码分析文章时，通过Profiler的方法调用栈就比较容易的看到哪些方法是通过插桩实现的，方便快速熟悉陌生的项目，建议大家多学习使用该工具。

借用官网中提供的图片，下图中的“2”可用于获取hprof文件。平时开发过程中，Rocord一段App操作后，可以把操作后的堆栈信息用这种方式dump出来，后续便可使用AS导入hprof文件，方便分析。

<img src="https://developer.android.com/studio/images/profile/memory-profiler-callouts_2x.png" style="zoom:65%;" />

将本地Hprof本地文件导入到Profiler中：

![matrix_profiler_hprof](matrix_profiler_hprof.png)

上图中各个列对应含义：

- **Allocations**：堆中的分配数。
- **Native Size**：此对象类型使用的原生内存总量（以字节为单位）。只有在使用 Android 7.0 及更高版本时，才会看到此列。您会在此处看到采用 Java 分配的某些对象的内存，因为 Android 对某些框架类（如 `Bitmap`）使用原生内存。
- **Shallow Size**：此对象类型使用的 Java 内存总量（以字节为单位）。
- **Retained Size**：为此类的所有实例而保留的内存总大小（以字节为单位）。



##### 如何获取引用链

前面部分提到获取泄漏Activity的信息，对解决问题提供帮助还需要计算泄漏Activity对象到GCRoots的强引用链。GCRoots对象特点是，他们虽然不被其他生命周期更长的对象持有，但JVM特性导致这类对象不会被GC回收。因此，从这类对象出发，经过一系列强引用的对象也都无法被回收。这部分可以看看[这篇文章](https://www.dynatrace.com/resources/ebooks/javabook/how-garbage-collection-works/)。

GCRoots包括下面几种对象：

* 静态成员，因为Java中的类，即被JVM system class loader加载的类无法卸载，也就无法被回收，典型的就是类的静态成员以及被静态成员持有的对象都是无法被GC的
* 局部变量或方法参数持有的对象
* JNI Reference，包括JNILocalReference、JNIGlobalReference持有的对象
* 活动的Thread实例
* synchronized关键字用到的对象，比如经常用到的double check的单例模式中，用synchronized修饰的类class就无法被GC回收

如果某个Activity被泄漏，则必然存在从它到某个GC Root的强引用链。只要将这条强引用链找出来，开发者就能根据引用链上的对象找到合适的修改点快速解决问题。

Hprof文件格式可以参考[这份文档](http://hg.openjdk.java.net/jdk8/jdk8/jdk/raw-file/43cb25339b55/src/share/demo/jvmti/hprof/manual.html#mozTocId848088)Binary Dump Format一节，文档中GCRoots的Tag格式如下：





### 如何检测重复Bitmap对象





### 参考

[Matrix ResourceCanary Wiki](https://github.com/Tencent/matrix/wiki/Matrix-Android-ResourceCanary)

[HPROF Agent](http://hg.openjdk.java.net/jdk8/jdk8/jdk/raw-file/43cb25339b55/src/share/demo/jvmti/hprof/manual.html)

[使用 Memory Profiler 查看 Java 堆和内存分配](https://developer.android.com/studio/profile/memory-profiler#save-hprof)

