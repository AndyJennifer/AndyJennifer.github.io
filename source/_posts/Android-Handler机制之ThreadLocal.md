---
title: Android Handler机制之ThreadLocal
date: 2019-02-23 22:05:18
categories:
- Handler相关
tags: 
- 异步任务
---


![小积木.jpg](https://upload-images.jianshu.io/upload_images/2824145-04bd2a2f4dcf1849.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

>该文章属于Android Handler系列文章，如果想了解更多，请点击{% post_link Android-Handler机制之总目录 %}


### 前言
要想了解Android 的Handle机制，我们首先要了解ThreadLocal，根据字面意思我们都能猜出个大概。就是线程本地变量。那么我们把变量存储在本地有什么好处呢？其中的原理又是什么呢？下面我们就一起来讨论一下ThreadLocal的使用与原理。

### ThreadLocal简单介绍
该类提供线程局部变量。这些变量不同于它们的正常变量，即每一个线程访问自身的局部变量时，都有它自己的，独立初始化的副本。该变量通常是与线程关联的私有静态字段，列如用于ID或事物ID。大家看了介绍后，有可能还是不了解其主要的主要作用，简单的画个图帮助大家理解。


![ThreadLocal示意图.png](https://upload-images.jianshu.io/upload_images/2824145-550960f5459468ad.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

从图上可以看出，通过ThreadLocal，每个线程都能获取自己线程内部的私有变量，有可能大家觉得无图无真相，“你一个人在那里神吹，我怎么知道你说的对还是不对呢？”，下面我们通过具体的例子详细的介绍，来看下面的代码。
```
class ThreadLocalTest {
	//会出现内存泄漏的问题，下文会描述
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
在上诉代码中，我们在主线程中设置mThreadLocal的值为"线程main",在线程A中设置为”线程A“，在线程B中设置为”线程B",运行程序打印结果如下图所示：

```
main
线程A
线程B
```
从上面结果可以看出，虽然是在不同的线程中访问的同一个变量mThreadLocal，但是他们通过ThreadLocl获取到的值却是不一样的。也就验证了上面我们所画的图是正确的了，那么现在，我们已经知道了ThreadLocal的用法，那么我们现在来看看其中的内部原理。

### ThreadLocal原理
为了帮助大家快速的知晓ThreadLocal原理，这里我将ThreadLocal的原理用下图表示出来了：

![threadLocal.png](https://upload-images.jianshu.io/upload_images/2824145-7b553ff229bcd302.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

在上图中我们可以发现，整个ThreadLocal的使用都涉及到线程中`ThreadLocalMap`,虽然我们在外部调用的是ThreadLocal.set(value)方法，但本质是通过线程中的`ThreadLocalMap中的set(key,value)方法`，那么通过该情况我们大致也能猜出get方法也是通过ThreadLocalMap。那么接下来我们一起来看看ThreadLocal中set与get方法的具体实现与ThreadLocalMap的具体结构。

### ThreadLocal的set方法
在使用ThreadLocal时，我们会调用ThreadLocal的set(T value)方法对线程中的私有变量设置，我们来查看ThreadLocal的set方法
```
    public void set(T value) {
        Thread t = Thread.currentThread();//获取当前线程
        ThreadLocalMap map = getMap(t);//拿到线程的LocalMap
        if (map != null)
            map.set(this, value);//设值 key->当前ThreadLocal对象。value->为当前赋的值
        else
            createMap(t, value);//创建新的ThreadLocalMap并设值
    }
```
当调用set(T value) 方法时，方法内部会获取当前线程中的ThreadLocalMap，获取后进行判断，如果不为空，就**调用ThreadLocalMap的set方法**（其中key为当前ThreadLocal对象，value为当前赋的值）。反之，让当前线程创建新的ThreadLocalMap并设值，其中getMap()与createMap()方法具体代码如下：
```
  ThreadLocalMap getMap(Thread t) {
        return t.threadLocals;
    }
    
  void createMap(Thread t, T firstValue) {
        t.threadLocals = new ThreadLocalMap(this, firstValue);
    }
```

简简单单的通过ThreadLocalMap的set()方法，我们已经大致了解了。ThreadLocal为什么能操作线程内的私有数据了，ThreadLocal中所有的数据操作都与线程中的ThreadLocalMap有关，同时那我们接下来看看ThreadLocalMap相关代码。

#### ThreadLocalMap 内部结构

ThreadLocalMap是ThreadLocal中的一个静态内部类，官方的注释写的很全面，这里我大概的翻译了一下，**ThreadLocalMap是为了维护线程私有值创建的自定义哈希映射。其中线程的私有数据都是非常大且使用寿命长的数据**(其实想一想，为什么要存储这些数据呢，第一是为了把常用的数据放入线程中提高了访问的速度，第二是如果数据是非常大的，避免了该数据频繁的创建，不仅解决了存储空间的问题，也减少了不必要的IO消耗)。

ThreadLocalMap 具体代码如下：
```
 static class ThreadLocalMap {
		//存储的数据为Entry,且key为弱引用
        static class Entry extends WeakReference<ThreadLocal<?>> {
            /** The value associated with this ThreadLocal. */
            Object value;

            Entry(ThreadLocal<?> k, Object v) {
                super(k);
                value = v;
            }
        }
        //table初始容量
        private static final int INITIAL_CAPACITY = 16;
      
        //table 用于存储数据
        private Entry[] table;
        
	    //负载因子，用于数组容量扩容
        private int threshold; // Default to 0
        
		//负载因子，默认情况下为当前数组长度的2/3
        private void setThreshold(int len) {
            threshold = len * 2 / 3;
        }
	    //第一次放入Entry数据时，初始化数组长度，定义扩容阀值，
        ThreadLocalMap(ThreadLocal<?> firstKey, Object firstValue) {
            table = new Entry[INITIAL_CAPACITY];//初始化数组长度为16
            int i = firstKey.threadLocalHashCode & (INITIAL_CAPACITY - 1);
            table[i] = new Entry(firstKey, firstValue);
            size = 1;
            setThreshold(INITIAL_CAPACITY);//阀值为当前数组默认长度的2/3
        }

```
从代码中可以看出，虽然官方申明为ThreadLocalMap是一个哈希表，但是它与我们传统认识的HashMap等哈希表内部结构是不一样的。ThreadLocalMap内部仅仅维护了**Entry[] table,**数组。其中**Entry实体中对应的key为弱引用（下文会将为什么会用弱引用）**，在第一次放入数据时，会初始化数组长度（为16),定义数组扩容阀值（当前默认数组长度的2/3)。


#### ThreadLocalMap 的set()方法
```
 private void set(ThreadLocal<?> key, Object value) {

		    //根据哈希值计算位置
            Entry[] tab = table;
            int len = tab.length;
            int i = key.threadLocalHashCode & (len-1);
            
            //判断当前位置是否有数据，如果key值相同，就替换，如果不同则找空位放数据。
            for (Entry e = tab[i];
                 e != null;
                 e = tab[i = nextIndex(i, len)]) {//获取下一个位置的数据
                ThreadLocal<?> k = e.get();
			//判断key值相同否，如果是直接覆盖 （第一种情况）
                if (k == key) {
                    e.value = value;
                    return;
                }
			//如果当前Entry对象对应Key值为null,则清空所有Key为null的数据（第二种情况）
                if (k == null) {
                    replaceStaleEntry(key, value, i);
                    return;
                }
            }
            //以上情况都不满足，直接添加（第三种情况）
            tab[i] = new Entry(key, value);
            int sz = ++size;
            if (!cleanSomeSlots(i, sz) && sz >= threshold)//如果当前数组到达阀值，那么就进行扩容。
                rehash();
        }
```
直接通过代码理解比较困难,这里直接将set方法分为了三个步骤，下面我们我们就分别对这个三个步骤，分别通过图与代码的方式讲解。

##### 第一种情况， Key值相同
如果当前数组中，如果当前位置对应的Entry的key值与新添加的Entry的key值相同，直接进行覆盖操作。具体情况如下图所示

![key值相同情况.png](https://upload-images.jianshu.io/upload_images/2824145-70b6cc18bc656b19.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

如果当前数组中。存在key值相同的情况，ThreadLocal内部操作是直接覆盖的。这种情况就不过多的介绍了。

##### 第二种情况，如果当前位置对应Entry的Key值为null
第二种情况相对来说比较复杂，这里先给图，然后会根据具体代码来讲解。

![对应位置Key值为null.png](https://upload-images.jianshu.io/upload_images/2824145-7bdc448fcc55fc14.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

从图中我们可以看出来。当我们添加新Entry(key=19,value =200,index = 3)时，数组中已经存在旧Entry(key =null,value = 19),当出现这种情况是，方法内部会将新Entry的值全部赋值到旧Entry中，**同时会将所有数组中key为null的Entry全部置为null（图中大黄色数据）**。在源码中，当新Entry对应位置存在数据，且key为null的情况下，会走`replaceStaleEntry`方法。具体代码如下：
```
   private void replaceStaleEntry(ThreadLocal<?> key, Object value,
                                       int staleSlot) {
            Entry[] tab = table;
            int len = tab.length;
            Entry e;

	        //记录当前要清除的位置
            int slotToExpunge = staleSlot;
            
            //往前找，找到第一个过期的Entry(key为空)
            for (int i = prevIndex(staleSlot, len);
                 (e = tab[i]) != null;
                 i = prevIndex(i, len))
                if (e.get() == null)//判断引用是否为空，如果为空,擦除的位置为第一个过期的Entry的位置
                    slotToExpunge = i;

		    //往后找，找到最后一个过期的Entry(key为空)，
            for (int i = nextIndex(staleSlot, len);//这里要注意获得位置有可能为0，
                 (e = tab[i]) != null;
                 i = nextIndex(i, len)) {
                ThreadLocal<?> k = e.get();
                //在往后找的时候，如果获取key值相同的。那么就重新赋值。
                if (k == key) {
                	//赋值到之前传入的staleSlot对应的位置
                    e.value = value;
                    tab[i] = tab[staleSlot];
                    tab[staleSlot] = e;

                    //如果往前找的时候，没有过期的Entry,那么就记录当前的位置（往后找相同key的位置）
                    if (slotToExpunge == staleSlot)
                        slotToExpunge = i;
                        
                    //那么就清除slotToExpunge位置下所有key为null的数据
                    cleanSomeSlots(expungeStaleEntry(slotToExpunge), len);
                    return;
                }

			    //如果往前找的时候，没有过期的Entry,且key =null那么就记录当前的位置（往后找key==null位置）
                if (k == null && slotToExpunge == staleSlot)
                    slotToExpunge = i;
            }

            // 把当前key为null的对应的数据置为null，并创建新的Entry在该位置上
            tab[staleSlot].value = null;
            tab[staleSlot] = new Entry(key, value);

            //如果往后找，没有过期的实体， 
            //且staleSlot之前能找到第一个过期的Entry(key为空)，
            //那么就清除slotToExpunge位置下所有key为null的数据
            if (slotToExpunge != staleSlot)
                cleanSomeSlots(expungeStaleEntry(slotToExpunge), len);
        }

```
上面代码看起来比较繁杂，但是大家仔细梳理就会发现其实该方法，主要对四种情况进行了判断，具体情况如下图表所示：

![TIM截图20180731110649.png](https://upload-images.jianshu.io/upload_images/2824145-218dc09b044027f3.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

我们已经了解了replaceStaleEntry方法内部会清除key==null的数据，而其中具体的方法与expungeStaleEntry()方法与cleanSomeSlots()方法有关，所以接下来我们来分析这两个方法。看看其的具体实现。

 **expungeStaleEntry ()方法**
```
    private int expungeStaleEntry(int staleSlot) {
            Entry[] tab = table;
            int len = tab.length;

            // 将staleSlot位置下的数据置为null
            tab[staleSlot].value = null;
            tab[staleSlot] = null;
            size--;

            Entry e;
            int i;
            //往后找。
            for (i = nextIndex(staleSlot, len);
                 (e = tab[i]) != null;
                 i = nextIndex(i, len)) {
                ThreadLocal<?> k = e.get();
                if (k == null) {//清除key为null的数据
                    e.value = null;
                    tab[i] = null;
                    size--;
                } else {
                //如果key不为null,但是该key对应的threadLocalHashCode发生变化，
                //计算位置，并将元素放入新位置中。
                    int h = k.threadLocalHashCode & (len - 1);
                    if (h != i) {
                        tab[i] = null;
                        while (tab[h] != null)
                            h = nextIndex(h, len);
                        tab[h] = e;
                    }
                }
            }
            return i;//返回最后一个tab[i]) != null的位置
        }
```
expungeStaleEntry（）方法主要干了三件事，第一件，将staleSlot的位置对应的数据置为null,第二件,删除并删除此位置后对应相关联位置key = null的数据。第三件，如果如果key不为null,但是该key对应的threadLocalHashCode发生变化，计算变化后的位置，并将元素放入新位置中。

**cleanSomeSlots（）方法**
```
    private boolean cleanSomeSlots(int i, int n) {
            boolean removed = false;
            Entry[] tab = table;
            int len = tab.length;
            do {
                i = nextIndex(i, len);
                Entry e = tab[i];
                if (e != null && e.get() == null) {
                    n = len;
                    removed = true;
                    i = expungeStaleEntry(i);
                }
            } while ( (n >>>= 1) != 0);
            return removed;//如果有过期的数据被删除,就返回true,反之false
        }
```
在了解了expungeStaleEntry（）方法后，再来理解cleanSomeSlots（）方法就很简单了。其中第一个参数表示开始扫描的位置，第二个参数是扫描的长度。从代码我们明显的看出。就是简单的遍历删除所有位置下key==null的数据。


##### 第三种情况，当前对应位置为null

![没有数据的情况.png](https://upload-images.jianshu.io/upload_images/2824145-b33d9fa695210adf.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

**图上为了方便大家，理解清空上下数据的情况，我并没有重新计算位置（希望大家注意！！！）**

看到这里，为了方便大家避免不必要的查阅代码，我直接将代码贴出来了。代码如下。
```
tab[i] = new Entry(key, value);
int sz = ++size;
if (!cleanSomeSlots(i, sz) && sz >= threshold)
                rehash();
               
```
从上述代码其实，大家很明显的看出来，就是清除key==null的数据，判断当前数据的长度是不是到达阀值（默认没扩容前为INITIAL_CAPACITY *2/3，其中INITIAL_CAPACITY = 16），如果达到了重新计算数据的位置。关于rehash()方法，具体代码如下：
```
 private void rehash() {
         expungeStaleEntries();

         // Use lower threshold for doubling to avoid hysteresis
         if (size >= threshold - threshold / 4)
                resize();
        }
        
 //清空所有key==null的数据
 private void expungeStaleEntries() {
         Entry[] tab = table;
         int len = tab.length;
         for (int j = 0; j < len; j++) {
             Entry e = tab[j];
             if (e != null && e.get() == null)
                 expungeStaleEntry(j);
            }
        }
 //重新计算key!=null的数据。新的数组长度为之前的两倍      
 private void resize() {
			//对原数组进行扩容，容量为之前的两倍
            Entry[] oldTab = table;
            int oldLen = oldTab.length;
            int newLen = oldLen * 2;
            Entry[] newTab = new Entry[newLen];
            int count = 0;
			//重新计算位置
            for (int j = 0; j < oldLen; ++j) {
                Entry e = oldTab[j];
                if (e != null) {
                    ThreadLocal<?> k = e.get();
                    if (k == null) {
                        e.value = null; // Help the GC
                    } else {
                        int h = k.threadLocalHashCode & (newLen - 1);
                        while (newTab[h] != null)
                            h = nextIndex(h, newLen);
                        newTab[h] = e;
                        count++;
                    }
                }
            }
			//重新计算阀值（负载因子）为扩容之后的数组长度的2/3
            setThreshold(newLen);
            size = count;
            table = newTab;
        }
```
rehash内部所有涉及到的方法，我都列举出来了。可以看出在添加数据的时候，会进行判断是否扩容操作，如果需要扩容，会清除所有的key==null的数据，（也就是调用expungeStaleEntries（）方法，其中expungeStaleEntry（）方法已经介绍了，就不过多描述），同时会重新计算数据中的位置。

### ThreadLocal的get()方法
在了解了ThreadLocal的set()方法之后，我们看看怎么获取ThreadLocal中的数据，具体代码如下：
```
  public T get() {
        Thread t = Thread.currentThread();//获取当前线程
        ThreadLocalMap map = getMap(t);//拿到线程中的Map
        if (map != null) {
            //根据key值（ThreadLocal）对象，获取存储的数据
            ThreadLocalMap.Entry e = map.getEntry(this);
            if (e != null) {
                @SuppressWarnings("unchecked")
                T result = (T)e.value;
                return result;
            }
        }
        //如果ThreadLocalMap为空，创建新的ThreadLocalMap 
        return setInitialValue();
    }
```
其实ThreadLocal的get方法其实很简单，就是获取当前线程中的ThreadLocalMap对象，如果没有则创建，如果有，则根据当前的 key(当前ThreadLocal对象)，获取相应的数据。其中内部调用了ThreadLocalMap的getEntry（）方法区获取数据，我们继续查看getEntry（）方法。
```
 private Entry getEntry(ThreadLocal<?> key) {
            int i = key.threadLocalHashCode & (table.length - 1);
            Entry e = table[i];
            if (e != null && e.get() == key)
                return e;
            else
                return getEntryAfterMiss(key, i, e);
        }
```
getEntry（）方法内部也很简单，也只是根据当前key哈希后计算的位置，去找数组中对应位置是否有数据，如果有，直接将数据放回，如果没有，则调用getEntryAfterMiss（）方法，我们继续往下看 。

```
 private Entry getEntryAfterMiss(ThreadLocal<?> key, int i, Entry e) {
            Entry[] tab = table;
            int len = tab.length;

            while (e != null) {
                ThreadLocal<?> k = e.get();
                if (k == key)//如果key相同，直接返回
                    return e;
                if (k == null)//如果key==null,清除当前位置下所有key=null的数据。
                    expungeStaleEntry(i);
                else
                    i = nextIndex(i, len);
                e = tab[i];
            }
            return null;//没有数据直接返回null
        }
```
从上述代码我们可以知道，如果从数组中，获取的key==null的情况下，get方法内部也会调用expungeStaleEntry（）方法，去清除当前位置所有key==null的数据，**也就是说现在不管是调用ThreadLocal的set（）还是get()方法，都会去清除key==null的数据。**

### ThreadLocal内存泄漏的问题
通过整个ThreadLocal机制的探索，我相信大家肯定会有一个疑惑，`为什么ThreadLocalMap中采用是的是弱引用作为Key?`关于该问题，涉及到Java的回收机制。

#### 为什么使用弱引用
在Java中判断一个对象到底是不是需要回收，都跟引用相关。在Java中引用分为了4类。
- 强引用：只要引用存在，垃圾回收器永远不会回收Object obj = new Object();而这样 obj对象对后面new Object的一个强引用，只有当obj这个引用被释放之后，对象才会被释放掉。
- 软引用：是用来描述，一些还有但并非必须的对象，对于软引用关联着的对象，在系统将要发生内存溢出异常之前，将会把这些对象列进回收范围之中进行第二次回收。（SoftReference)
- 弱引用：也是用来描述非必须的对象，但是它的强度要比软引用更弱一些。被弱引用关联的对象只能生存到下一次垃圾收集发生之前，当垃圾收集器工作是，无论当前内存是否足够，都会回收掉被弱引用关联的对象。（WeakReference)
- 虚引用：也被称为幽灵引用，它是最弱的一种关系。一个对象是否有引用的存在，完全不会对其生存时间构成影响，也无法通过一个虚引用来取得一个实例对象。

通过该知识点的了解后，我们再来了解为什么ThreadLocal不能使用强引用，**如果key使用强引用，那么当引用ThreadLocal的对象被回收了，但ThreadLocalMap中还持有ThreadLocal的强引用，如果没有手动删除，ThreadLocal不会被回收，导致内存泄漏。**

#### 弱引用带来的问题
当我们知道了为什么采用弱引用来作为ThreadLocalMap中的key的知识点后，这个时候又会引申出另一个问题`不管是调用ThreadLocal的set（）还是get()方法，都会去清除key==null的数据。为毛我们要去清除那些key==null的Entry呢？`

为什么清除key==null的Entry主要有以下两个原因，具体如下所示：
- 从上面我们已经知道了，**ThreadLocalMap使用ThreadLocal的弱引用作为key**，也就是说，如果一个ThreadLocal没有外部强引用来引用它，那么系统 GC 的时候，这个ThreadLocal势必会被回收。这样一来，ThreadLocalMap中就会出现key为null的Entry，就没有办法访问这些key为null的Entry的value，
- 如果当前线程迟迟不结束的话，这些key为null的Entry的value就会一直存在一条强引用链：Thread Ref（当前线程引用） -> Thread -> ThreadLocalMap -> Entry -> value，那么将会导致这些Entry永远无法回收，造成内存泄漏。

通过以上分析，我们可以了解在ThreadLocalMap的设计中其实已经考虑到上述两种情况，也加上了一些防护措施。（在调用ThreadLocal的get(),set(),remove()方法的时候都会清除线程ThreadLocalMap里所有key为null的Entry）


### ThreadLocal使用注意事项
虽然ThreadLocal帮我们考虑了内存泄漏的问题，为我们加上了一些防护措施。但是在实际使用中，我们还是需要注意避免以下两种情况，下述两种情况仍然有可能会导致内存泄漏。

#### 避免使用static的ThreadLocal
使用static修饰的ThreadLocal，延长了ThreadLocal的生命周期，可能导致的内存泄漏。具体原因是在Java虚拟机在加载类的过程中为静态变量分配内存。static变量的生命周期取决于类的生命周期，也就是说类被卸载时，静态变量才会被销毁并释放内存空间。而类的生命周期结束与下面三个条件相关。
1. 该类所有的实例都已经被回收，也就是 Java 堆中不存在该类的任何实例。
2. 加载该类的ClassLoader已经被回收。
3. 该类对应的java.lang.Class对象没有任何地方被引用，没有在任何地方通过反射访问该类的方法。

#### 分配使用了ThreadLocal又不再调用get(),set(),remove()方法
其实理解起来也很简单，就是第一次调用了ThreadLocal设置数据后，就不在调用get()、set()、remove()方法。也就是说现在ThreadLocalMap中就只有一条数据。那么如果调用ThreadLocal的线程一直不结束的话，即使ThreadLocal已经被置为null（被GC回收）,也一直存在一条强引用链：Thread Ref（当前线程引用） -> Thread -> ThreadLocalMap -> Entry -> value，导致数据无法回收，造成内存泄漏。


### 总结
- ThreadLocal本质是操作线程中ThreadLocalMap来实现本地线程变量的存储的
- ThreadLocalMap是采用数组的方式来存储数据，其中key(弱引用)指向当前ThreadLocal对象，value为设的值
- ThreadLocal为内存泄漏采取了处理措施，在调用ThreadLocal的get(),set(),remove()方法的时候都会清除线程ThreadLocalMap里所有key为null的Entry
- 在使用ThreadLocal的时候，我们仍然需要注意，避免使用static的ThreadLocal，分配使用了ThreadLocal后，一定要根据当前线程的生命周期来判断是否需要手动的去清理ThreadLocalMap中清key==null的Entry。