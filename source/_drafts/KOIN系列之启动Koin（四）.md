---
title: KOIN系列之启动Koin（四）
tags:
  - koin
categories:
  - koin
---

### Koin是什么

Koin 是面向 Kotlin 开发人员的实用的轻量级依赖注入框架。用纯Kotlin编写，使用函数式解决作为主要概念。Koin 是一个DSL、一个轻量级容器和一个实用的API。

### 启动容器

Koin 是一个DSL、一个轻量级容器和一个实用的API。一旦在Koin模块中声明了定义，就可以启动Koin容器了。

### startKoin 函数

`startKoin` 函数是启动 Koin 容器的主要入口点。它需要运行Koin模块的列表。当模块被加载时，其内部定义的声明将会被Koin容器解析。

开启 koin

```kotlin
// 启动一个全局上下文 KoinApplication
startKoin {
    // 声明使用的module
    modules(coffeeAppModule)
}
```

一旦 `startKoin` 被调用，Koin 将读取你所有声明的模块和定义。当你执行任意 `get()` 或`inject()` 方法时，kolin将会检索出你所需的实例对象。

你的Koin容器可以有几个选项:

- `logger`-允许日志可用
- `properties()` 、 `fileProperties()` 或 `environmentProperties()` 从环境、koin.perperite文件，或其他配置中…中加载相关属性。

`startKoin 不能被多次调用，如果需要额外加载模块，请使用 loadKoinModules 函数。`

### 启动- KoinApplication和Koin实例的背后

当我们启动 Koin 时，我们会创建一个
Koin 容器配置对象--`KoinApplication` 。 该对应一旦被加载，它将生成一个由您的模块和选项配置的 `Koin` 实例，这个 `Koin` 实例接着会在`GlobalContext` 中被注册，以便供任何`KoinComponen` t类使用。

### 在调用 startKoin 后加载模块

虽然我们不能多次调用 `startKoin` 函数。但是我们可以直接使用 `loadKoinModules()` 函数。

这对于希望使用 Koin 的 SDK 开发人员来说，这个函数很有趣，因为他们不需要使用 starKoin() 函数，只需在库的开头使用 `loadKoinModules` 即可。

```kotlin
loadKoinModules(module1,module2 ...)
```

### 卸载模块

我们可以卸载多个Modlue，使用给定函数来释放他们的实例对象:

```kotlin
unloadKoinModules(module1,module2 ...)
```

### Koin上下文隔离

对于 SDK 开发人员，您还可以以一种非全局的方式使用 Koin :将 Koin 用于 library 的 DI（依赖注入），并通过隔离上下文的方式，来避免同时使用 library 与 Koin 之间的冲突。

标准的方式下启动 Koin:

```kotlin
// start a KoinApplication and register it in Global context
startKoin {
    // declare used modules
    modules(coffeeAppModule)
}
```

在这里，我们可以使用 `KoinComponent` :它将使用 `GlobalContext` 中的 Koin 实例对象。

但如果我们想使用一个被隔离的 Koin 实例，我们可以像下面这样声明:

```kotlin
// create a KoinApplication
val myApp = koinApplication {
    // declare used modules
    modules(coffeeAppModule)
}
```

你必须保持你的 `myApp` 实例在你的库中可用，并将其传递给你的自定义 `KoinComponent` 实现:

```kotlin
// Get a Context for your Koin instance
object MyKoinContext {
    var koinApp : KoinApplication? = null
}

// Register the Koin context
MyKoinContext.koinApp = KoinApp

```kotlin
abstract class CustomKoinComponent : KoinComponent {
    // Override default Koin instance, intially target on GlobalContext to yours
    override fun getKoin(): Koin = MyKoinContext?.koinApp.koin
}

现在，你注册你的上下文和运行你自己的被隔离的Koin 组件:

```kotlin
// Register the Koin context
MyKoinContext.koinApp = myApp

class ACustomKoinComponent : CustomKoinComponent(){
    // inject & get will target MyKoinContext
}

```

### 停止Koin -关闭所有资源

您可以关闭所有 Koin 资源并删除实例对象和声明。为此，您可以在任何地方使用 `stopKoin(`)函数来停止 Koin `GlobalContext` 。或者在KoinApplication 实例中，只需调用 `close()`

### 最后

站在巨人的肩膀上，才能看的更远~
