---
title: 自定义View事件篇进阶篇(七)-NestedScrolling(嵌套滑动)机制
tags:
  - 嵌套滑动
categories:
  - View事件篇
---

### 前言
在Lollipop(Android 5.0)时，谷歌推出了NestedScrolling机制，也就是嵌套滑动。该机制主要是解决在传统事件分发机制中比较难处理的嵌套滑动效果。从Lollipop（(Android 5.0)及以上版本所有的View都已经支持这套机制了。在Lollipop（(Android 5.0)及以下版本，也可以通过support v4包中提供的相关类进行支持。

### 传统滑动机制的局限性
在传统的事件分发机制中，当一个事件产生后，它的传递过程遵循如下顺序:父控件->子控件,事件总是先传递给父控件,当父控件不对事件拦截的时候，那么当前事件又会传递给它的子控件。一旦父控件需要拦截事件，那么子控件是没有机会接受该事件的。因此当在传统机制中，如果有嵌套滑动场景，我们需要手动解决事件冲突。具体嵌套滑动例子如下图所示：

![例子分析.gif](https://upload-images.jianshu.io/upload_images/2824145-0d7b89c637444baa.gif?imageMogr2/auto-orient/strip)


想要实现上图效果，在传统滑动机制中，我们需要调用父控件中onInterceptTouchEvent方法来拦截向上滑动。拦截后，父控件在onTouchEvent滑动至headerView隐藏，当headerView隐藏后，父控件就不继续滑动了（因为父控件拦截了事件，所以同一事件序列的事件，任然会传递给父控件,也就会调用其onTouchEvent方法），然后我们需要通过父控件中的onTouchEvent方法将剩余的事件强行传递给RecyclerView（或ListView），使其也能滚动。由此可见，只要项目中出现嵌套滑动，我们都必须为其写一套逻辑。这样解决问题的方式太单一且重复。所以为了帮助开发者们快速的处理该问题，谷歌推出了NestedScrolling机制。

### NestedScrolling机制
NestedScrolling机制与传统的事件分发机制最大的不同，就是NestedScrolling机制中的事件是可以共享的（事件由子控件传递到父控件，父控件不能拦截事件）。整个处理过程主要分为以下几个步骤：

- 当子控件收到滑动事件，准备要滑动时，会先询问父控件是否要滑动。
- 如果父控件需要滑动，那么就会通知子控件它具体消耗了多少滑动距离。然后交由子控件处理剩余的滑动距离。
- 子控件滑动结束后，如果滑动距离还有剩余，就会再问一下父控件是否需要在继续滑动剩下的距离。

为了实现该机制，谷歌在Lollipop(Android 5.0)时，为系统的自带的ViewGroup和View都增加了相应的API。同时为也了兼容低版本(5.0以下，View与ViewGroup是没有对应的API)，谷歌在support v4包中也提供了如下类与接口进行支撑：

- NestedScrollingParent（接口）
- NestedScrollingParent2（接口）
- NestedScrollingParentHelper（类）
- NestedScrollingChild（接口）
- NestedScrollingChild2（接口）
- NestedScrollingChildHelper（类）

其中NestedScrollingParent与NestedScrollingChild就是将5.0中ViewGoup与View增加的方法抽象为接口，NestedScrollingParentHelper与NestedScrollingChildHelper也就对应着5.0中ViewGoup与View增加的方法的具体实现。NestedScrollingParent2与NestedScrollingChild2接口是为了处理嵌套fling效果的，这个知识点会在下文进行介绍。这里大家可以选择忽略。

需要注意的是，如果你的Android平台在5.0以上，那么你可以直接使用系统ViewGoup与View自带的方法。但是为了向下兼容，建议还是使用support v4包提供的相应接口来实现嵌套滑动。本文也会以实现接口的方式来进行讲解。

#### NestedScrollingParent接口介绍

如果采用接口的方式实现嵌套滑动，我们需要父控件要实现NestedScrollingParent接口。接口具体方法如下：
```
    /**
     * 有嵌套滑动到来了，判断父控件是否接受嵌套滑动
     *
     * @param child            嵌套滑动对应的父类的子类(因为嵌套滑动对于的父控件不一定是一级就能找到的，可能挑了两级父控件的父控件，child的辈分>=target)
     * @param target           具体嵌套滑动的那个子类
     * @param nestedScrollAxes 支持嵌套滚动轴。水平方向，垂直方向，或者不指定
     * @return 父控件是否接受嵌套滑动， 只有接受了才会执行剩下的嵌套滑动方法
     */
    public boolean onStartNestedScroll(View child, View target, int nestedScrollAxes) {}

    /**
     * 当onStartNestedScroll返回为true时，也就是父控件接受嵌套滑动时，该方法才会调用
     */
    public void onNestedScrollAccepted(View child, View target, int axes) {}

    /**
     * 在嵌套滑动的子控件未滑动之前，判断父控件是否优先与子控件处理(也就是父控件可以先消耗，然后给子控件消耗）
     *
     * @param target   具体嵌套滑动的那个子类
     * @param dx       水平方向嵌套滑动的子控件想要变化的距离 dx<0 向右滑动 dx>0 向左滑动
     * @param dy       垂直方向嵌套滑动的子控件想要变化的距离 dy<0 向下滑动 dy>0 向上滑动
     * @param consumed 这个参数要我们在实现这个函数的时候指定，回头告诉子控件当前父控件消耗的距离
     *                 consumed[0] 水平消耗的距离，consumed[1] 垂直消耗的距离 好让子控件做出相应的调整
     */
    public void onNestedPreScroll(View target, int dx, int dy, int[] consumed) {}

    /**
     * 嵌套滑动的子控件在滑动之后，判断父控件是否继续处理（也就是父消耗一定距离后，子再消耗，最后判断父消耗不）
     *
     * @param target       具体嵌套滑动的那个子类
     * @param dxConsumed   水平方向嵌套滑动的子控件滑动的距离(消耗的距离)
     * @param dyConsumed   垂直方向嵌套滑动的子控件滑动的距离(消耗的距离)
     * @param dxUnconsumed 水平方向嵌套滑动的子控件未滑动的距离(未消耗的距离)
     * @param dyUnconsumed 垂直方向嵌套滑动的子控件未滑动的距离(未消耗的距离)
     */
    public void onNestedScroll(View target, int dxConsumed, int dyConsumed, int dxUnconsumed, int dyUnconsumed) {}

    /**
     * 嵌套滑动结束
     */
    public void onStopNestedScroll(View child) {}

    /**
     * 当子控件产生fling滑动时，判断父控件是否处拦截fling，如果父控件处理了fling，那子控件就没有办法处理fling了。
     *
     * @param target    具体嵌套滑动的那个子类
     * @param velocityX 水平方向上的速度 velocityX > 0  向左滑动，反之向右滑动
     * @param velocityY 竖直方向上的速度 velocityY > 0  向上滑动，反之向下滑动
     * @return 父控件是否拦截该fling
     */
    public boolean onNestedPreFling(View target, float velocityX, float velocityY) {}


    /**
     * 当父控件不拦截该fling,那么子控件会将fling传入父控件
     *
     * @param target    具体嵌套滑动的那个子类
     * @param velocityX 水平方向上的速度 velocityX > 0  向左滑动，反之向右滑动
     * @param velocityY 竖直方向上的速度 velocityY > 0  向上滑动，反之向下滑动
     * @param consumed  子控件是否可以消耗该fling，也可以说是子控件是否消耗掉了该fling
     * @return 父控件是否消耗了该fling
     */
    public boolean onNestedFling(View target, float velocityX, float velocityY, boolean consumed) {}

    /**
     * 返回当前父控件嵌套滑动的方向，分为水平方向与，垂直方法，或者不变
     */
    public int getNestedScrollAxes() {}
```

#### NestedScrollingChild接口介绍
如果采用接口的方式实现嵌套滑动，子控件需要实现NestedScrollingChild接口。接口具体方法如下：，
```
    /**
     * 开启一个嵌套滑动
     *
     * @param axes 支持的嵌套滑动方法，分为水平方向，竖直方向，或不指定
     * @return 如果返回true, 表示当前子控件已经找了一起嵌套滑动的view
     */
    public boolean startNestedScroll(int axes) {}

    /**
     * 在子控件滑动前，将事件分发给父控件，由父控件判断消耗多少
     *
     * @param dx             水平方向嵌套滑动的子控件想要变化的距离 dx<0 向右滑动 dx>0 向左滑动
     * @param dy             垂直方向嵌套滑动的子控件想要变化的距离 dy<0 向下滑动 dy>0 向上滑动
     * @param consumed       子控件传给父控件数组，用于存储父控件水平与竖直方向上消耗的距离，consumed[0] 水平消耗的距离，consumed[1] 垂直消耗的距离
     * @param offsetInWindow 子控件在当前window的偏移量
     * @return 如果返回true, 表示父控件已经消耗了
     */
    public boolean dispatchNestedPreScroll(int dx, int dy, @Nullable int[] consumed, @Nullable int[] offsetInWindow) {}


    /**
     * 当父控件消耗事件后，子控件处理后，又继续将事件分发给父控件,由父控件判断是否消耗剩下的距离。
     *
     * @param dxConsumed     水平方向嵌套滑动的子控件滑动的距离(消耗的距离)
     * @param dyConsumed     垂直方向嵌套滑动的子控件滑动的距离(消耗的距离)
     * @param dxUnconsumed   水平方向嵌套滑动的子控件未滑动的距离(未消耗的距离)
     * @param dyUnconsumed   垂直方向嵌套滑动的子控件未滑动的距离(未消耗的距离)
     * @param offsetInWindow 子控件在当前window的偏移量
     * @return 如果返回true, 表示父控件又继续消耗了
     */
    public boolean dispatchNestedScroll(int dxConsumed, int dyConsumed, int dxUnconsumed, int dyUnconsumed, @Nullable int[] offsetInWindow) {}

    /**
     * 子控件停止嵌套滑动
     */
    public void stopNestedScroll() {}


    /**
     * 当子控件产生fling滑动时，判断父控件是否处拦截fling，如果父控件处理了fling，那子控件就没有办法处理fling了。
     *
     * @param velocityX 水平方向上的速度 velocityX > 0  向左滑动，反之向右滑动
     * @param velocityY 竖直方向上的速度 velocityY > 0  向上滑动，反之向下滑动
     * @return 如果返回true, 表示父控件拦截了fling
     */
    public boolean dispatchNestedPreFling(float velocityX, float velocityY) {}

    /**
     * 当父控件不拦截子控件的fling,那么子控件会调用该方法将fling，传给父控件进行处理
     *
     * @param velocityX 水平方向上的速度 velocityX > 0  向左滑动，反之向右滑动
     * @param velocityY 竖直方向上的速度 velocityY > 0  向上滑动，反之向下滑动
     * @param consumed  子控件是否可以消耗该fling，也可以说是子控件是否消耗掉了该fling
     * @return 父控件是否消耗了该fling
     */
    public boolean dispatchNestedFling(float velocityX, float velocityY, boolean consumed) {}
    
    /**
     * 设置当前子控件是否支持嵌套滑动，如果不支持，那么父控件是不能够响应嵌套滑动的
     *
     * @param enabled true 支持
     */
    public void setNestedScrollingEnabled(boolean enabled) {}

    /**
     * 当前子控件是否支持嵌套滑动
     */
    public boolean isNestedScrollingEnabled() {}

    /**
     * 判断当前子控件是否拥有嵌套滑动的父控件
     */
    public boolean hasNestedScrollingParent() {}
```
###  嵌套滑动过程分析
通过上文，我相信大家能基本了解这两个接口方法的作用，但是我们并不知道这些方法之间对应的关系与调用的时机。我们已经知道嵌套滑动的机制，也了解了谷歌在5.0中为ViewGroup与View增加的方法。那么现在我们一起来了解如何调用这些方法，来实现我们想要的嵌套滑动效果吧。想要实现嵌套滑动效果，根据嵌套滑动的原理，事件必须传递到子控件，也就是说父控件是不能拦截事件的。当子控件想要将事件交给父控件进行预处理，那么必然会在其onTouchEvent方法，将事件传递给父控件。也就是如下情况：

#### 子控件如何找到嵌套滑动的父控件

#### 子控件如何将滑动事件传给父控件

#### 子控件如何将fling传给父控件

#### NestedScrollingChild与NestedScrollingParent方法调用的关系

### 嵌套滑动基本范式

```

```

### NestedScrollingChild2与NestedScrollingChild 区别
parent很可能并不想一下子消费整个fling手势，而是像响应一个scroll一样去处理

#### 

### 最后

站在巨人的肩膀上，才能看的更远~

https://www.jianshu.com/p/e333f11f29aa
https://www.jianshu.com/p/f09762df81a5
https://blog.csdn.net/suyimin2010/article/details/81900757

https://cloud.tencent.com/developer/article/1383134 coordinatorLayout资料

https://www.jianshu.com/p/1cf7e9ade0f8 fling 效果的解释

https://www.jianshu.com/p/c138055af5d2 判断recyclerView到底底部的方法
https://www.jianshu.com/p/f55abc60a879 NestedScrollingChild2与NestedScrollingChild 区别

https://www.jianshu.com/p/700ed36e83ff