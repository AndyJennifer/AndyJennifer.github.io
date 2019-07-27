---
title: 自定义View事件篇进阶篇(八)-NestedScrolling机制效果展示
tags:
  - 嵌套滑动
categories:
  - View事件篇
---

### 前言

在上篇文章中，我们分析了谷歌对NestedScrolling机制的设计，了解的不同接口的使用场景。现在就让我们一起结合一个实际的使用例子，来巩固之前学习的知识点吧。

### 效果展示

先看我们需要仿写的实际效果吧。如下图所示：

{% asset_img demo展示.gif %}

>上文展示的demo,在项目[NestedScrollingDemo](https://github.com/AndyJennifer/NestedScrollingDemo)有具体实现。

在上述Demo中，整个界面分为标题栏、展示图片、TabLayout、ViewPage。其中ViewPager中拥有多个Fragment。其中每个fragment中都对应着一个RecyclerView。整个Demo的实现效果如下所示：

- 当产生`向上`的手势滑动与fling时，如果展示图片没被父控件遮挡，那么父控件先拦截事件并滑动。当图片完全被遮挡时，子控件（RecyclerView)再接着处理。
- 当产生`向下`的手势滑动与fling时，如果展示图片没完全显示，那么父控件先拦截事件并滑动。当图片完全显示时，子控件（RecyclerView)再接着处理。
- 标题栏中的回退键，会随着父控件的滑动，有一个从白色到黑色的渐变效果。
- 标题栏中的透明度，会随着父控件的滑动，透明度从0到1的变化效果。

现在就让我们一起来实现该效果吧！！

### 接口使用分析

要实现嵌套滑动，我们首先想到的是要实现NestedScrollingChild与NestedScrollingParent接口，但是我们这里的Demo需要父控件处理部分fling,所以我们这里要使用NestedScrollingChild2与NestedScrollingParent2接口。又因为RecyclerView、NestedScrollView等滚动的View，在谷歌中都实现了NestedScrollingChild2接口，所以我们不用单独来处理子控件对手势滑动与fling的分发，我们只用关心父控件的处理就行了。

又因为整体布局为竖直方向，所以这里我们采用了继承LinearLayout并实现`NestedScrollingParent2`接口的方式。同时为了兼容低版本，我们也要使用`NestedScrollingParentHelper`这个帮助类。具体类实现类StickyNavLayout代码如下所示；

```java
public class StickyNavLayout extends LinearLayout implements NestedScrollingParent2 {

    private NestedScrollingParentHelper mNestedScrollingParentHelper = new NestedScrollingParentHelper(this);

    public StickyNavLayout(Context context) {
        this(context, null);
    }

    public StickyNavLayout(Context context, @Nullable AttributeSet attrs) {
        this(context, attrs, 0);
    }

    public StickyNavLayout(Context context, @Nullable AttributeSet attrs, int defStyleAttr) {
        super(context, attrs, defStyleAttr);
        setOrientation(VERTICAL);//设置布局方向为竖直方向。
    }

    @Override
    public boolean onStartNestedScroll(@NonNull View child, @NonNull View target, int axes, int type) {
         return (axes & ViewCompat.SCROLL_AXIS_VERTICAL) != 0
    }

    @Override
    public void onNestedScrollAccepted(@NonNull View child, @NonNull View target, int axes, int type) {
        mNestedScrollingParentHelper.onNestedScrollAccepted(child, target, axes, type);
    }

    @Override
    public void onNestedPreScroll(@NonNull View target, int dx, int dy, @NonNull int[] consumed, int type) {}


    @Override
    public void onNestedScroll(@NonNull View target, int dxConsumed, int dyConsumed, int dxUnconsumed, int dyUnconsumed, int type) {}

    @Override
    public boolean onNestedPreFling(View target, float velocityX, float velocityY) {
        return false;
    }

    @Override
    public void onStopNestedScroll(@NonNull View target, int type) {
        mNestedScrollingParentHelper.onStopNestedScroll(target, type);
    }

    @Override
    public int getNestedScrollAxes() {
        return mNestedScrollingParentHelper.getNestedScrollAxes();
    }
}
```

在上述代码中，我们需要注意以下几点：

- StickyNavLayout实现类默认方向为竖直方向。
- 为了让父控件处理竖直方向上的事件，我们需要在`onStartNestedScroll`方法判断`axes & ViewCompat.SCROLL_AXIS_VERTICAL`。
- 为了让子控件也处理fling，我们需要在`onNestedPreFling`方法中返回`false`。因为在嵌套滑动机制中，如果该方法返回true,那么子控件就没有机会处理fling了。
- 为了兼容低版本并获得正确的嵌套滑动状态，我们需要在onNestedScrollAccepted、onStopNestedScroll、onStopNestedScroll、中调用NestedScrollingParentHelper的相应方法。

### 布局设置

