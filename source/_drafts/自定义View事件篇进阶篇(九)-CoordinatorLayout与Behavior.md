---
title: 自定义View事件篇进阶篇(九)-CoordinatorLayout与Behavior
tags:
  - 嵌套滑动
categories:
  - View事件篇
---

### 前言

在上篇文章中，我们介绍了NestedScrolling(嵌套滑动)机制，介绍了子控件与父控件嵌套滑动的处理。现在我们来了解谷歌大大为我们提供的另一个交互布局CoordainatorLayout。

通过阅读该文，你能了解如下知识点：

- CoordainatorLayout中Behavior中的基础使用
- CoordainatorLayout中Behavior的注册过程
- CoordainatorLayout与Behavior的测量与布局的关系
- CoordainatorLayout与Behavior嵌套滑动的事件传递关系
- Behavior的单独处理事件的过程

> 该博客中涉及到的示例，在[NestedScrollingDemo](https://github.com/AndyJennifer/NestedScrollingDemo)项目中都有实现，大家可以按需自取。

### CoordainatorLayout简介

在了解CoordainatorLayout之前，我们需要知道它和NestedScrolling机制有什么不同。在NestedScrolling机制中，我们都知道，参与角色只有子控件和父控件。这种关系都是一对一的。而在CoordinatorLayout中子控件除了可以和它的父控件进行交互以外，还可以与其他兄弟控件进行交互。也就是关系可以为一对一或者一对多。

在官方的解释中，CoordainatorLayout是一个加强版的FrameLayout,重要用途有一下两点：

- 作为应用程序的根布局
- 作为一个可以与一个或多个的子控件交互的容器。

需要注意的是CoordainatorLayout,本质是一个ViewGroup，它并没有继承FrameLayout，而是实现了NestedScrollingParent接口。为了达到多个子控件的交互，谷歌在CoordainatorLayout中提供了一个交互的插件--Behavior,通过Behavior,CoordinatorLayout可以定义与它 child 的交互或者是某些 child 之间的交互。那现在我们先了解Behavior这个类中的方法吧。

### Behavior方法介绍

在下图中，我把Behavior中常用的方法梳理了下来如下所示：

| 方法名称 | 作用 |
|------|------------|
| boolean layoutDependsOn  | 0          |  
| boolean onDependentViewChanged  | 100        |
| void onDependentViewRemoved| 1000       |
| boolean onStartNestedScroll  | 0          |  
| void onNestedScrollAccepted  | 100        |
| void onStopNestedScroll| 1000       | 
| void onNestedScroll | 0          |  
| void onNestedPreScroll | 100        | 
| boolean onNestedFling| 1000       | 
| boolean onInterceptTouchEvent | 0          |  
| boolean onTouchEvent | 100        |
| boolean onLayoutChild| 1000       |
| boolean onMeasureChild | 0          |  

### Behavior的依赖关系

如果一个View依赖另外一个View,那么可能需要下面三个方法：

```

public boolean layoutDependsOn(CoordinatorLayout parent, V child, View dependency) { return false; }

public boolean onDependentViewChanged(CoordinatorLayout parent, V child, View dependency) {  return false; }

public void onDependentViewRemoved(CoordinatorLayout parent, V child, View dependency) {}
```

确定一个View依赖另外一个View的时候，是通过`layoutDependsOn`这个方法。其中child是依赖对象，而dependency是被依赖对象，其中该方法的返回值是判断是否依赖对应view。如果返回true。那么表示依赖。反之不依赖。一般情况下，在我们自定义Behavior时，我们需要重写该方法。当`layoutDependsOn`方法返回true时，后面的`onDependentViewChanged`与`onDependentViewRemoved`方法才会调用。

下面我们就看一种简单的例子，来讲解在使用CoordainatorLayout下各个兄弟控件之间的依赖产生的交互效果。


{% 效果展示.gif %}


大致布局如下所示：

```
<?xml version="1.0" encoding="utf-8"?>
<android.support.design.widget.CoordinatorLayout
    xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    android:layout_width="match_parent"
    android:layout_height="match_parent">


    <com.jennifer.andy.nestedscrollingdemo.view.DependedView
        android:layout_width="80dp"
        android:layout_height="40dp"
        android:layout_gravity="center"
        android:background="#f00"
        android:gravity="center"
        android:textColor="#fff"
        android:textSize="18sp"/>

    <TextView
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="跟随兄弟"
        app:layout_behavior=".ui.cdl.behavior.BrotherFollowBehavior"/>

    <TextView
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="变色兄弟"
        app:layout_behavior=".ui.cdl.behavior.BrotherChameleonBehavior"/>

</android.support.design.widget.CoordinatorLayout>
```

在上图中我们创建了一个随手势滑动的`DependedView`。具体代码如下所示：

```
public class DependedView extends View {

    private float mLastX;
    private float mLastY;
    private final int mDragSlop;//最小的滑动距离


    public DependedView(Context context) {
        this(context, null);
    }

    public DependedView(Context context, AttributeSet attrs) {
        this(context, attrs, 0);
    }

    public DependedView(Context context, AttributeSet attrs, int defStyleAttr) {
        super(context, attrs, defStyleAttr);
        mDragSlop = ViewConfiguration.get(context).getScaledTouchSlop();
    }

    @Override
    public boolean onTouchEvent(MotionEvent event) {
        int action = event.getAction();
        switch (action) {
            case MotionEvent.ACTION_DOWN:
                mLastX = event.getX();
                mLastY = event.getY();
                break;

            case MotionEvent.ACTION_MOVE:
                int dx = (int) (event.getX() - mLastX);
                int dy = (int) (event.getY() - mLastY);
                if (Math.abs(dx) > mDragSlop || Math.abs(dy) > mDragSlop) {
                    System.out.println("dx--->" + dx + "dy--->" + dy + "--->" + mDragSlop);
                    ViewCompat.offsetTopAndBottom(this, dy);
                    ViewCompat.offsetLeftAndRight(this, dx);
                }
                mLastX = event.getX();
                mLastY = event.getY();
                break;

            default:
                break;

        }
        return true;
    }
}
```

变色Behavior

```
public class BrotherChameleonBehavior extends CoordinatorLayout.Behavior<View> {

    private ArgbEvaluator mArgbEvaluator = new ArgbEvaluator();

    public BrotherChameleonBehavior(Context context, AttributeSet attrs) {
        super(context, attrs);
    }

    @Override
    public boolean layoutDependsOn(CoordinatorLayout parent, View child, View dependency) {
        return dependency instanceof DependedView;
    }

    @Override
    public boolean onDependentViewChanged(CoordinatorLayout parent, View child, View dependency) {
        int color = (int) mArgbEvaluator.evaluate(dependency.getY() / parent.getHeight(), Color.WHITE, Color.BLACK);
        child.setBackgroundColor(color);
        return false;
    }
}
```

跟随Behavior

```
public class BrotherFollowBehavior extends CoordinatorLayout.Behavior<View> {

    public BrotherFollowBehavior(Context context, AttributeSet attrs) {
        super(context, attrs);
    }

    @Override
    public boolean layoutDependsOn(CoordinatorLayout parent, View child, View dependency) {
        return dependency instanceof DependedView;
    }

    @Override
    public boolean onDependentViewChanged(CoordinatorLayout parent, View child, View dependency) {

        child.setY(dependency.getBottom() + 50);
        child.setX(dependency.getX());
        return true;
    }
}
```

在没有实现Nested相关接口，子控件相互交互的原理，当我们的`DependedView`在屏幕中的位置发生改变的时候，会导致父控重绘，那么也就是会调用onDraw()方法。而CoordainatorLayout在`onAttachedToWindow`有设置了绘制监听`OnPreDrawListener`。如下所示：

```
  @Override
    public void onAttachedToWindow() {
        super.onAttachedToWindow();
        resetTouchBehaviors(false);
        if (mNeedsPreDrawListener) {
            if (mOnPreDrawListener == null) {
                mOnPreDrawListener = new OnPreDrawListener();
            }
            final ViewTreeObserver vto = getViewTreeObserver();
            vto.addOnPreDrawListener(mOnPreDrawListener);
        }
        if (mLastInsets == null && ViewCompat.getFitsSystemWindows(this)) {
            // We're set to fitSystemWindows but we haven't had any insets yet...
            // We should request a new dispatch of window insets
            ViewCompat.requestApplyInsets(this);
        }
        mIsAttachedToWindow = true;
    }
```

跟踪OnPreDrawListener监听：

```
  class OnPreDrawListener implements ViewTreeObserver.OnPreDrawListener {
        @Override
        public boolean onPreDraw() {
            onChildViewsChanged(EVENT_PRE_DRAW);
            return true;
        }
    }

```

我们发现其内部调用了`onChildViewsChanged(EVENT_PRE_DRAW);`方法。我们继续查看该方法。

```
  final void onChildViewsChanged(@DispatchChangeEvent final int type) {
        final int layoutDirection = ViewCompat.getLayoutDirection(this);
        final int childCount = mDependencySortedChildren.size();
        final Rect inset = acquireTempRect();
        final Rect drawRect = acquireTempRect();
        final Rect lastDrawRect = acquireTempRect();

        //获取内部的所有的子控件
        for (int i = 0; i < childCount; i++) {
            final View child = mDependencySortedChildren.get(i);
            final LayoutParams lp = (LayoutParams) child.getLayoutParams();

            //省略部分代码...

            //再次获取内部的所有的子控件
            for (int j = i + 1; j < childCount; j++) {
                final View checkChild = mDependencySortedChildren.get(j);
                final LayoutParams checkLp = (LayoutParams) checkChild.getLayoutParams();
                final Behavior b = checkLp.getBehavior();

                //调用当前子控件的Behavior的layoutDependsOn方法判断是否依赖
                if (b != null && b.layoutDependsOn(this, checkChild, child)) {
                    if (type == EVENT_PRE_DRAW && checkLp.getChangedAfterNestedScroll()) {
                        checkLp.resetChangedAfterNestedScroll();
                        continue;
                    }

                    final boolean handled;
                    switch (type) {
                        case EVENT_VIEW_REMOVED:
                            b.onDependentViewRemoved(this, checkChild, child);
                            handled = true;
                            break;
                        default:
                            // 如果依赖，那么就会走当前子控件的onDependentViewChanged方法。
                            handled = b.onDependentViewChanged(this, checkChild, child);
                            break;
                    }

                }
            }
        }
    //省略部分代码...
    }
```


### Behavior的测量

```
    @Override
    protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
        prepareChildren();
        ensurePreDrawListener();

        final int paddingLeft = getPaddingLeft();
        final int paddingTop = getPaddingTop();
        final int paddingRight = getPaddingRight();
        final int paddingBottom = getPaddingBottom();
        final int layoutDirection = ViewCompat.getLayoutDirection(this);
        final boolean isRtl = layoutDirection == ViewCompat.LAYOUT_DIRECTION_RTL;
        final int widthMode = MeasureSpec.getMode(widthMeasureSpec);
        final int widthSize = MeasureSpec.getSize(widthMeasureSpec);
        final int heightMode = MeasureSpec.getMode(heightMeasureSpec);
        final int heightSize = MeasureSpec.getSize(heightMeasureSpec);

        final int widthPadding = paddingLeft + paddingRight;
        final int heightPadding = paddingTop + paddingBottom;
        int widthUsed = getSuggestedMinimumWidth();
        int heightUsed = getSuggestedMinimumHeight();
        int childState = 0;

        final boolean applyInsets = mLastInsets != null && ViewCompat.getFitsSystemWindows(this);

        final int childCount = mDependencySortedChildren.size();
        for (int i = 0; i < childCount; i++) {
            final View child = mDependencySortedChildren.get(i);
            final LayoutParams lp = (LayoutParams) child.getLayoutParams();
            int childWidthMeasureSpec = widthMeasureSpec;
            int childHeightMeasureSpec = heightMeasureSpec;

            final Behavior b = lp.getBehavior();
            //调用Behavior的测量方法。
            if (b == null || !b.onMeasureChild(this, child, childWidthMeasureSpec, keylineWidthUsed,
                    childHeightMeasureSpec, 0)) {
                onMeasureChild(child, childWidthMeasureSpec, keylineWidthUsed,
                        childHeightMeasureSpec, 0);
            }

            widthUsed = Math.max(widthUsed, widthPadding + child.getMeasuredWidth() +
                    lp.leftMargin + lp.rightMargin);

            heightUsed = Math.max(heightUsed, heightPadding + child.getMeasuredHeight() +
                    lp.topMargin + lp.bottomMargin);
            childState = View.combineMeasuredStates(childState, child.getMeasuredState());
        }

        final int width = View.resolveSizeAndState(widthUsed, widthMeasureSpec,
                childState & View.MEASURED_STATE_MASK);
        final int height = View.resolveSizeAndState(heightUsed, heightMeasureSpec,
                childState << View.MEASURED_HEIGHT_STATE_SHIFT);
        setMeasuredDimension(width, height);
    }
```

上面的代码中，我精简了一些线索无关的代码。我们重点要关注 widthUsed 和 heightUsed 两个变量，它们的作用就是为了保存 CoordinatorLayout 中最大尺寸的子 View 的尺寸。并且，在对子 View 进行遍历的时候，CoordinatorLayout 有主动向子 View 的 Behavior 传递测量的要求，如果 Behavior 自主测量了 child，则以它的结果为准，否则将调用 measureChild() 方法亲自测量。


### Behavior的布局

### Behavior的设置

### Behavior对事件的响应

### 第一种是没有实现Nested相关接口

### 第二种是对实现了Nested接口的响应

自定义 Behavior 的总结
确定 CoordinatorLayout 中 View 与 View 之间的依赖关系，通过 layoutDependsOn() 方法，返回值为 true 则依赖，否则不依赖。
当一个被依赖项 dependency 尺寸或者位置发生变化时，依赖方会通过 Byhavior 获取到，然后在 onDependentViewChanged 中处理。如果在这个方法中 child 尺寸或者位置发生了变化，则需要 return true。
当 Behavior 中的 View 准备响应嵌套滑动时，它不需要通过 layoutDependsOn() 来进行依赖绑定。只需要在 onStartNestedScroll() 方法中通过返回值告知 ViewParent，它是否对嵌套滑动感兴趣。返回值为 true 时，后续的滑动事件才能被响应。
嵌套滑动包括滑动(scroll) 和 快速滑动(fling) 两种情况。开发者根据实际情况运用就好了。
Behavior 通过 3 种方式绑定：1. xml 布局文件。2. 代码设置 layoutparam。3. 自定义 View 的注解。

### CoordainatorLayout的事件处理过程

### onInterceptTouchEvent

```   public boolean onInterceptTouchEvent(MotionEvent ev) {
        final int action = ev.getActionMasked();
        //省略部分代码...
        final boolean intercepted = performIntercept(ev, TYPE_ON_INTERCEPT);
        //省略部分代码...
        return intercepted;
    }

```


```
    private boolean performIntercept(MotionEvent ev, final int type) {
        boolean intercepted = false;
        boolean newBlock = false;

        MotionEvent cancelEvent = null;

        final int action = ev.getActionMasked();

        final List<View> topmostChildList = mTempList1;
        getTopSortedChildren(topmostChildList);

        // Let topmost child views inspect first
        //获取所有子view
        final int childCount = topmostChildList.size();
        for (int i = 0; i < childCount; i++) {
            final View child = topmostChildList.get(i);
            //获取子类的Behavior
            final LayoutParams lp = (LayoutParams) child.getLayoutParams();
            final Behavior b = lp.getBehavior();

            if ((intercepted || newBlock) && action != MotionEvent.ACTION_DOWN) {
                // Cancel all behaviors beneath the one that intercepted.
                // If the event is "down" then we don't have anything to cancel yet.
                if (b != null) {
                    if (cancelEvent == null) {
                        final long now = SystemClock.uptimeMillis();
                        cancelEvent = MotionEvent.obtain(now, now,
                                MotionEvent.ACTION_CANCEL, 0.0f, 0.0f, 0);
                    }
                    switch (type) {
                        case TYPE_ON_INTERCEPT:
                            //调用拦截方法
                            b.onInterceptTouchEvent(this, child, cancelEvent);
                            break;
                        case TYPE_ON_TOUCH:
                            b.onTouchEvent(this, child, cancelEvent);
                            break;
                    }
                }
                continue;
            }

            if (!intercepted && b != null) {
                switch (type) {
                    case TYPE_ON_INTERCEPT:
                       //调用behavior的onInterceptTouchEvent，如果拦截就拦截
                        intercepted = b.onInterceptTouchEvent(this, child, ev);
                        break;
                    case TYPE_ON_TOUCH:
                        intercepted = b.onTouchEvent(this, child, ev);
                        break;
                }
                //注意这里，比较重要找到第一个behavior对象，并赋值
                if (intercepted) {
                    mBehaviorTouchView = child;
                }
            }

            // Don't keep going if we're not allowing interaction below this.
            // Setting newBlock will make sure we cancel the rest of the behaviors.
            final boolean wasBlocking = lp.didBlockInteraction();
            final boolean isBlocking = lp.isBlockingInteractionBelow(this, child);
            newBlock = isBlocking && !wasBlocking;
            if (isBlocking && !newBlock) {
                // Stop here since we don't have anything more to cancel - we already did
                // when the behavior first started blocking things below this point.
                break;
            }
        }

        topmostChildList.clear();

        return intercepted;//是否拦截与CoordinatorLayout中子view的behavior有关
    }
```


如果所有的子控件中的Behavior.onInterceptTouchEvent返回为false,那么CoordinatorLayout就不会拦截事件，事件就传递到了子控件中去了，如果我们的子控件实现是了NestedScrollingChild接口并且在onTouchEvent方法调用了相关API,那么根据嵌套滑动机制，会调用实现了NestedScrollingParent2接口的父控件的相应方法。又因为CoordinatorLayout实现了NestedScrollingParent2接口，那么就又回到了我们最开始的介绍的嵌套滑动机制了。我们查看CoordinatorLayout所有的嵌套滑动方法。

```
    public boolean onStartNestedScroll(View child, View target, int axes, int type) {
        boolean handled = false;

        final int childCount = getChildCount();
        //获取所有的子view的behavior,并调用viewBehavior.onStartNestedScroll
        for (int i = 0; i < childCount; i++) {
            final View view = getChildAt(i);
            if (view.getVisibility() == View.GONE) {
                // If it's GONE, don't dispatch
                continue;
            }
            final LayoutParams lp = (LayoutParams) view.getLayoutParams();
            final Behavior viewBehavior = lp.getBehavior();
            if (viewBehavior != null) {
                final boolean accepted = viewBehavior.onStartNestedScroll(this, view, child,
                        target, axes, type);
                handled |= accepted;
                lp.setNestedScrollAccepted(type, accepted);
            } else {
                lp.setNestedScrollAccepted(type, false);
            }
        }
        return handled;
    }
```

同样在onNestedScrollAccepted方法中，也会调用所有孩子的behavior的onNestedScrollAccepted方法

```
    @Override
    public void onNestedScrollAccepted(View child, View target, int nestedScrollAxes, int type) {
        mNestedScrollingParentHelper.onNestedScrollAccepted(child, target, nestedScrollAxes, type);
        mNestedScrollingTarget = target;

        final int childCount = getChildCount();
        for (int i = 0; i < childCount; i++) {
            final View view = getChildAt(i);
            final LayoutParams lp = (LayoutParams) view.getLayoutParams();
            if (!lp.isNestedScrollAccepted(type)) {
                continue;
            }

            final Behavior viewBehavior = lp.getBehavior();
            if (viewBehavior != null) {
                viewBehavior.onNestedScrollAccepted(this, view, child, target,
                        nestedScrollAxes, type);
            }
        }
    }
```
继续查看onNestedScroll方法，
```
    @Override
    public void onNestedScroll(View target, int dxConsumed, int dyConsumed,
            int dxUnconsumed, int dyUnconsumed) {
        onNestedScroll(target, dxConsumed, dyConsumed, dxUnconsumed, dyUnconsumed,
                ViewCompat.TYPE_TOUCH);
    }

    @Override
    public void onNestedScroll(View target, int dxConsumed, int dyConsumed,
            int dxUnconsumed, int dyUnconsumed, int type) {
        final int childCount = getChildCount();
        boolean accepted = false;

        for (int i = 0; i < childCount; i++) {
            final View view = getChildAt(i);
            if (view.getVisibility() == GONE) {
                // If the child is GONE, skip...
                continue;
            }

            final LayoutParams lp = (LayoutParams) view.getLayoutParams();
            if (!lp.isNestedScrollAccepted(type)) {
                continue;
            }

            final Behavior viewBehavior = lp.getBehavior();
            if (viewBehavior != null) {
                viewBehavior.onNestedScroll(this, view, target, dxConsumed, dyConsumed,
                        dxUnconsumed, dyUnconsumed, type);
                accepted = true;
            }
        }

        if (accepted) {
            onChildViewsChanged(EVENT_NESTED_SCROLL);
        }
    }

```
获取子控件中对应的Behavior，并处理

查看onNestedPreScroll
```
@Override
    public void onNestedPreScroll(View target, int dx, int dy, int[] consumed) {
        onNestedPreScroll(target, dx, dy, consumed, ViewCompat.TYPE_TOUCH);
    }

    @Override
    public void onNestedPreScroll(View target, int dx, int dy, int[] consumed, int  type) {
        int xConsumed = 0;
        int yConsumed = 0;
        boolean accepted = false;

        final int childCount = getChildCount();
        for (int i = 0; i < childCount; i++) {
            final View view = getChildAt(i);
            if (view.getVisibility() == GONE) {
                // If the child is GONE, skip...
                continue;
            }

            final LayoutParams lp = (LayoutParams) view.getLayoutParams();
            if (!lp.isNestedScrollAccepted(type)) {
                continue;
            }

            final Behavior viewBehavior = lp.getBehavior();
            if (viewBehavior != null) {
                mTempIntPair[0] = mTempIntPair[1] = 0;
                viewBehavior.onNestedPreScroll(this, view, target, dx, dy, mTempIntPair, type);

                xConsumed = dx > 0 ? Math.max(xConsumed, mTempIntPair[0])
                        : Math.min(xConsumed, mTempIntPair[0]);
                yConsumed = dy > 0 ? Math.max(yConsumed, mTempIntPair[1])
                        : Math.min(yConsumed, mTempIntPair[1]);

                accepted = true;
            }
        }

        consumed[0] = xConsumed;
        consumed[1] = yConsumed;

        if (accepted) {
            onChildViewsChanged(EVENT_NESTED_SCROLL);
        }
    }
```

这有该方法，同样的也是先获取子控件的Behavior对应的方法，需要主要的是该方法，是获取子控件中`最大`的消耗距离。

```
   final void onChildViewsChanged(@DispatchChangeEvent final int type) {
        final int layoutDirection = ViewCompat.getLayoutDirection(this);
        final int childCount = mDependencySortedChildren.size();
        final Rect inset = acquireTempRect();
        final Rect drawRect = acquireTempRect();
        final Rect lastDrawRect = acquireTempRect();

        for (int i = 0; i < childCount; i++) {
            final View child = mDependencySortedChildren.get(i);
            final LayoutParams lp = (LayoutParams) child.getLayoutParams();
            if (type == EVENT_PRE_DRAW && child.getVisibility() == View.GONE) {
                // Do not try to update GONE child views in pre draw updates.
                continue;
            }

            // Check child views before for anchor
            for (int j = 0; j < i; j++) {
                final View checkChild = mDependencySortedChildren.get(j);

                if (lp.mAnchorDirectChild == checkChild) {
                    offsetChildToAnchor(child, layoutDirection);
                }
            }

            // Get the current draw rect of the view
            getChildRect(child, true, drawRect);

            // Accumulate inset sizes
            if (lp.insetEdge != Gravity.NO_GRAVITY && !drawRect.isEmpty()) {
                final int absInsetEdge = GravityCompat.getAbsoluteGravity(
                        lp.insetEdge, layoutDirection);
                switch (absInsetEdge & Gravity.VERTICAL_GRAVITY_MASK) {
                    case Gravity.TOP:
                        inset.top = Math.max(inset.top, drawRect.bottom);
                        break;
                    case Gravity.BOTTOM:
                        inset.bottom = Math.max(inset.bottom, getHeight() - drawRect.top);
                        break;
                }
                switch (absInsetEdge & Gravity.HORIZONTAL_GRAVITY_MASK) {
                    case Gravity.LEFT:
                        inset.left = Math.max(inset.left, drawRect.right);
                        break;
                    case Gravity.RIGHT:
                        inset.right = Math.max(inset.right, getWidth() - drawRect.left);
                        break;
                }
            }

            // Dodge inset edges if necessary
            if (lp.dodgeInsetEdges != Gravity.NO_GRAVITY && child.getVisibility() == View.VISIBLE) {
                offsetChildByInset(child, inset, layoutDirection);
            }

            if (type != EVENT_VIEW_REMOVED) {
                // Did it change? if not continue
                getLastChildRect(child, lastDrawRect);
                if (lastDrawRect.equals(drawRect)) {
                    continue;
                }
                recordLastChildRect(child, drawRect);
            }

            // Update any behavior-dependent views for the change
            for (int j = i + 1; j < childCount; j++) {
                final View checkChild = mDependencySortedChildren.get(j);
                final LayoutParams checkLp = (LayoutParams) checkChild.getLayoutParams();
                final Behavior b = checkLp.getBehavior();
                //注意看这里，判断是否依赖，如果是依赖，
                if (b != null && b.layoutDependsOn(this, checkChild, child)) {
                    if (type == EVENT_PRE_DRAW && checkLp.getChangedAfterNestedScroll()) {
                        // If this is from a pre-draw and we have already been changed
                        // from a nested scroll, skip the dispatch and reset the flag
                        checkLp.resetChangedAfterNestedScroll();
                        continue;
                    }

                    final boolean handled;
                    //这里判断所依赖的对象是否移除或改变
                    switch (type) {
                        case EVENT_VIEW_REMOVED://移除
                            // EVENT_VIEW_REMOVED means that we need to dispatch
                            // onDependentViewRemoved() instead
                            b.onDependentViewRemoved(this, checkChild, child);
                            handled = true;
                            break;
                        default://改变
                            // Otherwise we dispatch onDependentViewChanged()
                            改变就会调用onDependentViewChanged，故两个子view产生了变化
                            handled = b.onDependentViewChanged(this, checkChild, child);
                            break;
                    }

                    if (type == EVENT_NESTED_SCROLL) {
                        // If this is from a nested scroll, set the flag so that we may skip
                        // any resulting onPreDraw dispatch (if needed)
                        checkLp.setChangedAfterNestedScroll(handled);
                    }
                }
            }
        }

        releaseTempRect(inset);
        releaseTempRect(drawRect);
        releaseTempRect(lastDrawRect);
    }
```

### CoordinatorLayout中Behavior要拦截事件

如果CoordinatorLayout中的子view对应的behavior.onInterceptTouchEvent返回true,那么就会导致CoordinatorLayout拦截事件，那么走自身的onTouchEvent。而该方法也会调用behavior的onTouchEvent方法。一般情况。默认情况下基本都是返回为false,所以我们不用担心，问题走了behavior的onTouchEvent方法，那嵌套机制怎么实现？？？如AppbarLayout的Behavior的父类Behavior，HeaderBehavior中的拦截方法。
```
    @Override
    public boolean onTouchEvent(MotionEvent ev) {
        boolean handled = false;
        boolean cancelSuper = false;
        MotionEvent cancelEvent = null;

        final int action = ev.getActionMasked();
        
         //只要CoordinatorLayout拦截事件，那么mBehaviorTouchView(第一个设置BeHavior的view）就不为空，接下来就会在调用behavior的onTouchEvent方法了
        if (mBehaviorTouchView != null || (cancelSuper = performIntercept(ev, TYPE_ON_TOUCH))) {
            // Safe since performIntercept guarantees that
            // mBehaviorTouchView != null if it returns true
            final LayoutParams lp = (LayoutParams) mBehaviorTouchView.getLayoutParams();
            final Behavior b = lp.getBehavior();
            if (b != null) {
                handled = b.onTouchEvent(this, mBehaviorTouchView, ev);
            }
        }

        // Keep the super implementation correct
        //下面的逻辑，是保证CoordinatorLayout正确的传统事件机制。
        if (mBehaviorTouchView == null) {
            handled |= super.onTouchEvent(ev);
        } else if (cancelSuper) {
            if (cancelEvent == null) {
                final long now = SystemClock.uptimeMillis();
                cancelEvent = MotionEvent.obtain(now, now,
                        MotionEvent.ACTION_CANCEL, 0.0f, 0.0f, 0);
            }
            super.onTouchEvent(cancelEvent);
        }

        if (!handled && action == MotionEvent.ACTION_DOWN) {

        }

        if (cancelEvent != null) {
            cancelEvent.recycle();
        }

        if (action == MotionEvent.ACTION_UP || action == MotionEvent.ACTION_CANCEL) {
            resetTouchBehaviors(false);
        }

        return handled;
    }
```
HeaderBehavior中的拦截方法。
```
    @Override
    public boolean onInterceptTouchEvent(CoordinatorLayout parent, V child, MotionEvent ev) {
        if (mTouchSlop < 0) {
            mTouchSlop = ViewConfiguration.get(parent.getContext()).getScaledTouchSlop();
        }

        final int action = ev.getAction();

        // Shortcut since we're being dragged
        if (action == MotionEvent.ACTION_MOVE && mIsBeingDragged) {
            return true;
        }

        switch (ev.getActionMasked()) {
            case MotionEvent.ACTION_DOWN: {
                mIsBeingDragged = false;
                final int x = (int) ev.getX();
                final int y = (int) ev.getY();
                if (canDragView(child) && parent.isPointInChildBounds(child, x, y)) {
                    mLastMotionY = y;
                    mActivePointerId = ev.getPointerId(0);
                    ensureVelocityTracker();
                }
                break;
            }

            case MotionEvent.ACTION_MOVE: {
                //注意这里 对activePointerId 赋值，通过event来获取手指的id，而是强制设置的
                final int activePointerId = mActivePointerId;
                if (activePointerId == INVALID_POINTER) {
                    // If we don't have a valid id, the touch down wasn't on content.
                    break;
                }
                final int pointerIndex = ev.findPointerIndex(activePointerId);
                if (pointerIndex == -1) {
                    break;
                }

                final int y = (int) ev.getY(pointerIndex);
                final int yDiff = Math.abs(y - mLastMotionY);
                if (yDiff > mTouchSlop) {
                    mIsBeingDragged = true;
                    mLastMotionY = y;
                }
                break;
            }

            case MotionEvent.ACTION_CANCEL:
            case MotionEvent.ACTION_UP: {
                mIsBeingDragged = false;
                mActivePointerId = INVALID_POINTER;
                if (mVelocityTracker != null) {
                    mVelocityTracker.recycle();
                    mVelocityTracker = null;
                }
                break;
            }
        }

        if (mVelocityTracker != null) {
            mVelocityTracker.addMovement(ev);
        }

        return mIsBeingDragged;
    }

```
  又因为 private int mActivePointerId = INVALID_POINTER;所以HeaderBehavior永远都不会拦截事件的。！！！！！！也就是说事件会传递下去，不会被behavior拦截。我顶你个肺。那你写这个干吗啊？？？？

### 最后
https://www.jianshu.com/p/f7989a2a3ec2 防UC
https://www.jianshu.com/p/82d18b0d18f4?utm_campaign=maleskine&utm_content=note&utm_medium=seo_notes&utm_source=recommendation 各种behavior的效果实现
站在巨人的肩膀上，才能看的更远~

### 总结
如果你是从头看到这里，我不知道你有没有这种感觉，像探索一样，经历了很长一段时间，顺着一条条线索，焦急、纠结，最终走出了一条道路。回首溯望，也许会有种风轻云淡的感觉。

这篇文章洋洋洒洒已经有千字以上了，因为篇幅过长，为了防止遗忘。现在可以将文章细节总结如下：

CoordinatorLayout 是一个普通的 ViewGroup，它的布局特性类似于 FrameLayout。
CoordinatorLayout 是超级 FrameLayout，它比 FrameLayout 更强悍的原因是它能与 Behavior 交互。
CoordinatorLayout 与 Behavior 相辅相成，它们一起构建了一个美妙的交互系统。
自定义 Behavior 主要有 2 个目的：1 确定一个 View 依赖另外一个 View 的依赖关系。2 指定一个 View 响应嵌套滑动事件。
确定两个 View 的依赖关系，有两种途径。一个是在 Behavior 中的 layoutDepentOn() 返回 true。另外一种就是直接通过 xml 锚定一个 View。当被依赖方尺寸和位置变化时，Behavior 中的 onDependentViewChanged 方法会被调用。如果在这个方法中改变了主动依赖的那个 view 的尺寸或者位置信息，应该在方法最后 return true。
嵌套滑动分为 nested scroll 和 fling 两种。Behavior 中相应的 View 是否接受响应由 onStartNestedScroll() 返回值决定。一般在 onNestedPreScroll() 处理相应的 nested scroll 响应，在 onPreFling 处理 fling 事件。但是这个不绝对，根据实际情况决定。
NestedScrollView 能够产生嵌套滑动事件是因为它本质上是一个 NestedScrollingChild 对象，而 CoordinatorLayout 能够响应是因为它本质上是一个 NestedScrollingParent 对象。
Behavior 是一种插件机制，如果没有 Behavior 的存在，CoordinatorLayout 和普通的 FrameLayout 无异。Behavior 的存在，可以决定 CoordinatorLayout 中对应的 childview 的测量尺寸、布局位置、触摸响应。