---
title: 为什么Java的泛型要用"擦除"实现
tags:
  - 泛型
categories:
  - 泛型
---

### 前言

在 Java 中的 `泛型`，常常被称之为 `伪泛型`，究其原因是因为在实际代码的运行中，将实际类型参数的信息擦除掉了[(Type Erasure)](https://docs.oracle.com/javase/tutorial/java/generics/erasure.html)。那是什么原因导致了 Java 做出这种妥协的呢？下面我就带着大家以 Java 语言设计者的角度，带领大家一起了解这里面的辛酸过往。

### 什么是真泛型

在了解 Java "伪泛型" 之前，我们先简单讲一讲"真泛型"与“伪泛型”的区别。

- 真泛型：泛型在程序运行期间是真实存在的。
- 伪泛型：擦除式方式实现，仅用于编译时类型检查，在运行时擦除。

### 编程语言实现“真泛型”的一般思路

一般编程语言引入泛型的思路，都是通过**编译时膨胀法**。

以 Java 语言实现"真泛型"为例，首先，对`泛型类型`、`方法`等的名字使用特别的编码，例如说将 `Factory<T>` 类生成为一个名为 `“Factory@@T”` 的类，这种特别的编码后的名字将被编译器识别，作为判断是否为泛型的依据。方法中使用占位符 `T` 的地方可以临时生成为 `Object` ，并带上特别的 `Annotation` 让 Java 编译器能得知这个是占位符。然后，如果编译时发现有对 `Factory<String>` 的使用，则将 `“Factory@@T”` 的所有逻辑复制一份，新建 `“Factory@String@”` 类，将原本的占位符 `T` 替换为 `String` 。然后在编译 `new Factory<String>()` 时生成 `new Factory@String@()` 即可。

> 泛型类、泛型接口统称为`泛型类型`

以 `Factory<String>` 与 `Factory<Integer>` 为例，其生成的代码为：

```java
//替换前的代码
Factory<String>() f1 = new Factory<String>();
Factory<Integer>() f2 = new Factory<Integer>();

//替换后的代码
Factory<String>() f1 = new Factory@String@()
Factory<Integer>() f2 = new Factory@Integer@();
```

- 其中 `Factory@String@` 类中的 `T` 已经被替换为 String。
- 其中 `Factory@Integer@()` 类中的 `T` 已经被替换为 Integer。

因为含有不同的 `实际类型参数` 的 `泛型类型` 都被替换为了不同的类，且泛型类型中的类型也都得到了确认。所以我们在程序中可以这么做：

```java
class Factory<T> {
    T data;
    public static void f(Object arg){
        if(arg instanceof T){}//success
        T var = new T();//success
        T[] array = new T[10];//success
    }
}
```

>|术语|中文含义|举例|
>|:--:|:--:|:--:|
>|Parameterized type|参数化类型|`List<String>`|
>|Actual type parameter|实际类型参数|`String`|
>|Formal type parameter|形式类型参数|`E`|

### Java 直接使用 “真泛型” 带来的问题

单从技术来说，Java 是完全 `100%` 能实现我们所说的 ”真泛型”。想要实现真泛型，有如下两件事Java 必须要处理：

- 修改 JVM 源代码，让 JVM 能正确的的读取和校验泛型信息。（之前的Java 是没有泛型这种概念的，所以需要修改）。
- 为了兼容老程序，需为原本不支持泛型的 API 平行添加一套泛型 API，主要是容器类型。

就拿 ArrayList 来说，也就是必须这么做：

- java.util.ArrayList 👈 Java 老版本
- java.util.generic.`ArrayList<T>` 👈 Java5 新增泛型版本

即使以上的事情都做了，Java 也并不能采用这种方案。试想如下情况：

![如何兼容老版本.png](https://upload-images.jianshu.io/upload_images/2824145-a1dbd4ff25490b37.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

如果我有一个 `Java 5 之下` 的 A 项目与第三方的 B1 lib，其中有 A 项目中引用了 B1 lib 中的某个 ArrayList ，随着 Java 的升级，B1 lib 的开发者为了使用 Java 新特性--泛型，故将代码迁移到了 Java 5，并重新生成了 B2 lib，那么 A 项目要兼容 B2 lib，那么 A 项目中必须升级到 Java 5 并同时修改代码。

A 项目为什么必须要升级 Java 5？

在 Java 中不支持高版本的 Java 编译生成的 class 文件在低版本的 JRE 上运行，如果尝试这么做，就会得到 [UnsupportedClassVersionError](https://docs.oracle.com/javase/8/docs/api/java/lang/UnsupportedClassVersionError.html) 错误。如下图所示：

![版本说明.png](https://upload-images.jianshu.io/upload_images/2824145-c502d0f35d47fa8e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

故 A 项目要适配 B2 lib，必要要把 Java 升级到 Java 5。

那现在我们再回过头来想想，Java 版本迭代都从 1.0 到 5.0了，有多少的开源框架，有多少项目，如果为了引入泛型，强行让开发者修改代码。这种情况，各位同学。自行脑补。估计数以万计的开发者拿着刀，在堵 Java 语言架构师的门吧。

### 逼不得已的类型擦除

在上节中，我们探讨了 Java 不能直接引入“真泛型” 的实际原因。因为“真泛型”的引入，势必会为原本不支持泛型的 API 平行添加一套泛型 API。而新增了API，对于 Java 开发者来说，又必须要做迁移。

那还有什么方案，能让开发者平滑的过渡到 Java 5， 又能让开发者使用泛型新特性呢？

![抽烟分析.jpg](https://upload-images.jianshu.io/upload_images/2824145-fbea75c0f7ae866e.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

有的，有的。Java 如果想摆脱用户新版本的迁移问题。Java 必要要做以下两件事情：

- 不再新增一套泛型 API，**直接把已有的类型原地泛型化**。
- 处理泛化前后类型的兼容。

下面，我们就来分别探讨一下这两件事做的目的与原因。

做第一件事，是保证了开发者不会因为 Java 的升级，而对以前的老代码进行修改，以 ArrayList 为例，**直接在原有包(java.util)下进行修改**，也就是这样：

```java
//👇Java老版本
class ArrayList{}

//👇Java5泛型版本
class ArrayList<T>{}
```

做第二件事的目的，还是以我们之前的例子进行分析，在 A 项目中，A 项目引用了 B1 lib 中的 ArrayList（用 list 变量记录），那么假设 A 项目升级到 Java 5后，还是引用的 B1 lib，那么必然会出现如下这种情况：

```java
ArrayList list = new ArrayList<String>();
```

- 左边：B1 lib 中的老版本 ArrayList
- 右边：A 项目 中的新泛型版本 `ArrayList<T>`

上述情况的出现，会导致一个问题。就是 b1 项目中的 ArrayList 是不知道 A 项目中的 Arraylist 已经泛型化了的，那么如何保证**泛型化后**的 ArrayList（`也就是ArrayList<T>`）与**老版本**的 ArrayList 等价呢？

如果按照我们之前讲解的 “真泛型” 思路， `new ArrayList<String>()` 实际会被替换为 `new ArrayList@String@()`，那么实际运行时的代码是这样：

```java
ArrayList list = new Factory@String@()
```

从代码逻辑上来看，根本就跑不通。因为 ArrayList 与 ArrayList@String@ 根本就不是同一类型， 所有 Java 是没有办法采用真泛型的。那怎么办呢？

最为直接的解决方案就是，不再为参数化类型创造新类了，同时在编译期间将`泛型类型`中的`类型参数`全部替换 `Object`（因为不创建新类了，那么在泛型类中的 T 对应的类型，只能用 Object 替换）。举个例子：

>在 Java 的实际实现中，会根据泛型类型中的类型参数有无边界，来选择是否替换为为其边界或 Object。这里只是方便大家理解。

声明了一个泛型类型 Node:

```Java
public class Node<T> {

    private T data;
    private Node<T> next;

    public Node(T data, Node<T> next) {
        this.data = data;
        this.next = next;
    }

    public T getData() { return data; }
    // ...
}
```

编译器将类型参数 `T` 替换为 Object 后：

```java
public class Node {

    private Object data;
    private Node next;

    public Node(Object data, Node next) {
        this.data = data;
        this.next = next;
    }

    public Object getData() { return data; }
    // ...
}
```

也就是说

```java
 //替换前
 Node node =new Node<String> ();
 //替换后
 Node node =new Node ();
```

那么这样就可以保证等价性了。

为了解决上述问题。Java 提供了一套”类型擦除“的方案:

1. 不再为参数化类型创建新类，同时将泛型类型中的类型参数全部替换为其边界或 Object。因此生成的字节码中只包含普通类、接口和方法。
2. 必要时插入类型转换以保持类型安全。
3. 生成桥方法以保留扩展泛型类型中的多态性。

```Java
public class Node<T> {

    private T data;
    private Node<T> next;

    public Node(T data, Node<T> next) {
        this.data = data;
        this.next = next;
    }

    public T getData() { return data; }
    // ...
}
```

由于类型参数 `T` 是无边界的的，Java编译器将其替换为 Object：

```java
public class Node {

    private Object data;
    private Node next;

    public Node(Object data, Node next) {
        this.data = data;
        this.next = next;
    }

    public Object getData() { return data; }
    // ...
}
```

在接下来的例子中，泛型 Node 类将使用带有边界的类型参数:

```java
public class Node<T extends Comparable<T>> {

    private T data;
    private Node<T> next;

    public Node(T data, Node<T> next) {
        this.data = data;
        this.next = next;
    }

    public T getData() { return data; }
    // ...
}
```

Java 编译器将有界类型参数 T 替换为第一个边界，Comparable:

```java
public class Node {

    private Comparable data;
    private Node next;

    public Node(Comparable data, Node next) {
        this.data = data;
        this.next = next;
    }

    public Comparable getData() { return data; }
    // ...
}
```

### Java 泛型的未来的展望与规划

通过整篇文章，相信大家已经了解了 Java 的泛型为什么要采用擦除的方式来实现了。可能有很多小伙伴对 Java 何时能够实现真正的泛型而心存疑惑。这里我想表达一下个人的观点，Java 是没有办法做到完全的 ”真泛型“ 的，但是 Java 也在努力的做一些部分的”真泛型“的工作。如 [Generics over Primitive Types](http://openjdk.java.net/jeps/218)

### 总结

[如果Java在全世界突然被禁用会怎样](https://www.bilibili.com/video/av95634813)

## 最后

站在巨人的肩膀上，才能看的更远~

- https://www.cnblogs.com/leyangzi/p/11379525.html
- https://juejin.im/post/5d6c6636f265da03c8153a03
- http://java.sun.com/docs/books/tutorial/java/generics/erasure.html
- http://cr.openjdk.java.net/~briangoetz/valhalla/specialization.html
- https://www.zhihu.com/question/28665443/answer/118148143
- https://docs.oracle.com/javase/tutorial/java/generics/index.html
- 《Think in Java》
- 《Effective Java》
- http://blog.zhaojie.me/2010/02/why-not-csharp-on-jvm-type-erasure.html#comment_iX2wuQ8q04I0057D
- 
https://blog.csdn.net/qq_36761831/article/details/82289786?depth_1-utm_source=distribute.pc_relevant.none-task-blog-BlogCommendFromMachineLearnPai2-4&utm_source=distribute.pc_relevant.none-task-blog-BlogCommendFromMachineLearnPai2-4


