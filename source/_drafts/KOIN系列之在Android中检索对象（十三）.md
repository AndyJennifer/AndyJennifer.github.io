---
title: KOIN系列之在Android中检索对象（十三）
tags:
  - koin
categories:
  - koin
---

### 前言

一旦你声明了一些模块并启动了 Koin ，那么如何在你的 Android 中的 Activity、Fragment或 Service 中检索你的实例?

### 将 Activity Fragment 及 Service 转换为 KoinComponent

Activity Fragment 及 Service 在 KoinComponent 扩展的基础上进行了延伸。我们有权使用：

- `by inject()` - 来自 Koin 容器的惰性创建（只有被请求的时候，才会创建实例对象）
- `get()` - 从 Koin 容器中立即获取实例
- `release()` - 从它的路径中释放模块的实例
- `getProperty()` / `setProperty()` - 获取/设置属性

一个模块声明了一个 'presenter '组件:

```kotlin
val androidModule = module {
    // 一个 Presenter 工厂对象
    factory { Presenter() }
}
```

我们可以声明一个属性为惰性注入:

```kotlin
class DetailActivity : AppCompatActivity() {

    //惰性注入一个 Presenter 实例对象
    override val presenter : Presenter by inject()

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        // ...
    }
```

或者我们可以直接获取一个实例对象：

```kotlin
class DetailActivity : AppCompatActivity() {

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)

        // 检索出Presenter对象
        val presenter : Presenter = get()
    }
```

### 在你的 API 中没有 inject() 或 get()

如果你希望在其他类中通过 `injcet()` 或 `get()` 方法获取实例对象，那么只需使用 `KoinComponent` 接口标记所需的类。

### 最后

站在巨人的肩膀上，才能看的更远~
