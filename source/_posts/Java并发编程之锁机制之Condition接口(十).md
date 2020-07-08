---
title: Java并发编程之锁机制之Condition接口(十)
date: 2019-02-23 21:40:23
categories:
- Java并发相关
tags: 
- 并发
---

{% asset_img book.jpg book %}

### 前言

在前面的文章中，我曾提到过，整个Lock接口下实现的锁机制中`AQS(AbstractQueuedSynchronizer，下文都称之为AQS)`与`Condition`才是真正的实现者。也就说`Condition`在整个同步组件的基础框架中也起着非常重要的作用，既然它如此重要与犀利，那么现在我们就一起去了解其内部的实际原理与具体逻辑。

>在阅读该文章之前，我由衷的建议先阅读{% post_link Java并发编程之锁机制之AQS(AbstractQueuedSynchronizer)(八) %}与{% post_link Java并发编程之锁机制之LockSupport工具(九) %}这两篇文章。因为整个Condtion的内部机制与逻辑都离不开以上两篇文章提到的知识点。

### Condition接口方法介绍

在正式介绍Condtion之前，我们可以先了解其中声明的方法。具体方法声明，如下表所示：

{% asset_img condition方法.png condition方法 %}

从该表中，我们可以看出其内部定义了`等待（以await开头系列方法）`与`通知（以singal开头的系列方法)`两种类型的方法，类似于Object对象的`wait()`与`notify()/NotifyAll()`方法来对线程的阻塞与唤醒。

### ConditionObject介绍

在实际使用中，Condition接口实现类是`AQS`中的内部类`ConditionObject`。在其内部维护了一个`FIFO(first in first out)`的队列（这里我们称之为`等待队列`，你也可以叫做阻塞队列，看每个人的理解），通过与`AQS中的同步队列`配合使用，来控制获取共享资源的线程。

#### 等待队列

等待队列是`ConditionObjec`中内部的一个`FIFO(first in first out)`的队列，在队列中的每个节点都包含了一个线程引用，且该线程就是在ConditionObject对象上阻塞的线程。需要注意的是，在等待队列中的节点是复用了`AQS`中`Node类`的定义。换句话说，在`AQS`中维护的同步队列与`ConditionObjec`中维护的等待队列中的节点类型都是`AQS.Node`类型。（关于`AbstractQueuedSynchronizer.Node`类的介绍，大家可以参看{% post_link Java并发编程之锁机制之AQS(AbstractQueuedSynchronizer)(八) %}文章中的描述）。

在ConditionObject类中也分别定义了`firstWaiter`与`lastWaiter`两个指针，分别指向等待队列中头部与尾部。当实际线程调用其以`await开头`的系列方法后。会将该线程构造为Node节点。添加等待队列中的尾部。关于等待队列的基本结构如下图所示：

{% asset_img condition内部结构.png condition内部结构 %}

对于等待队列中节点添加的方式也很简单，将`上一尾节点的nextWaiter指向新添加的节点`，同时使`lastWaiter`指向新添加的节点。

#### 同步队列与等待队列的对应关系

上文提到了整个Lock锁机制需要`AQS中的同步队列`与`ConditionObject的等待队列`配合使用，其对应关系如下图所示：
{% asset_img 同步队列与等待队列的关系.png 同步队列与等待队列的关系 %}

在Lock锁机制下，可以`拥有一个同步队列和多个等待队列`，与我们传统的Object监视器模型上，一个对象拥有一个同步队列和等待队列不同。lock中的锁可以伴有多个条件。

#### Condition的基本使用

为了大家能够更好的理解同步队列与等待队列的关系。下面通过一个有界队列`BoundedBuffer`来了解Condition的使用方式，该类是一个特殊的队列，当队列为空时，队列的获取操作将会阻塞当前"拿"线程，直到队列中有新增的元素，当队列已满时，队列的放入操作将会阻塞"放入"线程，直到队列中出现空位。具体代码如下所示：

