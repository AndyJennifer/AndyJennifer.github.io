---
title: Lambda表达式针对Java
date: 2019-02-23 22:00:51
categories:
- Java语法相关
tags: 
- Lambda
---

{% asset_img 天使爱美丽.jpg 天使爱美丽 %}

我们都知道Java 8 支持Lambda表达式，但是平时开发中也很难用到这个东东，但是作为专业的程序员，技多不压身（其实我是在学Kotlin中，发现里面大量的运用到了Lambda表达式，看的我一脸懵逼，所以只好来学习学习，不然怎么出去装逼，怎么骚浪贱）。好，收，让我们来看看Lambda的前世今生。

### 一、Lambda到底是什么鬼

说到Lambda，不得不说到**函数式编程**，说到函数式编程不得不说到**λ演算**，[λ演算](https://baike.baidu.com/item/%CE%BB%E6%BC%94%E7%AE%97/8019133?fr=aladdin)是数学家**Church**提出的。λ演算中最关键的要素就是**函数被当作变量处理**能够参与运算。（看到这里我社会大佬，才是这些数学家好嘛，打call,打call)。

继续继续，我们来看看Lambda的官方定义。
>A lambda expression is like a method: it provides a list of formal parameters and a body - an expression or block - expressed in terms of those parameters.
>翻译如下：Lambda表达式类似于一种方法：它提供了一个形参列表和一个表达式或用这些形参的代码块。

我去。反正官方的解释我是一点都没有看懂。个人觉得Lambda表达式更像是“语法糖”, 不仅使代码变的更加简洁，还是代码更有阅读性。说了这么多，无码无真相。让我们来探讨探讨Lambda表达式。开车开车，请上车~~~

#### 1.1 Lambda表达式分析

那在Java中如果我们要给一个变量赋值，我们一般的操作如下：

```java
 int a = 1;
 String b = "abc";
 boolean c = true;
```

但是如果我们根据λ演算要素，**将函数当做变量处理**，将函数赋值给一个变量。

```java
 addMethod = public void add(int a,int b){
     return a+b;
 }
```

哇，这代码真是蛇皮，这与我们实际编码中书写的Lambda表达式代码完全不一样好嘛，对啊！作为一个程序员，我们万事都要讲究一个**优雅**好嘛，不优雅，**你就是要我命**，你造否？所以为了代码简洁与优雅。Sun公司的大佬们移除了一些不必要的声明。我们一步一步的来分析。

##### 1.1.1 去除函数修饰符

```java
addMethod = void add(int a,int b){
    return a+b;
}
```

上面操作，我们发现了我们移除了public 修饰符，设想，如果我们需要将一个函数赋值给一个变量，那么起访问权限控制的，肯定是变量的修饰符，所以说函数的修饰符可以移除。我们继续往下走。

##### 1.1.2 去除函数名称

```java
addMethod = void (int a,int b){
    return a+b;
}
```

去除函数名称，主要的原因是，既然我已将函数赋值给变量了。那所有的操作都是与这个变量相关，所以函数的名称是可以不要的。对实际的操作没有影响。

##### 1.1.3 去除函数返回类型

```java
addMethod = (int a,int b){
    return a+b;
}
```

为什么会去掉函数返回类型呢？函数内部到底有没有返回值，你觉得编译器心里难道没有一点B数嘛，所以我们完全可以省略这个返回类型。

##### 1.1.4 去除参数类型

```java
addMethod = (a, b){
    return a+b;
}
```

去掉参数类型，同1.1.2原因，编译器知道传入的参数类型。所以可以省略。

##### 1.1.5 参数与函数体区分

```java
addMethod = (a, b)->{
    return a+b;
}
```

为了更好的将参数与函数体分开。Java中的语法是使用**"->"**来区分，这些代码看起来更简洁。

#### 1.2 Lambda表达式对应变量类型

到现在为止，我们已经成功的将一个函数赋值给一个变量。这个时候我们需要理解的是这个变量的具体类型是什么，我们都知道我们声明变量的时候，前方都会声明其类型。那这个变量的类型到底是什么呢？

在Java8中，**Lambda表达式所对应的类型都是接口**，Lambda所表达的就是那个接口对应函数的具体代码。可能大家这里不是很清楚。概念还是很模糊。我们看一下下面的例子，我相信大家就能清楚的明白了。

```java
 //声明一个Person接口
  public interface Person {
        void smile(int time);
    }
 //在java8以前我们实现接口
 Person p = new Person() {
            @Override
            public void smile(int time) {
                System.out.println("我笑了" + time + "秒");
            }
        };
 //通过Lambda实现接口
Person p = time -> System.out.println("我笑了" + time + "秒");
```

观察上述代码，我们发现。通过Lambda表达式来实现接口，相比之前的实现接口的方式，Lambda表达式是代码更加清晰，且代码量较少。

#### 1.2.1 Lambda表达式表达的方法个数

说到这里，我相信大家都有一个疑问，就是假如Person接口中不止有一个smile方法，还有其他方法呢？那这个时候我该怎么用Lambda表达式来表示呢？ 这个问题其实在最初的Lambda表达式分析那里已经侧面表现了。既然是将函数赋值给变量，那么一个变量对应的函数就只能**有且只有一个**，不然怎么知道你在调用的时候具体调用的哪个函数呢？你说呢，兄die.

说到这里，有的兄die就会说，我怎么才能让我的接口有且只有一个方法呢？,那就引出我们的**@FunctionalInterface**注解，直接通过英文直译的话意思是“函数式接口”。我去，什么鬼，不了解其意思，我们就直接看源码吧。

```java
An informative annotation type used to indicate that an interface
 * type declaration is intended to be a <i>functional interface</i> as
 * defined by the Java Language Specification.
 *
 * Conceptually, a functional interface has exactly one abstract
 * method.  Since {@linkplain java.lang.reflect.Method#isDefault()
 * default methods} have an implementation, they are not abstract.  If
 * an interface declares an abstract method overriding one of the
 * public methods of {@code java.lang.Object}, that also does
 * <em>not</em> count toward the interface's abstract method count
 * since any implementation of the interface will have an
 * implementation from {@code java.lang.Object} or elsewhere.

@Documented
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
public @interface FunctionalInterface {}
```

这里我截取了部分FunctionalInterface的部分注释，**@FunctionalInterface** 注解主要是用于修饰接口。且修饰的接口中有且只有一个方法，到这里我们就明白了，那么我可以用**@FunctionalInterface**注解来修饰我们的接口啦。这个接口是不是很牛逼啊。

### 二、Lambda表达式语法

在了解Lambda的由来，以及表现形式后，我们现在可以来了解一下Lambda的语法啦，知道原理，当然也必须知道怎么使用，对吧。（授人鱼不如授人以渔,你说是不是这个道理，哈哈哈，秀秀我的语文功底）。

- Lambad表达式总是在花括号中；
- 其参数（如果有的话）在 —>**之前**声明（参数类型可以省略）;
- 函数体（如果存在的话）在—>**之后**声明;

基本语法：

```java
(formal parameter list) -> { expression or statements}
```

#### 2.1 Lambda表达式例子

Lambda 表达式实际上是一种匿名方法实现（有的人说是匿名内部类实现，看个人怎么理解了），Lambda表达式的求值不会导致函数体的执行，而是在调用该方法后发生，下面是一些Lambda表达式的例子：

```java
        () -> {}                //无参，空的函数体⑤
        () -> 42                //无参，函数体返回42⑥
        () -> null              // 无参，函数体返回null⑦
        () -> { return 42; }    //  无参，函数体返回42
        () -> { System.gc(); }  // 无参，执行语句，函数体返Void
        () ->  System.gc()  // 无参，执行语句，函数体返Void⑨

        () -> {                 // 带返回语句的完整函数体
            if (true) return 12;
            else {
                int result = 15;
                for (int i = 1; i < 10; i++)
                result *= i;
                return result;
            }
        }

        (int x) -> x+1              // 声明类型的单个参数⑧
        (int x) -> { return x+1; }  // 声明类型的单个参数
        (x) -> x+1                  // 隐藏参数类型的单个参数③
        x -> x+1                    // 花括号可选④


        (String s) -> s.length()      // 声明类型的单个参数
        (Thread t) -> { t.start(); }  // 声明类型的单个参数
        s -> s.length()               // 隐藏参数类型的单个参数
        t -> { t.start(); }           // 隐藏参数类型的单个参数⑩

        (int x, int y) -> x+y  // 声明参数类型的多个参数①
        (x, y) -> x+y          // 隐藏参数类型的多个参数②
        (x, int y) -> x+y    // 错误：不能同时单个的声明类型或隐藏类型
        (x, final y) -> x+y  // 错误：没有推断类型的修饰符
```

相信大家从上面例子中可以看出一下语法规则，那我们将这语法规则分为两个部分**Lambda参数规则**与**lambda函数体规则**。

#### 2.2 Lambda参数规则

- 参数列表必须是以逗号分隔的形式。见①
- 参数列表的参数类型是可选的，如果未指定参数类型，将从上下文进行推断。见②
- 参数列表参数的个数大于2则必须用小括号括起来 。见②
- 参数列表参数的个数为1，则小括号可以省略。见⑩
- 如果函数体中，没有使用参数，那么必须指定空括号。见⑤⑥⑦

#### 2.3 Lambda函数体规则

- 如果函数体只包含单条表达式或语句，就不需要使用大括号。见⑧⑨

### 三、 Lambda使用注意事项

现在我们基本了解了Lambda的使用方式，已经使用场景，那么现在我们看看在Lambda使用中需要注意的问题。

#### 3.1 Lambda访问变量为final类型

我们都知道在匿名内部类中访问变量时，需要将变量修饰为**final**类型。因为如果该该内部类运行在另一线程中，必将会出现线程安全的问题。（这里肯定有很多读者就会提出这样一个问题，那这个内部类修饰变量为**final**和我们Lambda表达式有毛关系啊？）请听我细细道来。

如果你仔细阅读了该文，你应该知道Lambda表达式相当于声明匿名内部类。只是换了一种表达方式而已。那么本质是一样的。所以在访问变量的时候我们也需要注意这些问题。看下面代码。

```java
    public interface A {
        void fuck();
    }

    public void test() {
        int count = 10;
        A a = () -> { count++};//错误 count 类型为final类型。
    }
```

观察上述代码，大家肯定会疑惑，为什么我这里没有申明count为**final**类型。因为在java8中，在编译器编译的时候会将Lambda表达式访问的变量，自动声明为final类型。有点语法糖的味道。

### 总结

通过上文分析。我们发现Lambda表达式不仅使程序变得更加简洁。还能节省程序员的代码量，节省了很多码代码的时间。再有一个我觉得更重要的是，这样代码看起来很骚，你说是吗？

### 参考

站在巨人的肩膀上。可以看得更远。感谢下列博主，和官方文档。人类的知识是大家的。
[Lambda 表达式有何用处？如何使用](https://www.zhihu.com/question/20125256)
[Oracle Lambda官方文档](https://docs.oracle.com/javase/specs/jls/se8/html/jls-15.html#jls-15.27)

最后，附上我写的一个基于Kotlin 仿开眼的项目[SimpleEyes](https://github.com/AndyJennifer/SimpleEyes)(ps: 其实在我之前，已经有很多小朋友开始仿这款应用了，但是我觉得要做就做好。所以我的项目和其他的人应该不同，不仅仅是简单的一个应用。但是，但是。但是。重要的话说三遍。还在开发阶段，不要打我)，欢迎大家follow和start
