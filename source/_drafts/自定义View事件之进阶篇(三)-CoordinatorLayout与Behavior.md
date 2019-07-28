---
title: 自定义View事件篇进阶篇(三)-CoordinatorLayout与Behavior
tags:
  - 嵌套滑动
categories:
  - View事件篇
---

### 前言

在上篇文章中，我们介绍了NestedScrolling(嵌套滑动)机制，介绍了子控件与父控件嵌套滑动的处理。现在我们来了解谷歌大大为我们提供的另一个控件的交互布局`CoordainatorLayout`。CoordainatorLayout对于Android开发老司机来说肯定不会陌生，作为控制内部一个或多个的子控件协同交互的容器，开发者可以通过设置Behavior去控制多个控件的协同交互效果，测量尺寸、布局位置及触摸响应。作为谷歌推出的明星组件，分析CoordainatorLayout的文章已是数不胜数。而分析整个CoordainatorLayout原理的相关资料在网上很少，因此本文会把重点放在分析其内部原理上。

通过阅读该文，你能了解如下知识点：

- CoordainatorLayout中Behavior中的基础使用
- CoordainatorLayout中多个控件协同交互的原理
- CoordainatorLayout中Behavior的实例化过程
- Behavior实现嵌套滑动的原理与过程
- Behavior自定义布局的时机与过程
- Behavior自定义测量的时机与过程

> 该博客中涉及到的示例，在[NestedScrollingDemo](https://github.com/AndyJennifer/NestedScrollingDemo)项目中都有实现，大家可以按需自取。

### CoordainatorLayout简介

熟悉CoordinatorLayout的小伙伴，肯定知道CoordinatorLayout主要实现以下四个功能：

- 处理子控件的依赖下的交互
- 处理子控件的嵌套滑动
- 处理子控件的测量与布局
- 处理子控件的事件拦截与响应。

而上述四个功能，都依托于CoordainatorLayout中提供的一个叫做`Behavior`的“插件”。Behavior内部也提供了相应方法来对应这四个不同的功能，具体如下所示：

{% asset_img Behavior方法设置.jpg%}

>在下面的文章中不会介绍Behavior嵌套滑动相关方法的作用，如果需要了解这些方法的作用，建议参看{% post_link 自定义View事件之进阶篇(一)-NestedScrolling(嵌套滑动)机制 %}文章下的方法介绍。

那现在我们就一起来看看，谷歌是怎么围绕Behavior对上述四个功能进行设计的把。

#### 子控件依赖下的交互设计

对于子控件的依赖交互，谷歌是这样设计的：

{% asset_img 依赖下的交互.jpg %}

当CoordainatorLayout中子控件（childView1)的位置、大小等发生改变的时候，那么在CoordainatorLayout内部会通知所有依赖childView1的控件，并调用对应声明的Behavior，告知其依赖的childView1发生改变。那么如何判断依赖，接受到通知后如何处理。这些都交由Behavior来处理。

#### 子控件的嵌套滑动的设计

对于子控件的嵌套滑动，谷歌是这样设计的：

{% asset_img 嵌套滑动设计.jpg%}

CoordinatorLayout实现了NestedScrollingParent2接口。那么当事件（scroll或fling)产生后，内部实现了NestedScrollingChild接口的子控件会将事件分发给CoordinatorLayout，CoordinatorLayout又会将事件传递给所有的Behavior。接着在Behavior中实现子控件的嵌套滑动。那么再结合上文提到的Behavior中嵌套滑动的相关方法，我们可以得到如下流程：

{% asset_img 嵌套滑动整体流程.jpg%}

观察谷歌的设计，我们可以发现，相对于NestedScrolling机制（参与角色只有子控件和父控件），CoordainatorLayout中的交互角色更为丰富，在CoordainatorLayout下的**子控件可以与多个兄弟控件进行交互。**

#### 子控件的测量、布局、事件的设计

看了谷歌对子控件的嵌套滑动设计，我们再来看看子控件的测量、布局、事件的设计：

{% asset_img 布局与测量及事件的设计.jpg%}

