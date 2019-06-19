---
title: 'Java并发编程之线程篇之线程简介(二) '
tags:
- 线程
categories:
- 线程相关
---

### 前言

在上一篇文章中已经主要讲解了线程的由来，以及进程与线程的关系。接下来我们就继续讲解在Java中线程的相关知识。主要内容包括，Java构造与启动线程的方式
、线程优先级、线程的状态等知识点。希望大家继续保持一个热爱学习的心。快乐和我一起学习吧。

### Java程序中进程和线程的关系
在Java中，一个应用程序对应着一个JVM实例（也有地方称为JVM进程），一般来说名字默认为java.exe或者javaw.exe（windows下可以通过任务管理器查看）。Java采用的是单线程编程模型，即在我们自己的程序中如果没有主动创建线程的话，只会创建一个线程，通常称为主线程。但是要注意，虽然只有一个线程来执行任务，不代表JVM中只有一个线程，JVM实例在创建的时候，同时会创建很多其他的线程（比如垃圾收集器线程）。

由于Java采用的是单线程编程模型，因此在进行UI编程时要注意将耗时的操作放在子线程中进行，以避免阻塞主线程（在UI编程时，主线程即UI线程，用来处理用户的交互事件）

```
public class Main {

    public static void main(String[] args) {
        //获取当前进程中所有堆栈信息
        Map<Thread, StackTraceElement[]> allStackTraces = Thread.getAllStackTraces();
        //打印所有线程名称
        Set<Thread> threads = allStackTraces.keySet();
        for (Thread thread : threads) {
            System.out.println("线程名：" + thread.getName());
        }
    }
}
```
输出结果：
```
线程名：Finalizer
线程名：Reference Handler
线程名：Signal Dispatcher
线程名：Common-Cleaner
线程名：main
线程名：Monitor Ctrl-Break

```

### 构造线程
在Java中构造线程需要创建Thread对象，该对象在构造时，默认会调用Thread类中的`init`函数来构造线程所需要的属性，如线程所属的线程组、优先级、堆栈大小等。具体代码如下所示：

```
    /**
     * 初始化一个线程对象
     *
     * @param g 当前新线程所属的线程组
     * @param target 任务对象
     * @param name 当前新线程的名称
     * @param stackSize 当前新线程所需要的堆栈大小
     */
    private void init(ThreadGroup g, Runnable target, String name, long stackSize) {
        //获取创建当前新线程的线程，也就是当前线程的父线程
        Thread parent = currentThread();
        if (g == null) {
            g = parent.getThreadGroup();
        }

        g.addUnstarted();
        this.group = g;

        this.target = target;
        //复用父线程的优先级与daemon属性
        this.priority = parent.getPriority();
        this.daemon = parent.isDaemon();
        //设置当前新线程的名称
        setName(name);

        init2(parent);

        /* Stash the specified stack size in case the VM cares */
        this.stackSize = stackSize;
        //给当前新创建的线程分配id
        tid = nextThreadID();
    }
```
观察上述步骤，我们可以发现在init函数中有调用了`init2`函数，我们继续观察该函数，代码如下所示：

```
    private void init2(Thread parent) {
        //复用父线程的类加载器，
        this.contextClassLoader = parent.getContextClassLoader();
        this.inheritedAccessControlContext = AccessController.getContext();
        //复用父线程的可继承的ThreadLocal
        if (parent.inheritableThreadLocals != null) {
            this.inheritableThreadLocals = ThreadLocal.createInheritedMap(
                    parent.inheritableThreadLocals);
        }
    }
```
上述代码中，新线程复用了父线程的类加载器与可继承的ThreadLocal。对于ThreadLocal，之前我也写了文章{% post_link Android Handler机制之ThreadLocal %}。有需要的小伙伴可以看看。这里我们只需要注意的，线程中的`inheritableThreadLocals`一般不会使用。除非你需要新创建的线程复用父线程的`inheritableThreadLocals`。对于类加载器，这里就不过多介绍了。感兴趣的小伙伴可以查阅相关资料。

综上所述。我们可以知道一个新构造的线程对象是与其父线程（parent 线程)息息相关的。新创建的线程会复用父线程的。优先级、类加载器、可继承的ThreadLocal，同时在构造时，还会为该线程分配相应的线程id。至此Java构造线程的方法已经介绍完毕了。

### 构造线程任务
在Java中创建线程任务，一般有两种方式。第一种继承Thread类，并复写其run方法。第二种实现Runnable接口。下面分别对这两种方式进行介绍。具体代码如下所示：

```
public class Main {

    public static void main(String[] args) {
        //使用继承Thread方式
        MyThreadA myThreadA = new MyThreadA();
        myThreadA.start();
        //使用实现Runnable接口
        Thread thread = new Thread(new MyRunnableB());
        thread.start();
    }
    
    static class MyThreadA extends Thread {
        @Override
        public void run() {
            System.out.println("MyThreadA线程任务已经启动了");
        }
    }

    static class MyRunnableB implements Runnable {
        @Override
        public void run() {
            System.out.println("MyRunnableB线程任务已经启动了");
        }
    }
}
```
观察上述过程，可以发现不管采用何种方式来构建线程任务，我们都需要创建相应的Thread对象。并调用其start方法。而调用start方法的含义是，告诉当前Java虚拟机，当前线程已经初始化完毕。如果线程规划器空闲。则立即调用对于Thread的`run()`方法执行相应任务。

那实现Runnable接口与继承Thread来创建线程任务之间到底有什么区别呢？如果我们观察使用Runnable接口的方式，我们发现其实际调用了Thread类的构造函数。如下所示：

```
 public Thread(Runnable target) {
        this(null, target, "Thread-" + nextThreadNum(), 0);
    }
```
该构造函数最终会调用我们上文提到的`init`方法，而该方法最终会将我们传入的Runabale对象赋值给Thread的`target`属性。我们再仔细观察Thread的`run（）`方法，我们会发现其中有一个判断。如下所示：
```
  public void run() {
        if (target != null) {
            target.run();
        }
    }
```
那么现在区别就非常明显了。实现Runnable接口来创建线程任务，主要是为了将具体的任务抽象出来。这样就可以把相同的任务给不同的线程执行。而不用创建多个Thread对象来执行任务了。

### 线程的优先级
这里要将分时系统，然后将线程的优先级

### Daemon线程（守护线程）

### 线程的状态
画画线程的状态，讲讲查看线程状态的java命令

### 最后

