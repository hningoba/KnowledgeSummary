---
title: Matrix-TraceCanary实现分析
categories: 技术

---

本文主要介绍Matrix的Trace部分，主要涉及帧率、ANR、慢函数、启动耗时的检测逻辑。

<!--more-->

### Trace Canary主要特性

回顾下Trace Canary主要特性：

- 编译期动态修改字节码, 高性能记录执行耗时与调用堆栈
- 准确的定位到发生卡顿的函数，提供执行堆栈、执行耗时、执行次数等信息，帮助快速解决卡顿问题
- 自动涵盖卡顿、启动耗时、页面切换、慢函数检测等多个流畅性指标



看下Tracer模块的类继承关系：

```
- Tracer
 - FrameTracer
 - AnrTracer
 - EvilMethodTracer
 - StratupTracer
```

通过类名就能看出来每个类的大体功能，下面逐个介绍。



### 帧率检测

FrameTracer部分主要做帧率、掉帧、帧耗时等检测，具体实现逻辑在FrameTracer和UIThreadMonitor。

Demo中是在TestFpsActivity做的演示，onCreate()中通过FrameTracer.onStartTrace()开启检测，页面退出时通过FrameTracer.onCloseTrace()结束检测，并移除监控回调。

我们按照调用栈倒序的逻辑，从使用侧开始，看看具体帧率计算逻辑。

##### TestFpsActivity：

```
sample.tencent.matrix.trace.TestFpsActivity

protected void onCreate(@Nullable Bundle savedInstanceState) {
	// 启动帧率检测，使用FrameTrace计算FPS
	Matrix.with().getPluginByClass(TracePlugin.class).getFrameTracer().onStartTrace();			
// 帧率回调	Matrix.with().getPluginByClass(TracePlugin.class).getFrameTracer().addListener(mDoFrameListener);
}

protected void onDestroy() {
	super.onDestroy();
// 移除帧率回调	Matrix.with().getPluginByClass(TracePlugin.class).getFrameTracer().removeListener(mDoFrameListener);
// 关闭帧率检测
	Matrix.with().getPluginByClass(TracePlugin.class).getFrameTracer().onCloseTrace();
}
```

看下TestFpsActivity的FPS监控回调，代码如下。另外提一下，FrameTracer支持将回调方法IDoFrameListener.doFrameAsync()放到异步线程，前提是外部需要传入一个Executor。

##### IDoFrameListener.doFrameAsync()：

```
sample.tencent.matrix.trace.TestFpsActivity

// 异步线程
private static HandlerThread sHandlerThread = new HandlerThread("test");

// 帧率检测回调
private IDoFrameListener mDoFrameListener = new IDoFrameListener(new Executor() {
        Handler handler = new Handler(sHandlerThread.getLooper());

        @Override
        public void execute(Runnable command) {
        	//将回调放到异步线程执行
            handler.post(command);
        }
    }) {
        @Override
        public void doFrameAsync(String visibleScene, long taskCost, long frameCostMs, int droppedFrames, boolean isContainsFrame) {
            super.doFrameAsync(visibleScene, taskCost, frameCostMs, droppedFrames, isContainsFrame);
            // 计算总的掉帧数
            count += droppedFrames;
        }
    };
```

解释下IDoFrameListener.doFrameAsync()的参数含义：

* visibleScene：场景名称，默认是Activity的类名。这块代码是在Matrix初始化时，通过AppActiveMatrixDelegate.controller监听Activity生命周期，即registerActivityLifecycleCallbacks()。当Activity start时，将activity.getClass().getName()做为visibleScene。FrameTracer再通过AppMethodBeat.getVisibleScene()获取visibleScene。
* taskCost：主线程每一帧耗时
* frameCostMs：帧率检测时用不到
* droppedFrames：掉帧数
* isContainsFrame：是否包含一帧

##### FrameTracer.notifyListener()：

上面doFrameAsync()是在FrameTracer.notifyListener()中执行的，代码在下面。

```
com.tencent.matrix.trace.tracer.FrameTracer

    private void notifyListener(final String visibleScene, final long taskCostMs, final long frameCostMs, final boolean isContainsFrame) {
        long start = System.currentTimeMillis();
        try {
            synchronized (listeners) {
                for (final IDoFrameListener listener : listeners) {
                    if (config.isDevEnv()) {
                        listener.time = SystemClock.uptimeMillis();
                    }
                    // 计算掉帧数
                    final int dropFrame = (int) (taskCostMs / frameIntervalMs);

                    listener.doFrameSync(visibleScene, taskCostMs, frameCostMs, dropFrame, isContainsFrame);
                    if (null != listener.getExecutor()) {
                    	//通过TestFpsActivity传入的Executor，将回调放到异步
                        listener.getExecutor().execute(new Runnable() {
                            @Override
                            public void run() {
                            	// 回调给TestFpsActivity
                                listener.doFrameAsync(visibleScene, taskCostMs, frameCostMs, dropFrame, isContainsFrame);
                            }
                        });
                    }
                }
            }
        } finally {
            long cost = System.currentTimeMillis() - start;
            // debug模式下，回调doFrameAsync中执行超过17ms，log警告
            if (config.isDebug() && cost > frameIntervalMs) {
                MatrixLog.w(TAG, "[notifyListener] warm! maybe do heavy work in doFrameSync! size:%s cost:%sms", listeners.size(), cost);
            }
        }
    }
```

