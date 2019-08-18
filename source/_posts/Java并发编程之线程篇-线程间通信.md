---
title: Java并发编程之线程篇之线程间通信(四)
tags:
  - 线程
categories:
  - 线程相关
date: 2019-08-18 20:33:08
---


### 前言

在上篇文章{% post_link Java并发编程之线程篇之线程中断(三) %}中我们讲解了线程中断的相关知识点，现在我们来了解一下线程间的通信。线程间的通信在我们实际项目中是不可或缺的，多数情况下，我们需要创建多个线程，配合完成某项任务。合理并正确使用线程间的通信方式，是作为一个良好程序员必须掌握的技能。那现在就让我们来了解在Java中线程间通信的处理方式吧！阅读该篇文章你可能需要具备一下知识点：

- {% post_link Java并发编程之Java内存模型(一) %}
- {% post_link Java并发编程之Volatile(二) %}
- {% post_link Java并发编程之Synchronized(三) %}
- {% post_link Java并发编程之锁机制之Lock接口(七) %}
- {% post_link Java并发编程之锁机制之AQS(AbstractQueuedSynchronizer)(八) %}
- {% post_link Java并发编程之锁机制之LockSupport工具(九) %}
- {% post_link Java并发编程之锁机制之Condition接口(十) %}
- {% post_link Java并发编程之锁机制之(ReentrantLock)重入锁(十一) %}

### 线程的状态

在了解线程的通信知识之前，我们需要了解线程的状态，熟悉线程的状态，不仅有助于我们更好的排查多线程项目出现的死锁、线程安全等问题，还能更好的让我们分析与理解线程间的通信。下面就让我们来看一下线程有哪些状态吧！

在Java中线程的有以下5种状态，如下所示：

|   线程状态     | 含义             |
|:------------- |:---------------:|
| NEW     | 新建状态，线程已经创建，但是没有执行start()方法               |
| RUNNABLE| 可运行状态，线程可以在JVM中运行，但是还需要等待CPU分配资源      |  
| BLOCKED | 阻塞状态，当遇到synchronized且没有取得相应的锁，就会进入这个状态  |
| WAITING | 等待状态，当线程中wait()/join/Locksupport.park方法时，就会进入这个状态    |  
| TIMED_WAITING| 计时等待状态，当调用Thread.sleep()或者Object.wait(xx)或者Thread.join(xx)或者LockSupport.parkNanos或者LockSupport.partUntil时，进入该状态 |  
| TERMINATED     | 线程中断状态，线程被中断或者运行结束，就会进入这个状态            |

在上述表格中，线程的5种状态对应着Java的不同方法，具体如下图所示：

{% asset_img 线程状态.jpg %}

>需要注意的是，在上图中`标红`的两个状态，是操作系统中线程对应的状态，Java将这两种状态合并为**可运行状态(RUNNABLE)**。在操作系统中**就绪状态(READY)**表示线程已经准备完毕，等待CPU分配时间片。**运行中状态(RUNNING)**表示当线程分到时间片，线程开始正式执行。

### volatile的使用

在Java内存模型中，我们曾提到过，为了提示程序的运行速度，Java将内存分为了工作内存（线程独占，不与其他线程共享）与主内存。当多个线程同时访问同一个对象或者变量的时候，由于每个线程都需要将该对象或变量拷贝到自己的工作内存中。又因为线程的工作内存是私有且不与其他线程共享的。那么当一线程修改变量的值后，会导致对其他线程不可见。Java内存模型如下图所示：

{% asset_img Java内存模型.png %}

为了保证数据的可见性。Java提供了`volatile`关键字。volatile关键字修饰变量，就是告知线程对该变量的访问必须重主内存中获取。而对它的改变必须同步刷新到主内存中。这样就能保证线程对变量访问的可见性。关于volatile的使用，参看如下例子：

```java
class VolatileDemo {

    int a = 1;
    int b = 2;

    public void change() {
        a = 3;
        b = 4;
    }

    public void print(String threadName) {
        System.out.println(threadName + "--->" + "a = " + a + ";b = " + b);
    }

    public static void main(String[] args) {
        final VolatileDemo volatileDemo = new VolatileDemo();
        new Thread(new Runnable() {
            @Override
            public void run() {
                try {
                    Thread.sleep(10);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                volatileDemo.change();
            }
        }).start();

        for (int i = 0; i < 100; i++) {
            new Thread(new Runnable() {
                @Override
                public void run() {
                    try {
                        Thread.sleep(10);
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                    volatileDemo.print(Thread.currentThread().getName());
                }
            }).start();
        }

    }
}

```

程序输出结果：

