---
title: Handler机制分析
date: 2017-02-01 23:16:33
tags: [handler,消息机制]
categories: android
---

### Handler机制

#### 小的知识点

>* MessageQueue的数据结构-->单向链表，注意不要被prevMsg迷惑了（并不是双向链表）。
>* MessageQueue中的msg是按照优先级依次排列的，队头是最近要发生的msg。
>* Handler的post(Runnable r，...)方法，看下代码便知道，post的参数r会赋值给callback，后期发消息（发消息的机制和上述的完全一致）时会检测这个callback是否为空，不为空的话就交给callback处理（其实它就是个回调方法而已），为空的话，则交给handleMessage处理（常用的就是重写这个方法，然后处理事件）。

#### Handler中消息的异步和同步

>同步或异步是基于Handler来说的：定义一个同步消息的handler、或一个异步消息的handler。

```java
public Handler(Callback callback, boolean async) {
	//Handler构造函数，最终会走到这里
	//第一个参数callback就不解释了
	//第二个参数async作用就是定义此handler是否发异步msg
}
```

#### Handler基本流程

##### 对于主线程的第一个Handler来说

1. **构造主线程的Looper对象**在ActivityThread中prepareMainLooper，此时会先调用prepare方法去构造（new）一个Looper，然后赋值给sMainLooper。
>每个线程对应一个Looper。
2. **构造消息队列**在main线程new Looper时，在Looper的构造方法里就初始化了一个消息队列（mPtr是指向消息队列的一个指针（位置））。
3. **Looper循环取消息**在ActivityThread的main方法结尾调用了Looper.loop()--->走向一个死循环，循环queue里的消息，没有消息的话，则阻塞。
4. **往MessageQueue中放消息**queue中的消息就是handler.send的，handler的sendXXX方法最终会调用enqueueMessage（入队）方法（在入队前，设置了msg.target=this的操作，指明了这个接收这个消息的目标handler）。
5. 调用msg.target.dispatchMessage(msg)方法分发消息。

##### 对于子线程来说

* 子线程不会直接去创建Handler，需要手动进行prepare和loop()，如果不prepare的话，会报错-->因为在Handler的构造方法里会去取Looper对象。

#### 结合源码分析主线程Handler的创建流程

##### ActivityThread的main方法

```java
//初始化MainLooper
Looper.prepareMainLooper();
//循环消息队列【TAG1->一直死循环，为什么不会ANR？】
Looper.loop();
throw new RuntimeException("Main thread loop unexpectedly exited");
```

##### 初始化prepareMainLooper
```java
//mainLooper所做工作
public static void prepareMainLooper() {
	prepare(false);
	sMainLooper = myLooper();//sThreadLocal.get()
}
//prepare所做工作（参数quitAllowed表示是否允许退出）
private static void prepare(boolean quitAllowed) {
	sThreadLocal.set(new Looper(quitAllowed));
}

//new Looper所做工作
private Looper(boolean quitAllowed) {
	//初始化消息队列
	//【注意】参数quitAllowed最终传到了这里
	mQueue = new MessageQueue(quitAllowed);
	mThread = Thread.currentThread();
}
```

##### Looper.loop()方法（主要的操作）
```java
for(;;){
	//【TAG2->为什么可能会阻塞？阻塞之后有什么影响？】
	//当队列中没有消息时，再循环取msg就没有意义了，
	Message msg = queue.next();// might block(说明是个可能阻塞的线程)
	if(msg == null){
		return;
	}
	msg.target.dispatchMessage(msg);//指定的Handler发指定的消息
	msg.recycleUnchecked();
}
```

##### Handler发送消息

```java
handler.sendMessage(message);
....
//最终会把message放到消息队列中
msg.target = this;//指明这个msg的target是谁（即哪个handler来处理这个消息）
//如果此Handler是发异步消息的，则会把每个入队的msg都设置setAsynchronous(true)
if (mAsynchronous) {
	msg.setAsynchronous(true);
}
//入队操作
return queue.enqueueMessage(msg, uptimeMillis);
```

##### MessageQueue的enqueueMessage入队方法