可以看出dropFrame就是taskCostMs/frameIntervalMs。其中，frameIntervalMs默认是17ms，计算逻辑是Choreographer.mFrameIntervalNanos（屏幕刷新时间间隔，默认16ms）+1，计算代码如下：

```
com.tencent.matrix.trace.tracer.FrameTracer

this.frameIntervalMs = TimeUnit.MILLISECONDS.convert(UIThreadMonitor.getMonitor().getFrameIntervalNanos(), TimeUnit.NANOSECONDS) + 1;
```

```
com.tencent.matrix.trace.core.UIThreadMonitor

choreographer = Choreographer.getInstance();
// 通过反射获取Choreographer的mFrameIntervalNanos字段，默认是16ms
frameIntervalNanos = reflectObject(choreographer, "mFrameIntervalNanos");
```

##### FrameTracer.doFrame():

FrameTracer.notifyListener()调用过程是UIThreadMonitor.dispatchEnd() -> FrameTracer.doFrame() -> FrameTracer.notifyListener()。

```
com.tencent.matrix.trace.tracer.FrameTracer

@Override
    public void doFrame(String focusedActivityName, long start, long end, long frameCostMs, long inputCostNs, long animationCostNs, long traversalCostNs) {
        if (isForeground()) {
        	// IDoFrameListener.doFrameAsync()中的参数taskCostMs即为end - start。
            notifyListener(focusedActivityName, end - start, frameCostMs, frameCostMs >= 0);
        }
    }
```

##### UIThreadMonitor.dispatchEnd()

看下UIThreadMonitor.dispatchEnd()：

```
com.tencent.matrix.trace.core.UIThreadMonitor

    private void dispatchEnd() {

        if (isBelongFrame) {
            doFrameEnd(token);
        }

        // 每一帧的开始时间，在dispatchBegin()中赋值
        long start = token;
        // 当前时间，即每一帧的结束时间
        long end = SystemClock.uptimeMillis();

        synchronized (observers) {
            for (LooperObserver observer : observers) {
                if (observer.isDispatchBegin()) {
                	// AppMethodBeat.getVisibleScene()获取visibleScene
                	// isBelongFrame ? end - start : 0 - frameCost, 如果是帧耗时计算，则frameCost为0，否则和taskCost一样
                	// queueCost数组，存储对应Choreographer中三类task(input/animation/traversal)的耗时
                    observer.doFrame(AppMethodBeat.getVisibleScene(), token, SystemClock.uptimeMillis(), isBelongFrame ? end - start : 0, queueCost[CALLBACK_INPUT], queueCost[CALLBACK_ANIMATION], queueCost[CALLBACK_TRAVERSAL]);
                }
            }
        }

        // 计算结束时间
        dispatchTimeMs[3] = SystemClock.currentThreadTimeMillis();
        dispatchTimeMs[1] = SystemClock.uptimeMillis();

        AppMethodBeat.o(AppMethodBeat.METHOD_ID_DISPATCH);

        synchronized (observers) {
            for (LooperObserver observer : observers) {
                if (observer.isDispatchBegin()) {
                    observer.dispatchEnd(dispatchTimeMs[0], dispatchTimeMs[2], dispatchTimeMs[1], dispatchTimeMs[3], token, isBelongFrame);
                }
            }
        }

    }
```

解释下LooperObserver.dispatchEnd()方法中的几个参数的含义，后面都会用到。解释dispatchEnd()之前，看下UIThreadMonitor.dispatchBegin()的逻辑：

```
com.tencent.matrix.trace.core.UIThreadMonitor

private void dispatchBegin() {
        token = dispatchTimeMs[0] = SystemClock.uptimeMillis();
        dispatchTimeMs[2] = SystemClock.currentThreadTimeMillis();
        AppMethodBeat.i(AppMethodBeat.METHOD_ID_DISPATCH);

        synchronized (observers) {
            for (LooperObserver observer : observers) {
                if (!observer.isDispatchBegin()) {
                    observer.dispatchBegin(dispatchTimeMs[0], dispatchTimeMs[2], token);
                }
            }
        }
    }
```

主要是对token、dispatchTimeMs[0]、dispatchTimeMs[2]赋值。其中，前两个是uptimeMillis，dispatchTimeMs[2]赋值当前线程的活动（线程处于running状态）时间点。

看下LooperObserver.dispatchEnd()：

```
com.tencent.matrix.trace.listeners.LooperObserver

public void dispatchEnd(long beginMs, long cpuBeginMs, long endMs, long cpuEndMs, long token, boolean isBelongFrame)
```

* beginMs：主线程一帧方法开始执行时间点，即dispatchTimeMs[0]，在UIThreadMonitor.dispatchBegin()中赋值
* cpuBeginMs：当前线程活动期间，方法开始执行时间点
* endMs：主线程一帧方法结束执行时间点，即dispatchTimeMs[2]，在UIThreadMonitor.dispatchEnd()中赋值。所以，一帧耗时就是endMs-beginMs。
* cpuEndMs：当前线程活动期间，方法解释执行时间点
* token：等同于beginMs
* isBelongFrame：可简单理解为是否属于主线程任务。在Choreographer处理input任务时执行UIThreadMonitor.run()，内部将isBelongFrame置为true，这块逻辑可以看下UIThreadMonitor.onStart()