当我们把父控件（StickNavaLayout)的基本框架搭好后，现在就准备处理整个界面的布局了。观察Demo效果，我们发现当标题栏透明的时候，图片是完全展示的，那么也就说明标题栏布局的层级是在图片之上的。大致布局如下图所示：

{% asset_img 整体布局.png %}

继续观察Demo实现效果，我们可以发现得到如下几点：

- 当产生`向上`的手势滑动与fling)时，如果展示图片没被父控件遮挡，那么父控件先拦截事件并滑动。当图片完全被遮挡时，子控件再接着处理。
- 当产生`向下`的手势滑动与fling)时，如果展示图片没完全显示，那么父控件先拦截事件并滑动。当图片完全显示时，子控件再接着处理。

那么展示图片遮挡的效果是如何实现的呢？其实很简单，我们只需要在我们的父控件中添加一个与展示图片相同高度的透明的View就行了。那么当父控件在滚动的时候，就可以产生一种遮盖的效果啦。具体设计如下图所示：

{% asset_img StickyNavLayout布局.png %}

那么再对应Android的布局文件，整个界面的布局大概是下面这个样子：

```xml
<?xml version="1.0" encoding="utf-8"?>
<RelativeLayout
    xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    android:layout_width="match_parent"
    android:layout_height="match_parent">

    <!--展示图片-->
    <ImageView
        android:layout_width="match_parent"
        android:layout_height="200dp"
        android:scaleType="fitXY"
        android:src="@drawable/ic_launcher_background"/>

    <!--标题栏-->
    <include layout="@layout/layout_common_toolbar"/>

    <!--嵌套滑动父控件-->
    <com.jennifer.andy.nestedscrollingdemo.view.StickyNavLayout
        android:id="@+id/sick_layout"
        android:layout_width="match_parent"
        android:layout_height="match_parent">

        <!--透明TopView-->
        <View
            android:id="@+id/sl_top_view"
            android:layout_width="match_parent"
            android:layout_height="200dp"/>
        <!--TabLayout-->
        <android.support.design.widget.TabLayout
            android:id="@+id/sl_tab"
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:background="#fff"
            app:tabIndicatorColor="@color/colorPrimaryDark"/>
        <!--ViewPager-->
        <android.support.v4.view.ViewPager
            android:id="@+id/sl_viewpager"
            android:layout_width="match_parent"
            android:layout_height="match_parent"
            android:background="#fff"/>
    </com.jennifer.andy.nestedscrollingdemo.view.StickyNavLayout>

</RelativeLayout>
```

### 父控件滑动范围

在完成了整体界面的布局后，现在我们需要处理父控件的滚动。继续观察Demo，我们能发现父控件滚动的范围为展示图片的高度减去标题栏的高度。为了计算父控件的滚动范围，我们需要获取父控件内部的TopView（与展示图片高度相同的透明View)的高度。要获取父控件的子控件，我们可以通过`onFinishInflate`方法。具体代码如下所示：

```java
    private View mTopView;//与展示图片高度相同的透明View

   @Override
    protected void onFinishInflate() {
        super.onFinishInflate();
        mTopView = findViewById(R.id.sl_top_view);
    }
```

获取了子控件后,我们可以在onSizeChanged中得到，可以父控件可以滑动的距离（mCanScrollDistance = 展示图片的高度 - 标题栏的高度）。具体代码如下所示：

```java
    @Override
    protected void onSizeChanged(int w, int h, int oldw, int oldh) {
        mCanScrollDistance = mTopView.getMeasuredHeight() - getResources().getDimension(R.dimen.normal_title_height);
    }
```

因为标题栏的高度基本都是48dp,所以我这里并没单独去获取标题栏的高度，而是在values/dimens文件中声明了normal_title_height = 48dp。

### 父控件嵌套滑动实现

处理了父控件的滑动范围，现在到了最关键的嵌套滑动的处理了。当ViewPager中的RecyclerView接受到滑动后，会将滑动先分发给父控件，我们的父控件(StickyNavaLayout)需要判断是否进行消耗，而判断是否消耗的条件如下：

- 当向下滑动时，如果RecyclerView不能继续向下滑动且父控件(StickyNavaLayout)已经滑动了移动距离后，父控件(StickyNavaLayout)需要消耗。
- 当向上滑动时，如果父控件(StickyNavaLayout)已经滑动了部分距离，那么父控件(StickyNavaLayout)需要消耗需要消耗。

具体代码如下所示：

```java
    @Override
    public void onNestedPreScroll(@NonNull View target, int dx, int dy, @NonNull int[] consumed, int type) {
        //如果子view欲向上滑动，则先交给父view滑动
        boolean hideTop = dy > 0 && getScrollY() < mCanScrollDistance;
        //如果子view欲向下滑动，必须要子view不能向下滑动后，才能交给父view滑动
        boolean showTop = dy < 0 && getScrollY() >= 0 && !target.canScrollVertically(-1);
        if (hideTop || showTop) {
            scrollBy(0, dy);
            consumed[1] = dy;// consumed[0] 水平消耗的距离，consumed[1] 垂直消耗的距离
        }
    }
```

