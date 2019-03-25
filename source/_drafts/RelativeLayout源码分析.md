---
title: RelativeLayout源码分析
tags:
- RelativeLayout
categories:
- 源码分析
---
### 前言
RelativeLayout中会有两次测量，因为在RelativeLayout中，View的排列方式是基于彼此的依赖关系。在确定每个子View的位置的时候，就需要先给所有的子View排序一下。又因为RelativeLayout允许A，B 2个子View，横向上B依赖A，纵向上A依赖B。所以需要横向纵向分别进行一次排序测量。

#### 这里解释 

```
        if (mDirtyHierarchy) {
            mDirtyHierarchy = false;
            sortChildren();//这里解释 为什么不走排序
        }
```
```
 public ViewGroup(Context context, AttributeSet attrs, int defStyleAttr, int defStyleRes) {
        super(context, attrs, defStyleAttr, defStyleRes);
        //会调用该方法，
        initViewGroup();
        initFromAttributes(context, attrs, defStyleAttr, defStyleRes);
    }
```

```
  private void initViewGroup() {
        // ViewGroup doesn't draw by default
        if (!debugDraw()) {
            setFlags(WILL_NOT_DRAW, DRAW_MASK);
        }
       //省略其他代码
    }
```

```
  void setFlags(int flags, int mask) {
      //省略其他代码
        if ((changed & DRAW_MASK) != 0) {
            if ((mViewFlags & WILL_NOT_DRAW) != 0) {
                if (mBackground != null
                        || mDefaultFocusHighlight != null
                        || (mForegroundInfo != null && mForegroundInfo.mDrawable != null)) {
                    mPrivateFlags &= ~PFLAG_SKIP_DRAW;
                } else {
                    mPrivateFlags |= PFLAG_SKIP_DRAW;
                }
            } else {
                mPrivateFlags &= ~PFLAG_SKIP_DRAW;
            }
            //重新测量，布局
            requestLayout();
            //重新绘制。
            invalidate(true);
        }
        //省略其他代码
  }
```

```
 @Override
    public void requestLayout() {
        super.requestLayout();
        mDirtyHierarchy = true;
    }
```

#### RelativeLayout测量方法


##### 添加子view到关系图中，并通过在横向与竖直方向上排序
会对其中的子view进行排序
```
 private void sortChildren() {
        final int count = getChildCount();
        if (mSortedVerticalChildren == null || mSortedVerticalChildren.length != count) {
            mSortedVerticalChildren = new View[count];
        }

        if (mSortedHorizontalChildren == null || mSortedHorizontalChildren.length != count) {
            mSortedHorizontalChildren = new View[count];
        }

        final DependencyGraph graph = mGraph;
        graph.clear();
        //添加所有的节点到关系图中
        for (int i = 0; i < count; i++) {
            graph.add(getChildAt(i));
        }
        //对RelativeLayout中的view进行竖直与横向方向上的排序。
        graph.getSortedViews(mSortedVerticalChildren, RULES_VERTICAL);
        graph.getSortedViews(mSortedHorizontalChildren, RULES_HORIZONTAL);
    }
```
其中排序的依据如下
```
    private static final int[] RULES_VERTICAL = {
            ABOVE, BELOW, ALIGN_BASELINE, ALIGN_TOP, ALIGN_BOTTOM
    };

    private static final int[] RULES_HORIZONTAL = {
            LEFT_OF, RIGHT_OF, ALIGN_LEFT, ALIGN_RIGHT, START_OF, END_OF, ALIGN_START, ALIGN_END
    };
```

