---
title: Jetpack最佳实践
tags:
  - null
categories:
  - null
---

### 前言

在了解JetPack之前需要了解MVP,MVVM

- liveData  https://v.qq.com/x/page/b07414b9djz.html
- ViewMode1 https://v.qq.com/x/page/t0763s9ma8o.html
- ViewMode2  https://v.qq.com/x/page/m0605c1sejh.html
- WorkManager 参考视屏： https://v.qq.com/x/page/r088792co6n.html
- Navigation 导航组件：https://v.qq.com/x/page/v0879xupgo0.html
- Pageing   分页库：https://v.qq.com/x/page/r077736iasg.html
- Room  https://v.qq.com/x/page/d0770r409g3.html


Google官方例子：https://github.com/android/architecture-components-samples
Google官方文档：https://developer.android.google.cn/topic/libraries/architecture

Google官方架构：https://github.com/android/architecture-samples/tree/dev-todo-mvvm-rxjava
切分支来看：

GoogleJetPack架构指南：
https://developer.android.google.cn/jetpack/docs/guide

### ViewModel使用注意的事项

优点1：可以使用Viewmodle 共享Fragments数据
优点2：ViewModel可以根据另一个生命组件使用。LiveData，来创建响应式的布局。

#### 使用注意事项

不需要传入Context,会导致内存泄漏
如果需要传入Context 继承还有ApplicationContext的AndroidViewModel
ViewModel不可以替代OnSaveInstanceState.（https://developer.android.google.cn/topic/libraries/architecture/saving-states）


如果我们的应用需要大量的数据，那么推荐创建一个Repository类作为唯一的数据层入口
同时我们也要注意不要重蹈Activity的覆辙，避免在ViewModel内中实现更多的职责，创建一个Presenter类来处理UI界面数据。

#### ViewModel配合 OnSaveInstanceState 来使用


要获取 user，我们的 ViewModel 需要访问 Fragment 参数。我们可以通过 Fragment 传递它们，或者更好的办法是使用 SavedState 模块，我们可以让 ViewModel 直接读取参数：

注意：SavedStateHandle(https://developer.android.google.cn/topic/libraries/architecture/viewmodel-savedstate#kotlin) 允许 ViewModel 访问相关 Fragment 或 Activity 的已保存状态和参数。

```kotlin
// UserProfileViewModel
    class UserProfileViewModel(
       savedStateHandle: SavedStateHandle
    ) : ViewModel() {
       val userId : String = savedStateHandle["uid"] ?:
              throw IllegalArgumentException("missing user id")
       val user : User = TODO()
    }

    // UserProfileFragment
    private val viewModel: UserProfileViewModel by viewModels(
       factoryProducer = { SavedStateVMFactory(this) }
       ...
    )
```


### 最后

站在巨人的肩膀上，才能看的更远~
