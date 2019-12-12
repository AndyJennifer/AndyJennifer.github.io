---
title: KOIN系列之Module是什么（三）
tags:
  - koin
categories:
  - koin
---

通过使用Koin，您可以在模块中描述相关声明。在本节中，我们将学习如何声明、组织和链接模块。

### module 是什么

Koin 模块（module) 是用来收集 Koin 声明的“空间”。使用 `module` 函数描述相关模块

```kotlin
val myModule = module {
    // Your definitions ...
}

```

### 使用多个 module

组件不一定要在同一个 module 中。module 是帮助您组织定义与声明的一块逻辑空间，并且可以依赖于其它 module 的定义。这种定义是懒加载的，只有在组件请求它时才会解析。

让我们举个例子，在不同的模块中链接组件:

```kotlin
// ComponentB <- ComponentA
class ComponentA()
class ComponentB(val componentA : ComponentA)

val moduleA = module {
    // Singleton ComponentA
    single { ComponentA() }
}

val moduleB = module {
    // Singleton ComponentB with linked instance ComponentA
    single { ComponentB(get()) }
}

```

`Koin 没有任何重要的概念。Koin 中的声明是懒加载的:一个 Koin 声明是随着 Koin 容器启动的，但是该声明并没有实例化。只有完成了对其类型的请求时，才会被创建`

当我们启动Koin容器时，我们只需要声明使用的模块列表:

```kotlin
// Start Koin with moduleA & moduleB
startKoin{
    modules(moduleA,moduleB)
}
```

Koin 将解析所有给定模块的依赖关系。

### 链接 module 的策略

由于模块之间的定义是懒加载的，我们可以使用模块实现不同的策略实现:每个模块中声明一个组件的实现。

```kotlin
class Repository(val datasource : Datasource)
interface Datasource
class LocalDatasource() : Datasource
class RemoteDatasource() : Datasource
```

我们可以在3个模块中声明这些组件: Repository 与每个 DataSource 中各实现一个:

```kotlin
class Repository(val datasource : Datasource)
interface Datasource
class LocalDatasource() : Datasource
class RemoteDatasource() : Datasource
```

然后我们只需要将正确的模块组合，并启动 Koin。

```kotlin
// 加载 Repository + Local Datasource definitions
startKoin {
    modules(repositoryModule,localDatasourceModule)
}

// 加载 Repository + Remote Datasource definitions
startKoin {
    modules(repositoryModule,remoteDatasourceModule)
}
```

### 覆盖声明或者 module

Koin 不允许您重复声明已经存在的声明(类型、名称、路径……)。如果你尝试这样做，那么就会得到一个错误：

```kotlin
val myModuleA = module {

    single<Service> { ServiceImp() }
}

val myModuleB = module {

    single<Service> { TestServiceImp() }
}

// 将会抛出一个 BeanOverrideException 异常
startKoin {
    modules(myModuleA,myModuleB)
}
```

为了覆盖声明，你可以使用 `override` 参数：

```kotlin
val myModuleA = module {

    single<Service> { ServiceImp() }
}

val myModuleB = module {

    //覆盖声明
    single<Service>(override#true) { TestServiceImp() }
}
```

```kotlin
val myModuleA = module {

    single<Service> { ServiceImp() }
}

// 允许覆盖模块中的所有声明
val myModuleB = module(override#true) {

    single<Service> { TestServiceImp() }
}
```

`在列出模块和覆盖声明时，顺序非常重要。您必须在模块列表的最后一个module中包含相关覆盖声明。`

### 最后

站在巨人的肩膀上，才能看的更远~
