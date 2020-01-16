---
title: Androidx 下 Fragment 懒加载的新方式
tags:
  - 懒加载
categories:
  - Fragment
---

### 前言

>年后最后一篇文章啦，在这里先祝大家新年快乐~最重要的抽中`全家福`，明年继续修福报🤣

接下来，为大家带来一篇充满的干货的文章，直入主题，以前我们处理 Fragment 的懒加载，我们通常会处理 `setUserVisibleHint + onHiddenChanged` 这两个函数，本篇文章将介绍如何在 Androidx 模式下使用 `setMaxLifecycle` 函数的方式来处理 Fragment 的懒加载，相信这种`新`的思路与方法会直接让大家高潮!!!!!😎。不服来战~我祖安人没怕过

在本文章中，我会详细介绍懒加载的相关知识，以及不同使用场景下这两种方案的差异。大家快拿好小板凳。一起来学习新知识吧！

>如果你熟悉老一套的Fragment懒加载机制，你可以直接查看 Androix 懒加载相关章节

### 懒加载是什么

### 为什么要使用懒加载

### 之前的处理方案

#### add+show+hide 模式下的老方案

在没有添加懒加载之前，只要使用 `add+show+hide` 的方式控制并显示 Fragment, 那么不管是否嵌套，在初始化时`（只调用了add+show）`，同级下的 Fragment 的相关生命周期函数都会被调用。且调用的生命周期函数如下所示：

`onAttach -> onCreate -> onCreatedView -> onActivityCreated -> onStart -> onResume`

>Fragment声明周期回顾：onAttach -> onCreate -> onCreatedView -> onActivityCreated -> onStart -> onResume -> onPause -> onStop -> onDestroyView -> onDestroy -> onDetach

什么是同级 Frament 呢？看下图

