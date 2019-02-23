![消息池.gif](https://upload-images.jianshu.io/upload_images/2824145-0be39b98baebb871.gif?imageMogr2/auto-orient/strip)

>该文章属于Android Handler系列文章，如果想了解更多，请点击
[《Android Handler机制之总目录》](https://www.jianshu.com/p/43bb31d8a742)

### 前言
在上篇文章[《Android Handler机制之Message及Message回收机制 》](https://www.jianshu.com/p/d0ef4edd4407)我们讲解了Message中所携带的信息及消息池中的实现方式。其中我们已经了解了Message中消息池是以链表的形式来完成。在完成了上篇文章后，我就一直想在Java或Android中是否已经为我们提供了一种对象池来帮助我们来实现缓存对象的实现呢。果真在Android中的android.support.v4.util下的Pools类就为我们提供了SimplePool、SynchronizedPool来创建对象池。下面我就对该包下的类进行讲解。

### Android中提供的对象池
在android.support.v4.util包下的Pools类中，分别声明了Pool接口，SimplePool实现类与SynchronizedPool实现类，其中具体的UML关系如下图所示：
![继承关系.png](https://upload-images.jianshu.io/upload_images/2824145-0cc634c790076912.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
在上图中，Pool的具体实现类为SimplePool，而SynchronizedPool为SimplePool的子类。

### 简单对象池（SimplePool）
在讨论了具体的UML关系后，现在我们来看看SimplePool的代码实现，具体代码如下图所示：
```
  public static class SimplePool<T> implements Pool<T> {
        private final Object[] mPool;//存储对象的数组
        private int mPoolSize;//当前对象池中的对象个数
		
        public SimplePool(int maxPoolSize) {
            if (maxPoolSize <= 0) {
                throw new IllegalArgumentException("The max pool size must be > 0");
            }
            mPool = new Object[maxPoolSize];//初始化对象池的最大容量
        }
		//从对象池中获取数据
        @Override
        @SuppressWarnings("unchecked")
        public T acquire() {
            if (mPoolSize > 0) {
                final int lastPooledIndex = mPoolSize - 1;
                T instance = (T) mPool[lastPooledIndex];
                mPool[lastPooledIndex] = null;
                mPoolSize--;//当前对象池中对象个数减1
                return instance;
            }
            return null;
        }
		
		//回收当前对象到对象池中，
        @Override
        public boolean release(@NonNull T instance) {
            if (isInPool(instance)) {//如果对象池中已经有当前对象了，会抛出异常
                throw new IllegalStateException("Already in the pool!");
            }
            if (mPoolSize < mPool.length) {
                mPool[mPoolSize] = instance;
                mPoolSize++;
                return true;
            }
            return false;
        }
		
		//判断当前对象是否在对象池中
        private boolean isInPool(@NonNull T instance) {
            for (int i = 0; i < mPoolSize; i++) {
                if (mPool[i] == instance) {
                    return true;
                }
            }
            return false;
        }
    }
```
对于SimplePool的代码其实很好理解，其对象池是以数组的方式来实现的。其中对象池的最大容量是通过用户手动设定。从对象池中获取数据是通过acquire方法。回收当前对象到对象池中是通过release方法。关于这两个方法的详细流程会在下文具体介绍。
#### 关于acquire方法
在acquire方法中，会从对象池中取出对象。具体列子如下图所示：
![acquire.png](https://upload-images.jianshu.io/upload_images/2824145-f78d7c7a39ac57c7.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
在上图中，当前对象池中存储了10个object对象，当前sPoolSize = 10。当调用acquire（）方法时，会获取最后一个对象（也就是 mPool[9],将该对象取出后，会将该位置置为null（mPool9] =null)，当前sPoolSize = 9。当再次调用acquire（）方法时，会获取mPool[8]位置下的对象。同理将该位置置为null，当前sPoolSize = 8。

总结：**acquire()方法总会取当前对象池中存储的最后一个数据。如果有则返回。同时将该位置置为null。反之返回为null。**

#### 关于release方法
在release方法中，会将对象缓存到对象池中。如果当前对象已经存在，会抛出异常。反之则存储。具体列子如下图所示：
![realase.png](https://upload-images.jianshu.io/upload_images/2824145-4a6a47a74c4d2913.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
在上图中。当前对象池存储了8个object对象。当前sPoolSize = 8。当调用realse方法将object9对象回收到对象池中去时，会存储在 mPool[8]位置下。且当前sPoolSize = 9，那么当再次调用realse方法将object10对象放入时，会将object10对象存储在mPool[9]位置下。

总结：**release( T instance)方法，总会将需要回收的对象存入当前对象池中存储的最后一个数据的下一个位置。如果当前回收的对象已经存在会抛出异常。反之则成功。**

### 同步对象池（SynchronizedPool）
在前面的文章中我们介绍了SimplePool的存取数据的主要实现。细心的小伙伴肯定都已经发现了。在多线程的情况下，如果使用SimplePool肯定是会出现问题的。但是Google已经为我们考虑到了，为我们提供了线程安全的对象池SynchronizedPool，下面我们就来看看SynchronizedPool的具体实现。具体代码如下
```
  public static class SynchronizedPool<T> extends SimplePool<T> {
        private final Object mLock = new Object();

        /**
         * Creates a new instance.
         *
         * @param maxPoolSize The max pool size.
         *
         * @throws IllegalArgumentException If the max pool size is less than zero.
         */
        public SynchronizedPool(int maxPoolSize) {
            super(maxPoolSize);
        }

        @Override
        public T acquire() {
            synchronized (mLock) {
                return super.acquire();
            }
        }

        @Override
        public boolean release(@NonNull T element) {
            synchronized (mLock) {
                return super.release(element);
            }
        }
    }
```
SynchronizedPool的代码理解起来也同样非常简单，直接继承SimplePool。并重写了SimplePool的两个方法。并为其加上了锁，保证了多线程情况下使用的安全性。

### 对象池的使用
上面我们讨论了两种不同的对象池的实现，下面我们来看看对于这两种对象池的使用。这里就使用官方的SynchronizedPool的使用例子（这里对SimplePool的使用也是通用的，根据是否需要多线程操作来选择不同的对象池）。
```
  public class MyPooledClass {
	 
	  //声明对象池的大小
      private static final SynchronizedPool<MyPooledClass> sPool =
              new SynchronizedPool<MyPooledClass>(10);
              
	  //从对象池中获取数据，如果为null,则创建
      public static MyPooledClass obtain() {
         MyPooledClass instance = sPool.acquire();
        return (instance != null) ? instance : new MyPooledClass();
       }
	  
	  //回收对象到对象池中。当然你也可以清除对象的状态
      public void recycle() {
           // 清除对象的状态，如果你自己需要的话，
          sPool.release(this);
      }
 }
```

### 总结
- 对于频繁创建的对象，可以考虑使用对象池。
- 实现对象池的方式有几种，可以采用数组的形式，也可以采用链表的形式。
- 在实现对象池缓存对象时，需要考虑到线程安全的问题。该加锁就加锁。
