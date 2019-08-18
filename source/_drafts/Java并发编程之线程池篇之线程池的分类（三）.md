---
title: 'Java并发编程之线程池篇之线程池的分类（三）'
tags: 
- 线程池的分类
categories:
- 线程池
---

### FixedThreadPool

```
    public static ExecutorService newFixedThreadPool(int nThreads) {
        return new ThreadPoolExecutor(nThreads, nThreads,
                                      0L, TimeUnit.MILLISECONDS,
                                      new LinkedBlockingQueue<Runnable>());
    }
```

### CachedThreadPool

```
    public static ExecutorService newCachedThreadPool() {
        return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                                      60L, TimeUnit.SECONDS,
                                      new SynchronousQueue<Runnable>());
    }
```

### ScheduledThreadPool
```
    public static ScheduledExecutorService newScheduledThreadPool(int corePoolSize) {
        return new ScheduledThreadPoolExecutor(corePoolSize);
    }
```

```
    public ScheduledThreadPoolExecutor(int corePoolSize,
                                       ThreadFactory threadFactory) {
        super(corePoolSize, Integer.MAX_VALUE,
              DEFAULT_KEEPALIVE_MILLIS, MILLISECONDS,
              new DelayedWorkQueue(), threadFactory);
    }
```
### SingleThreadExecutor

```
    public static ExecutorService newSingleThreadExecutor() {
        return new FinalizableDelegatedExecutorService
            (new ThreadPoolExecutor(1, 1,
                                    0L, TimeUnit.MILLISECONDS,
                                    new LinkedBlockingQueue<Runnable>()));
    }
```

### 多线程一定快吗？
有了线程之后，大家可能会觉得，程序就应该跑的很快了。当我们需要执行多个任务的时候，我们完全可以创建多个线程来执行相应的任务就行了啊，时间肯定比一个线程执行来的快呢！具体情况如下图所示：

{% asset_img 单线程多线程执行情况.png 单线程多线程执行情况 %}
从上图来看，好像创建多个线程来执行任务确实是比一个线程串行执行任务所花费的时间短，但是并不是所有的情况都是启动更多的线程就能让程序最大的并发执行。虽然我们的初衷是多线程执行任务让程序运行的更快。但是这样任然又许多的问题需要处理。比如上下文切换的问题。

#### 上下文的切换

对于单核CPU来说（对于多核CPU，此时就理解为一个核），CPU在一个时刻只能运行一个线程。CPU为了实现多线程的机制，通过时间片分配算法来循环执行任务时，当前线程执行一个时间片后会切换到下一个线程，但是在切换前会保持上一个线程的状态。以便下次切换回这个线程时，可以获取这个线程的状态，所以线程从保存再到加载到过程就是一次上下文切换。


>时间片是cpu分配给每个线程执行的时间，因为时间非常短，所以CPU通过不断的切换线程执行，让我们感觉到多个线程是同时执行的。时间片一般是几十毫秒每秒。

为了让大家实际了解线程的上下文的切换的开销。那么下面我们就看下面的这个例子：
```
class ConcurrencyTest {

    private static final long count = 100000000;

    public static void main(String[] args) throws InterruptedException {
        concurrency();
        serial();

    }

    private static void concurrency() throws InterruptedException {
        long start = System.nanoTime();
        //任务1
        Thread thread = new Thread(new Runnable() {
            @Override
            public void run() {
                int a = 0;
                for (long i = 0; i < count; i++) {
                    a += 5;
                }
            }
        });
        thread.start();

        //任务2
        int b = 0;
        for (long i = 0; i < count; i++) {
            b--;
        }
        thread.join();//当前main线程等待thread线程终止之后才从thread.join()返回

        //计算时间
        long time = System.nanoTime() - start;
        System.out.println("concurrency :" + time + "ns,b=" + b);
    }

    private static void serial() {
        long start = System.nanoTime();
        int a = 0;
        for (long i = 0; i < count; i++) {
            a += 5;
        }
        int b = 0;
        for (long i = 0; i < count; i++) {
            b--;
        }
        long time = System.nanoTime() - start;
        System.out.println("serial:" + time + "ns,b=" + b + ",a=" + a);
    }
}
```
在上述代码中，我将两个for循环中的内容，分别定义为任务1与任务2。我动态的修改循环的次数，可以得到图表：

{% asset_img 测试结果.png 测试结果 %}

从上表中我们可以看出，当循环次数不超过一百万次时，串行的执行速度始终比并发执行的速度快。也就验证了线程的创建和上下文的切换确实是有开销的。

### 如何解决上下文切换的开销？
在多线程的情况下，因为有线程的上下文切换。可能多线程的执行速度还没有串行执行的快。那么我们有不有办法解决这个问题呢？终于到我们的重头戏了，那就是使用协程，使用协程后，我们可以在单线程的模型下，实现多任务的调度，并在单线程里维持多个任务的切换。那使用协程就能减少线程上下文的开销了？你个糟老头，坏的很。你告诉我协程到底是什么东西啊！不急不急，在了解协程原理之前，我们要先了解操作系统中的线程实现方式。