```java
Thread-1--->a = 1;b = 2 //错误
Thread-3--->a = 1;b = 2 //错误
Thread-2--->a = 1;b = 2 //错误
Thread-5--->a = 3;b = 4
Thread-4--->a = 3;b = 4
Thread-6--->a = 3;b = 4
Thread-7--->a = 3;b = 4
Thread-8--->a = 3;b = 4
....省略其他
```

在上述代码中，如果我们不采用`volatile`关键字修饰a,b变量，那么会导致其他线程仍然获取的是自己本身工作内存中的a、b变量的值。为了保证访问公共变量对其他线程的可见性，我们需要将变量通过`volatile`来修饰。修改我们的代码：

```java
volatile  int a = 1;
volatile  int b = 2;
```

采用volatile修饰，输出结果如下

```java
Thread-2--->a = 3;b = 4
Thread-1--->a = 3;b = 4
Thread-6--->a = 3;b = 4
Thread-3--->a = 3;b = 4
Thread-4--->a = 3;b = 4
Thread-9--->a = 3;b = 4
Thread-10--->a = 3;b = 4
....省略其他
```

需要注意的是volatile只对单次的变量的操作具对其他线程有可见性，对应类似于`a++`，这种操作线读取a变量的值，进行运算后再重新对变量赋值的操作，仍然会出现线程安全的问题。对关于volatile的更多介绍，大家可以查看{% post_link Java并发编程之Volatile(二) %}文章。

### synchronized的使用

除了使用volatile实现线程的通信之外，我们还可以使用`synchronized`及`Object`中的配套方法`wait()/notify()、wait()/notifyAll`来实现线程的通信，在了解具体的实现之前，我们先来了解`synchronized`关键字在Java中的作用。

>关于synchronized的更多介绍，可以查看{% post_link Java并发编程之Synchronized(三) %}文章。

关键字synchronized可以修饰方法或者代码块，使用synchronized可以确保多个线程在同一时刻，只能有一个线程处于方法或者同步块中，它保证了线程对变量访问的可见性和排他性。如下代码所示：

```java
public class SyncCodeBlock {
   public int i;
   public void syncTask(){
       //同步代码库
       synchronized (this){
           i++;
       }
   }
}
```

然后我们通过javap指令反编译得到字节码。来继续分析synchronized关键字的实现细节，如下所示：

```java
 //===========主要看看syncTask方法实现================
  public void syncTask();
    descriptor: ()V
    flags: ACC_PUBLIC
    Code:
      stack=3, locals=3, args_size=1
         0: aload_0
         1: dup
         2: astore_1
         3: monitorenter  //注意此处，进入同步方法，表示获取到锁
         4: aload_0
         5: dup
         6: getfield      #2             // Field i:I
         9: iconst_1
        10: iadd
        11: putfield      #2            // Field i:I
        14: aload_1
        15: monitorexit   //注意此处，退出同步方法，释放锁
        16: goto          24
        19: astore_2
        20: aload_1
        21: monitorexit //注意此处，退出同步方法
        22: aload_2
        23: athrow
        24: return
      Exception table:
      //省略其他字节码.......
}
```

在上述字节码信息中，对于同步块的实现使用了`monitorenter`和`monitorexit`两个指令，其本质原理是对某个对象的监视器(monitor)的获取。而这个获取过程是排他的,也就是同一时刻只能有一个线程，获取到有synchronized所保护对象的监视器。在Java中，任何一个对象都有自己的监视器（monitor)，当这个对象有同步块或者这个对象的同步方法调用的时候，执行方法的线程必须先获取到该对象的监视器才能进入同步块，或者同步方法，而没有获取到监视器(monitor)的线程会阻塞在同步块或者同步方法的入口，进入BLOCKED状态。如下图所示：

{% asset_img synchronized中监视器获取关系.jpg %}

从上图中，我们可以得出，任意线程在对synchronized关键字修饰的Object进行访问的时候，首先要获得Object的监视器（monitor），如果获取失败，线程进入同步队列，且线程状态变为`BLOCKED`。当访问Object的前驱线程（moniterenter成功的线程）释放了Object的监视器（monitorexit)。则唤醒阻塞在同步队列中的线程，使其尝试对监视器的获取。其中对监视器的获取与释放，我们一般称之为获取锁与释放锁。在下文中我们都用获取锁与释放锁来表示这两个过程。

关于synchronized下同步队列的知识点补充：

synchronized在JVM中实现的锁机制是基于`同步队列`与`等待队列`的，这与`courrent`包下的`Lock`接口下的锁机制的实现方式十分类似。需要注意的是，在synchronized中wait()后的线程会进入一个FIFO的队列(同步队列)中，notify()/notifyAll()是一个有序的出队列的过程。

