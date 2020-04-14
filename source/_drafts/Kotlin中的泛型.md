---
title: Kotlin中的泛型
tags:
  - null
categories:
  - null
---

### 前言

## Kotlin 相关知识点

kotlin中泛型不可忽略的，Java可以忽略，但是Kotlin必须要显示泛型

Kotlin 泛型与java 不同的点
1.必须声明泛型
2.*号
3.In 与 Out
3.具体化类型参数。内联函数具体化类型参数与java的互调用性，

```java
 class InlineDemo {

    /**
     * 1.检查Kotlin下的內联函数为什么能获取到，具体的类型参数。
     * 2.检查Kotlin中内敛函数与Java的互相调用的兼容性
     */
     inline fun <reified T> isStr(): Boolean {
        val a = 2
        return a is T
    }

    fun fuck(): Boolean {
        val str = isStr<Man>()
        return str
    }

}
```

```java
$ javap -verbose InlineDemo.class
Classfile /Users/xuwentao/Downloads/idePorject/out/production/idePorject/re/test/test1/InlineDemo.class
  Last modified 2020年4月14日; size 1294 bytes
  MD5 checksum b1d81bec7bc67fe07c04c501012b349b
  Compiled from "InlineDemo.kt"
public final class re.test.test1.InlineDemo
  minor version: 0
  major version: 52
  flags: (0x0031) ACC_PUBLIC, ACC_FINAL, ACC_SUPER
  this_class: #2                          // re/test/test1/InlineDemo
  super_class: #4                         // java/lang/Object
  interfaces: 0, fields: 0, methods: 3, attributes: 3
Constant pool:
   #1 = Utf8               re/test/test1/InlineDemo
   #2 = Class              #1             // re/test/test1/InlineDemo
   #3 = Utf8               java/lang/Object
   #4 = Class              #3             // java/lang/Object
   #5 = Utf8               isStr
   #6 = Utf8               ()Z
   #7 = Utf8               <T:Ljava/lang/Object;>()Z
   #8 = Integer            0
   #9 = Utf8               T
  #10 = String             #9             // T
  #11 = Utf8               kotlin/jvm/internal/Intrinsics
  #12 = Class              #11            // kotlin/jvm/internal/Intrinsics
  #13 = Utf8               reifiedOperationMarker
  #14 = Utf8               (ILjava/lang/String;)V
  #15 = NameAndType        #13:#14        // reifiedOperationMarker:(ILjava/lang/String;)V
  #16 = Methodref          #12.#15        // kotlin/jvm/internal/Intrinsics.reifiedOperationMarker:(ILjava/lang/String;)V
  #17 = Utf8               a
  #18 = Utf8               I
  #19 = Utf8               this
  #20 = Utf8               Lre/test/test1/InlineDemo;
  #21 = Utf8               $i$f$isStr
  #22 = Utf8               fuck
  #23 = Utf8               java/lang/Integer
  #24 = Class              #23            // java/lang/Integer
  #25 = Utf8               valueOf
  #26 = Utf8               (I)Ljava/lang/Integer;
  #27 = NameAndType        #25:#26        // valueOf:(I)Ljava/lang/Integer;
  #28 = Methodref          #24.#27        // java/lang/Integer.valueOf:(I)Ljava/lang/Integer;
  #29 = Utf8               re/test/test1/Man
  #30 = Class              #29            // re/test/test1/Man
  #31 = Utf8               a$iv
  #32 = Utf8               this_$iv
  #33 = Utf8               str
  #34 = Utf8               Z
  #35 = Utf8               <init>
  #36 = Utf8               ()V
  #37 = NameAndType        #35:#36        // "<init>":()V
  #38 = Methodref          #4.#37         // java/lang/Object."<init>":()V
  #39 = Utf8               Lkotlin/Metadata;
  #40 = Utf8               mv
  #41 = Integer            1
  #42 = Integer            16
  #43 = Utf8               bv
  #44 = Integer            3
  #45 = Utf8               k
  #46 = Utf8               d1
  #47 = Utf8               \u0000\u0014\n\u0002\u0018\u0002\n\u0002\u0010\u0000\n\u0002\b\u0002\n\u0002\u0010\u000b\n\u0002\b\u0003\u0018\u00002\u00020\u0001B\u0005¢\u0006\u0002\u0010\u0002J\u0006\u0010\u0003\u001a\u00020\u0004J\u0011\u0010\u0005\u001a\u00020\u0004\"\u0006\b\u0000\u0010\u0006\u0018\u0001H\u0086\b¨\u0006\u0007
  #48 = Utf8               d2
  #49 = Utf8
  #50 = Utf8               idePorject
  #51 = Utf8               InlineDemo.kt
  #52 = Utf8               Code
  #53 = Utf8               LineNumberTable
  #54 = Utf8               LocalVariableTable
  #55 = Utf8               Signature
  #56 = Utf8               SourceFile
  #57 = Utf8               SourceDebugExtension
  #58 = Utf8               RuntimeVisibleAnnotations
{
  public final <T extends java.lang.Object> boolean isStr();
    descriptor: ()Z
    flags: (0x1011) ACC_PUBLIC, ACC_FINAL, ACC_SYNTHETIC
    Code:
      stack=2, locals=3, args_size=1
         0: ldc           #8                  // int 0
         2: istore_1
         3: iconst_2
         4: istore_2
         5: iconst_3
         6: ldc           #10                 // String T
         8: invokestatic  #16                 // Method kotlin/jvm/internal/Intrinsics.reifiedOperationMarker:(ILjava/lang/String;)V
        11: iconst_1
        12: ireturn
      LineNumberTable:
        line 17: 3
        line 18: 5
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            5       8     2     a   I
            0      13     0  this   Lre/test/test1/InlineDemo;
            3      10     1 $i$f$isStr   I
    Signature: #7                           // <T:Ljava/lang/Object;>()Z

  public final boolean fuck();
    descriptor: ()Z
    flags: (0x0011) ACC_PUBLIC, ACC_FINAL
    Code:
      stack=1, locals=5, args_size=1
         0: aload_0
         1: astore_2
         2: iconst_0
         3: istore_3
         4: iconst_2
         5: istore        4
         7: iload         4
         9: invokestatic  #28                 // Method java/lang/Integer.valueOf:(I)Ljava/lang/Integer;
        12: instanceof    #30                 // class re/test/test1/Man
        15: istore_1
        16: iload_1
        17: ireturn
      LineNumberTable:
        line 22: 0
        line 27: 4
        line 28: 7
        line 22: 15
        line 23: 16
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            7       8     4  a$iv   I
            2      13     2 this_$iv   Lre/test/test1/InlineDemo;
            4      11     3 $i$f$isStr   I
           16       2     1   str   Z
            0      18     0  this   Lre/test/test1/InlineDemo;

  public re.test.test1.InlineDemo();
    descriptor: ()V
    flags: (0x0001) ACC_PUBLIC
    Code:
      stack=1, locals=1, args_size=1
         0: aload_0
         1: invokespecial #38                 // Method java/lang/Object."<init>":()V
         4: return
      LineNumberTable:
        line 10: 0
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0       5     0  this   Lre/test/test1/InlineDemo;
}
SourceFile: "InlineDemo.kt"
SourceDebugExtension:
  SMAP
  InlineDemo.kt
  Kotlin
  *S Kotlin
  *F
  + 1 InlineDemo.kt
  re/test/test1/InlineDemo
  *L
  1#1,26:1
  17#1,2:27
  *E
  *S KotlinDebug
  *F
  + 1 InlineDemo.kt
  re/test/test1/InlineDemo
  *L
  22#1,2:27
  *E
RuntimeVisibleAnnotations:
  0: #39(#40=[I#41,I#41,I#42],#43=[I#41,I#8,I#44],#45=I#41,#46=[s#47],#48=[s#20,s#49,s#36,s#22,s#49,s#5,s#9,s#50])
    kotlin.Metadata(
      mv=[1,1,16]
      bv=[1,0,3]
      k=1
      d1=["\u0000\u0014\n\u0002\u0018\u0002\n\u0002\u0010\u0000\n\u0002\b\u0002\n\u0002\u0010\u000b\n\u0002\b\u0003\u0018\u00002\u00020\u0001B\u0005¢\u0006\u0002\u0010\u0002J\u0006\u0010\u0003\u001a\u00020\u0004J\u0011\u0010\u0005\u001a\u00020\u0004\"\u0006\b\u0000\u0010\u0006\u0018\u0001H\u0086\b¨\u0006\u0007"]
      d2=["Lre/test/test1/InlineDemo;","","()V","fuck","","isStr","T","idePorject"]
    )

```
### 最后

站在巨人的肩膀上，才能看的更远~
