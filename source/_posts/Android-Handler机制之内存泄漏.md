---
title: Android Handler机制之内存泄漏
date: 2019-02-23 22:11:10
categories:
- Handler相关
tags: 
- 异步任务
---

{% asset_img 溢出啦啦.jpg 溢出啦啦 %}

>该文章属于Android Handler系列文章，如果想了解更多，请点击{% post_link Android-Handler机制之总目录 %}

### 前言

整个Handler机制系列文章到此就结束了，相信大家基本已经将整个Handler机制消化的差不多了，现在就剩下最后一个知识点，在平时开发中使用Handler有可能会`导致内存泄漏`的问题。下面我们就一起去了解了解~~

### 内存泄漏

内存泄漏在官方的定义如下:
>内存泄漏（Memory Leak）是指程序中己动态分配的堆内存由于某种原因程序未释放或无法释放，造成系统内存的浪费，导致程序运行速度减慢甚至系统崩溃等严重后果。

那么翻译成人话，就是当一个对象不再被使用时，本该被系统回收，但却因为有另外一个正在使用的对象持有它的引用，导致其不能被回收，造成内存的浪费。

那么针对于Android系统来说，在Android系统中会为每个应用程序分配相应的内存（根据手机厂商的不同，分配的内存大小会有所差异）。也就是说对于每一个应用程序来说其内存是有限的。那么当某个应用程序产生的内存泄漏较多时，导致达到应用总的内存阀值，那么就会导致应用崩溃。

### Handler内存泄漏的情况讨论

为了分析Handler内存泄漏的具体情况，请参看以下示例代码：

```java
public class HandlerLeakageActivity extends BaseActivity {

    public static final int UPDATE_UI = 1;

    private Handler mHandler = new Handler() {
        @Override
        public void handleMessage(Message msg) {
            if (msg.what == UPDATE_UI) {
                updateUI();
            }
        }
    };

    @Override
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_handler_leakage);
        Message message = Message.obtain();
        message.what = UPDATE_UI;
        mHandler.sendMessageDelayed(message, 1000 * 3600 * 24);//发送延时24小时消息
    }

    //更新ui
    private void updateUI() {...}
 }
```

上述代码逻辑很简单，我们在HandlerLeakageActivity 中创建了内部类Handler，同时发送了一个延时为24小时的消息。当HandlerLeakageActivity 收到这个延迟消息后，那么接着会来更新UI，同时我们可以得到以下引用链：

{% asset_img handler_refrence.png handler_refrence %}
其中的内部类Handler 拥有当前Activity的引用，是因为`在Java中，非静态内部类会持有外部类的引用`，而Messagey拥有Handler的引用，是因为Message通过Looper的loop（）方法取出后，需要相应的Handler来处理消息（`msg.target ==发送消息的Handler`）。

那么在整个Handler机制下的引用关系如下图所示：
{% asset_img handler_leakage.png handler_leakage %}

参照上图，我们设想一种情况，假设我们在程序启动的时候，首先进入HandlerLeakageActivity ，然后又将其finish掉。那么就会出现，因为延迟消息的迟迟不能被取出执行，导致该Activity不能被系统回收。从而造成上文我们提到过的`内存泄漏`。

#### 那么问题来了，什么时候引用链会断开

在文章{% post_link Android-Handler机制之Message及Message回收机制 %}
中，我们曾经提到过，当消息被Looper通过Loop（）方法取出并执行的时候，会执行recycleUnchecked（）方法来重置消息中的数据，具体代码如下：

```java
void recycleUnchecked() {
        flags = FLAG_IN_USE;
        what = 0;
        arg1 = 0;
        arg2 = 0;
        obj = null;
        replyTo = null;
        sendingUid = -1;
        when = 0;
        target = null;//将关联handler对象置为null
        callback = null;
        data = null;
        //省略部分代码
    }
```

在该方法中将`target =null`，其中消息中的`target`为当前发送该消息的Handler对象。也就说只有消息被取出执行后，整个引用链才会断开，那么相应的Handler与使用该Handler的Activity才会被系统回收。

