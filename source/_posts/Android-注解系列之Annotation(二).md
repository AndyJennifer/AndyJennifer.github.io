---
title: Android 注解系列之Annotation(二)
date: 2019-02-23 22:03:41
categories:
- 注解相关
tags: 
- 注解
---

{% asset_img 居家程序员.jpg 居家程序员 %}

### 注解基本概念

注解（`也称为元数据`），为我们在代码中添加信息提供了一种形式化的方法，使我们可以在稍后某个时刻非常方便的使用这些数据。其中注解是总到引入到`JAVA SE5`的重要的语言变化之一。其可以提供用来完整的描述程序所需的信息，而这些信息是无法用Java表达的。因此，注解使得我们能够以将由编译器来测试和验证的格式，存储有关程序的额外信息。注解可以用来生成描述符文件。甚至是新的类定义，并且有助于减轻编写`样板`代码的负担。通过使用注解。我们可以将这些元数据保存在Java源代码中，并利用`annotation API为自己的注解构造处理工具`。

### 注解的声明

说了这么多，我们先来看看注解的声明。具体参看下面的例子：

```java
@Documented
@Retention(RetentionPolicy.RUNTIME)
@Target(value = {TYPE, FIELD, METHOD, PARAMETER, CONSTRUCTOR, LOCAL_VARIABLE, ANNOTATION_TYPE,PACKAGE})
public @interface HelloAnnotation {
}

```

观看上述例子，我们发现注解的声明其实有点类似于Java接口的声明。除了@符号以外，@HelloAnnotation 定义更像是一个空的接口。事实上，它与其他任何Java接口一样。注解也会被编译成class文件。在定义注解时需要一些`元注解`，如`@Retention`和`@Target`。要知道注解的正确使用，我们需要了解`元注解`的使用方法与作用。

#### 元注解

Java目前内置了5种元注解，元注解主要负责注解其他的注解。这里分别对其进行介绍:

##### @Target元注解

该注解主要表示该注解可以用于什么地方，其中可能ElementType参数为以下几种情况：

- TYPE：用于类、接口（`包括注解类型`）或enum声明
- FIELD：用于字段声明，包括enum实例
- METHOD：用于方法声明
- PARAMETER：用于参数声明
- CONSTRUCTOR：用于构造函数声明
- LOCAL_VARIABLE：用于局部变量声明
- ANNOTATION_TYPE：用于注解可也用于注解声明（应用于另一个注解上）
- PACKAGE：用于包声明
- TYPE_PARAMETER：用于类上泛型参数的说明（`JDK1.8加入`）
- TYPE_USE：用于任何类型的声明（`JDK1.8加入`）

这里为了方便大家理解，会对以上@Target作用范围进行介绍，其中关于`TYPE_PARAMETER与TYPE_USE`会单独着重介绍。下面我们还是以@HelloAnnotation 注解为例，其中该注解声明如下：

```java
@Target(value = {TYPE, FIELD, METHOD, PARAMETER, CONSTRUCTOR, LOCAL_VARIABLE, ANNOTATION_TYPE,PACKAGE})
public @interface HelloAnnotation {
}
```

那么在实际代码中，我们可以通过@HelloAnnotation 注解定义的Target去声明想声明的东西。具体代码如下所示：

```java
// TYPE：用于类、接口（`包括注解类型`）或enum声明
@HelloAnnotation
class AnnotationDemo {

    //CONSTRUCTOR：用于构造函数声明
    @HelloAnnotation
    public AnnotationDemo(String name) {
        this.name = name;
    }

    //FIELD：用于字段声明，包括enum实例
    @HelloAnnotation
    private String name;

    //METHOD：用于方法声明
    @HelloAnnotation
    public void sayHello() {
        System.out.println("hello every one");
    }

    //PARAMETER：用于参数声明
    public void saySometing(@HelloAnnotation String text) { }

    //LOCAL_VARIABLE：用于局部变量声明
    public int add(int a, int b) {
        @HelloAnnotation int total = 0;
        return a + b;
    }

    //ANNOTATION_TYPE：用于注解可也用于注解声明（应用于另一个注解上）
    @HelloAnnotation
    @interface AnnotationTwo {

    }
}
```

其中对包进行注解修饰，需要在当前包下创建`package-info.java`文件，而该文件的作用有以下三点：

