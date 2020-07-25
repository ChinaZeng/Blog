**IdleHandler**

如果你有时候想要做一些操作但是又不想影响ui渲染，那么`IdleHandler`就很符合你的需求，`IdleHandler`在`MessageQueue`空时触发回调，可以用来做一些不是太急需的初始化，比如webview预加载等。  
使用方式：
```kotlin
Looper.myQueue().addIdleHandler {
    //webview预加载等
   false
}
```

 触发时机源码：

 
 ```java
 
 // MessageQueue.java
 Message next() {
   //...
    int pendingIdleHandlerCount = -1; // -1 only during first iteration
    int nextPollTimeoutMillis = 0;
    for (;;) {
       
        nativePollOnce(ptr, nextPollTimeoutMillis);

        synchronized (this) {
            // Try to retrieve the next message.  Return if found.
            final long now = SystemClock.uptimeMillis();
            Message prevMsg = null;
            Message msg = mMessages;
            if (msg != null && msg.target == null) {
                // Stalled by a barrier.  Find the next asynchronous message in the queue.
                do {
                    prevMsg = msg;
                    msg = msg.next;
                } while (msg != null && !msg.isAsynchronous());
            }
            if (msg != null) {
                if (now < msg.when) {
                    // Next message is not ready.  Set a timeout to wake up when it is ready.
                    nextPollTimeoutMillis = (int) Math.min(msg.when - now, Integer.MAX_VALUE);
                } else {
                    // Got a message.
                    mBlocked = false;
                    if (prevMsg != null) {
                        prevMsg.next = msg.next;
                    } else {
                        mMessages = msg.next;
                    }
                    msg.next = null;
                    if (DEBUG) Log.v(TAG, "Returning message: " + msg);
                    msg.markInUse();
                    return msg;
                }
            } else {
                // No more messages.
                nextPollTimeoutMillis = -1;
            }

            // Process the quit message now that all pending messages have been handled.
            if (mQuitting) {
                dispose();
                return null;
            }




            // 当消息队列没有message时调用到这里
            // If first time idle, then get the number of idlers to run.
            // Idle handles only run if the queue is empty or if the first message
            // in the queue (possibly a barrier) is due to be handled in the future.
            if (pendingIdleHandlerCount < 0
                    && (mMessages == null || now < mMessages.when)) {
                pendingIdleHandlerCount = mIdleHandlers.size();
            }
            if (pendingIdleHandlerCount <= 0) {
                // No idle handlers to run.  Loop and wait some more.
                mBlocked = true;
                continue;
            }

            if (mPendingIdleHandlers == null) {
                mPendingIdleHandlers = new IdleHandler[Math.max(pendingIdleHandlerCount, 4)];
            }
            mPendingIdleHandlers = mIdleHandlers.toArray(mPendingIdleHandlers);
        }


       
        // Run the idle handlers.
        // We only ever reach this code block during the first iteration.
        for (int i = 0; i < pendingIdleHandlerCount; i++) {
            final IdleHandler idler = mPendingIdleHandlers[i];
            mPendingIdleHandlers[i] = null; // release the reference to the handler

            boolean keep = false;
            try {
                // 触发回掉
                keep = idler.queueIdle();
            } catch (Throwable t) {
                Log.wtf(TAG, "IdleHandler threw exception", t);
            }

            if (!keep) {
                synchronized (this) {
                    mIdleHandlers.remove(idler);
                }
            }
        }

        // Reset the idle handler count to 0 so we do not run them again.
        pendingIdleHandlerCount = 0;

        // While calling an idle handler, a new message could have been delivered
        // so go back and look again for a pending message without waiting.
        nextPollTimeoutMillis = 0;
    }
}

 ```
 
 
 
**同步屏障**
 
在`handler`使用过程中有时候需要`post`的消息优先执行，那么同步屏障就发生作用了。在实际运用中基本没有，但是我觉得还是需要掌握它，这些对应的函数也是`@hide`的。部分源码会有使用，比如源码ViewRootImpl里面
```java

 //ViewRootImpl.java
 void scheduleTraversals() {
    if (!mTraversalScheduled) {
        mTraversalScheduled = true;
        //这里使用了同步屏障，后续消息队列优先处理异步消息，待移除同步屏障才会执行同步消息
        mTraversalBarrier = mHandler.getLooper().getQueue().postSyncBarrier();
        //发送异步消息
        mChoreographer.postCallback(
                Choreographer.CALLBACK_TRAVERSAL, mTraversalRunnable, null);
        if (!mUnbufferedInputDispatch) {
            scheduleConsumeBatchedInput();
        }
        notifyRendererOfFramePending();
        pokeDrawLockIfNeeded();
    }
}

 void unscheduleTraversals() {
    if (mTraversalScheduled) {
        mTraversalScheduled = false;
        //接触同步屏障
        mHandler.getLooper().getQueue().removeSyncBarrier(mTraversalBarrier);
        mChoreographer.removeCallbacks(
                Choreographer.CALLBACK_TRAVERSAL, mTraversalRunnable, null);
    }
}


//Choreographer.java

public void postCallbackDelayed(int callbackType,
        Runnable action, Object token, long delayMillis) {
    //...
    postCallbackDelayedInternal(callbackType, action, token, delayMillis);
}

private void postCallbackDelayedInternal(int callbackType,
        Object action, Object token, long delayMillis) {
   //..
    Message msg = mHandler.obtainMessage(MSG_DO_SCHEDULE_CALLBACK, action);
    msg.arg1 = callbackType;
    //标记该消息是异步消息
    msg.setAsynchronous(true);
    mHandler.sendMessageAtTime(msg, dueTime);
}


```

