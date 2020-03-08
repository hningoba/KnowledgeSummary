---
title: Android图形渲染之Choreographer原理
categories: 技术
---

Choreographer可以认为是连接底层和应用层的中间人角色。对下，它负责注册并接收底层发送的Vsync信号；对上，负责在应用层协调下一帧的绘制、事件、动画过程。Choreographer配合SurfaceFlinger、Triple Buffer为Android系统提供稳定的帧率刷新环境。

<!--more-->



### Choreographer初始化

##### Vsync信号简介

​	VSYNC 信号可以认为是一种定时中断信号。为了配合主流屏幕60Hz的刷新频率，即16.6ms一次，系统也控制Vsync信号的周期为16.6ms。Vsync信号在应用层会唤醒Choreographer响应用户输入、APP绘制等操作，在合成层会唤醒SurfaceFlinger，将准备好的Surface进行合成操作。

​	Vsync信号主要由硬件HWC（Hardware Composer，硬件混合渲染器）生成。然后通过回调将事件发送到SurfaceFlinger和Choreographer。

```
typedef void (*HWC2_PFN_VSYNC)(hwc2_callback_data_t callbackData,
        hwc2_display_t display, int64_t timestamp);
```

​	DispSync将硬件Vsync信号转换成Choreographer和SurfaceFlinger使用的VSYNC和SF_VSYNC信号。

