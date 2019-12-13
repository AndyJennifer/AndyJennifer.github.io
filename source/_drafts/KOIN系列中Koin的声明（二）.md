---
title: KOIN系列之Koin的声明（二）
tags:
  - koin
categories:
  - koin
---

通过使用 Koin，您可以在模块中声明相关定义。在本节中，我们将学习如何定义、组织和链接模块。

### 书写一个模块

一个 Koin 模块是用来声明你组件的一片空间。使用 module 函数声明一个 Koin module:

```kotlin
val myModule = module {
   // your dependencies here
}
```

在该 module作用域下，你可以声明所拥有的组件。

### 定义一个单例

声明一个单例组件意味着 Koin 容器将保留该组件的唯一实例。在一个模块中使用一个 `single` 函数来声明一个单例:

```kotlin
class MyService()

val myModule = module {

    // 声明 MyService 的单例
    single { MyService() }
}
```

### 使用lambda表达式声明组件

single、factory 和 scoped 关键字允许您使用 lambda 表达式来声明组件。这个 lambda表达式描述了你构建组件的方式。通常我们通过它们的构造函数来实例化组件，但是您也可以使用任意表达式

```kotlin
single { Class constructor // Kotlin expression }
```

lambda 表达式返回的类型就是你声明的组件的主要类型

### 定义一个工厂对象

声明一个工厂组件，意味这将在您每次请求该对象时，Koin 容器将会每次都为你提供一个新的实例对象（该实例对象不会被 Koin 容器所保留，也不会因为随后的其他的声明而被注入）。

使用带有 lambda 表达式的 factory 函数来构建组件。

```kotlin
class Controller()

val myModule = module {

    // declare factory instance for Controller class
    factory { Controller() }
}
```

`因为每次请求 factory 定义时都会给出一个新实例，所以 Koin 容器不保留工厂实例对象，`

### 解析和注入依赖项

如果我们想通过依赖注入的方式来链接实例。那么我们只需在需要请求实例的组件中调用 `get()` 函数来解析 Koin module 中的声明，`get()` 函数通常用于构造函数中，以便注入值。

`为了在 Koin 容器中进行依赖项注入，我们必须以构造函数注入的方式来编写它:解析了类中构造函数的依赖。这样，Koin 将会以注入的方式创建你的对象`。

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

### 声明：绑定一个接口

一个 single 或者 factory 声明将会使用其给定的 lambda 表达式中的类型。比如 single{ T }，该声明所匹配的类型就是表达式所声明的类型 T

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

在 Koin 模块(module)中，我们可以使用 Kotlin 下的 `as` 操作符如下所示:

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

请注意，这里我们可以直接使用 get() 函数 解析 Service 的类型。但是如果有多个声明都绑定了 Service类型，就必须使用`bind<>()`函数。

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

 `get()` 和 `by inject()` 函数允许您在需要时指定对象的名称。这个名称是 `named()` 函数生成的 `qualifer`对象。

如果该类型已经绑定到一个对象上，默认情况下，Koin 将根据类型或名称绑定声明。

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

在任何 `single`、`factory` 或 `scoped` 的声明中，都可以使用注入参数：这些参数将会通过你的声明被注入以及使用。

```kotlin
class Presenter(val view : View)

val myModule = module {
    single{ (view : View) -> Presenter(view) }
}
```

与解析依赖项（使用 `get()`解析）相反，参数的注入是通过解析 API 传递的参数。这意味着这些参数的值是通过 `get()`和 `by inject()` 使用 `parametersOf` 函数传递的。

```kotlin
val presenter : Presenter by inject { parametersOf(view) }
```

### 使用声明标志

在 Koin DSL 中也提供了一些标志位。

#### 在开始的时候创建实例

可以将一个声明或一个模块标记为 `createOnStart` ，以便在开始时（或在需要时）创建。首先，在你的模块或对象声明上设置 `createOnStart` 标志。

在一个声明中添加 `createOnStart` 标志位

```kotlin
val myModuleA = module {

    single<Service> { ServiceImp() }
}

val myModuleB = module {

    // 该声明
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

`startKoin` 函数将自动创建含有 `createAtStart` 标记的对象以及modle。

```kotlin
// Start Koin modules
startKoin {
    modules(myModuleA,myModuleB)
}
```

`如果需要在特定时间加载某些声明（例如在后台线程而不是UI线程中）`，只需在所需的组件调用get/inject

### 处理泛型

Koin 不接受多种泛型类型参数。例如，在下面的模块中尝试定义两个List对象

```kotlin
module {
    single { ArrayList<Int>() }
    single { ArrayList<String>() }
}
```

Koin 是不允许这样定义的，而是您希望为另一个定义重写一个定义。

为了允许您使用这两个定义，您必须通过它们的名称或位置（module）来区分它们。例如：

```kotliln
module {
    single(named("Ints")) { ArrayList<Int>() }
    single(named("Strings")) { ArrayList<String>() }
}
```
