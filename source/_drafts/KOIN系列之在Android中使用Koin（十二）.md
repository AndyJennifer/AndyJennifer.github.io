---
title: KOIN系列之在Android中使用Koin（十二）
tags:
  - koin
categories:
  - koin
---

### 前言

`Koin-Android` 项目致力于为 Android 领域提供 Koin 支持。

### 在你的 Application 调用 startKoin()

在你的 `Application` 中，你可以使用 `startKoin` 函数将 Android context 通过 `androidContext` 注入 ，如下所示:

```kotlin
class MainApplication : Application() {

    override fun onCreate() {
        super.onCreate()

        startKoin {
            // Koin中的 Android 日志记录
            androidLogger()
            //注入 Android context
            androidContext(this@MainApplication)
            //使用的 module
            modules(myAppModules)
        }

    }
}
```

### 在其他地方使用 Android context 启动 Koin

如果你需要从另一个 Android 类中启动 Koin，你可以使用 `startKoin` 函数，并提供你的Android `Context` 实例，如下所示：

```kotlin
startKoin {
    //注入 Android context
    androidContext(/* 你自己的 android context */)
    //使用的 module
    modules(myAppModules)
}
```

### Koin 日志记录

在 `KoinApplication` 实例中，我们有一个扩展 `androidLogger`，其内部使用的是 `AndroidLogger`对象，这个logger是 Koin logger的一个 Android 实现。

如果这个日志记录器不适合您的需求，你可以按需修改。

```kotlin
class MainApplication : Application() {

    override fun onCreate() {
        super.onCreate()

        startKoin {
            //注入 Android context
            androidContext(/* 你自己的 android context */)
            //使用 Android 下的日志 默认是 INFO 级别
            androidLogger()
            //使用的 module
            modules(myAppModules)
        }
    }
}
```

### 获取 module 中的 Android context

在 koin DSL 中增加了如下关键字：

`androidContext()` 和 `androidApplication()` 函数允许你在一个 Koin module 中获取上下文对象，来帮助你简单地编写需要 `Application` 对象的表达式。

```kotlin
val appModule = module {

    //从Android资源文件中获取 R.string.mystring 资源，并创建 Presenter 实例
    factory {
        MyPresenter(androidContext().resources.getString(R.string.mystring))
    }
}
```

### 最后

站在巨人的肩膀上，才能看的更远~
