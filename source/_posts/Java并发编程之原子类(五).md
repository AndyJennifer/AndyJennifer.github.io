---
title: Java并发编程之原子类(五)
date: 2019-02-23 21:36:39
categories:
- Java并发相关
tags: 
- 并发
---

{% asset_img 天天.jpeg 天天 %}


### 前言
在上篇文章{% post_link Java并发编程之Synchronized(三) %}中，曾描述Java提供了两种方式来处理线程安全的问题。第一种是互斥同步（悲观锁），第二种是采用非阻塞式同步（乐观锁）。虽然以上两种方案都能解决线程安全的问题。但是在JDK1.5开始，就提供了java.util.concurrent.atomic包，这个包中的原子操作类提供了更为简单高效、线程安全的方式来更新一个变量的值。例如AtomicBoolean、AtomicLong、AtomicInteger等。（这里提到的Atomic系列类原理都是CAS操作，如果你对CAS操作并不是是很熟悉，建议先阅读{% post_link Java并发编程之Java的CAS操作(四) %}。

#### 原子类的使用方式

既然我们提到Atomic系列类是简单高效且线程安全的。光说没用，我们直接来实际例子，具体代码如下所示：
```
class AtomicDemo {
    private AtomicInteger mAtomicInteger = new AtomicInteger();//如果没有指定值，默认是1
    
    private void doAdd() {
        for (int i = 0; i < 5; i++) {
            int value = mAtomicInteger.addAndGet(1);
            System.out.println(Thread.currentThread().getName() + "--->" + value);
        }
    }

    public static void main(String[] args) {
        AtomicDemo demo = new AtomicDemo();
        new Thread(demo::doAdd, "线程1").start();
        new Thread(demo::doAdd, "线程2").start();
    }
}
//输出结果
线程1--->1
线程1--->2
线程1--->3
线程2--->4
线程1--->5
线程2--->6
线程1--->7
线程2--->8
线程2--->9
线程2--->10
```
在上述代码逻辑非常简单，主要是想通过通过2个线程将mAtomicInteger 的值分别加1,每个线程执行加1并操作5次。

>这里简单对AtomicInteger 中的addAndGet（int delta）方法进行介绍，该方法是以原子的方式将输入的值与ActomicInteger中的值进行相加，返回相加后ActomicInteger中的值。

通过执行代码我们发现，结果最后打印的结果是我们预期的10。但是如果我们将ActomicInteger修改为普通的int类型，我们会发现结果是千奇百怪（这里我就不贴代码了）有兴趣的小伙伴可以自己去试一试。

### 原子类
在Java中的并发包中了提供了以下几种类型的原子类来来解决线程安全的问题。分为基本数据类型原子类、数组类型原子类、引用类型原子类、字段类型原子类。因为其内部原理都差不多一致。这里会对每种类型的原子类抽一个来介绍。

####  基本数据类型原子类
基本数据类型原子类主要为以下几种：
- AtomicBoolen: boolean类型原子类
- AtomicInteger: int类型原子类
- AtomicLong:  long类型原子类

这里我们以AtomicInteger来进行讲解，具体代码如下：

```
public class AtomicInteger extends Number implements java.io.Serializable {
  
    private static final sun.misc.Unsafe U = sun.misc.Unsafe.getUnsafe();
    private static final long VALUE;

    private volatile int value;//注意该值用volatile修饰

    public AtomicInteger(int initialValue) {
        value = initialValue;
    }
    //以原子的方式将输入的值与ActomicInteger中的值进行相加，
    //注意：返回相加前ActomicInteger中的值
    public final int getAndAdd(int delta) {
        return U.getAndAddInt(this, VALUE, delta);
    }
    //以原子的方式将输入的值与ActomicInteger中的值进行相加，
    //注意：返回相加后的结果
    public final int addAndGet(int delta) {
        return U.getAndAddInt(this, VALUE, delta) + delta;
    }
    //以原子方式将当前ActomicInteger中的值加1,
    //注意：返回相加前ActomicInteger中的值
    public final int getAndIncrement() {
        return U.getAndAddInt(this, VALUE, 1);
    }
    //以原子方式将当前ActomicInteger中的值加1,
    //注意：返回相加后的结果
    public final int incrementAndGet() {
        return U.getAndAddInt(this, VALUE, 1) + 1;
    }

    //省略部分代码...
  }
```
在上述代码中，我只留了AtomicInteger 类一部分常用的方法。大家在使用其内部方法时一定要注意其返回的结果。例如**getAndAdd（）与addAndGet（）方法之间的返回值的区别**。既然我们已经说过了使用Actomic系列原子类是线程安全的。那么现在我们就来看看其具体原理。这里我们以getAndAdd（）方法为例进行讲解。