```java
class BoundedBuffer {

    final Lock lock = new ReentrantLock();
    final Condition notFull = lock.newCondition();
    final Condition notEmpty = lock.newCondition();

    final Object[] items = new Object[100];

    //依次为，放入的角标、拿的角标、数组中放入的对象总数
    int putptr, takeptr, count;

    /**
     * 添加一个元素
     * （1）如果当前数组已满，则把当前"放入"线程，加入到"放入"等待队列中，并阻塞当前线程
     * （2）如果当前数组未满，则将x元素放入数组中，唤醒"拿"线程中的等待线程。
     */
    public void put(Object x) throws InterruptedException {
        lock.lock();
        try {
            while (count == items.length)//如果已满，则阻塞当前"放入"线程
                notFull.await();
            items[putptr] = x;
            if (++putptr == items.length) putptr = 0;
            ++count;
            notEmpty.signal();//唤醒"拿"线程
        } finally {
            lock.unlock();
        }
    }

    /**
     * 拿一个元素
     * （1）如果当前数组已空，则把当前"拿"线程，加入到"拿"等待队列中，并阻塞当前线程
     * （2）如果当前数组不为空，则把唤醒"放入"等待队列中的线程。
     */
    public Object take() throws InterruptedException {
        lock.lock();
        try {
            while (count == 0)//如果为空，则阻塞当前"拿"线程
                notEmpty.await();
            Object x = items[takeptr];
            if (++takeptr == items.length) takeptr = 0;
            --count;
            notFull.signal();//唤醒"放入"线程
            return x;
        } finally {
            lock.unlock();
        }
    }
}
```

从代码中我们可以看出，在该类中我们创建了两个等待队列`notFull`与`notEmpty`。这两个等待队列的作用分别是，当请数组已满时，`notFull`用于存储阻塞的"放入"线程，`notEmpty`用于存储阻塞的"拿"线程。需要注意的是`获取一个Condition必须通过Lock的newCondition()方法`。关于`ReentrantLock`，在后续的文章中，我们会进行介绍。

### 阻塞实现 await()

在了解了ConditionObject的内部基本结构和与AQS中内部的同步队列的对应关系后，现在我们来看看其阻塞实现。调用ConditionObject的`await()`方法（或者以`await开头`的方法），会使当前线程进入等待队列，并释放同步状态，需要注意的是当该方法返回时，当前线程一定获取了同步状态（具体原因是当通过`signal()等系列方法`，线程才会从`await（）`方法返回，而唤醒该线程后会加入同步队列）。这里我们以`awati()`方法为例，具体代码如下所示：

```java
  public final void await() throws InterruptedException {
            //如果当前线程已经中断，直接抛出异常  
            if (Thread.interrupted())
                throw new InterruptedException();
            //(1)将当前线程加入等待队列
            Node node = addConditionWaiter();
            //(2)释放同步状态（也就是释放锁），同时将线程节点从同步队列中移除，并唤醒同步队列中的下一节点
            int savedState = fullyRelease(node);
            int interruptMode = 0;
            //(3)判断当前线程节点是否还在同步队列中，如果不在则阻塞线程
            while (!isOnSyncQueue(node)) {
                LockSupport.park(this);
                if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
                    break;
            }
            //(4)当线程被唤醒后，重新在同步队列中与其他线程竞争获取同步状态
            if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
                interruptMode = REINTERRUPT;
            if (node.nextWaiter != null) // clean up if cancelled
                unlinkCancelledWaiters();
            if (interruptMode != 0)
                reportInterruptAfterWait(interruptMode);
        }
```

从代码整体来看，整个方法分为以下四个步骤：

