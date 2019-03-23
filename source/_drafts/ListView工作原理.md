---
title: ListView工作原理
tags:
- ListView
categories:
- 源码解析
---


### ListView 继承结构

### adapter 


### RecycleBin机制
因为ListView与GridView都需要缓存数据，所以缓存实体类在AbsListView中实现。

```
   class RecycleBin {
        private RecyclerListener mRecyclerListener;

        //第一个存活view的位置
        private int mFirstActivePosition;


        //布局开始时，屏幕中存在的视图（就是是ListView开始时当前ListView中所有可见的View)
        //在布局结束时，会将mActiveViews移动到回收视图中去
        private View[] mActiveViews = new View[0];

        //回收视图
        private ArrayList<View>[] mScrapViews;

        private int mViewTypeCount;

        private ArrayList<View> mCurrentScrap;

        private ArrayList<View> mSkippedScrap;

        private SparseArray<View> mTransientStateViews;
        private LongSparseArray<View> mTransientStateViewsById;

        public void setViewTypeCount(int viewTypeCount) {
            if (viewTypeCount < 1) {
                throw new IllegalArgumentException("Can't have a viewTypeCount < 1");
            }
            //noinspection unchecked
            ArrayList<View>[] scrapViews = new ArrayList[viewTypeCount];
            for (int i = 0; i < viewTypeCount; i++) {
                scrapViews[i] = new ArrayList<View>();
            }
            mViewTypeCount = viewTypeCount;
            mCurrentScrap = scrapViews[0];
            mScrapViews = scrapViews;
        }

        public void markChildrenDirty() {
            if (mViewTypeCount == 1) {
                final ArrayList<View> scrap = mCurrentScrap;
                final int scrapCount = scrap.size();
                for (int i = 0; i < scrapCount; i++) {
                    scrap.get(i).forceLayout();
                }
            } else {
                final int typeCount = mViewTypeCount;
                for (int i = 0; i < typeCount; i++) {
                    final ArrayList<View> scrap = mScrapViews[i];
                    final int scrapCount = scrap.size();
                    for (int j = 0; j < scrapCount; j++) {
                        scrap.get(j).forceLayout();
                    }
                }
            }
            if (mTransientStateViews != null) {
                final int count = mTransientStateViews.size();
                for (int i = 0; i < count; i++) {
                    mTransientStateViews.valueAt(i).forceLayout();
                }
            }
            if (mTransientStateViewsById != null) {
                final int count = mTransientStateViewsById.size();
                for (int i = 0; i < count; i++) {
                    mTransientStateViewsById.valueAt(i).forceLayout();
                }
            }
        }

        public boolean shouldRecycleViewType(int viewType) {
            return viewType >= 0;
        }

        /**
         * Clears the scrap heap.
         */
        void clear() {
            if (mViewTypeCount == 1) {
                final ArrayList<View> scrap = mCurrentScrap;
                clearScrap(scrap);
            } else {
                final int typeCount = mViewTypeCount;
                for (int i = 0; i < typeCount; i++) {
                    final ArrayList<View> scrap = mScrapViews[i];
                    clearScrap(scrap);
                }
            }

            clearTransientStateViews();
        }

        /**
         * Fill ActiveViews with all of the children of the AbsListView.
         *
         * @param childCount The minimum number of views mActiveViews should hold
         * @param firstActivePosition The position of the first view that will be stored in
         *        mActiveViews
         */
        void fillActiveViews(int childCount, int firstActivePosition) {
            if (mActiveViews.length < childCount) {
                mActiveViews = new View[childCount];
            }
            mFirstActivePosition = firstActivePosition;

            //noinspection MismatchedReadAndWriteOfArray
            final View[] activeViews = mActiveViews;
            for (int i = 0; i < childCount; i++) {
                View child = getChildAt(i);
                AbsListView.LayoutParams lp = (AbsListView.LayoutParams) child.getLayoutParams();
                // Don't put header or footer views into the scrap heap
                if (lp != null && lp.viewType != ITEM_VIEW_TYPE_HEADER_OR_FOOTER) {
                    // Note:  We do place AdapterView.ITEM_VIEW_TYPE_IGNORE in active views.
                    //        However, we will NOT place them into scrap views.
                    activeViews[i] = child;
                    // Remember the position so that setupChild() doesn't reset state.
                    lp.scrappedFromPosition = firstActivePosition + i;
                }
            }
        }

        /**
         * Get the view corresponding to the specified position. The view will be removed from
         * mActiveViews if it is found.
         *
         * @param position The position to look up in mActiveViews
         * @return The view if it is found, null otherwise
         */
        View getActiveView(int position) {
            int index = position - mFirstActivePosition;
            final View[] activeViews = mActiveViews;
            if (index >=0 && index < activeViews.length) {
                final View match = activeViews[index];
                activeViews[index] = null;
                return match;
            }
            return null;
        }

        View getTransientStateView(int position) {
            if (mAdapter != null && mAdapterHasStableIds && mTransientStateViewsById != null) {
                long id = mAdapter.getItemId(position);
                View result = mTransientStateViewsById.get(id);
                mTransientStateViewsById.remove(id);
                return result;
            }
            if (mTransientStateViews != null) {
                final int index = mTransientStateViews.indexOfKey(position);
                if (index >= 0) {
                    View result = mTransientStateViews.valueAt(index);
                    mTransientStateViews.removeAt(index);
                    return result;
                }
            }
            return null;
        }

        /**
         * Dumps and fully detaches any currently saved views with transient
         * state.
         */
        void clearTransientStateViews() {
            final SparseArray<View> viewsByPos = mTransientStateViews;
            if (viewsByPos != null) {
                final int N = viewsByPos.size();
                for (int i = 0; i < N; i++) {
                    removeDetachedView(viewsByPos.valueAt(i), false);
                }
                viewsByPos.clear();
            }

            final LongSparseArray<View> viewsById = mTransientStateViewsById;
            if (viewsById != null) {
                final int N = viewsById.size();
                for (int i = 0; i < N; i++) {
                    removeDetachedView(viewsById.valueAt(i), false);
                }
                viewsById.clear();
            }
        }

        /**
         * @return A view from the ScrapViews collection. These are unordered.
         */
        View getScrapView(int position) {
            final int whichScrap = mAdapter.getItemViewType(position);
            if (whichScrap < 0) {
                return null;
            }
            if (mViewTypeCount == 1) {
                return retrieveFromScrap(mCurrentScrap, position);
            } else if (whichScrap < mScrapViews.length) {
                return retrieveFromScrap(mScrapViews[whichScrap], position);
            }
            return null;
        }

        /**
         * Puts a view into the list of scrap views.
         * <p>
         * If the list data hasn't changed or the adapter has stable IDs, views
         * with transient state will be preserved for later retrieval.
         *
         * @param scrap The view to add
         * @param position The view's position within its parent
         */
        void addScrapView(View scrap, int position) {
            final AbsListView.LayoutParams lp = (AbsListView.LayoutParams) scrap.getLayoutParams();
            if (lp == null) {
                // Can't recycle, but we don't know anything about the view.
                // Ignore it completely.
                return;
            }

            lp.scrappedFromPosition = position;

            // Remove but don't scrap header or footer views, or views that
            // should otherwise not be recycled.
            final int viewType = lp.viewType;
            if (!shouldRecycleViewType(viewType)) {
                // Can't recycle. If it's not a header or footer, which have
                // special handling and should be ignored, then skip the scrap
                // heap and we'll fully detach the view later.
                if (viewType != ITEM_VIEW_TYPE_HEADER_OR_FOOTER) {
                    getSkippedScrap().add(scrap);
                }
                return;
            }

            scrap.dispatchStartTemporaryDetach();

            // The the accessibility state of the view may change while temporary
            // detached and we do not allow detached views to fire accessibility
            // events. So we are announcing that the subtree changed giving a chance
            // to clients holding on to a view in this subtree to refresh it.
            notifyViewAccessibilityStateChangedIfNeeded(
                    AccessibilityEvent.CONTENT_CHANGE_TYPE_SUBTREE);

            // Don't scrap views that have transient state.
            final boolean scrapHasTransientState = scrap.hasTransientState();
            if (scrapHasTransientState) {
                if (mAdapter != null && mAdapterHasStableIds) {
                    // If the adapter has stable IDs, we can reuse the view for
                    // the same data.
                    if (mTransientStateViewsById == null) {
                        mTransientStateViewsById = new LongSparseArray<>();
                    }
                    mTransientStateViewsById.put(lp.itemId, scrap);
                } else if (!mDataChanged) {
                    // If the data hasn't changed, we can reuse the views at
                    // their old positions.
                    if (mTransientStateViews == null) {
                        mTransientStateViews = new SparseArray<>();
                    }
                    mTransientStateViews.put(position, scrap);
                } else {
                    // Otherwise, we'll have to remove the view and start over.
                    clearScrapForRebind(scrap);
                    getSkippedScrap().add(scrap);
                }
            } else {
                clearScrapForRebind(scrap);
                if (mViewTypeCount == 1) {
                    mCurrentScrap.add(scrap);
                } else {
                    mScrapViews[viewType].add(scrap);
                }

                if (mRecyclerListener != null) {
                    mRecyclerListener.onMovedToScrapHeap(scrap);
                }
            }
        }

        private ArrayList<View> getSkippedScrap() {
            if (mSkippedScrap == null) {
                mSkippedScrap = new ArrayList<>();
            }
            return mSkippedScrap;
        }

        /**
         * Finish the removal of any views that skipped the scrap heap.
         */
        void removeSkippedScrap() {
            if (mSkippedScrap == null) {
                return;
            }
            final int count = mSkippedScrap.size();
            for (int i = 0; i < count; i++) {
                removeDetachedView(mSkippedScrap.get(i), false);
            }
            mSkippedScrap.clear();
        }

        /**
         * Move all views remaining in mActiveViews to mScrapViews.
         */
        void scrapActiveViews() {
            final View[] activeViews = mActiveViews;
            final boolean hasListener = mRecyclerListener != null;
            final boolean multipleScraps = mViewTypeCount > 1;

            ArrayList<View> scrapViews = mCurrentScrap;
            final int count = activeViews.length;
            for (int i = count - 1; i >= 0; i--) {
                final View victim = activeViews[i];
                if (victim != null) {
                    final AbsListView.LayoutParams lp
                            = (AbsListView.LayoutParams) victim.getLayoutParams();
                    final int whichScrap = lp.viewType;

                    activeViews[i] = null;

                    if (victim.hasTransientState()) {
                        // Store views with transient state for later use.
                        victim.dispatchStartTemporaryDetach();

                        if (mAdapter != null && mAdapterHasStableIds) {
                            if (mTransientStateViewsById == null) {
                                mTransientStateViewsById = new LongSparseArray<View>();
                            }
                            long id = mAdapter.getItemId(mFirstActivePosition + i);
                            mTransientStateViewsById.put(id, victim);
                        } else if (!mDataChanged) {
                            if (mTransientStateViews == null) {
                                mTransientStateViews = new SparseArray<View>();
                            }
                            mTransientStateViews.put(mFirstActivePosition + i, victim);
                        } else if (whichScrap != ITEM_VIEW_TYPE_HEADER_OR_FOOTER) {
                            // The data has changed, we can't keep this view.
                            removeDetachedView(victim, false);
                        }
                    } else if (!shouldRecycleViewType(whichScrap)) {
                        // Discard non-recyclable views except headers/footers.
                        if (whichScrap != ITEM_VIEW_TYPE_HEADER_OR_FOOTER) {
                            removeDetachedView(victim, false);
                        }
                    } else {
                        // Store everything else on the appropriate scrap heap.
                        if (multipleScraps) {
                            scrapViews = mScrapViews[whichScrap];
                        }

                        lp.scrappedFromPosition = mFirstActivePosition + i;
                        removeDetachedView(victim, false);
                        scrapViews.add(victim);

                        if (hasListener) {
                            mRecyclerListener.onMovedToScrapHeap(victim);
                        }
                    }
                }
            }
            pruneScrapViews();
        }

        /**
         * At the end of a layout pass, all temp detached views should either be re-attached or
         * completely detached. This method ensures that any remaining view in the scrap list is
         * fully detached.
         */
        void fullyDetachScrapViews() {
            final int viewTypeCount = mViewTypeCount;
            final ArrayList<View>[] scrapViews = mScrapViews;
            for (int i = 0; i < viewTypeCount; ++i) {
                final ArrayList<View> scrapPile = scrapViews[i];
                for (int j = scrapPile.size() - 1; j >= 0; j--) {
                    final View view = scrapPile.get(j);
                    if (view.isTemporarilyDetached()) {
                        removeDetachedView(view, false);
                    }
                }
            }
        }

        /**
         * Makes sure that the size of mScrapViews does not exceed the size of
         * mActiveViews, which can happen if an adapter does not recycle its
         * views. Removes cached transient state views that no longer have
         * transient state.
         */
        private void pruneScrapViews() {
            final int maxViews = mActiveViews.length;
            final int viewTypeCount = mViewTypeCount;
            final ArrayList<View>[] scrapViews = mScrapViews;
            for (int i = 0; i < viewTypeCount; ++i) {
                final ArrayList<View> scrapPile = scrapViews[i];
                int size = scrapPile.size();
                while (size > maxViews) {
                    scrapPile.remove(--size);
                }
            }

            final SparseArray<View> transViewsByPos = mTransientStateViews;
            if (transViewsByPos != null) {
                for (int i = 0; i < transViewsByPos.size(); i++) {
                    final View v = transViewsByPos.valueAt(i);
                    if (!v.hasTransientState()) {
                        removeDetachedView(v, false);
                        transViewsByPos.removeAt(i);
                        i--;
                    }
                }
            }

            final LongSparseArray<View> transViewsById = mTransientStateViewsById;
            if (transViewsById != null) {
                for (int i = 0; i < transViewsById.size(); i++) {
                    final View v = transViewsById.valueAt(i);
                    if (!v.hasTransientState()) {
                        removeDetachedView(v, false);
                        transViewsById.removeAt(i);
                        i--;
                    }
                }
            }
        }

        /**
         * Puts all views in the scrap heap into the supplied list.
         */
        void reclaimScrapViews(List<View> views) {
            if (mViewTypeCount == 1) {
                views.addAll(mCurrentScrap);
            } else {
                final int viewTypeCount = mViewTypeCount;
                final ArrayList<View>[] scrapViews = mScrapViews;
                for (int i = 0; i < viewTypeCount; ++i) {
                    final ArrayList<View> scrapPile = scrapViews[i];
                    views.addAll(scrapPile);
                }
            }
        }

        /**
         * Updates the cache color hint of all known views.
         *
         * @param color The new cache color hint.
         */
        void setCacheColorHint(int color) {
            if (mViewTypeCount == 1) {
                final ArrayList<View> scrap = mCurrentScrap;
                final int scrapCount = scrap.size();
                for (int i = 0; i < scrapCount; i++) {
                    scrap.get(i).setDrawingCacheBackgroundColor(color);
                }
            } else {
                final int typeCount = mViewTypeCount;
                for (int i = 0; i < typeCount; i++) {
                    final ArrayList<View> scrap = mScrapViews[i];
                    final int scrapCount = scrap.size();
                    for (int j = 0; j < scrapCount; j++) {
                        scrap.get(j).setDrawingCacheBackgroundColor(color);
                    }
                }
            }
            // Just in case this is called during a layout pass
            final View[] activeViews = mActiveViews;
            final int count = activeViews.length;
            for (int i = 0; i < count; ++i) {
                final View victim = activeViews[i];
                if (victim != null) {
                    victim.setDrawingCacheBackgroundColor(color);
                }
            }
        }

        private View retrieveFromScrap(ArrayList<View> scrapViews, int position) {
            final int size = scrapViews.size();
            if (size > 0) {
                // See if we still have a view for this position or ID.
                // Traverse backwards to find the most recently used scrap view
                for (int i = size - 1; i >= 0; i--) {
                    final View view = scrapViews.get(i);
                    final AbsListView.LayoutParams params =
                            (AbsListView.LayoutParams) view.getLayoutParams();

                    if (mAdapterHasStableIds) {
                        final long id = mAdapter.getItemId(position);
                        if (id == params.itemId) {
                            return scrapViews.remove(i);
                        }
                    } else if (params.scrappedFromPosition == position) {
                        final View scrap = scrapViews.remove(i);
                        clearScrapForRebind(scrap);
                        return scrap;
                    }
                }
                final View scrap = scrapViews.remove(size - 1);
                clearScrapForRebind(scrap);
                return scrap;
            } else {
                return null;
            }
        }

        private void clearScrap(final ArrayList<View> scrap) {
            final int scrapCount = scrap.size();
            for (int j = 0; j < scrapCount; j++) {
                removeDetachedView(scrap.remove(scrapCount - 1 - j), false);
            }
        }

        private void clearScrapForRebind(View view) {
            view.clearAccessibilityFocus();
            view.setAccessibilityDelegate(null);
        }

        private void removeDetachedView(View child, boolean animate) {
            child.setAccessibilityDelegate(null);
            AbsListView.this.removeDetachedView(child, animate);
        }
    }

```