那么UIThreadMonitor.dispatchEnd()是谁执行的呢？从下面代码可以看出，UIThreadMonitor向LooperMonitor注册了监听器，用于监听每一帧的开始和结束。

```
com.tencent.matrix.trace.core.UIThreadMonitor

LooperMonitor.register(new LooperMonitor.LooperDispatchListener() {
            @Override
            public boolean isValid() {
                return isAlive;
            }

            @Override
            public void dispatchStart() {
                super.dispatchStart();
                UIThreadMonitor.this.dispatchBegin();
            }

            @Override
            public void dispatchEnd() {
                super.dispatchEnd();
                UIThreadMonitor.this.dispatchEnd();
            }

        });
```

其中，LooperDispatchListener的dispatchStart()和dispatchEnd()都是LooperMonitor.dispatch()执行的。由isBegin决定执行dispatchStart()还是dispatchEnd()。

##### LooperMonitor.dispatch():

```
com.tencent.matrix.trace.core.LooperMonitor

private void dispatch(boolean isBegin, String log) {
        for (LooperDispatchListener listener : listeners) {
            if (listener.isValid()) {
                if (isBegin) {
                    if (!listener.isHasDispatchStart) {
                        listener.onDispatchStart(log);
                    }
                } else {
                    if (listener.isHasDispatchStart) {
                        listener.onDispatchEnd(log);
                    }
                }
            } else if (!isBegin && listener.isHasDispatchStart) {
                listener.dispatchEnd();
            }
        }

    }
```

isBegin的逻辑看下下面代码，主要是根据参数x内容，判断是开始还是结束，即isBegin。

##### LooperMonitor.LooperPrinter:

```
com.tencent.matrix.trace.core.LooperMonitor

class LooperPrinter implements Printer {
       
        @Override
        public void println(String x) {
            ...

            if (!isHasChecked) {
            	// 根据参数x内容，判断是开始还是结束，即isBegin
                isValid = x.charAt(0) == '>' || x.charAt(0) == '<';
                isHasChecked = true;
                if (!isValid) {
                    MatrixLog.e(TAG, "[println] Printer is inValid! x:%s", x);
                }
            }

            if (isValid) {
                dispatch(x.charAt(0) == '>', x);
            }

        }
    }
```

在LooperMonitor初始化时，会向main looper注册一个LooperPrinter。

```
looper = Looper.getMainLooper()

looper.setMessageLogging(printer = new LooperPrinter(originPrinter));
```

##### Looper.loop():