### 解决方法

通过上文Handler内存泄漏的问题分析，导致这种情况的发生的原因是内部类Handler拥有当前Activity的引用。那么解决只要解决这个问题，我们就能处理Handler内存泄漏啦。

#### 使用静态内部类+弱引用的方式

```java
public class HandlerLeakageActivity extends BaseActivity {
    public static final int UPDATE_UI = 1;

    private MyHandler mHandler = new MyHandler(this);
    //使用静态内部类
    private static class MyHandler extends Handler {

        private final WeakReference<HandlerLeakageActivity> mWeakReference;
        MyHandler(HandlerLeakageActivity activity) {
            mWeakReference = new WeakReference<HandlerLeakageActivity>(activity);
        }

        @Override
        public void handleMessage(Message msg) {
            HandlerLeakageActivity activity = mWeakReference.get();
            if (activity != null) {
                activity.updateUI();
            }
        }
    }
        //更新ui
    private void updateUI() {...}
   }
```

为了保证不再持有当前Activity的引用，我们采用静态内部类的方式（`静态内部类不会持有外部类引用`），同时为了让Handler在处理消息的时候，能够调用外部类Activity的方法，所以我们这里采用`弱引用`的方式。

##### 为什么要使用弱引用

在Java中判断一个对象到底是不是需要回收，都跟引用相关。在Java中引用分为了4类。

- 强引用：只要引用存在，垃圾回收器永远不会回收Object obj = new Object();而这样 obj对象对后面new Object的一个强引用，只有当obj这个引用被释放之后，对象才会被释放掉。
- 软引用：是用来描述，一些还有但并非必须的对象，对于软引用关联着的对象，在系统将要发生内存溢出异常之前，将会把这些对象列进回收范围之中进行第二次回收。（SoftReference)
- 弱引用：也是用来描述非必须的对象，但是它的强度要比软引用更弱一些。被弱引用关联的对象只能生存到下一次垃圾收集发生之前，当垃圾收集器工作时，无论当前内存是否足够，都回回收掉被弱引用关联的对象。（WeakReference)
- 虚引用：也被称为幽灵引用，它是最弱的一种关系。一个对象是否有引用的存在，完全不会对其生存时间构成影响，也无法通过一个虚引用来取得一个实例对象。
  
#### 另创建一个类+弱引用的方式

如果你不想使用静态内部类+弱引用的方式，你也可以采用新建一个Handler类文件+弱引用的方式。这两种代码基本差不多，这里就不过多进行介绍了。

#### 当外部类生命周期结束时，清空消息

如果你不想采用上述的两种方式，还有一种方法就是在当前Activity被finish掉的时候，移除掉整个消息队列中的所有消息。这样就能保证Activity与Handler没有直接的引用关系啦。

关于消息的删除主要有三种方法，大家可以根据自己的项目需求来选择相应的方法。具体如下所示：

>（关于消息的删除，如果有同学不是很熟悉，请参看{% post_link Android-Handler机制之Message及Message回收机制 %}

```java
void removeMessages(Handler h, int what, Object object)
void removeMessages(Handler h, Runnable r, Object object)
void removeCallbacksAndMessages(Handler h, Object object)
void removeCallbacksAndMessages(Object token)
```

结合Activity的生命周期，具体代码如下所示：

```java
 @Override
    protected void onDestroy() {
        super.onDestroy();
        //这里token传null,会移除消息队列中所有当前Handler发送且未被执行的消息
        mHandler.removeCallbacksAndMessages(null);
```

这里我使用removeCallbacksAndMessages(Object token) 方法来清空消息，需要注意的是如果token=null,该方法会`移除消息队列中所有当前Handler发送且未被执行的消息`。

### 总结

- Handler在使用时，如果直接采用内部类，有可能会导致内存泄漏。
- Handler内存泄漏的主要原因是，内部类Handler拥有外部类Activity的引用，且不能保证消息的发送是否有较长延时。
- 解决Handler内存泄漏的主要方法有，采用静态内部类+弱引用，当外部类生命周期结束时，清空消息等。