### ListView的第一次layout();

#### AbsListView的onLayout
```
  protected void onLayout(boolean changed, int l, int t, int r, int b) {
        super.onLayout(changed, l, t, r, b);

        mInLayout = true;

        final int childCount = getChildCount();
        if (changed) {
            for (int i = 0; i < childCount; i++) {
                getChildAt(i).forceLayout();
            }
            mRecycler.markChildrenDirty();
        }

        layoutChildren();

        mOverscrollMax = (b - t) / OVERSCROLL_LIMIT_DIVISOR;

        // TODO: Move somewhere sane. This doesn't belong in onLayout().
        if (mFastScroll != null) {
            mFastScroll.onItemCountChanged(getChildCount(), mItemCount);
        }
        mInLayout = false;
    }
```

### ListView的layoutChildren方法。

```
  protected void layoutChildren() {
        final boolean blockLayoutRequests = mBlockLayoutRequests;
        if (blockLayoutRequests) {
            return;
        }

        mBlockLayoutRequests = true;

        try {
            super.layoutChildren();

            invalidate();

            if (mAdapter == null) {
                resetList();
                invokeOnItemScrollListener();
                return;
            }

            final int childrenTop = mListPadding.top;
            final int childrenBottom = mBottom - mTop - mListPadding.bottom;

            //在没有设置数据之前，肯定是为0
            final int childCount = getChildCount();

            //省略部分代码...
            boolean dataChanged = mDataChanged;
            if (dataChanged) {
                handleDataChanged();
            }

            //省略部分代码...

            //把所有的child放入RecycleBin中
            //如果可能的话，他们的view将会重用
            final int firstPosition = mFirstPosition;
            final RecycleBin recycleBin = mRecycler;
            if (dataChanged) {
                for (int i = 0; i < childCount; i++) {
                    recycleBin.addScrapView(getChildAt(i), firstPosition+i);
                }
            } else {
                //第一次进入的时候，childCount为0 
                recycleBin.fillActiveViews(childCount, firstPosition);
            }

            //移除当前ListView中的所有View
            detachAllViewsFromParent();
            //删除所有跳过废物堆中的所有视图
            recycleBin.removeSkippedScrap();

            //根据布局模式来控制ListView的现实情况
            switch (mLayoutMode) {
            case LAYOUT_SET_SELECTION:
                if (newSel != null) {
                    sel = fillFromSelection(newSel.getTop(), childrenTop, childrenBottom);
                } else {
                    sel = fillFromMiddle(childrenTop, childrenBottom);
                }
                break;
            case LAYOUT_SYNC:
                sel = fillSpecific(mSyncPosition, mSpecificTop);
                break;
            case LAYOUT_FORCE_BOTTOM:
                sel = fillUp(mItemCount - 1, childrenBottom);
                adjustViewsUpOrDown();
                break;
            case LAYOUT_FORCE_TOP:
                mFirstPosition = 0;
                sel = fillFromTop(childrenTop);
                adjustViewsUpOrDown();
                break;
            case LAYOUT_SPECIFIC:
                final int selectedPosition = reconcileSelectedPosition();
                sel = fillSpecific(selectedPosition, mSpecificTop);
                /**
                 * When ListView is resized, FocusSelector requests an async selection for the
                 * previously focused item to make sure it is still visible. If the item is not
                 * selectable, it won't regain focus so instead we call FocusSelector
                 * to directly request focus on the view after it is visible.
                 */
                if (sel == null && mFocusSelector != null) {
                    final Runnable focusRunnable = mFocusSelector
                            .setupFocusIfValid(selectedPosition);
                    if (focusRunnable != null) {
                        post(focusRunnable);
                    }
                }
                break;
            case LAYOUT_MOVE_SELECTION:
                sel = moveSelection(oldSel, newSel, delta, childrenTop, childrenBottom);
                break;
            default: //默认情况下，是常规布局模式，也就是代码会走到这里。
                if (childCount == 0) {
                    if (!mStackFromBottom) {//默认是ListView是从上往下开始布局
                        final int position = lookForSelectablePosition(0, true);
                        setSelectedPositionInt(position);
                        sel = fillFromTop(childrenTop);
                    } else {
                        final int position = lookForSelectablePosition(mItemCount - 1, false);
                        setSelectedPositionInt(position);
                        sel = fillUp(mItemCount - 1, childrenBottom);
                    }
                } else {
                    if (mSelectedPosition >= 0 && mSelectedPosition < mItemCount) {
                        sel = fillSpecific(mSelectedPosition,
                                oldSel == null ? childrenTop : oldSel.getTop());
                    } else if (mFirstPosition < mItemCount) {
                        sel = fillSpecific(mFirstPosition,
                                oldFirst == null ? childrenTop : oldFirst.getTop());
                    } else {
                        sel = fillSpecific(0, childrenTop);
                    }
                }
                break;
            }

         //省略部分代码...
    }

```

