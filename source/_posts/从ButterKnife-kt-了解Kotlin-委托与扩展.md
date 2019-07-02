---
title: 从ButterKnife.kt 了解Kotlin 委托与扩展
date: 2019-02-23 22:17:17
tags:
- kotlin
---

 最近学了**Kotlin**这门新语言用于Android开发。其中不免遇上**findViewById()**,原来自己都是通过ButterKnife 直接进行View的查找。所有难免想用Kotlin的实现。通过网上查找，发现JakeWharton大神已经实现[kotterknife](https://github.com/JakeWharton/kotterknife)。所以我们就直接来分析了。ps:(本文章只讨论基于Kotlin实现的findViewByid,不讨论Anko库)
   
## ButterKnife的使用

```
private val mIcon by bindView<ImageView>(R.id.iv_image)
```
通过以上代码，我们发现。只需要通过 by bindView<View类型>（id）就能获得我们需要找到的View,那其中涉及到的知识点是哪几个呢？

主要知识点
* [委托属性](https://www.kotlincn.net/docs/reference/delegation.html)
* [扩展函数](https://www.kotlincn.net/docs/reference/extensions.html)
* [高阶函数](https://www.kotlincn.net/docs/reference/lambdas.html)

如果你已经对这几个知识点都掌握了，那么就直接开始分析代码，查看bindView方法
```
public fun <V : View> View.bindView(id: Int)
    : ReadOnlyProperty<View, V> = required(id, viewFinder)
public fun <V : View> Activity.bindView(id: Int)
    : ReadOnlyProperty<Activity, V> = required(id, viewFinder)
public fun <V : View> Dialog.bindView(id: Int)
    : ReadOnlyProperty<Dialog, V> = required(id, viewFinder)
public fun <V : View> DialogFragment.bindView(id: Int)
    : ReadOnlyProperty<DialogFragment, V> = required(id, viewFinder)
public fun <V : View> SupportDialogFragment.bindView(id: Int)
    : ReadOnlyProperty<SupportDialogFragment, V> = required(id, viewFinder)
public fun <V : View> Fragment.bindView(id: Int)
    : ReadOnlyProperty<Fragment, V> = required(id, viewFinder)
public fun <V : View> SupportFragment.bindView(id: Int)
    : ReadOnlyProperty<SupportFragment, V> = required(id, viewFinder)
public fun <V : View> ViewHolder.bindView(id: Int)
    : ReadOnlyProperty<ViewHolder, V> = required(id, viewFinder)
```
我们发现该类中对常用的View、Activity、Dailog、Fragment等都扩展了一个bindView的方法，而其返回值都是Kotlin中标准库中的ReadyOnlyProperty接口。而具体实现是required(id,viewFinder)方法。

而ViewFinder又是什么呢？我们继续查看viewFinder
```
private val View.viewFinder: View.(Int) -> View?
    get() = { findViewById(it) }
private val Activity.viewFinder: Activity.(Int) -> View?
    get() = { findViewById(it) }
private val Dialog.viewFinder: Dialog.(Int) -> View?
    get() = { findViewById(it) }
private val DialogFragment.viewFinder: DialogFragment.(Int) -> View?
    get() = { dialog?.findViewById(it) ?: view?.findViewById(it) }
private val SupportDialogFragment.viewFinder: SupportDialogFragment.(Int) -> View?
    get() = { dialog?.findViewById(it) ?: view?.findViewById(it) }
private val Fragment.viewFinder: Fragment.(Int) -> View?
    get() = { view.findViewById(it) }
private val SupportFragment.viewFinder: SupportFragment.(Int) -> View?
    get() = { view!!.findViewById(it) }
private val ViewHolder.viewFinder: ViewHolder.(Int) -> View?
    get() = { itemView.findViewById(it) }
```
我们发现ViewFinder只不过是我们常用的View、Activity、Dailog、Fragment等的扩展属性 同时该属性拥有函数类型 T.()->View，并且ViewFinder最终的实现是其相应接受者的findViewById(id)函数。


我们继续点击required方法

```
@Suppress("UNCHECKED_CAST")
private fun <T, V : View> required(id: Int, finder: T.(Int) -> View?)
    = Lazy { t: T, desc -> t.finder(id) as V? ?: viewNotFound(id, desc) }
```
我们发现required方法返回的是一个Lazy对象，并且初始化了其initializer 属性。根据id寻找相应的View.如果没有找到就执行相应的viewNotFound方法。

可能有些小朋友看不懂这段代码。那么其完整格式为：
```
Lazy( { t: T, desc -> t.finder(id) as V? ?: viewNotFound(id, desc) })
```
在Kotlin中有一个约定，如果函数的最后一个参数是一个函数。并且你传递一个Lambda表达式作为相应的参数，你可以在圆括号外面声明它

转换后:
```
Lazy(){ t: T, desc -> t.finder(id) as V? ?: viewNotFound(id, desc) }
```
**注意**：如果Lambdashi 该调用的唯一参数。则调用中的圆括号可以省略
```
 Lazy { t: T, desc -> t.finder(id) as V? ?: viewNotFound(id, desc) }
```

接下来继续查看Lazy类:
```
// Like Kotlin's lazy delegate but the initializer gets the target and metadata passed to it
private class Lazy<T, V>(private val initializer: (T, KProperty<*>) -> V) : ReadOnlyProperty<T, V> {
  private object EMPTY
  private var value: Any? = EMPTY

  override fun getValue(thisRef: T, property: KProperty<*>): V {
    if (value == EMPTY) {
      value = initializer(thisRef, property)
    }
    @Suppress("UNCHECKED_CAST")
    return value as V
  }
}
```
通过观察我们发现该类实现了**ReadOnlyProperty**接口。我们知道在类委托中。可以实现**ReadOnlyProperty**(针对于val)或**ReadWriteProperty**(针对于var)接口。Lazy实现了ReadOnlyProperty那么在属性委托的时候会调用其**getValue**方法 (**第一个参数**：当前调用对象。**第二个参数**。当前属性的描述)，在getValue方法中，会调用其 initializer属性的对应的函数作为最终的返回值。

最终。整个流程分析完毕。我们可以发现Kotlin这门语言的强大之处。给我们编码带来了许多方便。还不快来一起学习Koltin。
	
最后，附上我写的一个基于Kotlin 仿开眼的项目[SimpleEyes](https://github.com/AndyJennifer/SimpleEyes)(ps: 其实在我之前，已经有很多小朋友开始仿这款应用了，但是我觉得要做就做好。所以我的项目和其他的人应该不同，不仅仅是简单的一个应用。但是，但是。但是。重要的话说三遍。还在开发阶段，不要打我)，欢迎大家follow和start
