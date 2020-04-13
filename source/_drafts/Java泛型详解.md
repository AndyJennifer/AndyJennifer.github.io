---
title: Java泛型详解
tags:
  - null
categories:
  - null
---

## 前言

好像泛型一直是比较难啃的骨头，本文针对那些对泛型有疑惑，

- 解决泛型的错误认知。
- 会简单的使用泛型，但是不知道怎么约束。
- 泛型的内部实现原理

## Java中泛型的实现

在 Java 中泛型的实现，主要分为如下三个步骤：

- 将泛型类型中的类型参数信息擦除，使泛型类型转换为原始类型。
- 将类型参数擦除到它的第一个边界
- 对泛型类型传递出去的值进行自动转型。

下面会对这三个步骤进行详细的讲解。

### 类型擦除

这里所指的”类型擦除“其实是指在 Java 编译器在`编译期间`将`泛型类型`中的类型参数信息擦除，使 `泛型类型` 转换为 `原始类型` 。

>`原始类型(Raw type)`是指 `泛型类型`擦除了`类型参数信息`，最后在字节码中的类型变量的真正类型。如 `List<Object>` 与 `List<String>` 其原始类型为 `List`。

我们一起来看下面的例子：

```java
    List<String> list = new ArrayList<String>();
    List<Integer> list2 = new ArrayList<Integer>();
    System.out.println(list.getClass() == list2.getClass());//输出 true
```

在上面例子中，我们定义了两个 ArrayList 集合，一个是 `ArrayList<String>`，只能存储字符串。一个是 `ArrayList<Integer>`，只能存储整形数据。分别调用其对象的 `getClass()` 方法，并比对其 Class 对象，我们能发现其输出结果为 `true`。 也就证明了泛型类型最终会被擦除到原始类型。

### 类型参数擦除

在 Java 泛型的实现中，除了将泛型类型擦除到原始类型以外，还会将类型参数擦除到它的第一个边界。请主要场景分，非限制类型擦除与限制类型擦除。

#### 非限制类型擦除

类型参数在未指定边界的情况下，将被擦除为Object

```java
public class Demo<T> {
    private T data;

    public void setData(T data) {
        this.data = data;
    }

    public T getData() {
        return data;
    }
}
```

通过 `javap -c Demo.class` 命令，查看字节码信息：

```java
  public re.test.test1.Demo();
    Code:
       0: aload_0
       1: invokespecial #1                  // Method java/lang/Object."<init>":()V
       4: return

  public void setData(T);
    Code:
       0: aload_0
       1: aload_1
       2: putfield      #2                  // Field data:Ljava/lang/Object;
       5: return

  public T getData();
    Code:
       0: aload_0
       1: getfield      #2                  // Field data:Ljava/lang/Object;
       4: areturn
}
```

在生成的class文件中，
Code 所指示内容为方法的实际字节码指令
putfield:设置对象实例中的字段。
getfield：获取对象实例中的字段。

我们能看到所引用的字段类型为：java/lang/Object

介绍一下字节码加载指令

#### 限制类型擦除

单个边界：

```java
public class Demo<T extends String> {
    private T data;

    public void setData(T data) {
        this.data = data;
    }

    public T getData() {
        return data;
    }
}
```

```java
 public re.test.test1.Demo();
    Code:
       0: aload_0
       1: invokespecial #1                  // Method java/lang/Object."<init>":()V
       4: return

  public void setData(T);
    Code:
       0: aload_0
       1: aload_1
       2: putfield      #2                  // Field data:Ljava/lang/String;
       5: return

  public T getData();
    Code:
       0: aload_0
       1: getfield      #2                  // Field data:Ljava/lang/String;
       4: areturn
}

```

存储的其实是String

多个边界：

<T extends String & Runnable> 擦除到String，举例说明

```java
public class Demo<T extends String & Runnable> {

    private T data;

    public void setData(T data) {
        this.data = data;
    }

    public T getData() {
        return data;
    }
}
```


```java
 public re.test.test1.Demo();
    Code:
       0: aload_0
       1: invokespecial #1                  // Method java/lang/Object."<init>":()V
       4: return

  public void setData(T);
    Code:
       0: aload_0
       1: aload_1
       2: putfield      #2                  // Field data:Ljava/lang/String;
       5: return

  public T getData();
    Code:
       0: aload_0
       1: getfield      #2                  // Field data:Ljava/lang/String;
       4: areturn
}
```

在Java中如果有多个限制，需要通过 `&` 符号进行连接。并且会擦除到第一个边界：

小结：

#### 自动类型转换

在 Java 泛型的实现中，除了将泛型类型擦除到原始类型以外，还会将类型参数擦除到它的第一个边界。那这样就引起了一个问题，为什么我们在获取的时候，不需要进行强制类型转换呢？看下面的例子：

```java
public class Test {
    public static void main(String[] args) {
        Demo<String> demo = new Demo<String>();
        demo.setData("andy");
        String data = demo.getData();
    }
}
```

从上例中，我们能知道因为泛型擦除的缘故，Demo中实际的对象其实是Object,而我们在获取数据时是不需要强制转换呢？

javac -c Test.class

```java
 public re.test.test1.Test();
    Code:
       0: aload_0
       1: invokespecial #1                  // Method java/lang/Object."<init>":()V
       4: return

  public static void main(java.lang.String[]);
    Code:
       0: new           #2                  // class re/test/test1/Demo
       3: dup
       4: invokespecial #3                  // Method re/test/test1/Demo."<init>":()V
       7: astore_1
       8: aload_1
       9: ldc           #4                  // String andy
      11: invokevirtual #5                  // Method re/test/test1/Demo.setData:(Ljava/lang/Object;)V
      14: aload_1
      15: invokevirtual #6                  // Method re/test/test1/Demo.getData:()Ljava/lang/Object;
      18: checkcast     #7                  // class java/lang/String
      21: astore_2
      22: return
}
```