#### synchronized下的等待/通知机制实现

在上文中，我们提到如果使用synchronized来实现线程间的通信，我们需要结合Object中的配套方法`wait()/notify()、wait()/notifyAll`。我们先来看看一看Obejet中这系列方法的说明：

|   方法名称     | 描述             |
|:------------- |:---------------:|
| wait()     | 调用该方法的线程进入`WAITING`状态，只有等待另外线程的通知或被中断才会返回，需要注意，线程调用wait()方法前，需要获得对象的监视器。当调用wait()方法后，会释放对象的监视器            |
| wait(long)     | 调用该方法的线程进入`TIMED_WAITING`状态，这里的参数时间是毫秒，等待对应毫秒事件，如果没有收到其他线程通知，则超时返回|
|wait(long,int)  | 调用该方法的线程进入`TIMED_WAITING`状态，基本作用同wiat(long)，第二个参数代表为纳秒，也就是等待时间为毫秒+纳秒。|
|notify()        |通知一个在对象监视器上等待的线程，使其从wait()方法返回，而返回的前提是该线程获取到了对象的监视器。
|notifyAll()     |通知所有在监视器上等待的线程，具体唤醒那个线程由CPU决定|

使用Object的wait()/notify()、wait()/notifyall()，其实是我们经常使用的`等待/通知机制`，所谓的等待/通知机制是指一个线程A调用了对象O的wait()方法进入等待状态，而另一个线程B调用了对象O的notify或者notifyAll方法。线程A收到通知后从对象O的wait()方法返回，进而执行后续的操作。
下面，我们来通过一个例子来了解使用synchronized完成线程的通信，具体例子如下所示:

```java
class SynchronizedDemo {

    static boolean flag = true;
    static Object lock = new Object();

    public static void main(String[] args) throws InterruptedException {
        new Thread(new WaitRunnable(), "WaitThread").start();
        TimeUnit.SECONDS.sleep(1);//这里睡眠，是保证Wait线程先执行
        new Thread(new NotifyRunnable(), "NotifyThread").start();
    }

    static class WaitRunnable implements Runnable {
        @Override
        public void run() {
            synchronized (lock) {
                while (flag) {//注意,通过while循环来判断条件
                    String name = Thread.currentThread().getName();
                    try {
                        System.out.println(name + "--->wait in " + new SimpleDateFormat("HH:mm:ss").format(new Date()));
                        lock.wait();
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                    System.out.println(name + "--->wake up in " + new SimpleDateFormat("HH:mm:ss").format(new Date()));
                }

            }
        }
    }

    static class NotifyRunnable implements Runnable {
        @Override
        public void run() {
            String name = Thread.currentThread().getName();

            synchronized (lock) {
                System.out.println(name + "--->notify all in " + new SimpleDateFormat("HH:mm:ss").format(new Date()));
                lock.notifyAll();
                flag = false;
            }

            /**
             * 这里再次加锁，是为了验证当调用对象的notifyAll方法时，
             * 如果线程不执行monitorexit(也就是释放锁),那么是不会唤醒其他线程的
             */
            synchronized (lock) {
                try {
                    TimeUnit.SECONDS.sleep(2);
                    System.out.println(name + "--->hold lock again in " + new SimpleDateFormat("HH:mm:ss").format(new Date()));
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        }
    }

}
```

输出结果如下:

```java
WaitThread--->wait in 23:10:11
NotifyThread--->notify all in 23:10:12
NotifyThread--->hold lock again in 23:10:14
WaitThread--->wake up in 23:10:14
```

从上文中，我们可以得出以下结论：