布局模式：
- static final int LAYOUT_NORMAL = 0; 常规布局
- static final int LAYOUT_FORCE_TOP = 1; 显示第一个item
- static final int LAYOUT_SET_SELECTION = 2; 显示选中的item
- static final int LAYOUT_FORCE_BOTTOM = 3; 显示最后一个item
- static final int LAYOUT_SPECIFIC = 4; 使mselecteditem出现在特定位置，并从中构建其余视图。顶部由MSpecificTop指定。
- static final int LAYOUT_SYNC = 5; 由于数据更改而同步的布局。还原msyncPosition使其顶部位于mspecificTop
- static final int LAYOUT_MOVE_SELECTION = 6; 根据定位键显示布局


```
    private View fillFromTop(int nextTop) {
        mFirstPosition = Math.min(mFirstPosition, mSelectedPosition);
        mFirstPosition = Math.min(mFirstPosition, mItemCount - 1);
        if (mFirstPosition < 0) {
            mFirstPosition = 0;
        }
        return fillDown(mFirstPosition, nextTop);
    }

```
该方法依次从上往下填充ListView，实际调用的方法是fillDown()方法。接着继续看

```
    private View fillDown(int pos, int nextTop) {
        View selectedView = null;

        int end = (mBottom - mTop);
        if ((mGroupFlags & CLIP_TO_PADDING_MASK) == CLIP_TO_PADDING_MASK) {
            end -= mListPadding.bottom;
        }

        while (nextTop < end && pos < mItemCount) {
            // is this the selected item?
            boolean selected = pos == mSelectedPosition;
            View child = makeAndAddView(pos, nextTop, true, mListPadding.left, selected);

            nextTop = child.getBottom() + mDividerHeight;
            if (selected) {
                selectedView = child;
            }
            pos++;
        }

        setVisibleRangeHint(mFirstPosition, mFirstPosition + getChildCount() - 1);
        return selectedView;
    }
```
fillDown方法比较好理解，就是一个一个的创建child，判断当前child距离顶部的高度是否超过屏幕显示的高度，如果超过那么就跳出循环。
我们可以发现在该方法中又调用了makeAndAddView()方法。