```
  private static class DependencyGraph {
      //省略部分代码
      void add(View view) {
            final int id = view.getId();
            final Node node = Node.acquire(view);

            if (id != View.NO_ID) {
                mKeyNodes.put(id, node);
            }

            mNodes.add(node);
        }
  }
```
排序的方式
```
        void getSortedViews(View[] sorted, int... rules) {
            final ArrayDeque<Node> roots = findRoots(rules);
            int index = 0;

            Node node;
            while ((node = roots.pollLast()) != null) {
                final View view = node.view;
                final int key = view.getId();

                sorted[index++] = view;

                final ArrayMap<Node, DependencyGraph> dependents = node.dependents;
                final int count = dependents.size();
                for (int i = 0; i < count; i++) {
                    final Node dependent = dependents.keyAt(i);
                    final SparseArray<Node> dependencies = dependent.dependencies;

                    dependencies.remove(key);
                    if (dependencies.size() == 0) {
                        roots.add(dependent);
                    }
                }
            }

            if (index < sorted.length) {
                throw new IllegalStateException("Circular dependencies cannot exist"
                        + " in RelativeLayout");
            }
        }
```
getSortedViews 方法最后会返回一个有序的View列表，排序的规则是View之间的依赖关系。举个简单的例子，如果View A在View C的前面，View B在View A的前面，那么整个依赖关系图为 B->A-C，那么返回的有序的View列表，必然是按照上面的顺序进行排列的。

```
 private ArrayDeque<Node> findRoots(int[] rulesFilter) {
            final SparseArray<Node> keyNodes = mKeyNodes;
            final ArrayList<Node> nodes = mNodes;
            final int count = nodes.size();

        
            //在找到没有依赖关系的节点之前，先清除关系图中的所有节点的依赖关系
            for (int i = 0; i < count; i++) {
                final Node node = nodes.get(i);
                node.dependents.clear();
                node.dependencies.clear();
            }

            // Builds up the dependents and dependencies for each node of the graph
            for (int i = 0; i < count; i++) {
                final Node node = nodes.get(i);

                //获取节点中的布局参数
                final LayoutParams layoutParams = (LayoutParams) node.view.getLayoutParams();
                //获取View中依赖View的id数组
                final int[] rules = layoutParams.mRules;
                final int rulesCount = rulesFilter.length;

                // Look only the the rules passed in parameter, this way we build only the
                // dependencies for a specific set of rules
                for (int j = 0; j < rulesCount; j++) {
                    //获取有依赖关系的ViewId
                    final int rule = rules[rulesFilter[j]];
                    if (rule > 0) {
                        //判断关系图中，是否有该获取依赖的规则,判断是否有依赖关系
                        //这里获取依赖的节点
                        final Node dependency = keyNodes.get(rule);
                        // Skip unknowns and self dependencies
                        if (dependency == null || dependency == node) {
                            continue;
                        }
                        //添加依赖关系
                        // Add the current node as a dependent
                        dependency.dependents.put(node, this);
                        // Add a dependency to the current node
                        node.dependencies.put(rule, dependency);
                    }
                }
            }

            final ArrayDeque<Node> roots = mRoots;
            roots.clear();

            //没有依赖关系的节点会添加到roots集合中去
            for (int i = 0; i < count; i++) {
                final Node node = nodes.get(i);
                if (node.dependencies.size() == 0) roots.addLast(node);
            }

            return roots;
        }
```
获取没有直接依赖其他节的集合roots，同时构造关系图中每个View之间的依赖关系,
也就是说假如依赖关系为 a  b->toleftof c 那么roots为 ac, 排序后数组为 cba

#### 第一次测量
第一次测量会测量水平方向上所有子View的尺寸
```
 protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
     
        //省略其他代码
        View[] views = mSortedHorizontalChildren;
        int count = views.length;

        for (int i = 0; i < count; i++) {
            View child = views[i];
            if (child.getVisibility() != GONE) {
                LayoutParams params = (LayoutParams) child.getLayoutParams();
                int[] rules = params.getRules(layoutDirection);
                //根据依赖View，重新设置当前View的布局参数
                applyHorizontalSizeRules(params, myWidth, rules);
                //测量横向上child
                measureChildHorizontal(child, params, myWidth, myHeight);

                if (positionChildHorizontal(child, params, myWidth, isWrapContentWidth)) {
                    offsetHorizontalAxis = true;
                }
            }
        }
        //省略其他代码
 }
```