因为CoordainatorLayout主要负责的是子控件之间的交互，内部控件的测量与布局，就简单的类似FrameLayout处理方式就好了。在特殊的情况下，如子控件需要处理宽高和布局的时候，那么交由Behavior内部的`onMeasureChild`与`onLayoutChild`方法来进行处理。同理对于事件的拦截与处理，如果子控件需要拦截并消耗事件，那么交由给Behavior内部的`onInterceptTouchEvent`与`onTouchEvent`方法进行处理。

可能有的小伙伴会想，为什么会将这四种功能对于的方法将这些功能都交由Behavior实现。其实原因非常简单，因为将所有功能都对应在Behavior中，那么对于子控件来说，这种插件化的方式就非常解耦了，我们的子控件无需将效果写死在自身中，我们只需要对应不同的Behavior，就可以实现不同的效果了。如下所示：

{% asset_img 控件对应多个Behavior.jpg%}

### CoordainatorLayout下的多个子控件的依赖交互

了解了CoordainatorLayout中四种功能的设计后，现在我们通过一个例子来讲解CoordainatorLayout下多个子控件的交互。在讲解具体的例子之前，我们先回顾一下Behavior中对子控件依赖交互提供的方法。如下所示：

```java
public boolean layoutDependsOn(CoordinatorLayout parent, V child, View dependency) { return false; }
public boolean onDependentViewChanged(CoordinatorLayout parent, V child, View dependency) {return false; }
public void onDependentViewRemoved(CoordinatorLayout parent, V child, View dependency) {}
```

**layoutDependsOn方法介绍：**

确定一个控件（childView1)依赖另外一个控件(childView2)的时候，是通过`layoutDependsOn`这个方法。其中child是依赖对象(childView1)，而dependency是被依赖对象(childView2)，该方法的返回值是判断是否依赖对应view。如果返回true。那么表示依赖。反之不依赖。一般情况下，在我们自定义Behavior时，我们需要重写该方法。当`layoutDependsOn`方法返回true时，后面的`onDependentViewChanged`与`onDependentViewRemoved`方法才会调用。

**onDependentViewChanged方法介绍：**

当一个控件（childView1)所依赖的另一个控件(childView2)位置、大小发生改变的时候，该方法会调用。其中该方法的返回值，是由childView1来决定的，如果childView1在接受到childView2的改变通知后，childView1的位置或大小发生改变，那么就返回true,反之返回false。

**onDependentViewRemoved方法介绍：**

当一个控件（childView1)所依赖的另一个控件(childView2)被删除的时候，该方法会调用。

#### Demo展示

下面我们就看一种简单的例子，来讲解在使用CoordainatorLayout下各个兄弟控件之间的依赖产生的交互效果。

{% asset_img 效果展示.gif %}

在上述Demo中，我们创建了一个随手势滑动的`DependedView`,并设定了另外两个依赖DependedView的TextView的Behavior，BrotherChameleonBehavior（变色小弟）与BrotherFollowBehavior（跟随小弟）。具体代码如下所示：

```java
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

DependedView逻辑非常简单，就是重写了onTouchEvent，监听滑动，并设置DependedView的位置。我们继续查看另外两个TextView的Behavior。

BrotherChameleonBehavior（变色小弟）代码如下所示：

>在CoordainatorLayout中要实现子控件的依赖交互，我们需要继承CoordinatorLayout.Behavior。并实现layoutDependsOn、onDependentViewChanged、onDependentViewRemoved方法，因为我们Demo中不设计关于依赖控件的删除，故没有重写onDependentViewRemoved方法。

```java
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

BrotherFollowBehavior（跟随小弟)代码如下所示：

```java
public class BrotherFollowBehavior extends CoordinatorLayout.Behavior<View> {

    public BrotherFollowBehavior(Context context, AttributeSet attrs) {
        super(context, attrs);
    }

    @Override
    public boolean layoutDependsOn(CoordinatorLayout parent, View child, View dependency) {
        return dependency instanceof DependedView;//判断依赖的是否是DependedView
    }

    @Override
    public boolean onDependentViewChanged(CoordinatorLayout parent, View child, View dependency) {
        //如果DependedView的位置、大小改变，跟随小弟始终在DependedView下面
        child.setY(dependency.getBottom() + 50);
        child.setX(dependency.getX());
        return true;
    }
}
```