- 为标注在包上Annotation提供便利
- 声明包的私有类和常量
- 提供包的整体注释说明（ 如果是项目是分“包”开发，也就是说一个包实现一个业务逻辑或功能点、或模块、或组件，则需要对一个包有很好的说明，说明这个包是干啥的，有啥作用，版本变迁，特别说明等等）

```java
/**
 * 主要是为了测试包的注解
 */
@HelloAnnotation
package annotation;
```

##### 新增的TYPE_PARAMETER与TYPE_USE

在JDK1.8中增加了`TYPE_PARAMETER与TYPE_USE`,其中TYPE_PARAMETER用于修饰`类上的泛型参数`。

```java
@Retention(RetentionPolicy.RUNTIME)
@Target(value = {TYPE_PARAMETER})
public @interface HelloAnnotation {
}
class A<@HelloAnnotation T> {}
```

TYPE_USE用于`任何类型声明(也包含修饰类上的泛型参数）`，具体情况如下所示：

```java
@Retention(RetentionPolicy.RUNTIME)
@Target(value = {TYPE_USE})
public @interface HelloAnnotation {
}

@HelloAnnotation
class AnnotationDemo<@HelloAnnotation T> {

    @HelloAnnotation
    public AnnotationDemo(String name) {
        this.name = name;
    }

    @HelloAnnotation
    private String name;

    public void saySometing(@HelloAnnotation String text) { }

    public int add(int a, int b) {
        @HelloAnnotation int total = 0;
        return a + b;
    }

    @HelloAnnotation
    @interface AnnotationTwo { }
}
```

##### @Retention元注解

该注解表该注解在什么级别下被保存，可选的RetentionPolicy参数为：

- SOURCE：该类型的注解只会保留在源码里，经过编译器编译后，生成的class文件里是不会存在该注解信息的。
- CLASS：注解在class文件中可用，但是会被VM丢弃（该类型的注解信息会被保留在源码与class文件中，在执行的时候，不会被加载到虚拟机中）。`注意：当注解未定义未定义Retention值时，默认值是CLASS级别`
- RUNTIME：VM将在运行期间也保留注解，因此可以通过反射机制读取到注解的信息（该类型的注解信息会被保留在源码、class文件和虚拟机执行期间）。
  
##### @Documented元注解

该注解表示，是否将注解包含在JavaDoc中，具体列子如下图所示

```java
@Documented
@Retention(RetentionPolicy.RUNTIME)
public @interface HelloAnnotation {
}


@HelloAnnotation
class AnnotationDemo {

    @HelloAnnotation
    public AnnotationDemo(String name) {
        this.name = name;
    }

    @HelloAnnotation
    private String name;

    public void saySometing(@HelloAnnotation String text) { }

    public int add(int a, int b) {
        @HelloAnnotation int total = 0;
        return a + b;
    }

}
```

我们跳转到项目的目录，打开命令行，执行`javadoc -encoding utf-8 -charset utf-8 -package annotation`命令(这里我是所有的文件都是放在annotaton包下的，所以你可以根据你自己的包名为该包下的所有.java文件生成Doc文档)。运行完命令后我们找到自动生成的Doc文档。点击后如下图所示：

{% asset_img doc文档.png doc文档 %}

从上图中我们可以发现，如果为注解指定了`@Documented`元注解，那么在生成的Doc文档中是会有相应注解的（`如图上红箭头所指`）。

##### @Inherited元注解

该注解表示，允许子类继承父类中的注解。其实理解起来也简单。看下面的列子：

```java
@Target(ElementType.TYPE)
@Inherited
@Retention(RetentionPolicy.RUNTIME)
@interface Hello {
}


@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@interface World {
}

@Hello
class Person {
}


@World
class Man extends Person {
    public static void main(String[] args) {
        Annotation[] annotations = Man.class.getAnnotations();
        for (Annotation annotation : annotations) {
            System.out.println(annotation.annotationType());
        }
    }
}
//输出结果
interface annotation.Hello
interface annotation.World
```

在上述代码中，创建了@Hello与@World注解，其中`@Hello使用@Inherited修饰`,同时我们也创建了用`@Hello修饰的Person类`及其用`@World修饰的Man子类`,然后我们通过`Man.class.getAnnotations()`方法获取Man类中的注解（下文会对注解使用以及赋值进行介绍），得到上述代码中的输出结果。也就证明了@Inherited元注解可以让子类继承父类中的注解的结论。

##### @Repeatable元注解(JDK 1.8之后新加入的)

该注解是JDK 1.8新加入的，该注解表示，可以在`同一个地方多次使用同一种注解类型`。也就是说在JDK 1.8之前是无法在同一个类型上使用相同的注解的。

