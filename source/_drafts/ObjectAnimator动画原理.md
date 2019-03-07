---
title: ObjectAnimator动画原理
tags:
- 动画
categories:
- 源码分析
---

### ObjectAnimator动画原理
总体来说ObjectAnimator，ValueAnimator最后只是调用了Draw()方法重新绘制了view,并没有改变View在父容器中的相对位置，那为什么ObjectAnimator能响应事件呢，因为在View中有一个变化的矩阵，记录了ObjectAnimator执行的一系列变换操作。ViewGroup在处理事件后的时候，判断View内部的变化矩阵算出转换后的坐标点，最后判断
```
@Override
    public void start() {
        AnimationHandler.getInstance().autoCancelBasedOn(this);
        if (DBG) {
            Log.d(LOG_TAG, "Anim target, duration: " + getTarget() + ", " + getDuration());
            for (int i = 0; i < mValues.length; ++i) {
                PropertyValuesHolder pvh = mValues[i];
                Log.d(LOG_TAG, "   Values[" + i + "]: " +
                    pvh.getPropertyName() + ", " + pvh.mKeyframes.getValue(0) + ", " +
                    pvh.mKeyframes.getValue(1));
            }
        }
        super.start();
    }
```

####  调用ValueAnimator的start方法

```
   private void start(boolean playBackwards) {
        if (Looper.myLooper() == null) {
            throw new AndroidRuntimeException("Animators may only be run on Looper threads");
        }
        mReversing = playBackwards;
        mSelfPulse = !mSuppressSelfPulseRequested;
        // Special case: reversing from seek-to-0 should act as if not seeked at all.
        if (playBackwards && mSeekFraction != -1 && mSeekFraction != 0) {
            if (mRepeatCount == INFINITE) {
                // Calculate the fraction of the current iteration.
                float fraction = (float) (mSeekFraction - Math.floor(mSeekFraction));
                mSeekFraction = 1 - fraction;
            } else {
                mSeekFraction = 1 + mRepeatCount - mSeekFraction;
            }
        }
        mStarted = true;
        mPaused = false;
        mRunning = false;
        mAnimationEndRequested = false;
        // Resets mLastFrameTime when start() is called, so that if the animation was running,
        // calling start() would put the animation in the
        // started-but-not-yet-reached-the-first-frame phase.
        mLastFrameTime = -1;
        mFirstFrameTime = -1;
        mStartTime = -1;
        addAnimationCallback(0);

        if (mStartDelay == 0 || mSeekFraction >= 0 || mReversing) {
            // If there's no start delay, init the animation and notify start listeners right away
            // to be consistent with the previous behavior. Otherwise, postpone this until the first
            // frame after the start delay.
            startAnimation();
            if (mSeekFraction == -1) {
                // No seek, start at play time 0. Note that the reason we are not using fraction 0
                // is because for animations with 0 duration, we want to be consistent with pre-N
                // behavior: skip to the final value immediately.
                setCurrentPlayTime(0);
            } else {
                setCurrentFraction(mSeekFraction);
            }
        }
    }

```

```
private void addAnimationCallback(long delay) {
        if (!mSelfPulse) {
            return;
        }
        getAnimationHandler().addAnimationFrameCallback(this, delay);
    }
```

####  AnimationHandler内部