在上述代码中，我们通过调用View的canScrollVertically(int direction)方法来判断是否能够向下滑动，其中当**direcation**为`负数`时，是检查对应View是否能够向下滑动，能，返回为true，反之返回false。当**direcation**为`正数`时，是检查对应View是否能够向上滑动，能，返回为true,反之返回false。

需要注意的是在onNestedPreScroll方法中，我们并没有区分是手势滑动还是fling，也就是区分type为`TYPE_TOUCH(0)`还是`TYPE_NON_TOUCH(1)`。因为不管是手势滑动还是fling。在Demo效果中父控件都需要处理。所以我们并没有进行判断。

当我们处理了onNestedPreScroll方法后，我们还需要处理`onNestedScroll`方法。因为根据嵌套滑动机制，当父控件预处理后，子控件会再消耗剩余的距离，如果子控件消耗后，还有剩余的距离。那么就又会传递给父控件。也就是会走onNestedScroll方法。在该方法中，我们只需要单独处理子控件的剩余的`向下fling`。具体代码如下所示：

```java
  @Override
    public void onNestedScroll(@NonNull View target, int dxConsumed, int dyConsumed, int dxUnconsumed, int dyUnconsumed, int type) {
        if (dyUnconsumed < 0 && type == ViewCompat.TYPE_NON_TOUCH) {//表示已经向下滑动到头，且为fling
            scrollBy(0, dyUnconsumed);
        }
    }

```

>当子控件产生fling时，如果子控件消耗不完，那么就会传递给父控件。也就是dyConsumed肯定是有值的，又因为我们只关心向下的fling。所以上述代码这样判断。

完成了嵌套滑动的处理后，我们还需要对父控件(StickyNavaLayout)的滚动范围进行校验，我们直接重写scrollTo方法。进行判断就好了。

```java
    @Override
    public void scrollTo(int x, int y) {
        if (y < 0) {
            y = 0;
        }
        if (y > mCanScrollDistance) {
            y = (int) mCanScrollDistance;
        }
        if (getScrollY() != y) super.scrollTo(x, y);
    }
```

父控件(StickyNavaLayout)的滚动范围为0-mCanScrollDistance。其中mCanScrollDistance = 展示图片的高度 - 标题栏的高度。

### ViewPager高度的矫正

到现在大家可能觉得基本的嵌套滑动就结束了。但是如果你这样写的话你会发现一个问题：就是当我们的父控件(StickyNavaLayout)滚动到标题栏下后，我们会发现我们的ViewPager并没有填充屏幕剩下的距离，而是会有一个空白距离。如下所示：

{% asset_img 空白区域.png %}

是因为我们的父控件(StickyNavaLayout)继承了LinearLayout且ViewPager的高度为`match_parent`，那么根据View的测量规则，ViewPager实际的高度为屏幕中剩余的高度。所以父控件(StickyNavaLayout)滚动到标题栏下后，会出现一段空白，那么为了使ViewPager填充整个屏幕，我们需要重新设置ViewPager的高度。也就是我们需要重写父控件(StickyNavaLayout)的onMeasure方法。具体代码如下所示：

```java
    @Override
    protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
        //先测量一次
        super.onMeasure(widthMeasureSpec, heightMeasureSpec);
        //ViewPager修改后的高度= 总高度-TabLayout的高度
        ViewGroup.LayoutParams lp = mViewPager.getLayoutParams();
        lp.height = getMeasuredHeight() - mNavView.getMeasuredHeight();
        mViewPager.setLayoutParams(lp);
        //因为ViewPager修改了高度，所以需要重新测量
        super.onMeasure(widthMeasureSpec, heightMeasureSpec);
    }
```

### 渐变效果实现

现在我们就剩下最后两个效果了，回退键渐变与标题栏的透明度的变化了，其实实现也非常简单，因为我们的父控件(StickyNavaLayout)有一个最大滑动的范围，那么我们就可以得到当前父控件滑动的距离与最大滑动范围的比例，拿到这个比例后，我们可以设置标题栏的透明度。也可以通过谷歌提供的ArgbEvaluator得到渐变颜色。具体的实现方式，读者朋友可以自行思考解决。因为篇幅的限制，这里就不在讲解具体的实现方式了。有需要的小伙伴，可以参看项目[NestedScrollingDemo](https://github.com/AndyJennifer/NestedScrollingDemo)中的[NestedScrolling2DemoActivity](https://github.com/AndyJennifer/NestedScrollingDemo/blob/master/app/src/main/java/com/jennifer/andy/nestedscrollingdemo/ui/nested/NestedScrolling2DemoActivity.java)中的具体实现。

### 最后

整个Demo就讲解完毕了，大家有什么问题，欢迎提出~