###### JDK1.8之前

```java

@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
public @interface FilterPath {
    String value();
}

//在JDK1.8之前是在一个地方多次使用同一种注解
@FilterPath("hello/java")
@FilterPath("hello/android")
class FileOperate {
}

```

在上述代码中，我们声明了@FilterPath注解，你有可能注意到了其中的`String value();`这段语句，这里大家先不着急理解这段代码到底是什么意思，大家就理解成给该`注解提供字符串赋值`操作就行了（下文会对注解使用以及赋值进行介绍）。如果我们采用以上代码，编译器是会报错的。所以为了处理这种情况，在JDK1.8之前，如果想实现类似于上述相同的功能，我们一般采用下面的这种方式：

```java

@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
public @interface FilterPath {
    String []value();
}

@FilterPath({"hello/java","hello/android"})
class FileOperate {
}
```

将@FilterPath注解中的`String value();`修改为`String []value();`,也就是说让该注解接受字符串数组。

###### JDK1.8之后

在JDK1.8之后我们可以使用@Repeatable，但是使用该注解也有一定的限制。下面我们一起来看看：

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Repeatable(FilterPaths.class)//添加@Repeatable元注解，注意其中的值
public @interface FilterPath {
    String value();
}

@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
public @interface FilterPaths {
    FilterPath[] value();//注解其中的数组类型为FilterPath
}

//现在可以在同一个地方使用同一注解啦~
@FilterPath("hello/java")
@FilterPath("hello/android")
class FileOperate {
}
```

在上述代码中，我们创建了`@FilterPath与@FilterPaths`两个注解，需要注意的是我们在@FilterPath注解上增加了元注解`@Repeatable(FilterPaths.class)`其中的参数`FilterPaths.class是指明接受同一个类型上重复注解的容器，`（也就是接受重复的@FilterPath注解），那么我们再看@FilterPaths中声明了 `FilterPath[] value();`，也就是其接受@FilterPath注解类型。

###### @Repeatable元注解使用注意事项

为了处理@Repeatble注解，JDK1.8在AnnotatedElement接口中提供了`getAnnotationsByType与getAnnotationsByType`方法，(注意：如果我们采用传统的方法，也就是`getAnnotation(Class<A> annotationClass)`方法来获取声明的注解，我们需要传入`注解容器的class`,而`不是需要重复的注解的class`）。这里我们还是以JDK1.8之后中提到的代码为例：

```java
    public static void main(String[] args) {

        //从该类上获取FilterPath注解信息
        FilterPath filterPath = FileOperate.class.getAnnotation(FilterPath.class);
        System.out.println(filterPath);

        //从该类上获取FilterPaths注解信息
        System.out.println("----------------------------");
        FilterPaths filterPaths = FileOperate.class.getAnnotation(FilterPaths.class);
        System.out.println(filterPaths);
        for (FilterPath path : filterPaths.value()) {
            System.out.println(path.value());
        }

        //通过getAnnotationsByType
        System.out.println("----------------------------");
        FilterPath[] annotationsByType = FileOperate.class.getAnnotationsByType(FilterPath.class);
        if (annotationsByType != null) {
            for (FilterPath path : annotationsByType) {
                System.out.println(path.value());
            }
        }

        //通过getDeclaredAnnotationsByType
        System.out.println("----------------------------");
        FilterPath[] declaredAnnotationsByType = FileOperate.class.getDeclaredAnnotationsByType(FilterPath.class);
        if (declaredAnnotationsByType != null) {
            for (FilterPath path : declaredAnnotationsByType) {
                System.out.println(path.value());
            }
        }
    }


