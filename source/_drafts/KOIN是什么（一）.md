---
title: KOIN是什么(一)
tags:
  - koin
categories:
  - koin
---

### KOIN 是什么

一个面向Kotlin开发人员的实用的轻量级依赖注入框架。用纯 Kotlin 编写，只使用函数解析:没有代理，没有代码生成，没有反射。Koin是一个DSL、一个轻量级容器和一个实用的API

### Koin DSL

由于 Kotlin 语言的强大功能，Koin 提供了一种 DSL 来帮助您描述应用程序，而不是对其进行注释或生成代码。Koin 提供了Kotlin DSL，它提供了一个智能的函数式API来准备您的依赖注入。

#### Application & Module DSL

Koin提供了几个关键字来描述 Koin 应用程序的元素

- 应用程序 DSL :描述Koin容器配置
- 模块 DSL :用来描述必须注入的组件

#### Application DSL

KoinApplication 实例是 Koin 容器实例配置。这将允许您配置日志记录、属性加载和模块。

要构建一个新的 KoinApplication ，可以使用以下函数:

- `koinApplication { }` 创建一个 `koinApplication` 容器配置
- `startKoin { }` 创建一个 `KoinApplication` 容器配置，并在  `GlobalContext` 中注册它，以允许使用GlobalContext API

要配置 `KoinApplication` 实例，可以使用以下任何函数:

- `logger()` 描述要使用的级别和logger实现(默认使用EmptyLogger)
- `modules()` 设置要加载到容器中的Koin模块列表(列表或vararg列表)
- `properties()`  将 HashMap 属性加载到Koin容器中
- `fileProperties()` 将属性从给定文件加载到Koin容器中
- `environmentProperties()` 将操作系统环境中的属性加载到Koin容器中

### KoinApplication instance: Global vs Local

正如上面所看到的，我们可以用两种方式描述Koin容器配置:koinApplication或startKoin函数。

- `koinApplication` 描述了一个 Koin 容器实例
- `startKoin` 描述了一个 Koin 容器实例，并在 Koin 中的`GlobalContext`中注册
  
通过将容器配置注册到 `GlobalContext` 中，全局API可以直接使用它。任何 `KoinComponent` 都引用Koin实例。默认情况下，我们使用来自 `GlobalContext` 的那个。

有关自定义 Koin 实例的更多信息可以查看相应章节。

### Starting Koin

启动 Koin 意味着在 `GlobalContext` 中运行一个 KoinApplication 实例。

要用模块来启动 Koin 容器，我们可以像这样使用 startKoin 函数:

```kotlin
// 在 GlobalContext 中运行一个 KoinApplication 实例。
startKoin {
    // 声明使用的日志
    logger()
    // 声明使用的模块
    modules(coffeeAppModule)
}
```

### Module DSL

一个 Koin 模块将收集您为应用程序注入/合并的声明。要创建一个新模块，只需使用以下函数:

- module { // module content } - 创建一个 Koin 模块
要描述模块中的内容，可以使用以下函数:

- factory { //definition } - 提供一个工厂对象声明
- single { //definition } - 提供一个单例对象声明(也作为对象别名)
- get() - 解析组件依赖项(也可以使用名称、范围或参数)
- bind() - 为给定的对象声明，添加要绑定的类型
- binds() - 为给定的bean定义添加类型数组
- scope { // scope group } - 为范围声明定义一个逻辑组
- scoped { //definition }- 提供仅存在于 scope 中的对象声明

### Writing a module

Koin模块是用来声明所有组件的空间。使用模块函数声明一个Koin模块:

```kotlin
val myModule = module {
   // 在这里声明你自己的依赖项
}
```

在这个模块中，您可以声明如下所述的组件。

### 最后

站在巨人的肩膀上，才能看的更远~