```
 private View makeAndAddView(int position, int y, boolean flow, int childrenLeft,
            boolean selected) {
        if (!mDataChanged) {
            //从RecyclerBin中获取缓存的视图
            final View activeView = mRecycler.getActiveView(position);
            if (activeView != null) {
                //如果相应位置上的缓存视图不为null，那么就复用该视图
                setupChild(activeView, position, y, flow, childrenLeft, selected, true);
                return activeView;
            }
        }

        // Make a new view for this position, or convert an unused view if
        // possible.
        //创建新的视图，如果当前缓存视图不可用的花
        final View child = obtainView(position, mIsScrap);

        // This needs to be positioned and measured.
        setupChild(child, position, y, flow, childrenLeft, selected, mIsScrap[0]);

        return child;
    }
```
当缓存视图为空时，会调用obtainView，该方法会获取一个视图并让他显示并与指定位置上的数据进行关联。
其中 final boolean[] mIsScrap = new boolean[1];
```
    View obtainView(int position, boolean[] outMetadata) {
        Trace.traceBegin(Trace.TRACE_TAG_VIEW, "obtainView");

        outMetadata[0] = false;

        // 从RecyclerBin中的暂时状态View集合中获取相关View,
        final View transientView = mRecycler.getTransientStateView(position);
        if (transientView != null) {
            final LayoutParams params = (LayoutParams) transientView.getLayoutParams();

            // 如果当前视图没有发生改变，那么重新绑定数据。
            if (params.viewType == mAdapter.getItemViewType(position)) {
                final View updatedView = mAdapter.getView(position, transientView, this);
                // 如果重新绑定数据失败，那么把当前mAdapter.getView（）获取的view添加到垃圾堆中
                if (updatedView != transientView) {
                    setItemViewLayoutParams(updatedView, position);
                    mRecycler.addScrapView(updatedView, position);
                }
            }

            outMetadata[0] = true;

            // Finish the temporary detach started in addScrapView().
            transientView.dispatchFinishTemporaryDetach();
            return transientView;
        }

        //从回收视图中获取相关view,
        final View scrapView = mRecycler.getScrapView(position);
        final View child = mAdapter.getView(position, scrapView, this);
        if (scrapView != null) {
            if (child != scrapView) {
                 // 如果重新绑定数据失败，那么把当前mAdapter.getView（）获取的view添加到垃圾堆中
                mRecycler.addScrapView(scrapView, position);
            } else if (child.isTemporarilyDetached()) {
                outMetadata[0] = true;

                // Finish the temporary detach started in addScrapView().
                child.dispatchFinishTemporaryDetach();
            }
        }

        if (mCacheColorHint != 0) {
            child.setDrawingCacheBackgroundColor(mCacheColorHint);
        }

        if (child.getImportantForAccessibility() == IMPORTANT_FOR_ACCESSIBILITY_AUTO) {
            child.setImportantForAccessibility(IMPORTANT_FOR_ACCESSIBILITY_YES);
        }

        setItemViewLayoutParams(child, position);

        if (AccessibilityManager.getInstance(mContext).isEnabled()) {
            if (mAccessibilityDelegate == null) {
                mAccessibilityDelegate = new ListItemAccessibilityDelegate();
            }
            if (child.getAccessibilityDelegate() == null) {
                child.setAccessibilityDelegate(mAccessibilityDelegate);
            }
        }

        Trace.traceEnd(Trace.TRACE_TAG_VIEW);

        return child;
    }
```