比较重要的布局文件怎么能忘了呐，对应的布局如下：

```xml
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

#### 原理讲解

大家肯定会很好奇，为什么简简单单的设置了两个Behavior,DependedView位置发生改变的时候就能通知依赖的两个TextView呢？这要从DependedView的onTouchEvent方法说起。在onTouchEvent方法中，我们根据手势修改了DependedView的位置，我们都知道当子控件位置、大小发生改变的时候，会导致父控件重绘。也就是会调用`onDraw`方法。而CoordainatorLayout在`onAttachedToWindow`中使用了`ViewTreeObserver`，并设置了绘制前监听器`OnPreDrawListener`。如下所示：

```java
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
       //省略部分代码：
    }
```

熟悉ViewTreeObserver的小伙伴一定清楚，该类主要是监测整个View树的变化（状态变化，或者内部的View可见性变化等），我们继续跟踪OnPreDrawListener，查看CoordainatorLayou在绘制前做了什么。

```java
  class OnPreDrawListener implements ViewTreeObserver.OnPreDrawListener {
        @Override
        public boolean onPreDraw() {
            onChildViewsChanged(EVENT_PRE_DRAW);
            return true;
        }
    }
```

我们发现其内部调用了`onChildViewsChanged(EVENT_PRE_DRAW);`方法。我们继续查看该方法。

```java
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
                    //省略部分代码....
                    final boolean handled;
                    switch (type) {
                        case EVENT_VIEW_REMOVED:
                            b.onDependentViewRemoved(this, checkChild, child);
                            handled = true;
                            break;
                        default:
                            // 如果依赖，那么就会走当前子控件Behavior中的onDependentViewChanged方法。
                            handled = b.onDependentViewChanged(this, checkChild, child);
                            break;
                    }

                }
            }
        }
    //省略部分代码...
    }
```

观察代码，我们发现程序中使用了一个名为`mDependencySortedChildren`的集合，通过遍历该集合，我们可以获取集合中控件的`LayoutParam`，得到LayoutParam后，我们可以继续获取相应的`Behavior`。并调用其`layoutDependsOn`方法找到所依赖的控件，如果找到了当前控件所依赖的另一控件，那么就调用Behavior中的`onDependentViewChanged`方法。到这里，我相信大家应该明白多个控件依赖交互的原理。现在还剩下mDependencySortedChildren集合了。我们看看这个集合中是存储了什么东西。

```java
 @Override
    protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
        prepareChildren();
        //省略部分代码...
    }
```

`mDependencySortedChildren`中元素是在onMeasure方法中的`prepareChildren()`中进行添加的，我们继续查看该方法。方法如下所示：

```java
  private void prepareChildren() {
        mDependencySortedChildren.clear();
        mChildDag.clear();
        //遍历内部所有孩子
        for (int i = 0, count = getChildCount(); i < count; i++) {
            final View view = getChildAt(i);

            final LayoutParams lp = getResolvedLayoutParams(view);
            lp.findAnchorView(this, view);

            mChildDag.addNode(view);

            // 再次迭代获取子类控件，找到依赖控件并添加到"(mchildDag)图"中
            for (int j = 0; j < count; j++) {
                if (j == i) {
                    continue;
                }
                final View other = getChildAt(j);
                if (lp.dependsOn(this, view, other)) {
                    if (!mChildDag.contains(other)) {
                        //添加到图中
                        mChildDag.addNode(other);
                    }
                    // 添加边（依赖的view)
                    mChildDag.addEdge(other, view);
                }
            }
        }

        mDependencySortedChildren.addAll(mChildDag.getSortedList());
        //省略部分代码
    }
