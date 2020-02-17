---
title: Jepatc系列之ViewModel
tags:
  - ViewModel
categories:
  - Jetpack
---

讲解大纲：

ViewModel的介绍:
ViewModel的引入原因：
ViewModel的优点：1.不会因为配置该变而销毁，2.多个Fragment可以共享ViewModel,3.可以配合LiveData使用。(结合源码进行介绍)

ViewModel的注意事项：1.不要传入Context，会导致内存泄漏。
ViewModel与保存界面状态区别与配合使用。

### 前言

在Google中的生命周期管理库中，提供了ViewModel组件，

在Google的官方介绍中，称其为一个提供和管理UI界面数据， 并且可感知生命周期的组件。

ViewModel的一大特点就是不会因为设置变更而被销毁（正常的声明周期还是会销毁）。

这样做的好处就是，保证我们的数据，不会丢失，也是更值得推荐的软件开发模式。

在以往的Android应用程序开发过程中，数据逻辑的对象和常量常常被保存在Activity和Fragment里，
随着 activity代码越来越冗长，维护这些类，也就变得更加困难，这样做为我们之后的软件开发以及维护布下一个陷阱，同时这一做法也违背了单一职责的设计原则。

通过使用ViewModel，可以充分的划清界限，让每个类各司其职。ViewModel负责管理界面数据，UI组件则负责显示数据，和获取用户的操作。

### ViewModle简介

### ViewModel引入的原因

#### 老一套恢复数据的局限性

应用的某个 Activity 中可能包含用户列表。因配置更改而重新创建 Activity 后，新 Activity 必须重新提取用户列表。对于简单的数据，Activity 可以使用 onSaveInstanceState() 方法从 onCreate() 中的捆绑包恢复其数据，但此方法仅适合可以序列化再反序列化的少量数据，而不适合数量可能较大的数据，如用户列表或位图。

可能让用户重新请求网络数据，或者重新查询数据库。

#### Fragment与Activity会越来越冗余

Fragment除了显示界面数据的展示，响应用户的操作，还要处理从网络或数据库加载数据。这样会使activity代码越来越冗长，让我们之后软件的开发与维护造成了困难，同时这一做法也违背了单一职责的设计原则。我们应该将数据的加载从Fragment或Activity中分离出来，将工作委托给其他类。例如ViewModel?

这里可以画图。

#### ViewModel优点简介

#### 不会因为配置改变 而销毁

ViewModel是一个可感知生命周期的组件，这意味着

#### 可共享

### 什么样的情况下会保存数据

`onSaveInstanceState`与`onRestoreInstanceState`这两个方法。

#### 资源相关的配置发生改变导致Activity被杀死并重新创建时

#### 资源内存不足导致低优先及的Activity被杀死

按照进程优先级。


### 举一个简单的例子

一个数据列表，如果保证在配置发生改变的时候，不会重复的加载数据呢？

这个数据的恢复和我们界面的数据恢复是不一样的。界面数据的保存，系统的工作流程是这样的。首先Activity会调用
`onSaveInstanceState`去保存数据，然后Activity会委托Window去保存数据，接着Window在委托它上面的顶级容去保存数据。顶层容器是一个ViewGroup，最后顶层容器再去通知它的子元素来保存数据。

- 重写configChange，让Activity不重复调用加载数据方法
- 根据生命周期，冲走一次逻辑。缺点重复加载。
- 重写`onSaveInstanceState`与`onRestoreInstanceState`这两个方法。但是存储的数据大小有限制。

所以为了解决这个问题。

>ViewModel保存的数据一般是需要持久化的数据，

### 基本使用

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
        // Do an asynchronous operation to fetch users.
    }
}
```

```java
public class MyActivity extends AppCompatActivity {
    public void onCreate(Bundle savedInstanceState) {
        // Create a ViewModel the first time the system calls an activity's onCreate() method.
        // Re-created activities receive the same MyViewModel instance created by the first activity.

        MyViewModel model = ViewModelProviders.of(this).get(MyViewModel.class);
        model.getUsers().observe(this, users -> {
            // update UI
        });
    }
}
```

### 原理分析

每一个 Activity 对应一个 ViewModelProvider  对应一个ViewModelStore。


```java
   public static ViewModelProvider of(@NonNull FragmentActivity activity) {
        return new ViewModelProvider(activity);
    }

