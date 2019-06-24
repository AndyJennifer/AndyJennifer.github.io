---
title: 《自定义View事件篇进阶篇(七)-嵌套滑动机制》
tags:
  - 嵌套滑动
categories:
  - View事件篇
---

### 前言



### 基本思路

### View的方法

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
是为了解决5.0之前view是不支持嵌套滑动的
### 最后
https://www.jianshu.com/p/e333f11f29aa
https://www.jianshu.com/p/f09762df81a5
https://blog.csdn.net/qq_42944793/article/details/88417127
站在巨人的肩膀上，才能看的更远~