```

prepareChildren方法中，会遍历内部所有的子控件，并将子控件添加到`mChildDag`集合中，大家不用去关系`mChildDag`的数据结构是什么，大家只要知道该数据类型是一种叫图的数据结构就行了。在方法最后，又会将`mChildDag`中的数据，全部添加到mDependencySortedChildren中去。

#### Behavior的实例化

现在我们来讲解下一个知识点，在上述文章中，我们只描述了CoordainatorLayout中子控件的依赖交互原理，讲解了Behavior依赖相关方法的调用时机，我们并没有讲解Behavior是如果同xml配置，生成实例Behavior对象的。现在我们来看看Behavior是何时被实例化的。

```java
 public static class LayoutParams extends MarginLayoutParams {
        Behavior mBehavior;
 }
```

在CoordainatorLayout中自定义了布局参数`LayoutParams`，并在LayoutParms声明了`Behavior`，同时重写了`generateLayoutParams`方法。

```java
   @Override
    public LayoutParams generateLayoutParams(AttributeSet attrs) {
        return new LayoutParams(getContext(), attrs);
    }
```

熟悉自定义View的小伙伴一定熟悉`generateLayoutParams`方法，当我们自定义父控件时，如果我们的子控件需要一些特殊的布局参数，我们一般会自定义`LayoutParams`。比如Relativelayout中声明的LayoutParms中包含`alight_top`,`to_left_of`一样。好了回过头来，我们继续查看LayoutParams的构造函数。代码如下所示：

```java
LayoutParams(Context context, AttributeSet attrs) {
            super(context, attrs);

            //省略部分代码....

            //判断是否声明了Behavior
            mBehaviorResolved = a.hasValue(
                    R.styleable.CoordinatorLayout_Layout_layout_behavior);
            if (mBehaviorResolved) {
                mBehavior = parseBehavior(context, attrs, a.getString(
                        R.styleable.CoordinatorLayout_Layout_layout_behavior));
            }
            a.recycle();

            if (mBehavior != null) {
                // If we have a Behavior, dispatch that it has been attached
                mBehavior.onAttachedToLayoutParams(this);
            }
        }
```

当子控件的布局参数实例化的时候，会在布局文件中判断是否声明了`layout_behavior`，如果声明了就调用`parseBehavior`方法来实例化Behavior对象。具体代码如下所示：

```java
  static Behavior parseBehavior(Context context, AttributeSet attrs, String name) {
        if (TextUtils.isEmpty(name)) {
            return null;
        }

        final String fullName;
        if (name.startsWith(".")) {
            // Relative to the app package. Prepend the app package name.
            fullName = context.getPackageName() + name;
        } else if (name.indexOf('.') >= 0) {
            // Fully qualified package name.
            fullName = name;
        } else {
            // Assume stock behavior in this package (if we have one)
            fullName = !TextUtils.isEmpty(WIDGET_PACKAGE_NAME)
                    ? (WIDGET_PACKAGE_NAME + '.' + name)
                    : name;
        }

        try {
            Map<String, Constructor<Behavior>> constructors = sConstructors.get();
            if (constructors == null) {
                constructors = new HashMap<>();
                sConstructors.set(constructors);
            }
            Constructor<Behavior> c = constructors.get(fullName);
            if (c == null) {
                final Class<Behavior> clazz = (Class<Behavior>) context.getClassLoader()
                        .loadClass(fullName);
                c = clazz.getConstructor(CONSTRUCTOR_PARAMS);
                c.setAccessible(true);
                constructors.put(fullName, c);
            }
            return c.newInstance(context, attrs);
        } catch (Exception e) {
            throw new RuntimeException("Could not inflate Behavior subclass " + fullName, e);
        }
    }