- （1）通过 `addConditionWaiter()`方法将线程节点加入到等待队列中。
- （2）通过`fullyRelease(Node node)`方法释放同步状态（也就是释放锁），同时将线程节点从`同步队列`中移除，并唤醒`同步队列中的下一节点`。
- （3）通过`isOnSyncQueue(Node node)`方法判断当前线程节点是否在`同步队列`中，如果不在，则通过`LockSupport.park(this);`阻塞当前线程。
- （4）当线程被唤醒后，调用`acquireQueued(node, savedState)`方法，重新在同步队列中与其他线程竞争获取同步状态

因为每个步骤涉及到的逻辑都稍微有一点复杂，这里为了方便大家理解，分别对以上四个步骤涉及到的方法分别进行介绍。

#### addConditionWaiter()方法

该方法主要将同步队列中的需要阻塞的线程节点加入到等待队列中，关于`addConditionWaiter()`方法具体代码如下所示：

```java
    private Node addConditionWaiter() {
            Node t = lastWaiter;
            // (1)如果当前尾节点中中对应的线程已经中断，
            //则移除等待队列中所有的已经中断或已经释放同步状态的线程节点
            if (t != null && t.waitStatus != Node.CONDITION) {
                unlinkCancelledWaiters();
                t = lastWaiter;
            }
            //(2)构建等待队列中的节点
            Node node = new Node(Node.CONDITION);

            //(3)将该线程节点添加到队列中，同时构建firstWaiter与lastWaiter的指向
            if (t == null)
                firstWaiter = node;
            else
                t.nextWaiter = node;
            lastWaiter = node;
            return node;
        }
```

该方法的逻辑也比较简单，分为以下三个步骤：

- （1）获取等待队列中的尾节点，如果当前尾节点已经中断，那么则通过`unlinkCancelledWaiters()`方法移除等待队列中所有的`已经中断`或`已经释放同步状态（也就是释放锁）`的线程节点
- （2）构建等待队列中的节点，注意，是通过`New`的形式，那么就说明与同步队列中的线程节点不是同一个。（对Node状态枚举不清楚的小伙伴，可以参看{% post_link Java并发编程之锁机制之AQS(AbstractQueuedSynchronizer)(八) %}文章下的Node状态枚举介绍）。
- （3）将该线程节点添加到等待队列中去，同时构建firstWaiter与lastWaiter的指向，可以看出等待队列总是以`FIFO(first in first out )`的形式添加线程节点。

##### unlinkCancelledWaiters()方法

因为在`addConditionWaiter()`方法的步骤（1）中，调用了`unlinkCancelledWaiters`移除了所有的`已经中断`的线程节点，那我们看一个该方法的实现。如下所示：

```java
  private void unlinkCancelledWaiters() {
            //获取等待队列中的头节点
            Node t = firstWaiter;
            Node trail = null;
            //遍历等待队列，将已经中断的线程节点从等待队列中移除。
            while (t != null) {
                Node next = t.nextWaiter;
                if (t.waitStatus != Node.CONDITION) {
                    t.nextWaiter = null;
                    if (trail == null)
                        firstWaiter = next;
                    else
                        trail.nextWaiter = next;
                    if (next == null)//重新定义lastWaiter的指向
                        lastWaiter = trail;
                }
                else
                    trail = t;
                t = next;
            }
        }
```

该方法具体流程如下图所示：
{% asset_img condition.png condition %}

#### fullyRelease(Node node)

在将阻塞线程加入到等待队列后，会将该线程节点从同步队列中移除，释放同步状态（也就是释放锁），并唤醒同步队列中的下一节点。具体代码如下所示：

```java
  final int fullyRelease(Node node) {
        try {
            int savedState = getState();
            if (release(savedState))
                return savedState;
            throw new IllegalMonitorStateException();
        } catch (Throwable t) {
            node.waitStatus = Node.CANCELLED;
            throw t;
        }
    }
```

 `release(int arg)`方法会释放当前线程的同步状态， 并唤醒`同步队列中`的下一线程节点，使其尝试获取同步状态，因为该方法已经在{% post_link Java并发编程之锁机制之AQS(AbstractQueuedSynchronizer)(八) %}文章下的`unparkSuccessorNode node)`方法的下分析过了，所以这里就不再进行分析了。希望大家参考上面提到的文章进行理解。

