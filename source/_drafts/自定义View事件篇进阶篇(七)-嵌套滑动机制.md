---
title: 自定义View事件篇进阶篇(七)-嵌套滑动机制
tags:
  - 嵌套滑动
categories:
  - View事件篇
---

### 前言



### 基本思路

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
这里主要讲两个接口之间的区别,主要是处理fling效果，如果父view消耗掉了fling,那么子view时不能处理fling效果的,但是fling效果又是嵌套滑动，又会调用startNestedScroll。就会又走，NestedScrollingParent的onStartNestedScroll与onNestedScrollAccepted，所以在实现了NestedScrollingParent中是不能判断到底是手动滑动，还是fling效果的滑动，就会造成动画效果的异常。所以增加了这个类型的判断。


### 最后
https://www.jianshu.com/p/e333f11f29aa
https://www.jianshu.com/p/f09762df81a5
https://blog.csdn.net/qq_42944793/article/details/88417127
https://blog.csdn.net/suyimin2010/article/details/81900757

https://cloud.tencent.com/developer/article/1383134 coordinatorLayout资料
站在巨人的肩膀上，才能看的更远~