```

parseBehavior方法其实很简单，就是根据相应的Behavior全限定名称，通过反射调用其构造函数(自定义Behavior的时候，一定要写构造函数），并实例化其对象。当然实例化Behavior的方法不止一种，Google还为我们提供了注解的方法设置Behavior。例如AppBarLayout中的设置：

```java
@CoordinatorLayout.DefaultBehavior(AppBarLayout.Behavior.class)
public class AppBarLayout extends LinearLayout {}
```

当然使用注解的方式，其原理也是通过反射调用相应Behavior构造函数，并实例化对象。只是需要通过合适的时间解析注解罢了，因为篇幅的限制，这里不再讲解注解实例化Behavior的原理与时机了，有兴趣的小伙伴可以自行研究。

### Behavior实现嵌套滑动的原理与过程

在上文CoordinatorLayout简介中，我们简单介绍了CoordinatorLayout嵌套滑动事件的传递过程与Behavior嵌套滑动的相关方法。从CoordinatorLayout到Behavior的整个流程相对来说较为复杂，整体流程如下所示：

{% asset_img 嵌套滑动流程图.jpg %}

现在我们就抽丝剥茧，详细了解其内部实现机制与整体过程。

#### CoordainatorLayout的事件传递过程

Behavior的嵌套滑动其实都是围绕CoordainatorLayout的的`onInterceptTouchEvent`与`onTouchEvent`方法展开的。那我们先从onInterceptTouchEvent方法讲起，具体代码如下所示：

```java
public boolean onInterceptTouchEvent(MotionEvent ev) {
        final int action = ev.getActionMasked();
        //省略部分代码...
        final boolean intercepted = performIntercept(ev, TYPE_ON_INTERCEPT);
        //省略部分代码...
        return intercepted;
    }
```

在CoordainatorLayout的的`onInterceptTouchEvent`方法中，内部其实是调用了`performIntercept`来处理是否拦截事件，我们继续查看performIntercept方法。具体代码如下所示：

```java
    private boolean performIntercept(MotionEvent ev, final int type) {
        boolean intercepted = false;
        boolean newBlock = false;

        MotionEvent cancelEvent = null;

        final int action = ev.getActionMasked();
        //获取内部的控件集合，并按照z轴进行排序
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
                if (b != null) {
                    //省略部分代码....
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
            //省略部分代码....
        }
        //省略部分代码....
        return intercepted;//是否拦截与CoordinatorLayout中子view的behavior有关
    }
```

整个方法代码的逻辑并不是很难，主要分为两个步骤：

- 获取内部的控件集合（**topmostChildList**），并按照z轴进行排序
- 循环遍历**topmostChildList**，获取控件的Behavior，并调用Behavior的onInterceptTouchEvent方法判断是否拦截事件，如果拦截事件，则事件又会交给CoordinatorLayout的`onTouchEvent`方法处理。

>这里我们先不考虑Behavior拦截事件，一般情况下，Behavior的`onInterceptTouchEvent`方法基本都是返回false。特殊情况下Behavior事件拦截处理，大家可以在理解本文章所有的知识点后，结合官方提供的`BottomSheetBehavior`、`SwipeDismissBehavior`等进行深入的研究，这里因为篇幅的限制就不再深入的探讨了。

那么假设现在所有的子控件中的Behavior.onInterceptTouchEvent返回为`false`,那么CoordinatorLayout就不会拦截事件，根据事件传递机制，事件就传递到了子控件中去了。如果我们的子控件实现是了NestedScrollingChild接口（如RecyclerView或NestedScrollView),并且在onTouchEvent方法调用了相关嵌套滑动API,那么再根据嵌套滑动机制，会调用实现了NestedScrollingParent2接口的父控件的相应方法。又因为CoordinatorLayout实现了NestedScrollingParent2接口。那么就又回到了我们最开始的介绍的嵌套滑动机制了。这里的理解非常重要！！！！！非常重要！！！！非常重要！！！如果没有理解，建议多读几遍。

既然最终会调用CoordinatorLayout的嵌套滑动方法。那我们来介绍CoordinatorLayout下比较有代表性的嵌套滑动方法，那么先来看onStartNestedScroll方法。具体代码如下：

``` java
    public boolean onStartNestedScroll(View child, View target, int axes, int type) {
        boolean handled = false;

        final int childCount = getChildCount();
        for (int i = 0; i < childCount; i++) {
            final View view = getChildAt(i);
            if (view.getVisibility() == View.GONE) {
                //如果当前控件隐藏，则不传递
                continue;
            }
            final LayoutParams lp = (LayoutParams) view.getLayoutParams();
            final Behavior viewBehavior = lp.getBehavior();
            if (viewBehavior != null) {
                //判断Behavior是否接受嵌套滑动事件
                final boolean accepted = viewBehavior.onStartNestedScroll(this, view, child,
                        target, axes, type);
                handled |= accepted;
                //设置当前子控件接受接受嵌套滑动
                lp.setNestedScrollAccepted(type, accepted);
            } else {
                lp.setNestedScrollAccepted(type, false);
            }
        }
        return handled;
    }
