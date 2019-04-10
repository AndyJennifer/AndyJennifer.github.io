---
title: Android Handler机制之Message及Message回收机制
date: 2019-02-23 22:08:55
categories:
- Handler相关
tags: 
- 异步任务
---

![小松鼠.jpg](https://upload-images.jianshu.io/upload_images/2824145-93bba9f12e53bb0d.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


>该文章属于Android Handler系列文章，如果想了解更多，请点击
[《Android Handler机制之总目录》](https://www.jianshu.com/p/43bb31d8a742)

### 前言
在前面的文章中我们讲解了Handler、Looper、MessageQueue的具体关系，了解了具体的消息循环的流程。下面将一起来探讨最为整个消息循环的消息载体Message。

### Message中可以携带的信息
Message中可以携带的数据比较丰富，下面对一些常用的数据进行了分析。
```
/**
 * 用户定义的消息代码，以便当接受到消息是关于什么的。其中每个Hanler都有自己的命名控件，不用担心会冲突
 */	
 public int what;
/**
 * 如果你只想存很少的整形数据，那么可以考虑使用arg1与arg2,
 * 如果需要传输很多数据可以使用Message中的setData(Bundle bundle)
 */
 public int arg1;
/**
 * 如果你只想存很少的整形数据，那么可以考虑使用arg1与arg2,
 * 如果需要传输很多数据可以使用Message中的setData(Bundle bundle)
 */
 public int arg2;
/**
 * 发送给接受方的任意对象，在使用跨进程的时候要注意obj不能为null
 */
 public Object obj;
/**
 * 在使用跨进程通信Messenger时，可以确定需要谁来接收
 */
 public Messenger replyTo;
/**
 * 在使用跨进程通信Messenger时，可以确定需要发消息的uid
 */
 public int sendingUid = -1;
/**
 * 如果数据比较多，可以直接使用Bundle进行数据的传递
 */
 Bundle data;
```
其中关于what的值为什么不会冲突的原因是，之前我们讲过的handler是与线程进行绑定的。也就是说不同消息循环消息的发送，处理的线程是不一样的。当然是不会冲突的。对于Messenger，因为涉及到Binder机制，这里就不过多的描述了，有兴趣的小伙伴可以自行查询相关资料学习。


### 创建消息的方式
官方建议使用Message.obtain()系列方法来获取Message实例，因为其Message实例是直接从Handler的消息池中获取的，可以循环利用，不必另外开辟内存空间，效率比直接使用new Message（）创建实例要高。其中具体创建消息的方式，我已经为大家分好类了。具体分类如下：
```
//无参数
public static Message obtain() {...}
//带Messag参数
public static Message obtain(Message orig) {}
//带Handler参数
public static Message obtain(Handler h) {}
public static Message obtain(Handler h, Runnable callback){}
public static Message obtain(Handler h, int what){}
public static Message obtain(Handler h, int what, Object obj){}
public static Message obtain(Handler h, int what, int arg1, int arg2){}
public static Message obtain(Handler h, int what,int arg1, int arg2, Object obj) {}
```
其中在Message的obtain带参数的方法中，内部都会调用无参的obtain()方法来获取消息后。然后并根据其传入的参数，对Message进行赋值。（关于具体的obtain方法会在下方消息池实现原理中具体描述）

### 消息池实现原理
既然官方建议使用消息池来获取消息，那么在了解其内部机制之前，我们来看看Message中的消息池的设计。具体代码如下：
```
private static final Object sPoolSync = new Object();//控制获取从消息池中获取消息。保证线程安全
private static Message sPool;//消息池
private static int sPoolSize = 0;//消息池中回收的消息数量
private static final int MAX_POOL_SIZE = 50;//消息池最大容量
```
从Message的消息池设计，我们大概能看出以下几点：
1. 该消息池在同一个消息循环中是共享的（sPool声明为static)，
2. 消息池中的最大容量为50，
3. 从消息池获取消息是线程安全的。

####  从消息池中获取消息
在上文中，我们已经知道了在使用消息池获得消息时，都会调用无参的obtain（）方法。具体代码如下：
```
 public static Message obtain() {
        synchronized (sPoolSync) {
            if (sPool != null) {
                Message m = sPool;
                sPool = m.next;
                m.next = null;
                m.flags = 0; //重新标识当前Message没有使用过
                sPoolSize--;
                return m;
            }
        }
        return new Message();//如果为空直接返回
    }
```
从上述代码中，我们可以了解，也就是当前 消息池不为空（sPool !=null)的情况下，那么我们就可以从消息池中获取数据，相应的消息池中的消息数量会减少。**消息池的内部实现是以链表的形式**，其中spol指针指向当前链表的头结点，从消息池中获取消息是**以移除链表中sPool所指向的节点的形式**，具体原理如下图所示：
![获取消息.png](https://user-gold-cdn.xitu.io/2018/9/22/16601c88465668b4?w=883&h=562&f=png&s=24696)

#### 回收消息到消息池
在Meaage的消息回收中，消息的实际回收方法是recycleUnchecked（）方法，具体如下图所示：
```
   void recycleUnchecked() {
	    //用于表示当前Message消息已经被使用过了
        flags = FLAG_IN_USE;
        //情况之前Message的数据
        what = 0;
        arg1 = 0;
        arg2 = 0;
        obj = null;
        replyTo = null;
        sendingUid = -1;
        when = 0;
        target = null;
        callback = null;
        data = null;
		//判断当前消息池中的数量是不是小于最大数量，其中 MAX_POOL_SIZE=50
        synchronized (sPoolSync) {
            if (sPoolSize < MAX_POOL_SIZE) {
                next = sPool;
                sPool = this;
                sPoolSize++;//记录当前消息池中的数量
            }
        }
    }
```
在recycleUnchecked（）方法中，大致分为三步，第一步将该条回收的消息状态设置为正在使用，第二步将Message所有的存储信息都变为初始值，第三步，如果当前消息池仍能够存储回收的消息，那么就将消息存储在消息池中。**其中将回收消息加入消息池中是使用链表的形式**，具体回收消息到消息池如下图所示：
![加入消息.png](https://user-gold-cdn.xitu.io/2018/9/22/16601c88462010a4?w=883&h=591&f=png&s=21165)

###  Message 消息回收时机
这里为了方便大家梳理逻辑，我提前将几种会调用消息进行回收的情况都描述出来了，具体的情况如下所示：

#### 当Handler指定删除单条消息，或所有消息的时候
```
void removeMessages(Handler h, int what, Object object)
void removeMessages(Handler h, Runnable r, Object object)
void removeCallbacksAndMessages(Handler h, Object object)
```
当使用Handler删除某条消息的时候，会分别调用MessageQueue的
- removeMessages(Handler h, int what, Object object)
- removeCallbacksAndMessages(Handler h, Object object) 
- removeMessages(Handler h, Runnable r, Object object) 

这三个方法逻辑比较类似。这里直接选取removeCallbacksAndMessages（）方法来进行讲解。具体代码如下：

```
 void removeCallbacksAndMessages(Handler h, Object object) {
        if (h == null) {
            return;
        }
        synchronized (this) {
            Message p = mMessages;
            //  第一次循环
            while (p != null && p.target == h
                    && (object == null || p.obj == object)) {
                 //下面操作会将满足回收条件的消息，从消息队列中移除
                Message n = p.next;
                mMessages = n;
                p.recycleUnchecked();
                p = n;
            }
            // 第二次循环
            while (p != null) {
                Message n = p.next;
                if (n != null) {
                    if (n.target == h && (object == null || n.obj == object)) {
                    //下面操作会将满足回收条件的消息，从消息队列中移除
                        Message nn = n.next;
                        n.recycleUnchecked();
                        p.next = nn;
                        continue;
                    }
                }
                p = n;
            }
        }
    }
```
在removeCallbacksAndMessages(Handler h, Object object)方法中，在该方法中分别进行了两次循环，肯定有很多读者朋友会很好奇，为什么这里会进行两次循环呢？下面我就具体来讲解一下。

我们都知道，在Handler机制中，`多个handler对应同一个MessageQueue对应同一个Looper，Handler与MessageQueue与Looper之间的关系是N：1：1`。也就是说在MessageQueue中我们可以有多个不同Handler发送的Message。那么我们再结合上面的代码，我们来分析这两次循环。

##### 第一次循环
根据上文对代码的理解，第一次循环会将MessageQueue中，当前Handler发送的所有消息移除，`注意!!!!!!!!!这里并不会将整个MessageQueue中的当前Handler发送的消息全部移除`，而是在遍历过程中，如果有其他Handler发送的消息,那么就会将mMessages指向头结点并跳出循环。如下图所示：

![第一次循环.png](https://upload-images.jianshu.io/upload_images/2824145-451f619f15b42f36.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

##### 第二次循环
经过上文的分析，我们已经知道了，在进行第一次循环后，已经将在removeCallbacksAndMessages方法执行时所有对应的Handler发送的消息移除掉了，但是MessageQueue中可能任然会残留没有移除掉的消息。那么第二次循环，根据代码来理解的话，我们可以得到下图：

![第二次循环.png](https://upload-images.jianshu.io/upload_images/2824145-8d6a841222333799.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

到这里有可能有小伙伴就会想，为什么不执行一次循环就将所有的对应Handler发送的消息全部移除了呢？这里之所以要执行两次循环的原因是，你并不能保证当移除消息的时候，对应的Handler就不继续发送消息了，也就是说该Handler发送的消息仍然会被添加到MessageQueue中，`所以为了保证将整个MessageQueue中该Handler发送的消息全部被移除，在第一次循环移除之后，我们必须要再执行一次循环移除操作`。

#### 当Loooper取出消息时
```
    public static void loop() {
		 //省略部分代码
        for (;;) {
            Message msg = queue.next(); // might block
            if (msg == null) {
                // No message indicates that the message queue is quitting.
                return;
            }
			//省略部分代码
            try {
                msg.target.dispatchMessage(msg);
                end = (slowDispatchThresholdMs == 0) ? 0 : SystemClock.uptimeMillis();
            } finally {
                if (traceTag != 0) {
                    Trace.traceEnd(traceTag);
                }
            }
		    //省略部分代码
		    
		    //回收消息
            msg.recycleUnchecked();
        }
    }
```
我们都知道消息的取出是通过Looper类中的loop方法。从代码中我们可以看出，当消息取出并执行相应操作后。最后会将消息回收。
#### 当Looper取消循环消息队列的时候
```
public void quitSafely() { mQueue.quit(true);}
public void quit() { mQueue.quit(false); }
```
当退出消息队列的时候，也就是调用Loooper的quitSafely（）或quit（）方法，从代码中我们可以看出，会调用其内部的MessageQueue的quit(boolean safe)方法。我们继续跟踪代码。

```
   void quit(boolean safe) {
        if (!mQuitAllowed) {//注意，主线程是不能退出消息循环的
            throw new IllegalStateException("Main thread not allowed to quit.");
        }

        synchronized (this) {
            if (mQuitting) {//如果当前循环消息已经退出了，直接返回
                return;
            }
            mQuitting = true;
			
            if (safe) {//如果是安全退出
                removeAllFutureMessagesLocked();
            } else {//如果不是安全退出
                removeAllMessagesLocked();
            }

            // We can assume mPtr != 0 because mQuitting was previously false.
            nativeWake(mPtr);
        }
    }
```
在MessageQueue的quit(boolean safe)方法中，会将mQuitting （用于判断当前消息队列是否已经退出）置为true，同时会根据当前是否安全退出的标志 (safe)来走不同的逻辑,如果安全则走removeAllFutureMessagesLocked（）方法，如果不是安全退出则走removeAllMessagesLocked（）方法。下面分别对这两个方法进行讨论。

##### 非安全退出
```
    private void removeAllMessagesLocked() {
        Message p = mMessages;
        while (p != null) {
            Message n = p.next;
            p.recycleUnchecked();
            p = n;
        }
        mMessages = null;
    }
```
非安全退出其实很简单，就是将所有消息队列中的消息全部回收。具体示意图如下所示：

![回收全部消息.png](https://user-gold-cdn.xitu.io/2018/9/22/16601c884690d0ad?w=1240&h=517&f=png&s=91646)

##### 安全退出
```
   private void removeAllFutureMessagesLocked() {
        final long now = SystemClock.uptimeMillis();
        Message p = mMessages;//当前队列中的头消息
        if (p != null) {
            if (p.when > now) {//判断时间，如果Message的取出时间比当前时间要大直接移除
                removeAllMessagesLocked();
            } else {
                Message n;
                for (;;) {//继续判断，取队列中所有大于当前时间的消息
                    n = p.next;
                    if (n == null) {
                        return;
                    }
                    if (n.when > now) {
                        break;
                    }
                    p = n;
                }
                p.next = null;
                do {//将所有所有大于当前时间的消息的消息回收
                    p = n;
                    n = p.next;
                    p.recycleUnchecked();
                } while (n != null);
            }
        }
    }
```
观察上诉代码，在该方法中，会判断当前消息队列中的头消息的时间是否大于当前时间，如果大于当前时间就会removeAllMessagesLocked（）方法（也就是回收全部消息），反之，则回收部分消息，同时没有被回收的消息任然可以被取出执行。具体示意图如下所示：


![回收部分消息.png](https://user-gold-cdn.xitu.io/2018/9/22/16601c8846acbc2c?w=1240&h=487&f=png&s=89553)



#### 当消息队列退出的，但是仍然发送消息过来的时候

在Looper调用quit()方法时，也就是Looper退出消息循环的时候，我们已经知道了其内部会调用MessageQueue的quit(boolean safe)方法。当MessageQueue退出的时候，会将mQuitting置为true。那么当对应的Handler发送消息时，我们都知道会调用MessageQueue的enqueueMessage（Message msg, long when）方法。那么现在我们观察下列代码：

```
boolean enqueueMessage(Message msg, long when) {
	   ...省略部分代码
        synchronized (this) {
          ...省略部分代码
            if (mQuitting) {
                IllegalStateException e = new IllegalStateException(
                        msg.target + " sending message to a Handler on a dead thread");
                Log.w(TAG, e.getMessage(), e);
                msg.recycle();
                return false;
            }
            ...省略部分代码
     }
```
观察该代码我们得知，当循环消息退出的时候，如果这个时候Handler继续发送消息来。会将该消息回收。但是现在这里有个问题。既然我们的消息队列已经结束循环了。那么我们回收该消息又有什么用呢？我们又不能重新的开启消息循环。不知道Google这里为什么会这么设计。


### 总结
- 在使用Handler发消息时，建议使用Message.obtin()方法，从消息池中获取消息。
- 在Message中消息池是使用链表的形式来存储消息的。
- 在Message中消息池中最大允许存储50条的消息。
- 在使用Handler移除某条消息的时候，该消息有可能会被消息池回收。（会判断消息池是否仍然能存储消息）