AtomicInteger内部会调用其中sun.misc.Unsafe方法中getAndAddInt的方法。具体代码如下：
```
 public final int getAndAdd(int delta) {
        return U.getAndAddInt(this, VALUE, delta);
    }
```
而sun.misc.Unsafe方法中getAndAddInt方法又会调用jdk.internal.misc.Unsafe的getAndAddInt，具体代码如下：
```
 public final int getAndAddInt(Object o, long offset, int delta) {
        return theInternalUnsafe.getAndAddInt(o, offset, delta);
    }
```
jdk.internal.misc.Unsafe的getAndAddInt（）方法的声明如下：
```
public final int getAndAddInt(Object o, long offset, int delta) {
        int v;
        do {
            v = getIntVolatile(o, offset);//先获取内存中存储的值
        } while (!weakCompareAndSetInt(o, offset, v, v + delta));//如果不是期望的结果值，就一直循环
        return v;
    }
    
//该函数返回值代表CAS操作是否成功    
public final boolean weakCompareAndSetInt(Object o, long offset,
                                          int expected,
                                          int x) {
     return compareAndSetInt(o, offset, expected, x);//执行CAS操作
    }
```
从上述代码中我们可以得出，会先获取内存中存储的值，最终会调用compareAndSetInt（）方法来完成最终的原子操作。其中compareAndSetInt（）方法的返回值代表着该次CAS操作是否成功。如果不成功。那么会一直循环。直到成功为止（也就是循环CAS操作）。

这里简要的对CAS操作进行描述：CAS操作内部实现原理是缓存锁，在其操作期间，会修改对应操作对象的内存地址。同时其会保证各个处理器的缓存是一致的，如果处理器发现自己的数据对应的内存地址被修改，就会将当前缓存的数据处理为无效，同时该处理器会重新从系统内存中把数据处理到缓存中。如果你对CAS操作还是不熟悉，建议先阅读{% post_link Java并发编程之Java的CAS操作(四) %}，在回过头来看这篇文章。

>这里有一个小的问题，大家可以思考一下。我们都知道对于long与double数据类型，在java内存模型中long与double具有非原子协定。但是现在商用的虚拟机都把关于long和double变量的读写操作视为具有原子性的操作。那这里为什么会出现一个AtomicLong？或者出现了AtomicLong为什么没有出现ActomicDouble这个类呢？


#### 数组类型原子类
对于数组类型的原子类，在Java中，主要通过原子的方式更新数组里面的某个元素，数组类型原子类主要有以下几种：
- AtomicIntegerArray：Int数组类型原子类
- AtomicLongArray：long数组类型原子类
- AtomicReferenceArray：引用类型原子类（关于AtomicReferenceArray即引用类型原子类会在下文介绍）

这里我们还是以AtomicIntegerArray为例，因为其内部原理都是循环CAS操作，所以我们这里就描述其使用方式，具体代码如下：
```
class AtomicDemo {

    private int[] value = new int[]{0, 1, 2};
    private AtomicIntegerArray mAtomicIntegerArray = new AtomicIntegerArray(value);

    private void doAdd() {
        for (int i = 0; i < 5; i++) {
            int value = mAtomicIntegerArray.addAndGet(0, 1);
            System.out.println(Thread.currentThread().getName() + "--->" + value);
        }
    }

    public static void main(String[] args) {
        AtomicDemo demo = new AtomicDemo();
        new Thread(demo::doAdd, "线程1").start();
        new Thread(demo::doAdd, "线程2").start();
    }
  //程序输出结果如下：
线程1--->1
线程1--->2
线程1--->4
线程2--->3
线程1--->5
线程2--->6
线程1--->7
线程2--->8
线程2--->9
线程2--->10
}
```