```

```java
 public ViewModelProvider(@NonNull ViewModelStoreOwner owner) {
        this(owner.getViewModelStore(), owner instanceof HasDefaultViewModelProviderFactory
                ? ((HasDefaultViewModelProviderFactory) owner).getDefaultViewModelProviderFactory()
                : NewInstanceFactory.getInstance());
    }
```

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

当我们创建好ViewModel，接着调用 get 方法时，

```java
 public <T extends ViewModel> T get(@NonNull Class<T> modelClass) {
        String canonicalName = modelClass.getCanonicalName();
        if (canonicalName == null) {
            throw new IllegalArgumentException("Local and anonymous classes can not be ViewModels");
        }
        return get(DEFAULT_KEY + ":" + canonicalName, modelClass);
    }
```

```java
 public <T extends ViewModel> T get(@NonNull String key, @NonNull Class<T> modelClass) {
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
        if (mFactory instanceof KeyedFactory) {
            viewModel = ((KeyedFactory) (mFactory)).create(key, modelClass);
        } else {
            viewModel = (mFactory).create(modelClass);
        }
        mViewModelStore.put(key, viewModel);
        return (T) viewModel;
    }

```


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
     *  Clears internal storage and notifies ViewModels that they are no longer used.
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

ViewModelStore 其实存储的就是 Activity 中所有的ViewModel

### Fragment可共享的原理

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
```java

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

也就是同一FragmentManager下获取Fragment中的ViewModel

这里，可以介绍一下 FragmentManager :https://www.jianshu.com/p/fd71d65f0ec6


### ViewModel如何判断是否移除

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
                       //👇调用clear
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

当Activity 消失的时候清除所有的ViewModel

```java
   public boolean isChangingConfigurations() {
        return mChangingConfigurations;
    }
```

该函数，会在Activity被销毁，并且重建的时候该Activity会返回true, 那么也就是桌正常Activity生命周期走的话，ViewModel是会销毁的。




### ViewModel 使用范围

只要您的应用安装在用户的设备上，持续性本地存储（例如数据库或共享偏好设置）就会继续存在（除非用户清除应用的数据）。虽然此类本地存储空间会在系统启动的活动和应用进程终止后继续存在，但由于必须从本地存储空间读取到内存，因此检索成本高昂。这种持久性本地存储通常已经属于应用架构的一部分，用于存储您打开和关闭 Activity 时不想丢失的所有数据。

ViewModel 和已保存实例状态均不是长期存储解决方案，因此不能替代本地存储空间，例如数据库。您只应该使用这些机制来暂时存储瞬时界面状态，对于其他应用数据，应使用持久性存储空间。请参阅应用架构指南，详细了解如何充分利用本地存储空间长期保留您的应用模型数据（例如在重启设备后）。

 
### ViewModel使用注意的事项

优点1：可以使用Viewmodle 共享Fragments数据（共享的原理的就是 ViewModelProviders 去找相关的ViewModel)
优点2：ViewModel可以根据另一个生命组件使用。LiveData，来创建响应式的布局。

#### 使用注意事项

不需要传入Context,会导致内存泄漏
如果需要传入Context 继承含有ApplicationContext的 AndroidViewModel 
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

### authodisposeViewModel

在ViewModel进行销毁的是，如果我们在ViewModel仍然进行网络请求，
当您使用RxJava时，架构组件ViewModel的一个常见用例是您订阅ViewModel本身中的数据流。这对于提出正在运行的网络请求是有益的。由于您正在ViewModel中订阅，请求仍将完成。然后使用LiveData或类似BehaviorRelay的东西将ViewModel链接到视图。在这种情况下，您将在ViewModel中使用CompositeDisposable并在ViewModel的onCleared中调用dispose来处理一次性文件。

终止viewModel中的网络请求，主要目的就是这个。

### 最后

https://juejin.im/post/5a17d49b6fb9a0451704e229

- ViewMode1 https://v.qq.com/x/page/t0763s9ma8o.html
- ViewMode2  https://v.qq.com/x/page/m0605c1sejh.html
站在巨人的肩膀上，才能看的更远~