#### isOnSyncQueue(Node node)

该方法主要用于判断当前线程节点是否在同步队列中。具体代码如下所示：

```java
  final boolean isOnSyncQueue(Node node) {
        //判断当前节点 waitStatus ==Node.CONDITION或者当前节点上一节点为空,则不在同步队列中
        if (node.waitStatus == Node.CONDITION || node.prev == null)
            return false;
        //如果当前节点拥有下一个节点，则在同步队列中。
        if (node.next != null) // If has successor, it must be on queue
            return true;
        //如果以上条件都不满足，则遍历同步队列。检查是否在同步队列中。
        return findNodeFromTail(node);
    }
```

如果你还记得AQS中的同步队列，那么你应该知道同步队列中的Node节点才会使用其内部的`pre`与`next`字段，那么在同步队列中因为只使用了`nextWaiter`字段，所以我们就能很简单的通过这两个字段是否为`==null`，来判断是否在同步队列中。当然也有可能有一种特殊情况。有可能需要阻塞的线程节点还没有加入到同步队列中，那么这个时候我们需要遍历同步队列来判断是该线程节点是否已存在。具体代码如下所示：

```java
 private boolean findNodeFromTail(Node node) {
        for (Node p = tail;;) {
            if (p == node)
                return true;
            if (p == null)
                return false;
            p = p.prev;
        }
    }
```

这里之所以使用同步队列`tail（尾节点）`来遍历，如果`node.netx!=null`，那么就说明当前线程已经在同步队列中。那么我们需要处理的情况肯定是针对`node.next==null`的情况。所以需要从尾节点开始遍历。

#### acquireQueued(final Node node, int arg)

当线程被唤醒后（具体原因是当通过`signal()等系列方法`，线程才会从`await（）`方法返回）会调用该方法将该线程节点加入到同步队列中。该方法我在{% post_link Java并发编程之锁机制之AQS(AbstractQueuedSynchronizer)(八) %}中具体描述过了。这里就不在进行过多的解析。

#### 阻塞流程

在理解了整个阻塞的流程后，现在我们来归纳总结一下，整个阻塞的流程。具体流程如下图所示：

{% asset_img 阻塞流程.png 阻塞流程 %}

- （1）将该线程节点从同步队列中移除，并释放其同步状态。
- （2）构造新的阻塞节点，加入到等待队列中。

### 唤醒实现 signal()

当需要唤醒线程时，会调用ConditionObject中的`singal开头的系列方法`，该系列方法会唤醒等待队列中的`首个`线程节点，在唤醒该节点之前，会先讲该节点移动到`同步队列`中。这里我们以`singal()`方法为例进行讲解，具体代码如下：

```java
  public final void signal() {
            //（1）判断当前线程是否获取到了同步状态（也就是锁）
            if (!isHeldExclusively())
                throw new IllegalMonitorStateException();
            Node first = firstWaiter;
            //（2）获取等待队列中的首节点，然后将其移动到同步队列，然后再唤醒该线程节点
            if (first != null)
                doSignal(first);
        }
```

该方法主要逻辑分为以下两个步骤：

- （1）通过`isHeldExclusively()`方法，判断当前线程是否获取到了同步状态（也就是锁）。
- （2）通过`doSignal(Node first)`方法，获取等待队列中的首节点，然后将其移动到同步队列，然后再唤醒该线程节点。

下面我们会分别对上面涉及到的两个方法进行描述。

#### isHeldExclusively()方法

`isHeldExclusively()`方法是AQS中的方法，默认交给其子类实现，主要用于判断当前调用`singal()`方法的线程，是否在同步队列中，且已经获取了同步状态。具体代码如下所示：

