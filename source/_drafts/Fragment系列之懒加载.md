---
title: Fragment系列之懒加载
tags:
  - fragment
categories:
  - fragment
---

### 前言

### FragmentPagerAdapter

在FragmentPagerAdapter中，增加了如下构造函数：

```java
  public FragmentPagerAdapter(@NonNull FragmentManager fm,
            @Behavior int behavior) {
        mFragmentManager = fm;
        mBehavior = behavior;
    }
```

```java
   @Retention(RetentionPolicy.SOURCE)
    @IntDef({BEHAVIOR_SET_USER_VISIBLE_HINT, BEHAVIOR_RESUME_ONLY_CURRENT_FRAGMENT})
    private @interface Behavior { }

     /**
     * Indicates that {@link Fragment#setUserVisibleHint(boolean)} will be called when the current
     * fragment changes.
     *
     * @deprecated This behavior relies on the deprecated
     * {@link Fragment#setUserVisibleHint(boolean)} API. Use
     * {@link #BEHAVIOR_RESUME_ONLY_CURRENT_FRAGMENT} to switch to its replacement,
     * {@link FragmentTransaction#setMaxLifecycle}.
     * @see #FragmentPagerAdapter(FragmentManager, int)
     */
    @Deprecated
    public static final int BEHAVIOR_SET_USER_VISIBLE_HINT = 0;

    /**
     * Indicates that only the current fragment will be in the {@link Lifecycle.State#RESUMED}
     * state. All other Fragments are capped at {@link Lifecycle.State#STARTED}.
     *
     * @see #FragmentPagerAdapter(FragmentManager, int)
     */
    public static final int BEHAVIOR_RESUME_ONLY_CURRENT_FRAGMENT = 1;
```

### ViewPager

在传统的ViewPager中，当ViewPager滑动到指定页时，会调用 `setPrimaryItem` 方法

```java
    public void setPrimaryItem(@NonNull ViewGroup container, int position, @NonNull Object object) {
        Fragment fragment = (Fragment)object;
        if (fragment != mCurrentPrimaryItem) {
            if (mCurrentPrimaryItem != null) {
                mCurrentPrimaryItem.setMenuVisibility(false);
                if (mBehavior == BEHAVIOR_RESUME_ONLY_CURRENT_FRAGMENT) {
                    if (mCurTransaction == null) {
                        mCurTransaction = mFragmentManager.beginTransaction();
                    }
                    mCurTransaction.setMaxLifecycle(mCurrentPrimaryItem, Lifecycle.State.STARTED);
                } else {
                    mCurrentPrimaryItem.setUserVisibleHint(false);
                }
            }
            fragment.setMenuVisibility(true);
            if (mBehavior == BEHAVIOR_RESUME_ONLY_CURRENT_FRAGMENT) {
                if (mCurTransaction == null) {
                    mCurTransaction = mFragmentManager.beginTransaction();
                }
                mCurTransaction.setMaxLifecycle(fragment, Lifecycle.State.RESUMED);
            } else {
                fragment.setUserVisibleHint(true);
            }

            mCurrentPrimaryItem = fragment;
        }
    }
```

也就是说如果当前Fragment 是主 Fragment ,也就会调用

`mCurTransaction.setMaxLifecycle(fragment, Lifecycle.State.RESUMED);`

#### setMaxLifecycle

```java
  /**
     * Set a ceiling for the state of an active fragment in this FragmentManager. If fragment is
     * already above the received state, it will be forced down to the correct state.
     *
     * <p>The fragment provided must currently be added to the FragmentManager to have it's
     * Lifecycle state capped, or previously added as part of this transaction. The
     * {@link Lifecycle.State} passed in must at least be {@link Lifecycle.State#CREATED}, otherwise
     * an {@link IllegalArgumentException} will be thrown.</p>
     *
     * @param fragment the fragment to have it's state capped.
     * @param state the ceiling state for the fragment.
     * @return the same FragmentTransaction instance
     */
    @NonNull
    public FragmentTransaction setMaxLifecycle(@NonNull Fragment fragment,
            @NonNull Lifecycle.State state) {
        addOp(new Op(OP_SET_MAX_LIFECYCLE, fragment, state));
        return this;
    }
```

最终会调用 FragmentManagerImpl中的 `setMaxLifecycle` 方法，设置 fragment.mMaxState中的值。