这里我们调用了mAdapter.getView(position, scrapView, this)，那么我们平时中的listView的Adapter时，getView（）方法

```

@Override
public View getView(int position, View convertView, ViewGroup parent) {
	Fruit fruit = getItem(position);
	View view;
	if (convertView == null) {
		view = LayoutInflater.from(getContext()).inflate(resourceId, null);
	} else {
		view = convertView;
	}
	ImageView fruitImage = (ImageView) view.findViewById(R.id.fruit_image);
	TextView fruitName = (TextView) view.findViewById(R.id.fruit_name);
	fruitImage.setImageResource(fruit.getImageId());
	fruitName.setText(fruit.getName());
	return view;
```
也就是说，当我们第一次设置数据时，在当前界面可见的所有child对应的view都是我们通过LayoutInflater.from()出来的。那么当我们获取相应视图后，那么setupChild方法最终，

```
    private void setupChild(View child, int position, int y, boolean flowDown, int childrenLeft,
            boolean selected, boolean isAttachedToWindow) {
        Trace.traceBegin(Trace.TRACE_TAG_VIEW, "setupListItem");

        final boolean isSelected = selected && shouldShowSelector();
        final boolean updateChildSelected = isSelected != child.isSelected();
        final int mode = mTouchMode;
        final boolean isPressed = mode > TOUCH_MODE_DOWN && mode < TOUCH_MODE_SCROLL
                && mMotionPosition == position;
        final boolean updateChildPressed = isPressed != child.isPressed();
        //判断当前child是否需要重绘
        final boolean needToMeasure = !isAttachedToWindow || updateChildSelected
                || child.isLayoutRequested();

        // Respect layout params that are already in the view. Otherwise make
        // some up...
        // 获取当前child的布局参数
        AbsListView.LayoutParams p = (AbsListView.LayoutParams) child.getLayoutParams();
        if (p == null) {
            p = (AbsListView.LayoutParams) generateDefaultLayoutParams();
        }
        p.viewType = mAdapter.getItemViewType(position);
        p.isEnabled = mAdapter.isEnabled(position);

        // Set up view state before attaching the view, since we may need to
        // rely on the jumpDrawablesToCurrentState() call that occurs as part
        // of view attachment.
        if (updateChildSelected) {
            child.setSelected(isSelected);
        }

        if (updateChildPressed) {
            child.setPressed(isPressed);
        }

        if (mChoiceMode != CHOICE_MODE_NONE && mCheckStates != null) {
            if (child instanceof Checkable) {
                ((Checkable) child).setChecked(mCheckStates.get(position));
            } else if (getContext().getApplicationInfo().targetSdkVersion
                    >= android.os.Build.VERSION_CODES.HONEYCOMB) {
                child.setActivated(mCheckStates.get(position));
            }
        }

        if ((isAttachedToWindow && !p.forceAdd) || (p.recycledHeaderFooter
                && p.viewType == AdapterView.ITEM_VIEW_TYPE_HEADER_OR_FOOTER)) {
            attachViewToParent(child, flowDown ? -1 : 0, p);

            // If the view was previously attached for a different position,
            // then manually jump the drawables.
            if (isAttachedToWindow
                    && (((AbsListView.LayoutParams) child.getLayoutParams()).scrappedFromPosition)
                            != position) {
                child.jumpDrawablesToCurrentState();
            }
        } else {
            p.forceAdd = false;
            if (p.viewType == AdapterView.ITEM_VIEW_TYPE_HEADER_OR_FOOTER) {
                p.recycledHeaderFooter = true;
            }
            //注意，这里会将child添加到ListView中
            addViewInLayout(child, flowDown ? -1 : 0, p, true);
            // add view in layout will reset the RTL properties. We have to re-resolve them
            child.resolveRtlPropertiesIfNeeded();
        }

        //如果当前child 需要测量（没有添加到Window上，或其他条件），那么会调用child的measure方法
        if (needToMeasure) {
            final int childWidthSpec = ViewGroup.getChildMeasureSpec(mWidthMeasureSpec,
                    mListPadding.left + mListPadding.right, p.width);
            final int lpHeight = p.height;
            final int childHeightSpec;
            if (lpHeight > 0) {
                childHeightSpec = MeasureSpec.makeMeasureSpec(lpHeight, MeasureSpec.EXACTLY);
            } else {
                childHeightSpec = MeasureSpec.makeSafeMeasureSpec(getMeasuredHeight(),
                        MeasureSpec.UNSPECIFIED);
            }
            child.measure(childWidthSpec, childHeightSpec);
        } else {
            cleanupLayoutState(child);
        }

        final int w = child.getMeasuredWidth();
        final int h = child.getMeasuredHeight();
        final int childTop = flowDown ? y : y - h;
        //如果需要测量，那么也会调用child的layout方法。
        if (needToMeasure) {
            final int childRight = childrenLeft + w;
            final int childBottom = childTop + h;
            child.layout(childrenLeft, childTop, childRight, childBottom);
        } else {
            child.offsetLeftAndRight(childrenLeft - child.getLeft());
            child.offsetTopAndBottom(childTop - child.getTop());
        }

        if (mCachingStarted && !child.isDrawingCacheEnabled()) {
            child.setDrawingCacheEnabled(true);
        }

        Trace.traceEnd(Trace.TRACE_TAG_VIEW);
    }
```

