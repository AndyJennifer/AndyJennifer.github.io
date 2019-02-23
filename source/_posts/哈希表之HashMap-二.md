---
title: 哈希表之HashMap(二)
date: 2019-02-23 21:54:01
categories:
- Java集合相关
tags: 
- Java
---

![你说是什么图就是什么图.jpg](https://upload-images.jianshu.io/upload_images/2824145-c6655b07bfd9b105.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

>在写这篇文章之前，看了很多关于HashMap解析的文章。对于大多数人来说，可了跟着别人的文章走一遍。大家都能了解HashMap的内部结构，使用方法以及注意事项。我还是觉得知道用是一回事。知道原理是另一回事。只有了解了其数据结构设计初衷。才能更好的使用它。此系列文章主要分为两个部分，具体目录如下：


* [哈希表初识(一)](https://www.jianshu.com/p/345019b77571)
* 哈希表之HashMap(二)

提示：该篇文章作为彻底理解哈希表的第二个部分。主要讲了HashMap在Java中基于JDK1.8(不同版本HashMap可能实现不同)的具体实现。如果你对哈希表还不算太熟，建议先阅读上一篇文章，我相信等你看完之后，在回来看这篇文章，会有一种**飞翔的感觉**。

### 前言
在Java中java.util包下，定义了Map接口来实现键值对的映射关系。常用的类为HashMap,LinkedHashMap,TreeMap。其主要的类关系如下图所示：

![Map类关系图.png](https://upload-images.jianshu.io/upload_images/2824145-23a8b487bb761aac.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

在平时项目的开发中，我们主要使用的是HashMap及其子类，那我们接下来就了解一下HashMap的主要特征。

* 采用数组+链表的形式对数据进行存储。
* 根据hashCode值存储数据，访问速度较快。
* 有且只有一个key为null的数组。
* 遍历是无序的。
* 线程非安全。

### 内部结构
既然上文提到了数组+链表的形式，大家是否想起我们上篇文章提到的**链地址法**呢？如果你忘记了链地址法的具体实现，没关系，让我们一起看看在Java中HashMap具体的内部结构,具体的结构如下图所示：(注意:**在JDK1.8中如果链表的长度大于8时会将该链表转换为红黑树**)

![内部结构.png](https://upload-images.jianshu.io/upload_images/2824145-bd641cc0ee4fd0b9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

从图中看出，HashMap底层存储的是**Node**节点，本质是一个映射（键值对）。上图中，每个**黑色圆点**就是一个**Node**对象，**数组table**对应Node<K,V>[] table。

查看Node对应源码:

```
 static class Node<K,V> implements Map.Entry<K,V> {
        final int hash;
        final K key;
        V value;
        Node<K,V> next;

        Node(int hash, K key, V value, HashMapEntry<K,V> next) {...}

        public final K getKey()       {...}
        public final V getValue()      {...}
        public final String toString() {...}

        public final int hashCode(){...}

        public final V setValue(V newValue) {...}

        public final boolean equals(Object o) {...}
    }

```

从代码中我们可以看出，**Node**是HashMap的一个内部类，实现了Map.Entry接口。该类中保存了当前存储数据的hash值，关键字、和当前存储数据、及下一个Node节点的引用。既然我们已经知道了HashMap到底存储的是什么东西，那么我们继续看看HashMap的初始化。

###HashMap初始化

在我们初始化HashMap实例对象的时候，我们默认调用是其参数为空的构造函数，查看具体实现：

```
    public HashMap() {
    	  //DEFAULT_LOAD_FACTOR = 0.75
        this.loadFactor = DEFAULT_LOAD_FACTOR;
    }
```

从上，不知道大家看到熟悉的东西没**loadFactor**,还记得上篇文章我们提到的**装载因子**(我们不可能等到数组快满时，才进行扩容操作，因为会影响效率），我们发现默认情况下，HashMap初始容量为16，且装载因子为0.75，也就是当容量为12(当前数组容量*装载因子)时，进行下一次添加数据的时候，会对HashMap内部的数组进行扩容。

### HashMap放入键值对

#### 查看put方法

```
 public V put(K key, V value) {
        return putVal(hash(key), key, value, false, true);
    }
```
在HashMap调用put方法，放入键值对时，会先调用hash方法计算当前key对象的哈希值，对应hash方法如下：

```
static final int hash(Object key) {
        int h;
        return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
    }
```
hash方法内部会获取当前key的hashCode,通过当前hashCode与当前hashCode右移后的数字，进行异或运算得到哈希值。


#### 查看putVal方法
```
final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                   boolean evict) {
        Node<K,V>[] tab; Node<K,V> p; int n, i;
         //第一步，如果当前table为空，则初始化
        if ((tab = table) == null || (n = tab.length) == 0)
            n = (tab = resize()).length;
        //第二步，如果当前数据未放入，则添加
        if ((p = tab[i = (n - 1) & hash]) == null)
            tab[i] = newNode(hash, key, value, null);
        else {
            Node<K,V> e; K k;
                //第三步，如果当前key已存在，则进行覆盖操作
            if (p.hash == hash &&
                ((k = p.key) == key || (key != null && key.equals(k))))
                e = p;
                //第四步，判断该链表是否是红黑树
            else if (p instanceof TreeNode)
                e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
            else {//第五步，是否是链表
                for (int binCount = 0; ; ++binCount) {
                    if ((e = p.next) == null) {
                        p.next = newNode(hash, key, value, null);
                        //如果当前链表长度大于等于8则转会红黑树处理
                        if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                            treeifyBin(tab, hash);
                        break;
                    }
                    //如果是链表中的key已存在，则进行覆盖操作
                    if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k))))
                        break;
                    p = e;
                }
            }
            if (e != null) { 
                V oldValue = e.value;
                //第六步，是否覆盖已存在的key对应的value
                if (!onlyIfAbsent || oldValue == null)
                    e.value = value;
                afterNodeAccess(e);
                return oldValue;
            }
        }
        ++modCount;
        //如果添加完后，当前数组容量大于临界值，对数组进行扩容。
        if (++size > threshold)
            resize();
        afterNodeInsertion(evict);
        return null;
    }
```

从上述代码中我们可以看出，HashMap添加元素主要分为六个步骤。经过这六个步骤完成了相应的键值对的映射。下面我们将具体的来分析这六个步骤。

####（一）创建table数组
如果当前数组为空，会调用相应resize()方法。创建相应table数组。这里省略了扩容数组代码，因为其比较复杂，下面我们会单独进行分析。

```
 final Node<K,V>[] resize() {
        Node<K,V>[] oldTab = table;
        int oldCap = (oldTab == null) ? 0 : oldTab.length;
        int oldThr = threshold;
        int newCap, newThr = 0;
        //判断当前数组是否被创建
        if (oldCap > 0) {
            if (oldCap >= MAXIMUM_CAPACITY) {
                threshold = Integer.MAX_VALUE;
                return oldTab;
            }
            //如果当前数组到达临界值
            //数组容量为原来的2倍
            //新的临界值为原来的2倍
            else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                     oldCap >= DEFAULT_INITIAL_CAPACITY)
                newThr = oldThr << 1; // double threshold
        }
        else if (oldThr > 0)
            newCap = oldThr;
        else {            
            //默认没有数据的情况下，初始化数组，与临界值
            newCap = DEFAULT_INITIAL_CAPACITY;
            newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
        }
        if (newThr == 0) {
            float ft = (float)newCap * loadFactor;
            newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                      (int)ft : Integer.MAX_VALUE);
        }
        //设置当前临界值
        threshold = newThr;
        @SuppressWarnings({"rawtypes","unchecked"})
            Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
        table = newTab;
        
     		....省略扩容代码
     		
     	 //返回新的数组
        return newTab;
    }
```
上面的代码很好理解。判断当前数组是否为空，如果为空，则初始化当前数组，且当前数组容量为DEFAULT_INITIAL_CAPACITY=16,且临界值为12（16*0.75），最后该方法会将创建的数组进行返回。

####（二）如果当前数据未放入，则添加

```
 if ((p = tab[i = (n - 1) & hash]) == null)
            tab[i] = newNode(hash, key, value, null);
  
```

上述代码，知根据当前key值计算出来的hash值。获取对应数组中的下标，如果当前数组单元没有放入数据，则添加数据到相应的数组单元中。

####（三）如果当前key已存在，则进行覆盖操作

```
 if (p.hash == hash &&
                ((k = p.key) == key || (key != null && key.equals(k))))
        e = p;        
```
上面代码也是很好理解，如果当前数组单元有数据，且相同hash值且key值相同，那么就进行替换操作。

####（四）判断当前是否是红黑树

```
  else if (p instanceof TreeNode)
                e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
```
如果当前数组单元对应的是红黑树，那么调用相应红黑树添加方法。这里我们不讨论红黑树，这里我们只要知道。在使用红黑树的时候，查找效率是要优于传统的链表就好了。

####（五）、（六）添加元素到链表中

```
     for (int binCount = 0; ; ++binCount) {
                    if ((e = p.next) == null) {
                        p.next = newNode(hash, key, value, null);
                        if (binCount >= TREEIFY_THRESHOLD - 1) 
                        	 //如果当前链表长度大于8，转换为红黑树
                            treeifyBin(tab, hash);
                        break;
                    }
                    //获取hash值相同与key值相同，直接返回当前节点。
                    if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k))))
                        break;
                    p = e;
                }
            }
            if (e != null) { //替换相同key值的value
                V oldValue = e.value;
                if (!onlyIfAbsent || oldValue == null)
                    e.value = value;
                afterNodeAccess(e);
                return oldValue;
            }
```
这里我把第五步与第六部合并来讲解。从代码代码大家就可以理解。获取数组单元链表的长度，如果当前链表长度大于8，转换为红黑树，如果存在相同hash值或者key值相同的节点。直接替换对应的value,反之。添加键值对到相应链表中。


### HashMap的扩容机制

上面我们省略了扩容代码的具体，下面我们来仔细探讨一下HashMap的扩容机制。
主要扩容代码如下：

```
  //oldTab是原来的talble 数组
  if (oldTab != null) {
  			  //遍历原来数组单元中对应的链表，oldCap是原来数组的容量
            for (int j = 0; j < oldCap; ++j) {
                Node<K,V> e;
                if ((e = oldTab[j]) != null) {
                    oldTab[j] = null;
                    //如果数组单元只有一个节点则计算其新位置，
                    if (e.next == null)
                        newTab[e.hash & (newCap - 1)] = e;
                    //如果是红黑树，特殊处理
                    else if (e instanceof TreeNode)
                        ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                    //获取数组单元中的链表中的节点，并且重新定义位置。
                    else { // preserve order
                        Node<K,V> loHead = null, loTail = null;
                        Node<K,V> hiHead = null, hiTail = null;
                        Node<K,V> next;
                        do {
                            next = e.next;
                            //原位置的节点
                            if ((e.hash & oldCap) == 0) {
                                if (loTail == null)
                                    loHead = e;
                                else
                                    loTail.next = e;
                                loTail = e;
                            }
                            //原位置+oldCap的节点
                            else {
                                if (hiTail == null)
                                    hiHead = e;
                                else
                                    hiTail.next = e;
                                hiTail = e;
                            }
                        } while ((e = next) != null);
                        //把原位置的节点放入
                        if (loTail != null) {
                            loTail.next = null;
                            newTab[j] = loHead;
                        }
                        //把新位置的节点放入
                        if (hiTail != null) {
                            hiTail.next = null;
                            newTab[j + oldCap] = hiHead;
                        }
                    }
                }
            }
        }
```
直接去理解这段代码很难，根据上篇文章的经验，我们知道在数组进行扩容的时候，需要根据hash值去与新的数组长度进行取余运算（hash&length -1),但是从上述代码中，我们没有发现进行取余的操作。这是怎么回事呢？没事大家一起来看下图。

![取余流程.png](https://upload-images.jianshu.io/upload_images/2824145-6d1e5de02c67b206.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

上图中，我们假设某个节点hash值为1111 1111 1111 1111 11111 0000 1011 1111，并且在添加该值时，数组进行了扩容操作（为原来的数组长度的2倍）。我们发现节点在重新计算角标的时候，因为数组的长度变为之前的两倍，所以在新数组中的bit位中，始终要比原来的高一位（图中红色以表示区分），那么我们就可以根据下图得知。

![实例分析.png](https://upload-images.jianshu.io/upload_images/2824145-c6b296754a7c63c9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

从上图可以得知，只要我们通过**e.hash & oldCap==0**，我们就可以得知，该节点的新位置是在原位置，还是在原来的位置基础上+oldCap。不得不说这段代码非常优雅与巧妙，提高的效率不是吹的（因为没有重新取余去计算角标）。


### 参考
站在巨人的肩膀上。可以看得更远。
[Java 8系列之重新认识HashMap]--美团技术团队

### 最后
最后，附上我写的一个基于Kotlin 仿开眼的项目[SimpleEyes](https://github.com/AndyJennifer/SimpleEyes)(ps: 其实在我之前，已经有很多小朋友开始仿这款应用了，但是我觉得要做就做好。所以我的项目和其他的人应该不同，不仅仅是简单的一个应用。但是，但是。但是。重要的话说三遍。还在开发阶段，不要打我)，欢迎大家follow和start.



