---
title: Java并发编程之synchronized
date: 2019-02-23 21:35:03
categories:
- Java并发相关
tags: 
- Java
---

![学习.jpeg](https://upload-images.jianshu.io/upload_images/2824145-4e592f4f3b8d49b5.jpeg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

>该文章属于《Java并发编程》系列文章，如果想了解更多，请点击《Java并发编程之总目录》

### 前言
上篇文章我们讲了**volatile**关键字，我们大致了解了其为轻量级的同步机制，现在我们来讲讲我们关于同步的另一个兄弟**synchronized**。synchronized作为开发中常用的同步机制，也是我们处理线程安全的常用方法。相信大家对其都比较熟悉。但是对于其内部原理与底层代码实现大家有可能不是很了解，下面我就和大家一起彻底了解synchronized的使用方式与底层原理。

### 线程安全的问题
>线程安全的定义：当多个线程访问某个类时，不管运行时环境采用何种调度方式或者这些线程将如何交替执行，并且在主调代码中不需要任何额外的同步或协同，这个类都能表现出正确的行为，那么这个类就是线程安全的。

在具体讲解synchronized之前，我们需要了解一下什么是线程安全，为什么会出现线程线程不安全的问题。请看下列代码：

```
class ThreadNotSafeDemo {
    private static class Count {
        private int num;
        private void count() {
            for (int i = 1; i <= 10; i++) {
                num += i;
            }
            System.out.println(Thread.currentThread().getName() + "-" + num);
        }
    }
    public static void main(String[] args) {
        Runnable runnable = new Runnable() {
            Count count = new Count();
            public void run() {
                count.count();
            }
        };
		//创建10个线程，
        for (int i = 0; i < 10; i++) {
            new Thread(runnable).start();
        }
    }
}
```
上述代码中，我们创建Count类，在该类中有一个count()方法，计算从1一直加到10的和，在计算完后输出当前线程的名称与计算的结果，我们期望线程输出的结果是首项为55且等差为55的等差数列。但是结果并不是我们期望的。具体结果如下图所示：

![输出结果.png](https://upload-images.jianshu.io/upload_images/2824145-cfa91431d1aaf3b3.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

我们可以看见，线程并没有按照我们之间想的那样，线程按照从Thread-0到Thread-9依次排列，并且Thread-0与Thread-1线程输出的结果是错误的。

之所以会出现这样的情况，是CPU在调度的时候线程是可以交替执行的，具体来讲是因为当前线程Thread-0求和后，（求和后num值为55），在即将执行打印语句时，突然CPU开始调度执行Thread-1去执行count()方法，那么Thread-0就会停留在即将打印语句的位置，当Thread-1执行计算和后（求和后num值为100），这个时候CPU又开始调度Thread-0执行打印语句。则Thread-1开始暂停，而这个时候num值已经为110了，所以Thread-0打印输出的结果为110。

### 线程安全的实现方法
上面我们了解了之所以会出现线程安全的问题，主要原因就是因为存在多条线程共同操作共享数据，同时CPU的调度的时候线程是可以交替执行的。导致了程序的语义发生改变，所以会出现与我们预期的结果违背的情况。因此为了解决这个问题，在Java中提供了两种方式来处理这种情况。

#### 互斥同步(悲观锁)
互斥同步是指当存在多个线程操作共享数据时，需要保证同一时刻有且只有一个线程在操作共享数据，其他线程必须等到该线程处理完数据后再进行。

在Java中最基本的互斥同步就是**synchronized**(这里我们讨论的是jdk1.6之前，在jdk1.6之后Java团队对锁进行了优化，后面文章会具体描述)，也就是说当一个共享数据被当前正在访问的线程加上互斥锁后，在同一个时刻，其他线程只能处于等待的状态，直到当前线程处理完毕释放该锁。

除了synchronized之外，我们还可以使用java.util.concurrent包下的**ReentrantLock**来实现同步。
#### 非阻塞式同步（乐观锁）
互斥同步主要的问题就是进行线程阻塞和唤醒锁带来的性能问题，为了解决这性能问题，我们有另一种解决方案，当多个线程竞争某个共享数据时，没有获得锁的线程不会阻塞，而是不断的尝试去获取锁，直到成功为止。这种方案的原理就是使用**循环CAS操作**来实现。

### synchronized的三种使用方式
了解了synchronized的解决的问题，那么我们继续来看看在Java中在Java中synchronized的使用情况。

在Java中synchronized主要有三种使用的情况。下面分别列出了这几种情况
- 修饰普通的实例方法，对于普通的同步方法，锁式当前实例对象
- 修饰静态方法，对于静态同步方法，锁式当前类的Class对象
- 修饰代码块，对于同步方法块，锁是Synchronized配置的对象

#### 证明当前普通的同步方法，锁式当前实例对象
为了证明普通的同步方法中，锁是当前对象。请观察以下代码：
```
class SynchronizedDemo {

    public synchronized void normalMethod() {
        doPrint(5);
    }
 
    public void blockMethod() {//注意,同步块方法块中，配置的是当前类的对象
        synchronized (this) {
            doPrint(5);
        }
    }
	//打印当前线程信息与角标值
    private static void doPrint(int index) {
        while (index-- > 0) {
            System.out.println(Thread.currentThread().getName() + "--->" + index);
            try {
                Thread.sleep(500);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }

    public static void main(String[] args) {
        SynchronizedDemo demo = new SynchronizedDemo();
        new Thread(() -> demo.normalMethod(), "testNormalMethod").start();
        new Thread(() -> demo.normalMethod(), "testBlockMethod").start();
    }
 }
```
在上诉代码中，分别创建了两个方法，normalMethod（）与blockMethod（）方法，其中normalMethod()方法为普通的同步方法，blockMethod（）方法中，是一个同步块且配置的对象是当前类的对象。在Main()方法中，分别创建两个线程执行两个不同的方法。
##### 程序输出结果
![输出结果.png](https://upload-images.jianshu.io/upload_images/2824145-7616a244b81ac54f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
观察程序输出结果，我们可以看到normalMethod方法是由于blockMethod方法执行的，且blockMethod方法是在normalMethod方法执行完成之后在执行的。也就证明了我们的对于普通的同步方法锁式当前实例对象的结论。
#### 证明对于静态同步方法，锁式当前类的Class对象
```
class SynchronizedDemo {
    public void blockMethod() {
        synchronized (SynchronizedDemo.class) {//注意,同步块方法块中，配置的是当前类的Class对象
            doPrint(5);
        }
    }
    public static synchronized void staticMethod() {
        doPrint(5);
    }
    /**
     * 打印当前线程信息
     */
    private static void doPrint(int index) {
        while (index-- > 0) {
            System.out.println(Thread.currentThread().getName() + "--->" + index);
            try {
                Thread.sleep(500);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }
    public static void main(String[] args) {
        SynchronizedDemo demo = new SynchronizedDemo();
        new Thread(() -> demo.blockMethod(), "testBlockMethod").start();
        new Thread(() -> demo.staticMethod(), "testStaticMethod").start();
    }

}
```
在有了第一个结论的证明后，对于静态同步方法的锁对象就不再进行描述了（但是大家要注意一下，同步方法块中配置的对象是当前类的Class对象）。下面直接给出输出结果：
![TIM截图20180821140901.png](https://upload-images.jianshu.io/upload_images/2824145-b44650256c9ed7b6.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

观察结果，也很明显的证明了对于静态同步方法，锁式当前类的Class对象的结论


### Synchronized的原理
>下面文章主要是讲解jdk1.6之后Java团队对锁进行了优化之后的原理，优化之后涉及到偏向锁、轻量级锁、重量级锁。其中该文章都涉及jdk源码，这里把最新的jdk源码分享给大家----->[jdk源码](https://pan.baidu.com/s/1Lk9yp8cEpSAnLvw5NJdqZg)）


在了解Synchronized的原理的原理之前，我们需要知道三个知识点**第一个是CAS操作，**、**第二个是Java对象头（其中Synchronized使用的锁就在对象头中）**、**第三个是jdk1.6对锁的优化**。在了解以上三个知识点后，再去理解其原理就相对轻松一点。关于CAS操作已经在上篇文章《Java并发编程之Java CAS操作》进行过讲解，下面我们来讲解关于Java对象头与锁优化的知识点。

#### Java对象的内存布局
在Java虚拟机中，对象在内存的存储的布局可以分为3块区域：对象头（Header)、实例数据（Instance Data)、对其填充（Padding)。其中虚拟机中的对象头包括三部分信息，分别为"Mark Word"、类型指针、记录数组长度的数据（可选），具体情况如下图所示：

![对象存储结构.png](https://upload-images.jianshu.io/upload_images/2824145-81a3128a336b0a21.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

#### Java对象头的组成

- “Mark Word“：第一部分用于存储对象自身的运行时数据。如哈希码（HashCode）、GC分代年龄、**锁状态标志**、线**程持有的锁**、偏向锁ID、偏向锁时间戳等，这部分的数据在长度32位与64位的虚拟机中分别为32bit和64bit，官方称为“Mark Word"。
- 类型指针：对象头的另一部分是类型指针，即对象指向它的类元数据的指针，虚拟机通过这个指针来确定这个对象是哪个类的实例。（**Java SE 1.6中为了减少获得锁和释放锁带来的性能消耗而引入的偏向锁和轻量级锁**)
- 记录数组长度数据：对象头剩下的一部分是用于记录数组长度的数据（如果当前对象不是数组，就没有这一部分数据），如果对象是一个Java数组，那在对象头中还必须有一块用于记录数组长度的数据。因为虚拟机可以通过普通Java对象的元数据信息来确定Java对象的大小，但是从数组中的元数据中无法确定数组的大小。

####  “Mark Word“数据结构
其中关于"Mark Word"，因为存储对象头信息是与对象身定义的数据无关的额外的存储成本，考虑到虚拟机的空间效率，"Mark Word"被设计成一个被设计成一个非固定的数据结构以便在极小的空间存储尽量多的信息。它会根据对象的状态复用自己的存储区域。在JVM中，“Mark Word"的实现是在markOop.hpp文件中的markOopDesc类。通过注释我们大致了解”Mark Word"的结构，具体代码如下：

```
hash:保存对象的哈希码
age:GC分代年龄
biased_lock:偏向锁标志
lock:锁状态标志
JavaThread*  当前线程
epoch:保存偏向时间戳

//  32 bits:
//  --------
//             hash:25 ------------>| age:4    biased_lock:1 lock:2 (normal object)
//             JavaThread*:23 epoch:2 age:4    biased_lock:1 lock:2 (biased object)
// 省略部分代码


//  64 bits:
//  --------
//  unused:25 hash:31 -->| unused:1   age:4    biased_lock:1 lock:2 (normal object)
//  JavaThread*:54 epoch:2 unused:1   age:4    biased_lock:1 lock:2 (biased object)
// 省略部分代码

```

在上述代码中，分成了2种不同位数的操作系统，32位与64位。其中关于当前锁的状态标志markOopDesc类中也进行了详细的说明，具体代码如下：
```
  enum { locked_value             = 0,//轻量级锁 对应[00] 
         unlocked_value           = 1,//无锁状态  对应[01]
         monitor_value            = 2,//重量级锁 对应[10]
         marked_value             = 3,//GC标记  对应[11]
         biased_lock_pattern      = 5//是否是偏向锁  对应[101] 其中biased_lock一个bit位，lock两个bit位
  };
```
那么根据上述代码，我们以32位操作系统为例，可以生成如下两张表：
##### 在无锁状态下，32位JVM的“Mark Word"的默认存储结构
![无锁状态.png](https://upload-images.jianshu.io/upload_images/2824145-16e9628223a24026.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
在无锁状态下，“Mark Word“的32bit空间中，25bit用于存储对象哈希码，4bit用于存储对象分代年龄，2bit用于存储锁标志**（其中01标识当前线程为无锁状态）**，1bit固定为0。

##### 在有锁状态态下，32位JVM的“Mark Word"的默认存储结构
![有锁状态.png](https://upload-images.jianshu.io/upload_images/2824145-c834f8223e543eac.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
在有锁的状态下，23个bit位用于存储当前线程id,2个bit位用于存储偏向锁时间戳，4个bit为用于存储分代年龄（用于GC),1个bit位存储当前是否是偏向锁，最后的2bit用于当前锁的不同状态。其中00标识当前锁为轻量级锁，10标识为重量级锁，01标识当前锁为偏向锁。

### synchronized锁优化
Java SE 1.6为了减少获得锁和释放锁带来的性能消耗，引入了“偏向锁”和“轻量级锁”，在Java SE 1.6中，锁一共有4种状态，级别从低到高依次是：无锁状态、偏向锁状态、轻量级锁状态和重量级锁状态，这几个状态会随着竞争情况逐渐升级。锁可以升级但不能降级，意味着偏向锁升级成轻量级锁后不能降级成偏向锁。这种锁升级却不能降级的策略，目的是为了提高获得锁和释放锁的效率。下面会对各种锁进行介绍。
- **偏向锁**
在大多数情况下，锁不仅不存在多线程竞争，而且总是由同一线程多次获得，为了让线程获得锁的代价更低而引入了偏向锁，当一个线程访问同步块，并获取锁是，会在对象头中的“Mark word"和栈帧中的锁记录里存储锁偏向的线程ID。以后该线程在进入和退出同步块时，不需要进行CAS操作来加锁和解锁。只需简单地测试一下对象头的”Mark Word“里是否存储着指向当前线程的偏向锁。如果测试成功，表示线程已经获得了锁。如果测试失败，则需要再测试一下“Mark Word”中偏向锁的标识是否设置成1（表示当前是偏向锁）：如果没有设置，则使用CAS竞争锁；如果设置了，则尝试使用CAS将对象头的偏向锁指向当前线程。
- **轻量级锁**
线程在执行同步块之前，JVM会先在当前线程的栈桢中创建用于存储锁记录的空间，并将对象头中的Mark Word复制到锁记录中，官方称为**Displaced Mark Word**。然后线程尝试使用CAS将对象头中的Mark Word替换为指向锁记录的指针。如果成功，当前线程获得锁，如果失败，表示其他线程竞争锁，当前线程便尝试使用自旋来获取锁。
- **重量级锁**
轻量级解锁时，会使用原子的CAS操作将Displaced Mark Word替换回到对象头，如则表示没有竞争发生。如果失败，表示当前锁存在竞争，锁就会膨胀成重量级锁，重量级锁会导致竞争的线程**互斥同步。**
### synchronized底层代码实现
在了解了上述知识点后，我们来了解一下synchronized底层代码实现。从JVM规范中可以看到Synchonized在JVM里的实现原理，JVM基于进入和退出Monitor对象来实现方法同步和代码块同步，但两者的实现细节不一样。**代码块同步是使用monitorenter和monitorexit指令实现的**，而**方法同步是使用字节码同步指令ACC_SYNCHRONIZED来实现**的，细节在JVM规范里并没有详细说明。但是方法的同步同样可以使用这两个指令来实现。那我们这里我们就以synchronized代码块底层原理来进行讲解。

>**字节码同步指令ACC_SYNCHRONIZED原理**：JVM通过使用管程（Monitor)来支持同步，JVM可以从方法常量池的方法表结构中的ACC_SYNCHRONIZED访问标志来得知一个方法是否声明为同步方法，当方法调用时，调用指令将会检查方法的ACC_SYNCHRONIZED访问标志是否被设置，如果设置了，执行线程就要求先成功持有管程（Monitor)，然后才能执行方法，最后当方法完成（无论是正常完成还是非正常完成）时释放管程，在方法执行期间，执行线程持有了管程，其他任何线程都无法在获取到同一个管程。

### synchronized代码块底层原理
在了解 synchronized代码块底层原理之前，我们先了解我们常用的synchronized代码块使用方式。
```
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
然后我们通过javap指令反编译得到字节码。
```
 //===========主要看看syncTask方法实现================
  public void syncTask();
    descriptor: ()V
    flags: ACC_PUBLIC
    Code:
      stack=3, locals=3, args_size=1
         0: aload_0
         1: dup
         2: astore_1
         3: monitorenter  //注意此处，进入同步方法
         4: aload_0
         5: dup
         6: getfield      #2             // Field i:I
         9: iconst_1
        10: iadd
        11: putfield      #2            // Field i:I
        14: aload_1
        15: monitorexit   //注意此处，退出同步方法
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
从上诉代码中，我们可以明白当我们声明synchronized代码块的时候，编译器会我们生产相应的monitorenter  与monitorexit 指令。当我们的JVM把字节码加载到内存的时候，会对这两个指令进行解析。其中关于monitorenter 与monitorenter的指令解析是通过InterpreterRuntime.cpp文件中的**InterpreterRuntime::monitorenter**与**InterpreterRuntime::monitorexit**两个函数分别实现的。

- InterpreterRuntime::monitorenter(JavaThread* thread, BasicObjectLock* elem)
- InterpreterRuntime::monitorexit(JavaThread* thread, BasicObjectLock* elem)

在了解具体的方法实现之间，我们需要了解两个参数信息，第一个参数猜都都猜出来，当前线程的指针，第二个参数为BasicObjectLock类型的指针，那我们来看看BasicObjectLock到底是什么东西。

#### BasicObjectLock
关于BasicObjectLock的类具体声明是在basicLock.hpp文件下。
```
class BasicLock {
  friend class VMStructs;
  friend class JVMCIVMStructs;
 private:
  //指向"Mark Word“也就是我们提到过的markOopDesc的指针。这里markOop是markOopDesc的别名
  volatile markOop _displaced_header;
 public:
  //获取"Mark Word“也就是我们提到过的markOopDesc，这里markOop是markOopDesc的别名
  markOop      displaced_header() const               { return _displaced_header; }
  void         set_displaced_header(markOop header)   { _displaced_header = header; }
  //省略部分代码
};

class BasicObjectLock {
  friend class VMStructs;
 private:
  BasicLock _lock; //拥有BasicLock对象
  oop       _obj;                                    
  //省略部分代码
};
```
从该文件中，我们知道在BasicLock类中指向"Mark Word“的指针，同时BasicObjectLock 也拥有BasicLock对象，那么BasicObjectLock 就能访问”Mark Word“中的内容了。那现在我们再来看上面提到的两个对应的方法。

#### InterpreterRuntime::monitorenter方法
```
IRT_ENTRY_NO_ASYNC(void, InterpreterRuntime::monitorenter(JavaThread* thread, BasicObjectLock* elem))
  //省略部分代码
  if (UseBiasedLocking) {//判断是否使用偏向锁
	//如果是使用偏向锁，则进入快速获取，避免不必要的膨胀。
    ObjectSynchronizer::fast_enter(h_obj, elem->lock(), true, CHECK);
  } else {//否则直接走轻量级锁的获取
    ObjectSynchronizer::slow_enter(h_obj, elem->lock(), CHECK);
  }
  //省略部分代码
```
当monitorenter方法执行时，会先判断当前是否开启偏向锁（偏向锁在Java 6和Java 7里是默认启用的，但是它在应用程序启动几秒钟之后才激活，如有必要可以使用JVM参数来关闭延迟：-XX:BiasedLockingStartupDelay=0。如果你确定应用程序里所有的锁通常情况下处于竞争状态，可以通过JVM参数关闭偏向锁：-XX:-UseBiasedLocking=false，那么程序默认会进入轻量级锁状态），如果没有开启会直接走轻量级锁的获取，也就是slow_enter（）方法。

##### 偏向锁的获取
ObjectSynchronizer::fast_enter（）方法是在sychronizer.cpp文件进行声明的，具体代码如下：
```
void ObjectSynchronizer::fast_enter(Handle obj, BasicLock* lock,
                                    bool attempt_rebias, TRAPS) {
  if (UseBiasedLocking) {//如果使用偏向锁
    if (!SafepointSynchronize::is_at_safepoint()) {//如果不在安全点，获取当前偏向锁的状态（可能撤销与重偏向）
      BiasedLocking::Condition cond = BiasedLocking::revoke_and_rebias(obj, attempt_rebias, THREAD);
      if (cond == BiasedLocking::BIAS_REVOKED_AND_REBIASED) {//如果是撤销与重偏向直接返回
        return;
      }
    } else {//如果在安全点，有可能会撤销偏向锁
      assert(!attempt_rebias, "can not rebias toward VM thread");
      BiasedLocking::revoke_at_safepoint(obj);
    }
   //省略部分代码
  }
  slow_enter(obj, lock, THREAD);//如果不使用偏向锁，则走轻量级锁的获取
}
```
在该方法中如果当前JVM支持偏向锁，会需要等待全局安全点（在这个时间点上没有正在执行的字节码），如果当前不在安全点中，会调用revoke_and_rebias（）方法来获取当前偏向锁的状态（可能为撤销或撤销后重偏向）。如果在安全点，会根据当前偏向锁的状态来判断是否需要撤销偏向锁。**其中revoke_and_rebias（）方法是在biasedLocking.cpp中进行声明的。**

#### BiasedLocking::revoke_and_rebias（）方法
```
BiasedLocking::Condition BiasedLocking::revoke_and_rebias(Handle obj, bool attempt_rebias, TRAPS) {
  assert(!SafepointSynchronize::is_at_safepoint(), "must not be called while at safepoint");
  markOop mark = obj->mark();
  
  //第一步，如果没有其他线程占用该对象（mark word中线程id为0，后三位为101，且不尝试重偏向)
  //这里“fast enter（）方法"传入的attempt_rebias为true
  if (mark->is_biased_anonymously() && !attempt_rebias) {
    //一般来讲，只有在重新计算对象hashCode的时候才会进入该分支，
    //所以直接用用CAS操作将对象设置为无锁状态
    markOop biased_value       = mark;
    markOop unbiased_prototype = markOopDesc::prototype()->set_age(mark->age());
    markOop res_mark = obj->cas_set_mark(unbiased_prototype, mark);//cas 操作从新设置偏向锁的状态
    if (res_mark == biased_value) {//如果CAS操作失败，说明存在竞争，偏向锁为撤销状态
      return BIAS_REVOKED;
    }
  } else if (mark->has_bias_pattern()) {
    //第二步，判断当前偏向锁是否已经锁定(不管mark word中线程id是否为null),尝试重偏向
    Klass* k = obj->klass();
    markOop prototype_header = k->prototype_header();
    if (!prototype_header->has_bias_pattern()) {
     //第三步如果有线程对该对象进行了全局锁定（即同步了静态方法/属性），则取消偏向操作
      markOop biased_value       = mark;
      markOop res_mark = obj->cas_set_mark(prototype_header, mark);
      assert(!obj->mark()->has_bias_pattern(), "even if we raced, should still be revoked");
      return BIAS_REVOKED;//偏向锁为撤销状态
    } else if (prototype_header->bias_epoch() != mark->bias_epoch()) {  //第四步，如果偏向锁时间过期，（这个时候有另一个线程通过偏向锁获取到了这个对象的锁）
      if (attempt_rebias) {//第五步，如果偏向锁开启，重新通过cas操作更新时间戳与分代年龄。
        assert(THREAD->is_Java_thread(), "");
        markOop biased_value       = mark;
        markOop rebiased_prototype = markOopDesc::encode((JavaThread*) THREAD, mark->age(), prototype_header->bias_epoch());
        markOop res_mark = obj->cas_set_mark(rebiased_prototype, mark);
        if (res_mark == biased_value) {
          return BIAS_REVOKED_AND_REBIASED;//撤销偏移后重偏向。
        }
      } else {//第六步，如果偏向锁关闭，通过CAS操作更新分代年龄
        markOop biased_value       = mark;
        markOop unbiased_prototype = markOopDesc::prototype()->set_age(mark->age());
        markOop res_mark = obj->cas_set_mark(unbiased_prototype, mark);
        if (res_mark == biased_value) {
          return BIAS_REVOKED;////如果CAS操作失败，说明存在竞争，偏向锁为撤销状态
        }
      }
    }
  }
 //省略部分代码...
}
```
偏向锁的获取由BiasedLocking::revoke_and_rebias方法实现，主要分为五个步骤
1. 第一步，判断当前偏向锁中"Mark word"中线程id是否为null，且attempt_rebias =false。如果满足条件，尝试通过CAS操作将当前对象设置为无锁状态。如果CAS操作失败，说明存在竞争，偏向锁为撤销状态。
2. 第二步，判断当前偏向锁是否已经锁定（不管mark word中线程id是否为null）,会根据当前条件走第三、第四、第五步。
3. 第三步，如果有线程对该对象进行了全局锁定（即同步了静态方法/属性），偏向锁为撤销状态。
4. 第四步，判断偏向锁时间是否过期（这个时候有另一个线程通过偏向锁获取到了这个对象的锁），接着走第五步、第六步的条件判断
5. 第五步，在偏向锁时间过期的条件下，如果偏向锁开启，那么通过CAS操作更新时间戳与分代年龄、线程ID，如果失败,表明该对象的锁状态已经从撤销偏向到了另一线程。当前偏向锁的状态为撤销后重偏向。
6. 第六步，在偏向锁时间过期的条件下，如果偏向锁默认关闭，那么通过CAS操作更新分代年龄，如果失败，说明存在线程的竞争，偏向锁为撤销状态。

#### 偏向锁的撤销
在上文中我们提到了在调用fast_enter（）方法时，如果在安全点，这时会根据偏向锁的状态来判断是否需要撤销偏向锁，也就是调用revoke_at_safepoint（）方法。其中该方法也是在biasedLocking.cpp中进行声明的，具体代码如下：
```
void BiasedLocking::revoke_at_safepoint(Handle h_obj) {
  assert(SafepointSynchronize::is_at_safepoint(), "must only be called while at safepoint");
  oop obj = h_obj();
  HeuristicsResult heuristics = update_heuristics(obj, false);//获得偏向锁偏向与撤销的次数
  if (heuristics == HR_SINGLE_REVOKE) {//如果是一次撤销
    revoke_bias(obj, false, false, NULL, NULL);
  } else if ((heuristics == HR_BULK_REBIAS) ||//如果是多次撤销或多次偏向
             (heuristics == HR_BULK_REVOKE)) {
    bulk_revoke_or_rebias_at_safepoint(obj, (heuristics == HR_BULK_REBIAS), false, NULL);
  }
  clean_up_cached_monitor_info();
}
```
观察代码我们可以发现，会根据当前偏向锁偏向与撤销的次数走不同的方法。这里我们以revoke_bias()方法为例，来进行讲解。具体代码如下：
```
static BiasedLocking::Condition revoke_bias(oop obj, bool allow_rebias, bool is_bulk, JavaThread* requesting_thread, JavaThread** biased_locker) {
  //省略部分代码...
  uint age = mark->age();
  markOop   biased_prototype = markOopDesc::biased_locking_prototype()->set_age(age);
  markOop unbiased_prototype = markOopDesc::prototype()->set_age(age);
  
  JavaThread* biased_thread = mark->biased_locker();
  if (biased_thread == NULL) {//判断当前偏向锁中，偏向线程id是否为null
    if (!allow_rebias) {//如果不允许重偏向，则使其偏向锁不可用。
      obj->set_mark(unbiased_prototype);
    }
	//省略部分代码...
    return BiasedLocking::BIAS_REVOKED;
  }

 //判断当前偏向锁偏向的线程是否存在
  bool thread_is_alive = false;
  if (requesting_thread == biased_thread) {
    thread_is_alive = true;
  } else {
    ThreadsListHandle tlh;
    thread_is_alive = tlh.includes(biased_thread);
  }
  if (!thread_is_alive) {//如果当前偏向锁偏向的线程不存活
    if (allow_rebias) {
      obj->set_mark(biased_prototype);//如果允许偏向，则将偏向锁中的 线程id置为null
    } else {
      obj->set_mark(unbiased_prototype);//否则，将偏向锁设置为无锁状态 也就是01
    }
    return BiasedLocking::BIAS_REVOKED;
  }

  //遍历当前锁记录，找到拥有锁的线程，将需要的displaced headers 写到线程堆栈中。
  GrowableArray<MonitorInfo*>* cached_monitor_info = get_or_compute_monitor_info(biased_thread);
  BasicLock* highest_lock = NULL;
  for (int i = 0; i < cached_monitor_info->length(); i++) {
    MonitorInfo* mon_info = cached_monitor_info->at(i);
    if (oopDesc::equals(mon_info->owner(), obj)) {
      markOop mark = markOopDesc::encode((BasicLock*) NULL);
      highest_lock = mon_info->lock();
      highest_lock->set_displaced_header(mark);//将dispalece headers 写入堆栈中
    } 	
    //省略部分代码...
  }
  if (highest_lock != NULL) {//将需要的displaced headers 写到线程堆栈
   //省略部分代码...
    highest_lock->set_displaced_header(unbiased_prototype);
   //省略部分代码...
    obj->release_set_mark(markOopDesc::encode(highest_lock));
 
  //省略部分代码...
  } else {//将对象的头恢复到未锁定或无偏状态
     //省略部分代码...
    if (allow_rebias) {
      obj->set_mark(biased_prototype);
    } else {
      // Store the unlocked value into the object's header.
      obj->set_mark(unbiased_prototype);
    }
  }
  //获取偏向锁指向的线程
  if (biased_locker != NULL) {
    *biased_locker = biased_thread;
  }

  return BiasedLocking::BIAS_REVOKED;
}
```
在偏向锁的撤销，需要等待全局全局点（这个时间点没有在执行的字节码），它会首先暂停拥有偏向锁的线程，然后检查持有偏向锁的线程是否活着，如果线程不处于活动状态。会更将偏向锁设置为无锁状态，如果线程仍然活着，拥有偏向锁的栈
会被执行，遍历偏向对象的锁记录，栈中的锁记录和对象头的Mark Word要么重新偏向于其他线程，要么恢复到无锁或者标记对象不适合作为偏向锁，最后唤醒暂停的线程。

#### 轻量级锁的获取
在上文中我们说过当monitorenter指令执行时，如果当前偏向锁没有开启或多个线程竞争偏向锁导致偏向锁升级为轻量级锁时，那么会直接走轻量级的锁的获取。在讲解轻量级锁的获取之前，需要讲解一个知识点”Displaced Mark Word"。

##### 轻量级锁获与“Displaced Mark Word”
在代码进入同步块，执行轻量级锁获取之前，如果此同步对象没有被锁定（锁标志为01状态），JVM会在当前线程的帧栈中建立一个名为锁记录（Lock Record)的空间，用于存储对象目前的"Mark Word"的拷贝（官方把这份拷贝加了一个Displaced前缀，及Displaced Mark Word）。虚拟机将使用CAS操作尝试将对象的“Mark word"更新为指向Lock Record的指针，如果这个更新动作成功了，那么这个现场就拥有了该对象的锁，及该对象处于轻量级锁定状态。关于轻量级锁的获取，具体示意图如下：