同步屏障是怎么实现的呢？在`MessageQueue`调用`postSyncBarrier`函数的时候，会插入一个`targer==null`的`Message`到`MessageQueue`头部，接下来插入需要执行的异步消息到`MessageQueue`，执行`epoll`唤醒，在执行`next()`函数的时候时候就会
判断消息队列头部的`message`的`target==null`,就会优先处理同步消息，直到调用`MessageQueue.removeSyncBarrier`同步屏障移除才会处理同步消息。
```java
 public int postSyncBarrier() {
        return postSyncBarrier(SystemClock.uptimeMillis());
    }

private int postSyncBarrier(long when) {
// Enqueue a new sync barrier token.
// We don't need to wake the queue because the purpose of a barrier is to stall it.
synchronized (this) {
    final int token = mNextBarrierToken++;
    
    //在这里插入了一个message，这个message的target==null
    final Message msg = Message.obtain();
    msg.markInUse();
    msg.when = when;
    msg.arg1 = token;
    
    Message prev = null;
    Message p = mMessages;
    if (when != 0) {
        while (p != null && p.when <= when) {
            prev = p;
            p = p.next;
        }
    }
    if (prev != null) { // invariant: p == prev.next
        msg.next = p;
        prev.next = msg;
    } else {
        msg.next = p;
        mMessages = msg;
    }
    return token;
}
}

@UnsupportedAppUsage
Message next() {
    // Return here if the message loop has already quit and been disposed.
    // This can happen if the application tries to restart a looper after quit
    // which is not supported.
    final long ptr = mPtr;
    if (ptr == 0) {
        return null;
    }

    int pendingIdleHandlerCount = -1; // -1 only during first iteration
    int nextPollTimeoutMillis = 0;
    for (;;) {
        if (nextPollTimeoutMillis != 0) {
            Binder.flushPendingCommands();
        }

        nativePollOnce(ptr, nextPollTimeoutMillis);

        synchronized (this) {
            // Try to retrieve the next message.  Return if found.
            final long now = SystemClock.uptimeMillis();
            Message prevMsg = null;
            Message msg = mMessages;
            //msg.target == null 判断为null拿异步消息
            if (msg != null && msg.target == null) {
                // Stalled by a barrier.  Find the next asynchronous message in the queue.
                do {
                    prevMsg = msg;
                    //while循环拿到异步消息
                    msg = msg.next;
                } while (msg != null && !msg.isAsynchronous());
            }
             if (msg != null) {
                if (now < msg.when) {
                    // Next message is not ready.  Set a timeout to wake up when it is ready.
                    nextPollTimeoutMillis = (int) Math.min(msg.when - now, Integer.MAX_VALUE);
                } else {
                    // Got a message.
                    mBlocked = false;
                    if (prevMsg != null) {
                        prevMsg.next = msg.next;
                    } else {
                        mMessages = msg.next;
                    }
                    msg.next = null;
                    if (DEBUG) Log.v(TAG, "Returning message: " + msg);
                    msg.markInUse();
                    return msg;
                }
            } else {
                // No more messages.
                nextPollTimeoutMillis = -1;
            }
         //...
}


```

参考理解文章:  
https://juejin.im/post/5d4e6af7e51d4561ba48fdb0


**卡顿函数检测**

在`Looper.loop()`执行之后不断的从`MessageQueue`获取消息不断处理，一个消息对应一个事件，在这个事件处理前后都会通过`logging`进行打印，由于`android`的`handler`机制,我们可以通过这两个函数检测出这个消息事件的执行时间，这对于卡顿优化来说很有用。具体的检测实现方式本文不铺开说了，可以看看经典的卡顿检测工具[blockcanary](https://github.com/markzhai/AndroidPerformanceMonitor)源码。

对应源码如下:

```java

/**
* Run the message queue in this thread. Be sure to call
* {@link #quit()} to end the loop.
*/
public static void loop() {
final Looper me = myLooper();
if (me == null) {
    throw new RuntimeException("No Looper; Looper.prepare() wasn't called on this thread.");
}
final MessageQueue queue = me.mQueue;
//...

for (;;) {
    Message msg = queue.next(); // might block
    //...

    // This must be in a local variable, in case a UI event sets the logger
    final Printer logging = me.mLogging;
    //在处理dispatchMessage之前回掉
    if (logging != null) {
        logging.println(">>>>> Dispatching to " + msg.target + " " +
                msg.callback + ": " + msg.what);
    }
    //....
        msg.target.dispatchMessage(msg);
   //...
//在处理dispatchMessage之后回掉
    if (logging != null) {
        logging.println("<<<<< Finished to " + msg.target + " " + msg.callback);
    }
    //...
   
}
```


使用方式:


```java

Looper.getMainLooper().setMessageLogging(object : Printer {
    //这里可能有的rom修改了源码，优化可以根据计数，开始和结束都是成对出现的。
    private val START = ">>>>> Dispatching"
    private val END = "<<<<< Finished"
    override fun println(x: String) {
        if (x.startsWith(START)) {
            //调用前
        }
        if (x.startsWith(END)) {
           //调用后
        }
    }
})

```

卡顿函数检测相关参考文档:

https://github.com/markzhai/AndroidPerformanceMonitor  
https://cloud.tencent.com/developer/article/1156121