看到这里有些同学应该眼前一亮，很多APM计算帧率都采用了这个逻辑，我在[Android图形渲染之Choreographer原理](https://hningoba.github.io/2019/11/28/Android Choreographer原理/) 最后也提到使用“Looper & Printer”计算帧率，再看下Looper中的代码：

```
android.os.Looper.java

public static void loop() {
        final Looper me = myLooper();
        if (me == null) {
            throw new RuntimeException("No Looper; Looper.prepare() wasn't called on this thread.");
        }
        final MessageQueue queue = me.mQueue;

        ...

        for (;;) {
            Message msg = queue.next(); // might block
            if (msg == null) {
                // No message indicates that the message queue is quitting.
                return;
            }

            // This must be in a local variable, in case a UI event sets the logger
            // logging对象外部自定义，传入一个自定义Printer实现每一帧的监听计算
            final Printer logging = me.mLogging;
            if (logging != null) {
            		// 每一帧开始时打印的log
                logging.println(">>>>> Dispatching to " + msg.target + " " +
                        msg.callback + ": " + msg.what);
            }

            ...
            
            try {
            		// 消息执行
                msg.target.dispatchMessage(msg);
                dispatchEnd = needEndTime ? SystemClock.uptimeMillis() : 0;
            } finally {
                if (traceTag != 0) {
                    Trace.traceEnd(traceTag);
                }
            }
            
            ...

            if (logging != null) {
            		// 每一帧结束时打印的log
                logging.println("<<<<< Finished to " + msg.target + " " + msg.callback);
            }

            ...
        }
    }
```

主线程（main looper）每一帧执行(dispatchMessage())开始和结束时，Looper通过一个Printer对象分别打印”>>>>> Dispatching to…”和”<<<<< Finished to”。通过Looper.setMessageLogging()设置我们自定义的Printer，通过监听主线程方法执行前后的两个日志字符串，就可以计算一帧的耗时。

##### 整体代码执行过程：

```
- LooperMonitor.LooperPrinter.println()
- LooperMonitor.dispatch()
- UIThreadMonitor.dispatchBegin()/dispatchEnd()
- FrameTracer.doFrame()
- FrameTracer.notifyListener()
- TestFpsActivity.IDoFrameListener.doFrameSync()/doFrameAsync()
```

##### 总结：

* 帧率检测使用“Looper & Printer”方式，Printer的日志字符串对应主线程每一帧方法执行的开始和结束，进而计算每一帧耗时。



### ANR检测

ANR检测主要代码在AnrTracer。在demo中，TestTraceMainActivity.testANR()内部执行耗时改大一点（大于5000ms），比如改到7800，就能看到AS中输出如下ANR警告log：

<img src="https://raw.githubusercontent.com/hningoba/KnowledgeSummary/master/img/matrix_anr_log.png" width="80%" />

看下AnrTracer如何统计ANR，收集上述log信息的。主要代码在AnrTracer：

##### AnrTracer：

```
com.tencent.matrix.trace.tracer.AnrTracer

@Override
    public void dispatchBegin(long beginMs, long cpuBeginMs, long token) {
        super.dispatchBegin(beginMs, cpuBeginMs, token);
        // 构造AnrTask
        anrTask = new AnrHandleTask(AppMethodBeat.getInstance().maskIndex("AnrTracer#dispatchBegin"), token);
        
        // 延迟5s发消息，Constants.DEFAULT_ANR = 5 * 1000
        // token是方法开始执行的时间
        // (SystemClock.uptimeMillis() - token)：校正方法的开始时间
        anrHandler.postDelayed(anrTask, Constants.DEFAULT_ANR - (SystemClock.uptimeMillis() - token));
    }

    @Override
    public void dispatchEnd(long beginMs, long cpuBeginMs, long endMs, long cpuEndMs, long token, boolean isBelongFrame) {
        super.dispatchEnd(beginMs, cpuBeginMs, endMs, cpuEndMs, token, isBelongFrame);
        // 移除anrTask
        if (null != anrTask) {
            anrTask.getBeginRecord().release();
            anrHandler.removeCallbacks(anrTask);
        }
    }
```

前面讲FrameTracer时提到过，每一帧执行前后分别回调Tracer.dispatchBegin()和Tracer.dispatchEnd()。

在dispatchBegin()中发送了一个延迟消息执行AnrTask，延迟时间约为5s（Constants.DEFAULT_ANR = 5 * 1000），在dispatchEnd()移除对应的任务。所以，如果主线程一帧执行任务超过5s，AnrTask就会执行。

##### AnrTask：

看下AnrTask的逻辑：

```
com.tencent.matrix.trace.tracer.AnrTracer.AnrHandleTask

@Override
public void run() {
    long curTime = SystemClock.uptimeMillis();
    boolean isForeground = isForeground();
    
    // process 进程信息
    int[] processStat = Utils.getProcessPriority(Process.myPid());
    long[] data = AppMethodBeat.getInstance().copyData(beginRecord);
    beginRecord.release();
    // 业务场景，FrameTracer中提到过
    String scene = AppMethodBeat.getVisibleScene();

    // memory 内存信息，其中VmSize从“/proc/{PID}/status”文件中获取
    long[] memoryInfo = dumpMemory();

    // Thread state
    Thread.State status = Looper.getMainLooper().getThread().getState();
    
    // 获取线程调用堆栈
    StackTraceElement[] stackTrace = Looper.getMainLooper().getThread().getStackTrace();
    // 限制堆栈长度
    String dumpStack = Utils.getStack(stackTrace, "|*\t\t", 12);

    // frame
    UIThreadMonitor monitor = UIThreadMonitor.getMonitor();
    // 获取input/animation/traversal三类任务耗时
    long inputCost = monitor.getQueueCost(UIThreadMonitor.CALLBACK_INPUT, token);
    long animationCost = monitor.getQueueCost(UIThreadMonitor.CALLBACK_ANIMATION, token);
    long traversalCost = monitor.getQueueCost(UIThreadMonitor.CALLBACK_TRAVERSAL, token);

    // trace
    LinkedList<MethodItem> stack = new LinkedList();
    if (data.length > 0) {
        TraceDataUtils.structuredDataToStack(data, stack, true, curTime);
        TraceDataUtils.trimStack(stack, Constants.TARGET_EVIL_METHOD_STACK, new TraceDataUtils.IStructuredDataFilter() {
            @Override
            public boolean isFilter(long during, int filterCount) {
                return during < filterCount * Constants.TIME_UPDATE_CYCLE_MS;
            }

            @Override
            public int getFilterMaxCount() {
                return Constants.FILTER_STACK_MAX_COUNT;
            }

            @Override
            public void fallback(List<MethodItem> stack, int size) {
                MatrixLog.w(TAG, "[fallback] size:%s targetSize:%s stack:%s", size, Constants.TARGET_EVIL_METHOD_STACK, stack);
                Iterator iterator = stack.listIterator(Math.min(size, Constants.TARGET_EVIL_METHOD_STACK));
                while (iterator.hasNext()) {
                    iterator.next();
                    iterator.remove();
                }
            }
        });
    }

    StringBuilder reportBuilder = new StringBuilder();
    StringBuilder logcatBuilder = new StringBuilder();
    
    // 获取ANR时长
    long stackCost = Math.max(Constants.DEFAULT_ANR, TraceDataUtils.stackToString(stack, reportBuilder, logcatBuilder));

    // stackKey
    String stackKey = TraceDataUtils.getTreeKey(stack, stackCost);
    
    // 打印ANR日志
    MatrixLog.w(TAG, "%s \npostTime:%s curTime:%s",
            printAnr(scene, processStat, memoryInfo, status, logcatBuilder, isForeground, stack.size(), stackKey, dumpStack, inputCost, animationCost, traversalCost, stackCost), token, curTime); // for logcat

    if (stackCost >= Constants.DEFAULT_ANR_INVALID) {
        MatrixLog.w(TAG, "The checked anr task was not executed on time. "
                + "The possible reason is that the current process has a low priority. just pass this report");
        return;
    }
    
    // report 启动IssuesListActivity页面，展示ANR信息
    ...

}
```

##### 总结：

在异步线程执行AnrTask，计算进程、调用堆栈、内存使用、trace信息，后续将信息log出来，并启动IssuesListActivity展示ANR结果。



### 慢函数检测

像ANR检测部分提到的，将TestTraceMainActivity.testANR()实现耗时改成7800，AS会输出如下慢函数log。从log中可以看出发生了Jankiness，方法耗时7804ms，和我们的修改基本吻合。

<img src="https://raw.githubusercontent.com/hningoba/KnowledgeSummary/master/img/matrix_evil_mthod_log.png" width="80%" />

下面看下慢函数的实现逻辑，主要代码在EvilMethodTracer。

##### EvilMethodTracer：

```
com.tencent.matrix.trace.tracer.EvilMethodTracer

@Override
    public void dispatchEnd(long beginMs, long cpuBeginMs, long endMs, long cpuEndMs, long token, boolean isBelongFrame) {
        super.dispatchEnd(beginMs, cpuBeginMs, endMs, cpuEndMs, token, isBelongFrame);
        long start = config.isDevEnv() ? System.currentTimeMillis() : 0;
        try {
            long dispatchCost = endMs - beginMs;
            // 一帧耗时超过evilThresholdMs（默认700ms），执行慢函数逻辑计算
            if (dispatchCost >= evilThresholdMs) {
                long[] data = AppMethodBeat.getInstance().copyData(indexRecord);
                long[] queueCosts = new long[3];
                System.arraycopy(queueTypeCosts, 0, queueCosts, 0, 3);
                String scene = AppMethodBeat.getVisibleScene();
                // 异步线程执行AnalyseTask
                MatrixHandlerThread.getDefaultHandler().post(new AnalyseTask(isForeground(), scene, data, queueCosts, cpuEndMs - cpuBeginMs, endMs - beginMs, endMs));
            }
        } finally {
            ...
        }
    }
```

从上面代码可以看出，当主线程一帧内耗时超过evilThresholdMs（默认700ms，可以通过TraceConfig配置），执行慢函数的警告，打印上面提到的log。

其中，慢函数的耗时，就是上面代码中的``endMs - beginMs``，这个逻辑在“帧率检测 - UIThreadMonitor.dispatchEnd()”中有具体讲到。

具体慢函数打印的log逻辑如下：

##### AnalyseTask:

```
com.tencent.matrix.trace.tracer.EvilMethodTracer.AnalyseTask

private class AnalyseTask implements Runnable {
        ...

        void analyse() {

            // process
            int[] processStat = Utils.getProcessPriority(Process.myPid());
            String usage = Utils.calculateCpuUsage(cpuCost, cost);
            LinkedList<MethodItem> stack = new LinkedList();
            if (data.length > 0) {
                TraceDataUtils.structuredDataToStack(data, stack, true, endMs);
                TraceDataUtils.trimStack(stack, Constants.TARGET_EVIL_METHOD_STACK, new TraceDataUtils.IStructuredDataFilter() {
                    @Override
                    public boolean isFilter(long during, int filterCount) {
                        return during < filterCount * Constants.TIME_UPDATE_CYCLE_MS;
                    }

                    @Override
                    public int getFilterMaxCount() {
                        return Constants.FILTER_STACK_MAX_COUNT;
                    }

                    @Override
                    public void fallback(List<MethodItem> stack, int size) {
                        MatrixLog.w(TAG, "[fallback] size:%s targetSize:%s stack:%s", size, Constants.TARGET_EVIL_METHOD_STACK, stack);
                        Iterator iterator = stack.listIterator(Math.min(size, Constants.TARGET_EVIL_METHOD_STACK));
                        while (iterator.hasNext()) {
                            iterator.next();
                            iterator.remove();
                        }
                    }
                });
            }


            StringBuilder reportBuilder = new StringBuilder();
            StringBuilder logcatBuilder = new StringBuilder();
            long stackCost = Math.max(cost, TraceDataUtils.stackToString(stack, reportBuilder, logcatBuilder));
            String stackKey = TraceDataUtils.getTreeKey(stack, stackCost);

					// 和AnrTracer一样，打印log，启动IssuesListActivity页面，展示EvilMethod信息
            MatrixLog.w(TAG, "%s", printEvil(scene, processStat, isForeground, logcatBuilder, stack.size(), stackKey, usage, queueCost[0], queueCost[1], queueCost[2], cost)); // for logcat

            ...
        }

    }
```

##### 总结：

AnalyseTask的逻辑和AnrTask类似，只是log中的部分内容不一样，就不具体展开了。



### 启动耗时检测

##### 三种启动方式：

[官方](https://developer.android.com/topic/performance/vitals/launch-time)定义了三种启动方式：

* 冷启动：彻底杀死应用进程后启动APP的方式。系统会为应用创建进程、主线程，会执行Application、launch Activity的初始化方法。
* 热启动：没有杀死应用进程情况下启动APP的方式，比如应用切到后台。热启动中，系统的所有工作就是将您的 Activity 带到前台。这种情况不会执行Application的初始化方法，如果应用的所有 Activity 都还驻留在内存中，则应用可以无须重复对象初始化、布局扩充和呈现。
* 温启动：
  * 有两种常见场景：
    * 点击回退键方式退出应用
    * 应用在后台被系统回收
  * 启动逻辑介于冷启动和热启动之间。如果是回退键方式退出应用再重启，不会执行Application初始化，但是需要初始化Activity。如果是系统回收后的重启，Application和Activity都会初始化，但是可以通过savedInstanceState拿到退出前Activity的状态。



##### 冷启动分析

我们这里只对冷启动做分析，冷启动Demo后，可以看到如下log：

```
Matrix.StartupTracer: [report] applicationCost:42 firstScreenCost:244 allCost:2306 isWarmStartUp:false
```

显示非热启动（即冷启动），Application耗时42ms，首屏耗时244ms，总耗时2306ms。通过log可以知道是StartupTracer做的上报。

计算应用的启动耗时，就需要知道Application的启动开始、启动完成时间点，launch Activity的启动开始、启动完成时间点。通过下图StartupTracer的类注释，可以大概了解如下几个字段的含义和对应的耗时区间：

<img src="https://raw.githubusercontent.com/hningoba/KnowledgeSummary/master/img/matrix_StartupTracer_field_annotation.png" />

简单解释下上图中几个关键统计点：

* Application初始化耗时(applicationCost)：
  * 指Application的初始化耗时。
  * 起始点对应Application.onCreate()，由``ActivityThreadHacker.sApplicationCreateBeginTime``字段标记，该字段在ActivityThreadHacker.hackSysHandlerCallback()赋值。
  * 结束点对应handle launch activity message的时间点（通过hook ActivityThread的mH.mCallback实现拦截主线程message），由``ActivityThreadHacker.sApplicationCreateEndTime``字段标记，该字段在``ActivityThreadHacker.HackCallback.handleMessage()``中赋值。
* 首屏耗时(firstScreenCost)：
  * 指app启动到第一个Activity(launch activity)初始化完成的耗时，粗略包含applicationCost + launchActivity初始化耗时。
  * 起始点和applicationCost的起始点一样，由``ActivityThreadHacker.sApplicationCreateBeginTime``字段标记。
  * 结束点对应开屏页的onWindowFocusChange()（但是代码跟踪显示是IssueListActivity.onWindowFocusChange()），在StartupTracer.onActivityFocused()中标记。
* 冷启动耗时(coldCost)
  * app启动到第一个对用户有意义的Activity（对应图中的careActivity）初始化完成耗时。应用一般将闪屏页即launch activity仅作为logo展示、应用初始化的工作，其后的第一个Activity做为主页Activity，这个Activity就是careActivity，所以把careActivity的初始化完成做为coldCost的结束点。
* 温启动耗时(warmCost)
  * 因为Application不会重新初始化，只统计Activity的初始化耗时。
  * 起始点是launch Activity初始化的开始点。
  * 结束点是launch Activity onWindfocusChanged()执行点。



##### StartupTracer：

前面讲到启动耗时统计逻辑在StartupTracer。看下StartupTracer.onActivityFocused()，该方法在Activity.onWindfocusChanged()内部执行，这部分通过插桩实现。

```
com.tencent.matrix.trace.tracer.StartupTracer

    @Override
    public void onActivityFocused(String activity) {
    	// coldCost == 0时认为是冷启动状态
        if (isColdStartup()) {
            if (firstScreenCost == 0) {
            	// 第一个Activity.onWindfocusChanged()执行时统计首屏耗时
                this.firstScreenCost = uptimeMillis() - ActivityThreadHacker.getEggBrokenTime();
            }
            
            //闪屏页已经初始化，其之后的第一个Activity.onWindfocusChanged()执行时，开始统计冷启动耗时
            if (hasShowSplashActivity) {
                coldCost = uptimeMillis() - ActivityThreadHacker.getEggBrokenTime();
            } else {
            	// 判断闪屏页是否启动，splashActivities在MatrixApplication.onCraete是配置的
                if (splashActivities.contains(activity)) {
                    hasShowSplashActivity = true;
                } else if (splashActivities.isEmpty()) {
                    MatrixLog.i(TAG, "default splash activity[%s]", activity);
                    coldCost = firstScreenCost;
                } else {
                    ...
                }
            }
            
            // 冷启动后，执行AnalyseTask
            if (coldCost > 0) {
                analyse(ActivityThreadHacker.getApplicationCost(), firstScreenCost, coldCost, false);
            }

        } else if (isWarmStartUp()) {
            // 温启动和冷启动逻辑类似，记录的开始、结束点前面概念部分已经讲到
        }

    }
```

再解释下上面部分代码：

* isColdStartup()：coldCost为0时即为冷启动。
* coldCost：闪屏页初始化后，其之后的第一个Activity(careActivity)的onWindfocusChanged()执行时，计算冷启动耗时。
* applicationCost应用耗时：即ActivityThreadHacker.getApplicationCost()，起始点、结束点计算逻辑后面再展开。
* ActivityThreadHacker.getEggBrokenTime()：Application初始化的开始点。
* splashActivities：保持闪屏页列表，在TraceConfig中初始化。demo中是在MatrixApplication.onCraete()手动配置splash Activity。

有了上面这些耗时统计，AnalyseTask利用这些数据，进行堆栈优化、数据整理，打印出前面的启动耗时log。



##### Application初始化耗时：

前面讲到，Application初始化耗时的获取方式是``ActivityThreadHacker.getApplicationCost()``：

```
com.tencent.matrix.trace.hacker.ActivityThreadHacker

public static long getApplicationCost() {
        return ActivityThreadHacker.sApplicationCreateEndTime - ActivityThreadHacker.sApplicationCreateBeginTime;
    }
```

内部记录了Application初始的开始点sApplicationCreateBeginTime和结束点sApplicationCreateEndTime。那么这两个点在什么时机赋值的呢？

通过跟踪代码，可以发现sApplicationCreateBeginTime是在``ActivityThreadHacker.hackSysHandlerCallback()``中赋值。在其内部打个断电，冷启动APP后，方法调用栈如下图：

![matrix_ActivityThreadHacker_method_trace](matrix_ActivityThreadHacker_method_trace.png)

从上图代码调用流程中可以看到，MatrixApplication.onCreate()调用AppMethodBeat.i()（通过插桩实现），进而执行ActivityThreadHacker.hackSysHandlerCallback()，即sApplicationCreateBeginTime对应Application.onCreate()。

sApplicationCreateEndTime在``ActivityThreadHacker.HackCallback.handleMessage()``中赋值。

下面讲下``ActivityThreadHacker``相关逻辑。

##### ActivityThreadHacker：

```
com.tencent.matrix.trace.hacker.ActivityThreadHacker

public static void hackSysHandlerCallback() {
        try {
            sApplicationCreateBeginTime = SystemClock.uptimeMillis();
            sApplicationCreateBeginMethodIndex = AppMethodBeat.getInstance().maskIndex("ApplicationCreateBeginMethodIndex");
            Class<?> forName = Class.forName("android.app.ActivityThread");
            Field field = forName.getDeclaredField("sCurrentActivityThread");
            field.setAccessible(true);
            // step1. 通过反射获取ActivityThread.sCurrentActivityThread对象
            Object activityThreadValue = field.get(forName);
            
            Field mH = forName.getDeclaredField("mH");
            mH.setAccessible(true);
            // step2. 通过反射获取sCurrentActivityThread的mH对象
            Object handler = mH.get(activityThreadValue);
            
            Class<?> handlerClass = handler.getClass().getSuperclass();
            Field callbackField = handlerClass.getDeclaredField("mCallback");
            callbackField.setAccessible(true);
            Handler.Callback originalCallback = (Handler.Callback) callbackField.get(handler);
            HackCallback callback = new HackCallback(originalCallback);
            // step3. 将mH中的mCallback设置成HackCallback
            callbackField.set(handler, callback);
        } catch (Exception e) {
            MatrixLog.e(TAG, "hook system handler err! %s", e.getCause().toString());
        }
    }
```

上面这部分代码，通过注释可以了解到，本质上是通过反射，将ActivityThreadHacker.HackCallback设置成主线程Handler的mCallback。这样，就可以拦截主线程消息做一些工作。对ActivityThread还不太了解的同学可以看看这篇文章：[理解Application创建过程](http://gityuan.com/2017/04/02/android-application/)。

拦截了主线程消息做的事情看看下面代码：

```
com.tencent.matrix.trace.hacker.ActivityThreadHacker

private final static class HackCallback implements Handler.Callback {
        private static final int LAUNCH_ACTIVITY = 100;
        private static final int CREATE_SERVICE = 114;
        private static final int RECEIVER = 113;
        public static final int EXECUTE_TRANSACTION = 159; // for Android 9.0
        private static boolean isCreated = false;
        private static int hasPrint = 10;

        private final Handler.Callback mOriginalCallback;

        HackCallback(Handler.Callback callback) {
            this.mOriginalCallback = callback;
        }

        @Override
        public boolean handleMessage(Message msg) {

            if (!AppMethodBeat.isRealTrace()) {
                return null != mOriginalCallback && mOriginalCallback.handleMessage(msg);
            }

            boolean isLaunchActivity = isLaunchActivity(msg);
            if (hasPrint > 0) {
                MatrixLog.i(TAG, "[handleMessage] msg.what:%s begin:%s isLaunchActivity:%s", msg.what, SystemClock.uptimeMillis(), isLaunchActivity);
                hasPrint--;
            }
            if (isLaunchActivity) {
                ActivityThreadHacker.sLastLaunchActivityTime = SystemClock.uptimeMillis();
                ActivityThreadHacker.sLastLaunchActivityMethodIndex = AppMethodBeat.getInstance().maskIndex("LastLaunchActivityMethodIndex");
            }

            if (!isCreated) {
                if (isLaunchActivity || msg.what == CREATE_SERVICE || msg.what == RECEIVER) { // todo for provider
                    ActivityThreadHacker.sApplicationCreateEndTime = SystemClock.uptimeMillis();
                    ActivityThreadHacker.sApplicationCreateScene = msg.what;
                    isCreated = true;
                }
            }

            return null != mOriginalCallback && mOriginalCallback.handleMessage(msg);
        }
}
```





##### Launch Activity初始化耗时检测：

ActivityThreadHacker中有个方法是获取launch activity的启动时间点：

```
public static long getLastLaunchActivityTime() {
        return ActivityThreadHacker.sLastLaunchActivityTime;
    }
```

那么，这个时间点是怎么获取的呢？如何判断一个Activity是launch Activity？看下面代码：

```
com.tencent.matrix.trace.hacker.ActivityThreadHacker

private boolean isLaunchActivity(Message msg) {
    if (Build.VERSION.SDK_INT > Build.VERSION_CODES.O_MR1) {
        if (msg.what == EXECUTE_TRANSACTION && msg.obj != null) {
            try {
                if (null == method) {
                    Class clazz = Class.forName("android.app.servertransaction.ClientTransaction");
                    method = clazz.getDeclaredMethod("getCallbacks");
                    method.setAccessible(true);
                }
                List list = (List) method.invoke(msg.obj);
                if (!list.isEmpty()) {
                    return list.get(0).getClass().getName().endsWith(".LaunchActivityItem");
                }
            } catch (Exception e) {
                MatrixLog.e(TAG, "[isLaunchActivity] %s", e);
            }
        }
        return msg.what == LAUNCH_ACTIVITY;
    } else {
        return msg.what == LAUNCH_ACTIVITY;
    }
}
```







##### 总结：

* Application初始化开始、结束节点：
* Launch Activity初始化开始、结束节点：



插桩结果：

<img src="https://raw.githubusercontent.com/hningoba/KnowledgeSummary/master/img/matrix_TestTraceMainActivity_dex.png" />



插桩代码：

matrix-gradle-plugin : MethodTracer.insertWindowFocusChangeMethod()

```
 private void insertWindowFocusChangeMethod(ClassVisitor cv, String classname) {
        MethodVisitor methodVisitor = cv.visitMethod(Opcodes.ACC_PUBLIC, TraceBuildConstants.MATRIX_TRACE_ON_WINDOW_FOCUS_METHOD,
                TraceBuildConstants.MATRIX_TRACE_ON_WINDOW_FOCUS_METHOD_ARGS, null, null);
        methodVisitor.visitCode();
        methodVisitor.visitVarInsn(Opcodes.ALOAD, 0);
        methodVisitor.visitVarInsn(Opcodes.ILOAD, 1);
        methodVisitor.visitMethodInsn(Opcodes.INVOKESPECIAL, TraceBuildConstants.MATRIX_TRACE_ACTIVITY_CLASS, TraceBuildConstants.MATRIX_TRACE_ON_WINDOW_FOCUS_METHOD,
                TraceBuildConstants.MATRIX_TRACE_ON_WINDOW_FOCUS_METHOD_ARGS, false);
        traceWindowFocusChangeMethod(methodVisitor, classname);
        methodVisitor.visitInsn(Opcodes.RETURN);
        methodVisitor.visitMaxs(2, 2);
        methodVisitor.visitEnd();

    }
```



### Trace插桩

在讲应用启动耗时，提到了几个插桩点，主要实现在MethodTracer，看下这部分代码。对ASM和gradle Transform不了解的同学可以先看看我写的这两篇文章了解下基本用法：[自定义Gradle插件介绍](https://github.com/hningoba/KnowledgeSummary/blob/master/Android随便看看/自定义Gradle插件介绍.md)、[ASM用法介绍](https://github.com/hningoba/KnowledgeSummary/blob/master/Android随便看看/ASM用法介绍.md)。

前面讲启动耗时检测统计Application耗时时提到，demo中MatrixApplication执行AppMethodBeat.i()，进而执行``ActivityThreadHacker.hackSysHandlerCallback()``，在这里记录sApplicationCreateBeginTime。

跟踪代码就会发现，MatrixApplication.onCreate()中找不到AppMethodBeat.i()的调用。这个逻辑其实是通过ASM进行插桩实现的。具体代码执行逻辑在matrix-gradle-plugin的MethodTracer：

```
com.tencent.matrix.trace.MethodTracer.TraceClassAdapter

@Override
        protected void onMethodEnter() {
            TraceMethod traceMethod = collectedMethodMap.get(methodName);
            if (traceMethod != null) {
                traceMethodCount.incrementAndGet();
                mv.visitLdcInsn(traceMethod.id);
                // 插入静态方法AppMethodBeat.i()
                mv.visitMethodInsn(INVOKESTATIC, TraceBuildConstants.MATRIX_TRACE_CLASS, "i", "(I)V", false);
            }
        }
```

其中，TraceBuildConstants.MATRIX_TRACE_CLASS为"com/tencent/matrix/trace/core/AppMethodBeat"。``mv.visitMethodInsn``这行代码表示插入静态方法``AppMethodBeat.i()``。



##### MethodTracer

下面简单看下MethodTracer都执行了哪些操作







### 遗留问题

* 编译期动态修改字节码，改了哪些内容？
* UIThreadMonitor.onStart()逻辑，如何配合Choreographer的？
* AnrTask中trace部分的逻辑，比如TraceDataUtils？



## 参考

[Matrix](https://github.com/Tencent/matrix)

[Matrix Wiki](https://github.com/Tencent/matrix/wiki/Matrix-Android-TraceCanary)