---
title: AutoDispose使用及原理解析
tags:
  - null
categories:
  - null
---

### 前言

基本依赖：

``` java
    //autodispose
    implementation 'com.uber.autodispose:autodispose-android:1.4.0'
    implementation 'com.uber.autodispose:autodispose-android-archcomponents:1.4.0'
```

基本使用：

```java
myObservable
    .doStuff()
    .autoDispose(AndroidLifecycleScopeProvider.from(this)
    .subscribe(s -> ...);
```

其中 this 为：LifecycleOwner

### 使用原理

当我们调用autoDispose时，实际是走的这段流程

```java
 .autoDispose(AndroidLifecycleScopeProvider.from(this)
```

接着调用：

```java
 public static <T> AutoDisposeConverter<T> autoDisposable(final ScopeProvider provider) {
    checkNotNull(provider, "provider == null");
    return autoDisposable(completableOf(provider));
  }
```

```java
  public static Completable completableOf(ScopeProvider scopeProvider) {
    return Completable.defer(
        () -> {
          try {
            //👇这里会调用AndroidLifecycleScopeProvider的requestScope
            return scopeProvider.requestScope();
          } catch (OutsideScopeException e) {
            Consumer<? super OutsideScopeException> handler =
                AutoDisposePlugins.getOutsideScopeHandler();
            if (handler != null) {
              handler.accept(e);
              return Completable.complete();
            } else {
              return Completable.error(e);
            }
          }
        });
  }
```

最终会调用LifecycleScopes中的resolveScopeFromLifecycle 方法

```java

  public static <E> CompletableSource resolveScopeFromLifecycle(
      final LifecycleScopeProvider<E> provider, final boolean checkEndBoundary)
      throws OutsideScopeException {
    E lastEvent = provider.peekLifecycle();//👈获取之前的声明周期事件
    CorrespondingEventsFunction<E> eventsFunction = provider.correspondingEvents();//生命周期事件处理
    if (lastEvent == null) {
      throw new LifecycleNotStartedException();
    }
    E endEvent;
    try {
      endEvent = eventsFunction.apply(lastEvent);
    } catch (Exception e) {
      if (checkEndBoundary && e instanceof LifecycleEndedException) {
        Consumer<? super OutsideScopeException> handler =
            AutoDisposePlugins.getOutsideScopeHandler();
        if (handler != null) {
          try {
            handler.accept((LifecycleEndedException) e);

            // Swallowed the end exception, just silently dispose immediately.
            return Completable.complete();
          } catch (Exception e1) {
            return Completable.error(e1);
          }
        }
        throw e;
      }
      return Completable.error(e);
    }
    return resolveScopeFromLifecycle(provider.lifecycle(), endEvent);
  }
```

其中 correspondingEvents（）调用的是 correspondingEvents 也就是，我们在AndroidLifecycleScopeProvider中设置的
CorrespondingEventsFunction 代码如下

```java
  private static final CorrespondingEventsFunction<Lifecycle.Event> DEFAULT_CORRESPONDING_EVENTS =
      lastEvent -> {
        switch (lastEvent) {
          case ON_CREATE:
            return Lifecycle.Event.ON_DESTROY;
          case ON_START:
            return Lifecycle.Event.ON_STOP;
          case ON_RESUME:
            return Lifecycle.Event.ON_PAUSE;
          case ON_PAUSE:
            return Lifecycle.Event.ON_STOP;
          case ON_STOP:
          case ON_DESTROY:
          default:
            throw new LifecycleEndedException("Lifecycle has ended! Last event was " + lastEvent);
        }
      };
```

从代码中可以看出，那么当声明周期为 on_stop 或者 on_destory 时，会直接抛出异常。再结合之前的逻辑：

```java
  try {
      endEvent = eventsFunction.apply(lastEvent);
    } catch (Exception e) {
      if (checkEndBoundary && e instanceof LifecycleEndedException) {
        //👇获取自定义的外部处理，我们可以通过AutoDisposePlugins.setOutsideScopeHandler来设置
        Consumer<? super OutsideScopeException> handler =
            AutoDisposePlugins.getOutsideScopeHandler();
        if (handler != null) {
          try {
            handler.accept((LifecycleEndedException) e);

            // Swallowed the end exception, just silently dispose immediately.
            return Completable.complete();
          } catch (Exception e1) {
            return Completable.error(e1);
          }
        }
        //👇这里将这个异常抛出
        throw e;
      }
      return Completable.error(e);
    }
```

最终抛出的异常又会被Scopes 捕获：

```java
  public static Completable completableOf(ScopeProvider scopeProvider) {
    return Completable.defer(
        () -> {
          try {
            return scopeProvider.requestScope();
          } catch (OutsideScopeException e) {
            //👇捕获异常，注意OutsideScopeException 是 LifecycleEndedException的父类 所以这里我们
            //之前生命周期抛出的异常是能被捕获的，
            Consumer<? super OutsideScopeException> handler =
                AutoDisposePlugins.getOutsideScopeHandler();
            if (handler != null) {
              handler.accept(e);
              return Completable.complete();
            } else {
              //👇如果没有自定义处理，直接抛出eroor
              return Completable.error(e);
            }
          }
        });
  }
```

当收到Error方法时，会中断订阅关系，

```java
public static <T> AutoDisposeConverter<T> autoDisposable(final CompletableSource scope) {}
```

这里 查看 AutoDisposeConverter

