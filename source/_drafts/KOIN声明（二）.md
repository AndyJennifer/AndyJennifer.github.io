---
title: KOIN声明(二)
tags:
  - null
categories:
  - null
---

通过使用 Koin，您可以在模块中声明相关定义。在本节中，我们将学习如何声明、组织和链接模块。

### 书写一个模块

一个 Koin 模块是用来声明所有组件的空间。使用 module 函数声明一个 Koin 模块:

```kotlin
val myModule = module {
   // your dependencies here
}
```

在该 module 下，你可以声明所拥有的组件。

### 定义一个单例

声明一个单例组件意味着 Koin 容器将保留您声明的组件的唯一实例。在一个模块中使用一个 single 函数来声明一个单例:

```kotlin
class MyService()

val myModule = module {

    // 声明 MyService 的单例
    single { MyService() }
}
```

### 使用 lambda 声明组件

single、factory 和 scoped 关键字帮助您使用 lambda 表达式声明组件。这个 lambda 描述了您构建组件的方式。通常我们通过它们的构造函数实例化组件，但是您也可以使用任何表达式。

```kotlin
single { Class constructor // Kotlin expression }
```

lambda 的结果类型是你声明的组件的主要类型

### 定义一个工厂对象

一个工厂组件声明，意味着它将在您每次请求此定义时为您提供一个新实例。使用带有 lambda 表达式的 factory 函数来构建组件。

```kotlin
class Controller()

val myModule = module {

    // declare factory instance for Controller class
    factory { Controller() }
}
```

`Koin 容器不保留工厂实例，因为每次请求定义时都会给出一个新实例。`

### 解析和注释依赖项

现在我们可以声明组件定义了，我们想要用依赖注入来链接实例。要解析Koin模块中的实例，只需将get()函数用于请求的所需组件实例。此get()函数通常用于构造函数中，以注入构造函数值。

`要使用Koin容器进行依赖项注入，我们必须以构造函数注入的形式编写它:解析类构造函数中的依赖项。通过这种方式，您的实例将使用来自 Koin 的注入实例创建`。

让我们一下几个类为例：

```kotlin
// Presenter <- Service
class Service()
class Controller(val view : View)

val myModule = module {

    // declare Service as single instance
    single { Service() }
    // 声明一个单例Controller对象， 并通过 get(）函数解析并传入View 对象
    single { Controller(get()) }
}
```

### Definition: binding an interface

一个 single 或者一个 factory 的声明 将会使用其给定的 lambda 表达式中的类型。比如 single{ T }，定义的匹配类型是该表达式中唯一匹配的类型。

让我们以一个类及其实现的接口为例:

```kotlin
// Service interface
interface Service{

    fun doSomething()
}

// Service Implementation
class ServiceImp() : Service {

    fun doSomething() { ... }
}
```

在Koin模块中，我们可以使用 Kotlin 下的 `as` 操作符如下所示:

```kotlin
val myModule = module {

    // 只匹配 Service 类型
    single { ServiceImp() }

    // 只匹配 Service 类型
    single { ServiceImp() as Service }

}
```

你也可以使用推断类型表达式:

```kotlin
val myModule = module {

    // 只匹配 Service 类型
    single { ServiceImp() }

    // 只匹配 Service 类型
    single<Service> { ServiceImp() }

}
```

`第二种方式的风格声明是首选的，在接下来的文章中，我也会使用该方式`

### Additional type binding

在某些情况下，我们希望一个定义匹配多个类型。

让我们举一个类和接口的例子:

```kotlin
// Service interface
interface Service{

    fun doSomething()
}

// Service Implementation
class ServiceImp() : Service{

    fun doSomething() { ... }
}
```

要定义绑定其他类型，我们使用 `bind` 操作符和一个 class 对象:

```kotlin
val myModule = module {

    // 将会匹配 ServiceImp 与 Service 类型
    single { ServiceImp() } bind Service::class
}
```

Note here, that we would resolve the Service type directly with get(). But if we have multiple definitions binding Service, we have to use the bind<>() function.

请注意，我们将使用 get() 直接解析 service 的类型。但是如果有多个定义都绑定了 Service，就必须使用`bind<>()`函数。

### Definition: naming & default bindings

你可以为你的声明指定一个名字，以帮助你区分关于同一类型的两个声明:

只需要求你的声明及其名称:

```kotlin
val myModule = module {
    single<Service>(named("default")) { ServiceImpl() }
    single<Service>(named("test")) { ServiceImpl() }
}

val service : Service by inject(name = named("default"))
```

 `get()` 和`by inject()`函数允许您在需要时指定声明名称。这个名称是`named()`函数生成的限定符。

默认情况下，Koin 将根据类型或名称绑定声明，如果该类型以及绑定到一个声明上

```kotlin
val myModule = module {
    single<Service> { ServiceImpl1() }
    single<Service>(named("test")) { ServiceImpl2() }
}
```

那么

- val service : Service by inject() 将触发 ServiceImpl1 的声明
- val service : Service by inject(named("test")) 将触发 ServiceImpl2 的声明

### Declaring injection parameters

In any single, factory or scoped definition, you can use injection parameters: parameters that will be injected and used by your definition:

```kotlin
class Presenter(val view : View)

val myModule = module {
    single{ (view : View) -> Presenter(view) }
}
```

In contrary to resolved dependencies (resolved with get()), injection parameters are parameters passed through the resolution API. This means that those parameters are values passed with get() and by inject(), with the parametersOf function:

```kotlin
val presenter : Presenter by inject { parametersOf(view) }
```

Further reading in the <<injection-parameters.adoc#injectionparameters,injection parameters section>>.

### Using definition flags

Koin DSL also proposes some flags.

#### Create instances at start

A definition or a module can be flagged as createOnStart, to be created at start (or when you want). First set the createOnStart flag on your module or on your definition.

CreateAtStart flag on a definition

```kotlin
val myModuleA = module {

    single<Service> { ServiceImp() }
}

val myModuleB = module {

    // eager creation for this definition
    single<Service>(createAtStart=true) { TestServiceImp() }
}
```

CreateAtStart flag on a module

```kotlin
val myModuleA = module {

    single<Service> { ServiceImp() }
}

val myModuleB = module(createAtStart=true) {

    single<Service>{ TestServiceImp() }
}
```

The startKoin function will automatically create definitions instances flagged with createAtStart.

```kotlin
// Start Koin modules
startKoin {
    modules(myModuleA,myModuleB)
}
```

`if you need to load some definition at a special time (in a background thread instead of UI for example), just get/inject the desired components.`

### 处理泛型

Koin definitions doesn’t take in accounts generics type argument. For example, the module below tries to define 2 definitions of List:

```kotlin
module {
    single { ArrayList<Int>() }
    single { ArrayList<String>() }
}
```

Koin won’t start with such definitions, understanding that you want to override one definition for the other.

To allow you, use the 2 definitions you will have to differentiate them via their name, or location (module). For example:

```kotliln
module {
    single(named("Ints")) { ArrayList<Int>() }
    single(named("Strings")) { ArrayList<String>() }
}
```
