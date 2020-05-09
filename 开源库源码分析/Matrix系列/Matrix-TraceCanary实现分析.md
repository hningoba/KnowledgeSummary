---
title: Matrix - TraceCanary源码分析
categories: 技术
---

本文主要介绍Matrix的Trace部分，主要涉及帧率、ANR、慢函数、启动耗时的检测逻辑。

<!--more-->

### Trace Canary主要特性

Trace Canary主要特性：

- 编译期动态修改字节码, 高性能记录执行耗时与调用堆栈
- 准确的定位到发生卡顿的函数，提供执行堆栈、执行耗时、执行次数等信息，帮助快速解决卡顿问题
- 自动涵盖卡顿、启动耗时、页面切换、慢函数检测等多个流畅性指标



Tracer模块主要结构：

<img src="https://raw.githubusercontent.com/hningoba/KnowledgeSummary/master/img/matrix-tracer.png" style="zoom:85%;" />

其中：

* FrameTracer负责帧率检测
* AnrTracer负责ANR问题检测
* EvilMethodTracer负责检测慢函数
* StartupTracer负责应用启动耗时检测



### 帧率检测

FrameTracer部分主要做帧率、掉帧、帧耗时等检测，具体实现逻辑在FrameTracer和UIThreadMonitor。

Demo中的入口在TestFpsActivity，onCreate()中通过FrameTracer.onStartTrace()开启检测，页面退出时通过FrameTracer.onCloseTrace()结束检测，并移除监控回调。

从使用侧开始，看看帧率的计算逻辑。

##### TestFpsActivity：

```
sample.tencent.matrix.trace.TestFpsActivity

protected void onCreate(@Nullable Bundle savedInstanceState) {
	// 启动帧率检测，使用FrameTrace计算FPS
	Matrix.with().getPluginByClass(TracePlugin.class).getFrameTracer().onStartTrace();
	// 添加帧率回调
	Matrix.with().getPluginByClass(TracePlugin.class).getFrameTracer().addListener(mDoFrameListener);
}

protected void onDestroy() {
	super.onDestroy();
	// 移除帧率回调
	Matrix.with().getPluginByClass(TracePlugin.class).getFrameTracer().removeListener(mDoFrameListener);
	// 关闭帧率检测
	Matrix.with().getPluginByClass(TracePlugin.class).getFrameTracer().onCloseTrace();
}
```

看下TestFpsActivity的FPS监控回调部分：