#### 引用类型原子类
在{% post_link Java并发编程之Java的CAS操作(四) %}文章中我们曾经提到过两个问题，**第一个问题**：虽然我们能通过循环CAS操作来完成对一个变量的原子操作，但是对于多个变量进行操作时，自旋CAS操作就不能保证其原子性。**第二个问题**：ABA问题，因为CAS在操作值的时候，需要检查值有没有发生变化，如果没有发生变化则更新，但是如果一个值原来是A，变成了B，又变成了A，那么使用CAS进行检查时会发现她的值并没有发生变化。那么会导致程序出问题。

为了解决上述提到的两个问题，Java为我们提供了AtomicReference等系列引用类型原子类，来保证引用对象之间的原子性，即可以把多个变量放在一个对象里来进行CAS操作与ABA问题。主要类型原子类如下：
- AtomicReference：
- AtomicReferenceFieldUpdater：
- AtomicMarkableReference：
- AtomicStampedReference：

##### 多个变量的CAS操作
这里我们先解决第一个问题，关系多个变量的CAS操作，我们先以AtomicReference来进行讲解，具体代码如下所示：
（这里提一嘴，关于引用类型的原子类，内部都调用的是**compareAndSwapObject**（）方法来实现CAS操作的。）
```
class AtomicDemo {

    Person mPerson = new Person("红红", 1);
    private AtomicReference<Person> mAtomicReference = new AtomicReference<>(mPerson);

    private class Person {
        String name;
        int age;

        Person(String name, int age) {
            this.name = name;
            this.age = age;
        }
    }


    private void updatePersonInfo(String name, int age) throws Exception {
        System.out.println(Thread.currentThread().getName() + "更新前--->" + mAtomicReference.get().name + "---->" + mAtomicReference.get().age);
        mAtomicReference.getAndUpdate(person -> new Person(name, age));
    }

    public static void main(String[] args) {
        AtomicDemo demo = new AtomicDemo();
        new Thread(() -> demo.updatePersonInfo("蓝蓝", 2), "线程1").start();
   
        Thread.sleep(1000);
        System.out.println("暂停一秒--->" + demo.mAtomicReference.get().name + "---->" + demo.mAtomicReference.get().age);
     
        System.out.println("更新后---->" + demo.mAtomicReference.get().name + "---->" + demo.mAtomicReference.get().age);

    }

}
//输出结果
线程1更新前--->红红---->1
暂停一秒--->蓝蓝---->2
更新后---->蓝蓝---->2
```
上述代码中创建了Person 类，且当前AtomicReference传入的是当前 mPerson =new Person("红红", 1)，在Main方法中创建线程1使其调用mAtomicReference.getAndUpdate(new Person("蓝蓝"，2))来更新Person信息。更新完成后休眠一秒后，获取更新结果并打印。从结果上来看，的确是对多个变量进行了更新的操作。

##### ABA问题
关于ABA问题，大家已经知道其出现的原因，现在我们就用具体例子让大家来了解一下。ABA会引发的问题。
这里我们以具体的例子来进行讲解。具体例子如下所示：

{% asset_img aba.png aba %}


观察上图，我们初始化了一个单向的链表结构，其中Header指向链表头节点，其中A节点的下一节点为B节点。
这个时候我们希望通过线程1，通过CAS操作将链表中的B节点放入头节点中,且B的next节点为A节点。具体为代码如下所示：

```
if(header.compareAndSet(A,B)){
	B.next = A;
	A.next = null;
}

```

当线程1已经拿到header.compareAndSet(A,B)的结果正准备执行下一行代码时，突然线程2介入，将A、B两个节点移除，同时重新将A、C、D三个节点依次加入链表中。当线程2操作完毕的时候，这个时候线程1接着执行。线程1在执行的时候，会检查当前链表中A是否为头节点，当前情况A是头节点（通过线程2添加的）。那么就会执行剩余代码也就是（B.next =A, A.next = null)。那么通过线程1操作完成后，就出现上图中当前链表中C、D两个节点丢失的情况。所以为了解决ABA问题，Java中提供了**AtomicStampedReference**来解决。

