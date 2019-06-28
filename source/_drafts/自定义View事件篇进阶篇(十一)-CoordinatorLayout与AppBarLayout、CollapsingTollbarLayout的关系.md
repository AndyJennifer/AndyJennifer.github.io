---
title: 自定义View事件篇进阶篇(九)-CoordinatorLayout与AppBarLayout、CollapsingTollbarLayout的关系
tags:
  - null
categories:
  - null
---

### 前言

一般我们使用CoordinatorLayout与AppBarLayout、CollapsingTollbarLayout的关系时这样的布局关系。
```
<?xml version="1.0" encoding="utf-8"?>
<android.support.design.widget.CoordinatorLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    tools:context="com.othershe.behaviortest.test1.TestActivity1">

    <android.support.design.widget.AppBarLayout
        android:id="@+id/app_bar"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        app:elevation="0dp">

 <android.support.design.widget.CollapsingToolbarLayout
            android:id="@+id/toolbar_layout"
            android:layout_width="match_parent"
            android:layout_height="match_parent"
            app:contentScrim="#00ffffff"
            app:layout_scrollFlags="scroll|exitUntilCollapsed">

            <ImageView
                android:layout_width="match_parent"
                android:layout_height="200dp"
                android:background="@mipmap/bg"
                android:fitsSystemWindows="true"
                android:scaleType="fitXY"
                app:layout_collapseMode="parallax"
                app:layout_collapseParallaxMultiplier="0.7" />

        </android.support.design.widget.CollapsingToolbarLayout>

    </android.support.design.widget.AppBarLayout>

    <android.support.v7.widget.RecyclerView
        android:id="@+id/my_list"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        app:layout_behavior="@string/appbar_scrolling_view_behavior" />


    
</android.support.design.widget.CoordinatorLayout>
```
### 事件是怎么从AppBarLayout传递到CollapsingToolbarLayout中去的
因为在onAttachedToWindow中设置了AppBarLayout的位置改变监听：

```
    @Override
    protected void onAttachedToWindow() {
        super.onAttachedToWindow();

        // Add an OnOffsetChangedListener if possible
        final ViewParent parent = getParent();
        if (parent instanceof AppBarLayout) {
            // Copy over from the ABL whether we should fit system windows
            ViewCompat.setFitsSystemWindows(this, ViewCompat.getFitsSystemWindows((View) parent));

            if (mOnOffsetChangedListener == null) {
                mOnOffsetChangedListener = new OffsetUpdateListener();
            }
            ((AppBarLayout) parent).addOnOffsetChangedListener(mOnOffsetChangedListener);

            // We're attached, so lets request an inset dispatch
            ViewCompat.requestApplyInsets(this);
        }
    }

```

而该监听调用的方法是AppBarLayout中的

```
    void dispatchOffsetUpdates(int offset) {
        // Iterate backwards through the list so that most recently added listeners
        // get the first chance to decide
        if (mListeners != null) {
            for (int i = 0, z = mListeners.size(); i < z; i++) {
                final OnOffsetChangedListener listener = mListeners.get(i);
                if (listener != null) {
                    listener.onOffsetChanged(this, offset);
                }
            }
        }
    }
```
```
       int setHeaderTopBottomOffset(CoordinatorLayout coordinatorLayout,
                AppBarLayout appBarLayout, int newOffset, int minOffset, int maxOffset) {
            final int curOffset = getTopBottomOffsetForScrollingSibling();
            int consumed = 0;

            if (minOffset != 0 && curOffset >= minOffset && curOffset <= maxOffset) {
                // If we have some scrolling range, and we're currently within the min and max
                // offsets, calculate a new offset
                newOffset = MathUtils.clamp(newOffset, minOffset, maxOffset);
                if (curOffset != newOffset) {
                    final int interpolatedOffset = appBarLayout.hasChildWithInterpolator()
                            ? interpolateOffset(appBarLayout, newOffset)
                            : newOffset;

                    final boolean offsetChanged = setTopAndBottomOffset(interpolatedOffset);

                    // Update how much dy we have consumed
                    consumed = curOffset - newOffset;
                    // Update the stored sibling offset
                    mOffsetDelta = newOffset - interpolatedOffset;

                    if (!offsetChanged && appBarLayout.hasChildWithInterpolator()) {
                        // If the offset hasn't changed and we're using an interpolated scroll
                        // then we need to keep any dependent views updated. CoL will do this for
                        // us when we move, but we need to do it manually when we don't (as an
                        // interpolated scroll may finish early).
                        coordinatorLayout.dispatchDependentViewsChanged(appBarLayout);
                    }

                    // Dispatch the updates to any listeners

                    //注意这里
                    appBarLayout.dispatchOffsetUpdates(getTopAndBottomOffset());

                    // Update the AppBarLayout's drawable state (for any elevation changes)
                    updateAppBarLayoutDrawableState(coordinatorLayout, appBarLayout, newOffset,
                            newOffset < curOffset ? -1 : 1, false);
                }
            } else {
                // Reset the offset delta
                mOffsetDelta = 0;
            }

            return consumed;
        }
```
又走
```
int setHeaderTopBottomOffset(CoordinatorLayout parent, V header, int newOffset) {
        return setHeaderTopBottomOffset(parent, header, newOffset,
                Integer.MIN_VALUE, Integer.MAX_VALUE);
    }

```

```
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
                        //这里调用
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

也就是AppBarLayout的snapToChildIfNeeded方法有关系

### 最后
站在巨人的肩膀上，才能看的更远~