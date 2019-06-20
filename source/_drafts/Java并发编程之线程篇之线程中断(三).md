---
title: Java并发编程之线程篇之线程中断(三)
tags:
- 线程
categories:
- 线程相关
---


### 前言
在上篇文章{% post_link Java并发编程之线程篇之线程简介(二) %}中我们基本了解了如何创建一个线程并执行相应任务，但是并没有提到如何中断一个线程。例如：我们有一个下载程序线程，该线程在没有下载成功之前是不会退出的，假如这个时候用户不想下载了，那我们该如何正确的中断这个线程呢？下面我们就来学习如何正确的中断一个线程吧。

>对于过时的suspend()、resume()和stop()方法，这里就不介绍了，有兴趣的小伙伴可以查阅相关资料。

### Java线程的中断机制
当我们需要中断某个线程时，看似我们只需要调一个中断方法就行了。但是Java中并没有提供一个实际的方法来中断某个线程（不考虑过时的stop()方法），只提供了一个中断标志位，来表示线程在运行期间已经被其他线程进行了中断操作。也就是说线程只有通过自身来检查这个标志位，来判断自己是否被中断了。如果被中断，那么就退出run()方法。

在Java中提供了三个方法分别设置或判断中断标志位，具体方法如下所示：

```
   //类方法，中断当前线程
   public void interrupt() {
        if (this != Thread.currentThread())
            checkAccess();

        synchronized (blockerLock) {
            Interruptible b = blocker;
            if (b != null) {
                interrupt0();           // 设置中断标志位
                b.interrupt(this);
                return;
            }
        }
        interrupt0();//设置中断标志位
    }

    //静态方法，判断当前线程是否中断，清除中断标志位
    public static boolean interrupted() {
        return currentThread().isInterrupted(true);
    }
    
    //类方法，判断当前线程是否中断，不清除中断标志位
    public boolean isInterrupted() {
        return isInterrupted(false);
    }
```


### 最后