```
 public void addAnimationFrameCallback(final AnimationFrameCallback callback, long delay) {
        if (mAnimationCallbacks.size() == 0) {
            getProvider().postFrameCallback(mFrameCallback);
        }
        if (!mAnimationCallbacks.contains(callback)) {
            mAnimationCallbacks.add(callback);
        }

        if (delay > 0) {
            mDelayedCallbackStartTime.put(callback, (SystemClock.uptimeMillis() + delay));
        }
    }
```
监听屏幕每帧的刷新
```
private final Choreographer.FrameCallback mFrameCallback = new Choreographer.FrameCallback() {
        @Override
        public void doFrame(long frameTimeNanos) {
            doAnimationFrame(getProvider().getFrameTime());
            if (mAnimationCallbacks.size() > 0) {
                getProvider().postFrameCallback(this);
            }
        }
    };

```
当屏幕刷新每一帧的时候，会回调ValueAnimator的doAnimationFrame()方法
```
  public final boolean doAnimationFrame(long frameTime) {
        if (mStartTime < 0) {
            // First frame. If there is start delay, start delay count down will happen *after* this
            // frame.
            mStartTime = mReversing ? frameTime : frameTime + (long) (mStartDelay * sDurationScale);
        }

        // Handle pause/resume
        if (mPaused) {
            mPauseTime = frameTime;
            removeAnimationCallback();
            return false;
        } else if (mResumed) {
            mResumed = false;
            if (mPauseTime > 0) {
                // Offset by the duration that the animation was paused
                mStartTime += (frameTime - mPauseTime);
            }
        }

        if (!mRunning) {
            // If not running, that means the animation is in the start delay phase of a forward
            // running animation. In the case of reversing, we want to run start delay in the end.
            if (mStartTime > frameTime && mSeekFraction == -1) {
                // This is when no seek fraction is set during start delay. If developers change the
                // seek fraction during the delay, animation will start from the seeked position
                // right away.
                return false;
            } else {
                // If mRunning is not set by now, that means non-zero start delay,
                // no seeking, not reversing. At this point, start delay has passed.
                mRunning = true;
                startAnimation();
            }
        }

        if (mLastFrameTime < 0) {
            if (mSeekFraction >= 0) {
                long seekTime = (long) (getScaledDuration() * mSeekFraction);
                mStartTime = frameTime - seekTime;
                mSeekFraction = -1;
            }
            mStartTimeCommitted = false; // allow start time to be compensated for jank
        }
        mLastFrameTime = frameTime;
        // The frame time might be before the start time during the first frame of
        // an animation.  The "current time" must always be on or after the start
        // time to avoid animating frames at negative time intervals.  In practice, this
        // is very rare and only happens when seeking backwards.
        final long currentTime = Math.max(frameTime, mStartTime);
        boolean finished = animateBasedOnTime(currentTime);

        if (finished) {
            endAnimation();
        }
        return finished;
    }

```
只用关注animateBasedOnTime（）方法

```
   boolean animateBasedOnTime(long currentTime) {
        boolean done = false;
        if (mRunning) {
            final long scaledDuration = getScaledDuration();
            final float fraction = scaledDuration > 0 ?
                    (float)(currentTime - mStartTime) / scaledDuration : 1f;
            final float lastFraction = mOverallFraction;
            final boolean newIteration = (int) fraction > (int) lastFraction;
            final boolean lastIterationFinished = (fraction >= mRepeatCount + 1) &&
                    (mRepeatCount != INFINITE);
            if (scaledDuration == 0) {
                // 0 duration animator, ignore the repeat count and skip to the end
                done = true;
            } else if (newIteration && !lastIterationFinished) {
                // Time to repeat
                if (mListeners != null) {
                    int numListeners = mListeners.size();
                    for (int i = 0; i < numListeners; ++i) {
                        mListeners.get(i).onAnimationRepeat(this);
                    }
                }
            } else if (lastIterationFinished) {
                done = true;
            }
            mOverallFraction = clampFraction(fraction);
            float currentIterationFraction = getCurrentIterationFraction(
                    mOverallFraction, mReversing);
            animateValue(currentIterationFraction);
        }
        return done;
    }
```
会调用animateValue(float fraction)方法,先调用ObjectAnimator的animateValue，然后在调用ValueAnimator的animateValue

```
 @Override
    void animateValue(float fraction) {
        final Object target = getTarget();
        if (mTarget != null && target == null) {
            // We lost the target reference, cancel and clean up. Note: we allow null target if the
            /// target has never been set.
            cancel();
            return;
        }

        super.animateValue(fraction);
        int numValues = mValues.length;
        for (int i = 0; i < numValues; ++i) {
            mValues[i].setAnimatedValue(target);//设置属性
        }
    }
```

```
void animateValue(float fraction) {
        fraction = mInterpolator.getInterpolation(fraction);
        mCurrentFraction = fraction;
        int numValues = mValues.length;
        for (int i = 0; i < numValues; ++i) {
            mValues[i].calculateValue(fraction);
        }
        if (mUpdateListeners != null) {
            int numListeners = mUpdateListeners.size();
            for (int i = 0; i < numListeners; ++i) {
                mUpdateListeners.get(i).onAnimationUpdate(this);
            }
        }
    }
```
该方法会调用 PropertyValuesHolder中的calculateValue(float fraction)，这里以IntPropertyValuesHolder为例