```

在该方法中，我们会发现会获取所有的内部的控件，并调用对应Behavior的`onStartNestedScroll`方法，需要注意的是，如果当前Behavior接受嵌套滑动事件（accepted = true)，那么就会调用`lp.setNestedScrollAccepted(type, accepted)`,这段代码非常重要，会影响Behavior后续的嵌套方法的执行。我们接着看CoordinatorLayout下的`onNestedScrollAccepted`方法。代码如下所示：

```java
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

同样在onNestedScrollAccepted方法中，也会调用所有控件的Behavior的onNestedScrollAccepted方法，需要注意的是，在该方法中增加了`if (!lp.isNestedScrollAccepted(type))`的判断，也就是说只有Behavior的`onStartNestedScroll`方法返回true的时候，该方法才会执行。接下来继续查看onNestedScroll方法。具体代码如下所示：

```java
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

同样的，在onNestedScroll方法中，也会判断当前控件对应Behavior是否接受嵌套滑动事件，如果接受就调用对应方法。在代码的最后一行，我们会发现又调用了`onChildViewsChanged(EVENT_NESTED_SCROLL)`。该行代码在CoordinatorLayout下多出嵌套滑动方法中都会调用，我们先看onNestedPreScroll方法。然后再来介绍`onChildViewsChanged(EVENT_NESTED_SCROLL)`方法调用下的逻辑处理。onNestedPreScroll代码如下所示：

```java
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
            //这里也调用了onChildViewsChanged方法
            onChildViewsChanged(EVENT_NESTED_SCROLL);
        }
    }
```

同样的在该方法中，也是调用子控件的Behavior对应的方法，并最后调用了`onChildViewsChanged(EVENT_NESTED_SCROLL)`。该方法与其他方法的最大的不同就是，用`int[] mTempIntPair = new int[2]`记录了控件在X轴与Y轴的距离，比较并获取内部子控件中`最大`的消耗距离后，最后将最大的消耗距离，通过`int[]consumed`数组在传回NestedScrollingChild。

在CoordinatorLayout下的比较重要的嵌套滑动方法基本上讲解完毕了。余下的`onNestedPreFling`与`onNestedFling`方法都大同小异，这里就不再讲解了，现在讲解一下当`onChildViewsChanged(EVENT_NESTED_SCROLL)`方法调用下的逻辑处理。代码如下所示：

```java
   final void onChildViewsChanged(@DispatchChangeEvent final int type) {
        final int layoutDirection = ViewCompat.getLayoutDirection(this);
        final int childCount = mDependencySortedChildren.size();
        // 省略部分代码...
        for (int i = 0; i < childCount; i++) {
            // 省略部分代码...
            for (int j = i + 1; j < childCount; j++) {
                final View checkChild = mDependencySortedChildren.get(j);

                //获取对应控件的Behavior
                final LayoutParams checkLp = (LayoutParams) checkChild.getLayoutParams();
                final Behavior b = checkLp.getBehavior();

                if (b != null && b.layoutDependsOn(this, checkChild, child)) {
                    //这里是理解难点，需要多次回味。
                    if (type == EVENT_PRE_DRAW && checkLp.getChangedAfterNestedScroll()) {
                        //检查当前控件的嵌套滑动的标志位，如果为true,表示已经嵌套滑动过了，那么就跳过
                        checkLp.resetChangedAfterNestedScroll();
                        continue;
                    }

                    final boolean handled;
                    //这里判断所依赖的对象是否移除或改变
                    switch (type) {
                        case EVENT_VIEW_REMOVED://移除
                            //当类型为EVENT_VIEW_REMOVED时，表示该控件移除，我们要通知依赖该控件的其他控件，该控件已经被移除了
                            b.onDependentViewRemoved(this, checkChild, child);
                            handled = true;
                            break;
                        default:
                            //默认情况下，通知通知依赖该控件的其他控件，该控件发生了改变
                            handled = b.onDependentViewChanged(this, checkChild, child);
                            break;
                    }

                    if (type == EVENT_NESTED_SCROLL) {
                        // If this is from a nested scroll, set the flag so that we may skip
                        // any resulting onPreDraw dispatch (if needed)
                        //如果当前是嵌套滑动，那么就需要设置该标志位为true,方便跳过OnPreDraw方法
                        checkLp.setChangedAfterNestedScroll(handled);
                    }
                }
            }
        }
        //省略部分代码
    }
