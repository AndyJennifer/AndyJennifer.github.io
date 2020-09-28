---
title: LeakCannay原理解析
tags:
- 内存泄露
categories:
- 源码分析
---

### 前言

### 整体原理

- 监听 Activity 的生命周期
- 在 onDestroy 的时候，创建相应的 Refrence 和 RefrenceQueue，并启动后台进程去检测
- 一段时间之后，从 RefrenceQueue 读取，若读取不到相应 activity 的 Refrence，有可能发生泄露了，这个时候，再促发 gc，一段时间之后，再去读取，若在从 RefrenceQueue 还是读取不到相应 activity 的 refrence，可以断定是发生内存泄露了
- 发生内存泄露之后，dump，分析 hprof 文件，找到泄露路径

原文：https://blog.csdn.net/gdutxiaoxu/article/details/80738581 


### 整体过程

```java
    private fun initLeakCanary() {
      if (LeakCanary.isInAnalyzerProcess(this)) {
            return
        }
        LeakCanary.install(this)
    }

```

```java
 public static @NonNull RefWatcher install(@NonNull Application application) {
    return refWatcher(application).listenerServiceClass(DisplayLeakService.class)
        .excludedRefs(AndroidExcludedRefs.createAppDefaults().build())
        .buildAndInstall();
  }
```

- 其中DisplayLeakService本质是一个前台的IntentService主要用于分析内存泄露日志，并提供通知。
- AndroidExcludedRefs.createAppDefaults().build()主要用于排除开发中可以忽略掉的泄露路径

```java
  public @NonNull RefWatcher buildAndInstall() {
    if (LeakCanaryInternals.installedRefWatcher != null) {
      throw new UnsupportedOperationException("buildAndInstall() should only be called once.");
    }
    RefWatcher refWatcher = build();
    if (refWatcher != DISABLED) {
      if (enableDisplayLeakActivity) {
        LeakCanaryInternals.setEnabledAsync(context, DisplayLeakActivity.class, true);
      }
      //如果观察Activity
      if (watchActivities) {
        ActivityRefWatcher.install(context, refWatcher);
      }
      //如果观察Fragment
      if (watchFragments) {
        FragmentRefWatcher.Helper.install(context, refWatcher);
      }
    }
    LeakCanaryInternals.installedRefWatcher = refWatcher;
    return refWatcher;
  }
```

#### ActivityRefWatcher

```java
public static void install(@NonNull Context context, @NonNull RefWatcher refWatcher) {
    Application application = (Application) context.getApplicationContext();
    ActivityRefWatcher activityRefWatcher = new ActivityRefWatcher(application, refWatcher);

    application.registerActivityLifecycleCallbacks(activityRefWatcher.lifecycleCallbacks);
  }
```

其中监听的声明

```java
  private final Application.ActivityLifecycleCallbacks lifecycleCallbacks =
      new ActivityLifecycleCallbacksAdapter() {
        @Override public void onActivityDestroyed(Activity activity) {
          refWatcher.watch(activity);
        }
      };

```

在Activity销毁的时候，最终会回调RefWatcher的watch(Object watchedReference)方法。来观察Activity中的内存泄露情况

```java
  public void watch(Object watchedReference, String referenceName) {
    if (this == DISABLED) {
      return;
    }
    checkNotNull(watchedReference, "watchedReference");
    checkNotNull(referenceName, "referenceName");
    //得到开始检测时间
    final long watchStartNanoTime = System.nanoTime();
    String key = UUID.randomUUID().toString();
    retainedKeys.add(key);
    //通过uuid,生成WeakReference
    final KeyedWeakReference reference =
        new KeyedWeakReference(watchedReference, key, referenceName, queue);
    //开始检测生成的弱引用，
    ensureGoneAsync(watchStartNanoTime, reference);
  }
```

执行任务，开始检测。内部是通过Handler

```java
  private void ensureGoneAsync(final long watchStartNanoTime, final KeyedWeakReference reference) {
    watchExecutor.execute(new Retryable() {
      @Override public Retryable.Result run() {
        return ensureGone(reference, watchStartNanoTime);
      }
    });
  }
```

```java
  Retryable.Result ensureGone(final KeyedWeakReference reference, final long watchStartNanoTime) {
    long gcStartNanoTime = System.nanoTime();
    long watchDurationMs = NANOSECONDS.toMillis(gcStartNanoTime - watchStartNanoTime);
    //移除已经被回收的activity引用
    removeWeaklyReachableReferences();

    if (debuggerControl.isDebuggerAttached()) {
      // The debugger can create false leaks.
      return RETRY;
    }
    //判断当前activity是否被gc
    if (gone(reference)) {
      return DONE;
    }
    //如果activity没有被回收，强制GC
    gcTrigger.runGc();
    //再次移除已经被回收的activity引用
    removeWeaklyReachableReferences();
    //如果这个时候对象还没有被回收，说明发生了内存泄露
    if (!gone(reference)) {
      long startDumpHeap = System.nanoTime();
      long gcDurationMs = NANOSECONDS.toMillis(startDumpHeap - gcStartNanoTime);
      //获取dump文件
      File heapDumpFile = heapDumper.dumpHeap();
      if (heapDumpFile == RETRY_LATER) {
        // Could not dump the heap.
        return RETRY;
      }
      long heapDumpDurationMs = NANOSECONDS.toMillis(System.nanoTime() - startDumpHeap);

     //分析内存
      HeapDump heapDump = heapDumpBuilder.heapDumpFile(heapDumpFile).referenceKey(reference.key)
          .referenceName(reference.name)
          .watchDurationMs(watchDurationMs)
          .gcDurationMs(gcDurationMs)
          .heapDumpDurationMs(heapDumpDurationMs)
          .build();
      //分析heap prof 文件，找到内存泄漏点
      heapdumpListener.analyze(heapDump);
    }
    return DONE;
  }

```

```java
  @Override public void analyze(@NonNull HeapDump heapDump) {
    checkNotNull(heapDump, "heapDump");
    HeapAnalyzerService.runAnalysis(context, heapDump, listenerServiceClass);
  }
```

```java
  @Override protected void onHandleIntentInForeground(@Nullable Intent intent) {
    if (intent == null) {
      CanaryLog.d("HeapAnalyzerService received a null intent, ignoring.");
      return;
    }
    String listenerClassName = intent.getStringExtra(LISTENER_CLASS_EXTRA);
    HeapDump heapDump = (HeapDump) intent.getSerializableExtra(HEAPDUMP_EXTRA);

    HeapAnalyzer heapAnalyzer =
        new HeapAnalyzer(heapDump.excludedRefs, this, heapDump.reachabilityInspectorClasses);

    AnalysisResult result = heapAnalyzer.checkForLeak(heapDump.heapDumpFile, heapDump.referenceKey,
        heapDump.computeRetainedHeapSize);
    AbstractAnalysisResultService.sendResultToListener(this, listenerClassName, heapDump, result);
  }
```