```java
    protected boolean isHeldExclusively() {
        throw new UnsupportedOperationException();
    }
```

#### doSignal(Node first)方法

那我们继续跟踪`doSignal(Node first)`方法，具体方法如下：

```java
     private void doSignal(Node first) {
            do {
                //（1）将等待队列中的首节点从等待队列中移除，并重新制定firstWaiter的指向
                if ( (firstWaiter = first.nextWaiter) == null)
                    lastWaiter = null;
                first.nextWaiter = null;
            } while (!transferForSignal(first) &&
            //（2）将等待队列中的首节点，加入同步队列中，并重新唤醒该节点
                     (first = firstWaiter) != null);
        }
```

该方法也很简单，分为两个步骤：

- （1）将等待队列中的首节点从等待队列中移除，并设置firstWaiter的指向为首节点的下一个节点。 为了方便大家理解该步骤所描述的逻辑，这里画了具体的图，具体情况如下图所示：
{% asset_img 移除首节点.png 移除首节点 %}
- （2）通过 `transferForSignal(Node node)`方法，将等待队列中的首节点，加入到同步队列中去，然后重新唤醒该线程节点。

##### transferForSignal(Node node)方法

因为步骤（2）中`transferForSignal(Node node)`方法较为复杂，所以会对该方法进行详细的讲解。具体代码如下所示：

```java
    final boolean transferForSignal(Node node) {

        //（1）将该线程节点的状态设置为初始状态，如果失败则表示当前线程已经中断了
        if (!node.compareAndSetWaitStatus(Node.CONDITION, 0))
            return false;
        //（2）将该节点放入同步队列中，
        Node p = enq(node);
        int ws = p.waitStatus;
        //（3）获取当前节点的状态并判断，尝试将该线程节点状态设置为Singal，如果失败则唤醒线程
        if (ws > 0 || !p.compareAndSetWaitStatus(ws, Node.SIGNAL))
            LockSupport.unpark(node.thread);
        return true;
    }
```

该方法分为三个步骤：

- （1）将该线程节点的状态设置为初始状态，如果失败则表示当前线程已经中断了，直接返回。
- （2）通过`enq(Node node)`方法，将该线程节点放入`同步队列`中。
- （3）当将该线程节点放入同步队列后，获取当前节点的状态并判断，如果该节点的`waitStatus>0`或者通过`compareAndSetWaitStatus(ws, Node.SIGNAL)`将该节点的状态设置为Singal，如果失败则通过`LockSupport.unpark(node.thread)`唤醒线程。

上述步骤中，着重讲`enq(Node node)`方法，关于`LockSupport.unPark(Thread thread)`方法的理解，大家可以阅读{% post_link Java并发编程之锁机制之LockSupport工具(九) %}。下面我们就来分析`enq(Node node)`方法。具体代码如下所示：

```java
   private Node enq(Node node) {
        for (;;) {
            //(1)获取同步队列的尾节点
            Node oldTail = tail;
            //(2)如果尾节点不为空，则将该线程节点加入到同步队列中
            if (oldTail != null) {
                //将当前节点的prev指向尾节点
                U.putObject(node, Node.PREV, oldTail);
                //将同步队列中的tail指针，指向当前节点
                if (compareAndSetTail(oldTail, node)) {
                    oldTail.next = node;
                    return oldTail;
                }
            } else {
                //(3)如果当前同步队列为空，则构造同步队列
                initializeSyncQueue();
            }
        }
    }
```

观察该方法，我们发现该方法通过`死循环(当然你也可以叫做自旋)`的方式来添加该节点到同步队列中去。该方法分为以下步骤：

- （1）获取同步队列的尾节点
- （2）如果尾节点不为空，则将该线程节点加入到同步队列中
- （3）如果当前同步队列为空，则通过`initializeSyncQueue();`构造同步队列。