### ListView的第二次layout

```
   public void setAdapter(ListAdapter adapter) {
    
        //省略部分代码...
        requestLayout();
    }
```
我们都知道requestLayout()方法会调用View的onLayout方法，也就是最后还是会走到ListView的layoutChildren方法中去

```
    protected void layoutChildren() {
        final boolean blockLayoutRequests = mBlockLayoutRequests;
        if (blockLayoutRequests) {
            return;
        }

        mBlockLayoutRequests = true;

        try {
            super.layoutChildren();

            invalidate();

            if (mAdapter == null) {
                resetList();
                invokeOnItemScrollListener();
                return;
            }

            final int childrenTop = mListPadding.top;
            final int childrenBottom = mBottom - mTop - mListPadding.bottom;
            final int childCount = getChildCount();
           
            //省略部分代码...
            final int firstPosition = mFirstPosition;
            final RecycleBin recycleBin = mRecycler;
            if (dataChanged) {
                for (int i = 0; i < childCount; i++) {
                    recycleBin.addScrapView(getChildAt(i), firstPosition+i);
                }
            } else {
                recycleBin.fillActiveViews(childCount, firstPosition);
            }

            //清除之前的ListView中添加的数据(主要是清除第一次添加的view,后面就可以直接从回收视图中取了)
            detachAllViewsFromParent();
            recycleBin.removeSkippedScrap();

            switch (mLayoutMode) {
            case LAYOUT_SET_SELECTION:
                if (newSel != null) {
                    sel = fillFromSelection(newSel.getTop(), childrenTop, childrenBottom);
                } else {
                    sel = fillFromMiddle(childrenTop, childrenBottom);
                }
                break;
            case LAYOUT_SYNC:
                sel = fillSpecific(mSyncPosition, mSpecificTop);
                break;
            case LAYOUT_FORCE_BOTTOM:
                sel = fillUp(mItemCount - 1, childrenBottom);
                adjustViewsUpOrDown();
                break;
            case LAYOUT_FORCE_TOP:
                mFirstPosition = 0;
                sel = fillFromTop(childrenTop);
                adjustViewsUpOrDown();
                break;
            case LAYOUT_SPECIFIC:
                final int selectedPosition = reconcileSelectedPosition();
                sel = fillSpecific(selectedPosition, mSpecificTop);
                /**
                 * When ListView is resized, FocusSelector requests an async selection for the
                 * previously focused item to make sure it is still visible. If the item is not
                 * selectable, it won't regain focus so instead we call FocusSelector
                 * to directly request focus on the view after it is visible.
                 */
                if (sel == null && mFocusSelector != null) {
                    final Runnable focusRunnable = mFocusSelector
                            .setupFocusIfValid(selectedPosition);
                    if (focusRunnable != null) {
                        post(focusRunnable);
                    }
                }
                break;
            case LAYOUT_MOVE_SELECTION:
                sel = moveSelection(oldSel, newSel, delta, childrenTop, childrenBottom);
                break;
            default:
                if (childCount == 0) {
                    if (!mStackFromBottom) {
                        final int position = lookForSelectablePosition(0, true);
                        setSelectedPositionInt(position);
                        sel = fillFromTop(childrenTop);
                    } else {
                        final int position = lookForSelectablePosition(mItemCount - 1, false);
                        setSelectedPositionInt(position);
                        sel = fillUp(mItemCount - 1, childrenBottom);
                    }
                } else {
                    //当childCount不为0的时候
                    if (mSelectedPosition >= 0 && mSelectedPosition < mItemCount) {
                        sel = fillSpecific(mSelectedPosition,
                                oldSel == null ? childrenTop : oldSel.getTop());
                    } else if (mFirstPosition < mItemCount) {
                        sel = fillSpecific(mFirstPosition,
                                oldFirst == null ? childrenTop : oldFirst.getTop());
                    } else {
                        sel = fillSpecific(0, childrenTop);
                    }
                }
                break;
            }
            //省略部分代码
        }
```

```
        void fillActiveViews(int childCount, int firstActivePosition) {
            if (mActiveViews.length < childCount) {
                mActiveViews = new View[childCount];
            }
            mFirstActivePosition = firstActivePosition;

            //获取当前可见的item个数，
            final View[] activeViews = mActiveViews;
            for (int i = 0; i < childCount; i++) {
                View child = getChildAt(i);
                AbsListView.LayoutParams lp = (AbsListView.LayoutParams) child.getLayoutParams();
                //不会将header与Footer的视图放入存活视图中去的
                if (lp != null && lp.viewType != ITEM_VIEW_TYPE_HEADER_OR_FOOTER) {
                    activeViews[i] = child;
                    //记录当前在回收视图中的位置，逐次递增
                    lp.scrappedFromPosition = firstActivePosition + i;
                }
            }
        }
```
也就是说当设置adapter的时候，会将ListView中的当前屏幕可见的所有的child视图放入回收视图中。



最后通过，fillSpecific（）方法又会回到makeAndAddView方法，那么现在我们就可以从回收视图中去获取以存在的视图啦

```
 private View makeAndAddView(int position, int y, boolean flow, int childrenLeft,
            boolean selected) {
        if (!mDataChanged) {
            //从RecyclerBin中获取缓存的视图
            final View activeView = mRecycler.getActiveView(position);
            if (activeView != null) {
                //如果相应位置上的缓存视图不为null，那么就复用该视图
                setupChild(activeView, position, y, flow, childrenLeft, selected, true);
                return activeView;
            }
        }

        // Make a new view for this position, or convert an unused view if
        // possible.
        //创建新的视图，如果当前缓存视图不可用的花
        final View child = obtainView(position, mIsScrap);

        // This needs to be positioned and measured.
        setupChild(child, position, y, flow, childrenLeft, selected, mIsScrap[0]);

        return child;
    }
```
### 滑动时缓存视图的控制
我们只知道在ListView添加数据的时候，回收视图时是怎么添加的，但是实际的缓存视图的控制是在滑动的时候。因为缓存ListView与GridView都需要滑动回收视图，所以缓存数据的是在AbsListView中进行操作的。


