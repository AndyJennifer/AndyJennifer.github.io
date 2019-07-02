---
layout: “android
title: Handler机制之Message的发送与取出”
date: 2019-02-23 22:07:13
categories:
- Handler相关
tags: 
- 异步任务
---


>该文章属于Android Handler系列文章，如果想了解更多，请点击{% post_link Android-Handler机制之总目录 %}

### 前言
在前面的文章中，我们已经大概了解了ThreadLocal的内部原理，以及Handler发消息的大概流程。如果小伙伴如果对Handler机制不熟，建议阅读{% post_link Android-Handler机制之ThreadLocal %}与{% post_link Android-Handler机制之Handler-、MessageQueue-、Looper %}。该篇文章主要着重讲解Message的发送与取出的具体逻辑细节。在此之前，我们先回顾一下Handler发送消息的具体流程。

{% asset_img HandlerLooperMessage关系.png HandlerLooperMessage关系 %}
### 消息的发送
我们都知道当调用Handler发送消息的时候，不管是调用sendMessage,sendEmptyMessage,sendMessageDelayed还是其他发送一系列方法。最终都会调用**sendMessageDelayed(Message msg, long delayMillis)**方法。

```
  public final boolean sendMessageDelayed(Message msg, long delayMillis)
    {
        if (delayMillis < 0) {
            delayMillis = 0;
        }
        return sendMessageAtTime(msg, SystemClock.uptimeMillis() + delayMillis);
    }
```
该方法会调用sendMessageAtTime()方法。其中第二个参数是执行消息的时间，是通过从开机到现在的毫秒数（手机睡眠的时间不包括在内）+ 延迟执行的时间。这里不使用System.currentTimeMillis() ，是因为该时间是可以修改的。会导致延迟消息等待时间不准确。该方法内部会调用sendMessageAtTime()方法，我们接着往下走。

```
   public boolean sendMessageAtTime(Message msg, long uptimeMillis) {
        MessageQueue queue = mQueue;
        if (queue == null) {
            RuntimeException e = new RuntimeException(
                    this + " sendMessageAtTime() called with no mQueue");
            Log.w("Looper", e.getMessage(), e);
            return false;
        }
        return enqueueMessage(queue, msg, uptimeMillis);
    }
```

这里获取到线程中的MessageQueue对象mQueue（在构造函数通过Looper获得的），并调用enqueueMessage()方法，enqueueMessage()方法最终内部会调用MessageQueue的enqueueMessage()方法,那现在我们就直接看MessageQueue中把消息加入消息队列中的方法。

### 消息的加入
当通过handler调用一系列方法如sendMessage()、sendMessageDelayed()方法时，最终会调用MessageQueue的enqueueMessage()方法。现在我们就来看看，消息是怎么加入MessageQueue(消息队列)中去的。