这里对`Node enq(Node node)`中的步骤（2）补充一个知识点。我们来看一下调用`U.putObject(node, Node.PREV, oldTail);`语句，内部是如何将当前的节点的prev指向尾节点的。在`AQS（AbstractQueuedSynchronizer）`中的Node类中有如下静态变量和语句。这里我省略了一下不重要的代码。具体代码如下所示：

```java
private static final sun.misc.Unsafe U = sun.misc.Unsafe.getUnsafe();
//省略部分代码
static final long PREV;
    static {
         try {
            //省略部分代码
            PREV = U.objectFieldOffset
               (Node.class.getDeclaredField("prev"));
              } catch (ReflectiveOperationException e) {
               throw new Error(e);
            }
        }
    }
```

其中`Node.class.getDeclaredField("prev")`语句很好理解，就是获取Node类中`pre`字段，如果有则返回相应Field字段，反之抛出NoSuchFieldException异常。关于Unfase中的`objectFieldOffset(Field f)`方法，我曾经在{% post_link Java并发编程之锁机制之LockSupport工具(九) %}描述过类似的情况。这里我简单的再解释一遍。该方法用于获取某个字段相对 Java对象的“起始地址”的偏移量，也就是说每个字段在类对应的内存中存储是有`“角标”`的，那么也就是说我们现在的`PREV`静态变量就代表着Node中`prev`字段在内存中的“角标”。

当获取到"角标"后，我们再通过`U.putObject(node, Node.PREV, oldTail);`该方法第一个参数是操作对象，第二个参数是操作的内存“角标”，第三个参数是期望值。那么最后，也就完成了将当前节点的prev字段指向同步队列的尾节点。

当理解了该知识点后，剩下的`将同步队列中的tail指针，指向当前节点`与`如果当前同步队列为空，则构造同步队列`这两个操作就非常好理解了。由于篇幅的限制，在这里我就不在进行描述了。希望读者朋友们，能阅读源代码，举一反三。关于这两个方法的代码如下所示：

```java
   private static final sun.misc.Unsafe U = sun.misc.Unsafe.getUnsafe();
    private static final long STATE;
    private static final long HEAD;
    private static final long TAIL;
    static {
        try {
            STATE = U.objectFieldOffset
                (AbstractQueuedSynchronizer.class.getDeclaredField("state"));
            HEAD = U.objectFieldOffset
                (AbstractQueuedSynchronizer.class.getDeclaredField("head"));
            TAIL = U.objectFieldOffset
                (AbstractQueuedSynchronizer.class.getDeclaredField("tail"));
        } catch (ReflectiveOperationException e) {
            throw new Error(e);
        }
        Class<?> ensureLoaded = LockSupport.class;
    }

    private final void initializeSyncQueue() {
        Node h;
        if (U.compareAndSwapObject(this, HEAD, null, (h = new Node())))
            tail = h;
    }

    private final boolean compareAndSetTail(Node expect, Node update) {
        return U.compareAndSwapObject(this, TAIL, expect, update);
    }
```

#### 唤醒流程

在理解了唤醒的具体逻辑后，现在来总结一下，唤醒的具体流程。具体如下图所示：

{% asset_img 唤醒流程.png 唤醒流程 %}

- 将等待队列中的`头`节点线程，移动到同步队列中。
- 当移动到同步队列中后。唤醒该线程。是该线程参与同步状态的竞争。

整体流程其实不算太复杂，大家只需要注意，`当我们将等待队列中的线程节点加入到同步队列之后，才会唤醒线程`。

### 最后

该文章参考以下图书，站在巨人的肩膀上。可以看得更远。

- 《Java并发编程的艺术》

### 推荐阅读

- {% post_link Java并发编程之锁机制之引导篇(六)%}
- {% post_link Java并发编程之锁机制之AQS(AbstractQueuedSynchronizer)(八) %}
- {% post_link Java并发编程之锁机制之LockSupport工具(九) %}
