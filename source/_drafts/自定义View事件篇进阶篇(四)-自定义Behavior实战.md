---
title: 自定义View事件篇进阶篇(四)-自定义Behavior实战
tags:
  - 嵌套滑动
categories:
  - View事件篇
---

### 前言

在上篇文章{% post_link 自定义View事件篇进阶篇(三)-CoordinatorLayout与Behavior %}中，我们介绍了CoordainatorLayout下的Behavior机制，为了帮助大家更好的理解并运用Behavior，现在我们通过一个例子，来巩固我们之前学习的知识点。

> 该博客中涉及到的示例，在[NestedScrollingDemo](https://github.com/AndyJennifer/NestedScrollingDemo)项目中都有实现，大家可以按需自取。

### 效果展示

我们先看一下，我们需要实现的效果吧，如下图所示：

![例子展示](自定义View事件篇进阶篇(四)-自定义Behavior实战/例子展示.gif)

从效果上来看，这种效果类似于CoordinatorLayout与AppBarLayout之间的配合效果，但是我们并没有使用AppBarLayout。整个效果涉及到的控件就只有CoordinatorLayout、TextView、及RecyclerView及自定义的Behavior。我们先来分析一下整个例子的设计思路。

### 实现分析

#### 布局与测量的分析

从例子中我们可以看出，在移动前后，RecyclerView始终是在TextView的下方。也就是如下图这种关系：

![布局关系](自定义View事件篇进阶篇(四)-自定义Behavior实战/布局关系.jpg)

那么也就是说，我们需要Behavior在`onLayoutChild`方法中来处理RecyclerView与TextView的位置关系。

但是除了处理RecyclerView的位置关系以外，从例子效果中，我们还可以看出，RecyclerView与TextView之间有着一个联动的关系（这里指的是RecyclerView与TextView之间的位置关系，而不是RecyclerView中的内容)。当TextView逐渐上移的时候，下方的RecyclerView也随着往上移动。也就是我们需要处理根据TextView的位置关系来重新设定RecyclerView在屏幕中的位置。那么我们可以确定的是例子中的RecyclerView是依赖TextView的！！！

>当然控件的位置改变，我们可以使用View.setTransationY或View.offsetTopAndBottom或其他改变位置的方式。本文例子中的代码采用offsetTopAndBottom的方式来处理上下的移动。

那么一提到重新设置位置，那么我们是否需要重新设定`RecyclerView的高度`呢？，答案当然是的！！

从效果上来看，当TextView滑动至隐藏后，RecyclerView在后续的内容滑动中，始终都是填充整个屏幕的。说到这里，不知道大家是否还记得在上篇文章{% post_link 自定义View事件篇进阶篇(三)-CoordinatorLayout与Behavior %}中我们曾提到过，当我们使用Behavior时，如果控件的位置发生了改变，且控件的高度为`match_parent`时，我们需要在CoordinatorLayout测量该控件的高度之前，让控件自主的去测量高度，也就是重写Behavior的`onMeasureChild`方法。来保证控件填充整个屏幕。如果不重写的话。会出现下图所示的空白局域：

![空白区域](自定义View事件篇进阶篇(四)-自定义Behavior实战/空白区域.jpg)

#### 嵌套滑动的分析

在分析了布局与测量后，我们继续分析嵌套滑动效果，整个嵌套滑动效果并不复杂，分为向上、向下两个效果。

- 向上滑动：
当TextView仍在屏幕中显示时，那么RecyclerView就与TextView一起向上滑动。当TextView滑动至屏幕外时，RecyclerView才能处理内部内容的滚动。
- 向下滑动：
当TextView已经被划出屏幕且RecylerView中的内容不能继续向下滑动时，那么就将TextView滑动至显示。否则RecyclerView单独处理内部内容的滚动。

从前期的分析来看，我们需要创建两个Behavior,一个Behavior用于处理TextView嵌套滑动。另一个Behavior来处理RecyclerView的位置关系。那下面就让我们一起来实现这两个Behavior吧。

### 布局代码实现

```java
public class HeaderScrollingViewBehavior extends CoordinatorLayout.Behavior<View> {

    public HeaderScrollingViewBehavior() {
    }

    public HeaderScrollingViewBehavior(Context context, AttributeSet attrs) {
        super(context, attrs);
    }
}
```

