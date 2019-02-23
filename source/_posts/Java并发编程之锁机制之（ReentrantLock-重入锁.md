---
title: Java并发编程之锁机制之（ReentrantLock)重入锁
date: 2019-02-23 21:29:12
categories:
- Java并发相关
tags: 
- Java
---


![小兔子.jpg](https://upload-images.jianshu.io/upload_images/2824145-b990e9d2f1578e83.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

>最近在忙公司的项目，现在终于有时间来写博客啦~开心开心

### 前言
通过前面的文章，我们已经了解了`AQS(AbstractQueuedSynchronizer)`内部的实现与基本原理。现在我们来了解一下，Java中为我们提供的Lock机制下的锁实现--`ReentrantLock（重入锁）`，阅读该篇文章之前，希望你已阅读以下文章。
- [ Java并发编程之锁机制之Lock接口](https://www.jianshu.com/p/6874d9b4f3d8)
- [ Java并发编程之锁机制之AQS(AbstractQueuedSynchronizer)](https://www.jianshu.com/p/a372528f47a3)
- [Java并发编程之锁机制之LockSupport工具](https://www.jianshu.com/p/d0e84096d108)
- [Java并发编程之锁机制之Condition接口](https://www.jianshu.com/p/a22855b8820a)


### ReentrantLock基本介绍
`ReentrantLock`是一种`可重入`的`互斥锁`，它具有与使用`synchronized `方法和语句所访问的隐式监视器锁相同的一些基本行为和语义，但功能更强大。

`ReentrantLock` 将由最近成功获得锁，并且还没有释放该锁的线程所拥有。当锁没有被另一个线程所拥有时，调用 lock 的线程将成功获取该锁并返回。如果当前线程已经拥有该锁，此方法将立即返回。可以使用` isHeldByCurrentThread()` 和 `getHoldCount() `方法来检查此情况是否发生。

此类的构造方法接受一个可选的`公平`参数。当设置为 true 时(也是当前`ReentrantLock为公平锁的情况`)，在多个线程的争用下，这些锁倾向于将访问权授予等待时间最长的线程。否则此锁将无法保证任何特定访问顺序。与采用默认设置（使用不公平锁）相比，使用公平锁的程序在许多线程访问时表现为很低的总体吞吐量（即速度很慢，常常极其慢），但是在获得锁和保证锁分配的均衡性时差异较小。不过要注意的是，公平锁不能保证线程调度的公平性。因此，使用公平锁的众多线程中的一员可能获得多倍的成功机会，这种情况发生在其他活动线程没有被处理并且目前并未持有锁时。还要注意的是，未定时的 tryLock 方法并没有使用公平设置。因为即使其他线程正在等待，只要该锁是可用的，此方法就可以获得成功。

### ReentrantLock 类基本结构
通过上文的简单介绍后，我相信很多小伙伴还是一脸懵逼，只知道上文我们提到了`ReentrantLock`与`synchronized `相比有相同的语义，同时其内部分为了`公平锁`与`非公平锁`两种锁的类型，且该锁是支持`重进入`的。那么为了方便大家理解这些知识点，我们先从其类的基本结构讲起。具体类结构如下图所示：

![ReentrantLock.png](https://upload-images.jianshu.io/upload_images/2824145-435e7270822a79cd.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

从上图中我们可以看出，在` ReentrantLock`类中，定义了三个静态内部类，**Sync**、**FairSync（公平锁）**、**NonfairSync（非公平锁**）。其中`Sync`继承了`AQS（AbstractQueuedSynchronizer）`，而`FairSync`与`NonfairSync`又分别继承了`Sync`。关于`ReentrantLock `基本类结构如下所示：
```
public class ReentrantLock implements Lock, java.io.Serializable {
    private final Sync sync;

	//默认无参构造函数，默认为非公平锁
    public ReentrantLock() {
        sync = new NonfairSync();
    }
	//带参数的构造函数，用户自己来决定是公平锁还是非公平锁
    public ReentrantLock(boolean fair) {
        sync = fair ? new FairSync() : new NonfairSync();
    }
    //抽象基类继承AQS，公平锁与非公平锁继承该类，并分别实现其lock()方法
    abstract static class Sync extends AbstractQueuedSynchronizer {
        abstract void lock();
        //省略部分代码..
    }
    
	//非公平锁实现
    static final class NonfairSync extends Sync {...}
    
    //公平锁实现
    static final class FairSync extends Sync {....}
   
    //锁实现，根据具体子类实现调用
    public void lock() {
        sync.lock();
    }
	//响应中断的获取锁
    public void lockInterruptibly() throws InterruptedException {
        sync.acquireInterruptibly(1);
    }
	//尝试获取锁，默认采用非公平锁方法实现
    public boolean tryLock() {
        return sync.nonfairTryAcquire(1);
    }
	//超时获取锁
    public boolean tryLock(long timeout, TimeUnit unit)
            throws InterruptedException {
        return sync.tryAcquireNanos(1, unit.toNanos(timeout));
    }
	//释放锁
    public void unlock() {
        sync.release(1);
    }
    //创建锁条件（从Condetion来理解，就是创建等待队列）
    public Condition newCondition() {
        return sync.newCondition();
    }
    //省略部分代码....
}
```
>这里为了方便大家理解`ReentrantLock`类的整体结构，我省略了一些代码及重新排列了一些代码的顺序。

从代码中我们可以看出。整个`ReentrantLock`类的实现其实都是交给了其内部`FairSync`与`NonfairSync`两个类。在`ReentrantLock`类中有两个构造函数，其中不带参数的构造函数中默认使用的`NonfairSync（非公平锁）`。另一个带参数的构造函数，用户自己来决定是`FairSync（公平锁）`还是非公平锁。

### 重进入实现
在上文中，我们提到了`ReentrantLock`是支持重进入的，那什么是重进入呢？`重进入是指任意线程在获取到锁之后能够再次获取该锁，而不会被锁阻塞`。那接下来我们看看这个例子，如下所示：
```
class ReentrantLockDemo {
    private static final ReentrantLock lock = new ReentrantLock();
    
    public static void main(String[] args) {
        Thread thread = new Thread(new Runnable() {
            @Override
            public void run() {
                methodA();
            }
        });
        thread.start();
    }
    
    public static void methodA() {
        lock.lock();
        try {
            System.out.println("我已经进入methodA方法了");
            methodB();//方法A中继续调用方法B
        } finally {
            lock.unlock();
        }
    }

    public static void methodB() {
        lock.lock();
        try {
            System.out.println("我已经进入methodB方法了");
        } finally {
            lock.unlock();
        }
    }
}
//输出结果
我已经进入methodA方法了
我已经进入methodB方法了
```
在上述代码中我们声明了一个线程调用methodA()方法。同时在该方法内部我们又调用了methodB()方法。从实际的代码运行结果来看，当前线程进入方法A之后。在方法B中再次调用`lock.lock();`时，该线程并没有被阻塞。也就是说`ReentrantLock`是支持重进入的。那下面我们就一起来看看其内部的实现原理。

因为`ReenTrantLock`将具体实现交给了`NonfairSync（非公平锁）`与`FairSync(公平锁)`。同时又因为上述提到的两个锁，关于重进入的实现又非常相似。所以这里将采用`NonfairSync（非公平锁）`的重进入的实现，来进行分析。希望读者朋友们阅读到这里的时候需要注意，不是我懒哦，是真的很相似哦。

好了下面我们来看代码。关于NonfairSync代码如下所示：
```
static final class NonfairSync extends Sync {
        final void lock() {
            if (compareAndSetState(0, 1))////直接获取同步状态成功，那么就不再走尝试获取锁的过程
                setExclusiveOwnerThread(Thread.currentThread());
            else
                acquire(1);
        }

        protected final boolean tryAcquire(int acquires) {
            return nonfairTryAcquire(acquires);
        }
    }
```
当我们调用lock()方法时，通过CAS操作将AQS中的state的状态设置为1，如果成功，那么表示获取同步状态成功。那么会接着调用`setExclusiveOwnerThread(Thread thread)`方法来设置当前占有锁的线程。如果失败，则调用`acquire(int arg)`方法来获取同步状态（该方法是属于AQS中的独占式获取同步状态的方法，对该方法不熟悉的小伙伴，建议阅读[ Java并发编程之锁机制之AQS(AbstractQueuedSynchronizer)](https://www.jianshu.com/p/a372528f47a3)）。而该方法内部会调用`tryAcquire(int acquires)`来尝试获取同步状态。通过观察，我们发现最终会调用`Sync`类中的`nonfairTryAcquire(int acquires)`方法。我们继续跟踪。
```
    final boolean nonfairTryAcquire(int acquires) {
		    //获取当前线程
            final Thread current = Thread.currentThread();
            int c = getState();
            //(1)判断同步状态，如果未设置，则设置同步状态
            if (c == 0) {
                if (compareAndSetState(0, acquires)) {
                    setExclusiveOwnerThread(current);
                    return true;
                }
            }
            //(2)如果当前线程已经获取了同步状态，则增加同步状态的值。
            else if (current == getExclusiveOwnerThread()) {
                int nextc = c + acquires;
                if (nextc < 0) // overflow
                    throw new Error("Maximum lock count exceeded");
                setState(nextc);
                return true;
            }
            return false;
        }
```
从代码上来看，该方法主要走两个步骤，具体如下所示：
- （1）先判断同步状态， 如果未曾设置，则设置同步状态，并设置当前占有锁的线程。
- （2）判断是否是同一线程，如果当前线程已经获取了同步状态（也就是获取了锁），那么增加同步状态的值。

也就是说，如果同一个锁获取了锁N(`N为正整数`)次，那么对应的同步状态`（state)`也就等于N。那么接下来的问题来了，`如果当前线程重复N次获取了锁，那么该线程是否需要释放锁N次呢？`答案当然是必须的。当我们调用`ReenTrantLock`的unlock()方法来释放同步状态（也就是释放锁）时，内部会调用` sync.release(1);`。最终会调用`Sync`类的`tryRelease(int releases)`方法。具体代码如下所示：
```
  protected final boolean tryRelease(int releases) {
            int c = getState() - releases;
            if (Thread.currentThread() != getExclusiveOwnerThread())
                throw new IllegalMonitorStateException();
            boolean free = false;
            if (c == 0) {
                free = true;
                setExclusiveOwnerThread(null);
            }
            setState(c);
            return free;
        }
```
从代码中，我们可以知道，每调用一次`unlock()`方法会将当前同步状态减一。也就是说如果当前线程获取了锁N次，那么获取锁的相应线程也需要调用`unlock()`方法N次。这也是为什么我们在之前的重入锁例子中，为什么`methodB`方法中也要释放锁的原因。


### 非公平锁
在ReentrantLock中有着`非公平锁`与`公平锁`的概念，这里我先简单的介绍一下`公平`这两个字的含义。**这里的公平是指线程获取锁的顺序。也就是说锁的获取顺序是按照当前线程请求的绝对时间顺序，当然前提条件下是该线程获取锁成功**。

那么接下来，我们来分析在ReentrantLock中的非公平锁的具体实现。

>这里需要大家具备`AQS(AbstractQueuedSynchronizer)`类的相关知识。如果大家不熟悉这块的知识。建议大家阅读 [ Java并发编程之锁机制之AQS(AbstractQueuedSynchronizer)](https://www.jianshu.com/p/a372528f47a3)。
```
    static final class NonfairSync extends Sync {
        private static final long serialVersionUID = 7316153563782823691L;

        final void lock() {
            if (compareAndSetState(0, 1))//直接获取同步状态成功，那么就不再走尝试获取锁的过程
                setExclusiveOwnerThread(Thread.currentThread());
            else
                acquire(1);
        }
        //省略部分代码...
    }
```
当在ReentrantLock在`非公平锁的模式下`，去调用lock（）方法。那么接下来最终会走`AQS(AbstractQueuedSynchronizer)`下的`acquire(int arg)（独占式的获取同步状态）`，也就是如下代码：
```
  public final void acquire(int arg) {
        if (!tryAcquire(arg) &&
            acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
            selfInterrupt();
    }
```
那么结合之前我们所讲的AQS知识，在多个线程在`独占式`请求共享状态下（也就是请求锁）的情况下，在AQS中的同步队列中的线程节点情况如下图所示：
![aqs同步队列中线程节点的情况.png](https://upload-images.jianshu.io/upload_images/2824145-b899bb6db2704676.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

那么我们试想一种情况，当Nod1中的线程执行完相应任务后，释放锁后。这个时候本来该唤醒当前线程节点的`下一个节点`,也就是`Node2中的线程`。这个时候突然另一线程突然来获取线程（这里我们用节点`Node5`来表示）。具体情况如下图所示：

![突然线程请求锁的情况.png](https://upload-images.jianshu.io/upload_images/2824145-30ff585cc5da6453.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

那么根据AQS中独占式获取同步状态的逻辑。只要`Node5对应的线程获取同步状态成功`。那么就会出现下面的这种情况，具体情况如下图所示：

![线程抢占后最终的情况.png](https://upload-images.jianshu.io/upload_images/2824145-a071107319267a4a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

从上图中我们可以看出，由于Node5对象的线程抢占了获取同步状态(获取锁)的机会，本身应该被唤醒的`Node2`线程节点。因为获取同步状态失败。所以只有再次的陷入阻塞。那么综上。我们可以知道。`非公平锁获取同步状态（获取锁）时不会考虑同步队列中中等待的问题。会直接尝试获取锁。也就是会存在后申请，但是会先获得同步状态（获取锁）的情况。`

### 公平锁
理解了非公平锁，再来理解公平锁就非常简单了。下面我们来看一下公平锁与非公平锁的加锁的源码：
![非公平锁与公平锁源码区别.png](https://upload-images.jianshu.io/upload_images/2824145-b6351764588542ab.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
从源码我们可以看出，非公平锁与公平锁之间的代码唯一区别就是多了一个判断条件`!hasQueuedPredecessors()(图中红框所示)`。那我们查看其源码（该代码在AQS中，强烈建议阅读[ Java并发编程之锁机制之AQS(AbstractQueuedSynchronizer)](https://www.jianshu.com/p/a372528f47a3)）
```
    public final boolean hasQueuedPredecessors() {
        Node t = tail;
        Node h = head;
        Node s;
        return h != t &&
            ((s = h.next) == null || s.thread != Thread.currentThread());
    }
```
代码理解理解起来非常简单，就是判断当前当前head节点的next节点是不是当前请求同步状态（请求锁）的线程。也就是语句
` ((s = h.next) == null || s.thread != Thread.currentThread()`。那么接下来结合AQS中的同步队列我们可以得到下图：

![公平锁抢占情况.png](https://upload-images.jianshu.io/upload_images/2824145-28ccbe3b0cb6c3e3.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

那么综上我们可以得出，公平锁保证了线程请求的同步状态（请求锁）的顺序。不会出现另一个线程抢占的情况。

### 最后
该文章参考以下图书，站在巨人的肩膀上。可以看得更远。
- 《Java并发编程的艺术》
### 推荐阅读
- [Java并发编程之锁机制之引导篇](https://www.jianshu.com/p/4ead70bdab56)
- [《Java并发编程之锁机制之AQS》](https://www.jianshu.com/p/a372528f47a3)
- [《Java并发编程之锁机制之LockSupport工具》](https://www.jianshu.com/p/d0e84096d108)