```java
  public void setMaxLifecycle(Fragment f, Lifecycle.State state) {
        if ((mActive.get(f.mWho) != f
                || (f.mHost != null && f.getFragmentManager() != this))) {
            throw new IllegalArgumentException("Fragment " + f
                    + " is not an active fragment of FragmentManager " + this);
        }
        f.mMaxState = state;
    }
```

翻译：为这个碎片管理器中的活动碎片的状态设置上限。如果fragment已经在接收状态之上，那么它将被强制回到正确的状态。

#### Lifecycle.State

```java
   public enum State {
        /**
         * Destroyed state for a LifecycleOwner. After this event, this Lifecycle will not dispatch
         * any more events. For instance, for an {@link android.app.Activity}, this state is reached
         * <b>right before</b> Activity's {@link android.app.Activity#onDestroy() onDestroy} call.
         */
        DESTROYED,

        /**
         * Initialized state for a LifecycleOwner. For an {@link android.app.Activity}, this is
         * the state when it is constructed but has not received
         * {@link android.app.Activity#onCreate(android.os.Bundle) onCreate} yet.
         */
        INITIALIZED,

        /**
         * Created state for a LifecycleOwner. For an {@link android.app.Activity}, this state
         * is reached in two cases:
         * <ul>
         *     <li>after {@link android.app.Activity#onCreate(android.os.Bundle) onCreate} call;
         *     <li><b>right before</b> {@link android.app.Activity#onStop() onStop} call.
         * </ul>
         */
        CREATED,

        /**
         * Started state for a LifecycleOwner. For an {@link android.app.Activity}, this state
         * is reached in two cases:
         * <ul>
         *     <li>after {@link android.app.Activity#onStart() onStart} call;
         *     <li><b>right before</b> {@link android.app.Activity#onPause() onPause} call.
         * </ul>
         */
        STARTED,

        /**
         * Resumed state for a LifecycleOwner. For an {@link android.app.Activity}, this state
         * is reached after {@link android.app.Activity#onResume() onResume} is called.
         */
        RESUMED;

        /**
         * Compares if this State is greater or equal to the given {@code state}.
         *
         * @param state State to compare with
         * @return true if this State is greater or equal to the given {@code state}
         */
        public boolean isAtLeast(@NonNull State state) {
            return compareTo(state) >= 0;
        }
    }
```

### 流程

在fragmentPagerAdapter中，其实fragment已经被实例化了，但是还没走声明周期。这个时候设置fragment.mMaxState中的值为
`Lifecycle.State.STARTED`

第一步：

```java
    public Object instantiateItem(@NonNull ViewGroup container, int position) {
        if (mCurTransaction == null) {
            mCurTransaction = mFragmentManager.beginTransaction();
        }

        final long itemId = getItemId(position);

        // Do we already have this fragment?
        String name = makeFragmentName(container.getId(), itemId);
        Fragment fragment = mFragmentManager.findFragmentByTag(name);
        if (fragment != null) {
            if (DEBUG) Log.v(TAG, "Attaching item #" + itemId + ": f=" + fragment);
            mCurTransaction.attach(fragment);
        } else {
            fragment = getItem(position);
            if (DEBUG) Log.v(TAG, "Adding item #" + itemId + ": f=" + fragment);
            mCurTransaction.add(container.getId(), fragment,
                    makeFragmentName(container.getId(), itemId));
        }
        if (fragment != mCurrentPrimaryItem) {
            fragment.setMenuVisibility(false);
            if (mBehavior == BEHAVIOR_RESUME_ONLY_CURRENT_FRAGMENT) {
                mCurTransaction.setMaxLifecycle(fragment, Lifecycle.State.STARTED);
            } else {
                fragment.setUserVisibleHint(false);
            }
        }

        return fragment;
    }
```

第二步：当前Fragment为主 fragment 时，继续修改fragment.mMaxState中的值为
`Lifecycle.State.RESUMED`

