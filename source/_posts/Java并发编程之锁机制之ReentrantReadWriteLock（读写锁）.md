![蓝天.jpg](https://upload-images.jianshu.io/upload_images/2824145-12d950ee54698c34.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 前言
在前面的文章中，我们讲到了ReentrantLock(重入锁)，接下来我们讲`ReentrantReadWriteLock（读写锁）`，该锁具备重入锁的`可重入性`、`可中断获取锁`等特征，但是与`ReentrantLock`不一样的是，在`ReentrantReadWriteLock`中，维护了一对锁，一个`读锁`一个`写锁`，而读写锁在同一时刻允许多个`读`线程访问。但是在写线程访问时，所有的读线程和其他的写线程均被阻塞。在阅读本片文章之前，希望你已阅读过以下几篇文章：
- [ Java并发编程之锁机制之Lock接口](https://www.jianshu.com/p/6874d9b4f3d8)
- [ Java并发编程之锁机制之AQS(AbstractQueuedSynchronizer)](https://www.jianshu.com/p/a372528f47a3)
- [Java并发编程之锁机制之LockSupport工具](https://www.jianshu.com/p/d0e84096d108)
- [Java并发编程之锁机制之Condition接口](https://www.jianshu.com/p/a22855b8820a)
- [Java并发编程之锁机制之（ReentrantLock)重入锁](https://www.jianshu.com/p/1068960ecd64)

### 基本结构
在具体了解`ReentrantReadWriteLock`之前，我们先看一下其整体结构，具体结构如下图所示：
![ReentrantReadWriteLock.png](https://upload-images.jianshu.io/upload_images/2824145-fbf5a4a890fd66f1.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

从整体图上来看，`ReentrantReadWriteLock`实现了`ReadWriteLock`接口，其中在`ReentrantReadWriteLock`中分别声明了以下几个静态内部类：
- `WriteLock`与`ReadLock`（维护的一对读写锁）：单从类名我们可以看出这两个类的作用，就是控制读写线程的锁
- `Sync`及其子类`NofairSync`与`FairSync`：如果你阅读过 [Java并发编程之锁机制之（ReentrantLock)重入锁](https://www.jianshu.com/p/1068960ecd64)中公平锁与非公平锁的介绍，那么我们也可以猜测出`ReentrantReadWriteLock（读写锁）`是支持公平锁与非公平锁的。
- `ThreadLoclHoldCounter`及`HoldCounter`：涉及到锁的重进入，在下文中我们会具体进行描述。


### 基本使用
在使用某些种类的`Collection`时，可以使用`ReentrantReadWriteLock` 来提高并发性。通常，在预期`Collection` 很大，且`读取线程`访问它的次数`多于写入线程`的情况下，且所承担的操作开销高于同步开销时，这很值得一试。例如，以下是一个使用 TreeMap（我们假设预期它很大，并且能被同时访问） 的字典类。
```
class RWDictionary {
    private final Map<String, Data> m = new TreeMap<String, Data>();
    private final ReentrantReadWriteLock rwl = new ReentrantReadWriteLock();
    private final Lock r = rwl.readLock();//获取读锁
    private final Lock w = rwl.writeLock();//获取写锁
    
	//读取Map中的对应key的数据
    public Data get(String key) {
        r.lock();
        try { return m.get(key); }
        finally { r.unlock(); }
    }
    //读取Map中所有的key
    public String[] allKeys() {
        r.lock();
        try { return m.keySet().toArray(); }
        finally { r.unlock(); }
    }
    //往Map中写数据
    public Data put(String key, Data value) {
        w.lock();
        try { return m.put(key, value); }
        finally { w.unlock(); }
    }
    //清空数据
    public void clear() {
        w.lock();
        try { m.clear(); }
        finally { w.unlock(); }
    }
 }
```
在上述例子中，我们分别对TreeMap中的读取操作进行了加锁的操作。当我们调用`get(String key)`方法，去获取`TreeMap`中对应key值的数据时，需要先获取读锁。那么其他线程对于`写锁的获取将会被阻塞`，而对获取读锁的线程不会阻塞。同理，当我们调用`put(String key, Data value)`方法，去更新数据时，我们需要获取写锁。那么其他线程对于写锁与读锁的获取都将会被阻塞。只有当获取写锁的线程释放了锁之后。其他读写操作才能进行。

这里可能会有小伙伴会有疑问，`为什么当获取写锁成功后，会阻塞其他的读写操作？`，这里其实是为了保证数据可见性。如果不阻塞其他读写操作，假如读操作优先与写操作，那么在数据更新之前，读操作获取的数据与写操作更新后的数据就会产生不一致的情况。

>需要注意的是：`ReentrantReadWriteLock`最多支持 `65535` 个递归写入锁和`65535`个读取锁。试图超出这些限制将导致锁方法抛出 Error。具体原因会在下文进行描述。

### 实现原理
到现在为止，我们已经基本了解了`ReentrantReadWriteLock`的基本结构与基本使用。我相信大家肯定对其内部原理感到好奇，下面我会带着大家一起去了解其内部实现。这里我会对整体的一个原理进行分析，内部更深的细节会在下文进行描述。因为我觉得只有理解整体原理后，再去理解其中的细节。那么对整个`ReentrantReadWriteLock（读写锁）`的学习来说，要容易一点。

####  整体原理
在前文中，我们介绍了`ReentrantReadWriteLock`的基本使用，我们发现整个读写锁对线程的控制是交给了`WriteLock`与`ReadLock`。当我们调用读写锁的`lock()`方法去获取相应的锁时，我们会执行以下代码：
```
 public void lock() { sync.acquireShared(1);}
```
也就是会调用`sync.acquireShared(1)`，而`sync`又是什么呢？从其构造函数中我们也可以看出:
```
 public ReentrantReadWriteLock(boolean fair) {
        sync = fair ? new FairSync() : new NonfairSync();
        readerLock = new ReadLock(this);
        writerLock = new WriteLock(this);
    }
```
其中关于`FairSync`与`NonfairSync`的声明如下所示：
```
//同步队列
abstract static class Sync extends AbstractQueuedSynchronizer {省略部分代码...}
//非公平锁
static final class NonfairSync extends Sync{省略部分代码...}
//公平锁
static final class FairSync extends Sync {省略部分代码...}
```
这里我们又看到了我们熟悉的`AQS`，也就是说`WriteLock`与`ReadLock`这两个锁，其实是通过AQS中的同步队列来对线程的进行控制的。那么结合我们之前的AQS的知识，我们可以得到下图：
>（如果你对AQS不熟，那么你可以阅读该篇文章----->[ Java并发编程之锁机制之AQS(AbstractQueuedSynchronizer)](https://www.jianshu.com/p/a372528f47a3)).
>
![读写锁状态关系图.png](https://upload-images.jianshu.io/upload_images/2824145-1a0de5f2e62b3ed7.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
这里我省略了`为什么维护的是同一个同步队列的原因`，这个问题留给大家。

#### 读写状态设计
虽然现在我们已经知道了，`WriteLock`与`ReadLock`这两个锁维护了`同一个同步队列`，但是我相信大家都会有个疑问，同步队列中只有一个`int`类型的`state`变量来表示当前的同步状态。那么其内部是怎么将两个读写状态分开，并且达到控制线程的目的的呢？

在`ReentrantReadWriteLock`中的同步队列，其实是将同步状态分为了两个部分，其中`高16位`表示`读状态`，`低16位`表示`写状态`，具体情况如下图所示：

![读写锁状态划分.png](https://upload-images.jianshu.io/upload_images/2824145-d6fb58d42d2a34ab.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

在上图中，我们能得知，读写状态能表示的最大值为`65535(排除负数)`，也就是说允许锁重进入的次数为65535次。

接下来 我们单看高16位，这里表示当前线程已经获取了写锁，且重进入了七次。同样的这里如果我们也只但看低16位，那么就表示当前线程获取了读锁，且重进入了七次。`这里大家需要注意的是，在实际的情况中，读状态与写状态是不能被不同线程同时赋值的。因为根据ReentrantReadWriteLock的设计来说，读写操作线程是互斥的。上图中这样表示，只是为了帮助大家理解同步状态的划分`。

到现在为止我们已经知道同步状态的划分，那接下来又有新的问题了。`如何快速的区分及获取读写状态呢？`其实也非常简单。
- 读状态：想要获取读状态，只需要将当前同步变量`无符号右移16位`
- 写状态：我们只需要将当前同步状态（这里用S表示）进行这样的操作`S&0x0000FFFF)`，也就是`S&(1<<16-1)`。

也就是如下图所示（可能图片不是很清楚，建议在pc端上观看）：

![读写锁状态原理.png](https://upload-images.jianshu.io/upload_images/2824145-114bde4afbe4afed.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 细节分析
在了解了`ReentrantReadWriteLock`的整体原理及读写状态的划分后，我们再来理解其内部的读写线程控制就容易的多了，下面的文章中，我会对读锁与写锁的获取分别进行讨论。

#### 读锁的获取
因为当调用`ReentrantReadWriteLock`中的`ReadLock`的lock()方法时，最终会走`Sync`中的`tryAcquireShared(int unused)`方法，来判断能否获取写锁。那现在我们就来看看该方法的具体实现。具体代码如下所示：
```
   protected final int tryAcquireShared(int unused) {
            Thread current = Thread.currentThread();
            int c = getState();
            //（1）判断当前是否有写锁，有直接返回
            if (exclusiveCount(c) != 0 &&
                getExclusiveOwnerThread() != current)
                return -1;
               
            int r = sharedCount(c);
             //(2)获取当前读锁的状态，判断是否小于最大值，
             //同时根据公平锁，还是非公平锁的模式，判断当前线程是否需要阻塞，
            if (!readerShouldBlock() &&
                r < MAX_COUNT &&
	                compareAndSetState(c, c + SHARED_UNIT)) {
                //（3)如果是不要阻塞，且写状态小于最大值，则设置当前线程重进入的次数
                if (r == 0) {
			        //如果当前读状态为0，则设置当前读线程为，当前线程为第一个读线程。
                    firstReader = current;
                    firstReaderHoldCount = 1;
                } else if (firstReader == current) {
		            //计算第一个读线程，重进入的次数
                    firstReaderHoldCount++;
                } else {
	                //通过ThreadLocl获取读线程中进入的锁
                    HoldCounter rh = cachedHoldCounter;
                    if (rh == null || rh.tid != getThreadId(current))
                        cachedHoldCounter = rh = readHolds.get();
                    else if (rh.count == 0)
                        readHolds.set(rh);
                    rh.count++;
                }
                return 1;//获取共享同步状态成功
            }
            //(4)当获取读状态失败后，继续尝试获取读锁，
            return fullTryAcquireShared(current);
        }
```
- （1）根据当前的同步状态，判断是否存在写锁，且当前拥有写锁的线程不是当前线程，那么直接返回`-1`，需要注意的是如果该方法返回值为负数，那么会将**该请求线程加入到AQS的同步队列**中。（对该方法不是很熟的小伙伴，建议查看 [ Java并发编程之锁机制之AQS(AbstractQueuedSynchronizer)](https://www.jianshu.com/p/a372528f47a3)
- （2）获取当前读锁的状态，判断是否小于最大值，同时根据公平锁，还是非公平锁的模式，判断当前线程是否需要阻塞
- （3）如果条件（2）满足，则设置分别`第一个读取线程重进入的次数`及`后续线程`重进入的次数
- （4）如果条件（2）不满足，在再次尝试获取读锁。

在读锁的获取中，涉及到的方法较为复杂，所以下面会对每个步骤中涉及到的方法，进行介绍。

##### 步骤（1）中如何判断是否有写锁？
在读锁的获取中的步骤（1）中，代码中会调用`exclusiveCount(int c)`方法来判当前是否存在写锁。而该方法是属于`Sync`中的方法，具体代码如下所示：
```
    abstract static class Sync extends AbstractQueuedSynchronizer {

        static final int SHARED_SHIFT   = 16;
        static final int SHARED_UNIT    = (1 << SHARED_SHIFT);
        static final int MAX_COUNT      = (1 << SHARED_SHIFT) - 1;//最大状态数为2的16次方-1
        static final int EXCLUSIVE_MASK = (1 << SHARED_SHIFT) - 1;

        /*返回当前的读状态*/
        static int sharedCount(int c)    { return c >>> SHARED_SHIFT; }
        /*返回当前的写状态 */
        static int exclusiveCount(int c) { return c & EXCLUSIVE_MASK; }
       }
```
从代码中我们可以看出，只是简单的执行了` c & EXCLUSIVE_MASK`，也就是`S&0x0000FFFF`，结合我们上文中我们所讲的读写状态的区分，我相信`exclusiveCount(int c)`与`sharedCount(int c)`方法是不难理解的。

##### 步骤（2）中如何判断是公平锁与非公平锁。
在步骤（2）中，我们发现调用了`readerShouldBlock()`方法，而该方法是`Sync`类中的抽象方法。在ReentrantReadWriteLock类中，公平锁与非公平锁进行了相应的实现，具体代码如下图所示：
```
	//公平锁
    static final class FairSync extends Sync {
        private static final long serialVersionUID = -2274990926593161451L;
        final boolean writerShouldBlock(){return hasQueuedPredecessors();}
        final boolean readerShouldBlock(){return hasQueuedPredecessors();
        }
    }
    //非公平锁
    static final class NonfairSync extends Sync {
        final boolean writerShouldBlock() { return false;}
        final boolean readerShouldBlock() {return apparentlyFirstQueuedIsExclusive();}
    }
```
这里就不再对公平锁与非公平锁进行分析了。在文章 [Java并发编程之锁机制之（ReentrantLock)重入锁](https://www.jianshu.com/p/1068960ecd64)中已经对这个知识点进行了分析。有兴趣的小伙伴可以参考该文章。

##### 步骤（3）中为毛要记录第一个获取写锁的线程？线程的重进入是如何实现的？
在ReentrantReadWriteLock类中分别定义了`Thread firstReader`与`int firstReaderHoldCount`变量来记录当前`第一个`获取写锁的线程以及其重进入的次数。官方的给的解释是`便于跟踪与记录线程且这种记录是非常廉价的`。也就是说，之所以单独定义一个变量来记录第一个获取获取写锁的线程，是为了在众多的读线程中区分线程，也是为了以后的调试与跟踪。

当我们解决了第一个问题后，现在我们来解决第二个问题。这里我就不在对第一个线程如何记录重进入次数进行分析了。我们直接看其他读线程的重进入次数设置。这里因为篇幅的限制，我就直接讲原理，其他线程的重进入的次数判断是通过`ThreadLocal`来实现的。通过在每个线程中的内存空间保存`HodlerCount`类（用于记录当前线程获取锁的次数），来获取相应的次数。具体代码如下所示：
```
   static final class HoldCounter {
            int count;//记录当前线程进入的次数
            final long tid = getThreadId(Thread.currentThread());
        }
    
   static final class ThreadLocalHoldCounter extends ThreadLocal<HoldCounter> {
            public HoldCounter initialValue() {
                return new HoldCounter();
            }
        }
     
   private transient ThreadLocalHoldCounter readHolds;
```
如果有小伙伴不熟悉`ThreadLocal`，可以参看该篇文章[《Android Handler机制之ThreadLocal》](https://www.jianshu.com/p/2a34d30806d4)

##### 步骤（4）中继续尝试获取读锁？
当第一次获取读锁失败的时候，会调用`fullTryAcquireShared(Thread current)`方法会继续尝试获取锁。该函数返回的三个条件为：
- 当前已经存在写锁了。直接加入AQS中的同步队列中。
- 当前写锁的次数超过最大值，直接抛出异常
- 获取读锁成功。直接返回

具体代码如下所示：
```
    final int fullTryAcquireShared(Thread current) {
            HoldCounter rh = null;
            for (;;) {//注意这里的for循环
                int c = getState();
                if (exclusiveCount(c) != 0) {//（1）存在写锁直接返回
                    if (getExclusiveOwnerThread() != current)
                        return -1;
                } else if (readerShouldBlock()) {
                    // Make sure we're not acquiring read lock reentrantly
                    if (firstReader == current) {
                        // assert firstReaderHoldCount > 0;
                    } else {
                        if (rh == null) {
                            rh = cachedHoldCounter;
                            if (rh == null || rh.tid != getThreadId(current)) {
                                rh = readHolds.get();
                                if (rh.count == 0)
                                    readHolds.remove();
                            }
                        }
                        if (rh.count == 0)
                            return -1;
                    }
                }
                if (sharedCount(c) == MAX_COUNT)//(2)锁迭代次数超过最大值。抛出异常
                    throw new Error("Maximum lock count exceeded");
                if (compareAndSetState(c, c + SHARED_UNIT)) {//(3)获取锁成功，记录次数
                    if (sharedCount(c) == 0) {
                        firstReader = current;
                        firstReaderHoldCount = 1;
                    } else if (firstReader == current) {
                        firstReaderHoldCount++;
                    } else {
                        if (rh == null)
                            rh = cachedHoldCounter;
                        if (rh == null || rh.tid != getThreadId(current))
                            rh = readHolds.get();
                        else if (rh.count == 0)
                            readHolds.set(rh);
                        rh.count++;
                        cachedHoldCounter = rh; // cache for release
                    }
                    return 1;
                }
            }
        }
```
因为该方法和上文提到的`tryAcquireShared(int unused)`方法较为类似。所以这里就不再对其中的逻辑再次讲解。大家需要注意的是该方法会`自旋式的获取锁`。
 
#### 写锁的获取
了解了读锁的获取，再来了解写锁的获取就非常简单了。写锁的获取最终会走`Sync`中的`tryAcquire(int acquires)`方法。具体代码如下所示：
```
   protected final boolean tryAcquire(int acquires) {
            Thread current = Thread.currentThread();
            //（1）获取同步状态 = 写状态+读状态，单独获取写状态
            int c = getState();
            int w = exclusiveCount(c);
            //（2）如果c!=0则表示有线程操作
            if (c != 0) {
                // （2.1）没有写锁线程，则表示有读线程，则直接获取失败，并返回
                if (w == 0 || current != getExclusiveOwnerThread())
                    return false;
                    
                 //（2.2）如果w>0则，表示当前线程为写线程，则计算当前重进入的次数，如果已经饱和，则抛出异常
                if (w + exclusiveCount(acquires) > MAX_COUNT)
                    throw new Error("Maximum lock count exceeded");
                    
                // （2.3）获取成功，直接记录当前写状态
                setState(c + acquires);
                return true;
            }
            //（3）没有线程获取读写锁，根据当前锁的模式与设置写状态是否成功，判断是否需要阻塞线程
            if (writerShouldBlock() ||
                !compareAndSetState(c, c + acquires))
                return false;
            //(4)第一次进入，获取成功   
            setExclusiveOwnerThread(current);
            return true;
        }
```
为了帮助大家理解，我这里将该方法分为了一下几个步骤：
- （1）获取同步状态 `c`（写状态+读状态），并单独获取写状态`w`。
- （2）如果`c!=0`则表示有线程操作。
- （2.1）没有写锁线程，则表示有读线程，则直接获取失败，并返回。
- （2.2）如果`w>0`则，表示当前线程为写线程，则计算当前重进入的次数，如果已经饱和，`则抛出异常`。
- （2.3）获取成功，直接记录当前写状态。
- （3）在（2）条件不满足的条件下，没有线程获取读写锁，根据当前锁的模式与设置写状态是否成功，判断是否需要阻塞线程
- （4）在（2）（3）条件都不满足的情况下，则为第一次进入，那么就获取成功 。

相信结合以上步骤。再来理解代码就非常容易了。

### 锁降级
读写锁除了保证写操作对读操作的可见性以及并发性的提升之外，读写锁也能简化读写交互的编程方式，试想一种情况，在程序中我们需要定义一个共享的用作缓存数据结构，并且其大部分时间提供读服务（例如查询和搜索），而写操作占有的时间很少，但是我们又希望写操作完成之后的更新需要对后续的读操作可见。那么该怎么实现呢？参看如下例子：
```
public class CachedData {
    Object data;
    volatile boolean cacheValid;
    final ReentrantReadWriteLock rwl = new ReentrantReadWriteLock();
    
    void processCachedData() {
        rwl.readLock().lock();
        if (!cacheValid) {
            //如果缓存过期，释放读锁，并获取写锁
            rwl.readLock().unlock();
            rwl.writeLock().lock();（1）
            try {
                //重新检查缓存是否过期，因为有可能在当前线程操作之前，其他写线程有可能改变缓存状态
                if (!cacheValid) {
                    data = ...//重新写入数据
                    cacheValid = true;
                }
                // 获取读锁
                rwl.readLock().lock();（2）
            } finally {
	            //释放写锁
                rwl.writeLock().unlock(); （3）
            }
        }

        try {
            use(data);//操作使用数据
        } finally {
            rwl.readLock().unlock();//最后释放读锁
        }
    }
}
```
在上述例子中，如果数据缓存过期，也就是cacheValid变量（volatile 修饰的布尔类型）被设置为false，那么所有调用processCachedData（）方法的线程都能感知到变化，但是只有一个线程能过获取到写锁。其他线程会被阻塞在读锁和写锁的lock()方法上。当前线程获取写锁完成数据准备之后，再获取读锁，随后释放写锁（上述代码的（1）（2）（3）三个步骤），**这种在拥有写锁的情况下，在获取读锁。随后释放写锁的过程，称之为锁降级（在读写锁内部实现中，是支持锁锁降级的）**。

那接下来，我个问题想问大家，`为什么当线程获取写锁，修改数据完成后，要先获取读锁呢，而不直接释放写锁呢？`，其实原因很简单，如果当前线程直接释放写锁，那么这个时候如果有其他线程获取了写锁，并修改了数据。那么对于当前释放写锁的线程是无法感知数据变化的。先获取读锁的目的，就是保证没有其他线程来修改数据啦。
 
### 总结

- ReentrantReadWriteLock最多支持 `65535` 个递归写入锁和`65535`个读取锁。
- ReentrantReadWriteLock中用同一`int`变量的`高16位`表示`读状态`，`低16位`表示`写状态`。
- ReentrantReadWriteLock支持公平锁与非公平锁模式。
- ReentrantReadWriteLock支持锁的降级。
