---
title: 'Jvm系列之快速理解运行时数据区域(二)'
tags:
- Jvm相关
categories:
- Jvm
---

### 前言
在上篇文章{% post_link Jvm系列之运行时数据区域(一) %}文章中，我总结了Jvm运行时数据区域的划分。相信大多数小伙伴看到这些概念性的东西就会头大。所以接下来这篇文章中，我会着重讲解程序计数器、虚拟机栈、栈帧、操作数栈、与局部变量变量表在实际代码中的应用。希望通过该篇文章，能让大家理解相关知识点。


### 程序计数器、pc寄存器（pc program register)
在上篇文章{% post_link Jvm系列之运行时数据区域(一) %}文章中，我们介绍了程序计数器是`线程私有的内存区域`，主要存储的内容为当期线程所执行的字节码的行号指令器，我们都知道，在Java程序中，每个类对应相应的class文件。而JVM在执行字节码时，把字节码解释成具体平台上的机器执行。故在多个线程访问相同的字节码时，我们需要一个内存区域，来记录不同线程执行字节码的行号。

也就是如下图所示：

![程序计数器](https://user-gold-cdn.xitu.io/2019/6/4/16b23175df8f7be6?w=961&h=470&f=png&s=68772)


因为多线程操作，是通过CPU切换时间片处理的，需要保证线程在恢复的时候，能恢复到正确的指令行数。


### Java虚拟机栈
在Java虚拟机栈中，我们知道了Java虚拟机栈为线程的私有内存区域，且主要存储的内容为栈帧，而栈帧中又包含了局部变量表，操作数栈、动态链接，方法返回地址等。也就是下图的这种结构：

![java虚拟机栈](https://user-gold-cdn.xitu.io/2019/6/4/16b23192df764306?w=679&h=595&f=png&s=31013)

单纯的理解上图比较难，所以这里我们以一段代码为例，如下所示：

```
public class Main {

    public static void main(String[] args) {
        int result = add(10,20);
        System.out.println(result);
    }

    public static int add(int x,int y){
        return x+y;
    }
}

```

```
C:\Users\W9001729\IdeaProjects\pratice\out\production\HelloWorld>javap -c Main
Compiled from "Main.java"
public class Main {
  public Main();
    Code:
       0: aload_0
       1: invokespecial #1                  // Method java/lang/Object."<init>":()V
       4: return

  public static void main(java.lang.String[]);
    Code:
       0: bipush        10
       2: bipush        20
       4: invokestatic  #2                  // Method add:(II)I
       7: istore_1
       8: getstatic     #3                  // Field java/lang/System.out:Ljava/io/PrintStream;
      11: iload_1
      12: invokevirtual #4                  // Method java/io/PrintStream.println:(I)V
      15: return

  public static int add(int, int);
    Code:
       0: iload_0
       1: iload_1
       2: iadd
       3: ireturn
}
```


```
public class Main {

    public static void main(String[] args) {
        int result = add(10,20,30);
        System.out.println(result);
    }

    public static int add(int x,int y,int z){
        return x+y+z;
    }
}
```

```
C:\Users\W9001729\IdeaProjects\pratice\out\production\HelloWorld>javap -c Main
Compiled from "Main.java"
public class Main {
  public Main();
    Code:
       0: aload_0
       1: invokespecial #1                  // Method java/lang/Object."<init>":()V
       4: return

  public static void main(java.lang.String[]);
    Code:
       0: bipush        10
       2: bipush        20
       4: bipush        30
       6: invokestatic  #2                  // Method add:(III)I
       9: istore_1
      10: getstatic     #3                  // Field java/lang/System.out:Ljava/io/PrintStream;
      13: iload_1
      14: invokevirtual #4                  // Method java/io/PrintStream.println:(I)V
      17: return

  public static int add(int, int, int);
    Code:
       0: iload_0
       1: iload_1
       2: iadd
       3: iload_2
       4: iadd
       5: ireturn
}
```

### 参考
站在巨人的肩膀上才能看的更远~
- [Java虚拟机—栈帧、操作数栈和局部变量表] (https://zhuanlan.zhihu.com/p/45354152)
- [JVM-String常量池与运行时常量池](https://blog.csdn.net/Sugar_Rainbow/article/details/68150249)
-[Class文件中的常量池详解（上）](https://blog.csdn.net/wangtaomtk/article/details/52267548)
-[JVM 指令集整理](https://juejin.im/entry/588085221b69e60059035f0a)