- 使用wait()、notify()和notifyAll时需要先获取对象的监视器（执行monitorenter指令成功）
- 调用wait()方法后，线程状态由RUNNING变为WAITING，并将该线程加入等待队列。
- notify()或notifyAll()方法调用后，等待线程依旧不会从wait()返回，需要调用notify()或notfifyAll()的线程释放对象的监视器（也就是执行monitorexit指令）后，等待线程才会有机会从wait()返回。
- notify()方法将等待队列中的一个等待线程从等待队列移到同步队列中，而notifyAll()方法则是将等待队列中所有的线程全部移动到同步队列，被移动的线程状态由WAITING变为BLOCKED。
- 从wait()方法返回的前提是获得了调用对象的监视器(执行monitorenter指令成功）。

### Lock下的等待/通知机制实现

除了使用synchronized完成线程的通信之外，我们还可以使用`courrent`包下的`Lock`接口，这里以`ReentrantLock`为例。具体例子如下所示：

```java
class LockDemo {

    static boolean flag = true;
    static Lock lock = new ReentrantLock();
    static Condition codition = lock.newCondition();


    public static void main(String[] args) throws InterruptedException {
        new Thread(new WaitRunnable(), "WaitThread").start();
        TimeUnit.SECONDS.sleep(1);//这里睡眠，是保证Wait线程先执行
        new Thread(new NotifyRunnable(), "NotifyThread").start();
    }

    static class WaitRunnable implements Runnable {
        @Override
        public void run() {
            lock.lock();
            try {
                while (flag) {
                    String name = Thread.currentThread().getName();
                    System.out.println(name + "--->wait in " + new SimpleDateFormat("HH:mm:ss").format(new Date()));
                    codition.await();
                    System.out.println(name + "--->wake up in " + new SimpleDateFormat("HH:mm:ss").format(new Date()));
                }
            } catch (InterruptedException e) {
                e.printStackTrace();
            } finally {
                lock.unlock();
            }

        }
    }

    static class NotifyRunnable implements Runnable {
        @Override
        public void run() {
            lock.lock();
            try {
                String name = Thread.currentThread().getName();
                System.out.println(name + "--->notify all in " + new SimpleDateFormat("HH:mm:ss").format(new Date()));
                flag = false;
                codition.signalAll();
            } finally {
                lock.unlock();
            }
        }
    }
}

```

输出结果:

```java
WaitThread--->wait in 23:39:34
NotifyThread--->notify all in 23:39:35
WaitThread--->wake up in 23:39:35
```

关于Lock使用及原理，大家可以查看以下几篇文章，这里就不再进行分析了。

- {% post_link Java并发编程之锁机制之Lock接口(七) %}
- {% post_link Java并发编程之锁机制之AQS(AbstractQueuedSynchronizer)(八) %}
- {% post_link Java并发编程之锁机制之LockSupport工具(九) %}
- {% post_link Java并发编程之锁机制之Condition接口(十) %}
- {% post_link Java并发编程之锁机制之(ReentrantLock)重入锁(十一) %}

### 等待/通知的经典范式

从上方的例子中，我们可以总结并得到非常经典的等待/通知方式，该范式分别针对等待方法（消费者)和通知方(生产者)。

#### 等待方

等待方遵循如下原则：

1. 获取对象的锁。
2. 如果条件不满足，那么条件不满足，那么调用对象的wait()方法。被通知后仍然要继续检查条件。
3. 条件满足则执行相应逻辑。

对应伪代码分别如下：

**使用synchronized方式：**

``` java
synchronized(对象) {
    while(条件不满足){
        对象.wait();
    }
    对应的处理逻辑
}
```

**使用lock方式：**

```java
    lock.lock();
    try{
        while(条件不满足){
            condition.wait();
        }
    }finally{
        lock.unlock();
    }
```

#### 通知方代码

通知方遵循如下原则：

1. 获得对象的锁
2. 改变条件
3. 通知所有等待在对象上的线程。

对应伪代码分别如下：

**使用synchronized方式：**

```java
synchronized(对象){
    改变条件
    对象.notifyAll()
}
```

**使用lock方式：**

```java
    lock.lock();
    try{
        改变条件
        condition.singleAll()；
    }finally{
        lock.unlock();
    }
```

### Thread.join的使用

除了使用上面我们介绍的经典范式以外，我们还可以使用Thread.join()方法。join方法的使用含义如下：

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
            mAThread.join();//使B线程等待，需要等待A线程执行完毕后，才能继续执行
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

- `主线程(main线程）`等待`A线程`执行完毕后，才继续执行
- `B线程`等待`A线程`执行完毕后，才继续执行。

我们查看输出结果：

```xml
main-->start      //main线程启动
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
                wait(0);//当前线程存活，那么会使当前线程等待(当前线程指运行xxThread.join的线程，而不是xxThread)
            }
        } else {
            throw new IllegalArgumentException("timeout value is negative");
        }
    }
```

我们简单的分析一下代码，当B线程调用A线程的`join()`方法时，当前锁对象为A线程。在join()方法内部会调用`wait(0)`，该方法会使B线程等待。只有当A线程执行完毕后，也就是A线程终止后。才会唤醒B线程。

>线程执行完毕或线程终止时，会调用线程自身的notifyAll()方法，会通知所有等待在该线程对象的线程。

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

这里提供一个线程交替打印奇数偶数的例子，来帮助大家巩固所学的知识点。有兴趣的小伙伴，可以查看项目[PrintOddEventNumber](https://github.com/AndyJennifer/PrintOddEventNumber)。

### 参考

该文章参考以下图书，站在巨人的肩膀上。可以看得更远。

- 《Java并发编程的艺术》