![同级Fragment.jpg](https://upload-images.jianshu.io/upload_images/2824145-7227e891da1318e8.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

>上图中，都是使用 `add+show+hide` 的方式控制 Fragment,

在上图两种模式中:

- Fragment_1、Fragment_2、Fragment_3 属于同级 Fragment
- Fragment_a、Fragment_b、Fragment_c 属于同级 Fragment
- Fragment_d、Fragment_e、Fragment_f 属于同级 Fragment

那这种方式会带来什么问题呢？结合上图我们来分别分析。

- 非嵌套模式

  因为同级的 Fragment 生命周期函数会被调用，所以，如果我们没有对 Fragment 进行懒加载处理，那么我们就会无缘无故的加载不必要的 Fragment , 也就会加载一些额外的数据（我们会在 Fragment 相关生命周期函数中，请求网络或其他数据操作）。

- 嵌套模式
  
  嵌套模式下更为严重了，结合上图，当 Fragment_1 加载时，如果你在 Fragment_1 生命周期函数中使用 `show+add+hide` 的方式添加 Fragment_a、Fragment_b、Fragment_c , 那么 Fragment_b 又会在其生命周期函数中继续加载 Fragment_d、Fragment_e、Fragment_f，所以又有很多的数据被无缘无故的加载了。

虽然不可见的 Fragment 生命周期函数会被调用，但是对与不可见的 Fragment， 其布尔属性 `mHidden` 始终为 `false` ，只有通过 `FragmentManager` 调用 `show()` 函数时，该属性的值才会发生变化。也就是我们可以在 `onResume()` 函数中通过 `isHidden()` 函数来获取该属性的值，来判断 Fragment 是否可见。

>`"不可见的Fragment"`，这里是指：实际不可见，但是相关可见生命周期函数(例如 `onResume` 方法）被调用的 Fragment

又因为当通过 `show+hide` 时，Fragment 任何的生命周期方法都不会调用，只有`onHiddenChanged` 方法会被调用，所以如果我们想在 `add+show+hide` 模式下控制 Fragment 的懒加载，我们需要这样处理:

```kotlin
class Fragment:Fragment(){

    private var isLoaded = false //控制是否执行懒加载

    override fun onResume() {
        super.onResume()
        judgeLazyInit()

    }
    override fun onHiddenChanged(hidden: Boolean) {
        super.onHiddenChanged(hidden)
        isVisibleToUser = !hidden
        judgeLazyInit()
    }

    private fun judgeLazyInit() {
        if (!isLoaded && !isHidden && isCallResume) {
            lazyInit()
            isLoaded = true
        }
    }
}
```

>上述例子是在 `onResume` 方法中进行懒加载操作，当然你可以在其他生命周期函数中控制。

#### ViewPager+Fragment 模式下的老方案

>没有设置 pageLimit

使用 FragmentPagerAdapter+ViewPager 时，切换回上一个Fragment页面时（已经初始化完毕），不会回调任何生命周期方法以及onHiddenChanged()，只有setUserVisibleHint(boolean isVisibleToUser)会被回调，所以如果你想进行一些懒加载，需要在这里处理。

setUserVisibleHint 方法会优先于声明周期的方法调用：

```log
2020-01-16 16:19:39.793 5503-5503/com.jennifer.andy.androidxlazyload D/OneFragment: setUserVisibleHint:isVisibleToUser-->false
2020-01-16 16:19:39.793 5503-5503/com.jennifer.andy.androidxlazyload D/TwoFragment: setUserVisibleHint:isVisibleToUser-->false
2020-01-16 16:19:39.793 5503-5503/com.jennifer.andy.androidxlazyload D/OneFragment: setUserVisibleHint:isVisibleToUser-->true
2020-01-16 16:19:39.794 5503-5503/com.jennifer.andy.androidxlazyload D/OneFragment: onAttach:
2020-01-16 16:19:39.794 5503-5503/com.jennifer.andy.androidxlazyload D/OneFragment: onCreate:
2020-01-16 16:19:39.795 5503-5503/com.jennifer.andy.androidxlazyload D/TwoFragment: onAttach:
2020-01-16 16:19:39.795 5503-5503/com.jennifer.andy.androidxlazyload D/TwoFragment: onCreate:
2020-01-16 16:19:39.799 5503-5503/com.jennifer.andy.androidxlazyload D/OneFragment: onViewCreated:
2020-01-16 16:19:39.799 5503-5503/com.jennifer.andy.androidxlazyload D/OneFragment: onActivityCreated:
2020-01-16 16:19:39.799 5503-5503/com.jennifer.andy.androidxlazyload D/OneFragment: onStart:
2020-01-16 16:19:39.800 5503-5503/com.jennifer.andy.androidxlazyload D/OneFragment: onResume:
2020-01-16 16:19:39.802 5503-5503/com.jennifer.andy.androidxlazyload D/TwoFragment: onViewCreated:
2020-01-16 16:19:39.802 5503-5503/com.jennifer.andy.androidxlazyload D/TwoFragment: onActivityCreated:
2020-01-16 16:19:39.803 5503-5503/com.jennifer.andy.androidxlazyload D/TwoFragment: onStart:
2020-01-16 16:19:39.803 5503-5503/com.jennifer.andy.androidxlazyload D/TwoFragment: onResume:
```

当我们直接点击 TAB 切换到 FragmentThree 时，FragmentOne 会走销毁流程。

```log
2020-01-16 16:46:09.763 6344-6344/com.jennifer.andy.androidxlazyload D/ThreeFragment: setUserVisibleHint:isVisibleToUser-->false
2020-01-16 16:46:09.763 6344-6344/com.jennifer.andy.androidxlazyload D/OneFragment: setUserVisibleHint:isVisibleToUser-->false
2020-01-16 16:46:09.764 6344-6344/com.jennifer.andy.androidxlazyload D/ThreeFragment: setUserVisibleHint:isVisibleToUser-->true
2020-01-16 16:46:09.764 6344-6344/com.jennifer.andy.androidxlazyload D/ThreeFragment: lazyInit:!!!!!!!
2020-01-16 16:46:09.764 6344-6344/com.jennifer.andy.androidxlazyload D/ThreeFragment: onAttach: 
2020-01-16 16:46:09.764 6344-6344/com.jennifer.andy.androidxlazyload D/ThreeFragment: onCreate: 
2020-01-16 16:46:09.767 6344-6344/com.jennifer.andy.androidxlazyload D/ThreeFragment: onViewCreated: 
2020-01-16 16:46:09.768 6344-6344/com.jennifer.andy.androidxlazyload D/ThreeFragment: onActivityCreated: 
2020-01-16 16:46:09.768 6344-6344/com.jennifer.andy.androidxlazyload D/ThreeFragment: onStart: 
2020-01-16 16:46:09.768 6344-6344/com.jennifer.andy.androidxlazyload D/ThreeFragment: onResume: 
2020-01-16 16:46:10.114 6344-6344/com.jennifer.andy.androidxlazyload D/OneFragment: onStop: 
2020-01-16 16:46:10.114 6344-6344/com.jennifer.andy.androidxlazyload D/OneFragment: onDestroyView: 
```

当我们直接点击 Fragment_one tab 的时候，FragmentOne 会先调用，setUserVisibleHint方法。所以我们之前的懒加载会有错误，

```log
2020-01-16 16:48:06.242 6344-6344/com.jennifer.andy.androidxlazyload D/OneFragment: setUserVisibleHint:isVisibleToUser-->false
2020-01-16 16:48:06.242 6344-6344/com.jennifer.andy.androidxlazyload D/ThreeFragment: setUserVisibleHint:isVisibleToUser-->false
2020-01-16 16:48:06.242 6344-6344/com.jennifer.andy.androidxlazyload D/OneFragment: setUserVisibleHint:isVisibleToUser-->true
2020-01-16 16:48:06.245 6344-6344/com.jennifer.andy.androidxlazyload D/OneFragment: onViewCreated: 
2020-01-16 16:48:06.245 6344-6344/com.jennifer.andy.androidxlazyload D/OneFragment: onActivityCreated: 
2020-01-16 16:48:06.246 6344-6344/com.jennifer.andy.androidxlazyload D/OneFragment: onStart: 
2020-01-16 16:48:06.246 6344-6344/com.jennifer.andy.androidxlazyload D/OneFragment: onResume: 
2020-01-16 16:48:06.246 6344-6344/com.jennifer.andy.androidxlazyload D/OneFragment: lazyInit:!!!!!!!
2020-01-16 16:48:06.590 6344-6344/com.jennifer.andy.androidxlazyload D/ThreeFragment: onStop: 
2020-01-16 16:48:06.590 6344-6344/com.jennifer.andy.androidxlazyload D/ThreeFragment: onDestroyView: 
```

所以我们完整版的代码如下：

#### 复杂 Fragment 嵌套的情况

基本上所有场景都能覆盖到，结合上面的所有场景，ViewPager+Fragem

列出所有的组合场景。因为我们对这上述两种模式都进行了懒加载的处理方案，所以在你的界面不管你怎么组合这两个模式，都能保证实际对用户可见的Fragment执行了懒加载。

### Androidx 下的处理方案

最大的生命周期的控制，只管当前Fragment是否可以和用户进行交互，也就是只要你Fragment没有办法和用户进行交互，那就不要执行onResume方法了。你他妈又不能和用户进行交互，你执行onResume方法干嘛？

#### ViewPager+Fragment模式下的方案

为什么先讲解ViewPager呢？

#### add+show+hide模式下的新方案

##### 非嵌套模式下是正常的

##### 嵌套模式下

对于Fragment的嵌套，及时使用了setMax方法。同级不可见的Fragment， 第一次仍然TMD任然要调用可见生命周期方法，

#### ViewPager2的处理方案

即使使用ViewPager2也只是对一级界面进行懒加载，如果ViewPager2中涉及到Fragment的嵌套，仍然需要处理嵌套Fragment的逻辑。


### 这两种方式的对比


#### 老一套的懒加载

1.优点：不用去控制 FragmentManager的 add+show+hide 方法，所有的懒加载都是在Fragment内部控制，也就是 `setUserVisibleHint + onHiddenChanged` 这两个函数来控制。理解上来说要相对容易。

2.缺点：实际不可见的Fragment,但是其可见生命周期函数任然会被调用，这种反常规的逻辑，无法容忍。

#### 新一套的懒加载

1.优点：`在非特殊的情况下(缺点）`，只有实际的可见Fragment，其可见生命周期方法才会被调用，这些是符合方法设计初衷。

2.缺点：对于Fragment的嵌套，及时使用了setMax方法。同级不可见的Fragment， TMD任然要调用可见生命周期方法


### 更多的详细例子

这里要写项目的ReadMe，然后创建传统模式下的懒加载的LazyFragment. 提示用户怎么去修改代码到传统模式。

### 最后

https://juejin.im/post/5adcb0e36fb9a07aa7673fbc

https://www.jianshu.com/p/fd71d65f0ec6

站在巨人的肩膀上，才能看的更远~