```java
    public void setPrimaryItem(@NonNull ViewGroup container, int position, @NonNull Object object) {
        Fragment fragment = (Fragment)object;
        if (fragment != mCurrentPrimaryItem) {//如果不是当前主fragment
            if (mCurrentPrimaryItem != null) {
                mCurrentPrimaryItem.setMenuVisibility(false);
                if (mBehavior == BEHAVIOR_RESUME_ONLY_CURRENT_FRAGMENT) {
                    if (mCurTransaction == null) {
                        mCurTransaction = mFragmentManager.beginTransaction();
                    }
                    mCurTransaction.setMaxLifecycle(mCurrentPrimaryItem, Lifecycle.State.STARTED);
                } else {
                    mCurrentPrimaryItem.setUserVisibleHint(false);
                }
            }
            fragment.setMenuVisibility(true);
            if (mBehavior == BEHAVIOR_RESUME_ONLY_CURRENT_FRAGMENT) {//如果是当前主fragment
                if (mCurTransaction == null) {
                    mCurTransaction = mFragmentManager.beginTransaction();
                }
                mCurTransaction.setMaxLifecycle(fragment, Lifecycle.State.RESUMED);
            } else {
                fragment.setUserVisibleHint(true);
            }

            mCurrentPrimaryItem = fragment;
        }
    }
```

第四步：在FragmentManagerImpl类中 调用 moveToState 方法（这个巨长的方法是控制fragment的声明周期的，比对当前fragment的状态与最大的控制）

