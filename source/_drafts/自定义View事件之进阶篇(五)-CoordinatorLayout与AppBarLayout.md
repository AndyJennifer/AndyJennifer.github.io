---
title: 自定义View事件篇进阶篇(五)-CoordinatorLayout与AppBarLayout
tags:
  - 嵌套滑动
categories:
  - View事件篇
---

### 前言


```

        @Override
        public boolean onStartNestedScroll(CoordinatorLayout parent, AppBarLayout child,
                View directTargetChild, View target, int nestedScrollAxes, int type) {
            // Return true if we're nested scrolling vertically, and we have scrollable children
            // and the scrolling view is big enough to scroll
            final boolean started = (nestedScrollAxes & ViewCompat.SCROLL_AXIS_VERTICAL) != 0
                    && child.hasScrollableChildren()
                    && parent.getHeight() - directTargetChild.getHeight() <= child.getHeight();

            if (started && mOffsetAnimator != null) {
                // Cancel any offset animation
                mOffsetAnimator.cancel();
            }

            // A new nested scroll has started so clear out the previous ref
            mLastNestedScrollingChildRef = null;

            return started;
        }


```

```
    boolean hasScrollableChildren() {
        return getTotalScrollRange() != 0;
    }
```

```
    public final int getTotalScrollRange() {
        if (mTotalScrollRange != INVALID_SCROLL_RANGE) {
            return mTotalScrollRange;
        }

        int range = 0;
        for (int i = 0, z = getChildCount(); i < z; i++) {
            final View child = getChildAt(i);
            final LayoutParams lp = (LayoutParams) child.getLayoutParams();
            final int childHeight = child.getMeasuredHeight();
            final int flags = lp.mScrollFlags;

            if ((flags & LayoutParams.SCROLL_FLAG_SCROLL) != 0) {
                // We're set to scroll so add the child's height
                //注意这里
                range += childHeight + lp.topMargin + lp.bottomMargin;

                if ((flags & LayoutParams.SCROLL_FLAG_EXIT_UNTIL_COLLAPSED) != 0) {
                    // For a collapsing scroll, we to take the collapsed height into account.
                    // We also break straight away since later views can't scroll beneath
                    // us
                    range -= ViewCompat.getMinimumHeight(child);
                    break;
                }
            } else {
                // As soon as a view doesn't have the scroll flag, we end the range calculation.
                // This is because views below can not scroll under a fixed view.
                break;
            }
        }
        return mTotalScrollRange = Math.max(0, range - getTopInset());
    }
```

其实就是获取的当前控件的总的高度与childHeight + lp.topMargin + lp.bottomMargin;就是为了判断AppBarLayout中有子控件。

```
        @Override
        public void onNestedPreScroll(CoordinatorLayout coordinatorLayout, AppBarLayout child,
                View target, int dx, int dy, int[] consumed, int type) {
            if (dy != 0) {
                int min, max;
                if (dy < 0) {
                    // We're scrolling down
                    min = -child.getTotalScrollRange();
                    max = min + child.getDownNestedPreScrollRange();
                } else {
                    // We're scrolling up
                    min = -child.getUpNestedPreScrollRange();
                    max = 0;
                }
                if (min != max) {
                    consumed[1] = scroll(coordinatorLayout, child, dy, min, max);
                }
            }
        }

```
上述方法只是为了确认AppBarLayout中有控件，有控件这样才滚动。就是为了防止内部控件为0的情况


```

    final int scroll(CoordinatorLayout coordinatorLayout, V header,
            int dy, int minOffset, int maxOffset) {
        return setHeaderTopBottomOffset(coordinatorLayout, header,
                getTopBottomOffsetForScrollingSibling() - dy, minOffset, maxOffset);
    }

```

滚动范围只能在哪个之间，超过了就不滚动了

### 从CoordinatorLayout的拦截方法说起
因为ScrollingViewBehavior中没有拦截事件，又因为CoordinatorLayout中是否拦截事件是根据内部的子控件中的Behavior是否拦截事件决定的，那么事件最终就会走到RecyclerView,又因为RecyclerView内部实现了NestedScrollingChild2接口，那么也就是说CoordinatorLayout会接受到NestedScrollingChild的发送的嵌套滑动事件后会调用onStartNestedScroll方法，在onStartNestedScroll方法中，又会调用所有子控件的Behavior的onStartNestedScroll。如下所示：

```
    @Override
    public boolean onStartNestedScroll(View child, View target, int axes, int type) {
        boolean handled = false;

        final int childCount = getChildCount();
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
也就是说coordinatorLayout把嵌套滑动对应的方法全部传入子控件的Behavior中去了，顺序为NestedScrollingChild->事件->coordinatorLayout->事件->coordinatorLayout中子控件的Behavior。

### 讲讲AppBarLayout的默认Behavior

AppBarLayout并没有显示的在布局中设置Behavior,而是通过使用`@CoordinatorLayout.DefaultBehavior(AppBarLayout.Behavior.class)`注解的方法设置的。那怎么找到这个默认behavior,请查看CoordinatorLayout中getResolvedLayoutParams(View child)方法

CoordinatorLayout中的OnMeasure->prepareChildren()->getResolvedLayoutParams()


### 讲讲AppBarLayout中子view的滑动标志

滑动标志有scroll,snap,enterAlways,enterAlwasCollpased,enterUntilCollpased。当结束嵌套滑动时，会走snapToChildIfNeeded方法。

```

        @Override
        public void onStopNestedScroll(CoordinatorLayout coordinatorLayout, AppBarLayout abl,
                View target, int type) {
            if (type == ViewCompat.TYPE_TOUCH) {
                // If we haven't been flung then let's see if the current view has been set to snap
                snapToChildIfNeeded(coordinatorLayout, abl);
            }

            // Keep a reference to the previous nested scrolling child
            mLastNestedScrollingChildRef = new WeakReference<>(target);
        }