```
sample.tencent.matrix.trace.TestFpsActivity

// 异步线程
private static HandlerThread sHandlerThread = new HandlerThread("test");

// 帧率检测回调
private IDoFrameListener mDoFrameListener = new IDoFrameListener(new Executor() {
        Handler handler = new Handler(sHandlerThread.getLooper());

        @Override
        public void execute(Runnable command) {
            //将回调方法放到异步线程执行
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

回调接口``IDoFrameListener``支持将同步和异步回调方法：doFrameSync()和doFrameAsync，需要使用侧传入一个Executor。

解释下IDoFrameListener.doFrameAsync()的参数含义：

* visibleScene：场景名称，默认是Activity的类名
* taskCost：主线程每一帧耗时
* frameCostMs：帧率检测时用不到
* droppedFrames：掉帧数
* isContainsFrame：是否包含一帧

这里提一下visibleScene的获取逻辑：Matrix初始化时，``AppActiveMatrixDelegate.controller()``内部通过``Application. registerActivityLifecycleCallbacks()``监听Activity生命周期。当Activity start时，将``activity.getClass().getName()``做为visibleScene。FrameTracer再通过``AppMethodBeat.getVisibleScene()``获取visibleScene。

继续看下回调方法``doFrameAsync()``的上游逻辑。

##### FrameTracer.notifyListener()：

``IDoFrameListener.doFrameAsync()``是在``FrameTracer.notifyListener()``中调用的：

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

从上面代码可以看出掉帧数dropFrame就是``taskCostMs/frameIntervalMs``。

* taskCostMs：一帧的执行耗时
* frameIntervalMs：默认是17ms，计算逻辑是Choreographer.mFrameIntervalNanos+1。

其中，Choreographer.mFrameIntervalNanos是屏幕刷新时间间隔，默认16ms。计算代码如下：

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

上面计算掉帧的方式：taskCostMs/17，其实并不是非常严谨，比如taskCostMs=33时，应该丢了两帧，但是计算结果是1。

上面提到的``FrameTracer.notifyListener()``由``FrameTracer.doFrame()``触发，看下``FrameTracer.doFrame()``：

##### FrameTracer.doFrame():

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

这个方法没有做太多事情，主要是计算掉帧数用的``taskCostMs``是``end - start``的结果。

``FrameTracer.doFrame()``由``UIThreadMonitor.dispatchEnd()``触发，看下后者的实现。

##### UIThreadMonitor.dispatchEnd()

```
com.tencent.matrix.trace.core.UIThreadMonitor

    private void dispatchEnd() {
        if (isBelongFrame) {
            doFrameEnd(token);
        }

        // 对应每一帧的开始时间，在dispatchBegin()中赋值
        long start = token;
        // 对应每一帧的结束时间
        long end = SystemClock.uptimeMillis();

        synchronized (observers) {
            for (LooperObserver observer : observers) {
                if (observer.isDispatchBegin()) {
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

这部分内容主要做了这几件事情：

* 获取每一帧耗时的开始、结束时间戳，即start、end，将差值通过observer.doFrame()传给消费方
* 执行observer.doFrame()，该方法主要消费方是AnrTracer、EvilMethodTracer、FrameTracer
* 赋值dispatchTimeMs[1]，dispatchTimeMs[3]，其中dispatchTimeMs数组含义后面会提到
* 执行observer.dispatchEnd()，该方法主要消费方是AnrTracer、EvilMethodTracer



看完了UIThreadMonitor.dispatchEnd()，再看下UIThreadMonitor.dispatchBegin()：

```
com.tencent.matrix.trace.core.UIThreadMonitor

private void dispatchBegin() {
        // 对dispatchTimeMs[0]、dispatchTimeMs[2]赋值
        token = dispatchTimeMs[0] = SystemClock.uptimeMillis();
        dispatchTimeMs[2] = SystemClock.currentThreadTimeMillis();
        AppMethodBeat.i(AppMethodBeat.METHOD_ID_DISPATCH);

        //每一帧开始执行时的回调
        synchronized (observers) {
            for (LooperObserver observer : observers) {
                if (!observer.isDispatchBegin()) {
                    observer.dispatchBegin(dispatchTimeMs[0], dispatchTimeMs[2], token);
                }
            }
        }
    }
```

这里简单说下``SystemClock.uptimeMillis()``和``SystemClock.currentThreadTimeMillis()``的区别，详细内容可以看[官方文档](https://developer.android.com/reference/android/os/SystemClock.html)：

* SystemClock.uptimeMillis()：记录从开机到现在的时长，系统深睡眠（CPU睡眠、黑屏、系统等待外部输入对其唤醒）的时间不算在内，且不受系统设置时间调整的影响。是大多数时间间隔计算方法的基础，比如Thread.sleep(long)、Object.wait(long)、System.nanoTime()
* SystemClock.currentThreadTimeMillis()：这个比较容易理解，当前线程的活动时长（线程处于running状态）



##### LooperObserver.dispatchEnd()

解释下LooperObserver.dispatchEnd()的参数含义，后续对理解AnrTracer、EvilMethodTracer的实现有帮助：

```
com.tencent.matrix.trace.listeners.LooperObserver

public void dispatchEnd(long beginMs, long cpuBeginMs, long endMs, long cpuEndMs, long token, boolean isBelongFrame)
```

* beginMs：主线程一帧任务的开始时间点，对应dispatchTimeMs[0]，在UIThreadMonitor.dispatchBegin()中赋值
* cpuBeginMs：当前线程活动期间，方法开始执行时间点，对应dispatchTimeMs[2]
* endMs：主线程一帧方法结束执行时间点，对应dispatchTimeMs[1]，在UIThreadMonitor.dispatchEnd()中赋值。所以，一帧耗时就是endMs-beginMs。
* cpuEndMs：当前线程活动期间，方法解释执行时间点，对应dispatchTimeMs[3]
* token：等同于beginMs
* isBelongFrame：可简单理解为是否属于主线程任务。在Choreographer处理input任务时执行UIThreadMonitor.run()，内部将isBelongFrame置为true，这块逻辑可以看下UIThreadMonitor.onStart()



上面提到，一帧任务执行结束时会执行``UIThreadMonitor.dispatchEnd()``，该方法在``LooperMonitor``注册的监听器中执行的。看下这块逻辑：

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

UIThreadMonitor向LooperMonitor注册了监听器，用于监听每一帧的开始和结束。

其中，LooperDispatchListener的dispatchStart()和dispatchEnd()都是LooperMonitor.dispatch()执行的。由isBegin决定执行dispatchStart()还是dispatchEnd()。看下``LooperMonitor.dispatch()``：

##### LooperMonitor.dispatch():

```
com.tencent.matrix.trace.core.LooperMonitor

private void dispatch(boolean isBegin, String log) {
        for (LooperDispatchListener listener : listeners) {
            if (listener.isValid()) {
                if (isBegin) {
                    if (!listener.isHasDispatchStart) {
                        // 内部回调dispatchStart()
                        listener.onDispatchStart(log);
                    }
                } else {
                    if (listener.isHasDispatchStart) {
                        // 内部回调dispatchEnd()
                        listener.onDispatchEnd(log);
                    }
                }
            } else if (!isBegin && listener.isHasDispatchStart) {
                listener.dispatchEnd();
            }
        }

    }
```

isBegin的逻辑看下面代码：

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
                // x.charAt(0) == '>'时，isBegin为true
                dispatch(x.charAt(0) == '>', x);
            }

        }
    }
```

即，入参字符串``x``以``>``开头时，isBegin为true。

上面逻辑是自定义LooperPrinter的实现方法。在LooperMonitor初始化时，会向main looper注册一个LooperPrinter：

```
com.tencent.matrix.trace.core.LooperMonitor

public class LooperMonitor {
	private LooperPrinter printer;
	//主线程looper
	looper = Looper.getMainLooper()

	// 将前面提到的自定义LooperPrinter塞给主线程looper
	looper.setMessageLogging(printer = new LooperPrinter(originPrinter));
}

```

上面代码主要是把自定义LooperPrinter塞给主线程looper。对Looper比较熟悉的同学其实已经有眉目了，我们回顾下looper的逻辑：

##### Looper.loop()

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

很多APM计算帧率都采用了这个逻辑，我在[Android图形渲染之Choreographer原理](https://hningoba.github.io/2019/11/28/Android Choreographer原理/) 最后也提到使用“Looper & Printer”计算帧率。

Looper在每个消息消费前后``msg.target.dispatchMessage(msg)``，会通过一个Printer对象分别打印``>>>>> Dispatching to…``和``<<<<< Finished to``两个日志。通过Looper.setMessageLogging()可以设置我们自定义的Printer，通过监听主线程方法执行前后的两个日志字符串，就可以计算一帧的耗时。

前面LooperMonitor中就通过Looper.setMessageLogging()将自定义的Printer传给了Looper，所以可以拦截每个消息执行前后的日志。

到这里，每一帧的监听、掉帧逻辑及帧率计算方式就讲完了。

##### 总结：

回顾下整体代码的执行过程：

```
- Looper.loop()		//主线程消息执行也是looper机制
- LooperMonitor.LooperPrinter.println()		//消息执行前后会打印两个日志
- LooperMonitor.dispatch()	//拦截到两个日志后进行分发
- UIThreadMonitor.dispatchBegin()/dispatchEnd()	//记录相关时间节点，分发每一帧执行前后的回调方法
- FrameTracer.doFrame()
- FrameTracer.notifyListener()
- TestFpsActivity.IDoFrameListener.doFrameSync()/doFrameAsync()	//业务侧获取帧率回调
```

总体来说，帧率检测使用了“Looper & Printer”方式。Printer的日志字符串对应主线程每一帧方法执行的开始和结束，通过给主线程Looper设置自定义Printer，即可拦截日志。根据日志内容即可得知每一帧执行的前后节点，进而记录每一帧的开始、结束时间戳，从而可以计算每一帧的耗时、帧率、掉帧等核心指标。



### ANR检测

ANR检测的主要实现在AnrTracer。

demo中，将TestTraceMainActivity.testANR()内部执行耗时改大一点（大于5000ms），比如改到7800，就能看到AS中输出如下ANR警告log：

<img src="https://raw.githubusercontent.com/hningoba/KnowledgeSummary/master/img/matrix_anr_log.png" style="zoom:70%;" />

看下AnrTracer如何统计ANR，收集上述log信息：

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

前面讲FrameTracer时提到过，每一帧执行前后分别回调``Tracer.dispatchBegin()``和``Tracer.dispatchEnd()``。

上面代码在``dispatchBegin()``内部发送了一个延迟消息执行AnrTask，延迟时间约为5s（Constants.DEFAULT_ANR = 5 * 1000），在``dispatchEnd()``移除对应的任务。所以，如果主线程一帧执行任务超过5s，AnrTask就会执行。

要注意的一点是，ANR log中，``may be happen ANR(5007ms)!``，这里的5007ms时间并不是一帧任务的耗时（因为我们把测试方法耗时改成了7800ms），而是AnrTracer中AnrTask的消息延迟时间。

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
    
    // 线程调用堆栈
    StackTraceElement[] stackTrace = Looper.getMainLooper().getThread().getStackTrace();
    // 堆栈裁剪
    String dumpStack = Utils.getStack(stackTrace, "|*\t\t", 12);

    // frame
    UIThreadMonitor monitor = UIThreadMonitor.getMonitor();
    // 获取input/animation/traversal三类任务耗时
    long inputCost = monitor.getQueueCost(UIThreadMonitor.CALLBACK_INPUT, token);
    long animationCost = monitor.getQueueCost(UIThreadMonitor.CALLBACK_ANIMATION, token);
    long traversalCost = monitor.getQueueCost(UIThreadMonitor.CALLBACK_TRAVERSAL, token);

    // trace
    ...

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

主要内容都加了注释，可以参考下。

##### 总结：

* ANR的检测逻辑：在主线程一帧任务开始前发送（5s）的延迟消息，任务结束时移除该消息。如果主线程一帧任务执行时间超过了消息的延迟时间（认为这一帧导致了ANR），延迟消息将会执行，触发ANR信息收集任务

* 延迟消息（AnrTask）在异步线程执行，收集进程、调用堆栈、内存使用、trace信息，后续将信息log出来，并启动IssuesListActivity展示ANR结果。



### 慢函数检测

将``TestTraceMainActivity.testANR()``实现耗时改成7800ms，AS会输出如下慢函数log。从log中可以看出发生了Jankiness，方法耗时7804ms，和我们的修改基本吻合。

<img src="https://raw.githubusercontent.com/hningoba/KnowledgeSummary/master/img/matrix_evil_mthod_log.png" style="zoom:75%;" />

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
            // 一帧耗时超过evilThresholdMs（默认700ms），执行慢函数计算逻辑
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

主要逻辑在dispatchEnd()里面。从上面代码可以看出，当主线程一帧内耗时超过evilThresholdMs（默认700ms，可以通过TraceConfig配置），执行慢函数的警告，打印上面提到的log。

其中，慢函数的耗时，就是上面代码中的``endMs - beginMs``，这个逻辑在“帧率检测 - UIThreadMonitor.dispatchEnd()”中有具体讲到。

具体慢函数打印的log逻辑如下：

##### AnalyseTask:

```
com.tencent.matrix.trace.tracer.EvilMethodTracer.AnalyseTask

private class AnalyseTask implements Runnable {
        ...

        void analyse() {

            // process 进程信息
            int[] processStat = Utils.getProcessPriority(Process.myPid());
            String usage = Utils.calculateCpuUsage(cpuCost, cost);
            LinkedList<MethodItem> stack = new LinkedList();
            ...

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

其中，AnalyseTask的逻辑和AnrTask类似，只是log中的部分内容不一样，就不具体展开了。

##### 总结：

慢函数的检测逻辑比较简单，在主线程一帧执行结束时，判断帧耗时是否大于预先设置的慢函数阈值，超过该阈值，即认为慢函数发生。



### 启动耗时检测

##### App三种启动方式：

[官方](https://developer.android.com/topic/performance/vitals/launch-time)定义了三种启动方式：

* 冷启动：彻底杀死应用进程后启动APP的方式。系统会为应用创建进程、主线程，执行Application、launch Activity的初始化方法。
* 热启动：没有杀死应用进程情况下启动APP的方式，比如应用切到后台。热启动中，系统的所有工作就是将您的 Activity 带到前台。这种情况不会执行Application的初始化方法，如果应用的所有 Activity 都还驻留在内存中，则应用可以无须重复对象初始化、布局inflate和呈现。
* 温启动：
  * 有两种常见场景：
    * 点击回退键方式退出应用
    * 应用在后台被系统回收
  * 启动逻辑介于冷启动和热启动之间。如果是回退键方式退出应用再重启，不会执行Application初始化，但是需要初始化Activity。如果是系统回收后的重启，Application和Activity都会初始化，但是可以通过savedInstanceState拿到退出前Activity的状态。



##### 冷启动关键节点：

三种启动方式中，我们这里只对冷启动做分析。启动Demo后，可以看到如下log：

```
Matrix.StartupTracer: [report] applicationCost:42 firstScreenCost:244 allCost:2306 isWarmStartUp:false
```

log显示是冷启动(isWarmStartUp:false)方式，Application初始化耗时(applicationCost) 42ms，首屏耗时(firstScreenCost) 244ms，总耗时(allCost) 2306ms。通过log可以知道是StartupTracer做的上报。

StartupTracer给各个耗时统计点提供了注释：

<img src="https://raw.githubusercontent.com/hningoba/KnowledgeSummary/master/img/matrix_StartupTracer_field_annotation.png" />

简单解释下上图中几个关键统计点：

* Application初始化耗时(applicationCost)：
  * 起始点对应``Application.onCreate()``，由``ActivityThreadHacker.sApplicationCreateBeginTime``字段记录起始点时间戳，该字段在``ActivityThreadHacker.hackSysHandlerCallback()``赋值。
  * 结束点比较有意思，matrix是通过hook ActivityThread的mH.mCallback实现拦截主线程message，Application和Activity的初始化任务都是通过主线程Handler机制调度的。当接收到launch activity调度message时认为Application初始化完成，以此作为结束点。结束点由``ActivityThreadHacker.sApplicationCreateEndTime``字段记录其时间戳，该字段在``ActivityThreadHacker.HackCallback.handleMessage()``中赋值。
* 首屏耗时(firstScreenCost)：
  * 指app启动到第一个Activity(launch activity)初始化完成的耗时，可以认为包含了applicationCost + launch activity的初始化耗时。
  * 起始点和applicationCost的起始点一样，由``ActivityThreadHacker.sApplicationCreateBeginTime``字段记录时间戳。
  * 结束点对应launch activity的``onWindowFocusChange()``，在``StartupTracer.onActivityFocused()``中标记。``Activity.onWindowFocusChange()``执行时，认为Activity对用户可见。
* 冷启动耗时(coldCost)
  * app启动到第一个对用户有意义的Activity（对应图中的careActivity）初始化完成耗时。
  * 应用一般仅在闪屏页即launch activity做logo展示、应用初始化的工作，其后的第一个Activity做为主页Activity，这个Activity就是careActivity。所以把careActivity的初始化完成做为coldCost的结束点。
* 温启动耗时(warmCost)
  * 因为Application不会重新初始化，所以只统计Activity的初始化耗时，和firstActivity含义一致。
  * 起始点是launchActivity初始化的开始点。
  * 结束点是launchActivity onWindfocusChanged()执行点。

上面解释了几个耗时节点的概念，接下来分别看看具体的实现逻辑。



##### Application初始化耗时：

前面讲到，Application初始化耗时的获取方式是``ActivityThreadHacker.getApplicationCost()``：

```
com.tencent.matrix.trace.hacker.ActivityThreadHacker

public static long getApplicationCost() {
        return ActivityThreadHacker.sApplicationCreateEndTime - ActivityThreadHacker.sApplicationCreateBeginTime;
    }
```

其实就是``sApplicationCreateEndTime``和``sApplicationCreateBeginTime``的差值。

其中，``sApplicationCreateBeginTime``在``ActivityThreadHacker.hackSysHandlerCallback()``中赋值：

```
com.tencent.matrix.trace.hacker.ActivityThreadHacker

public static void hackSysHandlerCallback() {
        ...
        sApplicationCreateBeginTime = SystemClock.uptimeMillis();
        ...
}
```

那么，``ActivityThreadHacker.hackSysHandlerCallback()``什么时机执行呢？

在方法内部打个断点，冷启动APP后，可看到方法调用栈如下图：

<img src="https://raw.githubusercontent.com/hningoba/KnowledgeSummary/master/img/matrix_ActivityThreadHacker_method_trace.png" alt="matrix_ActivityThreadHacker_method_trace" style="zoom:85%;" />

从上图方法调用栈可以看到，``MatrixApplication.onCreate()``调用``AppMethodBeat.i()``（插桩实现），后续执行``ActivityThreadHacker.hackSysHandlerCallback()``，所以``sApplicationCreateBeginTime``在``Application.onCreate()``执行后被赋值。符合前面解释``applicationCost``概念时提到的起始点对应``Application.onCreate()``。

``sApplicationCreateEndTime``在``ActivityThreadHacker.HackCallback.handleMessage()``中赋值。

下面看下``ActivityThreadHacker``相关逻辑。

在讲``ActivityThreadHacker.HackCallback.handleMessage()``前先了解下``ActivityThreadHacker``这个类。先看下``ActivityThreadHacker.hackSysHandlerCallback()``：

```
com.tencent.matrix.trace.hacker.ActivityThreadHacker

public static void hackSysHandlerCallback() {
        try {
            // application初始化耗时过程起始点
            sApplicationCreateBeginTime = SystemClock.uptimeMillis();
            sApplicationCreateBeginMethodIndex = AppMethodBeat.getInstance().maskIndex("ApplicationCreateBeginMethodIndex");
            
            // step1. 通过反射获取ActivityThread.sCurrentActivityThread对象
            Class<?> forName = Class.forName("android.app.ActivityThread");
            Field field = forName.getDeclaredField("sCurrentActivityThread");
            field.setAccessible(true);
            Object activityThreadValue = field.get(forName);
            
            // step2. 通过反射获取sCurrentActivityThread的mH对象
            Field mH = forName.getDeclaredField("mH");
            mH.setAccessible(true);
            Object handler = mH.get(activityThreadValue);
            
            // step3. 将mH中的mCallback设置成HackCallback，拦截主线程消息
            Class<?> handlerClass = handler.getClass().getSuperclass();
            Field callbackField = handlerClass.getDeclaredField("mCallback");
            callbackField.setAccessible(true);
            Handler.Callback originalCallback = (Handler.Callback) callbackField.get(handler);
            HackCallback callback = new HackCallback(originalCallback);
            callbackField.set(handler, callback);
        } catch (Exception e) {
            MatrixLog.e(TAG, "hook system handler err! %s", e.getCause().toString());
        }
    }
```

上面这部分代码，除了记录Application的初始化开始节点``sApplicationCreateBeginTime``，还通过反射，将``ActivityThreadHacker.HackCallback``实例设置成主线程``Handler的mCallback``。这样，就可以拦截主线程消息做一些工作。

对``ActivityThread``、应用启动流程不熟悉的同学可以看看这篇文章：[Android应用启动流程分析](https://hningoba.github.io/2020/04/26/Android应用启动流程分析/)。

看下``ActivityThreadHacker.HackCallback``拦截主线程消息做了哪些工作：

```
com.tencent.matrix.trace.hacker.ActivityThreadHacker

private final static class HackCallback implements Handler.Callback {
        private static final int LAUNCH_ACTIVITY = 100;
        private static final int CREATE_SERVICE = 114;
        private static final int RECEIVER = 113;
        public static final int EXECUTE_TRANSACTION = 159; // for Android 9.0
        private static boolean isCreated = false;
        private static int hasPrint = 10;

        ...

        @Override
        public boolean handleMessage(Message msg) {
            ...
            // 是否启动页Activity，实现逻辑后面会讲到
            boolean isLaunchActivity = isLaunchActivity(msg);
            // 记录启动页Activity结束时间点
            if (isLaunchActivity) {
                ActivityThreadHacker.sLastLaunchActivityTime = SystemClock.uptimeMillis();
                ActivityThreadHacker.sLastLaunchActivityMethodIndex = AppMethodBeat.getInstance().maskIndex("LastLaunchActivityMethodIndex");
            }

            if (!isCreated) {
                // 当前message是启动页Activity时，认为Application初始化完成
                if (isLaunchActivity || msg.what == CREATE_SERVICE || msg.what == RECEIVER) {
                    ActivityThreadHacker.sApplicationCreateEndTime = SystemClock.uptimeMillis();
                    ActivityThreadHacker.sApplicationCreateScene = msg.what;
                    isCreated = true;
                }
            }

            return null != mOriginalCallback && mOriginalCallback.handleMessage(msg);
        }
}
```

上面代码主要意思是，当ActivityThread.HackCallback收到launch activity的初始化message时，记录Application的初始化结束点时间戳。

**总结：**

* Application初始化过程的起始点对应``Application.onCreate()``
* 结束点是接收到launch activity初始化消息的时刻



##### Launch Activity初始化耗时：

接下来看下launch activity即闪屏页的初始化耗时计算逻辑。

ActivityThreadHacker中有个方法是获取launch activity的启动时间点：

```
public static long getLastLaunchActivityTime() {
        return ActivityThreadHacker.sLastLaunchActivityTime;
    }
```

从上面《Application初始化耗时》部分，``ActivityThreadHacker.HackCallback.handleMessage()``中可以看到，接收到launch activity的消息时，对``ActivityThreadHacker.sLastLaunchActivityTime``赋值。

那怎么判断一个消息是launch activity的消息呢？看下面代码：

```
com.tencent.matrix.trace.hacker.ActivityThreadHacker

private boolean isLaunchActivity(Message msg) {
    // 区分版本，9.0及以上检测ClientTransaction
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
        // 9.0以下版本
        return msg.what == LAUNCH_ACTIVITY;
    }
}
```

在[Android应用启动流程分析](https://hningoba.github.io/2020/04/26/Android应用启动流程分析/)这篇文章中我们提到过，Android9.0引入了ClientTransaction辅助管理应用和页面的生命周期。Application初始化时，AMS通过``ActivityThread.ApplicationThread``向``ActivityThread.mH``发送一个what为``EXECUTE_TRANSACTION``，obj为``ClientTransaction``的消息，且该消息中的ClientTransaction带有一个LaunchActivityItem的Callback。

所以上面``isLaunchActivity()``就是通过这个思路检测launch activity的，发现符合条件的主线程消息，且消息的第一个callback类名以LaunchActivityItem为结尾，即认为发起launch activity初始化流程。



##### StartupTracer：

上面分别讲了应用冷启动各个阶段的耗时统计逻辑，在文章开头处我们提到StartupTracer负责应用启动耗时检测，接下来看看StartupTracer主要做了哪些事情。

看下StartupTracer的主要方法onActivityFocused()，该方法在``Activity.onWindfocusChanged()``中被调用。这个调用是通过插桩实现的，关于Matrix的插桩内容也比较有意思，后面单独文章来介绍。

``Activity.onWindfocusChanged()``被执行，就认为页面已经展现在用户面前，通常做为页面初始化过程的结束点。

```
com.tencent.matrix.trace.tracer.StartupTracer

    @Override
    public void onActivityFocused(String activity) {
    	// coldCost == 0时认为是冷启动状态
        if (isColdStartup()) {
            if (firstScreenCost == 0) {
            	// 第一个Activity的onWindowFocusChanged()执行时统计首屏耗时
                this.firstScreenCost = uptimeMillis() - ActivityThreadHacker.getEggBrokenTime();
            }
            
            //闪屏页已经初始化，其之后的第一个Activity.onWindowFocusChanged()执行时，开始统计冷启动耗时
            if (hasShowSplashActivity) {
                coldCost = uptimeMillis() - ActivityThreadHacker.getEggBrokenTime();
            } else {
            	// 判断闪屏页是否启动，splashActivities在MatrixApplication.onCraete是配置的
                if (splashActivities.contains(activity)) {
                    hasShowSplashActivity = true;
                } else if (splashActivities.isEmpty()) {
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
* firstScreenCost：应用启动到闪屏页Activity.onWindfocusChanged()执行的时长
* coldCost：闪屏页初始化后，其之后的第一个Activity(careActivity).onWindfocusChanged()执行时，计算冷启动耗时。本质上用户看到了第二个Activity，比firstScreenCost多了闪屏页展现时间+careActivity的初始化耗时。
* applicationCost：Application初始化耗时
* ActivityThreadHacker.getEggBrokenTime()：应用冷启动开始节点
* splashActivities：保存闪屏页列表，在TraceConfig中初始化。demo中是在MatrixApplication.onCraete()手动配置splash Activity。

有了上面这些耗时统计，AnalyseTask利用这些数据，进行堆栈优化、数据整理，打印出前面的启动耗时log。

##### 总结：

* Application初始化耗时(applicationCost)：
  * 统计``Application.onCreate()``到``ActivityThread.mH``接收``LaunchActivityItem``消息这段过程的时间
* 首屏耗时(firstScreenCost)：
  * 指app启动到闪屏页Activity(launch activity)初始化完成的耗时，可以认为包含了applicationCost + launch activity页面的初始化耗时
* 冷启动耗时(coldCost)
  * app启动(``Application.onCreate()``)到第一个对用户有意义的Activity（careActivity）初始化完成(``Activity.onWindowFocusChanged()``)的耗时



## 参考

[Matrix](https://github.com/Tencent/matrix)