```java
   void moveToState(Fragment f, int newState, int transit, int transitionStyle,
                     boolean keepActive) {
        // Fragments that are not currently added will sit in the onCreate() state.
        if ((!f.mAdded || f.mDetached) && newState > Fragment.CREATED) {
            newState = Fragment.CREATED;
        }
        if (f.mRemoving && newState > f.mState) {
            if (f.mState == Fragment.INITIALIZING && f.isInBackStack()) {
                // Allow the fragment to be created so that it can be saved later.
                newState = Fragment.CREATED;
            } else {
                // While removing a fragment, we can't change it to a higher state.
                newState = f.mState;
            }
        }
        // Defer start if requested; don't allow it to move to STARTED or higher
        // if it's not already started.
        if (f.mDeferStart && f.mState < Fragment.STARTED && newState > Fragment.ACTIVITY_CREATED) {
            newState = Fragment.ACTIVITY_CREATED;
        }
        // Don't allow the Fragment to go above its max lifecycle state
        // Ensure that Fragments are capped at CREATED instead of ACTIVITY_CREATED.
        if (f.mMaxState == Lifecycle.State.CREATED) {
            newState = Math.min(newState, Fragment.CREATED);
        } else {
            newState = Math.min(newState, f.mMaxState.ordinal());
        }
        if (f.mState <= newState) {
            // For fragments that are created from a layout, when restoring from
            // state we don't want to allow them to be created until they are
            // being reloaded from the layout.
            if (f.mFromLayout && !f.mInLayout) {
                return;
            }
            if (f.getAnimatingAway() != null || f.getAnimator() != null) {
                // The fragment is currently being animated...  but!  Now we
                // want to move our state back up.  Give up on waiting for the
                // animation, move to whatever the final state should be once
                // the animation is done, and then we can proceed from there.
                f.setAnimatingAway(null);
                f.setAnimator(null);
                moveToState(f, f.getStateAfterAnimating(), 0, 0, true);
            }
            //难道当前的状态，与newState状态，如果比他小，就调用相关的声明周期
            switch (f.mState) {
                case Fragment.INITIALIZING:
                    if (newState > Fragment.INITIALIZING) {
                        if (DEBUG) Log.v(TAG, "moveto CREATED: " + f);
                        if (f.mSavedFragmentState != null) {
                            f.mSavedFragmentState.setClassLoader(mHost.getContext()
                                    .getClassLoader());
                            f.mSavedViewState = f.mSavedFragmentState.getSparseParcelableArray(
                                    FragmentManagerImpl.VIEW_STATE_TAG);
                            Fragment target = getFragment(f.mSavedFragmentState,
                                    FragmentManagerImpl.TARGET_STATE_TAG);
                            f.mTargetWho = target != null ? target.mWho : null;
                            if (f.mTargetWho != null) {
                                f.mTargetRequestCode = f.mSavedFragmentState.getInt(
                                        FragmentManagerImpl.TARGET_REQUEST_CODE_STATE_TAG, 0);
                            }
                            if (f.mSavedUserVisibleHint != null) {
                                f.mUserVisibleHint = f.mSavedUserVisibleHint;
                                f.mSavedUserVisibleHint = null;
                            } else {
                                f.mUserVisibleHint = f.mSavedFragmentState.getBoolean(
                                        FragmentManagerImpl.USER_VISIBLE_HINT_TAG, true);
                            }
                            if (!f.mUserVisibleHint) {
                                f.mDeferStart = true;
                                if (newState > Fragment.ACTIVITY_CREATED) {
                                    newState = Fragment.ACTIVITY_CREATED;
                                }
                            }
                        }

                        f.mHost = mHost;
                        f.mParentFragment = mParent;
                        f.mFragmentManager = mParent != null
                                ? mParent.mChildFragmentManager : mHost.mFragmentManager;

                        // If we have a target fragment, push it along to at least CREATED
                        // so that this one can rely on it as an initialized dependency.
                        if (f.mTarget != null) {
                            if (mActive.get(f.mTarget.mWho) != f.mTarget) {
                                throw new IllegalStateException("Fragment " + f
                                        + " declared target fragment " + f.mTarget
                                        + " that does not belong to this FragmentManager!");
                            }
                            if (f.mTarget.mState < Fragment.CREATED) {
                                moveToState(f.mTarget, Fragment.CREATED, 0, 0, true);
                            }
                            f.mTargetWho = f.mTarget.mWho;
                            f.mTarget = null;
                        }
                        if (f.mTargetWho != null) {
                            Fragment target = mActive.get(f.mTargetWho);
                            if (target == null) {
                                throw new IllegalStateException("Fragment " + f
                                        + " declared target fragment " + f.mTargetWho
                                        + " that does not belong to this FragmentManager!");
                            }
                            if (target.mState < Fragment.CREATED) {
                                moveToState(target, Fragment.CREATED, 0, 0, true);
                            }
                        }

                        dispatchOnFragmentPreAttached(f, mHost.getContext(), false);
                        f.performAttach();
                        if (f.mParentFragment == null) {
                            mHost.onAttachFragment(f);
                        } else {
                            f.mParentFragment.onAttachFragment(f);
                        }
                        dispatchOnFragmentAttached(f, mHost.getContext(), false);

                        if (!f.mIsCreated) {
                            dispatchOnFragmentPreCreated(f, f.mSavedFragmentState, false);
                            f.performCreate(f.mSavedFragmentState);
                            dispatchOnFragmentCreated(f, f.mSavedFragmentState, false);
                        } else {
                            f.restoreChildFragmentState(f.mSavedFragmentState);
                            f.mState = Fragment.CREATED;
                        }
                    }
                    // fall through
                case Fragment.CREATED:
                    // We want to unconditionally run this anytime we do a moveToState that
                    // moves the Fragment above INITIALIZING, including cases such as when
                    // we move from CREATED => CREATED as part of the case fall through above.
                    if (newState > Fragment.INITIALIZING) {
                        ensureInflatedFragmentView(f);
                    }

                    if (newState > Fragment.CREATED) {
                        if (DEBUG) Log.v(TAG, "moveto ACTIVITY_CREATED: " + f);
                        if (!f.mFromLayout) {
                            ViewGroup container = null;
                            if (f.mContainerId != 0) {
                                if (f.mContainerId == View.NO_ID) {
                                    throwException(new IllegalArgumentException(
                                            "Cannot create fragment "
                                                    + f
                                                    + " for a container view with no id"));
                                }
                                container = (ViewGroup) mContainer.onFindViewById(f.mContainerId);
                                if (container == null && !f.mRestored) {
                                    String resName;
                                    try {
                                        resName = f.getResources().getResourceName(f.mContainerId);
                                    } catch (Resources.NotFoundException e) {
                                        resName = "unknown";
                                    }
                                    throwException(new IllegalArgumentException(
                                            "No view found for id 0x"
                                                    + Integer.toHexString(f.mContainerId) + " ("
                                                    + resName
                                                    + ") for fragment " + f));
                                }
                            }
                            f.mContainer = container;
                            f.performCreateView(f.performGetLayoutInflater(
                                    f.mSavedFragmentState), container, f.mSavedFragmentState);
                            if (f.mView != null) {
                                f.mInnerView = f.mView;
                                f.mView.setSaveFromParentEnabled(false);
                                if (container != null) {
                                    container.addView(f.mView);
                                }
                                if (f.mHidden) {
                                    f.mView.setVisibility(View.GONE);
                                }
                                f.onViewCreated(f.mView, f.mSavedFragmentState);
                                dispatchOnFragmentViewCreated(f, f.mView, f.mSavedFragmentState,
                                        false);
                                // Only animate the view if it is visible. This is done after
                                // dispatchOnFragmentViewCreated in case visibility is changed
                                f.mIsNewlyAdded = (f.mView.getVisibility() == View.VISIBLE)
                                        && f.mContainer != null;
                            } else {
                                f.mInnerView = null;
                            }
                        }

                        f.performActivityCreated(f.mSavedFragmentState);
                        dispatchOnFragmentActivityCreated(f, f.mSavedFragmentState, false);
                        if (f.mView != null) {
                            f.restoreViewState(f.mSavedFragmentState);
                        }
                        f.mSavedFragmentState = null;
                    }
                    // fall through
                case Fragment.ACTIVITY_CREATED:
                    if (newState > Fragment.ACTIVITY_CREATED) {
                        if (DEBUG) Log.v(TAG, "moveto STARTED: " + f);
                        f.performStart();
                        dispatchOnFragmentStarted(f, false);
                    }
                    // fall through
                case Fragment.STARTED:
                    if (newState > Fragment.STARTED) {
                        if (DEBUG) Log.v(TAG, "moveto RESUMED: " + f);
                        f.performResume();
                        dispatchOnFragmentResumed(f, false);
                        f.mSavedFragmentState = null;
                        f.mSavedViewState = null;
                    }
            }
        } else if (f.mState > newState) {//如果当前声明周期比newState大
            switch (f.mState) {
                case Fragment.RESUMED:
                    if (newState < Fragment.RESUMED) {
                        if (DEBUG) Log.v(TAG, "movefrom RESUMED: " + f);
                        f.performPause();
                        dispatchOnFragmentPaused(f, false);
                    }
                    // fall through
                case Fragment.STARTED:
                    if (newState < Fragment.STARTED) {
                        if (DEBUG) Log.v(TAG, "movefrom STARTED: " + f);
                        f.performStop();
                        dispatchOnFragmentStopped(f, false);
                    }
                    // fall through
                case Fragment.ACTIVITY_CREATED:
                    if (newState < Fragment.ACTIVITY_CREATED) {
                        if (DEBUG) Log.v(TAG, "movefrom ACTIVITY_CREATED: " + f);
                        if (f.mView != null) {
                            // Need to save the current view state if not
                            // done already.
                            if (mHost.onShouldSaveFragmentState(f) && f.mSavedViewState == null) {
                                saveFragmentViewState(f);
                            }
                        }
                        f.performDestroyView();
                        dispatchOnFragmentViewDestroyed(f, false);
                        if (f.mView != null && f.mContainer != null) {
                            // Stop any current animations:
                            f.mContainer.endViewTransition(f.mView);
                            f.mView.clearAnimation();
                            AnimationOrAnimator anim = null;
                            // If parent is being removed, no need to handle child animations.
                            if (f.getParentFragment() == null || !f.getParentFragment().mRemoving) {
                                if (mCurState > Fragment.INITIALIZING && !mDestroyed
                                        && f.mView.getVisibility() == View.VISIBLE
                                        && f.mPostponedAlpha >= 0) {
                                    anim = loadAnimation(f, transit, false,
                                            transitionStyle);
                                }
                                f.mPostponedAlpha = 0;
                                if (anim != null) {
                                    animateRemoveFragment(f, anim, newState);
                                }
                                f.mContainer.removeView(f.mView);
                            }
                        }
                        f.mContainer = null;
                        f.mView = null;
                        // Set here to ensure that Observers are called after
                        // the Fragment's view is set to null
                        f.mViewLifecycleOwner = null;
                        f.mViewLifecycleOwnerLiveData.setValue(null);
                        f.mInnerView = null;
                        f.mInLayout = false;
                    }
                    // fall through
                case Fragment.CREATED:
                    if (newState < Fragment.CREATED) {
                        if (mDestroyed) {
                            // The fragment's containing activity is
                            // being destroyed, but this fragment is
                            // currently animating away.  Stop the
                            // animation right now -- it is not needed,
                            // and we can't wait any more on destroying
                            // the fragment.
                            if (f.getAnimatingAway() != null) {
                                View v = f.getAnimatingAway();
                                f.setAnimatingAway(null);
                                v.clearAnimation();
                            } else if (f.getAnimator() != null) {
                                Animator animator = f.getAnimator();
                                f.setAnimator(null);
                                animator.cancel();
                            }
                        }
                        if (f.getAnimatingAway() != null || f.getAnimator() != null) {
                            // We are waiting for the fragment's view to finish
                            // animating away.  Just make a note of the state
                            // the fragment now should move to once the animation
                            // is done.
                            f.setStateAfterAnimating(newState);
                            newState = Fragment.CREATED;
                        } else {
                            if (DEBUG) Log.v(TAG, "movefrom CREATED: " + f);
                            boolean beingRemoved = f.mRemoving && !f.isInBackStack();
                            if (beingRemoved || mNonConfig.shouldDestroy(f)) {
                                boolean shouldClear;
                                if (mHost instanceof ViewModelStoreOwner) {
                                    shouldClear = mNonConfig.isCleared();
                                } else if (mHost.getContext() instanceof Activity) {
                                    Activity activity = (Activity) mHost.getContext();
                                    shouldClear = !activity.isChangingConfigurations();
                                } else {
                                    shouldClear = true;
                                }
                                if (beingRemoved || shouldClear) {
                                    mNonConfig.clearNonConfigState(f);
                                }
                                f.performDestroy();
                                dispatchOnFragmentDestroyed(f, false);
                            } else {
                                f.mState = Fragment.INITIALIZING;
                            }

                            f.performDetach();
                            dispatchOnFragmentDetached(f, false);
                            if (!keepActive) {
                                if (beingRemoved || mNonConfig.shouldDestroy(f)) {
                                    makeInactive(f);
                                } else {
                                    f.mHost = null;
                                    f.mParentFragment = null;
                                    f.mFragmentManager = null;
                                    if (f.mTargetWho != null) {
                                        Fragment target = mActive.get(f.mTargetWho);
                                        if (target != null && target.getRetainInstance()) {
                                            // Only keep references to other retained Fragments
                                            // to avoid developers accessing Fragments that
                                            // are never coming back
                                            f.mTarget = target;
                                        }
                                    }
                                }
                            }
                        }
                    }
            }
        }

        if (f.mState != newState) {
            Log.w(TAG, "moveToState: Fragment state for " + f + " not updated inline; "
                    + "expected state " + newState + " found " + f.mState);
            f.mState = newState;
        }
    }
```