checkcast：检查操作数栈的栈顶项（对对象或数组的引用）是否可以转换为给定类型

虽然泛型内部，是无法获取内部的泛型参数的信息，但是当声明了泛型的实际类型参数后，我们是可以知道具体的泛型类型的。

#### 泛型函数

#### 泛型函数仍然是擦除到边界

```java
    public <T extends String> void Fuck(T data) {
        System.out.println(data.length());
    }

    public static void main(String[] args) {
        Test test = new Test();
        test.Fuck("xixi");
    }


```

#### 泛型函数，泛型作为返回值

```java
    public <T> T Fuck(int number) {
        String str = String.valueOf(number);
        return (T) str;
    }

    public static void main(String[] args) {
        Test test = new Test();
        String a = test.Fuck(5);

    }
```

根据返回值类型，推断出泛型类型的实际类型参数：

```java
  public re.test.test1.Test();
    Code:
       0: aload_0
       1: invokespecial #1                  // Method java/lang/Object."<init>":()V
       4: return

  public <T> T Fuck(int);
    Code:
       0: iload_1
       1: invokestatic  #2                  // Method java/lang/String.valueOf:(I)Ljava/lang/String;
       4: astore_2
       5: aload_2
       6: areturn

  public static void main(java.lang.String[]);
    Code:
       0: new           #3                  // class re/test/test1/Test
       3: dup
       4: invokespecial #4                  // Method "<init>":()V
       7: astore_1
       8: aload_1
       9: iconst_5
      10: invokevirtual #5                  // Method Fuck:(I)Ljava/lang/Object;
      13: checkcast     #6                  // class java/lang/String
      16: astore_2
      17: return
}
```

#### 仍然可以获取到泛型信息

需要注意的是这里的擦除只是对代码中的泛型类型擦除，对于实际上使用泛型类型的类其泛型信息仍然保留在对应类的 Class 文件中。只是这类信息是提供给 `Java 编译器` 及反射时使用的。具体规范官方有更详细的描述-->[Signatures](https://docs.oracle.com/javase/specs/jvms/se7/html/jvms-4.html#jvms-4.3.4)
只是对泛型类型的参数信息进行擦除，对使用泛型类型的类，可以通过反射，获取泛型的实际类型参数。

https://www.cnblogs.com/fanweisheng/p/11136868.html

https://www.zhihu.com/question/346911525/answer/830285753

## 利用有界通配符来提升API的灵活性

https://github.com/sjsdfg/effective-java-3rd-chinese/blob/master/docs/notes/31. 使用限定通配符来增加API的灵活性.md

### 通配符

通配符只适用于变量，或方法参数，

#### 上界通配符(协变)

#### 下界通配符(逆变)

#### 无界通配符

? 与 ? extend  Object 的区别

List<?> 与List有什么区分，前者不能存null，

## 泛型与数组的关系

为什么不能创建一个泛型数组？？？

为什么创建一个泛型数组是非法的？ 因为它不是类型安全的。 如果这是合法的，编译器生成的强制转换程序在运行时可能会因为 ClassCastException 异常而失败。 这将违反泛型类型系统提供的基本保证。


```
T[] = new Int[];
```

数组和泛型之间的第二个主要区别是数组被具体化了（reified）[JLS，4.7]。 这意味着数组在运行时知道并强制执行它们的元素类型。 如前所述，如果尝试将一个 String 放入 Long 数组中，得到一个 ArrayStoreException 异常。 相反，泛型通过擦除（erasure）来实现[JLS，4.6]。 这意味着它们只在编译时执行类型约束，并在运行时丢弃（或擦除）它们的元素类型信息。 **擦除是允许泛型类型与不使用泛型的遗留代码自由互操作**（详见第 26 条），从而确保在 Java 5 中平滑过渡到泛型。

实际上引入泛型的主要目标有以下几点：

类型安全
泛型的主要目标是提高 Java 程序的类型安全
编译时期就可以检查出因 Java 类型不正确导致的 ClassCastException 异常
符合越早出错代价越小原则
消除强制类型转换
泛型的一个附带好处是，使用时直接得到目标类型，消除许多强制类型转换
所得即所需，这使得代码更加可读，并且减少了出错机会
潜在的性能收益
由于泛型的实现方式，支持泛型（几乎）不需要 JVM 或类文件更改
所有工作都在编译器中完成
编译器生成的代码跟不使用泛型（和强制类型转换）时所写的代码几乎一致，只是更能确保类型安全而已

## 总结

如果类型参数是无限的，则将泛型类型中的所有类型参数替换为其边界或对象。因此，生成的字节码只包含普通类、接口和方法。
必要时插入类型转换以保持类型安全。
生成桥方法以保留扩展泛型类型中的多态性。
类型擦除确保不会为参数化类型创建新类；因此，泛型不会产生运行时开销。当然这个是本文下节要描述的内容。


## 最后

//    https://www.cnblogs.com/leyangzi/p/11379525.html
//    https://juejin.im/post/5d6c6636f265da03c8153a03
//    http://java.sun.com/docs/books/tutorial/java/generics/erasure.html

《Think in Java》
《Effective Java》
站在巨人的肩膀上，才能看的更远~
### 最后

站在巨人的肩膀上，才能看的更远~
