---
title: 自定义View事件篇进阶篇(七)-NestedScrolling(嵌套滑动)机制
tags:
  - 嵌套滑动
categories:
  - View事件篇
---

### 前言
在Lollipop(Android 5.0)时，谷歌推出了NestedScrolling机制，也就是嵌套滑动。该机制主要是解决在传统事件分发机制中比较难处理的嵌套滑动效果。从Lollipop（(Android 5.0)及以上版本所有的View都已经支持这套机制了。在Lollipop（(Android 5.0)及以下版本，也可以通过support v4包中提供的相关类进行支持。

### 嵌套滑动机制与传统滑动机制的区别
在传统的事件分发机制中，当一个事件产生后，它的传递过程遵循如下顺序:父View->子View,事件总是先传递给父View,当父View不对事件拦截的时候，那么当前事件又会传递给它的子View。一旦父View需要拦截事件，那么子View是没有机会接受该事件的。因此当在传统机制中，如果有嵌套滑动场景，我们需要手动解决事件冲突。具体嵌套滑动例子如下图所示：

![例子分析.gif](https://upload-images.jianshu.io/upload_images/2824145-0d7b89c637444baa.gif?imageMogr2/auto-orient/strip)


想要实现上图效果，在传统滑动机制中，我们需要调用父view中onInterceptTouchEvent方法来拦截向上滑动。拦截后，父view在onTouchEvent滑动至headerView隐藏，当headerView隐藏后，父View就不继续滑动了，然后我们需要通过父View中的onTouchEvent方法将剩余的事件强行传递给RecyclerView。使其也能滚动。



嵌套滑动机制的原理

嵌套滑动机制 的基本原理可以认为是事件共享，即当子控件接收到滑动事件，准备要滑动时，会先通知父控件(startNestedScroll）；然后在滑动之前，会先询问父控件是否要滑动（dispatchNestedPreScroll)；如果父控件响应该事件进行了滑动，那么就会通知子控件它具体消耗了多少滑动距离；然后交由子控件处理剩余的滑动距离；最后子控件滑动结束后，如果滑动距离还有剩余，就会再问一下父控件是否需要在继续滑动剩下的距离

### 嵌套滑动实现


### View的方法

讲讲对应的什么周期方法，那个时间调用了view那个方法，会走那个的方法。
```
startNestedScroll（int axes)
stopNestedScroll()
dispatchNestedPreScroll(int, int, int[], int[])
dispatchNestedScroll(int, int, int, int, int[])
```


### ViewGroup
```
    public boolean onStartNestedScroll(View child, View target, int nestedScrollAxes);
    public void onNestedScrollAccepted(View child, View target, int nestedScrollAxes);
    public void onStopNestedScroll(View target);
    public void onNestedScroll(View target, int dxConsumed, int dyConsumed,
            int dxUnconsumed, int dyUnconsumed);
    public void onNestedPreScroll(View target, int dx, int dy, int[] consumed);
    public boolean onNestedFling(View target, float velocityX, float velocityY, boolean consumed);
    public boolean onNestedPreFling(View target, float velocityX, float velocityY);
```

### NestedScrollingChildHelper与NestedScrollingParentHelper

是为了解决5.0之前view是不支持嵌套滑动的,要实现接口，NestedScrollingParent与NestedScrollingChild

### NestedScrollingChild2与NestedScrollingChild 区别
parent很可能并不想一下子消费整个fling手势，而是像响应一个scroll一样去处理


### 最后
https://www.jianshu.com/p/e333f11f29aa
https://www.jianshu.com/p/f09762df81a5
https://blog.csdn.net/suyimin2010/article/details/81900757

https://cloud.tencent.com/developer/article/1383134 coordinatorLayout资料

https://www.jianshu.com/p/1cf7e9ade0f8 fling 效果的解释

https://www.jianshu.com/p/c138055af5d2 判断recyclerView到底底部的方法
https://www.jianshu.com/p/f55abc60a879 child2的判断
站在巨人的肩膀上，才能看的更远~