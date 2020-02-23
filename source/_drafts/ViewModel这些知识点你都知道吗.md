---
title: ViewModel这些知识点你都知道吗
tags:
  - ViewModel
categories:
  - Jetpack
---

### 前言

[ViewModel](https://developer.android.google.cn/topic/libraries/architecture/viewmodel) 作为 [Jetpack](https://developer.android.google.cn/jetpack) 中的明星组件，相信大家都对其有一定的了解。在 Google 的官方介绍中也详细的罗列了 ViewModel 的优点，如：

1. 可以提供和管理UI界面数据。(将加载数据与数据恢复从Activity or Fragment中解耦)
2. 可感知生命周期的组件。
3. 不会因配置改变而销毁。
4. 可以配合LiveData使用。
5. 多个Fragment可以共享同一Model
6. 等等等....

你也可以通过下列两个视频，更为详细的了解 ViewModel：

- [ViewMode男生讲解版](https://v.qq.com/x/page/t0763s9ma8o.html)
- [ViewMode女生讲解版](https://v.qq.com/x/page/m0605c1sejh.html)

在本篇文章中，不会讲解 ViewModel 的使用方式及使用 ViewModel 的原因，而是着重于讲解 ViewModel 的原理及额外注意事项。通过阅读本篇文章你能了解到：

- ViewModel 与 Activity的绑定过程
- 常见的数据恢复的方式
- ViewModel 在 Activity 中不会因配置改变而销毁的原理及流程。
- ViewModel 如何与 OnSaveInstanceState 配合使用。
- ViewModel 在 Fragment 中不会因配置改变而销毁的原理及流程。
- ViewModel 如何做到共享的。
- ViewModel 中使用网络请求时需要注意的事项。
  
希望通过该篇文章，大家能对 ViewModel 有更深入的了解。

### ViewModel 与 Activity的绑定过程

一般情况下，使用 ViewModel，我们一般会先声明自己的 ViewModel，并在 Activity 中的 `onCreate` 方法中，通过 `ViewModelProviders` 来创建 ViewModel。 如下代码所示：

>在谷歌的最新代码中，不推荐使用 ViweModelProvider`s` ，而是直接使用  `ViewModelProvider` 的构造函数来创建 `ViewModelProvider` 对象。

```java
public class MyViewModel extends ViewModel {
    private MutableLiveData<List<User>> users;
    public LiveData<List<User>> getUsers() {
        if (users == null) {
            users = new MutableLiveData<List<User>>();
            loadUsers();
        }
        return users;
    }

    private void loadUsers() {
        //执行异步操作操作获取用户信息
    }
}

public class MyActivity extends AppCompatActivity {
    public void onCreate(Bundle savedInstanceState) {
        //系统第一次调用Activity的onCreate()方法时创建ViewModel。
        //重新创建的Activity接收由第一个Activity创建的相同MyViewModel实例。
        MyViewModel model = ViewModelProviders.of(this).get(MyViewModel.class);
        model.getUsers().observe(this, users -> {
            // 更新ui
        });
    }
}
```

观察上述代码，我们能发现 ViewModel 的创建过程其实是与 `ViewModelProviders` 类相关的，我们查看该类到底做了什么。查看其 `of（）` 方法，代码如下所示：

```java
   public static ViewModelProvider of(@NonNull FragmentActivity activity) {
        return new ViewModelProvider(activity);
    }
```

在该方法中，其实是就是创建了 `ViewModelProvider` 对象，那么也就是说最终 ViewModel 的创建是与该对象相关的。查看该类构造函数：

```java
    public ViewModelProvider(@NonNull ViewModelStoreOwner owner) {
        this(owner.getViewModelStore(), owner instanceof HasDefaultViewModelProviderFactory
                ? ((HasDefaultViewModelProviderFactory) owner).getDefaultViewModelProviderFactory()
                : NewInstanceFactory.getInstance());
    }

    public ViewModelProvider(@NonNull ViewModelStoreOwner owner, @NonNull Factory factory) {
        this(owner.getViewModelStore(), factory);
    }

    public ViewModelProvider(@NonNull ViewModelStore store, @NonNull Factory factory) {
        mFactory = factory;
        mViewModelStore = store;
    }
```

观察上述构造函数，我们可以发现 ViewModelProvider 所需要的对象就两个，一个 `ViewModelStore` ，一个 `Factory`，且遵循如下规则：

- 规则1：当没有传递 Factory 对象时，如果当前类实现了 HasDefaultViewModelProviderFactory 接口 ，默认会调用 getDefaultViewModelProviderFactory()。
- 规则2：当规则2条件不满足时，默认会调用 ViewModelProvider 中的 NewInstanceFactory.getInstance()方法创建 Factory 对象。

NewInstanceFactory 声明如下:

```java
    public static class NewInstanceFactory implements Factory {

        private static NewInstanceFactory sInstance;

        /**
         * Retrieve a singleton instance of NewInstanceFactory.
         *
         * @return A valid {@link NewInstanceFactory}
         */
        @NonNull
        static NewInstanceFactory getInstance() {
            if (sInstance == null) {
                sInstance = new NewInstanceFactory();
            }
            return sInstance;
        }

        @SuppressWarnings("ClassNewInstance")
        @NonNull
        @Override
        public <T extends ViewModel> T create(@NonNull Class<T> modelClass) {
            //noinspection TryWithIdenticalCatches
            try {
                return modelClass.newInstance();
            } catch (InstantiationException e) {
                throw new RuntimeException("Cannot create an instance of " + modelClass, e);
            } catch (IllegalAccessException e) {
                throw new RuntimeException("Cannot create an instance of " + modelClass, e);
            }
        }
    }
```

NewInstanceFactory 中的方法也非常简单，也就是通过 create 方法，获取传入的 ViewModel 的 Class 对象，并通过反射创建该model对象。注意！！！**调用的是其无参的构造函数。**

查看了 NewInstanceFactory 类，我们再把思路聚焦在 ViewModelStore 上，通过观察 ViewModelProvider 的构造函数，我们能发现 ViewModelStore 的内容是通过 `ViewModelStoreOwner` 接口的实现类调用 getViewModelStore()方法获取。我们接着查看 `ViewModelStore` 类:

>注意：androix 包下的 Activity 与 Fragment 都默认实现了 ViewModelStoreOwner 接口。

```java
public class ViewModelStore {

    private final HashMap<String, ViewModel> mMap = new HashMap<>();

    final void put(String key, ViewModel viewModel) {
        ViewModel oldViewModel = mMap.put(key, viewModel);
        if (oldViewModel != null) {
            oldViewModel.onCleared();
        }
    }

    final ViewModel get(String key) {
        return mMap.get(key);
    }

    Set<String> keys() {
        return new HashSet<>(mMap.keySet());
    }

    /**
     * 当内部的 ViewModel 不再使用时，清除所占的内存
     */
    public final void clear() {
        for (ViewModel vm : mMap.values()) {
            //下面调用ViewModel的clear方法
            vm.clear();
        }
        mMap.clear();
    }
}
```

观察代码我们发现 `ViewModelStore` 内部其实维护了一个可存入 ViewModel 的 HashMap, 也就是说 ViewModeStore 只是一个存储的 ViewModel 的容器，那我们再回到 `ViewModelProviders.of(this).get(MyViewModel.class);` 那段代码，最终的 ViewModel 的创建其实是通过 ViewModelProvider 的 get 方法，我们查看该方法实现：

```java
 public <T extends ViewModel> T get(@NonNull Class<T> modelClass) {
        String canonicalName = modelClass.getCanonicalName();
        if (canonicalName == null) {
            throw new IllegalArgumentException("Local and anonymous classes can not be ViewModels");
        }
        return get(DEFAULT_KEY + ":" + canonicalName, modelClass);
    }
```

该方法会将传入的 class 对象的**底层类规范名称**作为 `key`，接着并调用含有两个参数的 get 方法，继续跟踪：

```java
 public <T extends ViewModel> T get(@NonNull String key, @NonNull Class<T> modelClass) {
        //👇根据key值从ViewModelStore中取对应的ViewModel
        ViewModel viewModel = mViewModelStore.get(key);
        if (modelClass.isInstance(viewModel)) {
            if (mFactory instanceof OnRequeryFactory) {
                ((OnRequeryFactory) mFactory).onRequery(viewModel);
            }
            return (T) viewModel;
        } else {
            //noinspection StatementWithEmptyBody
            if (viewModel != null) {
                // TODO: log a warning.
            }
        }
        //👇如果没有获取到已有的 ViewModel 根据传入的Factory创建新的 VideModel
        if (mFactory instanceof KeyedFactory) {
            viewModel = ((KeyedFactory) (mFactory)).create(key, modelClass);
        } else {
            viewModel = (mFactory).create(modelClass);
        }
        //👇将新的 ViewModel 存入ViewModelStore，并返回
        mViewModelStore.put(key, viewModel);
        return (T) viewModel;
    }

```

观察代码，我们能发现该方法主要是在 ViewModelStore 中判断传入的 ViewModel 是否存在，如果存在，则直接返回。反之，根据传入的Facory的create方法，创建ViewModel，并将创建好的 ViewModel 放入 ViewModelStore中。

那么结合所有的流程，我们能得到下图：

![Activity下ViewModel的创建过程.png](https://upload-images.jianshu.io/upload_images/2824145-fecc9582d2892c82.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### ViewModel 如何做到不会因为配置改变而销毁

我们都知道 ViewModel 不会因为 Activity 的配置发生改变而销毁，也就是如下所示：

![viewmodel-lifecycle.png](https://upload-images.jianshu.io/upload_images/2824145-10c47be88330a248.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

观察上图，我相信小伙伴们肯定有如下疑惑：

- 当 Activity 因为配置发生改变时，会重新创建一个新的Activity。那老的Activity中的ViewModel是如何传递给新的Activity的呢？
- ViewModel 又是如何感知配置是否改变，进而判断是否销毁的呢？

那下面我们就着手来解决这些问题吧


#### 恢复数据的几种方式

资源相关的配置发生改变导致Activity被杀死并重新创建时
资源内存不足导致低优先及的Activity被杀死

ViewModel是一个可感知生命周期的组件，这意味着因为Activity配置发生改变的是后，我们会重新创建Fragment与Activity,那之前的数据是怎么保存的呢。

用户期望 Activity 的界面状态在整个配置变更（例如旋转或切换到多窗口模式）期间保持不变。然而，发生此类配置变更时，系统会默认销毁 Activity，从而消除存储在 Activity 实例中的所有界面状态。同样，如果用户暂时从您的应用切换到其他应用，并在稍后返回您的应用，他们也希望界面状态保持不变。但是，当用户离开应用且您的 Activity 停止时，系统可能会销毁您应用的进程。

当 Activity 因系统限制遭到销毁时，您应组合使用 ViewModel、onSaveInstanceState() 和/或本地存储来保留用户的界面瞬态。如要详细了解用户期望与系统行为，以及如何在系统启动的 Activity 和进程遭到终止后最大程度地保留复杂的界面状态数据，请参阅保存界面状态。

此部分将概述实例状态的定义，以及如何实现 onSaveInstance() 方法，该方法是对 Activity 本身的回调。如果界面数据简单且轻量，例如原始数据类型或简单对象（比如 String），则您可以单独使用 onSaveInstanceState() 使界面状态在配置更改和系统启动的进程遭到终止时保持不变。但在大多数情况下，您应使用 ViewModel 和 onSaveInstanceState()（如保存界面状态中所述），因为 onSaveInstanceState() 会产生序列化或反序列化费用。



当 Activity 配置发生改变且走 onDestory 方法时，是不会清除所有的 ViewModel的，我们都知道在 Activity 配置发生改变时（比如选择屏幕），并且没有在清单文件中配置 `android:configChanges` 时，系统会杀死老的Activity并创建新的Activity。那现在有一个问题了，既然是新的 Activity，那老的Activity 中的 ViewModel 是如何传递给新的 Activity 呢？

> 在清单文件中配置 `android:configChanges` 参数，可以设置 Activity 在对应配置发生改变时，不会被重新创建，取而代之的是系统将调用 Activity 的 `onConfigurationChanged()` 方法。

https://developer.android.google.cn/topic/libraries/architecture/saving-states.html

##### 使用 onSaveInstanceState 与 onRestoreInstanceState

一个数据列表，如果保证在配置发生改变的时候，不会重复的加载数据呢？

这个数据的恢复和我们界面的数据恢复是不一样的。界面数据的保存，系统的工作流程是这样的。首先Activity会调用
`onSaveInstanceState`去保存数据，然后Activity会委托Window去保存数据，接着Window在委托它上面的顶级容去保存数据。顶层容器是一个ViewGroup，最后顶层容器再去通知它的子元素来保存数据。

- 重写configChange，让Activity不重复调用加载数据方法
- 根据生命周期，冲走一次逻辑。缺点重复加载。
- 重写`onSaveInstanceState`与`onRestoreInstanceState`这两个方法。但是存储的数据大小有限制。

应用的某个 Activity 中可能包含用户列表。因配置更改而重新创建 Activity 后，新 Activity 必须重新提取用户列表。对于简单的数据，Activity 可以使用 onSaveInstanceState() 方法从 onCreate() 中的捆绑包恢复其数据，但此方法仅适合可以序列化再反序列化的少量数据，而不适合数量可能较大的数据，如用户列表或位图。

可能让用户重新请求网络数据，或者重新查询数据库。


用户期望 Activity 的界面状态在整个配置变更（例如旋转或切换到多窗口模式）期间保持不变。然而，发生此类配置变更时，系统会默认销毁 Activity，从而消除存储在 Activity 实例中的所有界面状态。同样，如果用户暂时从您的应用切换到其他应用，并在稍后返回您的应用，他们也希望界面状态保持不变。但是，当用户离开应用且您的 Activity 停止时，系统可能会销毁您应用的进程。

当 Activity 因系统限制遭到销毁时，您应组合使用 ViewModel、onSaveInstanceState() 和/或本地存储来保留用户的界面瞬态。如要详细了解用户期望与系统行为，以及如何在系统启动的 Activity 和进程遭到终止后最大程度地保留复杂的界面状态数据，请参阅保存界面状态。

此部分将概述实例状态的定义，以及如何实现 onSaveInstance() 方法，该方法是对 Activity 本身的回调。如果界面数据简单且轻量，例如原始数据类型或简单对象（比如 String），则您可以单独使用 onSaveInstanceState() 使界面状态在配置更改和系统启动的进程遭到终止时保持不变。但在大多数情况下，您应使用 ViewModel 和 onSaveInstanceState()（如保存界面状态中所述），因为 onSaveInstanceState() 会产生序列化或反序列化费用。

##### 使用 Fragment 的 setRetainInstance

https://www.androiddesignpatterns.com/2013/04/retaining-objects-across-config-changes.html

##### 使用 getLastNonConfigurationInstance 与 onRetainNonConfigurationInstance

```java
    public ViewModelStore getViewModelStore() {
        if (getApplication() == null) {
            throw new IllegalStateException("Your activity is not yet attached to the "
                    + "Application instance. You can't request ViewModel before onCreate call.");
        }
        if (mViewModelStore == null) {
            NonConfigurationInstances nc =
                    (NonConfigurationInstances) getLastNonConfigurationInstance();
            if (nc != null) {
                // Restore the ViewModelStore from NonConfigurationInstances
                mViewModelStore = nc.viewModelStore;
            }
            if (mViewModelStore == null) {
                mViewModelStore = new ViewModelStore();
            }
        }
        return mViewModelStore;
    }
```

```java
 public Object getLastNonConfigurationInstance() {
        return mLastNonConfigurationInstances != null
                ? mLastNonConfigurationInstances.activity : null;
    }

```

```java
    public final Object onRetainNonConfigurationInstance() {
        Object custom = onRetainCustomNonConfigurationInstance();

        ViewModelStore viewModelStore = mViewModelStore;
        if (viewModelStore == null) {
            // No one called getViewModelStore(), so see if there was an existing
            // ViewModelStore from our last NonConfigurationInstance
            NonConfigurationInstances nc =
                    (NonConfigurationInstances) getLastNonConfigurationInstance();
            if (nc != null) {
                viewModelStore = nc.viewModelStore;
            }
        }

        if (viewModelStore == null && custom == null) {
            return null;
        }

        NonConfigurationInstances nci = new NonConfigurationInstances();
        nci.custom = custom;
        nci.viewModelStore = viewModelStore;
        return nci;
    }
```

### Activity ViewModel 如何判断是否移除

在 androidx.actvity ComponentActivity 中的构造函数

```java
    public ComponentActivity() {
        Lifecycle lifecycle = getLifecycle();
        //noinspection ConstantConditions
        if (lifecycle == null) {
            throw new IllegalStateException("getLifecycle() returned null in ComponentActivity's "
                    + "constructor. Please make sure you are lazily constructing your Lifecycle "
                    + "in the first call to getLifecycle() rather than relying on field "
                    + "initialization.");
        }
        if (Build.VERSION.SDK_INT >= 19) {
            getLifecycle().addObserver(new LifecycleEventObserver() {
                @Override
                public void onStateChanged(@NonNull LifecycleOwner source,
                        @NonNull Lifecycle.Event event) {
                    if (event == Lifecycle.Event.ON_STOP) {
                        Window window = getWindow();
                        final View decor = window != null ? window.peekDecorView() : null;
                        if (decor != null) {
                            decor.cancelPendingInputEvents();
                        }
                    }
                }
            });
        }
        getLifecycle().addObserver(new LifecycleEventObserver() {
            @Override
            public void onStateChanged(@NonNull LifecycleOwner source,
                    @NonNull Lifecycle.Event event) {
                if (event == Lifecycle.Event.ON_DESTROY) {
                    if (!isChangingConfigurations()) {
                       //👇在配置没发生改变且走到onDestory方法时，清除所有的ViewModel
                        getViewModelStore().clear();
                    }
                }
            }
        });

        if (19 <= SDK_INT && SDK_INT <= 23) {
            getLifecycle().addObserver(new ImmLeaksCleaner(this));
        }
    }
```




#### ViewModel需要配合 OnSaveInstanceState 来使用


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

#### 第一步将FragmentManagerViewModel存入Activity中的ViewModelStore中

在 FragmentActivity 中的onCreate方法中

```java
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        mFragments.attachHost(null /*parent*/);
        //省略更多...
    }
```

注意这里获取的是 Activity 中的 FragmentManager

```java
 void attachController(@NonNull FragmentHostCallback<?> host,
            @NonNull FragmentContainer container, @Nullable final Fragment parent) {
        //省略更多...
        // Get the FragmentManagerViewModel
        if (parent != null) {
            mNonConfig = parent.mFragmentManager.getChildNonConfig(parent);
        } else if (host instanceof ViewModelStoreOwner) {
            //👇第一次因为传入的是 Activity 故会走这里
            ViewModelStore viewModelStore = ((ViewModelStoreOwner) host).getViewModelStore();
            mNonConfig = FragmentManagerViewModel.getInstance(viewModelStore);
        } else {
            mNonConfig = new FragmentManagerViewModel(false);
        }
    }
```

因为这里传入的activity，也就是我们拿取的是Activity中的 ViewModelStore，接着当我们在Activity添加Fragment时,默认会走Fragment的声明周期，也就是如下代码所示：

```java
    void performAttach() {
        //👇又会调用attachController
        mChildFragmentManager.attachController(mHost, new FragmentContainer() {
            @Override
            @Nullable
            public View onFindViewById(int id) {
                if (mView == null) {
                    throw new IllegalStateException("Fragment " + this + " does not have a view");
                }
                return mView.findViewById(id);
            }

            @Override
            public boolean onHasView() {
                return (mView != null);
            }
        }, this);//👈注意这里的this传入的parent是当前Fragment
      //省略更多...
    }
```

我们继续回到 FragmentManager 中的 attachController

```java
 void attachController(@NonNull FragmentHostCallback<?> host,
            @NonNull FragmentContainer container, @Nullable final Fragment parent) {
        //省略更多...
        // Get the FragmentManagerViewModel
        if (parent != null) {//👈因为parent为this,故我们会获取当前Fragment中的FragmentManager
            mNonConfig = parent.mFragmentManager.getChildNonConfig(parent);
        } else if (host instanceof ViewModelStoreOwner) {
            ViewModelStore viewModelStore = ((ViewModelStoreOwner) host).getViewModelStore();
            mNonConfig = FragmentManagerViewModel.getInstance(a);
        } else {
            mNonConfig = new FragmentManagerViewModel(false);
        }
    }
```

>需要注意的是，当我们Fragment是其他Fragment的子Fragment时，获取的fragmentManager是，childFragmentManager,否则只Activity的FragmentManager。

在Activity 中的 FragmentManager中的 FragmentManagerViewModel 中创建 Fragment 的 FragmentManagerViewModel

```java
  FragmentManagerViewModel getChildNonConfig(@NonNull Fragment f) {
        FragmentManagerViewModel childNonConfig = mChildNonConfigs.get(f.mWho);
        if (childNonConfig == null) {
            childNonConfig = new FragmentManagerViewModel(mStateAutomaticallySaved);
            mChildNonConfigs.put(f.mWho, childNonConfig);
        }
        return childNonConfig;
    }
```

### 第二步创建ViewModelStore并存入父类的FragmentManaerViewModel中的mViewModelStores中

在Fragment创建ViewModel时，会为每个Fragment创建单独的ViewModelStore

```java
 public static ViewModelProvider of(@NonNull Fragment fragment) {
        return new ViewModelProvider(fragment);
    }
```

```java
 public ViewModelProvider(@NonNull ViewModelStoreOwner owner) {
        this(owner.getViewModelStore(), owner instanceof HasDefaultViewModelProviderFactory
                ? ((HasDefaultViewModelProviderFactory) owner).getDefaultViewModelProviderFactory()
                : NewInstanceFactory.getInstance());
    }
```

Fragment 下的 getViewModelStore() 实现：

```java
    public ViewModelStore getViewModelStore() {
        if (mFragmentManager == null) {
            throw new IllegalStateException("Can't access ViewModels from detached fragment");
        }
        return mFragmentManager.getViewModelStore(this);
    }
```

```java
FragmentManager mFragmentManager;
```

这里，可以介绍一下 FragmentManager :https://www.jianshu.com/p/fd71d65f0ec6

最终会走到FragmentManagerViewModel中的getViewModelStore 方法。

```java
  ViewModelStore getViewModelStore(@NonNull Fragment f) {
        ViewModelStore viewModelStore = mViewModelStores.get(f.mWho);
        if (viewModelStore == null) {
            viewModelStore = new ViewModelStore();
            mViewModelStores.put(f.mWho, viewModelStore);
        }
        return viewModelStore;
    }
     String mWho = UUID.randomUUID().toString();//这里的id获取
```

也就是在父类的FragmentManagerViewModel中，创建ViewModelStore，并存入那个mViewModelStores中

### ViewModel 使用范围

只要您的应用安装在用户的设备上，持续性本地存储（例如数据库或共享偏好设置）就会继续存在（除非用户清除应用的数据）。虽然此类本地存储空间会在系统启动的活动和应用进程终止后继续存在，但由于必须从本地存储空间读取到内存，因此检索成本高昂。这种持久性本地存储通常已经属于应用架构的一部分，用于存储您打开和关闭 Activity 时不想丢失的所有数据。

ViewModel 和已保存实例状态均不是长期存储解决方案，因此不能替代本地存储空间，例如数据库。您只应该使用这些机制来暂时存储瞬时界面状态，对于其他应用数据，应使用持久性存储空间。请参阅应用架构指南，详细了解如何充分利用本地存储空间长期保留您的应用模型数据（例如在重启设备后）。

 
#### 使用注意事项

不需要传入Context,会导致内存泄漏
如果需要传入Context 继承含有ApplicationContext的 AndroidViewModel 
ViewModel不可以替代OnSaveInstanceState.（https://developer.android.google.cn/topic/libraries/architecture/saving-states）


如果我们的应用需要大量的数据，那么推荐创建一个Repository类作为唯一的数据层入口
同时我们也要注意不要重蹈Activity的覆辙，避免在ViewModel内中实现更多的职责，创建一个Presenter类来处理UI界面数据。

### authodisposeViewModel

在ViewModel进行销毁的时候，如果我们在ViewModel仍然进行网络请求，
当您使用RxJava时，架构组件ViewModel的一个常见用例是您订阅ViewModel本身中的数据流。这对于提出正在运行的网络请求是有益的。由于您正在ViewModel中订阅，请求仍将完成。然后使用LiveData或类似BehaviorRelay的东西将ViewModel链接到视图。在这种情况下，您将在ViewModel中使用CompositeDisposable并在ViewModel的onCleared中调用dispose来处理一次性文件。

终止viewModel中的网络请求，主要目的就是这个。

### 最后

https://juejin.im/post/5a17d49b6fb9a0451704e229

- ViewMode1 https://v.qq.com/x/page/t0763s9ma8o.html
- ViewMode2  https://v.qq.com/x/page/m0605c1sejh.html
站在巨人的肩膀上，才能看的更远~