```
    @Override
    public boolean onTouchEvent(MotionEvent ev) {
        if (!isEnabled()) {
            // A disabled view that is clickable still consumes the touch
            // events, it just doesn't respond to them.
            return isClickable() || isLongClickable();
        }

        if (mPositionScroller != null) {
            mPositionScroller.stop();
        }

        if (mIsDetaching || !isAttachedToWindow()) {
            // Something isn't right.
            // Since we rely on being attached to get data set change notifications,
            // don't risk doing anything where we might try to resync and find things
            // in a bogus state.
            return false;
        }

        startNestedScroll(SCROLL_AXIS_VERTICAL);

        if (mFastScroll != null && mFastScroll.onTouchEvent(ev)) {
            return true;
        }

        initVelocityTrackerIfNotExists();
        final MotionEvent vtev = MotionEvent.obtain(ev);

        final int actionMasked = ev.getActionMasked();
        if (actionMasked == MotionEvent.ACTION_DOWN) {
            mNestedYOffset = 0;
        }
        vtev.offsetLocation(0, mNestedYOffset);
        switch (actionMasked) {
            case MotionEvent.ACTION_DOWN: {
                onTouchDown(ev);
                break;
            }

            case MotionEvent.ACTION_MOVE: {
                onTouchMove(ev, vtev);
                break;
            }

            case MotionEvent.ACTION_UP: {
                onTouchUp(ev);
                break;
            }

            case MotionEvent.ACTION_CANCEL: {
                onTouchCancel();
                break;
            }

            case MotionEvent.ACTION_POINTER_UP: {
                onSecondaryPointerUp(ev);
                final int x = mMotionX;
                final int y = mMotionY;
                final int motionPosition = pointToPosition(x, y);
                if (motionPosition >= 0) {
                    // Remember where the motion event started
                    final View child = getChildAt(motionPosition - mFirstPosition);
                    mMotionViewOriginalTop = child.getTop();
                    mMotionPosition = motionPosition;
                }
                mLastY = y;
                break;
            }

            case MotionEvent.ACTION_POINTER_DOWN: {
                // New pointers take over dragging duties
                final int index = ev.getActionIndex();
                final int id = ev.getPointerId(index);
                final int x = (int) ev.getX(index);
                final int y = (int) ev.getY(index);
                mMotionCorrection = 0;
                mActivePointerId = id;
                mMotionX = x;
                mMotionY = y;
                final int motionPosition = pointToPosition(x, y);
                if (motionPosition >= 0) {
                    // Remember where the motion event started
                    final View child = getChildAt(motionPosition - mFirstPosition);
                    mMotionViewOriginalTop = child.getTop();
                    mMotionPosition = motionPosition;
                }
                mLastY = y;
                break;
            }
        }

        if (mVelocityTracker != null) {
            mVelocityTracker.addMovement(vtev);
        }
        vtev.recycle();
        return true;
    }
```
因为我们只需要考虑ACTION_MOVE事件，所以我们继续看onTouchMove方法
```
    private void onTouchMove(MotionEvent ev, MotionEvent vtev) {
        if (mHasPerformedLongPress) {
            // Consume all move events following a successful long press.
            return;
        }

        int pointerIndex = ev.findPointerIndex(mActivePointerId);
        if (pointerIndex == -1) {
            pointerIndex = 0;
            mActivePointerId = ev.getPointerId(pointerIndex);
        }

        if (mDataChanged) {
            // Re-sync everything if data has been changed
            // since the scroll operation can query the adapter.
            layoutChildren();
        }

        final int y = (int) ev.getY(pointerIndex);

        switch (mTouchMode) {
            case TOUCH_MODE_DOWN:
            case TOUCH_MODE_TAP:
            case TOUCH_MODE_DONE_WAITING:
                // Check if we have moved far enough that it looks more like a
                // scroll than a tap. If so, we'll enter scrolling mode.
                if (startScrollIfNeeded((int) ev.getX(pointerIndex), y, vtev)) {
                    break;
                }
                // Otherwise, check containment within list bounds. If we're
                // outside bounds, cancel any active presses.
                final View motionView = getChildAt(mMotionPosition - mFirstPosition);
                final float x = ev.getX(pointerIndex);
                if (!pointInView(x, y, mTouchSlop)) {
                    setPressed(false);
                    if (motionView != null) {
                        motionView.setPressed(false);
                    }
                    removeCallbacks(mTouchMode == TOUCH_MODE_DOWN ?
                            mPendingCheckForTap : mPendingCheckForLongPress);
                    mTouchMode = TOUCH_MODE_DONE_WAITING;
                    updateSelectorState();
                } else if (motionView != null) {
                    // Still within bounds, update the hotspot.
                    final float[] point = mTmpPoint;
                    point[0] = x;
                    point[1] = y;
                    transformPointToViewLocal(point, motionView);
                    motionView.drawableHotspotChanged(point[0], point[1]);
                }
                break;
            case TOUCH_MODE_SCROLL://滑动的时候
            case TOUCH_MODE_OVERSCROLL:
                scrollIfNeeded((int) ev.getX(pointerIndex), y, vtev);
                break;
        }
    }
```


```
    private void scrollIfNeeded(int x, int y, MotionEvent vtev) {
        int rawDeltaY = y - mMotionY;
        int scrollOffsetCorrection = 0;
        int scrollConsumedCorrection = 0;
        if (mLastY == Integer.MIN_VALUE) {
            rawDeltaY -= mMotionCorrection;
        }
        if (dispatchNestedPreScroll(0, mLastY != Integer.MIN_VALUE ? mLastY - y : -rawDeltaY,
                mScrollConsumed, mScrollOffset)) {
            rawDeltaY += mScrollConsumed[1];
            scrollOffsetCorrection = -mScrollOffset[1];
            scrollConsumedCorrection = mScrollConsumed[1];
            if (vtev != null) {
                vtev.offsetLocation(0, mScrollOffset[1]);
                mNestedYOffset += mScrollOffset[1];
            }
        }
        final int deltaY = rawDeltaY;
        int incrementalDeltaY =
                mLastY != Integer.MIN_VALUE ? y - mLastY + scrollConsumedCorrection : deltaY;
        int lastYCorrection = 0;

        if (mTouchMode == TOUCH_MODE_SCROLL) {
            if (PROFILE_SCROLLING) {
                if (!mScrollProfilingStarted) {
                    Debug.startMethodTracing("AbsListViewScroll");
                    mScrollProfilingStarted = true;
                }
            }

            if (mScrollStrictSpan == null) {
                // If it's non-null, we're already in a scroll.
                mScrollStrictSpan = StrictMode.enterCriticalSpan("AbsListView-scroll");
            }

            if (y != mLastY) {
                // We may be here after stopping a fling and continuing to scroll.
                // If so, we haven't disallowed intercepting touch events yet.
                // Make sure that we do so in case we're in a parent that can intercept.
                if ((mGroupFlags & FLAG_DISALLOW_INTERCEPT) == 0 &&
                        Math.abs(rawDeltaY) > mTouchSlop) {
                    final ViewParent parent = getParent();
                    if (parent != null) {
                        parent.requestDisallowInterceptTouchEvent(true);
                    }
                }

                final int motionIndex;
                if (mMotionPosition >= 0) {
                    motionIndex = mMotionPosition - mFirstPosition;
                } else {
                    // If we don't have a motion position that we can reliably track,
                    // pick something in the middle to make a best guess at things below.
                    motionIndex = getChildCount() / 2;
                }

                int motionViewPrevTop = 0;
                View motionView = this.getChildAt(motionIndex);
                if (motionView != null) {
                    motionViewPrevTop = motionView.getTop();
                }

                // No need to do all this work if we're not going to move anyway
                boolean atEdge = false;
                if (incrementalDeltaY != 0) {
                    //判断是否是边缘，同时添加到回收视图中
                    atEdge = trackMotionScroll(deltaY, incrementalDeltaY);
                }
                //省略部分代码
    }
```

