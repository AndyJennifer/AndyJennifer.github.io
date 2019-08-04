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

volatile写代码，synchronized画图

### 等待/通知机制

1.讲讲synchonized wait notify 配合使用
2.讲讲lock condition的配合使用
3.讲讲两个的区别

### 等待/通知的经典范式
在多数情况下，主线程生成并起动了子线程，如果子线程里要进行大量的耗时的运算，主线程往往将于子线程之前结束，但是如果主线程处理完其他的事务后，需要用到子线程的处理结果，也就是主线程需要等待子线程执行完成之后再结束，这个时候就要用到join()方法了。

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
            mAThread.join();
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
            aThread.join();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println(Thread.currentThread().getName() + "--->end");
    }
}
```

输出结果
```xml
main-->start
[AThread]-->start
[AThread]loop at0
[AThread]loop at1
[BThread]-->start
[AThread]loop at2
[AThread]loop at3
[AThread]loop at4
[AThread]--->end
main--->end
[BThread]--->end
```

源码解析：

```java
    public final synchronized void join(final long millis)
    throws InterruptedException {
        if (millis > 0) {
            if (isAlive()) {
                final long startTime = System.nanoTime();
                long delay = millis;
                do {
                    wait(delay);
                } while (isAlive() && (delay = millis -
                        TimeUnit.NANOSECONDS.toMillis(System.nanoTime() - startTime)) > 0);
            }
        } else if (millis == 0) {
            while (isAlive()) {
                wait(0);
            }
        } else {
            throw new IllegalArgumentException("timeout value is negative");
        }
    }
```

### ThreadLocal

在上述文章中，我们都是讲解的多个线程之前的通信，那么在同一线程中，在某个时刻我们想获取线程中设置的变量，我们可以通过ThreadLocal,在之前的文章中 {% post_link Android-Handler机制之ThreadLocal %}，我们介绍过ThreadLocal的使用，

``` java
例子
```

### 最后

站在巨人的肩膀上，才能看的更远~