会调用 applyHorizontalSizeRules计算
```
 private void applyHorizontalSizeRules(LayoutParams childParams, int myWidth, int[] rules) {
        RelativeLayout.LayoutParams anchorParams;

        // VALUE_NOT_SET indicates a "soft requirement" in that direction. For example:
        // left=10, right=VALUE_NOT_SET means the view must start at 10, but can go as far as it
        // wants to the right
        // left=VALUE_NOT_SET, right=10 means the view must end at 10, but can go as far as it
        // wants to the left
        // left=10, right=20 means the left and right ends are both fixed
        childParams.mLeft = VALUE_NOT_SET;
        childParams.mRight = VALUE_NOT_SET;

        anchorParams = getRelatedViewParams(rules, LEFT_OF);
        if (anchorParams != null) {
            childParams.mRight = anchorParams.mLeft - (anchorParams.leftMargin +
                    childParams.rightMargin);
        } else if (childParams.alignWithParent && rules[LEFT_OF] != 0) {
            if (myWidth >= 0) {
                childParams.mRight = myWidth - mPaddingRight - childParams.rightMargin;
            }
        }

        anchorParams = getRelatedViewParams(rules, RIGHT_OF);
        if (anchorParams != null) {
            childParams.mLeft = anchorParams.mRight + (anchorParams.rightMargin +
                    childParams.leftMargin);
        } else if (childParams.alignWithParent && rules[RIGHT_OF] != 0) {
            childParams.mLeft = mPaddingLeft + childParams.leftMargin;
        }

        anchorParams = getRelatedViewParams(rules, ALIGN_LEFT);
        if (anchorParams != null) {
            childParams.mLeft = anchorParams.mLeft + childParams.leftMargin;
        } else if (childParams.alignWithParent && rules[ALIGN_LEFT] != 0) {
            childParams.mLeft = mPaddingLeft + childParams.leftMargin;
        }

        anchorParams = getRelatedViewParams(rules, ALIGN_RIGHT);
        if (anchorParams != null) {
            childParams.mRight = anchorParams.mRight - childParams.rightMargin;
        } else if (childParams.alignWithParent && rules[ALIGN_RIGHT] != 0) {
            if (myWidth >= 0) {
                childParams.mRight = myWidth - mPaddingRight - childParams.rightMargin;
            }
        }

        if (0 != rules[ALIGN_PARENT_LEFT]) {
            childParams.mLeft = mPaddingLeft + childParams.leftMargin;
        }

        if (0 != rules[ALIGN_PARENT_RIGHT]) {
            if (myWidth >= 0) {
                childParams.mRight = myWidth - mPaddingRight - childParams.rightMargin;
            }
        }
    }
```

#### 第二次测量
```
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
      views = mSortedVerticalChildren;
        count = views.length;
        final int targetSdkVersion = getContext().getApplicationInfo().targetSdkVersion;

        for (int i = 0; i < count; i++) {
            final View child = views[i];
            if (child.getVisibility() != GONE) {
                final LayoutParams params = (LayoutParams) child.getLayoutParams();

                applyVerticalSizeRules(params, myHeight, child.getBaseline());
                measureChild(child, params, myWidth, myHeight);
                if (positionChildVertical(child, params, myHeight, isWrapContentHeight)) {
                    offsetVerticalAxis = true;
                }

                if (isWrapContentWidth) {
                    if (isLayoutRtl()) {
                        if (targetSdkVersion < Build.VERSION_CODES.KITKAT) {
                            width = Math.max(width, myWidth - params.mLeft);
                        } else {
                            width = Math.max(width, myWidth - params.mLeft + params.leftMargin);
                        }
                    } else {
                        if (targetSdkVersion < Build.VERSION_CODES.KITKAT) {
                            width = Math.max(width, params.mRight);
                        } else {
                            width = Math.max(width, params.mRight + params.rightMargin);
                        }
                    }
                }

                if (isWrapContentHeight) {
                    if (targetSdkVersion < Build.VERSION_CODES.KITKAT) {
                        height = Math.max(height, params.mBottom);
                    } else {
                        height = Math.max(height, params.mBottom + params.bottomMargin);
                    }
                }

                if (child != ignore || verticalGravity) {
                    left = Math.min(left, params.mLeft - params.leftMargin);
                    top = Math.min(top, params.mTop - params.topMargin);
                }

                if (child != ignore || horizontalGravity) {
                    right = Math.max(right, params.mRight + params.rightMargin);
                    bottom = Math.max(bottom, params.mBottom + params.bottomMargin);
                }
            }
        }
}

```

