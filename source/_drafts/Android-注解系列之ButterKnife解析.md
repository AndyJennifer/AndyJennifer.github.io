---
title: 'Android 注解系列之ButterKnife解析 '
tags:
- ButterKnife
categories:
- 源码分析
---

```
 public static Unbinder bind(@NonNull Activity target) {
    View sourceView = target.getWindow().getDecorView();
    return createBinding(target, sourceView);
  }

  public static Unbinder bind(@NonNull View target) {
    return createBinding(target, target);
  }

  public static Unbinder bind(@NonNull Dialog target) {
    View sourceView = target.getWindow().getDecorView();
    return createBinding(target, sourceView);
  }

  public static Unbinder bind(@NonNull Object target, @NonNull Activity source) {
    View sourceView = source.getWindow().getDecorView();
    return createBinding(target, sourceView);
  }

  public static Unbinder bind(@NonNull Object target, @NonNull View source) {
    return createBinding(target, source);
  }
  
  public static Unbinder bind(@NonNull Object target, @NonNull Dialog source) {
    View sourceView = source.getWindow().getDecorView();
    return createBinding(target, sourceView);
  }
```

```
  private static Unbinder createBinding(@NonNull Object target, @NonNull View source) {
    Class<?> targetClass = target.getClass();
    if (debug) Log.d(TAG, "Looking up binding for " + targetClass.getName());
    Constructor<? extends Unbinder> constructor = findBindingConstructorForClass(targetClass);

    if (constructor == null) {
      return Unbinder.EMPTY;
    }

    //noinspection TryWithIdenticalCatches Resolves to API 19+ only type.
    try {
      return constructor.newInstance(target, source);
    } catch (IllegalAccessException e) {
      throw new RuntimeException("Unable to invoke " + constructor, e);
    } catch (InstantiationException e) {
      throw new RuntimeException("Unable to invoke " + constructor, e);
    } catch (InvocationTargetException e) {
      Throwable cause = e.getCause();
      if (cause instanceof RuntimeException) {
        throw (RuntimeException) cause;
      }
      if (cause instanceof Error) {
        throw (Error) cause;
      }
      throw new RuntimeException("Unable to create binding instance.", cause);
    }
  }
```

### findBindingConstructorForClass方法
```

  @Nullable @CheckResult @UiThread
  private static Constructor<? extends Unbinder> findBindingConstructorForClass(Class<?> cls) {
    Constructor<? extends Unbinder> bindingCtor = BINDINGS.get(cls);
    if (bindingCtor != null) {
      if (debug) Log.d(TAG, "HIT: Cached in binding map.");
      return bindingCtor;
    }
    String clsName = cls.getName();
    if (clsName.startsWith("android.") || clsName.startsWith("java.")) {
      if (debug) Log.d(TAG, "MISS: Reached framework class. Abandoning search.");
      return null;
    }
    try {
      Class<?> bindingClass = cls.getClassLoader().loadClass(clsName + "_ViewBinding");
      //noinspection unchecked
      bindingCtor = (Constructor<? extends Unbinder>) bindingClass.getConstructor(cls, View.class);
      if (debug) Log.d(TAG, "HIT: Loaded binding class and constructor.");
    } catch (ClassNotFoundException e) {
      if (debug) Log.d(TAG, "Not found. Trying superclass " + cls.getSuperclass().getName());
      bindingCtor = findBindingConstructorForClass(cls.getSuperclass());
    } catch (NoSuchMethodException e) {
      throw new RuntimeException("Unable to find binding constructor for " + clsName, e);
    }
    BINDINGS.put(cls, bindingCtor);
    return bindingCtor;
  }

```


### 生成的帮助类
```
public class MainActivity_ViewBinding implements Unbinder {
  private MainActivity target;

  @UiThread
  public MainActivity_ViewBinding(MainActivity target) {
    this(target, target.getWindow().getDecorView());
  }

  @UiThread
  public MainActivity_ViewBinding(MainActivity target, View source) {
    this.target = target;

    target.mFlexibleViewPager = Utils.findRequiredViewAsType(source, R.id.view_pager, "field 'mFlexibleViewPager'", FlexibleViewPager.class);
    target.mBottomNavigationBar = Utils.findRequiredViewAsType(source, R.id.bottom_navigation_bar, "field 'mBottomNavigationBar'", BottomBar.class);
  }

  @Override
  @CallSuper
  public void unbind() {
    MainActivity target = this.target;
    if (target == null) throw new IllegalStateException("Bindings already cleared.");
    this.target = null;

    target.mFlexibleViewPager = null;
    target.mBottomNavigationBar = null;
  }
}

```