---
title: Android 同步屏障机制(Sync Barrier)
categories: 技术
---

Android中的同步屏障机制本质上是让消息队列优先处理异步消息。在View渲染方面就是通过这种机制优先处理界面层相关任务。

<!--more-->

在[Android图形渲染之Choreographer原理](https://hningoba.github.io/2019/11/28/Android图形渲染之Choreographer原理/)这篇文章中提到过，为了优先处理UI渲染工作，Traversal流程中先向MessageQueue插入了同步屏障消息（MessageQueue.postSyncBarrier()），再发送一个异步消息（msg.setAsynchronous(true)）

```
android.view.ViewRootImpl.java

void scheduleTraversals() {
				...
            // MessageQueue中设置同步屏障，优先处理异步消息，即后面会讲到的TraversalRunnable
            mTraversalBarrier = mHandler.getLooper().getQueue().postSyncBarrier();
            // 将TraversalRunnable入队
            mChoreographer.postCallback(
                    Choreographer.CALLBACK_TRAVERSAL, mTraversalRunnable, null);
            ...
    }
```

```
android.view.Choreographer.java

private void postCallbackDelayedInternal(int callbackType,
            Object action, Object token, long delayMillis) {
        
            // 如果不在当前线程，发送一个异步消息
            Message msg = mHandler.obtainMessage(MSG_DO_SCHEDULE_CALLBACK, action);
            msg.arg1 = callbackType;
            // 将消息设置为异步
            msg.setAsynchronous(true);
            mHandler.sendMessageAtTime(msg, dueTime);
    }
```

### 添加同步屏障

通常我们使用Handler发送消息用到的post()、postAtTime()、postDelayed()最终都会走到Handler.enqueueMessage()。这些消息的target都是当前Handler。

```
android.os.Handler.java

private boolean enqueueMessage(MessageQueue queue, Message msg, long uptimeMillis) {
				// target是当前Handler。
				msg.target = this;
        if (mAsynchronous) {
            msg.setAsynchronous(true);
        }
        return queue.enqueueMessage(msg, uptimeMillis);
    }
```

但是，通过MessageQueue.postSyncBarrier()设置同步屏障时，Message的target并没有赋值，即为null。

```
android.os.MessageQueue.java

public int postSyncBarrier() {
        return postSyncBarrier(SystemClock.uptimeMillis());
    }

    private int postSyncBarrier(long when) {
        // Enqueue a new sync barrier token.
        // We don't need to wake the queue because the purpose of a barrier is to stall it.
        synchronized (this) {
            final int token = mNextBarrierToken++;
            final Message msg = Message.obtain();
            msg.markInUse();
            // 屏障消息只设置了when和arg1
            msg.when = when;
            msg.arg1 = token;
						...
        }
    }
```

所以，同步屏障本质上是在消息队列中入队了一个target为空的Message。那么同步屏障机制和同步、异步消息有什么关系呢？

### 异步消息处理

Looper.loop()进行消息轮询时会调用下面方法。

```
android.os.MessageQueue.java

Message next() {
        ...
        for (;;) {
            ...

            synchronized (this) {
                // Try to retrieve the next message.  Return if found.
                final long now = SystemClock.uptimeMillis();
                Message prevMsg = null;
                Message msg = mMessages;
                // 发现了同步屏障消息
                if (msg != null && msg.target == null) {
                    // Stalled by a barrier.  Find the next asynchronous message in the queue.
                    // 如果是同步消息则继续找，直到找到异步消息
                    do {
                        prevMsg = msg;
                        msg = msg.next;
                    } while (msg != null && !msg.isAsynchronous());
                }
                ...
                return msg;
    }
```

简单来说，就是找下一个要执行的消息时，如果发现了同步屏障，则往下找异步消息。所以有同步屏障时，Handler只会处理异步消息。

### 总结

1. 同步屏障本质上是一个target为空的Message；
2. 消息轮询时，如果发现了同步屏障消息，则只处理异步消息，所以Android通过这种方式保证UI渲染任务优先处理。