为了方便大家理解对AtomicStampedReference类的使用，提供了以下例子：具体代码如下所示：
```
class AtomicDemo {

    Person mPerson = new Person("红红", 1);
    private AtomicStampedReference<Person> mAtomicReference = new AtomicStampedReference<>(mPerson, 1);

    private class Person {
        String name;
        int age;

        Person(String name, int age) {
            this.name = name;
            this.age = age;
        }
    }


    /**
     * 更新信息
     *
     * @param name     名称
     * @param age      年龄
     * @param oldStamp CAS操作比较的旧的版本
     * @param newStamp 希望更新后的版本
     */
    private void updatePersonInfo(String name, int age, int oldStamp, int newStamp) {
        System.out.println(Thread.currentThread().getName() + "更新前--->" + mAtomicReference.getReference().name + "---->" + mAtomicReference.getReference().age);
        mAtomicReference.compareAndSet(mPerson, new Person(name, age), oldStamp, newStamp);
    }

    public static void main(String[] args) throws Exception {
        AtomicDemo demo = new AtomicDemo();
        new Thread(() -> demo.updatePersonInfo("蓝蓝", 2, 1, 2), "线程1").start();


        Thread.sleep(1000);

        System.out.println("暂停一秒--->" + demo.mAtomicReference.getReference().name + "---->" + demo.mAtomicReference.getReference().age);
        new Thread(() -> demo.updatePersonInfo("花花", 3, 1, 3), "线程2").start();


        Thread.sleep(1000);
        System.out.println("更新后---->" + demo.mAtomicReference.getReference().name + "---->" + demo.mAtomicReference.getReference().age);

    }

}
//输出结果
线程1更新前--->红红---->1
暂停一秒--->蓝蓝---->2
线程2更新前--->蓝蓝---->2
更新后---->蓝蓝---->2
```
在上述代码中，我们使用AtomicStampedReference类，其中在使用该类的时候，需要传入一个类似于版本（你也可以叫做邮戳，时间戳等，随你喜欢）的int类型的属性。在Main方法中我们分别创建了2个线程来进行CAS操作，其中线程1想做的操作是将版本为1的mPerson("红红"，1)修改为版本为2的Person("蓝蓝，2")。当线程1执行完毕后，紧接着线程2开始执行，线程2想做的操作是将版本为1的mPerson(“红红”，1)修改为版本3的Person("花花"，3)。从程序输出结果可以看出，线程2的操作是没有执行的。也就验证了AtomicStampedReference确实解决了ABA的问题。

#### 字段类型原子类
如果需要更新某个类中的某个字段，在Actomic系列中，Java提供了以下几个类来实现：
- AtomicIntegerFieldUpdater：int类型字段原子类
- AtomicLongFieldUpdater：long类型字段原子类
- AtomicReferenceFieldUpdater：引用型字段原子类

上面所说的三个类原理都差不多，这里我们以AtomicIntegerFieldUpdate类来讲解，具体代码如下：
```
class AtomicDemo {

    Person mPerson = new Person("红红", 1);
    private AtomicIntegerFieldUpdater<Person> mFieldUpdater = AtomicIntegerFieldUpdater.newUpdater(Person.class, "age");

    private class Person {
        String name;
        volatile int age;//使用volatile修饰

        Person(String name, int age) {
            this.name = name;
            this.age = age;
        }
    }


    /**
     * 更新信息
     *
     * @param age 年龄
     */
    private void updatePersonInfo(int age) {
        System.out.println("更新前--->" + mPerson.age);
        mFieldUpdater.addAndGet(mPerson, age);
    }

    private int getUpdateInfo() {
        return mFieldUpdater.get(mPerson);
    }

    public static void main(String[] args) throws Exception {
        AtomicDemo demo = new AtomicDemo();
        new Thread(() -> demo.updatePersonInfo(12), "线程1").start();
        Thread.sleep(1000);
        System.out.println("更新后--->" + demo.getUpdateInfo());
    }

}
//输出结果
更新前--->1
更新后--->13
```
这里对AtomicIntegerFieldUpdate不在进行过多的描述，大家需要主要的是在使用字段类型原子类的时候，**需要进行更新的字段，需要通过volatile来修饰。**

### 总结
- Atomic系列类为我们提供了简单高效、线程安全的方式来更新一个变量的值或一个引用的值。
- Atomic为处理多个变量原子更新的问题，为我们提供了AtomicReference类，为了解决ABA问题提供了AtomicStampedReference。在实际使用中，根据代码情况来使用不同Atomic的系列类。
- 在使用字段类型原子类的时候，需要将需要更新的字段，通过volatile来修饰。
