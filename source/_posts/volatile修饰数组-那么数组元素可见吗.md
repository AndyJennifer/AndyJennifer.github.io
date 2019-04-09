---
title: 'volatile修饰数组,那么数组元素可见吗?'
tags:
  - volatile
categories:
  - 奇思妙想
date: 2019-04-09 15:42:14
---


![bg.jpg](https://upload-images.jianshu.io/upload_images/2824145-4f106ccf9a5b864c.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 前言
最近一段时间，在看并发集合的源码，发现了一个非常有趣的现象。我们都知道并发集合，为了保持对其他线程的可见性，通常集合中的方法都会使用CAS、volatile、synchronized、Lock等方式。但是在CopyOnWriteArrayList与ConcurrentHashMap中，对其中的存放数据的数组的操作却截然不同。


### CopyOnWriteArrayList

在CopyOnWiteArrayList中有一个用volatile修饰的数组（实际存放数据的数据结构），而获取CopyOnWiteArrayList中的数据直接是调用get(int index)方法。并没有涉及到更多的保持可见性的操作。
```   

  private transient volatile Object[] elements;

    final Object[] getArray() {
        return elements;
    }

    public E get(int index) {
        return get(getArray(), index);
    }

    private E get(Object[] a, int index) {
        return (E) a[index];
    }

```

### ConcurrentHashMap
而在ConcurrentHashMap同样是获取用volatile修饰的数组中的元素，却调用的getObjectVolatile来获取数组中的元素。
```
    transient volatile Node<K,V>[] table;
    static final <K,V> Node<K,V> tabAt(Node<K,V>[] tab, int i) {
        return (Node<K,V>)U.getObjectVolatile(tab, ((long)i << ASHIFT) + ABASE);
    }
```

### 疑惑
我相信大家看到这里，大家和我都有几个疑惑。

- 在CopyOnWriteArrayList中为什么就直接使用角标获取数组中的元素，而ConcurrentHashMap却调用的是getObjectVolatile方法呢？

- 假如ConcurrentHashMap通过getObjectVolatile方法获取数组中的元素是正确的（也就是说volatile修饰的数组的引用，数组的元素对其他线程不可见，所以我们要使用getObjectVolatile，来保证数据的可见性），那么CopyOnWriteArrayList简简单单的就直接获取数组中的元素，就能保证可见性了，这是什么神奇的操作？
- 假如CopyOnWriteArrayList的操作是正确的（也就是volatile修饰数组的时候，其内部数组的元素也是可见的），那么对于ConcurrentHashMap中的getObjectVolatile方法是不是存在着多余呢？


### 开始实验

到了这里问题的根源，其实就是volatile修饰的数组时，数组的元素是否对其他线程存在着可见性。为了解决这个疑惑，所以我就写了几个例子。

#### 例子1

在例子1中我声明了一个volatile数组arr，然后通过一个线程1修改arr[19]=2，在线程2中判断数组中的arr[19]是否等于2，如果是则跳出循环，并打印。具体例子如下：

```
public class TestVolatile {
    public static volatile long[] arr = new long[20];
    public static void main(String[] args) throws Exception {
        //线程1
        new Thread(new Thread(){
            @Override
            public void run() {
                //Thread A
                try {
                    TimeUnit.MILLISECONDS.sleep(1000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                arr[19] = 2;
            }
        }).start();
        //线程2
        new Thread(new Thread(){
            @Override
            public void run() {
                //Thread B
                while (arr[19] != 2) {
                }
                System.out.println("Jump out of the loop!");
            }
        }).start();
    }
}
```
运行结果：Jump out of the loop!

`WTF!?`程序居然跳出循环了！！！！也就是说volatile修饰的数组确实可以保证数组中的元素对其他线程的可见性？难度是Java开发人员不同，每个人的理解不同，所以写出了不同的操作？

那ConcurrentHashMap中的getObjectVolatile方法到底是不是多余的呢？难道是因为ConcurrentHashMap中存储的是对象，所以要再次通过volatile判断？为了验证这个问题，我又写了例子2。

#### 例子2

在例子2中，我将数组修改为Object类型，并创建了Person类，因为Person类（包含名字，年龄）是简单的Java对象，我就没有在例子中声明了。具体例子如下：

```
public class TestVolatile {

    public static  volatile Object[] arr = new Object[20];
    
    public static void main(String[] args) throws Exception {
        new Thread(new Thread() {
            @Override
            public void run() {
                //线程1
                try {
                    TimeUnit.MILLISECONDS.sleep(1000);
                    arr[19] = new Person("xixi",12);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        }).start();
        new Thread(new Thread() {
            @Override
            public void run() {
                //线程2
                while (true) {
                    Person p = (Person) arr[19];
                    if (p!=null&&p.getAge()==12) {
                        break;
                    }
                }
                System.out.println("Jump out of the loop!");
            }
        }).start();

    }
}

```
运行结果：Jump out of the loop!

从结果来看，也就是说和数组中存储的内容是无关的，不管是基本数据类型，还是对象类型。`只要是通过volatile修饰的数组，数组中的元素对其他线程都是可见的`，这里为什么ConcurrentHashMap中的getObjectVolatile方法的原因也不得为知。如果大家知道原因的话，希望在博客下方发表您的留言，让大家都知道这个知识点。


#### 例子3
虽然说发现了一个神奇的地方，但是好奇心，又驱使我做了另一个实现。在例子3中，我并没有用volatile变量修饰数组，而是在程序中添加了一个用volatile修饰的变量vol。实际的逻辑和例子1差不多，但是在线程2中我读取了volatile修饰的变量的值。具体例子如下：
```
public class TestVolatile {
    //注意没用volatile修饰数组
    public static long[] arr = new long[20]; 
    //这里用volatile修饰另一个变量
    public static volatile int vol = 0; 

    public static void main(String[] args) throws Exception {
        //线程1
        new Thread(new Thread(){
            @Override
            public void run() {
                try {
                    TimeUnit.MILLISECONDS.sleep(1000);    
                    arr[19] = 2;
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        }).start();
        //线程2
        new Thread(new Thread(){
            @Override
            public void run() {
                while (true) {
                    //读取vol的值
                    int i = vol;
                    if (arr[19] == 2) {
                        break;
                    }
                }
                System.out.println("Jump out of the loop!");
            }
        }).start();
    }
}
```
运行结果：Jump out of the loop!

WFT!?程序居然又跳出循环了，我并没有用volatile修饰数组，也就是说数组中的元素对其他线程并不是可见的。那么程序居然会跳出循环，什么鬼？？？？

为了解决这个问题，我查阅了相关资料，结果在[stackoverflow](https://stackoverflow.com/questions/53753792/java-volatile-array-my-test-results-do-not-match-the-expectations)
找到了我想要的答案。在原文的解答中是这样解释的:
```
If Thread A reads a volatile variable, then all all variables visible to Thread A when reading the volatile variable will also be re-read from main memory.
```
`简单翻译一下，就是当一个线程在读取一个使用volatile修饰的变量的时候，会将该线程中所有使用的变量从主内存中重新读取。`

那么也就解释了在例子3中，线程2为什么要跳出循环的原因了。

#### 例子4

但是因为自己手贱，我又写了另一个例子，这个例子，我真的无法解释为什么会跳出循环的原因，具体例子如下：

```
public class TestVolatile {

    public static long[] arr = new long[20];
    public static boolean flag = false;


    public static void main(String[] args) throws Exception {
        new Thread(new Thread() {
            @Override
            public void run() {
                //线程1
                try {
                    TimeUnit.MILLISECONDS.sleep(1000);
                    arr[19] = 2;
                    flag = true;
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        }).start();
        new Thread(new Thread() {
            @Override
            public void run() {
                //线程2
                while (true) {
                    System.out.println("flag--->" + flag);
                    if (arr[19] == 2) {
                        System.out.println("flag in loop--->" + flag);
                        break;
                    }
                }
                System.out.println("Jump out of the loop!");
            }
        }).start();

    }
}

```
输出结果：
```
...
flag--->false
flag--->false
flag--->false
flag--->false
flag in loop--->true
Jump out of the loop!
```
按照可见性原理，按照逻辑思维，程序是不可能跳出循环的。但是线程2居然循环了一定次数后，居然TM的跳出循环了。这到底发生了什么事情。我再一次的怀疑我自己的人生，怀疑我的编译器，怀疑我的cpu。

### 总结
写这个博客，完全自己一点点的好奇，但是一点一点的，我发现我把自己圈进去了，有可能这就是技术的魅力吧，因为我本身能力也非常有限，有些例子确实无法解释，这里把我的疑问抛出来，希望和大家一起讨论讨论，希望得到大家的帮助。一起得到一个正确的答案。