```java
public interface AutoDisposeConverter<T>
    extends FlowableConverter<T, FlowableSubscribeProxy<T>>,
        ParallelFlowableConverter<T, ParallelFlowableSubscribeProxy<T>>,
        ObservableConverter<T, ObservableSubscribeProxy<T>>,
        MaybeConverter<T, MaybeSubscribeProxy<T>>,
        SingleConverter<T, SingleSubscribeProxy<T>>,
        CompletableConverter<CompletableSubscribeProxy> {}
```

因为AutoDispose中实现类有多个，这里以 AutoDisposeCompletable 为例：

```java
final class AutoDisposeCompletable extends Completable implements CompletableSubscribeProxy {

  private final Completable source;
  private final CompletableSource scope;

  AutoDisposeCompletable(Completable source, CompletableSource scope) {
    this.source = source;
    this.scope = scope;
  }

  @Override
  protected void subscribeActual(CompletableObserver completableObserver) {
    source.subscribe(new AutoDisposingCompletableObserverImpl(scope, completableObserver));
  }
}
```

也就是实际的解除操作是交给 AutoDisposingCompletableObserverImpl 来实现的，我们继续查看该类

```java
@Override
  public void onSubscribe(final Disposable d) {
    DisposableCompletableObserver o =
        new DisposableCompletableObserver() {
          @Override
          public void onError(Throwable e) {
            scopeDisposable.lazySet(AutoDisposableHelper.DISPOSED);
            AutoDisposingCompletableObserverImpl.this.onError(e);
          }

          @Override
          public void onComplete() {
            scopeDisposable.lazySet(AutoDisposableHelper.DISPOSED);
            AutoDisposableHelper.dispose(mainDisposable);
          }
        };
        //👇在订阅的时候，把dispose设置进scopeDisposable中了
    if (AutoDisposeEndConsumerHelper.setOnce(scopeDisposable, o, getClass())) {
      delegate.onSubscribe(this);
      scope.subscribe(o);
      //👇在订阅的时候，把dispose设置进mainDisposable中了
      AutoDisposeEndConsumerHelper.setOnce(mainDisposable, d, getClass());
    }
  }

 @Override
  public void onComplete() {
    if (!isDisposed()) {
      mainDisposable.lazySet(AutoDisposableHelper.DISPOSED);
      AutoDisposableHelper.dispose(scopeDisposable);
      delegate.onComplete();
    }
  }

  @Override
  public void onError(Throwable e) {
    if (!isDisposed()) {
      //👇这里如果接受到错误消息，直接就解除订阅
      mainDisposable.lazySet(AutoDisposableHelper.DISPOSED);
      AutoDisposableHelper.dispose(scopeDisposable);
      delegate.onError(e);
    }
  }
```
  
这里会调用：AutoDisposableHelper.dispose(scopeDisposable);

```java
static boolean dispose(AtomicReference<Disposable> field) {
    Disposable current = field.get();
    Disposable d = DISPOSED;
    if (current != d) {
      current = field.getAndSet(d);
      if (current != d) {
        if (current != null) {
          current.dispose();
        }
        return true;
      }
    }
    return false;
  }
```

也就是这里解除订阅

### 生命周期注册过程

```java
  public static AndroidLifecycleScopeProvider from(
      Lifecycle lifecycle, CorrespondingEventsFunction<Lifecycle.Event> boundaryResolver) {
    return new AndroidLifecycleScopeProvider(lifecycle, boundaryResolver);
  }

```

```java
  private AndroidLifecycleScopeProvider(
      Lifecycle lifecycle, CorrespondingEventsFunction<Lifecycle.Event> boundaryResolver) {
    this.lifecycleObservable = new LifecycleEventsObservable(lifecycle);
    this.boundaryResolver = boundaryResolver;
  }

```


也就是说与LifecycleEventsObservable有关，我们继续查看该类：

```java
class LifecycleEventsObservable extends Observable<Event> {}
```

LifecycleEventsObservable 默认实现了Observable，


```java
  @Override
  protected void subscribeActual(Observer<? super Event> observer) {
    ArchLifecycleObserver archObserver =
        new ArchLifecycleObserver(lifecycle, observer, eventsObservable);
    observer.onSubscribe(archObserver);
    if (!isMainThread()) {
      observer.onError(
          new IllegalStateException("Lifecycles can only be bound to on the main thread!"));
      return;
    }
    //👇这里监听了lifecycle的事件
    lifecycle.addObserver(archObserver);
    if (archObserver.isDisposed()) {
      lifecycle.removeObserver(archObserver);
    }
  }
```

```java
  static final class ArchLifecycleObserver extends MainThreadDisposable
      implements LifecycleObserver {
    private final Lifecycle lifecycle;
    private final Observer<? super Event> observer;
    private final BehaviorSubject<Event> eventsObservable;

    ArchLifecycleObserver(
        Lifecycle lifecycle,
        Observer<? super Event> observer,
        BehaviorSubject<Event> eventsObservable) {
      this.lifecycle = lifecycle;
      this.observer = observer;
      this.eventsObservable = eventsObservable;
    }

    //在解除订阅关系的时候，取消订阅生命周期
    @Override
    protected void onDispose() {
      lifecycle.removeObserver(this);
    }
  
     //👇这里接受生命周期的回调
    @OnLifecycleEvent(Event.ON_ANY)
    void onStateChange(@SuppressWarnings("unused") LifecycleOwner owner, Event event) {
      if (!isDisposed()) {
        if (!(event == ON_CREATE && eventsObservable.getValue() == event)) {//防止发射重复生命周期事件
          // Due to the INITIALIZED->ON_CREATE mapping trick we do in backfill(),
          // we fire this conditionally to avoid duplicate CREATE events.
          eventsObservable.onNext(event);
        }
        observer.onNext(event);
      }
    }
  }
```

### 最后

站在巨人的肩膀上，才能看的更远~