```
 @Override
        void calculateValue(float fraction) {
            mIntAnimatedValue = mIntKeyframes.getIntValue(fraction);
        }
```
得到对于帧数上的动画值。

ValueAnimator中的 setAnimatedValue方法，具体实现类，根据你传入的是int 还是folat 生成IntPropertyValuesHolder，或FloatPropertyValuesHolder。
```
void setAnimatedValue(Object target) {
        if (mProperty != null) {
            mProperty.set(target, getAnimatedValue());
        }
        if (mSetter != null) {
            try {
                mTmpValueArray[0] = getAnimatedValue();
                mSetter.invoke(target, mTmpValueArray);
            } catch (InvocationTargetException e) {
                Log.e("PropertyValuesHolder", e.toString());
            } catch (IllegalAccessException e) {
                Log.e("PropertyValuesHolder", e.toString());
            }
        }
    }
```

这里以View竖直方向移动为例。那么
```
 public void setTranslationY(float translationY) {
        if (translationY != getTranslationY()) {
            invalidateViewProperty(true, false);
            mRenderNode.setTranslationY(translationY);//注意这里重点
            invalidateViewProperty(false, true);

            invalidateParentIfNeededAndWasQuickRejected();
            notifySubtreeAccessibilityStateChangedIfNeeded();
        }
    }

```
会调用invalidateViewProperty导致界面重绘
```
    void invalidateViewProperty(boolean invalidateParent, boolean forceRedraw) {
        if (!isHardwareAccelerated()
                || !mRenderNode.isValid()
                || (mPrivateFlags & PFLAG_DRAW_ANIMATION) != 0) {
            if (invalidateParent) {
                invalidateParentCaches();
            }
            if (forceRedraw) {
                mPrivateFlags |= PFLAG_DRAWN; // force another invalidation with the new orientation
            }
            invalidate(false);
        } else {
            damageInParent();
        }
    }
```
### ViewGroup处理事件


```
 if (!canViewReceivePointerEvents(child)
     || !isTransformedTouchPointInView(x, y, child, null)) {
        ev.setTargetAccessibilityFocus(false);
             continue;
                }
         newTouchTarget = getTouchTarget(child);
         if (newTouchTarget != null) {
           // Child is already receiving touch within its bounds.
           // Give it the new pointer in addition to the ones it is handling.
       newTouchTarget.pointerIdBits |= idBitsToAssign;
          break;
          }

```
```
    protected boolean isTransformedTouchPointInView(float x, float y, View child,
            PointF outLocalPoint) {
        final float[] point = getTempPoint();
        point[0] = x;
        point[1] = y;
        transformPointToViewLocal(point, child);
        //获取转换后的坐标
        //判断当前坐标是否在当前的view中
        final boolean isInView = child.pointInView(point[0], point[1]);
        if (isInView && outLocalPoint != null) {
            outLocalPoint.set(point[0], point[1]);
        }
        return isInView;
    }
```

该方法获取子View转换后的矩阵，判断当前坐标点，经过矩阵转换后，(如当前坐标点为100，100）准换后（20，20）是否在当前view中
```
public void transformPointToViewLocal(float[] point, View child) {
        point[0] += mScrollX - child.mLeft;
        point[1] += mScrollY - child.mTop;

        if (!child.hasIdentityMatrix()) {
            child.getInverseMatrix().mapPoints(point);
        }
    }
```
判断当前坐标点，是否在View中
```
    public boolean pointInView(float localX, float localY, float slop) {
        return localX >= -slop && localY >= -slop && localX < ((mRight - mLeft) + slop) &&
                localY < ((mBottom - mTop) + slop);
    }
```

### 传统动画

```
 public void startAnimation(Animation animation) {
        animation.setStartTime(Animation.START_ON_FIRST_FRAME);
        setAnimation(animation);
        invalidateParentCaches();
        invalidate(true);
    }
```
并没有改变矩阵的位置，所以那么当事件传递过来的时候，所以还是原来的位置。