```java
boolean enqueueMessage(Message msg, long when) {
	synchronized (this) {
		msg.markInUse();
		msg.when = when;
		Message p = mMessages;
		boolean needWake;
		//这个when就是根据handler.sendMessageAtTime传入的时
		//间和当前时间（System.current..）计算出来的
		if (p == null || when == 0 || when < p.when) {
			//队列为空 || 不延时 || 当前的延时消息先于队尾的延时消息发生 的情况
			//也就是说如果插入的msg比队头的msg要早发生，这个插入的msg要作为新的队头（先进后出）
			//【也就是说：唤醒队列取消息的时间要提前了】
			// New head, wake up the event queue if blocked.
			msg.next = p;
			mMessages = msg;
			needWake = mBlocked;
		} else {
			//比较出队的优先级，把msg插入到合适的位置
			// Inserted within the middle of the queue.  Usually we don't have to wake
			// up the event queue unless there is a barrier at the head of the queue
			// and the message is the earliest asynchronous message in the queue.
			/**
			* 上述的翻译：不需要唤醒队列，
			* 除非在队列头部有个屏障 并且 当前的消息msg是异步消息---needWake=true
			* */
			needWake = mBlocked && p.target == null && msg.isAsynchronous();
			Message prev;
			for (;;) {
				//下面的操作就是在队列中合适（优先级）的位置再
				//插入一个newMsg，并修改newMsg的指向
				prev = p;
				p = p.next;
				if (p == null || when < p.when) {
					break;
				}
				//【不理解】
				//如果需要唤醒 并且 p是异步消息--->就不需要唤醒了，what？什么意思
				if (needWake && p.isAsynchronous()) {
					needWake = false;
				}
			}
			msg.next = p; // invariant: p == prev.next
			prev.next = msg;
		}

		// We can assume mPtr != 0 because mQuitting is false.
		if (needWake) {
			nativeWake(mPtr);
		}
	}
	return true;
}
```