//输出结果
null
----------------------------
@annotation.FilterPaths(value=[@annotation.FilterPath(value=hello/java), @annotation.FilterPath(value=hello/android)])
hello/java
hello/android
----------------------------
hello/java
hello/android
----------------------------
hello/java
hello/android
```

从输出结果来看，我们`并不能通过getAnnotation(FilterPath.class)`获取注解（获得的注解为null），而是需要通过getAnnotation(`FilterPaths.class`)来获取）。同时如果我们采`getAnnotationsByType(FilterPath.class)`或`getDeclaredAnnotationsByType(FilterPath.class)`就能获取到正确的值。这里需要注意`getAnnotationsByType`与`getDeclaredAnnotationsByType`方法的区别，如果子类调用`getAnnotationsByType`方法且该子类的父类中声明了用@Inherited修饰的注解，那么可以获得父类中的注解。而`getDeclaredAnnotationsByType`是获取不到父类中声明的注解的。

### 注解的使用与支持属性类型

#### 注解支持属性类型

在了解了注解的定义与元注解之后，我们一起来了解注解元素中可以定义的属性。其中支持的具体类型如下所示：

- 所有的基本类型（int、float、boolean)等
- String
- Class
- enum
- Annotaion
- 以及以上类型的数组

如果你使用了其他类型，那么编译器就会报错，`注意！！！！也不能使用任何类型的包装类型`。不过由于自动打包的存在，这也不算什么限制。注解亦可以作为元素的类型。也就是说`注解可以嵌套`。

#### 注解添加属性

我们已经知道了注解中属性的支持类型，现在就开始为注解添加属性吧。其基本语法是: `类型 属性名();`，请看如下例子：

```java
//声明枚举
enum Week {
    MONDAY,
    TUESDAY,
    WEDNESDAY,
    THURSDAY,
    FRIDAY,
    SATURDAY,
    SUNDAY
}


@Target(ElementType.TYPE_USE)
@Retention(RetentionPolicy.RUNTIME)
public @interface HelloAnnotation {
    String text();
}


@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@interface WorldAnnotation {

    //基本类型及其数组类型
    int intAttr();
    float floatAttr() ;
    boolean booleanAttr() ;

    int[] intArray() ;
    float[] floatArray() ;
    boolean[] booleanArry() ;

    //String类型及其数组类型
    String stringAttr() ;
    String[] stringArray();

    //Class类型及其数组类型
    Class classAttr() ;
    Class[] classArray();

    //enum类型及其数组类型
    Week day() ;
    Week[] week();

    //注解嵌套
    HelloAnnotation HelloAnnotation();
}
```

在上图中我将注解的支持的所有类型都展示出来了，这样我相信大家都能非常好的理解了。

##### 为属性指定缺省值（默认值）

属性除了用户指定值外，还支持默认值，其语法为：`属性 属性名() default 默认值;`，那么结合上述的例子：

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@interface WorldAnnotation {

    //基本类型及其数组类型
    int intAttr() default -1;

    float floatAttr() default -1f;

    boolean booleanAttr() default false;

    int[] intArray() default {1, 2, 3};

    float[] floatArray() default {1f, 2f, 3f};

    boolean[] booleanArry() default {true, false, true};

    //String类型及其数组类型
    String stringAttr() default "";

    String[] stringArray();

    //Class类型及其数组类型
    Class classAttr() default Class.class;

    Class[] classArray();

    //enum类型
    Week day() default Week.MONDAY;

    //enum数组类型
    Week[] week() default {Week.MONDAY,Week.THURSDAY};

    //注解嵌套
    HelloAnnotation HelloAnnotation() default @HelloAnnotation(text = "word");
}
```

也就是说当用户自己没有指定相应属性值的时候，如果属性设置了默认值，那么该属性的值就是默认值。但是设置属性的默认值时有限制的。具体内容看下面的介绍。

###### 注解默认值限制

虽然限制我们已经可以在注解定义我们想要的信息，但是在Java中，注解中的元素类型必须要么有默认值，要么在使用注解是提供元素的值。其次`对于非基本类型的元素`，无论是在源代码中声明时，或是在注解接口中定义默认值时，都`不能以null作为其值`。这个约束使得处理器很难发现一个元素的存在和缺失的状态，因为在每个注解的声明中，所有的元素都存在，并且都具有相应的值，为了绕开这个约束，我们只能定义一些特殊的值，`例如空字符串或负数，以此表示某个元素不存在`。也就说像这样的代码编译器是不会通过的：

```java
    String stringAttr() default null;//错误！！！！
    String[] stringArray() default null;//错误！！！
```

##### value属性

如果一个注解中有一个`名称为value`的属性，且你只想设置value属性(即其他属性都采用默认值或者你只有一个value属性)，那么可以省略掉`“value=”`部分。具体代码如下：

```java
//第一种情况，只有一个vaule属性，那么你在使用时候可以直接@WorldAnnotation("hello")
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@interface WorldAnnotation {
    String value();
}

//第二种情况，有多个属性但是其属性都有默认值，
//你只使用value属性，那么你在使用时候可以直接@WorldAnnotation("hello")
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@interface WorldAnnotation {
    int intAttr() default -1;
    float floatAttr() default -1f;
    String value();
}
```

