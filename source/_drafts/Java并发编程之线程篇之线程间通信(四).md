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
例子
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