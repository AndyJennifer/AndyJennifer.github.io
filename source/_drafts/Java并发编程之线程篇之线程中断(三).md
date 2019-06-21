---
title: Java并发编程之线程篇之线程中断(三)
tags:
- 线程
categories:
- 线程相关
---


### 前言
在上篇文章{% post_link Java并发编程之线程篇之线程简介(二) %}中我们基本了解了如何创建一个线程并执行相应任务，但是并没有提到如何中断一个线程。例如：我们有一个下载程序线程，该线程在没有下载成功之前是不会退出的，假如这个时候用户不想下载了，那我们该如何中断这个下载线程呢？下面我们就来学习如何正确的中断一个线程吧。

>对于过时的suspend()、resume()和stop()方法，这里就不介绍了，有兴趣的小伙伴可以查阅相关资料。

### Java线程的中断机制
当我们需要中断某个线程时，看似我们只需要调一个中断方法（调用之后线程就不执行了）就行了。但是Java中并没有提供一个实际的方法来中断某个线程（不考虑过时的stop()方法），只提供了一个中断标志位，来表示线程在运行期间已经被其他线程进行了中断操作。也就是说线程只有通过自身来检查这个标志位，来判断自己是否被中断了。在Java中提供了三个方法来设置或判断中断标志位，具体方法如下所示：

```
   //类方法，设置当前线程中标志位
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
在上述方法中，我们可以通过`interrupt()`来设置相应线程中断标志，通过Thread类`静态方法interrupted()`和类方法` isInterrupted()`来判断对应线程是否被其他线程中断。其中interrupted（）与isInterrupted（）方法的主要区别如下：

- interrupted 判断当前线程是否中断（如果是中断，则会清除中断的状态标志，也就是如果中断了线程，第一次调用这个方法返回true，第二次继续调用则返回false。
- isInterrupted 判断线程是否已经中断（不清除中断的状态标志）。

### 使用interrupt()中断线程
在上文中，我们了解了线程的如何设置中断标志位与如何判断标志位，那现在我们来使用`interrupt()`方法来中断一个线程。先看下面这个例子。

```
class InterruptDemo {

    public static void main(String[] args) {
        Thread thread = new Thread(new Runnable() {
            @Override
            public void run() {
                for (int i = 0; i < 1000000; i++) {
                    System.out.println("i=" + i);
                }
            }
        });
        thread.start();
        try {
            Thread.sleep(2000);
            thread.interrupt();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}
//输出结果：
i=593210
i=593211
i=593212
i=593213
i=593214
i=593215
i=593216

```
运行上述代码，观察输出结果，我们发现线程并没有被终止，原因是因为`interrupt()方法只会设置线程中断标志位，并不会真正的中断线程`。也就是说我们只有自己来判断线程是否终止。一般情况下，当我们检查到线程被中断（也就是线程标志位为true)时，会抛出一个`InterruptedException`异常，来中断线程任务。具体代码如下：

```
class InterruptDemo {

    public static void main(String[] args) {
        Thread thread = new Thread(new Runnable() {
            @Override
            public void run() {
                try {
                    for (int i = 0; i < 1000000; i++) {
                        if (Thread.interrupted()) {
                            System.out.println("检测到线程被中断");
                            throw new InterruptedException();
                        }
                        System.out.println("i=" + i);
                    }
                } catch (InterruptedException e) {
                    //执行你自己的中断逻辑
                    System.out.println("线程被中断了，你自己判断该如何处理吧");
                    e.printStackTrace();
                }
            }
        });
        thread.start();
        try {
            Thread.sleep(2000);
            thread.interrupt();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}
//输出结果：
i=218626
i=218627
i=218628
i=218629
i=218630
检测到线程被中断
线程被中断了，你自己判断该如何处理吧
java.lang.InterruptedException
	at InterruptDemo$1.run(InterruptDemo.java:18)
	at java.base/java.lang.Thread.run(Thread.java:835)
```
在上述代码中，我们通过在线程中判断`Thread.interrupted()`来判断线程是否中断，当线程被中断后，我们抛出InterruptedException异常。然后通过try/catch来捕获该异常来执行我们自己的中断逻辑。当然我们也可以通过`Thread.currentThread().isInterrupted()`来判断。这两个方法的区别已经在上文介绍了，这里就不过多的介绍了。

### 中断线程的另一种方式
在上文中提到的使用interrupt()来中断线程以外，我们还可以通过一个boolean来控制是否中断线程。具体例子如下所示：

```
class InterruptDemo {

    public static void main(String[] args) {
        RunnableA runnableA = new RunnableA();
        Thread thread = new Thread(runnableA);
        thread.start();
        try {
            Thread.sleep(2000);
            runnableA.interruptThread();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }

    static class RunnableA implements Runnable {
        boolean isInterrupt;

        @Override
        public void run() {
            for (int i = 0; i < 1000000; i++) {
                if (!isInterrupt) {
                    System.out.println("i=" + i);
                } else {
                    System.out.println("线程结束运行了");
                    break;
                }
            }
        }

        void interruptThread() {
            isInterrupt = true;
        }
    }
}
//输出结果：
i=240399
i=240400
i=240401
i=240402
i=240403
i=240404
i=240405
线程结束运行了
```
上述代码中，我们通过判断`isInterrupt`的值来判断是否跳出循环。run（）方法的结束，就标志着线程已经执行完毕了。

### 最后
站在巨人的肩膀上，才能看的更远~

- 《Java并发编程的艺术》