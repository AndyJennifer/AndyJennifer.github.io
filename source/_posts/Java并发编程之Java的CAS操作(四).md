---
title: Java并发编程之Java的CAS操作(四)
date: 2019-02-23 21:31:59
categories:
- Java并发相关
tags: 
- 并发
---

![你猜.jpeg](https://upload-images.jianshu.io/upload_images/2824145-edf382cbd2befdee.jpeg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


### 前言
在上一篇文章中{% post_link Java并发编程之Java内存模型(一) %}我们描述过，物理机计算机的数据缓存不一致的时候，我们一般采用两种方式来处理。一，通过总线加锁的形式，二，通过缓存一致性协议来操作。而体现缓存一致性的正是CAS操作，CAS操作在整个Java并发框架中起着非常重要的作用。如果大家能把CAS的由来和原理彻底搞清楚，我相信对于其他关于Java中并发的问题都能迎刃而解。

### 缓存锁
在{% post_link Java并发编程之Java内存模型(一) %}文章中，在物理机计算机中当处理器中数据缓存不一致的时候，一般采用总线锁。但是总线锁把CPU和内存之前的通信锁住了，那么在锁定期间，其他的处理器是不能操作其他内存地址的数据。所以总线锁的开销比较大， 所以随着技术的进步，现在计算机已经采用了缓存锁来替代总线锁来进行性能的优化。


#### 缓存锁的原理

![cpu高速缓存.jpg](https://upload-images.jianshu.io/upload_images/2824145-b514647393adbdb1.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

我们都知道在CPU数据处理中，频繁使用的内存会缓存在处理器的L1、L2和L3高速缓存里，那么数据的操作都在处理器内部缓存中进行。并不需要声明总线锁，在目前的处理器中可以使用“缓存锁定”的方式来处理数据不一致的情况，这里所谓的“缓存锁定”是指内存区域如果被缓存在处理器的缓存中，并且在操作期间被锁定，那么当它执行锁操作会写到内存时，处理器并不会像锁总线的那样声明LOCK#信号，而是修改其对应的内存地址。同时最重要的是**其允许缓存一致性来保证数据的一致性。**
>缓存一致性核心思想：在多处理器下，为了保证各个处理器的缓存是一致的，就会实现缓存一致性协议，每个处理通过嗅探在总线上传播的数据来检查自己的缓存的值是不是过期了，当处理器发现自己缓存的数据对应的内存地址被修改，就会将当前处理器缓存的数据处理为无效，当处理器对这个数据进行修改的操作的时候，会重新从系统内存中把数据读到处理器缓存中。

### 缓存锁与CAS(Compare-and-Swap)的关系
为了实现缓存锁，在物理计算机中，Intel处理器提供了很多Lock**前缀**（注意是带Lock前缀，前缀，前缀）的指令。例如，位测试和修改指令：BTS、BTR、BTC；交换指令XADD、**CMPXCHG**，以及其他一些操作数和逻辑指令（如ADD、OR）等，被这些指令操作的内存区域就会加锁，导致其他处理器不能同时访问它。（不同的处理器实现缓存锁的指令不同，在sparc-TSO使用casa指令，而在ARM和PowerPc架构下，则需要使用一对ldrex/strex指令。）

而在Java中涉及到缓存锁的主要是CAS操作，CAS操作正是使用了不同处理器下提供的缓存锁的指令。

### CAS(Compare-and-Swap)简介
CAS指令需要三个操作数，分别是内存地址（**在Java内存模型中可以简单理解为主内存中变量的内存地址**）、旧值（**在Java内存模型中，可以理解工作内存中缓存的主内存的变量的值**）和新值。CAS操作执行时，当且仅当主内存对应的值等于旧值时，处理器用新值去更新旧值，否则它就不执行更新。但是无论是否更新了主内存中的值，都会返回旧值，上述的处理过程是一个原子操作。

对于概念类的东西，大家理解起来比较困难，这里简单举个例子如下图所示：
![CAS操作与缓存一致性.png](https://upload-images.jianshu.io/upload_images/2824145-fa9d21aa7e16f0df.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
在上图中，分别有两条线程A与B，**其中线程A优先与线程B执行a++操作**，,线程A工作内存缓存a的值为10，主内存中的a的值也为10，这个时候如果进行CAS操作，会与主内存中的a的值进行对比，如果相等会将执行a++操作后的值也就是11同步到主内存中，这个时候主内存中的值为11。当线程A执行完后，线程B接着执行，可是线程B中工作内存中缓存的a的值为8，根据缓存一致性原则。会重新去主内存读取a的值（11），此时线程B中工作内存中缓存的a的值为11，接着执行a++运算后a的值为12，此时将a的值12同步到主内存中。

#### CAS在Java中的实现
在Java中，CAS操作由sun.misc.Unsafe类里面的compareAndSwapInt()和compareAndSwapLong()，compareAndSwapObject几个方法实现。这里我们就使用compareAndSwapInt来讲解，具体代码如下：
```
  //native层
	  private static final jdk.internal.misc.Unsafe theInternalUnsafe
	   = jdk.internal.misc.Unsafe.getUnsafe();
   
	 
    public final boolean compareAndSwapInt(Object o, long offset,
                                           int expected,
                                           int x) {
        return theInternalUnsafe.compareAndSetInt(o, offset, expected, x);
    }
```
在sun.misc.Unsafe方法中，compareAndSwapInt有4个参数，第一个参数object是当前对象，第二个参数offest表示该变量在内存中的偏移地址（CAS底层是根据内存偏移位置来获取的），第三个参数expected为旧值，第四个参数x为新值。在该方法具体的细节是交给jdk.internal.misc.Unsafe类的compareAndSetInt（）方法来处理的。继续查看theInternalUnsafe下的compareAndSetInt（）方法。
```
	public final native boolean compareAndSetInt(Object o, long offset,
                                                 int expected,
                                                 int x);
```
在jdk.internal.misc.Unsafe中的compareAndSetInt也是一个本地方法。
```
    @HotSpotIntrinsicCandidate
    public final native boolean compareAndSetInt(Object o, long offset,
                                                 int expected,
                                                 int x);
```
这里具体的本地方法是在hotspot下的unsafe.cpp类具体实现的。compareAndSetInt调用unsafe.cpp中的JNI方法具体实现如下：
```
UNSAFE_ENTRY(jboolean, Unsafe_CompareAndSetInt(JNIEnv *env, jobject unsafe, jobject obj, jlong offset, jint e, jint x)) {
  oop p = JNIHandles::resolve(obj);
  if (p == NULL) {
    volatile jint* addr = (volatile jint*)index_oop_from_field_offset_long(p, offset);
    return RawAccess<>::atomic_cmpxchg(x, addr, e) == e;
  } else {
    assert_field_offset_sane(p, offset);
    return HeapAccess<>::atomic_cmpxchg_at(x, p, (ptrdiff_t)offset, e) == e;
  }
} UNSAFE_END
```
unsafe.cpp最终会调用atomic.cpp而atomic.cpp会根据不同的处理调用不同的处理器指令，这里我们还是以Intel的处理器为例，atomic.cpp最终会调用atomic_windows_x86.cpp中的operator()方法。（这里我省略了unsafe.cpp与atomic.cpp的内部细节，本身这里对C++也不是很很熟，不想误导大家，如果大家对源码比较感兴趣，这里把相关jdk源码分享给大家[jdk源码](https://pan.baidu.com/s/1Lk9yp8cEpSAnLvw5NJdqZg)）。

atomic_windows_x86.cpp中operator()方法具体如下：
```
template<>
template<typename T>
inline T Atomic::PlatformCmpxchg<4>::operator()(T exchange_value,
                                                T volatile* dest,
                                                T compare_value,
                                                atomic_memory_order order) const {
  STATIC_ASSERT(4 == sizeof(T));
  // alternative for InterlockedCompareExchange
  __asm {
    mov edx, dest 
    mov ecx, exchange_value
    mov eax, compare_value
    lock cmpxchg dword ptr [edx], ecx
  }
}
```
简单的对atomic_windows_x86.cpp中的operator()的方法进行介绍，第一个参数exchange_value为新值，第二个参数volatile* dest为变量内存地址（也就是主内存中变量地址），第三个参数compare_value为旧值（也就是工作内存中缓存的变量值）。其中在方法中，**asm是C++中的关键字**，主要作用为启动内联汇编，同时其能写在任何C++合法语句之处。它不能单独出现，必须接汇编指令、一组被大括号包含的指令或一对空括号。

那么针对于operrator中的汇编语句块进行分析，要内容分为四个部分（这里我们就把edx，ecx，eax当做存储数据的容器）：
1. mov edx, dest 将变量的内存地址赋值到edx中。
2.  mov ecx, exchange_value 将新值赋值到ecx中。
3. mov eax,compare_value  将旧值赋值到eax中。
4. lock cmpxchg dword ptr [edx], ecx  ，在了解该语句之前，我们先说三个知识点：

**cmpxchg汇编指令:**主要操作逻辑是比较eax与第一操作数的值，如果相等，那么第二操作数的值装载到第一操作数，如果不相等，那么第一操作数的值装载到eax中，其中cmpxchg 格式如下：cmpxchg 第一操作数,第二个操作数。举个例子:
##### eax对应的值与第一操作数的值相等
```
int main(){
	int a=0,b=0,c=0;
 
	__asm{
		mov eax,100; //eax 赋值为100
		mov a,eax; //将eax的值赋值给变量a，那么a的值为100
	}
	cout << "a := " << a << endl;//打印a的值
	b = 99;
	c = 11;
	__asm{
		mov ebx,b //ebx赋值为99
		cmpxchg c,ebx// eax为100，c为11，不相等，那么eax的值为11
		mov a,eax //将eax的值赋值给变量a，那么a最终的值为11
	}
	cout << "b := " << b << endl;//打印b的值
	cout << "c := " << c << endl;//打印c的值
	cout << "a := " << a << endl;//打印a的值
	return 0;
}
```
对应输出结果为a= 100,b=99,c =99,a =11。
#####  eax对应的值与第一操作数的值不相等
```
#include<iostream>
using namespace std;
int main(){
	int a=0,b=0,c=0;
 
	__asm{
		mov eax,100;
		mov a,eax// a的值为99
	}
	cout << "a := " << a << endl;//打印a的值
	b = 99;
	c = 99;
	__asm{
		mov eax,99 //eax 值为99
		mov ebx,777// ebx 值为777
		cmpxchg c,ebx// 比较eax与c的值，相等 那么c对应的值为ebx的值，也就是777
		mov a,eax//将eax的值赋值给变量a 
	}
	cout << "b := " << b << endl;//打印b的值
	cout << "c := " << c << endl;//打印c的值
	cout << "a := " << a << endl;//打印a的值
	return 0;
}
```
对应输出结果为a= 100,b=99,c =777,a =99。

**dword汇编指令：**dword ptr [edx] 简单来说，就是获取edx中内存地址中的具体的数据值。

**lock汇编指令**：lock指令做的事情比较多。这里要分为三个部分。
- 在Pentium及之前的处理器中，带有lock前缀的指令在执行期间会锁住总线。在新的处理器中，Intel使用缓存锁定来保证指令执行的原子性，缓存锁定将大大降低lock前缀指令的执行开销。
- 禁止该指令与前面和后面的读写指令重排序。
- 把写缓冲区的所有数据刷新到内存中。
额外提一句。上面的第2点和第3点所具有的内存屏障效果，保证了CAS同时具有volatile读和volatile写的内存语义。

在了解了上诉知识点后，我们再来理解语句4就很好理解了。如果主内存中的值与旧值（也就是工作内存中缓存的变量值）不同，那么工作内存中的缓存的变量值（也就是旧值）就为主内存中的值。如果相同。那么主内存中的值就为最新的值。

### CAS会出现的三大问题
虽然通过CAS操作可以很好的提高我们在处理数据的时候的效率，但是任然会出现许多问题。但是Java的开发团队已经为我们提供了一些处理方案，现在我们就来看看CAS有哪三大问题。
#### ABA问题
因为CAS需要在操作值的时候，检查值有没有发生变化，如果没有发生变化则更新，但是如果一个值原来是A，变成了B，又变成了A，那么使用CAS进行检查时会发现它的值没有发生变化，但是实际上却变化了。ABA问题的解决思路就是使用版本号。在变量前面追加上版本号，每次变量更新的时候把版本号加1，那么A→B→A就会变成1A→2B→3A。从Java 1.5开始，JDK的Atomic包里提供了一个类AtomicStampedReference来解决ABA问题。这个类的compareAndSet方法的作用是首先检查当前引用是否等于预期引用，并且检查当前标志是否等于预期标志，如果全部相等，则以原子方式将该引用和该标志的值设置为给定的更新值。关于AtomicStampedReference的使用，有兴趣的小伙伴可以自行查看相关源码实现。

#### 循环时间开销太大
在后期的文章我们会讲述自旋CAS，关于自旋CAS,因为后期关于锁的文章会具体描述，这里我就简单描述一下，在Java中有很多的并发框架都使用了自旋CAS来获取相应的锁，会一直循环直到获取到相应的锁后，然后执行相应的操作。那么当其自旋时CAS，会一直占用CPU的资源。如果自旋CAS长时间不成功，会给CPU带来非常大的执行开销。

#### 只能保证一个共享变量的原子操作
当对一个共享变量执行操作时，我们可以使用循环CAS的方式来保证原子操作，但是对多个共享变量操作时，自旋CAS就无法保证操作的原子性，这个时候就可以用锁。还有一个取巧的办法，就是把多个共享变量合并成一个共享变量来操作。比如，有两个共享变量i＝2，j=a，合并一下ij=2a，然后用CAS来操作ij。从Java 1.5开始，JDK提供了AtomicReference类来保证引用对象之间的原子性，就可以把多个变量放在一个对象里来进行CAS操作。关于AtomicReference的使用，有兴趣的小伙伴可以自行查看相关源码实现。

### 总结
 - 对于物理计算机中的缓存锁，在Java中是使用CAS操作来实现的。
 - CAS操作中会出现三个问题，ABA问题。循环时间开销太大，只能保证一个共享变量的原子操作。