---
title: Kotlin与Java的互操作
tags:
  - null
categories:
  - null
---

### 前言

### @JvmField 实例字段

```kotlin
class User(id:String){

    val ID = id;
}
```

反编译后的java代码

```java
public final class User {
   @NotNull
   private final String ID;

   @NotNull
   public final String getID() {
      return this.ID;
   }

   public User(@NotNull String id) {
      Intrinsics.checkParameterIsNotNull(id, "id");
      super();
      this.ID = id;
   }
}
```

使用@JvmField注解

```java
class  User(id:String){
    @JvmField
    val ID = id;
}

```

```java
public final class User {
   @JvmField
   @NotNull
   public final String ID;

   public User(@NotNull String id) {
      Intrinsics.checkParameterIsNotNull(id, "id");
      super();
      this.ID = id;
   }
}

```

//Instructs the Kotlin compiler not to generate getters/setters for this property and expose it as a field.


### 静态字段

### 静态方法


### 扩展函数

```java
fun MutableList<Int>.swap(index1: Int, index2: Int) {
    val tmp = this[index1] // “this”对应该列表
    this[index1] = this[index2]
    this[index2] = tmp
}
```

编译后

```java
  public static final void swap(@NotNull List $this$swap, int index1, int index2) {
      Intrinsics.checkParameterIsNotNull($this$swap, "$this$swap");
      int tmp = ((Number)$this$swap.get(index1)).intValue();
      $this$swap.set(index1, $this$swap.get(index2));
      $this$swap.set(index2, tmp);
   }
```

### 最后

站在巨人的肩膀上，才能看的更远~
