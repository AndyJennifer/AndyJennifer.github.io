---
title: ViewModel这些知识点你都知道吗
tags:
  - ViewModel
categories:
  - Jetpack
---

### 前言

[ViewModel](https://developer.android.google.cn/topic/libraries/architecture/viewmodel) 作为 [Jetpack](https://developer.android.google.cn/jetpack) 中的明星组件，相信大家都对其有一定的了解。在 Google 的官方介绍中也详细的罗列了 ViewModel 的优点，如：

1. 可以提供和管理UI界面数据。(将加载数据与数据恢复从 Activity or Fragment中解耦)
2. 可感知生命周期的组件。
3. 不会因配置改变而销毁。
4. 可以配合 LiveData 使用。
5. 多个 Fragment 可以共享同一 ViewModel。
6. 等等等....

你也可以通过下列两个视频，更为详细的了解 ViewModel：

- [ViewMode男生讲解版](https://v.qq.com/x/page/t0763s9ma8o.html)
- [ViewMode女生讲解版](https://v.qq.com/x/page/m0605c1sejh.html)

在本篇文章中，不会讲解 ViewModel 的使用方式及使用 ViewModel 的原因，而是着重于讲解 ViewModel 的原理及额外注意事项。通过阅读本篇文章你能了解到：

- ViewModel 与 Activity 的绑定过程
- 常见的数据恢复的方式
- ViewModel 在 Activity 中不会因配置改变而销毁的原理及流程。
- ViewModel 为何与 OnSaveInstanceState 配合使用。
- ViewModel 在 Fragment 中不会因配置改变而销毁的原理及流程。
- ViewModel 如何做到共享的。
- ViewModel 中使用网络请求时需要注意的事项。
  
希望通过该篇文章，大家能对 ViewModel 有更深入的了解。

### ViewModel 与 Activity 的绑定过程

一般情况下，使用 `ViewModel`，我们一般会先声明自己的 ViewModel，并在 Activity 中的 `onCreate` 方法中使用 `ViewModelProviders` 来创建 ViewModel。 如下代码所示：

```java
 MyViewModel model = ViewModelProviders.of(this).get(MyViewModel.class);
```

>在谷歌的最新代码中，不推荐使用 `ViweModelProviders(注意是有s的呦)` ，而是直接使用  `ViewModelProvider` 的构造函数来创建 `ViewModelProvider` 对象。

通过使用 `ViewModelProviders` 类的 `of()` 方法，我们会得到一个 `ViewModelProvider` 对象。如下代码所示：

```java
   public static ViewModelProvider of(@NonNull FragmentActivity activity) {
        return new ViewModelProvider(activity);
    }
```

ViewModelProvider 类需要我们传递 `ViewModelStore` 与 `Factory` 对象。其构造函数声明如下：

```java
    //使用ViewModelStoreOwner对象构造函数
    public ViewModelProvider(@NonNull ViewModelStoreOwner owner) {
        this(owner.getViewModelStore(), owner instanceof HasDefaultViewModelProviderFactory
                ? ((HasDefaultViewModelProviderFactory) owner).getDefaultViewModelProviderFactory()
                : NewInstanceFactory.getInstance());
    }

    //使用ViewModelStoreOwner与Factory对象的构造函数
    public ViewModelProvider(@NonNull ViewModelStoreOwner owner, @NonNull Factory factory) {
        this(owner.getViewModelStore(), factory);
    }

    //使用ViewModelStore与Factory对象的构造函数
    public ViewModelProvider(@NonNull ViewModelStore store, @NonNull Factory factory) {
        mFactory = factory;
        mViewModelStore = store;
    }
```

在 ViewModelProvider 内部，拥有三种类型构造函数：

- `(ViewModelStoreOwner owner)`:
  - 该构造函数使用 owner 对象的 `getViewModelStore()` 方法来获取 `ViewModelStore` 对象，如果传入的 owner 对象也实现了 `HasDefaultViewModelProviderFactory` 接口时，那么会调用 `getDefaultViewModelProviderFactory()` 方法获取 Factory。反之，使用内部静态的 `NewInstanceFactory` 对象来创建 Factory 对象。
- `(ViewModelStoreOwner owner,  Factory factory)`:
  - 该构造函数使用 owner 对象的 `getViewModelStore()` 方法来获取 `ViewModelStore` 对象，使用传递的 Factory 对象
- `(ViewModelStore store, Factory factory)`：
  - 使用 `ViewModelStore` 与 `Factory` 对象的构造函数

#### Factory 接口

在 ViewModelProvider中，Factory 主要用于创建 ViewModel，Factory 的声明如下：

```java
    public interface Factory {
        /**
         * 通过给定的Class对象创建ViewModel对象
         * <p>
         *
         * @param modelClass 所需ViewModel的Class对象
         * @param <T>        ViewModel的泛型参数
         * @return 新创建的ViewModel对象
         */
        @NonNull
        <T extends ViewModel> T create(@NonNull Class<T> modelClass);
    }
```

通过实现 Factory 接口，我们可以实现自己想要的工厂以创建所需的 ViewModel。在 Android 中有多个类都实现了该接口，如：`KeyedFactory`，`AndroidViewModelFactory` 等，这里以默认的 `NewInstanceFactory` 为例：

```java
    public static class NewInstanceFactory implements Factory {

        private static NewInstanceFactory sInstance;

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
            try {
                //默认使用对应ViewModel类无参的构造函数创建实例对象
                return modelClass.newInstance();
            } catch (InstantiationException e) {
                throw new RuntimeException("Cannot create an instance of " + modelClass, e);
            } catch (IllegalAccessException e) {
                throw new RuntimeException("Cannot create an instance of " + modelClass, e);
            }
        }
    }
```

默认情况下， `NewInstanceFactory` 会调用 ViewModel 的**无参构造函数**创建实例对象，当然如果你需要在 ViewModel 中使用其他参数，你也可以传递自定义的 Factory。

#### ViewModelStore

ViewModelStore 内部维护了一个 HashMap，其 key 为 `DEFAULT_KEY` + `ViewModel的Class对象底层类规范名称`，其 value 为对应 ViewModel 对象。每个 Activity 与 Fragment 都对应着一个 ViewModelStore ，用于存储所需的 ViewModel。ViewModelStore 类声明如下所示：

> DEFAULT_KEY 值为："androidx.lifecycle.ViewModelProvider.DefaultKey"

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

#### Activity中创建与获取ViewModel流程

ViewModel 最终的创建与获取，需要 ViewProvider 类调用 `get(Class<T> modelClass)`方法（该方法内部通过 ViewModelStore 与 Factory 的配合，创建并保存了所需的ViewModel对象），具体代码如下所示：

```java
 public <T extends ViewModel> T get(@NonNull Class<T> modelClass) {
        String canonicalName = modelClass.getCanonicalName();
        if (canonicalName == null) {
            throw new IllegalArgumentException("Local and anonymous classes can not be ViewModels");
        }
        return get(DEFAULT_KEY + ":" + canonicalName, modelClass);
    }
```

该方法内部会调用 `get(String key, Class<T> modelClass)` 方法：

```java
 public <T extends ViewModel> T get(@NonNull String key, @NonNull Class<T> modelClass) {
        //👇根据key值从ViewModelStore中取对应的ViewModel
        ViewModel viewModel = mViewModelStore.get(key);
        //👇判断所传入的Class对象是否是ViewModel的Class类或其子类的对象，如果是，直接返回
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
        //👇如果为null，根据传入的Factory创建新的VideModel
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

在该方法中，会在 ViewModelStore 中根据传入的 key 获取并保存 ViewModel。其具体逻辑如下：

- 根据 key 值从 ViewModelStore 中取对应的 ViewModel。
- 判断所传入的 Class 对象是否是 ViewModel 的 Class 类或其子类的对象，如果是，直接返回。（当 `Object.isInstance(class)` 接受的参数为 `null` 时，该方法会返回 false）
- 如果获取的 ViewModel 为 null，会根据传入的 Factory 对象创建新的 VideModel，并将创建好的 ViewModel 放入 ViewModelStore中。

结合所有的流程，我们能得到 Activity 中创建与获取 ViewModel 的整体流程：

![Activity下ViewModel的创建过程.png](https://upload-images.jianshu.io/upload_images/2824145-fecc9582d2892c82.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### ViewModel 如何做到不会因为配置改变而销毁

我们都知道 ViewModel 不会因为 Activity 的配置发生改变而销毁，ViewModel 作用域如下所示：

![viewmodel-lifecycle.png](https://upload-images.jianshu.io/upload_images/2824145-10c47be88330a248.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

观察上图，我相信小伙伴们肯定有如下疑惑：

- 当 Activity 因配置发生改变时，系统会重新创建一个新的 Activity 。那老的 Activity 中的 ViewModel 是如何传递给新的 Activity 的呢？
- ViewModel 又是如何感知配置是否改变，进而判断是否销毁的呢？

要解决如上问题，我们需要了解 Android 中数据恢复的方式以及 Activity 生命周期中 ViewModel 实际处理流程。

#### 数据恢复

在 Android 系统中，需要数据恢复有如下两种场景：

- 场景1：资源相关的配置发生改变导致 Activity 被杀死并重新创建。
- 场景2：资源内存不足导致低优先级的 Activity 被杀死，当内存恢复时，Activity 又被重建。

针对上述场景，分别对应三种不同的数据恢复方式。

>对应场景1，不考虑在清单文件中配置 `android:configChanges` 的特殊情况。

##### 使用 onSaveInstanceState 与 onRestoreInstanceState

使用 onSaveInstanceState 与 onRestoreInstanceState 方法，能处理 Activity 因配置发生改变及进程被杀死时数据的恢复。当你的界面数据简单且轻量时，例如原始数据类型或简单对象（比如 String)，则我们可以采用该方式。如果你需要恢复的数据较为复杂，那你应该考虑使用 `ViewModle + onSaveInstanceState()` (为什么要配合使用，会在下文进行讲解)，因为使用 onSaveInstanceState() 会导致序列化或反序列化，而这，有一定的时间消耗。

onSaveInstanceState() 更为详细的介绍以及使用，可参考官方文档：

- [使用 onSaveInstanceState() 保存简单轻量的界面状态](https://developer.android.google.cn/guide/components/activities/activity-lifecycle.html#%E4%BD%BF%E7%94%A8-onsaveinstancestate-%E4%BF%9D%E5%AD%98%E7%AE%80%E5%8D%95%E8%BD%BB%E9%87%8F%E7%9A%84%E7%95%8C%E9%9D%A2%E7%8A%B6%E6%80%81)
- [使用保存的实例状态恢复 Activity 界面状态](https://developer.android.google.cn/guide/components/activities/activity-lifecycle.html#%E4%BD%BF%E7%94%A8%E4%BF%9D%E5%AD%98%E7%9A%84%E5%AE%9E%E4%BE%8B%E7%8A%B6%E6%80%81%E6%81%A2%E5%A4%8D-activity-%E7%95%8C%E9%9D%A2%E7%8A%B6%E6%80%81)

##### 使用 Fragment 的 setRetainInstance

当配置发生改变时，Fragment 会随着宿主 Activity 销毁与重建，当我们调用 Fragment 中的 `setRetainInstance(true)` 方法时，系统允许 Fragment 绕开销毁-重建的过程。使用该方法，将会发送信号给系统，让 Activity 重建时，保留 Fragment 的实例。需要注意的是：

- 使用该方法后，不会调用 Fragment 的 `onDestory()` 方法，但仍然会调用 `onDetach()` 方法
- 使用该方法后，不会调用 Fragment 的 `onCreate(Bundle)` 方法。因为 Fragment 没有被重建。
- 使用该方法后，Fragment 的 `onAttach(Activity)` 与 `onActivityCreated(Bundle)` 方法仍然会被调用。

以下示例代码展示了如何在配置发生改变时，保留 Fragment 实例，并进行数据的恢复。

```java
public class MainActivity extends AppCompatActivity {

    private SaveFragment mSaveFragment;

    public static final String TAG_SAVE_FRAGMENT = "save_fragment";

    @Override
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        FragmentManager fm = getSupportFragmentManager();
        mSaveFragment = (SaveFragment) fm.findFragmentByTag(TAG_SAVE_FRAGMENT);

        // fragment 不为空，是因为配置发生改变，Fragment 被重建
        if (mSaveFragment == null) {
            mSaveFragment = SaveFragment.newInstance();
            fm.beginTransaction().add(mSaveFragment, TAG_SAVE_FRAGMENT).commit();
        }

        //获取保存的数据
        int saveData = mSaveFragment.getSaveData();
    }
}
```

Fragment ：

```java
public class SaveFragment extends Fragment {

    private int saveData;

    public static SaveFragment newInstance() {
        Bundle args = new Bundle();
        SaveFragment fragment = new SaveFragment();
        fragment.setArguments(args);
        return fragment;
    }


    @Override
    public void onAttach(@NonNull Context context) {
        super.onAttach(context);
    }

    @Override
    public void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        //保存当前Fragment实例
        setRetainInstance(true);
        saveData = 1010;//通过网络请求或查询数据库，赋值需要保存的数据
    }

    @Override
    public void onDetach() {
        super.onDetach();
    }

    public int getSaveData() {
        return saveData;
    }
}
```

>关于 Fragment 的 setRetainInstance 更多用法与注意事项，可以参看这篇文章
[Handling Configuration Changes with Fragments](https://www.androiddesignpatterns.com/2013/04/retaining-objects-across-config-changes.html)

##### 使用 getLastNonConfigurationInstance 与 onRetainNonConfigurationInstance

内存保存及磁盘保存及序列化。

#### ViewModel的恢复方式

笔者是基于 SDK 版本 27 ，Lifecycle 版本 1.1.1 分析的。需要注意的是系统在 SDK 27 之前是通过一个不可见的 Fragment ，将 setRetainInstance() 设置为 true 进行处理的。笔者不再做过多分析，感兴趣的可自行研究。如分析有误，还多请指正。猜测是因为维护Frament栈。关于栈又又很多坑，所以Google又迁移回来了。

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

为了应对之前我们讲的上述两种场景的数据恢复。使用ViewModel在配置发生改变的时候不用再去请求网络或加载数据库，举一个搜索的例子。

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

### ViewModel与Fragment的绑定过程


#### FragmentManager栈视图

每个Fragment及宿主Activity(继承自FragmentActivity)都会在创建是，初始化一个FragmentManager对象，了解Fragment中的ViewModel与Activity的联系的关键，就是理清这些不同阶级的栈视图。

下面给出一个简要的关系图：

![FragmentManager栈对应关系.png](https://upload-images.jianshu.io/upload_images/2824145-9d85d056fb02e43c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

- 对于宿主Activity, `getSupportFragmentManager()`获取的 FragmentActivity 的 FragmentManager 对象;
- 对于 Fragment , `getFragmentManager` 是获取的父 Fragment (如果没有，则是 FragmentActivity )的 FragmentManager 对象，而 `getChildFragmentManager()`是获取自身的 FragmentManager 对象。


### 第一步将 FragmentManagerViewModel 存入 Activity 中的ViewModelStore中

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

在 Activity 中的 FragmentManager中的 FragmentManagerViewModel 中创建 Fragment 的 FragmentManagerViewModel

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

### 第二步创建 ViewModelStore 并存入对应FragmentManager 中的FragmentManaerViewModel中的mViewModelStores中

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

当 Fragment 的父Fragment 为空时，mFragmentManager 的值为宿主 Activity 的FragmentManager，反之，为父Fragment的FragmentManager，最终都会走到 FragmentManagerViewModel 中的 getViewModelStore 方法。

```java
  ViewModelStore getViewModelStore(@NonNull Fragment f) {
        ViewModelStore viewModelStore = mViewModelStores.get(f.mWho);
        if (viewModelStore == null) {
            viewModelStore = new ViewModelStore();
            //将创建好的ViewStore，放入FragmentManagerViewModel中
            mViewModelStores.put(f.mWho, viewModelStore);
        }
        return viewModelStore;
    }
     String mWho = UUID.randomUUID().toString();//这里的id获取
```

### ViewModel 使用范围

只要您的应用安装在用户的设备上，持续性本地存储（例如数据库或共享偏好设置）就会继续存在（除非用户清除应用的数据）。虽然此类本地存储空间会在系统启动的活动和应用进程终止后继续存在，但由于必须从本地存储空间读取到内存，因此检索成本高昂。这种持久性本地存储通常已经属于应用架构的一部分，用于存储您打开和关闭 Activity 时不想丢失的所有数据。

ViewModel 和已保存实例状态均不是长期存储解决方案，因此不能替代本地存储空间，例如数据库。您只应该使用这些机制来暂时存储瞬时界面状态，对于其他应用数据，应使用持久性存储空间。请参阅应用架构指南，详细了解如何充分利用本地存储空间长期保留您的应用模型数据（例如在重启设备后）。

#### 使用注意事项

不需要传入Context,会导致内存泄漏
如果需要传入Context 继承含有ApplicationContext的 AndroidViewModel 
ViewModel不可以替代OnSaveInstanceState.（https://developer.android.google.cn/topic/libraries/architecture/saving-states）


如果我们的应用需要大量的数据，那么推荐创建一个Repository类作为唯一的数据层入口
同时我们也要注意不要重蹈Activity的覆辙，避免在ViewModel内中实现更多的职责，创建一个Presenter类来处理UI界面数据。

### AuthodisposeViewModel

在ViewModel进行销毁的时候，如果我们在ViewModel仍然进行网络请求，
当您使用RxJava时，架构组件ViewModel的一个常见用例是您订阅ViewModel本身中的数据流。这对于提出正在运行的网络请求是有益的。由于您正在ViewModel中订阅，请求仍将完成。然后使用LiveData或类似BehaviorRelay的东西将ViewModel链接到视图。在这种情况下，您将在ViewModel中使用CompositeDisposable并在ViewModel的onCleared中调用dispose来处理一次性文件。

终止viewModel中的网络请求，主要目的就是这个。

### 最后

https://juejin.im/post/5a17d49b6fb9a0451704e229

- ViewMode1 https://v.qq.com/x/page/t0763s9ma8o.html
- ViewMode2  https://v.qq.com/x/page/m0605c1sejh.html
FragmentManager :https://www.jianshu.com/p/fd71d65f0ec6
站在巨人的肩膀上，才能看的更远~