![DispSync 流程](https://source.android.google.cn/devices/graphics/images/dispsync.png)



##### Choreographer初始化调用链

通过AS Profiler观察方法调用链，Activity启动后，执行完ActivitityThread.performResumeActivity()，再执行WindowManagerImpl.addView()，在其内部会执行ViewRootImpl的初始化。Choreographer的初始化就是在ViewRootImpl的构造方法中执行的。

![](https://github.com/hningoba/KnowledgeSummary/blob/master/img/Choreographer%E5%88%9D%E5%A7%8B%E5%8C%96%E9%93%BE.png?raw=true)

```
android.view.ViewRootImpl.java

public ViewRootImpl(Context context, Display display) {
	mChoreographer = Choreographer.getInstance();
}
```



##### Choreographer初始化过程

-> 初始化FrameHandler，绑定Looper，响应事件

-> 初始化FrameDisplayEventReceiver，与SurfaceFlinger建立联系，响应、调度Vsync

-> 初始化CallbackQueue

```
android.view.Choreographer.java

// 使用ThreadLocal保证Choreographer线程单例，但一般是主线程
private static final ThreadLocal<Choreographer> sThreadInstance =
            new ThreadLocal<Choreographer>() {
        @Override
        protected Choreographer initialValue() {
        // Choreographer所在线程必须有Looper，因为很多任务调度是通过FrameHandler执行的
            Looper looper = Looper.myLooper();
            if (looper == null) {
                throw new IllegalStateException("The current thread must have a looper!");
            }
            Choreographer choreographer = new Choreographer(looper, VSYNC_SOURCE_APP);
            if (looper == Looper.getMainLooper()) {
                mMainInstance = choreographer;
            }
            return choreographer;
        }
    };

// 初始化
private Choreographer(Looper looper, int vsyncSource) {
    mLooper = looper;
    mHandler = new FrameHandler(looper);
    mDisplayEventReceiver = USE_VSYNC
    	? new FrameDisplayEventReceiver(looper, vsyncSource)
    	: null;
    // 上一帧绘制时间点
    mLastFrameTimeNanos = Long.MIN_VALUE;
    // 帧间隔，16.6ms
    mFrameIntervalNanos = (long)(1000000000 / getRefreshRate());

    // 任务调度的回调都是放到该CallbackQueue中
    mCallbackQueues = new CallbackQueue[CALLBACK_LAST + 1];
    for (int i = 0; i <= CALLBACK_LAST; i++) {
    		mCallbackQueues[i] = new CallbackQueue();
    }
}
```

执行核心方法doFrame()和调度Vsync都是通过Handler实现。

```
private final class FrameHandler extends Handler {
        @Override
        public void handleMessage(Message msg) {
            switch (msg.what) {
                case MSG_DO_FRAME:	// 开始下一帧操作
                    doFrame(System.nanoTime(), 0);
                    break;
                case MSG_DO_SCHEDULE_VSYNC:
                    doScheduleVsync();	// 请求下一个Vsync信号
                    break;
                case MSG_DO_SCHEDULE_CALLBACK:
                    doScheduleCallback(msg.arg1);
                    break;
            }
        }
    }
```



### FrameDisplayEventReceiver

Choreographer对Vsync信号的响应主要体现在FrameDisplayEventReceiver。这个类负责对Vsync信号注册、调度、响应。

FrameDisplayEventReceiver有三个主要方法：

* onVsync()：响应Vsync信号
* scheduleVsync()：调度Vsync信号

* run()：执行doFrame()，doFrame()内部会做计算掉帧、响应Input、Animation、Traversal(measure/layout/draw)、Commit



##### 响应Vsync

HWC触发Vsync信号后，在Java层会回调DisplayEventReceiver.onVsync()，该方法是空实现，本质上走到子类的FrameDisplayEventReceiver.onVsync()方法。

```
private final class FrameDisplayEventReceiver extends DisplayEventReceiver
            implements Runnable {

        @Override
        public void onVsync(long timestampNanos, int builtInDisplayId, int frame) {
            if (builtInDisplayId != SurfaceControl.BUILT_IN_DISPLAY_ID_MAIN) {
                scheduleVsync();
                return;
            }
						
						// Vsync信号到来的时间戳，后面计算掉帧时会用到
            mTimestampNanos = timestampNanos;
            mFrame = frame;
            // Message的Callback为自身，即FrameDisplayEventReceiver
            Message msg = Message.obtain(mHandler, this);
            msg.setAsynchronous(true);
            // FrameHandler
            mHandler.sendMessageAtTime(msg, timestampNanos / TimeUtils.NANOS_PER_MS);
        }
```

​	本质上是通过FrameHandler发送了一个Callback为FrameDisplayEventReceiver自身的message，当Looper执行到该消息时会调用FrameDisplayEventReceiver.run()方法，该Looper一般是主线程的Looper。

​	其中mTimestampNanos记录了Vsync信号到来的时间点，后面计算掉帧时会用到。



##### run()

```
 @Override
 public void run() {
     mHavePendingVsync = false;
     doFrame(mTimestampNanos, mFrame);
 }
```

内部调用Choreographer.doFrame().



### Choreographer处理每一帧

doFrame()是Choreographer处理每一帧操作的核心。

内部先进行掉帧计算，记录帧信息，然后执行INPUT（用户输入）、ANIMATION、TRAVERSAL（measure/layout/draw）、COMMIT等回调。

* INPUT：输入事件
* ANIMATION：动画
* TRAVERSAL：页面刷新，响应measure、layout、draw
* COMMIT：处理commit回调，修正最后一帧的时间戳

##### **掉帧计算**

​	掉帧计算的逻辑在下面这段代码，Vsync信号到来时记录一个开始时间，即onVsync()中的mTimestampNanos。在主线程MessageQueue中轮到doFrame()的message执行后（Vsync信号到来有可能不会立马处理，因为这中间主线程可能被其他任务占用），在doFrame()中记录一个结束时间，即下面代码中的startNanos。这两个时间差值就是Vsync信号的处理时延，即掉帧时间。

​	平时开发中大家有可能在Logcat看到下面这个log："Skipped N frames!  The application may be doing too much work on its main thread." 掉帧超过30帧之后就会出这个提示。

```
/**
* @param frameTimeNanos 底层Vsync信号到达的时间戳
*/
void doFrame(long frameTimeNanos, int frame) {
        final long startNanos;
        synchronized (mLock) {
        	// mFrameScheduled调度Vsync的标记位，在scheduleFrameLocked()置为true，防止不必要的调用
            if (!mFrameScheduled) {
                return; // no work to do
            }

            long intendedFrameTimeNanos = frameTimeNanos;
            // 真正处理Vsync对应的某一帧的时间点
            startNanos = System.nanoTime();
            // 响应Vsync的时间点 - Vsync到来的时间点 = 耽误的时间，即掉帧时间
            final long jitterNanos = startNanos - frameTimeNanos;
            if (jitterNanos >= mFrameIntervalNanos) {
                final long skippedFrames = jitterNanos / mFrameIntervalNanos;
                
                // 掉帧超过30帧，Log提示
                if (skippedFrames >= SKIPPED_FRAME_WARNING_LIMIT) {
                    Log.i(TAG, "Skipped " + skippedFrames + " frames!  "
                            + "The application may be doing too much work on its main thread.");
                }
                final long lastFrameOffset = jitterNanos % mFrameIntervalNanos;
                frameTimeNanos = startNanos - lastFrameOffset;
            }
					....
}
```



回调的逻辑在doCallback()中。

```
/**
* @param frameTimeNanos 底层Vsync信号到达的时间戳
*/
void doFrame(long frameTimeNanos, int frame) {
        ....

        try {
            Trace.traceBegin(Trace.TRACE_TAG_VIEW, "Choreographer#doFrame");
            AnimationUtils.lockAnimationClock(frameTimeNanos / TimeUtils.NANOS_PER_MS);
            
            // 响应输入，执行大家熟悉的View.dispatchOnTouchEvent一系列流程
            mFrameInfo.markInputHandlingStart();
            doCallbacks(Choreographer.CALLBACK_INPUT, frameTimeNanos);
            
            // 响应动画
            mFrameInfo.markAnimationsStart();
            doCallbacks(Choreographer.CALLBACK_ANIMATION, frameTimeNanos);
            
            // 响应绘制流程，即measure、layout、draw
            mFrameInfo.markPerformTraversalsStart();
            doCallbacks(Choreographer.CALLBACK_TRAVERSAL, frameTimeNanos);
            
            // 处理commit回调，修正最后一帧的时间戳
            doCallbacks(Choreographer.CALLBACK_COMMIT, frameTimeNanos);
        } finally {
            AnimationUtils.unlockAnimationClock();
            Trace.traceEnd(Trace.TRACE_TAG_VIEW);
        }
    }
```



##### INPUT调用栈

有用户input交互后，从method trace可以看到，doCallbacks(Choreographer.CALLBACK_INPUT, frameTimeNanos)内部执行的是ConsumeBatchedInputRunnable。最终会走到DecorView.dispatchTouchEvent()，后续就是一系列touch event的事件分发逻辑。

<img src="https://github.com/hningoba/KnowledgeSummary/blob/master/img/Choreographer-Input%E8%B0%83%E7%94%A8%E9%93%BE.png?raw=true" style="zoom: 67%;" />

ConsumeBatchedInputRunnable相关代码：

```
android.view.ViewRootImpl.java

final class ConsumeBatchedInputRunnable implements Runnable {
        @Override
        public void run() {
            doConsumeBatchedInput(mChoreographer.getFrameTimeNanos());
        }
    }
    
    
void doConsumeBatchedInput(long frameTimeNanos) {
        if (mConsumeBatchedInputScheduled) {
            mConsumeBatchedInputScheduled = false;
            if (mInputEventReceiver != null) {
                if (mInputEventReceiver.consumeBatchedInputEvents(frameTimeNanos)
                        && frameTimeNanos != -1) {
                    scheduleConsumeBatchedInput();
                }
            }
            doProcessInputEvents();
        }
    }
```

根据method trace跟一下代码，最后会走到View.dispatchPointerEvent

```
android.view.ViewRootImpl.java

/**
* Delivers post-ime input events to the view hierarchy.
*/
final class ViewPostImeInputStage extends InputStage {
        @Override
        protected int onProcess(QueuedInputEvent q) {
            ...
            processPointerEvent(q);
            ...
        }
        
         private int processPointerEvent(QueuedInputEvent q) {
            ...
            // 后续走到View层的事件分发逻辑
            boolean handled = mView.dispatchPointerEvent(event);
          	...
        }       
}
```



##### ANIMATION调用栈

如果使用ValueAnimator执行动画，可以看到下图中的方法调用栈：

<img src="https://github.com/hningoba/KnowledgeSummary/blob/master/img/Choreographer-Animation%E8%B0%83%E7%94%A8%E9%93%BE.png?raw=true" style="zoom:67%;" />

​	动画执行时，Choreographer.doFrame()会响应AnimationHandler内部的Choreographer.FrameCallback.doFrame()，然后执行到ValueAnimator.onAnimationUpdate()，这个方法也就是业务测实现动画逻辑的地方。

​	执行动画时，Choreographer回调到AnimationHandler$Choreographer.FrameCallback.doFrame()：

```
AnimationHandler.java

 private final Choreographer.FrameCallback mFrameCallback = new Choreographer.FrameCallback() {
        @Override
        public void doFrame(long frameTimeNanos) {
            doAnimationFrame(getProvider().getFrameTime());
            if (mAnimationCallbacks.size() > 0) {
                getProvider().postFrameCallback(this);
            }
        }
    };

```

​	动画完成后，执行Choreographer.scheduleVsyncLocked()调度下一个Vsync（“调度Vsync”部分会再讲到），再次执行下一帧的动画流程。就这样循环，完成整个动画过程。使用View.postOnAnimation()流程也大致相同。可以借助profiler跟下代码。



​	那AnimationHandler中的Choreographer.FrameCallback是怎么注册到Choreographer中的呢？

​	答案是调用ValueAnimator.start()开始动画的时候。start之后会走到ValueAnimator.addAnimationCallback()

```
android.animation.ValueAnimator.java

private void addAnimationCallback(long delay) {
        if (!mSelfPulse) {
            return;
        }
        getAnimationHandler().addAnimationFrameCallback(this, delay);
    }
```

然后执行到AnimationHandler.addAnimationFrameCallback()

```
android.animation.AnimationHandler.java

public void addAnimationFrameCallback(final AnimationFrameCallback callback, long delay) {
        if (mAnimationCallbacks.size() == 0) {
        		// getProvider()就是AnimationHandler中的内部类MyFrameCallbackProvider
            getProvider().postFrameCallback(mFrameCallback);
        }
    }
    
private class MyFrameCallbackProvider implements AnimationFrameCallbackProvider {

        final Choreographer mChoreographer = Choreographer.getInstance();

        @Override
        public void postFrameCallback(Choreographer.FrameCallback callback) {
        		// 向Choreographer中注册FrameCallback，该callback会放到Choreographer的mCallbackQueues，当Choreographer.doFrame()执行doCallback()时就从Queue中找到该FrameCallback进行回调
            mChoreographer.postFrameCallback(callback);
        }
    }
```

其中，getProvider().postFrameCallback()中的mFrameCallback就是Choreographer回调AnimationHandler的FrameCallback。postFrameCallback()会把该callback会放到Choreographer的mCallbackQueues，当Choreographer.doFrame()执行doCallback()时就从Queue中找到该FrameCallback进行回调

```
android.view.Choreographer.java

private void postCallbackDelayedInternal(int callbackType,
            Object action, Object token, long delayMillis) {
      
        synchronized (mLock) {
            ...
            mCallbackQueues[callbackType].addCallbackLocked(dueTime, action, token);
						...
        }
    }
```

其他的动画方式也可以通过查看AS->profiler->CPU来看具体的方法调用栈，理解其实现思路。



##### TRAVERSAL调用栈

traversal过程主要是指View的measure、layout、draw。

像View.invalidate()、View.requestLayout()都会触发traversal流程。跟踪代码，会执行到下面：

```
android.view.ViewRootImpl.java

void scheduleTraversals() {
        if (!mTraversalScheduled) {
            mTraversalScheduled = true;
            // MessageQueue中设置同步屏障，优先处理异步消息，即后面会讲到的TraversalRunnable
            mTraversalBarrier = mHandler.getLooper().getQueue().postSyncBarrier();
            // 将TraversalRunnable入队，
            mChoreographer.postCallback(
                    Choreographer.CALLBACK_TRAVERSAL, mTraversalRunnable, null);
            ...
        }
    }
```

调度Traversal时，为了优先处理UI绘制操作，在消息队列中加入了同步屏障，且Traversal对应的Message设置为异步消息。这块知识点可以参考：[Android 同步屏障机制(Sync Barrier)]([https://hningoba.github.io/2019/12/06/Android%20%E5%90%8C%E6%AD%A5%E5%B1%8F%E9%9A%9C%E6%9C%BA%E5%88%B6(Sync%20Barrier)/#more](https://hningoba.github.io/2019/12/06/Android 同步屏障机制(Sync Barrier)/#more))

mChoreographer.postCallback()会执行到下面方法：

```
android.view.Choreographer.java

private void postCallbackDelayedInternal(int callbackType,
            Object action, Object token, long delayMillis) {
        synchronized (mLock) {
        	...
            mCallbackQueues[callbackType].addCallbackLocked(dueTime, action, token);

            // 如果不在当前线程，发送一个异步消息
            Message msg = mHandler.obtainMessage(MSG_DO_SCHEDULE_CALLBACK, action);
            msg.arg1 = callbackType;
            msg.setAsynchronous(true);
            mHandler.sendMessageAtTime(msg, dueTime);
        }
    }
```

上面得了流程会向底层调度Vsync信号。等下一个Vsync信号到来时，会执行doFrame()。进而在Choreographer.doCallbacks(Choreographer.CALLBACK_TRAVERSAL)中，从CallbackQueue中找到Traversal对应的CallbackRecord执行。

从下图可以看到Choreographer.doCallbacks(Choreographer.CALLBACK_TRAVERSAL)内部执行的是ViewRootImpl.TravasalRunnable。这一点上面也提到了。

<img src="https://github.com/hningoba/KnowledgeSummary/blob/master/img/Choreographer-Traversal%E8%B0%83%E7%94%A8%E9%93%BE.png?raw=true" style="zoom:67%;" />

TravasalRunnable.performTraversals()内部执行了大家比较熟悉的View绘制流程，即measure、layout、draw。

```
android.view.ViewRootImpl.java

 final class TraversalRunnable implements Runnable {
        @Override
        public void run() {
            doTraversal();
        }
    }
    
 void doTraversal() {
        if (mTraversalScheduled) {
            ...
            performTraversals();
						...
        }
	}

	private void performTraversals() {
			if (focusChangedDueToTouchMode || mWidth != host.getMeasuredWidth()
                        || mHeight != host.getMeasuredHeight() || contentInsetsChanged ||
                        updatedConfiguration) {
          	// 执行measure操作
         	 	performMeasure(childWidthMeasureSpec, childHeightMeasureSpec);
      }
      
      if (didLayout) {
      			// 执行layout操作
            performLayout(lp, mWidth, mHeight);
      }
      
      if (!cancelDraw && !newSurface) {
      			// 执行draw操作
			      performDraw();
      }
	}
```



##### 调度Vsync

上面讲Animation调用栈时提到每一帧动画结束后都向底层主动调度下一个Vsync，用来做下一帧的渲染工作。具体调度代码在下面：

```
android.view.Choreographer.java

    private void scheduleFrameLocked(long now) {
        if (!mFrameScheduled) {
            // 调度Vsync标记位
            mFrameScheduled = true;
            
            // 默认使用Vsync
            if (USE_VSYNC) {
                if (DEBUG_FRAMES) {
                    Log.d(TAG, "Scheduling next frame on vsync.");
                }

                // 如果在当前线程，直接调度Vsync，否则通过Handler发消息调度
                if (isRunningOnLooperThreadLocked()) {
                    scheduleVsyncLocked();
                } else {
                    Message msg = mHandler.obtainMessage(MSG_DO_SCHEDULE_VSYNC);
                    msg.setAsynchronous(true);
                    mHandler.sendMessageAtFrontOfQueue(msg);
                }
            } else {
                ...
            }
        }
    }
    
    
    private void scheduleVsyncLocked() {
        mDisplayEventReceiver.scheduleVsync();
    }
```

最后通过DisplayEventReceiver向JNI申请Vsync信号。

```
android.view.DisplayEventReceiver.java

public void scheduleVsync() {
		...
		nativeScheduleVsync(mReceiverPtr);
}
```

JNI层调度Vsync方法：

```
android_view_DisplayEventReceiver.cpp

static void nativeScheduleVsync(JNIEnv* env, jclass clazz, jlong receiverPtr) {
    sp<NativeDisplayEventReceiver> receiver =
            reinterpret_cast<NativeDisplayEventReceiver*>(receiverPtr);
    // 调用NativeDisplayEventDispatcher.scheduleVsync()
    status_t status = receiver->scheduleVsync();
    ...
}
```

NativeDisplayEventDispatcher继承自DisplayEventDispatcher，scheduleVsync()本质上是调用DisplayEventDispatcher：

```
DisplayEventDispatcher.cpp

status_t DisplayEventDispatcher::scheduleVsync() {
    if (!mWaitingForVsync) {
        ...
        // 调用DisplayEventReceiver.requestNextVsync()，请求SurfaceFlinger发送Vsync
        status_t status = mReceiver.requestNextVsync();
        if (status) {
            ALOGW("Failed to request next vsync, status=%d", status);
            return status;
        }

        mWaitingForVsync = true;
    }
    return OK;
}
```



当下一个Vsync信号到来时，JNI会回调DisplayEventReceiver.dispatchVsync()，然后执行FrameDisplayEventReceiver.onVsync()，这部分内容上面已经讲到。

```
android.view.DisplayEventReceiver.java

    // Called from native code.
    @SuppressWarnings("unused")
    private void dispatchVsync(long timestampNanos, int builtInDisplayId, int frame) {
        onVsync(timestampNanos, builtInDisplayId, frame);
    }

```





### 帧率计算

##### 方式一：FrameCallback

有些APM计算帧率的方式和处理动画的逻辑类似，利用FrameCallback的回调，在doFrame()中计算帧率。

```
// 注册自定义FrameCallback
Choreographer.getInstance().postFrameCallback(APM.mFrameCallback);

// 在回调中计算帧率
val APM.mFrameCallback = Choreographer.FrameCallback() {
		fun doFrame(long frameTimeNanos) {
			frameCounter ++
		}
}
```

开始和结束监听帧率时记录start和end时间戳，结束时通过FrameCallback中的帧数就可以计算帧率。

##### 方式二：Looper & Printer

也有工具使用消息机制Looper类中监听日志打印的逻辑来计算主线程卡顿问题，比如BlockCanary。利用该方法不仅可以计算每一帧的耗时情况，也可以计算一定时间内的帧数，从而计算帧率。

看下关键方法：

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

主线程每一帧执行前后，Looper通过一个Printer对象分别打印日志">>>>> Dispatching to..."和"<<<<< Finished to"。通过Looper.setMessageLogging()可以设置我们自定义的Printer。下面示例代码可以通过监听Looper的日志实现：

```
public class BlockPrinter implements Printer {

    private static final String BLOCK_START = ">>>>> Dispatching";
    private static final String BLOCK_END = "<<<<< Finished";
    private int frameCounter;
		
    @Override
    public void println(String x) {
        if (x.startsWith(BLOCK_START)) {
            long startTime;
        }
        if (x.startsWith(BLOCK_END)) {
            long endTime;
        }
        if (endTime - startTime > timeOut) {
            // 每一帧耗时
        }
        frameCounter ++
    }
}
```

通过这种方式可以计算每一帧的耗时和帧数。再像方式一一样，开始和结束时记录start和end时间戳，结束时通过预先记录好的帧数就可以计算帧率。



### 总结

1. Choreographer收到显示子系统发送的定时脉冲（Vsync）后，协调下一帧的animations、input、drawing工作。

2. 收到Vsync后会响应Choreographer.doFrame()，每一帧的INPUT、ANIMATION、TRAVERSAL都是从这个方法开始执行的。

3. Choreographer是线程私有的，每个Looper有自己的Choreographer实例，跨线程调度任务通过post callback方式。

4. 计算帧率可以借鉴动画的实现思路，自定义FrameCallback。
5. App没有更新操作，比如没有需要执行的动画、用户没有交互等，app层是不会渲染的。因为每一帧的操作都需要先向SurfaceFlinger调度Vsync。比如View.invalidate()、View.requestLayout()、View.postOnAnimation()、ValueAnimator.start()等都会触发Vsync调度。



### 参考

[Choreographer](https://developer.android.com/reference/android/view/Choreographer)

[Vsync](https://source.android.google.cn/devices/graphics/implement-vsync)

[Android 基于 Choreographer 的渲染机制详解](https://www.androidperformance.com/2019/10/22/Android-Choreographer/#)

[Choreographer原理](http://gityuan.com/2017/02/25/choreographer/)