```

整个方法分为一下几个步骤：

- 获取控件的Behavior,调用其`layoutDependsOn`方法判断是否依赖，找到依赖该控件的其他控件。
- 随后调用控件的LayoutParams的`getChangedAfterNestedScroll()`方法，检查当前控件的嵌套滑动的标志位，如果为true,表示已经嵌套滑动过了，那么就跳过。如果该标志位为`false`，那程序继续往下走。
- 如果找到依赖控件其嵌套滑动标志位也为`false`，那么接下来会调用依赖控件的Behavior的`onDependentViewChanged`方法，通知其他控件依赖的控件位置、大小发生了改变。
- 通知完毕后，如果其他的控件位置、大小发生了改变，那么需要在`onDependentViewChanged`方法中返回为`true(handlet=true)`,如果`type==EVENT_NESTED_SCROLL`那么需要调用`ChangedAfterNestedScroll`，设置当前控件已经嵌套滑动的标志位为`true`。

整个流程并不是很复杂，但是我向下大家会有一个疑问，就是为什么`type==EVENT_NESTED_SCROLL`时，需要设置控件的嵌套滑动标志位呢？为什么当该标志位为true的时候，就需要跳过循环呢？其实这两个问题并不难，我们看下图：

{% asset_img 逻辑理解.jpg %}

根据上图，我们来回顾一下整个机制的嵌套滑动过程。

- 当CoordinatorLayout中子控件的Behvior默认不拦截事件，且内部有NestedScrollingChild控件的时候。最终会调用到某个控件的Behavior的嵌套相关方法，这里以A控件为例。
- 在A控件部分相关嵌套方法中，会调用`onChildViewsChanged(EVENT_NESTED_SCROLL)`。在该方法中又会通知其他依赖A控件的其他控件。并调用onDependentViewChanged方法（上图中，蓝色与红色部分）。
- 因为A控件在执行部分嵌套滑动方法后，会导致父控件重绘，所以又会回到本文最初讲解的`onPreDraw`方法，在该方法中，又会调用`onChildViewsChanged(EVENT_PRE_DRAW)`（上图中黄色部分）。

根据当前整体流程，我们可以推断出，如果不通过设置控件的嵌套滑动标志位的话，那么其他依赖A控件的Behavior就会调用`两次onDependentViewChanged`,如果说其他控件都在该方法中发生了位置、或大小的改变。那么整个过程就会出现问题！！！！！。所以说我们需要一个标志位来区分绘制与嵌套滑动。

当然这个嵌套滑动的标志位，是与Behavior的`onDependentViewChanged`方法的返回值有关，所以在平时的开发中，我们一定要注意。如果我们当我们对目标控件的位置、大小造成了改变之后，我们一定要将该方法的返回值返回为`true`。

### Behavior的布局

还有最后两个知识点了，大家加油啊~~~

我们都知道CoordinatorLayout中被谷歌称为超级FrameLayout，其中的原因不仅因为其布局方式与测量方式与FrameLayout非常相似以外，最主要的原因是CoordinatorLayout可以将滑动事件、布局、测量交给子控件中的Behavior。现在我们就来看看CoordinatorLayout下的布局实现。查看其`onLayout`方法。

```java
 @Override
    protected void onLayout(boolean changed, int l, int t, int r, int b) {
        final int layoutDirection = ViewCompat.getLayoutDirection(this);
        final int childCount = mDependencySortedChildren.size();
        for (int i = 0; i < childCount; i++) {
            final View child = mDependencySortedChildren.get(i);
            if (child.getVisibility() == GONE) {
                // If the child is GONE, skip...
                continue;
            }

            final LayoutParams lp = (LayoutParams) child.getLayoutParams();
            final Behavior behavior = lp.getBehavior();
            //获取子控件的Behavior方法，并调用其onLayoutChild方法判断子控件是否需要自己布局
            if (behavior == null || !behavior.onLayoutChild(this, child, layoutDirection)) {
                onLayoutChild(child, layoutDirection);
            }
        }
    }
