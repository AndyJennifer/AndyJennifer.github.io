---
title: 'Jvm系列之快速理解运行时数据区域(二）'
tags:
- Jvm相关
categories:
- Jvm
---

### 前言

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
- [Java虚拟机—栈帧、操作数栈和局部变量表]{https://zhuanlan.zhihu.com/p/45354152}
- [JVM-String常量池与运行时常量池]{https://blog.csdn.net/Sugar_Rainbow/article/details/68150249}
-[Class文件中的常量池详解（上）]{https://blog.csdn.net/wangtaomtk/article/details/52267548}
-[JVM 指令集整理]{https://juejin.im/entry/588085221b69e60059035f0a}