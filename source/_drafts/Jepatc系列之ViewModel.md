---
title: Jepatc系列之ViewModel
tags:
  - null
categories:
  - null
---

### 前言

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


主线程对应同一个ViewModelProviders 对应同一个 AndroidViewModelFactory

```
    public static class AndroidViewModelFactory extends ViewModelProvider.NewInstanceFactory {

        private static AndroidViewModelFactory sInstance;

        /**
         * Retrieve a singleton instance of AndroidViewModelFactory.
         *
         * @param application an application to pass in {@link AndroidViewModel}
         * @return A valid {@link AndroidViewModelFactory}
         */
        @NonNull
        public static AndroidViewModelFactory getInstance(@NonNull Application application) {
            if (sInstance == null) {
                sInstance = new AndroidViewModelFactory(application);
            }
            return sInstance;
        }

        private Application mApplication;

        /**
         * Creates a {@code AndroidViewModelFactory}
         *
         * @param application an application to pass in {@link AndroidViewModel}
         */
        public AndroidViewModelFactory(@NonNull Application application) {
            mApplication = application;
        }

        @NonNull
        @Override
        public <T extends ViewModel> T create(@NonNull Class<T> modelClass) {
            if (AndroidViewModel.class.isAssignableFrom(modelClass)) {
                //noinspection TryWithIdenticalCatches
                try {
                    return modelClass.getConstructor(Application.class).newInstance(mApplication);
                } catch (NoSuchMethodException e) {
                    throw new RuntimeException("Cannot create an instance of " + modelClass, e);
                } catch (IllegalAccessException e) {
                    throw new RuntimeException("Cannot create an instance of " + modelClass, e);
                } catch (InstantiationException e) {
                    throw new RuntimeException("Cannot create an instance of " + modelClass, e);
                } catch (InvocationTargetException e) {
                    throw new RuntimeException("Cannot create an instance of " + modelClass, e);
                }
            }
            return super.create(modelClass);
        }
    }
```

每一个Activity 对应一个ViewModelProvider  对应一个ViewModelStore。


```java
  public static ViewModelProvider of(@NonNull FragmentActivity activity,
            @Nullable Factory factory) {
        Application application = checkApplication(activity);
        if (factory == null) {
            factory = ViewModelProvider.AndroidViewModelFactory.getInstance(application);
        }
        return new ViewModelProvider(activity.getViewModelStore(), factory);
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

ViewModelStore 其实存储的就是Activity 中所有的ViewModel


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

https://juejin.im/post/5a17d49b6fb9a0451704e229

站在巨人的肩膀上，才能看的更远~