![轻量级锁获取示意图.png](https://upload-images.jianshu.io/upload_images/2824145-2b7a9f4de35a2dee.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

##### ObjectSynchronizer::slow_enter（）方法
在了解了具体的轻量级锁获取流程后，我们来查看具体的实现slow_enter()方法。该方法是在sychronizer.cpp文件进行声明的。具体代码如下：
```
void ObjectSynchronizer::slow_enter(Handle obj, BasicLock* lock, TRAPS) {

  markOop mark = obj->mark();//第一步 获取锁对象的“mark word"
  
  if (mark->is_neutral()) {//第二步，判断当前锁是否是无锁状态 后两位标志位为01
    //第三步，如果是无锁状态，存储对象目前的“mark word"拷贝，
    //通过CAS尝试将锁对象Mark Word更新为指向lock Record对象的指针，
    lock->set_displaced_header(mark);
    if (mark == obj()->cas_set_mark((markOop) lock, mark)) {
      TEVENT(slow_enter: release stacklock); //如果更新成功，表示获得锁，则执行同步代码，
      return;
    }
  }
  //第四步，如果当前mark处于加锁状态，且线程帧栈中的owner指向当前锁，则执行同步代码， 
  else if (mark->has_locker() &&
             THREAD->is_lock_owned((address)mark->locker())) {
    lock->set_displaced_header(NULL);
    return;
  }
  //第五步，否则说明有多个线程竞争轻量级锁，轻量级锁需要膨胀升级为重量级锁；
  lock->set_displaced_header(markOopDesc::unused_mark());
  ObjectSynchronizer::inflate(THREAD,
                              obj(),
                              inflate_cause_monitor_enter)->enter(THREAD);
}
```
在轻量级锁的获取中，主要分为五个步骤，主要步骤如下：
1. 第一步：markOop mark = obj->mark()方法获取锁对象的markOop数据mark。
2. 第二步：mark->is_neutral()方法判断mark是否为无锁状态：mark的偏向锁标志位为 0，锁标志位为 01。
3. 第三步：如果处于无锁状态，存储对象目前的“Mark Word"拷贝,通过CAS尝试将锁对象的“Mark Word”更新为指向lock Record对象的指针，如果更新成功，表示竞争到锁，则执行同步代码。
4. 第四步：如果处于有锁状态，且线程帧栈中的owner指向当前锁，则执行同步代码， 
5. 第五步：如果都不满足，否则说明有多个线程竞争轻量级锁，轻量级锁需要膨胀升级为重量级锁。

适用情形：假设线程A和B同时执行到临界区if (mark->is_neutral())：
1. 线程AB都把Mark Word复制到各自的lock Record空间中，该数据保存在线程的栈帧上，是线程私有的；
2. 通过CAS操作保证只有一个线程可以把指向栈帧的指针复制到Mark Word，假设此时线程A执行成功，并返回继续执行同步代码块。
3. 线程B执行失败，退出临界区，通过ObjectSynchronizer::inflate方法开始膨胀锁（将轻量级锁膨胀为重量级锁）

#### 轻量级锁的撤销
在上文中，我们讲过当走完同步块的时候，会执行monitorexit指令，而轻量级锁的释放这正是在monitorexit执行的时候，也就是InterpreterRuntime::monitorexit（）。
```
IRT_ENTRY_NO_ASYNC(void, InterpreterRuntime::monitorexit(JavaThread* thread, BasicObjectLock* elem))
#ifdef ASSERT
  thread->last_frame().interpreter_frame_verify_monitor(elem);
#endif
  Handle h_obj(thread, elem->obj());
  assert(Universe::heap()->is_in_reserved_or_null(h_obj()),
         "must be NULL or an object");
  if (elem == NULL || h_obj()->is_unlocked()) {
    THROW(vmSymbols::java_lang_IllegalMonitorStateException());
  }
  ObjectSynchronizer::slow_exit(h_obj(), elem->lock(), thread);
  // Free entry. This must be done here, since a pending exception might be installed on
  // exit. If it is not cleared, the exception handling code will try to unlock the monitor again.
  elem->set_obj(NULL);
#ifdef ASSERT
  thread->last_frame().interpreter_frame_verify_monitor(elem);
#endif
IRT_END
```
在monitorexit（）方法中内部会调用slow_exit（）方法而slow_exit()方法内部会调用fast_exit（）方法，我们查看fast_exit（）方法。

```
void ObjectSynchronizer::fast_exit(oop object, BasicLock* lock, TRAPS) {
  markOop mark = object->mark();
  //省略部分代码...
  markOop dhw = lock->displaced_header();//获取线程堆栈中的Displaced Mark Word
  if (dhw == NULL) {//如果线程堆栈中的Displaced Mark Word为null
	#ifndef PRODUCT
    if (mark != markOopDesc::INFLATING()) {//如果当前轻量级锁不是在膨胀为重量级锁
      //省略部分代码...
      if (mark->has_monitor()) {//如果已经为重量级锁，直接返回
        ObjectMonitor * m = mark->monitor();
        assert(((oop)(m->object()))->mark() == mark, "invariant");
        assert(m->is_entered(THREAD), "invariant");
      }
    }
#endif
    return;
  }
 
  //如果当前线程拥有轻量级锁，那么通过CAS尝试把Displaced Mark Word替换到当前锁对象的Mark Word，
  //如果CAS成功，说明成功的释放了锁
  if (mark == (markOop) lock) {
    assert(dhw->is_neutral(), "invariant");
    if (object->cas_set_mark(dhw, mark) == mark) {
      TEVENT(fast_exit: release stack-lock);
      return;
    }
  }

  //如果CAS操作失败，说明其他线程在尝试获取轻量级锁，这个时候需要将轻量级锁升级为重量级锁
  ObjectSynchronizer::inflate(THREAD,
                              object,
                              inflate_cause_vm_internal)->exit(true, THREAD);
}
```
在偏向锁的释放中，会经历一下几个步骤。
1. 获取线程堆栈中的Displaced Mark Word
2. 如果线程堆栈中的Displaced Mark Word为null，如果已经为重量级锁，直接返回。
3. 如果当前线程拥有轻量级锁，那么通过CAS尝试把Displaced Mark Word替换到当前锁对象的Mark Word，如果CAS成功，说明成功的释放了锁
4. 如果CAS操作失败，说明其他线程在尝试获取轻量级锁，这个时候需要将轻量级锁升级为重量级锁。

#### 重量级锁的获取
在上文中我们提到过，在多个线程进行轻量级锁的获取时或在轻量级锁撤销时，有肯能会膨胀为重量级锁，那现在我们就来看看膨胀的具体过程
```
ObjectMonitor* ObjectSynchronizer::inflate(Thread * Self,
                                           oop object,
                                           const InflateCause cause) {
  EventJavaMonitorInflate event;

  for (;;) {//开始自旋
    const markOop mark = object->mark();
    // The mark can be in one of the following states:
    // *  Inflated     - just return
    // *  Stack-locked - coerce it to inflated
    // *  INFLATING    - busy wait for conversion to complete
    // *  Neutral      - aggressively inflate the object.
    // *  BIASED       - Illegal.  We should never see this

    //1.如果当前锁已经为重量级锁了，直接返回
    if (mark->has_monitor()) {
      ObjectMonitor * inf = mark->monitor();
      return inf;
    }


    //2.如果正在膨胀的过程中，在完成膨胀过程中，其他线程必须等待。
    if (mark == markOopDesc::INFLATING()) {
      TEVENT(Inflate: spin while INFLATING);
      ReadStableMark(object);
      continue;
    }

	//3.如果当前为轻量级锁，迫使其膨胀为重量级锁
    if (mark->has_locker()) {
      ObjectMonitor * m = omAlloc(Self);
      m->Recycle();
      m->_Responsible  = NULL;
      m->_recursions   = 0;
      m->_SpinDuration = ObjectMonitor::Knob_SpinLimit;   // Consider: maintain by type/class

      markOop cmp = object->cas_set_mark(markOopDesc::INFLATING(), mark);
      if (cmp != mark) {
        omRelease(Self, m, true);
        continue;       // Interference -- just retry
      }

      markOop dmw = mark->displaced_mark_helper();
      assert(dmw->is_neutral(), "invariant");

      m->set_header(dmw);

     
      m->set_owner(mark->locker());
      m->set_object(object);
      // TODO-FIXME: assert BasicLock->dhw != 0.

      // Must preserve store ordering. The monitor state must
      // be stable at the time of publishing the monitor address.
      guarantee(object->mark() == markOopDesc::INFLATING(), "invariant");
      object->release_set_mark(markOopDesc::encode(m));

      // Hopefully the performance counters are allocated on distinct cache lines
      // to avoid false sharing on MP systems ...
      OM_PERFDATA_OP(Inflations, inc());
      TEVENT(Inflate: overwrite stacklock);
      if (log_is_enabled(Debug, monitorinflation)) {
        if (object->is_instance()) {
          ResourceMark rm;
          log_debug(monitorinflation)("Inflating object " INTPTR_FORMAT " , mark " INTPTR_FORMAT " , type %s",
                                      p2i(object), p2i(object->mark()),
                                      object->klass()->external_name());
        }
      }
      if (event.should_commit()) {
        post_monitor_inflate_event(&event, object, cause);
      }
      return m;
    }

	//4.如果为无锁状态，重置监视器状态
    assert(mark->is_neutral(), "invariant");
    ObjectMonitor * m = omAlloc(Self);
    // prepare m for installation - set monitor to initial state
    m->Recycle();
    m->set_header(mark);
    m->set_owner(NULL);
    m->set_object(object);
    m->_recursions   = 0;
    m->_Responsible  = NULL;
    m->_SpinDuration = ObjectMonitor::Knob_SpinLimit;       // consider: keep metastats by type/class

    if (object->cas_set_mark(markOopDesc::encode(m), mark) != mark) {
      m->set_object(NULL);
      m->set_owner(NULL);
      m->Recycle();
      omRelease(Self, m, true);
      m = NULL;
      continue;
 
    }
  //省略部分代码...
   return m ;
}
```
在轻量级锁膨胀为重量级锁大致可以分为以下几个过程
1. 如果当前锁已经为重量级锁了，直接返回ObjectMonitor 对象。
2. 如果正在膨胀的过程中，在完成膨胀过程中，其他线程自旋等待。这里需要注意一点，虽然是自旋操作，但不会一直占用cpu资源，会通过spin/yield/park方式挂起线程。
3. 如果当前为轻量级锁，迫使其膨胀为重量级锁
4. 如果是无锁，重置ObjectMonitor 中的状态。

### 锁升级示意图
在了解了偏向锁、轻量级锁，与重量级锁的原理后，现在我们来总结一下整个锁升级的流程。具体如下图所示：
#### 偏向锁获得和撤销
![偏向锁获得和撤销示意图.png](https://upload-images.jianshu.io/upload_images/2824145-baf96b535929b672.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

#### 轻量级锁膨胀流程图
![轻量级锁膨胀示意图.png](https://upload-images.jianshu.io/upload_images/2824145-11f9e59e6e2af1e8.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


### 重量级锁的竞争
在上文中，我们主要介绍了整个锁升级的流程与源代码实现。而真正线程的等待与竞争我们还没有详细描述。下面我们就来讲讲当锁膨胀为重量级锁的时候，整个线程的竞争与等待过程。重量级锁的竞争是在objectMonitor.cpp中ObjectMonitor::enter()方法中实现的。

#### ObjectMonitor结构
在讲解具体的锁获取之前，我们需要了解**每个锁对象（这里指已经升级为重量级锁的对象）都有一个ObjectMonitor（对象监视器）**。也就是说每个线程获取锁对象都会通过ObjectMonitor。代码如下所示：（这里我省略了一些不必要的属性。大家只需要看一些关键的结构）
```
class ObjectMonitor {
 public:
  enum {
    OM_OK,                    // 没有错误
    OM_SYSTEM_ERROR,          // 系统错误
    OM_ILLEGAL_MONITOR_STATE, // 监视器状态异常
    OM_INTERRUPTED,           // 当前线程已经中断
    OM_TIMED_OUT              // 线程等待超时
  };
  volatile markOop   _header;       // 线程帧栈中存储的 锁对象的mark word拷贝

 protected:                         // protected for JvmtiRawMonitor
  void *  volatile _owner;          // 指向获得objectMonitor的线程或者 BasicLock对象
  volatile jlong _previous_owner_tid;  // 上一个获得objectMonitor的线程id
  volatile intptr_t  _recursions;   // 同一线程重入锁的次数，如果是0，表示第一次进入
  ObjectWaiter * volatile _EntryList; // 在进入或者重进入阻塞状态下的线程链表
                             
 protected:
  ObjectWaiter * volatile _WaitSet; // 处于等待状态下的线程链表
  volatile jint  _waiters;          //处于等待状态下的线程个数

```

#### 重量级锁的获取
在了解了ObjectMonitor 类中具体结构后，来看看具体的锁获取方法ObjectMonitor::enter（），具体代码如下所示：
```
void ObjectMonitor::enter(TRAPS) {

  Thread * const Self = THREAD;//当前进入enter方法的线程
 
 //通过CAS操作尝试吧monitor的_owner（ 指向获得objectMonitor的线程或者 BasicLock对象）设置为当前线程
  void * cur = Atomic::cmpxchg(Self, &_owner, (void*)NULL);
  
  if (cur == NULL) {//如果成功，当前线程获取锁成功，直接执行同步代码块
    assert(_recursions == 0, "invariant");
    assert(_owner == Self, "invariant");
    return;
  }
  
  //如果是同一线程，则记录当前重入的次数（上一步CAS操作不管成功还是失败，都会返回_owner指向的地址)
  if (cur == Self) {
    _recursions++;
    return;
  }

  //如果之前_owner指向的BasicLock在当前线程栈上，说明当前线程是第一次进入该monitor，
  //设置_recursions为1，_owner为当前线程，该线程成功获得锁并返回；
  if (Self->is_lock_owned ((address)cur)) {
    assert(_recursions == 0, "internal state error");
    _recursions = 1;
    _owner = Self;
    return;
  }
  //省略部
  分代码...
  
  //开始竞争锁
    for (;;) {
      jt->set_suspend_equivalent();
      EnterI(THREAD);
      if (!ExitSuspendEquivalent(jt)) break;
      _recursions = 0;
      _succ = NULL;
      exit(false, Self);
      jt->java_suspend_self();
    }
    Self->set_current_pending_monitor(NULL);

  }
//省略部分代码...
}
```
在重量级级锁的竞争步骤，主要分为以下几个步骤：
1. 通过CAS操作尝试吧monitor的_owner（ 指向获得objectMonitor的线程或者 BasicLock对象）设置为当前线程，如果CAS操作成功，表示线程获取锁成功，直接执行同步代码块。
2. 如果是同一线程重入锁，则记录当前重入的次数。
3. 如果2,3步骤都不满足，则开始竞争锁，走EnterI()方法。

##### EnterI()方法实现
```
void ObjectMonitor::EnterI(TRAPS) {
  Thread * const Self = THREAD;
  //省略部分代码...
  
  //把当前线程被封装成ObjectWaiter的node对象，状态设置成ObjectWaiter::TS_CXQ；
  ObjectWaiter node(Self);
  Self->_ParkEvent->reset();
  node._prev   = (ObjectWaiter *) 0xBAD;
  node.TState  = ObjectWaiter::TS_CXQ;//TS_CXQ：为竞争锁状态

 //在for循环中，通过CAS把node节点push到_cxq链表中；
  ObjectWaiter * nxt;
  for (;;) {
    node._next = nxt = _cxq;
    //如果CAS操作失败，继续尝试，是因为当期_cxq链表已经发生改变了
    if (Atomic::cmpxchg(&node, &_cxq, nxt) == nxt) break;
	//有可能在放入_cxq链表中时，已经获取到锁了，直接返回
    if (TryLock (Self) > 0) {
      assert(_succ != Self, "invariant");
      assert(_owner == Self, "invariant");
      assert(_Responsible != Self, "invariant");
      return;
    }
  }
 //将node节点push到_cxq链表之后，通过自旋尝试获取锁
  for (;;) {

    if (TryLock(Self) > 0) break;//尝试获取锁
    assert(_owner != Self, "invariant");

    if ((SyncFlags & 2) && _Responsible == NULL) {
      Atomic::replace_if_null(Self, &_Responsible);
    }
    //判断执行循环的次数，如果执行相应循环后，如果还是没有获取到锁，则通过park函数将当前线程挂起，等待被唤醒
    if (_Responsible == Self || (SyncFlags & 1)) {
      TEVENT(Inflated enter - park TIMED);
      Self->_ParkEvent->park((jlong) recheckInterval);
      // Increase the recheckInterval, but clamp the value.
      recheckInterval *= 8;
      if (recheckInterval > MAX_RECHECK_INTERVAL) { 其中MAX_RECHECK_INTERVAL为1000
        recheckInterval = MAX_RECHECK_INTERVAL;
      }
    } else {
      TEVENT(Inflated enter - park UNTIMED);
      Self->_ParkEvent->park();
    }
	//省略部分代码...
    OrderAccess::fence();
  }
 //省略部分代码...
  return;
}
```
关于EnterI()方法，可以分为以下步骤：
1. 把当前线程被封装成ObjectWaiter的node对象，同时将该线程状态设置为TS_CXQ（竞争状态）
2. 在for循环中，通过CAS把node节点push到_cxq链表中，如果CAS操作失败，继续尝试，是因为当期_cxq链表已经发生改变了继续for循环，如果成功直接返回。
3. 将node节点push到_cxq链表之后，通过自旋尝试获取锁（TryLock方法获取锁)，如果循环一定次数后，还获取不到锁，则通过park函数挂起。（并不会消耗CPU资源）

关于获取锁的TryLock方法如下所示：
##### TryLock方法
```
int ObjectMonitor::TryLock(Thread * Self) {
  void * own = _owner;
  if (own != NULL) return 0;
  if (Atomic::replace_if_null(Self, &_owner)) {
    return 1;
  }
  return -1;
}
```
该函数其实很简单，就是将锁中的_owner指针指向当前线程，如果成功返回1，反之返回-1。
#### 重量级锁的释放

```
void ObjectMonitor::exit(bool not_suspended, TRAPS) {
  Thread * const Self = THREAD;
  if (THREAD != _owner) {//如果当前锁对象中的_owner没有指向当前线程
    if (THREAD->is_lock_owned((address) _owner)) {
      //但是_owner指向的BasicLock在当前线程栈上,那么将_owner指向当前线程
      assert(_recursions == 0, "invariant");
      _owner = THREAD;
      _recursions = 0;
    } else {
	  //省略部分代码...
      return;
    }
  }
  
  //如果当前，线程重入锁的次数，不为0，那么就重新走ObjectMonitor::exit，直到重入锁次数为0为止
  if (_recursions != 0) {
    _recursions--;        // this is simple recursive enter
    TEVENT(Inflated exit - recursive);
    return;
  }
  //省略部分代码...
  for (;;) {

    if (Knob_ExitPolicy == 0) {
      OrderAccess::release_store(&_owner, (void*)NULL);   //释放锁
      OrderAccess::storeload();                        // See if we need to wake a successor
      if ((intptr_t(_EntryList)|intptr_t(_cxq)) == 0 || _succ != NULL) {
        TEVENT(Inflated exit - simple egress);
        return;
      }
      TEVENT(Inflated exit - complex egress);
    //省略部分代码...
    }
    //省略部分代码...
    ObjectWaiter * w = NULL;
    int QMode = Knob_QMode;
	
	//根据QMode的模式判断，
	
    //如果QMode == 2则直接从_cxq挂起的线程中唤醒	
    if (QMode == 2 && _cxq != NULL) {
      w = _cxq;
      ExitEpilog(Self, w);
      return;
	    }
     //省略部分代码... 省略的代码为根据QMode的不同，不同的唤醒机制
	 }
   } 
   //省略部分代码...
}
```
重量级锁的释放可以分为以下步骤：
1. 判断当前锁对象中的_owner没有指向当前线程，如果_owner指向的BasicLock在当前线程栈上,那么将_owner指向当前线程。
2. 如果当前锁对象中的_owner指向当前线程，则判断当前线程重入锁的次数，如果不为0，那么就重新走ObjectMonitor::exit（），直到重入锁次数为0为止。
3. 释放当前锁，并根据QMode的模式判断，是否将_cxq中挂起的线程唤醒。还是其他操作。

### 感想
写了这么久，终于写完了~~~  掌声在哪里？

该篇文章主要是根据先关博客与自己对源码的理解，发现其实有很多东西自己还是描述的不是很清楚。主要原因是C++代码看的我头大。个人感觉Java的整个锁的机制其实涉及到蛮多的东西，自己理解的只是冰山一角，如果大家对代码或者文章不理解，请轻喷。我也是看的半懂半懂的。原谅我啦~~~

### 参考
站在巨人的肩膀上能看的更远~~~
《深入理解Java虚拟机：JVM高级特性与最佳实践》
《Java并发编程的艺术》
[深入理解Java并发之synchronized实现原理](https://blog.csdn.net/javazejian/article/details/72828483)
[jdk源码剖析二: 对象内存布局、synchronized终极原理](https://www.cnblogs.com/dennyzhangdd/p/6734638.html)
[jdk源码](https://pan.baidu.com/s/1Lk9yp8cEpSAnLvw5NJdqZg)
