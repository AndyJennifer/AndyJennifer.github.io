---
title: Android Handler机制之循环消息队列的退出
date: 2019-02-23 22:10:33
categories:
- Handler相关
tags: 
- 异步任务
---


![啦啦.jpeg](https://upload-images.jianshu.io/upload_images/2824145-8bc6d10af2118f1b.jpeg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

>该文章属于Android Handler系列文章，如果想了解更多，请点击{% post_link Android-Handler机制之总目录 %}

### 前言
在上几篇文中我们介绍了整个消息的循环机制以及消息的回收。现在我们来看看整么退出循环消息队列。（到现在为止，整个Android Handler机制快要接近尾声了。不知道大家看了整个系列的文章，有没有对Handler机制有个深一点的了解。如果对你有所帮助，我也感到十分的开心与自豪~~~）。

### 消息队列的退出
要想结束循环消息队列，需要调用Loooper的quitSafely（）或quit（）方法，具体代码如下所示：
```
public void quitSafely() { mQueue.quit(true);}
public void quit() { mQueue.quit(false); }
```
而该两个方法的内部，都会调用Loooper中对应的MessageQueue的quit(boolean safe)方法。查看quit(boolean safe)方法：

```
   void quit(boolean safe) {
        if (!mQuitAllowed) {//注意，主线程是不能退出消息循环的
            throw new IllegalStateException("Main thread not allowed to quit.");
        }

        synchronized (this) {
            if (mQuitting) {//如果当前循环消息已经退出了，直接返回
                return;
            }
            mQuitting = true;//该标记十分重要，十分重要
			
            if (safe) {//如果是安全退出
                removeAllFutureMessagesLocked();
            } else {//如果不是安全退出
                removeAllMessagesLocked();
            }
            nativeWake(mPtr);
        }
    }
```
在MessageQueue的quit(boolean safe)方法中，会将**mQuitting （用于判断当前消息队列是否已经退出，该标志位十分有用，下文会提到）置为true**，同时会根据当前是否安全退出的标志 (safe)来走不同的逻辑,如果安全则走removeAllFutureMessagesLocked（）方法，如果不是安全退出则走removeAllMessagesLocked（）方法。下面分别对这两个方法进行讨论。

#### 非安全退出 removeAllMessagesLocked()方法
```
    private void removeAllMessagesLocked() {
        Message p = mMessages;//消息队列中的头节点
        while (p != null) {
            Message n = p.next;
            p.recycleUnchecked();//回收消息
            p = n;
        }
        mMessages = null;//将消息队列中的头节点置为null
    }
```
非安全退出其实很简单，就是遍历消息队列中的消息（消息队列内部结构是链表）将所有消息队列中的消息全部回收。同时将MessageQueue中的mMessages （消息队列中的头消息）置为null，其中关于Message的recycleUnchecked()方法，如果你对该方法不是很熟悉，建议先阅读{% post_link Android-Handler机制之Message及Message回收机制 %}。关于非安全退出时，消息队列中的回收示意图如下所示：

![回收全部消息.png](https://upload-images.jianshu.io/upload_images/2824145-012dc5d65a9d84b2.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

#### 退出消息循环的具体逻辑
上文中，我们描述了在非安全退出时MessageQueue中仅仅进行了消息的回收，而并没有真正涉及到循环消息队列的退出，现在我们就来看一看息循环退出的具体逻辑。我们都知道整个消息循环的消息获得，都是通过Loooper中的loop（）方法，具体代码如下所示：

```
 public static void loop() {
        final Looper me = myLooper();
	    //省略部分代码...
        for (;;) {
            Message msg = queue.next(); // might block
            if (msg == null) {
	            //如果没有消息，那么说明当前消息队列已经退出
                return;
            }
        //省略部分代码
        }
    }
```
从代码中我们可以退出，整个loop()方法的结束，会与MessageQueue的next（）方法相关，如果next（）方法获取的msg为null,那么Looper中的loop方法也直接结束。那么整个消息循环也退出了。那接下来我们来查看Message 中的next()方法，具体代码如下所示：
```
  Message next() {
	    //省略部分代码...
        for (;;) {
            if (nextPollTimeoutMillis != 0) {
                Binder.flushPendingCommands();
            }
            synchronized (this) {
               //省略部分代码..
                if (mQuitting) {
                    dispose();
                    return null;
                }
                //省略部分代码..
        }
    }
```
在Message中的next()方法中，如果mQuitting为true，该方法会直接就返回了。**大家还记得我们mQuitting是什么时候置为true的吗？**对！就是我们调用Loooper的quitSafely（）或quit（）方法（也就是调用MessageQueue.quit(boolean safe)）时，会将mQuitting置为true。

那么到现在整个循环消息队列的退出的逻辑就清楚了。主要分为以下几个步骤:
1. Looper调用quitSafely（）或quit（）方法时会导致mQuitting标志位为true。
2. Looper调用quitSafely（）或quit（）方法时，内部会分别走MessageQueue的removeAllFutureMessagesLocked（）与removeAllMessagesLocked（）,上述两种方法会回收消息队列中的消息。
3. Looper整个的消息循环是通过其loop（）方法。当MessageQueue中的next()获取的消息为空时，或导致整个循环消息队列的退出。  
4. MessageQueue中的next（）方法受mQuitting标志位影响，当mQuitting=true时，next()方法会返回null。

#### 安全退出removeAllFutureMessagesLocked()方法
现在为止，我们已经基本了解了整个循环消息队列退出的流程了。在了解了非安全退出的方法之后，我们再来看看安全退出时，涉及到的逻辑操作。上文我们已经提到过了，当Looper调用quitSafely（）方法时，内部会走MessagQueue的removeAllFutureMessagesLocked（）。具体代码如下：

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


![回收部分消息.png](https://upload-images.jianshu.io/upload_images/2824145-104a9f156f672df4.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

当使用安全退出循环消息队列时，整个退出逻辑与非安全退出有一定的区别。在上文中我们说过。当安全退出时，程序会判断消息队列会根据消息中的message.when来判断是否回收消息。那么在消息队列中没有被回收的消息是仍然能被取出执行的。具体代码如下所示：

```
   Message next() {
	    //省略部分代码...
        for (;;) {
            //省略部分代码...
            synchronized (this) {
	            //省略部分代码...
	            Message prevMsg = null;
                Message msg = mMessages;
                if (msg != null) {
                    if (now < msg.when) {
		              //省略部分代码...
                    } else {
					  //省略部分代码...
					   
					   //第一处。如果消息队列中仍然有消息，是会取出并执行的。
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
                }
                //省略部分代码...
                
                // 第二处，退出循环消息队列
                if (mQuitting) {
                    dispose();
                    return null;
                }
	      //省略部分代码...
        }
    }
```
从上述代码中我们可以明显的看见，当我们的消息队列中仍然有消息的时候，是会取出消息并返回的。不管mQuitting的值是否设置为true。那么当整个消息队列中的消息取完以后。才会走返回null(图上代码 第二处)。

### 当循环消息队列退出时，仍然发送消息
其实有很多小朋友们肯定会关注，"当我整个循环消息队列退出的时候，如果我仍然使用Handler发送消息，那么我的消息去那里了呢，是被抛弃了，还是有什么特殊处理呢?"，下面我们就带着这些疑惑来看看Handler机制中具体的处理。

我们都知道当调用Handler发送消息的时候，最终走的方法就是enqueueMessage（）方法，其中该方法内部又会走MessageQueue的enqueueMessage（Message msg, long when）方法。具体代码如下所示：
```
   private boolean enqueueMessage(MessageQueue queue, Message msg, long uptimeMillis) {
        msg.target = this;
        if (mAsynchronous) {
            msg.setAsynchronous(true);
        }
        return queue.enqueueMessage(msg, uptimeMillis);
    }
```
MessageQueue的enqueueMessage（Message msg, long when）方法，具体代码如下所示：
```
 boolean enqueueMessage(Message msg, long when) {
        if (msg.target == null) {
            throw new IllegalArgumentException("Message must have a target.");
        }
        if (msg.isInUse()) {
            throw new IllegalStateException(msg + " This message is already in use.");
        }

        synchronized (this) {
            if (mQuitting) {//如果当前消息循环队列已经退出，那么将该消息回收。
                IllegalStateException e = new IllegalStateException(
                        msg.target + " sending message to a Handler on a dead thread");
                Log.w(TAG, e.getMessage(), e);
                msg.recycle();//回收消息
                return false;//返回该消息没有加入消息队列的标志
            }
        //省略部分代码。。。
      }
```
通过观察MessageQueue的enqueueMessage（Message msg, long when）方法，我们能得出该方法内部会判断当前循环消息队里是否退出，如果已经结束，那么会将Handler发送的消息回收，同时会返回该消息没有加入消息队列的标志（return false)。

### 总结
- 循环消息队列的退出是通过调用Loooper的quitSafely（）或quit（）方法，两个退出方法都会导致消息队列中的消息回收。
-  quitSafely（）与quit()方法的区别是，quit()会直接回收消息队列中的消息，而quitSafely（）会根据当前的时间进行判断，如果消息的meesage.when比当前时间大，那么就会被回收，反之仍然被取出执行。
-  在整个循环消息队列退出的时候，如果在发送消息，那么该消息是会被会收的。