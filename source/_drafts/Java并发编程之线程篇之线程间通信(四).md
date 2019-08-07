---
title: Java并发编程之线程篇之线程间通信(四)
tags:
- 线程
categories:
- 线程相关
---

### 前言

在上篇文章{% post_link Java并发编程之线程篇之线程中断(三) %}中我们讲解了线程中断的相关知识点，现在我们来了解一下，线程间的通信。线程间的通信在我们实际项目中是不可获取的，多数情况下，我们需要多个线程，配合完成某项任务。合理的使用线程，压榨CPU资源，提高程序运行速度是我们需要掌握的必备技能。那现在就让我们来了解在Java中线程间通信的处理方式吧。阅读该篇文章你可能需要具备一下知识点：

- {% post_link Java并发编程之Java内存模型(一) %}
- {% post_link Java并发编程之Volatile(二) %}
- {% post_link Java并发编程之Synchronized(三) %}
- {% post_link Java并发编程之锁机制之Lock接口(七) %}
- {% post_link Java并发编程之锁机制之AQS(AbstractQueuedSynchronizer)(八) %}
- {% post_link Java并发编程之锁机制之LockSupport工具(九) %}
- {% post_link Java并发编程之锁机制之Condition接口(十) %}
- {% post_link Java并发编程之锁机制之(ReentrantLock)重入锁(十一) %}

### volatile与synchronized

讲讲volatile 原子类，synchonized（讲讲监视器）

volatile写代码，synchronized监视器画图

### 线程的状态


### 等待/通知机制

1.讲讲synchonized wait notify 配合使用
2.讲讲lock condition的配合使用
3.讲讲两个的区别

### 等待/通知的经典范式


#### 等待方伪代码

``` java
synchronized(对象) {
    while(条件不满足){
        对象.wait();
    }
    对应的处理逻辑
}
```

#### 通知方代码

```java
synchronized(对象){
    改变条件
    对象.notifyAll()
}
```

### 等待/通知超时的范式

### Thread.join的使用

当线程A调用线程B对象（bThread)的join方法，其含义是当前线程A等待线程B终止后，才从线程A中bThread.join()代码的调用处返回。线程除了join方法以外还提供了join(long millis)和void join(long millis, int nanos)这两个具备超时特性的方法。这两个方法的意义是如果在给定的时间内线程B没有终止。那么线程A将会从该方法中返回。下面我们来看一下join方法的使用例子，如下所示：

``` java
class AThread extends Thread {

    public AThread() {
        super("[AThread]");
    }

    @Override
    public void run() {
        String threadName = Thread.currentThread().getName();
        System.out.println(threadName + "-->start");
        try {
            for (int i = 0; i < 5; i++) {
                System.out.println(threadName + "loop at" + i);
                TimeUnit.SECONDS.sleep(1);
            }
            System.out.println(threadName + "--->end");
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}

class BThread extends Thread {

    private AThread mAThread;

    public BThread(AThread aThread) {
        super("[BThread]");
        this.mAThread = aThread;
    }

    @Override
    public void run() {
        String threadName = Thread.currentThread().getName();
        System.out.println(threadName + "-->start");
        try {
            mAThread.join();//阻塞B线程，需要等待A线程执行完毕后，才能继续执行
            System.out.println(threadName + "--->end");
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}

class ThreadJoinDemo {

    public static void main(String[] args) throws InterruptedException {
        System.out.println(Thread.currentThread().getName() + "-->start");
        AThread aThread = new AThread();
        BThread bThread = new BThread(aThread);
        try {
            aThread.start();
            TimeUnit.SECONDS.sleep(1);
            bThread.start();
            aThread.join();//主线程等待A线程执行完毕后，才继续执行
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println(Thread.currentThread().getName() + "--->end");
    }
}
```

在上述例子中，我们主要实现以下两个效果：

- 主线程等待A线程执行完毕后，才继续执行
- 线程等待A线程执行完毕后，才继续执行。

我们查看输出结果：

```xml
main-->start //main线程启动
[AThread]-->start //A线程启动
[AThread]loop at0 //A线程开始执行循环
[AThread]loop at1
[BThread]-->start //B线程开始启动，因为在B线程中调用了aThread.join（）那么B线程会等待A线程执行完毕后，才开始执行
[AThread]loop at2 //A线程继续执行
[AThread]loop at3
[AThread]loop at4
[AThread]--->end //A线程执行完毕后，
[BThread]--->end //A线程执行完毕后，唤醒B线程继续执行
main--->end //主线程执行完毕
```

整个程序是按照我们之前的逻辑在运行，下面我们来查看线程中join方法的实现原理，具体代码如下所示：

>join()方法内部会调用join(final long millis)方法。

```java
    //同步方法默认的锁为调用该方法的对象，也就是xxThread.join（）的xxThread
    public final synchronized void join(final long millis)
    throws InterruptedException {
        if (millis > 0) {//如果等待时间大于0
            if (isAlive()) {
                final long startTime = System.nanoTime();
                long delay = millis;
                do {
                    wait(delay);
                } while (isAlive() && (delay = millis -
                        TimeUnit.NANOSECONDS.toMillis(System.nanoTime() - startTime)) > 0);
            }
        } else if (millis == 0) {//如果等待时间为0，
            while (isAlive()) {
                wait(0);//当前线程存活，那么会阻塞当前线程(当前线程指运行xxThread.join的线程，而不是xxThread)
            }
        } else {
            throw new IllegalArgumentException("timeout value is negative");
        }
    }
```

我们简单的分析一下代码，当B线程调用A线程的`join()`方法时，当前锁对象为A线程。在join()方法内部会调用`wait(0)`方法来阻塞B线程。只有当A线程执行完毕后，也就是A线程终止后。才会唤醒B线程。

>线程执行完毕（线程终止）时，会调用线程自身的notifyAll()方法，会通知所有等待在该线程对象的线程。

### ThreadLocal

在上述文章中，我们都是讲解的多个线程之前的通信，那么在同一线程中，在某个时刻我们想获取线程中设置的变量，我们可以通过ThreadLocal，在之前的文章中 {% post_link Android-Handler机制之ThreadLocal %}，我们介绍过ThreadLocal的使用。下面我们通过一个例子来了解ThreadLocal的使用。具体例子如下：

```java
class ThreadLocalTest {
    private static ThreadLocal<String> mThreadLocal = new ThreadLocal<>();

    public static void main(String[] args) {
        mThreadLocal.set("线程main");
        new Thread(new A()).start();
        new Thread(new B()).start();
        System.out.println(mThreadLocal.get());
    }

    static class A implements Runnable {

        @Override
        public void run() {
            mThreadLocal.set("线程A");
            System.out.println(mThreadLocal.get());
        }
    }

    static class B implements Runnable {

        @Override
        public void run() {
            mThreadLocal.set("线程B");
            System.out.println(mThreadLocal.get());
        }
    }
}
```

输出结果：

```java
main
线程A
线程B
```

这里就不再介绍ThreadLocal的原理了，有兴趣的小伙伴可以查看{% post_link Android-Handler机制之ThreadLocal %}文章进行理解。

### 最后

站在巨人的肩膀上，才能看的更远~