第五步：fragment 定义的状态 mState 状态

```java

    static final int INITIALIZING = 0;     // Not yet created.
    static final int CREATED = 1;          // Created.
    static final int ACTIVITY_CREATED = 2; // Fully created, not started.
    static final int STARTED = 3;          // Created and started, not resumed.
    static final int RESUMED = 4;  
```

### 最重要的ViewPager合适调用的Fragment的声明周期方法的

在 Fragment 声明周期的时候，我们都知道当我们在使用FragmentPagerAdapter是不会销毁Fragment的，只会调用其 DestoryView 与 onDetach 方法。

```java
    void performDestroyView() {
        mChildFragmentManager.dispatchDestroyView();
        if (mView != null) {
            mViewLifecycleOwner.handleLifecycleEvent(Lifecycle.Event.ON_DESTROY);
        }
        mState = CREATED;
        mCalled = false;
        onDestroyView();
        if (!mCalled) {
            throw new SuperNotCalledException("Fragment " + this
                    + " did not call through to super.onDestroyView()");
        }
        // Handles the detach/reattach case where the view hierarchy
        // is destroyed and recreated and an additional call to
        // onLoadFinished may be needed to ensure the new view
        // hierarchy is populated from data from the Loaders
        LoaderManager.getInstance(this).markForRedelivery();
        mPerformedCreateView = false;
    }
```

### 相邻的Fragment生命周期调用

也就是ViewPager 控制Fragment的显示的时候，因为调用了setPrimaryItem 方法，将其他非主Fragment mMaxState中的值设置为`Lifecycle.State.STARTED`,也就是不会再调用其他Fragment的生命周期，只有主Frament的生命周期会调用

### 超过缓存范围的直接销毁

因为超过缓存范围的fragment会被销毁，会重新走声明周期，所以对懒加载没有影响。

### 最后



站在巨人的肩膀上，才能看的更远~