```

从代码中我们可以看出，在对子 View 进行遍历的时候，CoordinatorLayout有主动向子控件的Behavior传递布局的要求，如果Behavior调用`onLayoutChild`了方法自主布局了子控件，则以它的结果为准，否则将调用`onLayoutChild`方法亲自布局。这里就不对CoordinatorLayout下的`onLayoutChild`方法进行过多的描述了，大家知道这个方法类似于FrameLayout的布局就行了。

#### Behavior的布局时机

其实肯定会有小伙伴会疑惑，什么样的情况下，我们需要设置自主布局呢?（也就是behavior.onLayoutChild()方法返回true)。在上文中我们说过了CoordinatorLayout布局方式是类似于FrameLayout的。在FrameLayout的布局中是只支持Gravity来设置布局的。如果我们需要自主的摆放控件中的位置，那么我们就需要重写Behavior的`onLayoutChild`方法。并设置该方法返回结果为true。

### Behavior的测量

最后一个知识点了！！！！！Behavior的测量。依然还是通过CoordinatorLayout传递过来的。我们查看CoordinatorLayout的`onMeasure`方法。代码如下所示：

```java
    @Override
    protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
        //省略部分代码....
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

上面的代码中，我还是省略了一些不重要的代码。观察上述代码，我们发现该方法与CoordinatorLayout的布局逻辑非常相似，也是对子控件进行遍历，并调那个用子控件的Behavior的`onMeasureChild`方法，判断是否自主测量，如果为true,那么则以子控件的测量为准。当子控件测量完毕后。会通过`widthUsed` 和 `heightUsed` 这两个变量来保存CoordinatorLayout中子控件最大的尺寸。这两个变量的值，最终将会影响CoordinatorLayout的宽高。

#### Behavior的测量时机

还是相似的问题，在什么样的情况下，我们需要重写Behavior`onMeasureChild`方法来自主测量控件呢？当你的控件需要重新设置位置的时候你要考虑是否需要重写该方法。什么意思呢？看下图所示：

{% asset_img  空白区域.jpg %}

在上图中我们定义了两个控件A与B，我们假设这两个控件处于这三个条件下：

- A、B控件都在CoordinatorLayout下，且A、B控件位置关系为控件A在B控件的下方。
- A控件的高度为`match_parent`。
- A、B控件的嵌套滑动关系为：B控件先处理嵌套滑动事件，当控件B向上滑动至隐藏后，控件A才能开始滑动。

那么根据上述条件，在滚动的过程中，我们会发现一个问题，就是当我们的控件A逐渐滑动到顶部时，我们会发现屏幕下方会出现一个**空白区域**，那原因是什么呢？其实很简单，当控件A高度为`match_parent`时，根据View的测量规则，控件A实际的高度就是整个控件剩余的高度（屏幕高度-控件B的高度），所以当控件B滚出屏幕后，那么就会出现一段空白。

那么为了使控件A在滑动过程中始终填充整个屏幕，我们需要在CoordinatorLayout测量该控件的高度之前，让控件自主的去测量高度，那么这个时候，Behavior的`onMeasureChild`方法就派上用场了。我们可以重写该方法并设定当前控件A的高度为整个屏幕的高度。当然如何通过Behavior的onMeasureChild重新设定控件的高度是我们后续文章将讲解的知识，大家如果有兴趣的话，可以关注后续文章。

### 总结

看到这里的小伙伴真的非常值得鼓励。点赞！！！！！关于CoordinatorLayout的整个下的Behavior确实理解起来需要花费不少的时间。我本人从理解到写这篇博客零零散散也花费了两周多的时间。虽然说这块知识点比较偏门。但是还是希望能帮助到有需要的小伙伴们。能有幸帮助到大家，我也非常开心了。