### 注解与反射机制

在了解了注解的定义与属性的添加后，现在我们在来看看注解的实际运用情况。注解的使用需要与`Java的反射机制`结合使用。所以了解其中的了解两者之前的关系尤为重要。

#### 注解与反射机制的关系

众所周知，JAVA反射机制是在运行状态中，对于任意一个类，都能够知道这个类的所有属性和方法；对于任意一个对象，都能够调用它的任意方法和属性；这种动态获取信息以及动态调用对象方法的功能称为java语言的反射机制。那么从Java的整个类加载机制来看，过程是如下这样：

{% asset_img java类加载图.png java类加载图 %}

对于Java类的加载主要分为以下步骤：

- 将程序中的`*.java`文件通过javac命令编译成扩展名为`*.class`文件。其中`*.class`文件保存着Java代码转换后的虚拟机指令。
- 当需要使用某个类时，JVM(Java 虚拟机)将会加载它的`*.class`文件，并创建对应的`Class对象`。其中Class对象中不仅有着类的声明定义，还有 Constructor(类的构造器定义)、Field(类的成员变量定义)、Method(类的方法定义)、Package(类的包定义)。

##### 那注解到底和类的加载有什么关系呢

当我们声明了注解，且将注解的生命周期设置为`@Retention(RetentionPolicy.RUNTIME)`，那么在编译成class文件的时候，`会将注解添加到文件中去`，那么JVM根据class文件生成相应的`Class对象之后就会带有注解信息`。那么我们通过Class对象中的Constructor、Field、Method等类，就能获取其上声明的注解信息了。`那注解信息到底是以声明形式声明与表现的呢？`

这里我们还是以@HelloAnnotation 注解为例，当我们声明了注解后，通过`javap`命令获取编译后的的字节码信息

```java
@Target(ElementType.TYPE_USE)
@Retention(RetentionPolicy.RUNTIME)
public @interface HelloAnnotation {
    String value();

}
//通过javap -p HelloAnnotation.class
public interface annotation.HelloAnnotation extends java.lang.annotation.Annotation {
  public abstract java.lang.String value();
}
```

从上图中，我们可以得知经过编译后，其实注解最终会`继承Annotation接口`。也就是说注解最终会以`java.lang.annotation.Annotation`对象的形式在Class对象中进行展示或存储。

##### 注解的处理

为了方便处理接口信息以及实现面向对象的规则，其中`Constructor、Field、Method、Class、Package`类都实现了`AnnotatedElement`接口。具体关系如下图所示：

{% asset_img 整体继承关系图.png 整体继承关系图 %}

也就是最终的注解注解处理全部都交给了`AnnotatedElement`接口来实现。那现在我们来看看该接口的方法声明。

##### AnnotatedElement中的方法声明

在`AnnotatedElement`接口中为我们提供了以下几个方法来获取注解信息：

| 方法名称|     返回值|   方法说明|
| :--------: | :--------:| :------: |
| getAnnotation(Class<T> annotationClass)|   `<T extends Annotation>` T |  返回元素上指定类型的注解，如果无，则返回为null|
| getAnnotations()|    Annotation[ ] |  返回元素上存在的所有注解，`包括从父类继承的`|
| getAnnotationsByType(Class<T> annotationClass) `since 1.8`|   `<T extends Annotation>` T [ ]  |  返回元素上指定的类型的注解数组，`包括父类的注解`，如果无，返回长度为0的数组，该方法与getAnnotation(Class<T> annotationClass)的主要区别是，`该方法可以检查注解是不是重复的`。如果是这样，尝试通过“查看”容器注释来找到该类型的一个或多个注释。|
| getDeclaredAnnotation(Class<T> annotationClass)|  `<T extends Annotation>` T |  返回该元素上的指定类型的所有的注解，不包括父类的注解，如果无，返回长度为0的数组|
| getDeclaredAnnotationsByType(Class<T> annotationClass)  `since 1.8`|  `<T extends Annotation>` T [ ] |  同getAnnotationsByType(Class<T> annotationClass)方法类似，只是获取的注解中`不包括父类的注解`|
|  getDeclaredAnnotations()|   Annotation[ ]  |  返回该元素上的所有的注解，不包括父类的注解，如果无，返回长度为0的数组|

其中`getAnnotationsByType(Class<T> annotationClass)`与`getDeclaredAnnotationsByType(Class<T> annotationClass)`方法是`jdk 1.8`之后提供的接口`默认实现方法`。需要注意的是该两个方法是支持注解的`@Repeatable`，而其他方法是不支持的。那么在平时开发中，我们可以根据自己的项目需求选取不同的方法。

