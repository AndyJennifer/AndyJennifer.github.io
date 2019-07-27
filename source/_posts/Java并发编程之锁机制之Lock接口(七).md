---
title: Java并发编程之锁机制之Lock接口(七)
date: 2019-02-23 21:37:47
categories:
- Java并发相关
tags: 
- 并发
---

{% asset_img 小盒子.jpg 小盒子 %}

### 前言

在上篇文章{% post_link Java并发编程之锁机制之引导篇(六) %}中，我们大致了解了Lock接口（以及相关实现类）在并发编程重要作用。接下来我们就来具体了解Lock接口中声明的方法以及使用优势。

### Lock简介

Lock 接口实现类提供了比使用 synchronized 方法和语句可获得的更广泛的锁定操作。此实现允许更灵活的结构，可以具有差别很大的属性，可以支持多个相关的 `Condition` (Condition实现类ConditonObject来实现线程的通知/与唤醒机制，关于Condition后期会进行介绍)对象。

锁是用于控制多线程访问共享资源的工具。通常，锁提供对共享资源的独占访问：一次只有一个线程可以获取锁，对共享资源的所有访问都需要首先获取锁。但是，一些锁可以允许同时访问共享资源，例如`ReadWriteLock`。

虽然使用关键字synchronized修饰的方法或代码块，会使得在监视器模式（ObjectMonitor)下编程变得非常容易(通过synchronized块或者方法所提供的隐式获取释放锁的便捷性)。虽然这种方式简化了锁的管理，但是某些情况下，还是建议采用Lock接口（及其相关子类）提供的显示的锁的获取和释放。例如，针对一个场景，手把手进行锁获取和释放，先获得锁A，然后再获取锁B，当锁B获得后，释放锁A同时获取锁C，当锁C获得后，再释放B同时获取锁D，以此类推。这种场景下，
synchronized关键字就不那么容易实现了，而Lock接口的实现类`允许锁在不同的作用范围内获取和释放`，并允许以任何顺序获取和释放多个锁。

### Lock接口中的方法

关于Lock接口中涉及到的方法具体如下：（建议直接在PC端查看，手机上有可能看的不是很清楚）
{% asset_img lock_method.png lock_method %}
从上表中，我们就可以得出使用Lock接口实现的锁机制与使用传统的synchronized的区别

1. 尝试非阻塞地获取锁：当线程尝试获取锁，如果这一时刻锁没有被其他线程获取到，则成功获取并持有锁。
2. 能被中断的获取锁：与synchronized不同，获取到锁的线程能够响应中断，当获取到锁的线程被中断时，中断异常会被抛出，同时锁也会被释放。
3. 超时获取锁：在指定的截止时间之前获取锁，如果截止时间到了任然无法获取到锁，则返回。

### Lock简单使用与注意事项

其中Lock的使用方式也很简单，具体代码如下所示：

```java
Lock lock = ....;具体实现类
lock.lock();
try {
} finally {
lock.unlock();//建议在finally中释放锁
}
```

当锁定和解锁发生在不同的范围时，一定要注意确保在持有锁时执行的所有代码都受到try-finally或try-catch的保护，以确保在必要时释放锁。`不要将获取锁的过程写在try块中`，因为如果在获取锁（自定义锁的实现）时发生了异常，异常抛出的同时，也会导致锁无故释放（因为一旦发生异常，就会走finally语句，如果这个异常（可能是用户自定义异常，用户可以自己处理）需要线程1来处理，但是接着执行了lock.unlock()语句导致了锁的释放。那么其他线程就可以操作共享资源。有可能破坏程序的执行结果）。

### Lock相关实现类实现锁机制

为了使用Lock接口实现相关锁功能时，会涉及以下类和接口，这里还是把上篇文章提到的UML图展示出来：

{% asset_img lock.png lock %}

上图中，

 1. 绿色部分为：其中`ReentrantLock（重入锁）`、WriteLock、ReadLock都是Lock的实现类。`Segment为ReentrantLock的子类（在后续文章，ConcurrentHashMap的讲解中我们会提及）。` `ReentrantReadWriteLock （读写锁）`的实现使用了WriteLock与ReadLock类。
 2. 紫色部分为：其中`AbstractQueuedSynchronizer`与`AbstractQueuedLongSynchronizer`都为`AbstractOwnableSynchronizer`的子类，该两个类中都维护了一个同步队列，用于线程的并发执行。在该两个类中拥有名为`ConditionObject(为Conditon的实现类)`的内部类，只是其内部实现不同。在ConditionObject内部维护了一个等待队列，用于控制线程的等待与唤醒。

### 基本代码结构

在了解了Lock相关实现类实现锁机制后，这里给实现该锁机制的大致代码结构（根据不同需求，部分方法实现可能不一样，这里只是一个参考，并不是样本代码）。具体代码如下所示：

```java
class LockImpl implements Lock {

    private final sync mSync = new sync();
    @Override
    public void lock() {
        mSync.acquire(1);
    }
    @Override
    public void lockInterruptibly() throws InterruptedException {
        mSync.acquireInterruptibly(1);
    }

    @Override
    public boolean tryLock() {
        return mSync.tryAcquire(1);
    }

    @Override
    public boolean tryLock(long time, TimeUnit unit) throws InterruptedException {
        return mSync.tryAcquireNanos(1, unit.toNanos(time));
    }

    @Override
    public void unlock() {
        mSync.release(1);
    }

    @Override
    public Condition newCondition() {
        return mSync.newCondition();
    }

     //这里也可以继承AbstractQueuedLongSynchronizer
    private static class sync extends AbstractQueuedSynchronizer {
        @Override
        protected boolean isHeldExclusively() {...}
        @Override
        protected boolean tryAcquire(int arg) {...}
        @Override
        protected boolean tryRelease(int arg) {...}
        @Override
        protected int tryAcquireShared(int arg) {...}
        @Override
        protected boolean tryReleaseShared(int arg) {...}
        final ConditionObject newCondition() {...}
    }
}
```

从代码中我们可以看出，在整个Lock接口下实现的锁机制中，`AQS(这里我们将AbstractQueuedSynchronizer 或AbstractQueuedLongSynchronizer统称为AQS)`是实现锁的关键，整个锁的实现是在Lock类的实现类中聚合AQS来实现的，从代码层面上来说，Lock接口（及其实现类）是面向使用者的，它定义了使用者与锁交互的接口（比如可以允许两个线程并行访问），隐藏了实现细节。AQS与Condition才是真正的实现者，它简化了锁的实现方式，屏蔽了同步状态管理、线程的排队、等待与唤醒等底层操作。

### 总结

1. Lock接口（及其实现类）相比synchronized有如下优点：

- 锁的释放与获取不在是隐式的，允许锁在不同的作用范围内获取和释放`，并允许以任何顺序获取和释放多个锁。
- 能被中断的获取锁，获取到锁的线程能够响应中断，当获取到锁的线程被中断时，中断异常会被抛出，同时锁也会被释放
- 超时获取锁：在指定的截止时间之前获取锁，如果截止时间到了任然无法获取到锁，则返回。
  
1. 在使用Lock的时候注意，一定要确保必要时释放锁
2. 在整个Lock接口下实现的锁机制中,AQS(**上文进行了统称**)与Condition才是真正的实现者。