```java
@Override
    public boolean onMeasureChild(CoordinatorLayout parent, View child, int parentWidthMeasureSpec, int widthUsed, int parentHeightMeasureSpec, int heightUsed) {

        //获取当前滚动控件的测量模式
        final int childLpHeight = child.getLayoutParams().height;

        //只有当前滚动控件为match_parent/wrap_content时才重新测量其高度，因为固定高度不会出现底部空白的情况
        if (childLpHeight == ViewGroup.LayoutParams.MATCH_PARENT
                || childLpHeight == ViewGroup.LayoutParams.WRAP_CONTENT) {

            //获取当前child依赖的对象集合
            final List<View> dependencies = parent.getDependencies(child);

            final View header = findFirstDependency(dependencies);
            if (header != null) {
                if (ViewCompat.getFitsSystemWindows(header)
                        && !ViewCompat.getFitsSystemWindows(child)) {
                    // If the header is fitting system windows then we need to also,
                    // otherwise we'll get CoL's compatible measuring
                    ViewCompat.setFitsSystemWindows(child, true);

                    if (ViewCompat.getFitsSystemWindows(child)) {
                        // If the set succeeded, trigger a new layout and return true
                        child.requestLayout();
                        return true;
                    }
                }
                //获取当前父控件中可用的距离，
                int availableHeight = View.MeasureSpec.getSize(parentHeightMeasureSpec);
                if (availableHeight == 0) {

                    // If the measure spec doesn't specify a size, use the current height
                    availableHeight = parent.getHeight();
                }
                //计算当前滚动控件的高度。
                final int height = availableHeight - header.getMeasuredHeight() + getScrollRange(header);
                final int heightMeasureSpec = View.MeasureSpec.makeMeasureSpec(height,
                        childLpHeight == ViewGroup.LayoutParams.MATCH_PARENT
                                ? View.MeasureSpec.EXACTLY
                                : View.MeasureSpec.AT_MOST);

                //测量当前滚动的View的正确高度
                parent.onMeasureChild(child, parentWidthMeasureSpec,
                        widthUsed, heightMeasureSpec, heightUsed);

                return true;
            }
        }
        return false;
    }
```

```java
   /**
     * 从依赖集合中获取第一个
     */
    View findFirstDependency(List<View> views) {
        if (views != null && !views.isEmpty()) {
            return views.get(0);
        }
        return null;
    }

    /**
     * 矫正当前Gravity
     */
    private static int resolveGravity(int gravity) {
        return gravity == Gravity.NO_GRAVITY ? GravityCompat.START | Gravity.TOP : gravity;
    }


    /**
     * 获取当前View的滑动范围，一般情况下，为view的高度
     *
     * @param v
     * @return
     */
    int getScrollRange(View v) {
        return v.getMeasuredHeight();
    }
```

### 滑动效果实现

在讲解嵌套滑动效果之前，我们需要回顾一下Behavior实现嵌套滑动的原理与过程，如下图所示：

![Behavior整体流程](自定义View事件篇进阶篇(三)-CoordinatorLayout与Behavior/嵌套滑动整体流程.jpg)

回顾之间讲解的知识，如果我们的控件要通过Behavior实现嵌套滑动的话，那我们需要重写Behavior的相关方法。根据本文例子中展示的效果，我们需要自定义Behavior，并重写其`onStartNestedScroll`与`onNestedPreScroll`和`onNestedPreScroll`三个方法。

>文章中不会介绍Behavior嵌套滑动相关方法的作用，如果需要了解这些方法的作用，建议参看{% post_link 自定义View事件篇进阶篇(一)-NestedScrolling(嵌套滑动)机制 %}文章下的方法介绍。

```java
public class NestedHeaderBehavior extends CoordinatorLayout.Behavior<View> {


    private WeakReference<View> mNestedScrollingChildRef;
    private int mOffset;//记录当前布局的偏移量

    public NestedHeaderBehavior(Context context, AttributeSet attrs) {
        super(context, attrs);
    }

    @Override
    public boolean onLayoutChild(CoordinatorLayout parent, View child, int layoutDirection) {
        mNestedScrollingChildRef = new WeakReference<>(findScrollingChild(parent));
        return super.onLayoutChild(parent, child, layoutDirection);
    }

}
```

```java
    @Override
    public boolean onStartNestedScroll(@NonNull CoordinatorLayout coordinatorLayout, @NonNull View child, @NonNull View directTargetChild, @NonNull View target, int axes, int type) {
        //只要竖直方向上就拦截
        return (axes & ViewCompat.SCROLL_AXIS_VERTICAL) != 0;
    }
```

首先我们判断对应控件只会拦截，竖直方向上的事件。

```java
    @Override
    public void onNestedPreScroll(@NonNull CoordinatorLayout coordinatorLayout, @NonNull View child, @NonNull View target, int dx, int dy, @NonNull int[] consumed, int type) {
        View scrollingChild = mNestedScrollingChildRef.get();
        if (target != scrollingChild) {
            return;
        }
        int currentTop = child.getTop();
        int newTop = currentTop - dy;
        if (dy > 0) {//向上滑动
            //处理在范围内的滚动与fling
            if (newTop >= -child.getHeight()) {
                Log.i(TAG, "onNestedPreScroll:向上移动" + "currentTop--->" + currentTop + " newTop--->" + newTop);
                consumed[1] = dy;
                mOffset = -dy;
                ViewCompat.offsetTopAndBottom(child, -dy);
                coordinatorLayout.dispatchDependentViewsChanged(child);
            } else { //当超过后，单独处理
                consumed[1] = child.getHeight() + currentTop;
                mOffset = -consumed[1];
                ViewCompat.offsetTopAndBottom(child, -consumed[1]);
                coordinatorLayout.dispatchDependentViewsChanged(child);
            }
        }
        if (dy < 0) {//向下滑动
            if (newTop <= 0 && !target.canScrollVertically(-1)) {
                Log.i(TAG, "onNestedPreScroll:向下移动" + "currentTop--->" + currentTop + " newTop--->" + newTop);
                consumed[1] = dy;
                mOffset = -dy;
                ViewCompat.offsetTopAndBottom(child, -dy);
                coordinatorLayout.dispatchDependentViewsChanged(child);
            }
        }

    }

```

### 最后

站在巨人的肩膀上，才能看的更远~