```
   boolean trackMotionScroll(int deltaY, int incrementalDeltaY) {
        final int childCount = getChildCount();
        if (childCount == 0) {
            return true;
        }

        final int firstTop = getChildAt(0).getTop();
        final int lastBottom = getChildAt(childCount - 1).getBottom();

        final Rect listPadding = mListPadding;

        // "effective padding" In this case is the amount of padding that affects
        // how much space should not be filled by items. If we don't clip to padding
        // there is no effective padding.
        int effectivePaddingTop = 0;
        int effectivePaddingBottom = 0;
        if ((mGroupFlags & CLIP_TO_PADDING_MASK) == CLIP_TO_PADDING_MASK) {
            effectivePaddingTop = listPadding.top;
            effectivePaddingBottom = listPadding.bottom;
        }

         // FIXME account for grid vertical spacing too?
        final int spaceAbove = effectivePaddingTop - firstTop;
        final int end = getHeight() - effectivePaddingBottom;
        final int spaceBelow = lastBottom - end;

        final int height = getHeight() - mPaddingBottom - mPaddingTop;
        if (deltaY < 0) {
            deltaY = Math.max(-(height - 1), deltaY);
        } else {
            deltaY = Math.min(height - 1, deltaY);
        }

        if (incrementalDeltaY < 0) {
            incrementalDeltaY = Math.max(-(height - 1), incrementalDeltaY);
        } else {
            incrementalDeltaY = Math.min(height - 1, incrementalDeltaY);
        }

        final int firstPosition = mFirstPosition;

        // Update our guesses for where the first and last views are
        if (firstPosition == 0) {
            mFirstPositionDistanceGuess = firstTop - listPadding.top;
        } else {
            mFirstPositionDistanceGuess += incrementalDeltaY;
        }
        if (firstPosition + childCount == mItemCount) {
            mLastPositionDistanceGuess = lastBottom + listPadding.bottom;
        } else {
            mLastPositionDistanceGuess += incrementalDeltaY;
        }

        final boolean cannotScrollDown = (firstPosition == 0 &&
                firstTop >= listPadding.top && incrementalDeltaY >= 0);
        final boolean cannotScrollUp = (firstPosition + childCount == mItemCount &&
                lastBottom <= getHeight() - listPadding.bottom && incrementalDeltaY <= 0);

        if (cannotScrollDown || cannotScrollUp) {
            return incrementalDeltaY != 0;
        }

        final boolean down = incrementalDeltaY < 0;

        final boolean inTouchMode = isInTouchMode();
        if (inTouchMode) {
            hideSelector();
        }

        final int headerViewsCount = getHeaderViewsCount();
        final int footerViewsStart = mItemCount - getFooterViewsCount();

        int start = 0;
        int count = 0;

        if (down) {//如果是向下滑动
            int top = -incrementalDeltaY;
            if ((mGroupFlags & CLIP_TO_PADDING_MASK) == CLIP_TO_PADDING_MASK) {
                top += listPadding.top;
            }
            for (int i = 0; i < childCount; i++) {
                final View child = getChildAt(i);
                if (child.getBottom() >= top) {
                    break;
                } else {
                    count++;
                    int position = firstPosition + i;
                    if (position >= headerViewsCount && position < footerViewsStart) {
                        //添加到回收视图中，
                        child.clearAccessibilityFocus();
                        mRecycler.addScrapView(child, position);
                    }
                }
            }
        } else {//如果是向上滑动
            int bottom = getHeight() - incrementalDeltaY;
            if ((mGroupFlags & CLIP_TO_PADDING_MASK) == CLIP_TO_PADDING_MASK) {
                bottom -= listPadding.bottom;
            }
            for (int i = childCount - 1; i >= 0; i--) {
                final View child = getChildAt(i);
                if (child.getTop() <= bottom) {
                    break;
                } else {
                    start = i;
                    count++;
                    int position = firstPosition + i;
                    if (position >= headerViewsCount && position < footerViewsStart) {
                        //添加到回收视图中
                        child.clearAccessibilityFocus();
                        mRecycler.addScrapView(child, position);
                    }
                }
            }
        }

        mMotionViewNewTop = mMotionViewOriginalTop + deltaY;

        mBlockLayoutRequests = true;

        if (count > 0) {
            detachViewsFromParent(start, count);
            mRecycler.removeSkippedScrap();
        }

        // invalidate before moving the children to avoid unnecessary invalidate
        // calls to bubble up from the children all the way to the top
        if (!awakenScrollBars()) {
           invalidate();
        }

        offsetChildrenTopAndBottom(incrementalDeltaY);

        if (down) {
            mFirstPosition += count;
        }

        final int absIncrementalDeltaY = Math.abs(incrementalDeltaY);
        if (spaceAbove < absIncrementalDeltaY || spaceBelow < absIncrementalDeltaY) {
            fillGap(down);
        }

        mRecycler.fullyDetachScrapViews();
        if (!inTouchMode && mSelectedPosition != INVALID_POSITION) {
            final int childIndex = mSelectedPosition - mFirstPosition;
            if (childIndex >= 0 && childIndex < getChildCount()) {
                positionSelector(mSelectedPosition, getChildAt(childIndex));
            }
        } else if (mSelectorPosition != INVALID_POSITION) {
            final int childIndex = mSelectorPosition - mFirstPosition;
            if (childIndex >= 0 && childIndex < getChildCount()) {
                positionSelector(INVALID_POSITION, getChildAt(childIndex));
            }
        } else {
            mSelectorRect.setEmpty();
        }

        mBlockLayoutRequests = false;

        invokeOnItemScrollListener();

        return false;
    }
```

### 总结
ListView会先从activeViews获取缓存的view,如果没有获取到则会通过scrapViews中获取数据。