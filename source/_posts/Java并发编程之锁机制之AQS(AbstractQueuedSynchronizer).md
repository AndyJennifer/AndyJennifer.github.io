![爆炸.png](https://upload-images.jianshu.io/upload_images/2824145-0e04abe007aee52d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

>该文章属于《Java并发编程》系列文章，如果想了解更多，请点击《Java并发编程之总目录》

### 前言
在上篇文章[ 《Java并发编程之锁机制之Lock接口》](https://www.jianshu.com/p/6874d9b4f3d8)中，我们已经了解了，Java下整个Lock接口下实现的锁机制是通过`AQS(这里我们将AbstractQueuedSynchronizer 或AbstractQueuedLongSynchronizer统称为AQS)`与Condition来实现的。那下面我们就来具体了解AQS的内部细节与实现原理。

>PS:该篇文章会以`AbstractQueuedSynchronizer `来进行讲解，对AbstractQueuedLongSynchronizer有兴趣的小伙伴，可以自行查看相关资料。

### AQS简介
抽象队列同步器AbstractQueuedSynchronizer （以下都简称AQS），是用来构建锁或者其他同步组件的基础框架，它使用了一个int成员变量来表示同步状态，通过内置的FIFO(first-in-first-out)同步队列来控制获取共享资源的线程。

该类被设计为大多数同步组件的基类，这些同步组件都依赖于单个原子值（int）来控制同步状态，子类必须要定义获取获取同步与释放状态的方法，在AQS中提供了三种方法`getState()`、`setState(int newState)`及`compareAndSetState(int expect, int update)`来进行操作。同时子类应该为自定义同步组件的静态内部类，AQS自身没有实现任何同步接口，它仅仅是定义了若干同步状态获取和释放的方法来供自定义同步组件使用，同步器既可以支持独占式地获取同步状态，也可以支持共享式地获取同步状态，这样就可以方便实现不同类型的同步组件（ReentrantLock、ReentrantReadWriteLock和CountDownLatch等）。

### AQS类方法简介
AQS的设计是基于模板方法模式的，也就是说，使用者需要继承同步器并重写指定的方法，随后将同步器组合在自定义同步组件的实现中，并调用同步器提供的模板方法，而这些模板方法将会调用使用者重写的方法。

#### 修改同步状态方法
在子类实现自定义同步组件的时候，需要通过AQS提供的以下三个方法，来获取与释放同步状态。
- int getState() ：获取当前同步状态
- void setState(int newState) ：设置当前同步状态
- boolean compareAndSetState(int expect, int update) 使用CAS设置当前状态。

#### 子类中可以重写的方法
- boolean isHeldExclusively()：当前线程是否独占锁
- boolean tryAcquire(int arg)：独占式尝试获取同步状态，通过CAS操作设置同步状态，如果成功返回true，反之返回false
- boolean tryRelease(int arg)：独占式释放同步状态。
- int tryAcquireShared(int arg)：共享式的获取同步状态，返回大于等于0的值，表示获取成功，反之失败。
- boolean tryReleaseShared(int arg)：共享式释放同步状态。

#### 获取同步状态与释放同步状态方法
当我们实现自定义同步组件时，将会调用AQS对外提供的方法同步状态与释放的方法，当然这些方法内部会调用其子类的模板方法。这里将对外提供的方法分为了两类，具体如下所示：

- **独占式获取与释放同步状态**
1. void acquire(int arg)：独占式获取同步状态，如果当前线程获取同步状态成功，则返回，否则进入同步队列等待，该方法会调用tryAcquire(int arg)方法。
2. void acquireInterruptibly(int arg)：与 void acquire(int arg)基本逻辑相同，但是该方法`响应中断`,如果当前没有获取到同步状态，那么就会进入等待队列，如果当前线程被中断（`Thread().interrupt()`），那么该方法将会抛出InterruptedException。并返回
3. boolean tryAcquireNanos(int arg, long nanosTimeout)：`在acquireInterruptibly(int arg)的基础上`，增加了超时限制，如果当前线程没有获取到同步状态，那么将返回fase，反之返回true。
4. boolean release(int arg) ：独占式的释放同步状态


- **共享式获取与释放同步状态**
1. void acquireShared(int arg)：共享式的获取同步状态，如果当前线程未获取到同步状态，将会进入同步队列等待，`与独占式获取的主要区别是在同一时刻可以有多个线程获取到同步状态。`
2. void acquireSharedInterruptibly(int arg)：` 在acquireShared(int arg)的基本逻辑相同`，增加了响应中断。
3. boolean tryAcquireSharedNanos(int arg, long nanosTimeout)：`在acquireSharedInterruptibly的基础上`，增加了超时限制。
4. boolean releaseShared(int arg) ：共享式的释放同步状态

### AQS具体实现及内部原理
在了解了AQS中的针对不同方式获取与释放同步状态（`独占式与共享式`）与修改同步状态的方法后，现在我们来了解AQS中具体的实现及其内部原理。

#### AQS中FIFO队列
在上文中我们提到AQS中主要通过一个FIFO(first-in-first-out)来控制线程的同步。那么在实际程序中，AQS会将获取同步状态的线程构造成一个Node节点，并将该节点加入到队列中。如果该线程获取同步状态失败会阻塞该线程，当同步状态释放时，会把头节点中的线程唤醒，使其尝试获取同步状态。

##### Node节点结构
下面我们就通过实际代码来了解Node节点中存储的信息。Node节点具体实现如下：
```
    static final class Node {
        volatile int waitStatus;
        volatile Node prev;
        volatile Node next;
        volatile Thread thread;
        Node nextWaiter;
    }
```
Node节点是AQS中的`静态内部类`，下面分别对其中的属性（`注意其属性都用volatile 关键字进行修饰`)进行介绍。
- int waitStatus：等待状态主要包含以下状态
1. **SIGNAL = -1**：当前节点的线程如果释放了或取消了同步状态，将会将当前节点的状态标志位SINGAL，用于通知当前节点的下一节点，准备获取同步状态。
2. **CANCELLED = 1**：被中断或获取同步状态超时的线程将会被置为当前状态，且该状态下的线程不会再阻塞。
3. **CONDITION = -2**：当前节点在Condition中的等待队列上，（关于Condition会在下篇文章进行介绍），其他线程调用了Condition的singal()方法后，该节点会从等待队列转移到AQS的同步队列中，等待获取同步锁。
4. **PROPAGATE = -3**：与共享式获取同步状态有关，该状态标识的节点对应线程处于可运行的状态。
5. **0**：初始化状态。
- Node prev：当前节点在同步队列中的上一个节点。
- Node next：当前节点在同步队列中的下一个节点。
- Thread thread：当前转换为Node节点的线程。
-  Node nextWaiter：当前节点在Condition中等待队列上的下一个节点，（关于Condition会在下篇文章进行介绍）。

#### AQS同步队列具体实现结构
通过上文的描述我们大概了解了Node节点中存储的数据与信息，现在我们来看看整个AQS下同步队列的结构。具体如下图所示：
![aqs.png](https://upload-images.jianshu.io/upload_images/2824145-62c705bac15d1b1b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
在AQS中的同步队列中，分别有两个指针（你也可以叫做对象的引用），一个`head`指针指向队列中的头节点，一个`tail`指针指向队列中的尾节点。
##### AQS添加尾节点
 当一个线程成功获取了同步状态（或者锁），其他线程无法获取到同步状态，这个时候会将该线程构造成Node节点，并加入到同步队列中，而这个加入队列的过程必须要确保线程安全，所以在AQS中提供了一个基于CAS的设置尾节点的方法：`compareAndSetTail(Node expect,Nodeupdate)`，它需要传递当前线程“认为”的尾节点和当前节点，只有设置成功后，当前节点才正式与之前的尾节点建立关联。具体过程如下图所示：
 
   ![aqs_save_tail.png](https://upload-images.jianshu.io/upload_images/2824145-0c6db04f40503219.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
上图中，虚线部分为之前tail指向的节点。

##### AQS添加头节点
在AQS中的同步队列中，头节点是获取同步状态成功的节点，头节点的线程会在释放同步状态时，将会唤醒其下一个节点，而下一个节点会在获取同步状态成功时将自己设置为头节点，具体过程如下图所示：
![aqs_save_head.png](https://upload-images.jianshu.io/upload_images/2824145-7a0e99472ad06aad.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

上图中，`虚线部分为之前head指向的节点`。因为设置头节点是获取同步状态成功的线程来完成的，由于只有一个线程能够成功获取到同步状态，因此设置头节点的方法并不需要CAS来进行保证，只需要将原头节点的next指向断开就行了。

现在我们已经了解了AQS中同步队列的头节点与尾节点的设置过程。现在我们根据实际代码进行分析，因为涉及到不同状态对同步状态的获取(`独占式与共享式`)，所以下面会分别对这两种状态进行讲解。

#### 独占式同步状态获取与释放

##### 独占式同步状态获取
通过`acquire(int arg)`方法我们可以获取到同步状态，但是需要注意的是该方法并不会响应线程的中断与获取同步状态的超时机制。同时即使当前线程已经中断了，通过该方法放入的同步队列的Node节点（该线程构造的Node），也不会从同步队列中移除。具体代码如下所示：
```
  public final void acquire(int arg) {
        if (!tryAcquire(arg) &&
            acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
            selfInterrupt();
    }
```
在该方法中，主要通过`子类重写的方法tryAcquire(arg)`来获取同步状态，如果获取同步状态失败，则会将请求线程构造独占式Node节点（Node.EXCLUSIVE），同时将该线程加入同步队列的尾部（因为AQS中的队列是FIFO类型）。接着我们查看addWaiter(Node mode)方法具体细节：
```
 private Node addWaiter(Node mode) {
        Node node = new Node(Thread.currentThread(), mode);//将该线程构造成Node节点
  
        Node pred = tail;
        if (pred != null) {//尝试将尾指针 tail 指向当前线程构造的Node节点
            node.prev = pred;
            if (compareAndSetTail(pred, node)) {
              //如果成功，那么将尾指针之前指向的节点的next指向 当前线程构造的Node节点
                pred.next = node;
                return node;
            }
        }
        enq(node);//如果当前尾指针为null,则调用enq(final Node node)方法
        return node;
    }
```
在该方法中，主要分为两个步骤：
- 如果当前尾指针（tail)不为null，那么尝试将尾指针 tail 指向当前线程构造的Node节点，如果成功，那么将尾指针之前指向的节点的next指向当前线程构造的Node节点，并返回当前节点。
- 反之调用enq(final Node node)方法，将当前线程构造的节点加入同步队列中。

接下来我们继续查看enq(final Node node)方法。
```
  private Node enq(final Node node) {
        for (;;) {
            Node t = tail;
            if (t == null) {//如果当前尾指针为null,那么尝试将头指针 head指向当前线程构造的Node节点
                if (compareAndSetHead(new Node()))
                    tail = head;
            } else {//如果当前尾指针（tail)不为null，那么尝试将尾指针 tail 指向当前线程构造的Node节点
                node.prev = t;
                if (compareAndSetTail(t, node)) {
                    t.next = node;
                    return t;
                }
            }
        }
    }

```
在enq(final Node node)方法中，通过`死循环（你也可以叫做自旋）`的方式来保证节点的正确的添加。接下来，我们继续查看acquireQueued(final Node node, int arg)方法的处理。`该方法才是整个多线程竞争同步状态的关键，大家一定要注意看！！！`
```
    final boolean acquireQueued(final Node node, int arg) {
        boolean failed = true;
        try {
            boolean interrupted = false;
            for (;;) {
                final Node p = node.predecessor();//获取该节点的上一节点
                //如果上一节点是head锁指向的节点，且该节点获取同步状态成功
                if (p == head && tryAcquire(arg)) {
		            //设置head指向该节点，
                    setHead(node);
                    p.next = null; // 将上一节点的next指向断开
                    failed = false;
                    return interrupted;
                }
                //判断获取同步状态失败的线程是否需要阻塞
                if (shouldParkAfterFailedAcquire(p, node) &&
                    parkAndCheckInterrupt())//阻塞并判断当前线程是否已经中断了
                    interrupted = true;
            }
        } finally {
            if (failed)
            //如果线程中断了，那么就将该线程从同步队列中移除，同时唤醒下一节点
                cancelAcquire(node);
        }
    }
```

在该方法中主要分为三个步骤:

- 通过`死循环（你也可以叫做自旋）`的方式来获取同步状态，如果当前节点的`上一节点是head指向的节点`且`该节点获取同步状态成功`，那么会设置head指向该节点 ，同时将上一节点的next指向断开。
![aqs_self_rotate.png](https://upload-images.jianshu.io/upload_images/2824145-577539f984ab6e45.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![aqs_get_head.png](https://upload-images.jianshu.io/upload_images/2824145-7d9fbf3bebdddfda.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

- 如果当前节点的上一节点不是head指向的节点，或者获取当前节点同步状态失败，那么会先调用`shouldParkAfterFailedAcquire(Node pred, Node node)`方法来判断是需要否阻塞当前线程，如果该方法返回`true`，则调用`parkAndCheckInterrupt()`方法来阻塞线程。如果该方法返回`false`,那么该方法内部会把当前节点的上一节点的状态修改为Node.SINGAL。
- 在finally语句块中，判断当前线程是否已经中断。如果中断，则通过那么`cancelAcquire(Node node)`方法将该线程（对应的Node节点）从同步队列中移除，同时唤醒下一节点。


下面我们接着来看`shouldParkAfterFailedAcquire(Node pred, Node node)`方法，看看具体的阻塞具体逻辑，代码如下所示：
```
private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
        int ws = pred.waitStatus;
        if (ws == Node.SIGNAL)
	        //上一节点已经设置状态请求释放信号，因此当前节点可以安全地阻塞
            return true;
        if (ws > 0) {
	        //上一节点，已经被中断或者超时，那么接跳过所有状态为Node.CANCELLED
            do {
                node.prev = pred = pred.prev;
            } while (pred.waitStatus > 0);
            pred.next = node;
        } else {
	        //其他状态，则调用cas操作设置状态为Node.SINGAL
            compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
        }
        return false;
    }
```
在该方法中会获取上一节点的状态`(waitStatus)`，然后进行下面的三个步骤的判断。
- 如果上一节点状态为Node.SIGNAL，那么会阻塞接下来的线程`(函数 return true)`。
- 如果上一节点的状态大于0（从上文描述的waitStatus所有状态中，我们可以得知只有Node.CANCELLED大于0）那么会跳过整个同步列表中所有状态为Node.CANCELLED的Node节点。`(函数 return false)`。
- 如果上一节点是其他状态，则调用CAS操作设置其状态为Node.SINGAL。`(函数 return false)`。

##### 阻塞实现
当`shouldParkAfterFailedAcquire(Node pred, Node node)`方法返回`true`时，接着会调用parkAndCheckInterrupt（）方法来阻塞当前线程。该方法的返回值为当前线程是否中断。
```
 private final boolean parkAndCheckInterrupt() {
        LockSupport.park(this);
        return Thread.interrupted();
    }
```
在该方法中，主要阻塞线程的方法是通过LockSupport（在后面的文章中会具体介绍）的park来阻塞当前线程。
##### 从同步队列中移除，同时唤醒下一节点
通过对独占式获取同步状态的理解，我们知道 acquireQueued(final Node node, int arg)方法中最终会执行`finally`语句块中的代码，来判断当前线程是否已经中断。如果中断，则通过那么`cancelAcquire(Node node)方法将该线程从同步队列中移除`。那么接下来我们来看看该方法的具体实现。具体代码如下：
```
   private void cancelAcquire(Node node) {
        //如果当前节点已经不存在直接返回
        if (node == null)
            return;
		//（1）将该节点对应的线程置为null
        node.thread = null;

        //（2）跳过当前节点之前已经取消的节点
        Node pred = node.prev;
        while (pred.waitStatus > 0)
            node.prev = pred = pred.prev;

		//获取在（2）操作之后，节点的下一个节点
        Node predNext = pred.next;

	    //（3）将当前中断的线程对应节点状态设置为CANCELLED
        node.waitStatus = Node.CANCELLED;

        //（4）如果当前中断的节点是尾节点，那么则将尾节点重新指向
        if (node == tail && compareAndSetTail(node, pred)) {
            compareAndSetNext(pred, predNext, null);
        } else {
            //（5）如果中断的节点的上一个节点的状态，为SINGAL或者即将为SINGAL，
            //那么将该当前中断节点移除
            int ws;
            if (pred != head &&
                ((ws = pred.waitStatus) == Node.SIGNAL ||
                 (ws <= 0 && compareAndSetWaitStatus(pred, ws, Node.SIGNAL))) &&
                pred.thread != null) {
                Node next = node.next;
                if (next != null && next.waitStatus <= 0)
                    compareAndSetNext(pred, predNext, next);
            } else {
                unparkSuccessor(node);//（6）将该节点移除，同时唤醒下一个节点
            }

            node.next = node; // help GC
        }
    }
```
观察上诉代码，我们可以知道该方法干了以下这几件事
- （1）将中断线程对应的节点对应的线程置为null
- （2）跳过当前节点之前已经取消的节点（我们已经知道在Node.waitStatus的枚举中，只有CANCELLED 大于0 ）
![跳过已经取消的节点.png](https://upload-images.jianshu.io/upload_images/2824145-15ff4717041cc3bb.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

- （3）将当前中断的线程对应节点状态设置为CANCELLED
- （4）在（2）的前提下，如果当前中断的节点`是尾节点`，那么通过CAS操作将尾节点指向（2）操作后的的节点。

![重新设置尾节点Tail.png](https://upload-images.jianshu.io/upload_images/2824145-aa83713dcb855990.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

- （5）如果当前中断节点`不是尾节点`,且当前中断的节点的上一个节点的状态，为SINGAL或者即将为SINGAL，那么将该当前中断节点移除。
- （6）如果（5）条件不满足，那么调用`unparkSuccessor(Node node)`方法将该节点移除，同时唤醒下一个节点。具体代码如下：
```
    private void unparkSuccessor(Node node) {
         //重置该节点为初始状态
        int ws = node.waitStatus;
        if (ws < 0)
            compareAndSetWaitStatus(node, ws, 0);
        //获取中断节点的下一节点    
        Node s = node.next;
        //判断下一节点的状态，如果为Node.CANCELED状态
        if (s == null || s.waitStatus > 0) {
            s = null;
            //则通过尾节点向前遍历，获取最近的waitStatus<=0的节点
            for (Node t = tail; t != null && t != node; t = t.prev)
                if (t.waitStatus <= 0)
                    s = t;
        }
        //如果该节点不会null，则唤醒该节点中的线程。
        if (s != null)
            LockSupport.unpark(s.thread);
    }
```
这里为了方便大家理解，我还是将图补充了出来，（图片有可能不是很清晰，建议大家点击浏览大图），
![aqs.png](https://upload-images.jianshu.io/upload_images/2824145-95d89529fae7df5a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
整体来说，unparkSuccessor（Node node）方法主要是获取中断节点后的可用节点（Node.waitStatus<=0),然后将该节点对应的线程唤醒。

##### 独占式同步状态释放
当线程获取同步状态成功并执行相应逻辑后，需要释放同步状态，使得后继线程节点能够继续获取同步状态，通过调用AQS的relase(int arg)方法，可以释放同步状态。具体代码如下：
```
 public final boolean release(int arg) {
        if (tryRelease(arg)) {
            Node h = head;
            if (h != null && h.waitStatus != 0)
                unparkSuccessor(h);
            return true;
        }
        return false;
    }
```
在该方法中，会调用模板方法`tryRelease(int  arg)`，也就是说同步状态的释放逻辑，是需要用户来自己定义的。当`tryRelease(int  arg)`方法返回true后，如果当前头节点不为null且头节点waitStatus!=0，接着会调用`unparkSuccessor(Node node)`方法来唤醒下一节点`（使其尝试获取同步状态）`。关于unparkSuccessor(Node node)方法，上文已经分析过了，这里就不再进行描述了。

#### 共享式同步状态获取与释放
共享式获取与独占式获取最主要的区别在于`同一时刻是否能有多个线程同时获取到同步状态`。以文件的读写为例，如果一个程序在对文件进行`读操作`，那么这一时刻对于文件的`写操作均会被阻塞`。而`其他读操作能够同时进行`。如果对文件进行`写操作`，那么这一时刻`其他的读写操作都会被阻塞`，写操作要求对资源的独占式访问，而读操作可以是共享访问的。

##### 共享式同步状态获取
在了解了共享式同步状态获取与独占式获取同步状态的区别后，现在我们来看一看共享式获取的相关方法。在AQS中通过 acquireShared(int arg)方法来实现的。具体代码如下：
```
  public final void acquireShared(int arg) {
        if (tryAcquireShared(arg) < 0)
            doAcquireShared(arg);
    }
```
在该方法内部会调用模板方法`tryAcquireShared(int arg)`,同独占式获取获取同步同步状态一样，也是需要用户自定义的。当`tryAcquireShared(int arg)`方法返回值小于0时，表示没有获取到同步状态，则调用`doAcquireShared(int arg) `方法获取同步状态。反之，已经获取同步状态成功，则不进行任何的操作。关于`doAcquireShared(int arg)`方法具体实现如下所示：

```
   private void doAcquireShared(int arg) {
	    //（1）添加共享式节点在AQS中FIFO队列中
        final Node node = addWaiter(Node.SHARED);
        boolean failed = true;
        try {
            boolean interrupted = false;
            //(2)自旋获取同步状态
            for (;;) {
                final Node p = node.predecessor();
                if (p == head) {
                    int r = tryAcquireShared(arg);
                    if (r >= 0) {
	                    //当获取同步状态成功后，设置head指针
                        setHeadAndPropagate(node, r);
                        p.next = null; // help GC
                        if (interrupted)
                            selfInterrupt();
                        failed = false;
                        return;
                    }
                }
                //(3)判断线程是否需要阻塞
                if (shouldParkAfterFailedAcquire(p, node) &&
                    parkAndCheckInterrupt())
                    interrupted = true;
            }
        } finally {
	        //(4)如果线程已经中断，则唤醒下一节点
            if (failed)
                cancelAcquire(node);
        }
    }
```
整体来看，共享式获取的逻辑与独占式获取的逻辑几乎一样，还是以下几个步骤：
- (1)添加共享式节点在AQS中FIFO队列中,这里需要注意节点的构造为 `addWaiter(Node.SHARED)`,其中 Node.SHARED为Node类中的静态常量`(static final Node SHARED = new Node())`，且通过addWaiter（Node.SHARED)方法构造的`节点状态为初始状态，也就是waitStatus= 0`。


- (2)自旋获取同步状态，如果当前节点的上一节点为head节点，其获取同步状态成功，那么将调用` setHeadAndPropagate(node, r);`，重新设置head指向当前节点。`同时重新设置该节点状态waitStutas = Node.PROPAGATE(共享状态)`,然后直接退出doAcquireShared(int arg)方法。具体情况如下图所示：

![共享式自旋判断.png](https://upload-images.jianshu.io/upload_images/2824145-3794d9b6c9f71b05.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

- (3)如果不满足条件（2），那么会判断当前节点的上一节点不是head指向的节点，或者获取当前节点同步状态失败，那么会先调用`shouldParkAfterFailedAcquire(Node pred, Node node)`方法来判断是需要否阻塞当前线程，如果该方法返回`true`，则调用`parkAndCheckInterrupt()`方法来阻塞线程。如果该方法返回`false`,那么该方法内部会把当前节点的上一节点的状态修改为Node.SINGAL。具体情况如下图所示：

![共享式线程阻塞.png](https://upload-images.jianshu.io/upload_images/2824145-380e27c34be98551.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
- (4)如果线程已经中断，则唤醒下一节点

前面我们提到了，共享式与独占式获取同步状态的主要不同在于`其设置head指针的方式不同`，下面我们就来看看共享式设置head指针的方法`setHeadAndPropagate(Node node, int propagate)`。具体代码如下：
```
    private void setHeadAndPropagate(Node node, int propagate) {
	    //(1)设置head 指针，指向该节点
        Node h = head; // Record old head for check below
        setHead(node);
        
        //(2)判断是否执行doReleaseShared();
        if (propagate > 0 || h == null || h.waitStatus < 0 ||
            (h = head) == null || h.waitStatus < 0) {
            Node s = node.next;
            //如果当前节点的下一节点是共享式获取同步状态节点，则调用doReleaseShared（）方法
            if (s == null || s.isShared())
                doReleaseShared();
        }
    }
```
在setHeadAndPropagate(Node node, int propagate)方法中有两个参数。
`第一个参数node`是当前共享式获取同步状态的线程节点。
`第二个参数propagate`（中文意思，繁殖、传播）是共享式获取同步状态线程节点的个数。

其主要逻辑步骤分为以下两个步骤：
- (1)设置head 指针，指向该节点。`从中我们可以看出在共享式获取中，Head节点总是指向最进获取成功的线程节点！！！`
- (2)判断是否执行doReleaseShared()，从代码中我们可以得出，主要通过该条件`if (s == null || s.isShared())`，其中 s为当前节点的下一节点（也就是说同一时刻有可能会有多个线程同时访问）。当该条件为true时，会调用doReleaseShared()方法。关于怎么判断下一节点是否是否共享式线程节点，具体逻辑如下：
```
   //在共享式访问中，当前节点为SHARED类型
   final Node node = addWaiter(Node.SHARED);
   
   //在调用addWaiter 内部会调用Node构造方法，其中会将nextWaiter设置为Node.SHARED。
   Node(Thread thread, Node mode) {     // Used by addWaiter
            this.nextWaiter = mode;
            this.thread = thread;
        }
   //SHARED为Node类静态类    
   final boolean isShared() {
            return nextWaiter == SHARED;
        }
        
```
下面我们继续查看`doReleaseShared（）`方法的具体实现，具体代码如下所示：
```
 private void doReleaseShared() {
        for (;;) {
            Node h = head;
            if (h != null && h != tail) {
                int ws = h.waitStatus;
                if (ws == Node.SIGNAL) {
	                //(1)从上图中，我们可以得知在共享式的同步队列中，如果存在堵塞节点，
	                //那么head所指向的节点状态肯定为Node.SINGAL,
	                //通过CAS操作将head所指向的节点状态设置为初始状态，如果成功就唤醒head下一个阻塞的线程
                    if (!compareAndSetWaitStatus(h, Node.SIGNAL, 0))
                        continue;            // loop to recheck cases
                    unparkSuccessor(h);//唤醒下一节点线程，上文分析过该方法，这里就不在讲了
                }
				//(2)表示该节点线程已经获取共享状态成功,则通过CAS操作将该线程节点状态设置为Node.PROPAGATE
				//从上图中，我们可以得知在共享式的同步队列中，
                else if (ws == 0 &&
                         !compareAndSetWaitStatus(h, 0, Node.PROPAGATE))
                    continue;                // loop on failed CAS
            }
            if (h == head)                   //如果head指针发生改变一直循环，否则跳出循环
                break;
        }
    }
```
从代码中我们可以看出该方法主要分为两个步骤：
- (1)从上图中，我们可以得知在共享式的同步队列中，如果存在堵塞节点，`那么head所指向的节点状态肯定为Node.SINGAL`，通过CAS操作将head所指向的节点状态设置为初始状态，如果成功就唤醒head下一个阻塞的线程节点，反之继续循环。
- (2)如果(1)条件不满足，那么说明该节点已经获取成功的获取同步状态，那么通过CAS操作将该线程节点的状态设置为`waitStatus =  Node.PROPAGATE`,如果CAS操作失败，就一直循环。


##### 共享式同步状态释放
当线程获取同步状态成功并执行相应逻辑后，需要释放同步状态，使得后继线程节点能够继续获取同步状态，通过调用AQS的releaseShared(int arg)方法，可以释放同步状态。具体代码如下：
```
 public final boolean releaseShared(int arg) {
        if (tryReleaseShared(arg)) {
            doReleaseShared();
            return true;
        }
        return false;
    }
```

#### 独占式与共享式超时获取同步状态
因为独占式与共享式超时获取同步状态，与其本身的非超时获取同步状态逻辑几乎一样。所以下面就以独占式超时获取同步状态的相应逻辑进行讲解。

在独占式超时获取同步状态中，会调用`tryAcquireNanos(int arg, long nanosTimeout)`方法，其中具体nanosTimeout参数为你传入的超时时间（`单位纳秒`），具体代码如下所示：
```
public final boolean tryAcquireNanos(int arg, long nanosTimeout)
            throws InterruptedException {
        if (Thread.interrupted())
            throw new InterruptedException();
        return tryAcquire(arg) ||
            doAcquireNanos(arg, nanosTimeout);
    }
```
观察代码，我们可以得知如果当前线程已经中断，会`直接抛出InterruptedException`，如果当前线程能够获取同步状态（ 调用tryAcquire(arg)），那么就会直接返回，如果当前线程获取同步状态失败，则调用`doAcquireNanos(int arg, long nanosTimeout)`方法来超时获取同步状态。那下面我们接着来看该方法具体代码实现，代码如下图所示：
```
private boolean doAcquireNanos(int arg, long nanosTimeout)
            throws InterruptedException {
        if (nanosTimeout <= 0L)
            return false;
        //(1)计算超时等待的结束时间
        final long deadline = System.nanoTime() + nanosTimeout;
        final Node node = addWaiter(Node.EXCLUSIVE);
        boolean failed = true;
        try {
            for (;;) {
                final Node p = node.predecessor();
                //(2)如果获取同步状态成功，直接返回
                if (p == head && tryAcquire(arg)) {
                    setHead(node);
                    p.next = null; // help GC
                    failed = false;
                    return true;
                }
                //如果获取同步状态失败，计算的剩下的时间
                nanosTimeout = deadline - System.nanoTime();
                //(3)如果超时直接退出
                if (nanosTimeout <= 0L)
                    return false;
                //(4)如果没有超时，且nanosTimeout大于spinForTimeoutThreshold（1000纳秒）时，
                //则让线程等待nanosTimeout （剩下的时间，单位：纳秒。）
                if (shouldParkAfterFailedAcquire(p, node) &&
                    nanosTimeout > spinForTimeoutThreshold)
                    LockSupport.parkNanos(this, nanosTimeout);
                //(5)如果当前线程被中断，直接抛出异常    
                if (Thread.interrupted())
                    throw new InterruptedException();
            }
        } finally {
            if (failed)
                cancelAcquire(node);
        }
    }
```
整个方法为以下几个步骤：
- (1)在线程获取同步状态之前，先计算出超时等待的结束时间。（单位精确到纳秒）
- (2)通过自旋操作获取同步状态，如果成功，则直接返回
- (3)如果获取同步失败，则计算剩下的时间。如果已经超时了就直接退出。
- (4)如果没有超时，则判断当前剩余时间`nanosTimeout是否大于`spinForTimeoutThreshold（1000纳秒），如果大于，则通过 ` LockSupport.parkNanos(this, nanosTimeout)`方法让线程等待相应时间。（该方法会在根据传入的`nanosTimeout`时间，等待相应时间后返回。），`如果nanosTimeout小于等于`spinForTimeoutThreshold时，将不会使该线程进行超时等待，而是进入快速的自旋过程。原因在于，非常短的超时等待无法做到十分精确，如果这时再进行超时等待，相反会让nanosTimeout的超时从整体上表现得反而不精确。因此，在超时非常短的场景下，线程会进入无条件的快速自旋。
- (5)在没有走（4）步骤的情况下，表示当前线程已经被中断了，则`直接抛出InterruptedException`。


### 最后
到现在我们基本了解了整个AQS的内部结构与其独占式与共享式获取同步状态的实现，但是其中涉及到的线程的阻塞、等待、唤醒（与LockSupport工具类相关）相关知识点我们都没有具体介绍，后续的文章会对`LockSupport工具`以及后期关于锁相关的等待/通知模式相关的`Condition接口`进行介绍。希望大家继续保持着学习的动力~~。

### 总结
- 整个AQS是基于其内部的FIFO队列实现同步控制。请求的线程会封装为Node节点。
- AQS分为整体分为独占式与共享式获取同步状态。其支持线程的中断，与超时获取。