----
* 上边已经把消息放到MessageQueue里了
* Handler线程通信有一点不要忽略：实现通信的前提是在A线程中调用B线程的Handler进行发送消息！[参考](https://www.cnblogs.com/smyhvae/p/4003922.html) 然后在A中发的消息被放进了B的消息队列中。B接收消息后再处理。

----
##### MessageQueue的next()方法
```java
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
        //如果没有消息机会阻塞在这里（nextPollTimeoutMillis:阻塞的时间，时间过后会被唤醒）
        nativePollOnce(ptr, nextPollTimeoutMillis);
        //走到这里说明有消息---或者说已经被唤醒---
        synchronized (this) {
            // Try to retrieve取回 the next message.  Return if found.
            final long now = SystemClock.uptimeMillis();
            Message prevMsg = null;
            Message msg = mMessages;
            //1. 满足msg != null && msg.target == null的消息就是同步屏障消息
            if (msg != null && msg.target == null) {
                // Stalled by a barrier.  Find the next asynchronous message in the queue.
                //2. 一直循环直到找到一个消息：msg!=null并且是异步消息，跳出循环
                //3. 具体为什么在碰到同步屏障时找异步消息，可以看下下边关于barrier的中文注释
                do {
                    prevMsg = msg;
                    msg = msg.next;
                } while (msg != null && !msg.isAsynchronous());
            }
            //取到消息后--->等待执行或执行
            if (msg != null) {
                if (now < msg.when) {
                    // Next message is not ready.  
                    // Set a timeout to wake up when it is ready.
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
                    //返回要执行的msg
                    return msg;
                }
            } else {
                // No more messages.
                //消息队列没消息了，此时就应该阻塞了。这个阻塞应该是释放资源的吧。
                nextPollTimeoutMillis = -1;
            }
            //【 重要 】走到这里说明：消息延时执行 或 没有消息了 
            // Process the quit message now that all pending messages have been handled.
            if (mQuitting) {
                dispose();
                return null;
            }
            //既然暂时没有消息要执行了，那么可以做一些其它的操作：IdleHandler
            // If first time idle空闲, then get the number of idlers to run.
            //这个注释很清楚的解释了IdleHandler什么时候使用
            // Idle handles only run if the queue is empty or if the first message
            // in the queue (possibly a barrier) is due to be handled in the future.
            if (pendingIdleHandlerCount < 0
                    && (mMessages == null || now < mMessages.when)) {
                pendingIdleHandlerCount = mIdleHandlers.size();
            }
            if (pendingIdleHandlerCount <= 0) {
                // No idle handlers to run.  Loop and wait some more.
                // 阻塞
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
	            //1. 执行每个idleHandler中重写的queueIdle方法
	            //2. 关于queueIdle返回值下边会说
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

##### 延时消息的处理
**如果从队列中取消息时，消息还没有准备好怎么办（比如延时10分钟才执行）**
```java
if (now < msg.when) {//消息还没准备好
	// Next message is not ready.  Set
	// a timeout to wake up when it is ready.
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
```
##### Looper退出循环
```java
//Looper中的quit方法
public void quit() {
    mQueue.quit(false);
}

//Looper的quitSafely方法
public void quitSafely() {
    mQueue.quit(true);
}

//发现最终都走到了MessageQueue的quit方法
void quit(boolean safe) {
	//对于主线程来说，默认就是false！不可退出
    if (!mQuitAllowed) {
        throw new IllegalStateException("Main thread not allowed to quit.");
    }
    synchronized (this) {
        if (mQuitting) {
            return;
        }
        mQuitting = true;
        if (safe) {
	        //安全退出：会移除在当前时间点之后执行的message
	        //清除msg.when>nowTime的msg。因为有可能一个消息正在执行还没执行完
            removeAllFutureMessagesLocked();
        } else {
            removeAllMessagesLocked();
        }
        // We can assume mPtr != 0 because mQuitting was previously false.
        nativeWake(mPtr);
    }
}
```

#### nativeWake方法
先不写了

#### 关于同步屏障
**不清楚，只好先帖注释了**
```java
/*
向Looper的消息队列发布同步障碍。

1. 消息处理照常进行，直到消息队列遇到已发布的同步屏障。 遇到屏障时，队列中的后续同步消息将被暂停（阻止执行），直到通过调用{@link #removeSyncBarrier}并指定标识同步屏障的令牌来释放屏障。
2. 此方法用于立即推迟执行所有后续发布的同步消息，直到满足释放屏障的条件为止。异步消息（请参阅{@link Message＃isAsynchronous}免于屏障并继续照常处理。
3. 必须始终使用相同的令牌调用{@link #removeSyncBarrier}来匹配此调用，以确保消息队列恢复正常操作。否则应用程序可能会挂起！
*/
public int postSyncBarrier() {
    return postSyncBarrier(SystemClock.uptimeMillis());
}
```

#### IdleHandler接口
**注释很清楚**
```java
/**
 * Callback interface for discovering when a thread is going to block
 * waiting for more messages.
 */
public static interface IdleHandler {
    /**
     * Called when the message queue has run out of messages and will now
     * wait for more.  Return true to keep your idle handler active, false
     * to have it removed.  This may be called if there are still messages
     * pending in the queue, but they are all scheduled to be dispatched
     * after the current time.
     */
    //返回false：会在使用过后这个Idlehandler后把它移除，相当于是一次性的idle吧，true不移除
    boolean queueIdle();
}
```


### Message的构建
**传输消息，自然要先构造一个msg**
```java
//方式1
//这种方式获得的 message 已经有一个目标handler ---> this
//不需要再 setTarget(handler)
Message message = handler.obtainMessage();
message.what = 1001;
handler.sendMessage(message);
//方式2
//这种方式获得的 message 没有目标handler
//需要再 setTarget(handler)，可以设置成另一个handler吗？
Message obtainMsg = Message.obtain();
obtainMsg.setTarget(handler);
obtainMsg.what = 2002;
//但是这个方法执行下去会发现仍然会设置msg.target = this--> 哪个handler调send就是哪个handler，所以上边不需要手动setTarget，即使设置成了别的handler也没用
handler.sendMessage(message);
```

#### 一些感觉还不错的知识点
>* [MessageQueue](https://www.sohu.com/a/145311556_675634)
>* [为什么主线程不会因为Looper.loop()里的死循环卡死](https://www.zhihu.com/question/34652589)