#### 修正

##### 对宽度的wrap_content进行修正
```

        if (isWrapContentWidth) {
            // Width already has left padding in it since it was calculated by looking at
            // the right of each child view
            width += mPaddingRight;

            if (mLayoutParams != null && mLayoutParams.width >= 0) {
                width = Math.max(width, mLayoutParams.width);
            }

            width = Math.max(width, getSuggestedMinimumWidth());
            width = resolveSize(width, widthMeasureSpec);

            if (offsetHorizontalAxis) {
                for (int i = 0; i < count; i++) {
                    final View child = views[i];
                    if (child.getVisibility() != GONE) {
                        final LayoutParams params = (LayoutParams) child.getLayoutParams();
                        final int[] rules = params.getRules(layoutDirection);
                        if (rules[CENTER_IN_PARENT] != 0 || rules[CENTER_HORIZONTAL] != 0) {
                            centerHorizontal(child, params, width);
                        } else if (rules[ALIGN_PARENT_RIGHT] != 0) {
                            final int childWidth = child.getMeasuredWidth();
                            params.mLeft = width - mPaddingRight - childWidth;
                            params.mRight = params.mLeft + childWidth;
                        }
                    }
                }
            }
        }
```
##### 对高度的wrap_content修正
```
        if (isWrapContentHeight) {
            // Height already has top padding in it since it was calculated by looking at
            // the bottom of each child view
            height += mPaddingBottom;

            if (mLayoutParams != null && mLayoutParams.height >= 0) {
                height = Math.max(height, mLayoutParams.height);
            }

            height = Math.max(height, getSuggestedMinimumHeight());
            height = resolveSize(height, heightMeasureSpec);

            if (offsetVerticalAxis) {
                for (int i = 0; i < count; i++) {
                    final View child = views[i];
                    if (child.getVisibility() != GONE) {
                        final LayoutParams params = (LayoutParams) child.getLayoutParams();
                        final int[] rules = params.getRules(layoutDirection);
                        if (rules[CENTER_IN_PARENT] != 0 || rules[CENTER_VERTICAL] != 0) {
                            centerVertical(child, params, height);
                        } else if (rules[ALIGN_PARENT_BOTTOM] != 0) {
                            final int childHeight = child.getMeasuredHeight();
                            params.mTop = height - mPaddingBottom - childHeight;
                            params.mBottom = params.mTop + childHeight;
                        }
                    }
                }
            }
        }
```

##### 对水平或竖直方向上的Gravity修正
```
        if (horizontalGravity || verticalGravity) {
            final Rect selfBounds = mSelfBounds;
            selfBounds.set(mPaddingLeft, mPaddingTop, width - mPaddingRight,
                    height - mPaddingBottom);

            final Rect contentBounds = mContentBounds;
            Gravity.apply(mGravity, right - left, bottom - top, selfBounds, contentBounds,
                    layoutDirection);

            final int horizontalOffset = contentBounds.left - left;
            final int verticalOffset = contentBounds.top - top;
            if (horizontalOffset != 0 || verticalOffset != 0) {
                for (int i = 0; i < count; i++) {
                    final View child = views[i];
                    if (child.getVisibility() != GONE && child != ignore) {
                        final LayoutParams params = (LayoutParams) child.getLayoutParams();
                        if (horizontalGravity) {
                            params.mLeft += horizontalOffset;
                            params.mRight += horizontalOffset;
                        }
                        if (verticalGravity) {
                            params.mTop += verticalOffset;
                            params.mBottom += verticalOffset;
                        }
                    }
                }
            }
        }
```

##### 对从右到左的显示进行修正
```
        if (isLayoutRtl()) {
            final int offsetWidth = myWidth - width;
            for (int i = 0; i < count; i++) {
                final View child = views[i];
                if (child.getVisibility() != GONE) {
                    final LayoutParams params = (LayoutParams) child.getLayoutParams();
                    params.mLeft -= offsetWidth;
                    params.mRight -= offsetWidth;
                }
            }
        }

```
### 总结
整理流程为下面几个步骤
