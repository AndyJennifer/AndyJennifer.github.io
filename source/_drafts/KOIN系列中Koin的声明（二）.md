---
title: KOIN系列之Koin的声明（二）
tags:
  - koin
categories:
  - koin
---

通过使用 Koin，您可以在模块中声明相关定义。在本节中，我们将学习如何定义、组织和链接模块。

### 书写一个模块

一个 Koin 模块是用来声明所有组件的空间。使用 module 函数声明一个 Koin 模块:

```kotlin
val myModule = module {
   // your dependencies here
}
```

在该 module 下，你可以声明所拥有的组件。

### 定义一个单例

声明一个单例组件意味着 Koin 容器将保留该组件的唯一实例。在一个模块中使用一个 `single` 函数来声明一个单例:

```kotlin
class MyService()

val myModule = module {

    // 声明 MyService 的单例
    single { MyService() }
}
```

### 使用 lambda 声明组件

single、factory 和 scoped 关键字允许您使用 lambda 表达式声明组件。这个 lambda 描述了您构建组件的方式。通常我们通过它们的构造函数实例化组件，但是您也可以使用任何表达式。

```kotlin
single { Class constructor // Kotlin expression }
```

lambda 的返回的结果的类型就是你声明的组件的主要类型

### 定义一个工厂对象

声明一个工厂组件，意味这将在您每次请求该对象时为您提供一个新的实例对象（该实例对象不会被 Koin 容器所保留，也不会因为其他的声明被注入）。使用带有 lambda 表达式的 factory 函数来构建组件。

```kotlin
class Controller()

val myModule = module {

    // declare factory instance for Controller class
    factory { Controller() }
}
```

`因为每次请求定义时都会给出一个新实例，所以 Koin 容器不保留工厂实例，`

### 解析和注释依赖项

现在我们可以声明组件的定义了，我们想要用依赖注入来链接实例。要解析Koin模块中的实例，只需使用`get()` 函数用于请求的所需组件实例。此 `get()`函数通常用于构造函数中，以便注入构造函数值。

`要使用Koin容器进行依赖项注入，我们必须以构造函数注入的形式编写它:通过解析类构造函数中的依赖项，该构造函数中的参数对象实例 将会通过 Koin 的注入的方式创建`。

让我们以下几个类为例：

```kotlin
// Presenter <- Service
class Service()
class Controller(val view : View)

val myModule = module {

    // 声明一个 Service单例
    single { Service() }
    // 声明一个单例Controller对象， 并通过 get(）函数解析并传入View 对象
    single { Controller(get()) }
}
```

### 绑定一个接口

一个 single 或者 factory 对象的声明，将会使用其给定的 lambda 表达式中的类型。比如 single{ T }，该对象所匹配的类型就是表达式所声明的类型 T

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

`第二种方式的风格声明是首选的，在接下来的文档中，也会使用该方式`。

### 额外类型绑定

在某些情况下，我们希望一个声明匹配多个类型。

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

请注意，这里我们可以直接使用 get() 函数 解析到 Service 的类型。但是如果有多个对象都绑定了 Service，就必须使用`bind<>()`函数。

### 命名及默认绑定

你可以为你的定义指定一个名字，以帮助你区分同一类型的两个对象:

只需要求你声明对象是添加相应名称:

```kotlin
val myModule = module {
    single<Service>(named("default")) { ServiceImpl() }
    single<Service>(named("test")) { ServiceImpl() }
}

val service : Service by inject(name = named("default"))
```

 `get()` 和`by inject()` 函数允许您在需要时指定对象的名称。这个名称是`named()`函数生成的限定符。

如果该类型以及绑定到一个对象上，默认情况下，Koin 将根据类型或名称绑定对象，

```kotlin
val myModule = module {
    single<Service> { ServiceImpl1() }
    single<Service>(named("test")) { ServiceImpl2() }
}
```

那么

- `val service : Service by inject()` 将触发 ServiceImpl1 的声明
- `val service : Service by inject(named("test"))` 将触发 ServiceImpl2 的声明

### 声明注入参数

在任何 `single`、`factory` 或 `scoped` 的定义中，都可以使用注入参数：这些参数将会通过你的定义被注入以及使用。

```kotlin
class Presenter(val view : View)

val myModule = module {
    single{ (view : View) -> Presenter(view) }
}
```

In contrary to resolved dependencies (resolved with get()), injection parameters are parameters passed through the resolution API. This means that those parameters are values passed with get() and by inject(), with the parametersOf function:

与解析依赖项（使用 `get（）`解析）相反，注入参数是通过解析API传递的参数。这意味着这些参数是通过`get（）`和 `inject（）`以及 `parametersOf` 函数传递的值。

```kotlin
val presenter : Presenter by inject { parametersOf(view) }
```

### 使用标志位

Koin DSL 也提供了一些标志位。

#### 在开始的时候创建实例

可以将对象或模块标记为 `createOnStart` ，以便在开始时（或在需要时）创建。首先在模块或对象声明上设置createOnStart标志。

在对象中声明一个 `createOnStart` 标志位

```kotlin
val myModuleA = module {

    single<Service> { ServiceImp() }
}

val myModuleB = module {

    // 在当前对象声明的时候已经创建
    single<Service>(createAtStart=true) { TestServiceImp() }
}
```

在 modle 中声明一个 `createOnStart` 标志位

```kotlin
val myModuleA = module {

    single<Service> { ServiceImp() }
}

val myModuleB = module(createAtStart=true) {

    single<Service>{ TestServiceImp() }
}
```

`startKoin` 函数将自动创建含有`createAtStart`标记的对象以及modle。

```kotlin
// Start Koin modules
startKoin {
    modules(myModuleA,myModuleB)
}
```

`如果需要在特定时间加载某些对象或 module （例如在后台线程而不是UI线程中）`，只需使用get/inject函数来注入所需的组件。

### 处理泛型

Koin definitions doesn’t take in accounts generics type argument. For example, the module below tries to define 2 definitions of List:

Koin 不接受多种泛型类型参数。例如，在下面的模块中尝试定义两个List对象

```kotlin
module {
    single { ArrayList<Int>() }
    single { ArrayList<String>() }
}
```

Koin won’t start with such definitions, understanding that you want to override one definition for the other.

Koin 是不允许这样定义的，而是您希望为另一个定义重写一个定义。

为了允许您使用这两个定义，您必须通过它们的名称或位置（module）来区分它们。例如：

```kotliln
module {
    single(named("Ints")) { ArrayList<Int>() }
    single(named("Strings")) { ArrayList<String>() }
}
```