```
snapToChildIfNeeded会根据AppBarLayout中子控件设置的ScrollFlags，来控制AppBarLayout中子控件的滑动
```
        private void snapToChildIfNeeded(CoordinatorLayout coordinatorLayout, AppBarLayout abl) {
            final int offset = getTopBottomOffsetForScrollingSibling();
            final int offsetChildIndex = getChildIndexOnOffset(abl, offset);
            if (offsetChildIndex >= 0) {
                final View offsetChild = abl.getChildAt(offsetChildIndex);
                //注意这里，获取子控件的滑动标志ScrollFlags
                final LayoutParams lp = (LayoutParams) offsetChild.getLayoutParams();
                final int flags = lp.getScrollFlags();
                //
                if ((flags & LayoutParams.FLAG_SNAP) == LayoutParams.FLAG_SNAP) {
                    // We're set the snap, so animate the offset to the nearest edge
                    int snapTop = -offsetChild.getTop();
                    int snapBottom = -offsetChild.getBottom();

                    if (offsetChildIndex == abl.getChildCount() - 1) {
                        // If this is the last child, we need to take the top inset into account
                        snapBottom += abl.getTopInset();
                    }

                    if (checkFlag(flags, LayoutParams.SCROLL_FLAG_EXIT_UNTIL_COLLAPSED)) {
                        // If the view is set only exit until it is collapsed, we'll abide by that
                        snapBottom += ViewCompat.getMinimumHeight(offsetChild);
                    } else if (checkFlag(flags, LayoutParams.FLAG_QUICK_RETURN
                            | LayoutParams.SCROLL_FLAG_ENTER_ALWAYS)) {
                        // If it's set to always enter collapsed, it actually has two states. We
                        // select the state and then snap within the state
                        final int seam = snapBottom + ViewCompat.getMinimumHeight(offsetChild);
                        if (offset < seam) {
                            snapTop = seam;
                        } else {
                            snapBottom = seam;
                        }
                    }

                    final int newOffset = offset < (snapBottom + snapTop) / 2
                            ? snapBottom
                            : snapTop;
                    //然后执行动画
                    animateOffsetTo(coordinatorLayout, abl,
                            MathUtils.clamp(newOffset, -abl.getTotalScrollRange(), 0), 0);
                }
            }
        }
```
### 滑动标志的计算

```
 static final int FLAG_QUICK_RETURN = SCROLL_FLAG_SCROLL | SCROLL_FLAG_ENTER_ALWAYS;
 static final int FLAG_SNAP = SCROLL_FLAG_SCROLL | SCROLL_FLAG_SNAP;
 static final int COLLAPSIBLE_FLAGS = SCROLL_FLAG_EXIT_UNTIL_COLLAPSED
                | SCROLL_FLAG_ENTER_ALWAYS_COLLAPSED;

```
### 动画的执行
动画执行的方法。如下所示：
```
        private void animateOffsetTo(final CoordinatorLayout coordinatorLayout,
                final AppBarLayout child, final int offset, float velocity) {
            final int distance = Math.abs(getTopBottomOffsetForScrollingSibling() - offset);

            final int duration;
            velocity = Math.abs(velocity);
            if (velocity > 0) {
                duration = 3 * Math.round(1000 * (distance / velocity));
            } else {
                final float distanceRatio = (float) distance / child.getHeight();
                duration = (int) ((distanceRatio + 1) * 150);
            }

            animateOffsetWithDuration(coordinatorLayout, child, offset, duration);
        }

        private void animateOffsetWithDuration(final CoordinatorLayout coordinatorLayout,
                final AppBarLayout child, final int offset, final int duration) {
            final int currentOffset = getTopBottomOffsetForScrollingSibling();
            if (currentOffset == offset) {
                if (mOffsetAnimator != null && mOffsetAnimator.isRunning()) {
                    mOffsetAnimator.cancel();
                }
                return;
            }

            if (mOffsetAnimator == null) {
                mOffsetAnimator = new ValueAnimator();
                mOffsetAnimator.setInterpolator(AnimationUtils.DECELERATE_INTERPOLATOR);
                mOffsetAnimator.addUpdateListener(new ValueAnimator.AnimatorUpdateListener() {
                    @Override
                    public void onAnimationUpdate(ValueAnimator animation) {
                        setHeaderTopBottomOffset(coordinatorLayout, child,
                                (int) animation.getAnimatedValue());
                    }
                });
            } else {
                mOffsetAnimator.cancel();
            }

            mOffsetAnimator.setDuration(Math.min(duration, MAX_OFFSET_ANIMATION_DURATION));
            mOffsetAnimator.setIntValues(currentOffset, offset);
            mOffsetAnimator.start();
        }
```


###  RecyclerView没有设置layout_behavior
RecyclerView没有设置layout_behavior时，时覆盖在AppBarlayout上的，但是设置了之后就往下移了？
具体相关类如下：
```
android.support.design.widget.AppBarLayout$ScrollingViewBehavior
```
### 最后
站在巨人的肩膀上，才能看的更远~