```
boolean enqueueMessage(Message msg, long when) {
        if (msg.target == null) {
            throw new IllegalArgumentException("Message must have a target.");
        }
        if (msg.isInUse()) {
            throw new IllegalStateException(msg + " This message is already in use.");
        }

        synchronized (this) {
	        //如果当前消息循环已经结束，直接退出
            if (mQuitting) {
                IllegalStateException e = new IllegalStateException(
                        msg.target + " sending message to a Handler on a dead thread");
                Log.w(TAG, e.getMessage(), e);
                msg.recycle();
                return false;
            }

            msg.markInUse();
            msg.when = when;
            Message p = mMessages;//头部消息
            boolean needWake;
            //如果队列中没有消息，或者当前进入的消息比消息队列中的消息等待时间短，那么就放在消息队列的头部
            if (p == null || when == 0 || when < p.when) {
                // New head, wake up the event queue if blocked.
                msg.next = p;
                mMessages = msg;
                needWake = mBlocked;
            } else {
	            //判断唤醒条件，当前当前消息队列头部消息是屏障消息，且当前插入的消息为异步消息
	            //且当前消息队列处于无消息可处理的状态
                needWake = mBlocked && p.target == null && msg.isAsynchronous();
                Message prev;
                //循环遍历消息队列，把当前进入的消息放入合适的位置（比较等待时间）
                for (;;) {
                    prev = p;
                    p = p.next;
                    if (p == null || when < p.when) {
                        break;
                    }
                    if (needWake && p.isAsynchronous()) {
                        needWake = false;
                    }
                }
                //将消息插入合适的位置
                msg.next = p; 
                prev.next = msg;
            }

			//调用nativeWake，以触发nativePollOnce函数结束等待
            if (needWake) {
                nativeWake(mPtr);
            }
        }
        return true;
    }
```
**这里大家肯定注意到了nativeWake()方法，这里先不对该方法进行详细的描述，下文会对其解释及介绍。**
其实通过代码大家就应该发现了，在将消息加入到消息队列中时，已经将消息按照等待时间进行了排序。排序分为两种情况（**这里图中的message.when是与当前的系统的时间差**）：

 * 第一种：如果队列中没有消息，或者当前进入的消息比消息队列中头部的消息等待时间短，那么就放在消息队列的头部
 
 ![第一种情况.png](https://upload-images.jianshu.io/upload_images/2824145-f6048cc7d22e098c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


 * 第二种：反之，循环遍历消息队列，把当前进入的消息放入合适的位置（比较等待时间）
{% asset_img 第二种情况.png 第二种情况 %}

综上，我们了解了在我们使用Handler发送消息时，当消息进入到MessageQueue(消息队列)中时，已经按照等待时间进行了排序，且其头部对应的消息是Loop即将取出的消息。

#### 获取消息
我们都知道消息的取出，是通过Loooper.loop()方法，其中loop()方法内部会调用MessageQueue中的next()方法。那下面我们就直接来看next()方法。

```
 Message next() {
	    
	    //如果退出消息消息循环，那么就直接退出
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
			
			//执行native层消息机制层,
			//timeOutMillis参数为超时等待时间。如果为-1，则表示无限等待，直到有事件发生为止。
			//如果值为0，则无需等待立即返回。该方法可能会阻塞
            nativePollOnce(ptr, nextPollTimeoutMillis);

            synchronized (this) {
                //获取系统开机到现在的时间，如果使用System.currentMillis()会有误差，
                final long now = SystemClock.uptimeMillis();
                Message prevMsg = null;
                Message msg = mMessages;//头部消息
	            
                //判断是否是栅栏，同时获取消息队列最近的异步消息
                if (msg != null && msg.target == null) {
                    do {
                        prevMsg = msg;
                        msg = msg.next;
                    } while (msg != null && !msg.isAsynchronous());
                }
                //获取消息，判断等待时间，如果还需要等待则等待相应时间后唤醒
                if (msg != null) {
                    if (now < msg.when) {//判断当前消息时间，是不是比当前时间大，计算时间差
                        // Next message is not ready.  Set a timeout to wake up when it is ready.
                        nextPollTimeoutMillis = (int) Math.min(msg.when - now, Integer.MAX_VALUE);
                    } else {
                        // 不需要等待时间或者等待时间已经到了，那么直接返回该消息
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
                    //没有更多的消息了
                    nextPollTimeoutMillis = -1;
                }

                // Process the quit message now that all pending messages have been handled.
                //判断是否已经退出了
                if (mQuitting) {
                    dispose();
                    return null;
                }

                // If first time idle, then get the number of idlers to run.
                // Idle handles only run if the queue is empty or if the first message
                // in the queue (possibly a barrier) is due to be handled in the future.
                //获取空闲时处理任务的handler 用于发现线程何时阻塞等待更多消息的回调接口。
                if (pendingIdleHandlerCount < 0
                        && (mMessages == null || now < mMessages.when)) {
                    pendingIdleHandlerCount = mIdleHandlers.size();
                }
                //如果空闲时处理任务的handler个数为0，继续让线程阻塞
                if (pendingIdleHandlerCount <= 0) {
                    // No idle handlers to run.  Loop and wait some more.
                    mBlocked = true;
                    continue;
                }
				//判断当前空闲时处理任务的handler是否是为空
                if (mPendingIdleHandlers == null) {
                    mPendingIdleHandlers = new IdleHandler[Math.max(pendingIdleHandlerCount, 4)];
                }
                mPendingIdleHandlers = mIdleHandlers.toArray(mPendingIdleHandlers);
            }

            //只有第一次迭代的时候，才会执行下面代码
            for (int i = 0; i < pendingIdleHandlerCount; i++) {
                final IdleHandler idler = mPendingIdleHandlers[i];
                mPendingIdleHandlers[i] = null; // release the reference to the handler

                boolean keep = false;
                try {
                    keep = idler.queueIdle();
                } catch (Throwable t) {
                    Log.wtf(TAG, "IdleHandler threw exception", t);
                }
				//如果不保存空闲任务，执行完成后直接删除
                if (!keep) {
                    synchronized (this) {
                        mIdleHandlers.remove(idler);
                    }
                }
            }

            // 重置空闲的handler个数，因为不需要重复执行
            pendingIdleHandlerCount = 0;
			
			//当执行完空闲的handler的时候，新的native消息可能会进入，所以唤醒Native消息机制层
            nextPollTimeoutMillis = 0;
        }
    }
```
这里大家直接看MessageQueue的next()法肯定会很懵逼。妈的，这个nativePollOnce()方法是什么鬼，为毛它会阻塞呢？这个msg.isAsynchronous()判断又是怎么回事？妈的这个逻辑有点乱理解不了啊。大家不要慌，让我们带着这几个问题来慢慢分析。

### Native消息机制
其实在Android 消息处理机制中，不仅包括了Java层的消息机制处理，还包括了Native消息处理机制(与我们知道的Handler机制一样，也拥有Handler、Looper、MessageQueue)。这里我们不讲Native消息机制的具体代码细节，如果有兴趣的小伙伴，请查看----->[深入理解Java Binder和MessageQueue](https://blog.csdn.net/innost/article/details/47252865)

这里我们用一张图来表示Native消息与Jave层消息的关系（这里为大家提供了[Android源码](https://pan.baidu.com/s/1hAcAxFylMU0Aspg2dP9dBA)，大家可以按需下载），具体细节如下图所示：
![Native消息机制.png](https://upload-images.jianshu.io/upload_images/2824145-aebe13831a25a08e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240) (这里我用的别人的图，如有侵权，请联系我，马上删除)。

其实我们也可以从Java层中的MessageQueue中几个方法就可以看出来。其中声明了几个本地的方法。

```
    private native static long nativeInit();
    private native static void nativeDestroy(long ptr);
    private native void nativePollOnce(long ptr, int timeoutMillis);
    private native static void nativeWake(long ptr);
    private native static boolean nativeIsPolling(long ptr);
    private native static void nativeSetFileDescriptorEvents(long ptr, int fd, int events);
```
特别是在MessageQueue构造方法中。

```
    MessageQueue(boolean quitAllowed) {
        mQuitAllowed = quitAllowed;
        mPtr = nativeInit();//mPtr 其实是Native消息机制中MessageQueue的地址。
    }
```
在Java层中MessageQueue在初始化的时候，会调用本地方法去创建Native MessageQueue。并通过mPrt保存了Native中的MessageQueue的地址。

#### Native消息机制与Java层的消息机制有什么关系

想知道有什么关系，我们需要查看frameworks\base\core\jni\android_os_MessageQueue.cpp文件，
```
static jlong android_os_MessageQueue_nativeInit(JNIEnv* env, jclass clazz) {
    NativeMessageQueue* nativeMessageQueue = new NativeMessageQueue();
    if (!nativeMessageQueue) {
        jniThrowRuntimeException(env, "Unable to allocate native queue");
        return 0;
    }

    nativeMessageQueue->incStrong(env);
    return reinterpret_cast<jlong>(nativeMessageQueue);
}
```
其实nativeInit()方法很简单，初始化NativeMessageQueue对象然后返回其地址。现在我们继续查看NativeMessageQueue的构造函数。

```
NativeMessageQueue::NativeMessageQueue() :
        mPollEnv(NULL), mPollObj(NULL), mExceptionObj(NULL) {
    mLooper = Looper::getForThread();
    if (mLooper == NULL) {
        mLooper = new Looper(false);
        Looper::setForThread(mLooper);
    }
}
```
哇，我们看见了我们熟悉的"**Looper**",这段代码其实很好理解。Native Looper调用静态方法getForThread()，获取当前线程中的Looper对象。如果为空，则创建Native Looper对象。这里大家肯定会有个疑问。当前线程是指什么线程呢？想知道到底绑定是什么线程，我们需要进入Native Looper中查看setForThread()与getForThread()两个方法。


#### getForThread()从线程中获取设置的变量

```
/**
 * Returns the looper associated with the calling thread, or NULL if
 * there is not one.
 */
sp<Looper> Looper::getForThread() {
    int result = pthread_once(& gTLSOnce, initTLSKey);
    LOG_ALWAYS_FATAL_IF(result != 0, "pthread_once failed");
    return (Looper*)pthread_getspecific(gTLSKey);
}
```
这里pthread_getspecific()机制类似于Java层的ThreadLocal中的get()方法,是从线程中获取key值对应的数据。其中通过我们可以通过注释就能明白,Native Looper是存储在本地线程中的，而对应的线程，就是调用它的线程，而我们是在主线程中调用的。故Native Looper与主线程产生了关联。那么相应的setForThread()也是对主线程进行操作的了。接着看setForThread()方法。

#### setForThread()从线程中设置变量

```
/**
  * Sets the given looper to be associated with the calling thread.
  * If another looper is already associated with the thread, it is replaced. *
  * If "looper" is NULL, removes the currently associated looper.
  */ 
void Looper::setForThread(const sp<Looper>& looper) {
    sp<Looper> old = getForThread(); // also has side-effect of initializing TLS
    if (looper != NULL) {
        looper->incStrong((void*)threadDestructor);
    }
    pthread_setspecific(gTLSKey, looper.get());
    if (old != NULL) {
        old->decStrong((void*)threadDestructor);
    }
}
```
这里pthread_setspecific()机制类似于Java层的ThreadLocal中的set()方法。通过注释我们明白将Native looper放入调用线程，如果已经存在，就替换。如果为空就删除。

#### nativePollOnce()方法为什么会导致主线程阻塞？

经过上文的讨论与分析，大家现在已经知道了，在Android消息机制中不仅有 Java层的消息机制，还有Native的消息机制。既然要出里Native的消息机制。那么肯定有一个处理消息的方法。那么调用本地消息机制消息的方法必然就是nativePollOnce()方法。

```
static void android_os_MessageQueue_nativePollOnce(JNIEnv* env, jobject obj,
        jlong ptr, jint timeoutMillis) {
    NativeMessageQueue* nativeMessageQueue = reinterpret_cast<NativeMessageQueue*>(ptr);
    nativeMessageQueue->pollOnce(env, obj, timeoutMillis);
}
```
在nativePollOnce()方法中调用nativeMessageQueue的pollOnce()方法，我们接着走。

```
void NativeMessageQueue::pollOnce(JNIEnv* env, jobject pollObj, int timeoutMillis) {
    mPollEnv = env;
    mPollObj = pollObj;
    mLooper->pollOnce(timeoutMillis);
    mPollObj = NULL;
    mPollEnv = NULL;

    if (mExceptionObj) {
        env->Throw(mExceptionObj);
        env->DeleteLocalRef(mExceptionObj);
        mExceptionObj = NULL;
    }
}
```
这里我们发现pollOnce(timeoutMillis)内部调用的是Natave looper中的 pollOnce(int timeoutMillis, int* outFd, int* outEvents, void** outData)方法。继续看。
```
int Looper::pollOnce(int timeoutMillis, int* outFd, int* outEvents, void** outData) {
    int result = 0;
    for (;;) {
        while (mResponseIndex < mResponses.size()) {
            const Response& response = mResponses.itemAt(mResponseIndex++);
            int ident = response.request.ident;
            if (ident >= 0) {
                int fd = response.request.fd;
                int events = response.events;
                void* data = response.request.data;
#if DEBUG_POLL_AND_WAKE
                ALOGD("%p ~ pollOnce - returning signalled identifier %d: "
                        "fd=%d, events=0x%x, data=%p",
                        this, ident, fd, events, data);
#endif
                if (outFd != NULL) *outFd = fd;
                if (outEvents != NULL) *outEvents = events;
                if (outData != NULL) *outData = data;
                return ident;
            }
        }

        if (result != 0) {
#if DEBUG_POLL_AND_WAKE
            ALOGD("%p ~ pollOnce - returning result %d", this, result);
#endif
            if (outFd != NULL) *outFd = 0;
            if (outEvents != NULL) *outEvents = 0;
            if (outData != NULL) *outData = NULL;
            return result;
        }

        result = pollInner(timeoutMillis);
    }
}
```
由于篇幅的限制，这里就简单介绍一哈pollOnce()方法。**该方法会一直等待Native消息，其中 timeOutMillis参数为超时等待时间。如果为-1，则表示无限等待，直到有事件发生为止。如果值为0，则无需等待立即返回。** 那么既然nativePollOnce()方法有可能阻塞，那么根据上文我们讨论的MessageQueue中的enqueueMessage中的nativeWake()方法。大家就应该了然了。nativeWake()方法就是唤醒Native消息机制不再等待消息而直接返回。

#### nativePollOnce()一直循环为毛不造成主线程的卡死？
>到了这里，其实大家都会有个疑问，如果当前主线程的MessageQueue没有消息时，程序就会便阻塞在loop的queue.next()中的nativePollOnce()方法里，一直循环那么主线程为什么不卡死呢？这里就涉及到Linux pipe/epoll机制，此时主线程会释放CPU资源进入休眠状态，直到下个消息到达或者有事务发生，通过往pipe管道写端写入数据来唤醒主线程工作。这里采用的epoll机制，是一种IO多路复用机制，可以同时监控多个描述符，当某个描述符就绪(读或写就绪)，则立刻通知相应程序进行读或写操作，本质同步I/O，即读写是阻塞的。 所以说，主线程大多数时候都是处于休眠状态，并不会消耗大量CPU资源。--摘自[Gityuan知乎回答](https://www.zhihu.com/question/34652589/answer/90344494)

如果大家想消息了解Native 消息机制的处理机制，请查看----->[深入理解Java Binder和MessageQueue](https://blog.csdn.net/innost/article/details/47252865)



###屏障消息与异步消息

#### 屏障消息
在next()方法中，有一个屏障的概念(message.target ==null为屏障消息)，如下代码：

```
  if (msg != null && msg.target == null) {
                    // Stalled by a barrier.  Find the next asynchronous message in the queue.
                    do {
                        prevMsg = msg;
                        msg = msg.next;
                    } while (msg != null && !msg.isAsynchronous());
                }
```
其实通过代码，当出现屏障的时候，会滤过同步消息，而是直接获取其中的异步消息并返回。如下图所示：

{% asset_img 屏障与异步消息.png 屏障与异步消息 %}

**在Hadnler无参的构造函数中，默认设置的消息都是同步的。**那我们就可以知道在Android中消息分为了两种，一种是同步消息，另一种是异步消息。在官方的解释中，异步消息通常代表着中断，输入事件和其他信号，这些信号必须独立处理，即使其他工作已经暂停。

#### 异步消息
要设置异步消息，直接调用Message的setAsynchronous()方法，方法如下：
```
    /**
     * Sets whether the message is asynchronous, meaning that it is not
     * subject to {@link Looper} synchronization barriers.
     * <p>
     * Certain operations, such as view invalidation, may introduce synchronization
     * barriers into the {@link Looper}'s message queue to prevent subsequent messages
     * from being delivered until some condition is met.  In the case of view invalidation,
     * messages which are posted after a call to {@link android.view.View#invalidate}
     * are suspended by means of a synchronization barrier until the next frame is
     * ready to be drawn.  The synchronization barrier ensures that the invalidation
     * request is completely handled before resuming.
     * </p><p>
     * Asynchronous messages are exempt from synchronization barriers.  They typically
     * represent interrupts, input events, and other signals that must be handled independently
     * even while other work has been suspended.
     * </p><p>
     * Note that asynchronous messages may be delivered out of order with respect to
     * synchronous messages although they are always delivered in order among themselves.
     * If the relative order of these messages matters then they probably should not be
     * asynchronous in the first place.  Use with caution.
     * </p>
     *
     * @param async True if the message is asynchronous.
     *
     * @see #isAsynchronous()
     */
    public void setAsynchronous(boolean async) {
        if (async) {
            flags |= FLAG_ASYNCHRONOUS;
        } else {
            flags &= ~FLAG_ASYNCHRONOUS;
        }
    }
```
大家可以看到，设置异步消息，官方文档对其有详细的说明，侧面体现出了异步消息的重要性，那下面我就带着大家一起来理一理官方的注释说明。
- 如果设置了异步消息，异步消息将不会受到屏障的影响（从next()方法中，我们已经了解了，当出现屏障的时候，同步消息会直接被过滤。直接返回最近的异步消息）
- 在某些操作中，例如视图进行invalidation(视图失效，进行重绘)，会引入屏障消息（也就是将message.target ==null的消息放入消息队列中），已防止后续的同步消息被执行。同时同步消息的执行会等到视图重绘完成后才会执行。


#### 有哪些操作是异步消息呢？
这里我就直接通过ActivityThread中的几个异步消息给大家做一些简单介绍。这里我就不用代码展示了，用图片来表示更清晰明了。

{% asset_img ActivityThread中的异步消息.png ActivityThread中的异步消息 %}
在ActivityThread中，有一个sendMessage()多个参数方法。我们明显的看出，有四个消息是设置为异步消息的。DUMP_SERVICE、DUMP_HEAP、DUMP_ACTIVITY、DUMP_PROVIDER。从字面意思就可以看出来。回收service、回收堆内存、回收Activity、回收Provider都属于异步消息。

#### 屏障消息发送的时机

那么Android中在哪些情况下会发生屏障消息呢？其实最为常见的就是在我们界面进行绘制的时候，如在ViewRootImpl.scheduleTraversals（）中。
```
 void scheduleTraversals() {
        if (!mTraversalScheduled) {
            mTraversalScheduled = true;
            //发送屏障消息
            mTraversalBarrier = mHandler.getLooper().getQueue().postSyncBarrier();
            mChoreographer.postCallback(
                    Choreographer.CALLBACK_TRAVERSAL, mTraversalRunnable, null);
            if (!mUnbufferedInputDispatch) {
                scheduleConsumeBatchedInput();
            }
            notifyRendererOfFramePending();
            pokeDrawLockIfNeeded();
        }
    }
```
在调用scheduleTraversals()方法时，我们发现会发生一个屏障过去。具体代码如下：

```
private int postSyncBarrier(long when) {
      
        synchronized (this) {
	        //记录屏障消息的个数
            final int token = mNextBarrierToken++;
            final Message msg = Message.obtain();
            msg.markInUse();
            msg.when = when;
            msg.arg1 = token;
			
			//按照消息队列的等待时间，将屏障按照顺序插入
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
```
这里我们直接将围栏放在了消息队列中，同时重要的是我们并没有直接设置target,也就是tartget =null。其实现在我们可以想象，我们当我们正在进行界面的绘制的时候，我们是不希望有其他操作的，这个时候，要排除同步消息操作，也是可能理解的。


### IdleHandler(MessageQueuqe空闲时执行的任务)
在MessageQueue中的next()方法，出现了**IdleHandler**(MessageQueuqe空闲时执行的任务)，查看MessageQueue中IdleHander接口的说明：
```
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
        boolean queueIdle();
    }
```
当线程正在等待更多消息时，会回调该接口，同时queueIdl（）方法，**会在消息队列执行完所有的消息等待且在等待更多消息时会被调用**，如果返回true,表示该任务会在消息队列等待更多消息的时候继续执行，如果为false，表示该任务执行完成后就会被删除，不再执行。

其中MessageQueue通过使用addIdleHandler(@NonNull IdleHandler handler) 方法添加空闲时任务。具体代码如下：
```
 public void addIdleHandler(@NonNull IdleHandler handler) {
        if (handler == null) {
            throw new NullPointerException("Can't add a null IdleHandler");
        }
        synchronized (this) {
            mIdleHandlers.add(handler);
        }
    }

```
既然MessageQueue可以添加空闲时任务，那么我们就看看最为明显的ActivityThread中声明的GcIdler。在ActivityThread中的**H**收到**GC_WHEN_IDLE**消息后，会执行scheduleGcIdler，将GcIdler添加到MessageQueue中的空闲任务集合中。具体如下：

```
 void scheduleGcIdler() {
        if (!mGcIdlerScheduled) {
            mGcIdlerScheduled = true;
            //添加GC任务
            Looper.myQueue().addIdleHandler(mGcIdler);
        }
        mH.removeMessages(H.GC_WHEN_IDLE);
    }
```

ActivityThread中GcIdler的详细声明：
```
   //GC任务
   final class GcIdler implements MessageQueue.IdleHandler {
        @Override
        public final boolean queueIdle() {
            doGcIfNeeded();
            //执行后，就直接删除
            return false;
        }
    }
	// 判断是否需要执行垃圾回收。
    void doGcIfNeeded() {
        mGcIdlerScheduled = false;
        final long now = SystemClock.uptimeMillis();
	    //获取上次GC的时间
        if ((BinderInternal.getLastGcTime()+MIN_TIME_BETWEEN_GCS) < now) {
            //Slog.i(TAG, "**** WE DO, WE DO WANT TO GC!");
            BinderInternal.forceGc("bg");
        }
    }
```
GcIdler方法理解起来很简单、就是获取上次GC的时间，判断是否需要GC操作。如果需要则进行GC操作。这里ActivityThread中还声明了其他空闲时的任务。如果大家对其他空闲任务感兴趣，可以自行研究。

### 什么时候唤醒主线程呢？
通过上文的了解，大家已经知道了Native的消息机制可能会导致主线程阻塞，那么唤醒Native消息机制**（让Native消息机制不在等待Native消息，也就是nativePollOnce（）方法返回）**在整个Android的消息机制中尤为重要，这里放在这里给大家讲是因为唤醒的条件涉及到屏障消息与空闲任务。大家理解了这两个内容后再来理解唤醒的时机就相对容易一点了，这里我们分别对唤醒的两个时机进行讲解。

#### 在添加消息到消息队列中
```
boolean enqueueMessage(Message msg, long when) {
	    ...省略部分代码
        synchronized (this) {
	      ...省略部分代码
            msg.markInUse();
            msg.when = when;
            Message p = mMessages;//头部消息
            boolean needWake;
            //如果队列中没有消息，或者当前进入的消息比消息队列中的消息等待时间短，那么就放在消息队列的头部
            if (p == null || when == 0 || when < p.when) {
                // New head, wake up the event queue if blocked.
                msg.next = p;
                mMessages = msg;
                needWake = mBlocked;
            } else {
	            //判断唤醒条件，当前消息队列头部消息是屏障消息，且当前插入的消息为异步消息
	            //且当前消息队列处于无消息可处理的状态
                needWake = mBlocked && p.target == null && msg.isAsynchronous();
                Message prev;
                //循环遍历消息队列，把当前进入的消息放入合适的位置（比较等待时间）
                for (;;) {
                    prev = p;
                    p = p.next;
                    if (p == null || when < p.when) {
                        break;
                    }
                    if (needWake && p.isAsynchronous()) {
                        needWake = false;
                    }
                }
                //将消息插入合适的位置
                msg.next = p; 
                prev.next = msg;
            }

			//调用nativeWake，以触发nativePollOnce函数结束等待
            if (needWake) {
                nativeWake(mPtr);
            }
        }
        return true;
    }
```
上述代码，我们很明显的看见Native消息机制的唤醒，受到**needWake**这个变量影响，**needWake ==true**是在两个条件下。

- 第一个：如果当前消息按照等待时间排序是在消息队列的头部, **needWake = mBlocked**，且**mBlocked**会在当前消息队列中没有消息可以处理，且没有空闲任务的条件下为**true**（mBlocked变量的赋值会在下文讲解）。
- 第二个：如果当前**mBlocked=true（第一个条件判断）**，且消息队列头部消息是屏障消息，同时当前插入的消息为异步消息的条件。**needWake = true**。


#### 在空闲任务完成的时候唤醒
```
 Message next() {
		 ...省略部分代码
        int pendingIdleHandlerCount = -1; // -1 only during first iteration
        int nextPollTimeoutMillis = 0;
        for (;;) {
	         ...省略部分代码
			//执行native层消息机制层,
			//timeOutMillis参数为超时等待时间。如果为-1，则表示无限等待，直到有事件发生为止。
			//如果值为0，则无需等待立即返回。该方法可能会阻塞
            nativePollOnce(ptr, nextPollTimeoutMillis);
            synchronized (this) {
	             ...省略部分代码
            
                //获取空闲时处理任务的handler 用于发现线程何时阻塞等待更多消息的回调接口。
                if (pendingIdleHandlerCount < 0
                        && (mMessages == null || now < mMessages.when)) {
                    pendingIdleHandlerCount = mIdleHandlers.size();
                }
                if (pendingIdleHandlerCount <= 0) {
                //如果消息队列中没有消息可以处理，且没有空闲任务，那么就继续等待消息
                    mBlocked = true;
                    continue;
                }
			   ...省略部分代码
            }

	        ...省略执行空闲任务代码
	        
            // 重置空闲的handler个数，因为不需要重复执行
            pendingIdleHandlerCount = 0;
			
			//当执行完空闲的handler的时候，新的native消息可能会进入，所以唤醒Native消息机制层
            nextPollTimeoutMillis = 0;
        }
    }
```
这里我们可以看到 **mBlocked = true**的条件是在消息队列中没有消息可以处理，且也没有空闲任务的情况下。也就是当前**mBlocked = true**会影响到MessageQueue中enqueueMessage()方法是否唤醒主线程。

如果当前空闲任务完成后，**会将nextPollTimeoutMillis 置为0，**如果nextPollTimeoutMillis =0，会导致nativePollOnce直接返回，也就是会直接唤醒主线程（唤醒Native消息机制层）。

### MessageQueue取出消息整体流程

到目前为止，大家已经对整个消息的发送与取出有一个大概的了解了。这里我着重对MessageQueue取消息的流程画了一个简单的流程图。希望大家根据对取消息有个更好的理解。
{% asset_img 整体流程.jpg 整体流程 %}


### 总结
- Handler在发消息时，MessageQueue已经对消息**按照了等待时间**进行了排序。
- MessageQueue**不仅包含了Java层消息机制同时包含Native消息机制**
- Handler消息分为**异步消息**与**同步消息**两种。
- MessageQueue中存在**“屏障消息“**的概念，当出现屏障消息时，会执行最近的异步消息，同步消息会被过滤。
- MessageQueue在执行完消息队列中的消息等待更多消息时，**会处理一些空闲任务，如GC操作等。**

#### 感谢
站在巨人的肩膀上。可以看得更远。该篇文章参阅了一下几本图书与源码。我这里我给了百度云盘的下载链接。大家可以按需下载。

[深入理解Android 卷1,2,3](https://pan.baidu.com/s/1g8-Hj7SykM5o-NrgigvhMA)

[Android源码](https://pan.baidu.com/s/1hAcAxFylMU0Aspg2dP9dBA)

[Android开发艺术探索](https://pan.baidu.com/s/19M6xuFYekGXF0BSEM1Kh9w)