#### 注解的实际使用

通过了解注解的声明以及与反射机制之间的关系后，现在我们来实战一下。简单的写个例子彻底巩固注解的相关知识吧。这里我们简单通过`什么人在什么地方做了什么事`为例子：

```java
    //什么人
    @Target(ElementType.TYPE)
    @Retention(RetentionPolicy.RUNTIME)
    @interface Who {
        String name();
        int age();
    }

    //在哪里
    @Target(ElementType.FIELD)
    @Retention(RetentionPolicy.RUNTIME)
    @interface Where {
        String country();
        String province();
        String city();
    }

    //做了什么事
    @Target(ElementType.METHOD)
    @Retention(RetentionPolicy.RUNTIME)
    @interface DoSomething {
        String value();
    }
```

上述代码中，我们声明了三个注解，如果你认真看了前面我们说的注解的定义和使用话的理解起来非常简单，这里我们需要注意的是三个注解中的 `@Retention`都是设置为`(RetentionPolicy.RUNTIME)`，之所以设置为运行时，是因为根据类的加载机制，Class对象的生成是在JVM读取class文件的时候，也就是运行期间。那下面我们接着看具体的使用：

```java
@Who(name = "AndyJenifer", age = 18)
class Person {

    @Where(country = "中国", province = "四川", city = "成都")
    private String where;

    @DoSomething("写博客")
    public void doSomething() {

    }

    public static void main(String[] args) {
        Class<Person> personClass = Person.class;
        StringBuffer sb = new StringBuffer();

        //获取类上的注解
        Who who = personClass.getAnnotation(Who.class);
        sb.append(who.name());
        sb.append(who.age());

        //获取字段上的注解
        Field[] fields = personClass.getDeclaredFields();
        for (Field field : fields) {
            Annotation[] annotations = field.getAnnotations();
            if (annotations.length > 0 && annotations[0] instanceof Where) {
                Where where = (Where) annotations[0];
                sb.append(where.country());
                sb.append(where.province());
                sb.append(where.city());
            }
        }

        //获取方法上的注解
        Method[] methods = personClass.getMethods();
        for (Method method : methods) {
            Annotation[] annotations = method.getAnnotations();
            if (annotations.length > 0 && annotations[0] instanceof DoSomething) {
                DoSomething doSomething = (DoSomething) annotations[0];
                sb.append(doSomething.value());
            }
        }

        System.out.println(sb.toString());

    }
    //输出结果：AndyJenifer18中国四川成都写博客
}
```

上述代码理解起来还是比较容易，在Main方法中获取当前Person的Class对象，通过Class对象获取其中声明的字段与方法。得到相应的字段与方法后，再去拿上面声明的注解。然后组合信息并打印。细心的小伙伴肯定观察到了在获取相应字段的时候，我们调用的是`getDeclaredFields()`而不是方法`getFields()`(当然对于其他元素，如 Constructor、Field、Method、Package，都有类似的方法`getDeclaredXXXX()`与`getXXXX()`)。这里简单的说一下这两种方法的区别：

- getXXXX()：获得某个类的所有的公共（public）的元素（如Constructor、Field、Method、Package），包括父类声明的。
- getDeclaredXXXX()：获得某个类的所有声明的元素（如Constructor、Field、Method、Package），即包括`public、private和proteced`，但是`不包括父类声明的`。

### 思考

文章到这里，现在大家已经基本了解了注解的声明与使用。不知道小伙伴们有没有想过一个问题。如果我们声明了一个注解，然后希望该注解在项目的不同类中都会使用。那么当处理这些类的注解的时候，`我们是不是需要手动的找到所有的Class对象`？（不管你是通过类名也好，还是通过文件的方式来读取也好）。那这样是不是会很麻烦呢？在文章中我们也提到过，注解可以用来生成描述符文件。甚至是新的类定义，并且有助于减轻编写`样板`代码的负担。`那么怎么通过注解生成新的类的定义呢?`，`又怎么生成样板代码呢？`。如果大家有兴趣，我们将在后续文章继续讲述并解决这些问题。

### 最后

该文章参考以下博客与图书，站在巨人的肩膀上。可以看得更远。

[深入理解Java注解类型(@Annotation)](https://blog.csdn.net/javazejian/article/details/71860633)
《Think in java 》
