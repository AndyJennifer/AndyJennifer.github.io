---
title: Android-注解系列之EventBus3”加速引擎“（五）
tags:
  - EventBus
categories:
  - 源码分析
date: 2019-10-23 00:16:48
---


{% asset_img bus.jpg bus %}

### 前言

在上篇文章 {% post_link Android-注解系列之EventBus3原理（四）%}中我们讲解了 EventBus3 的内部原理，在该篇文章中我们将讲解 EventBus3 中的 `“加速引擎“`---索引类。阅读该篇文章我们能够学到如下知识点。

- EventBus3 索引类出现的原因
- EventBus3 索引类的使用
- EventBus3 索引类生成的过程
- EventBus3 混淆注意事项
  
>对 APT 技术不熟悉的小伙伴，可以查看文章 {% post_link Android-注解系列之APT工具(三) %}

### 前景回顾

在  {% post_link Android-注解系列之EventBus3原理（四）%}中，我们特别指出在 EventBus3 中优化了 `SubscriberMethodFinder` 获取类中包含 `@Subscribe` 注解的订阅方法的流程。使其能在 `EventBus.register()` 方法调用之前就能知道相关订阅事件的方法，这样就减少了程序在运行期间使用反射遍历获取方法所带来的时间消耗。优化点如下图中 `红色虚线框` 所示：

{% asset_img EventBus3优化.jpg EventBus3优化 %}

EventBus 作者 [Markus Junginger](http://androiddevblog.com/eventbus-3-droidcon/) 也给出了使用索引类前后 EventBus 的效率对比，如下图所示：

{% asset_img eventbus3-registration-perf-nexus9m.png eventbus3-registration-perf-nexus9m %}

从上图中，我们可以使用索引类后，EventBus 的效率有着明显的提升，而效率提升的背后，正是使用了 `APT` 技术所创建的`索引类`。那么接下来我们就来看一看 EventBus3 中是如何结合 `APT` 技术来进行优化的。

### 关键代码

阅读过 EventBus3 源码的小伙伴应该都知道，在 EventBus3 中获取类中包含 `@Subscribe` 注解的订阅方法有两种方式。

- 第一种：是直接在程序运行时反射获取
- 第二种：就是通过索引类。

而使用索引类的关键代码为 `SubscriberMethodFinder` 中的
[getSubscriberInfo()](https://github.com/greenrobot/EventBus/blob/master/EventBus/src/org/greenrobot/eventbus/SubscriberMethodFinder.java#L122) 方法与 [findUsingInfo()](https://github.com/greenrobot/EventBus/blob/master/EventBus/src/org/greenrobot/eventbus/SubscriberMethodFinder.java#L75) 方法 。
我们分别来看这两个方法。

#### findUsingInfo 方法

```java
    private List<SubscriberMethod> findUsingInfo(Class<?> subscriberClass) {
        FindState findState = prepareFindState();
        findState.initForSubscriber(subscriberClass);
        while (findState.clazz != null) {
            //👇关键代码，从索引类中获取 SubscriberInfo
            findState.subscriberInfo = getSubscriberInfo(findState);
            //方式1：如果 subscriberInfo 不为空，则从该对象中获取 SubscriberMethod 对象
            if (findState.subscriberInfo != null) {
                SubscriberMethod[] array = findState.subscriberInfo.getSubscriberMethods();
                for (SubscriberMethod subscriberMethod : array) {
                    if (findState.checkAdd(subscriberMethod.method, subscriberMethod.eventType)) {
                        findState.subscriberMethods.add(subscriberMethod);
                    }
                }
            } else {
                //方式2：如果 subscriberInfo 为空，那么直接通过反射获取
                findUsingReflectionInSingleClass(findState);
            }
            findState.moveToSuperclass();
        }
        return getMethodsAndRelease(findState);
    }
```

我们能从该方法中获得以下信息：

- EventBus3 中默认会调用 [getSubscriberInfo()](https://github.com/greenrobot/EventBus/blob/master/EventBus/src/org/greenrobot/eventbus/SubscriberMethodFinder.java#L122) 方法去获取 subscriberInfo 对象信息。
- 如果 subscriberInfo 不为空，则会从该对象中获取 `SubscriberMethod` 数组。
- 如果 subscriberInfo 为空，那么会直接通过`反射`去获取 `SubscriberMethod` 集合信息。

>SubscriberMethod 类中含有 `@Subscribe` 注解的方法信息封装（优先级，是否粘性，线程模式,订阅的事件），以及当前方法的 Method 对象(`java.lang.reflect` 包下的对象)。

也就说 EventBus 是否通过反射获取信息，是由 getSubscriberInfo（）方法来决定，那么我们查看该方法。

#### getSubscriberInfo 方法

```java
    private SubscriberInfo getSubscriberInfo(FindState findState) {
        if (findState.subscriberInfo != null && findState.subscriberInfo.getSuperSubscriberInfo() != null) {
            SubscriberInfo superclassInfo = findState.subscriberInfo.getSuperSubscriberInfo();
            if (findState.clazz == superclassInfo.getSubscriberClass()) {
                return superclassInfo;
            }
        }
        //👇这里是EventBus3中优化的关键，索引类
        if (subscriberInfoIndexes != null) {
            for (SubscriberInfoIndex index : subscriberInfoIndexes) {
                SubscriberInfo info = index.getSubscriberInfo(findState.clazz);
                if (info != null) {
                    return info;
                }
            }
        }
        return null;
    }
```

从代码逻辑中我们能得出，如果 `subscriberInfoIndexes` 集合不为空的话，那么就会从 `SubscriberInfoIndex（索引类）` 中去获取 `SubscriberInfo` 对象信息。该方法的逻辑并不复杂，唯一的疑惑就是这个 SubscriberInfoIndex(索引类) 对象是从何而来的呢？

聪明的小伙伴们已经想到了。对！！！就是通过 APT 技术自动生成的类。那么我们怎么使用 EventBus3 中的索引类？以及 EventBus3 中是如何生成的索引类的呢？ 不急不急，我们一个一个的解决问题。我们先来看看如何使用索引类。

### EventBus中索引类的使用

如果需要使用 EventBus3 中的索引类，我们可以在 App 的 `build.gradle` 中添加如下配置：

```gradle
android {
    defaultConfig {
        javaCompileOptions {
            annotationProcessorOptions {
                // 根据项目实际情况，指定索引类的名称和包名
                arguments = [ eventBusIndex : 'com.eventbus.project.EventBusIndex' ]
            }
        }
    }
}
dependencies {
    implementation 'org.greenrobot:eventbus:3.1.1'
    // 引入注解处理器
    annotationProcessor 'org.greenrobot:eventbus-annotation-processor:3.1.1'
}
```

>如果有小伙伴不熟悉 gradle 配置，可以查看 [AnnotationProcessorOptions](http://google.github.io/android-gradle-dsl/current/com.android.build.gradle.internal.dsl.AnnotationProcessorOptions.html)

在上述配置中，我们需要注意如下几点：

- 如果你不使用索引类，那么就没有必要设置 `annotationProcessorOptions` 参数中的值。也没有必要引入 EventBus 的注解处理器。
- 如果要使用索引类，并且也引入了 EventBus 的注解处理器（eventbus-annotation-processor），但却没有设置 arguments 的话，编译时就会报错：`No option eventBusIndex passed to annotation processor`。
- 索引类的生成，需要我们对代码重新编译。编译成功后，其该类对应路径为`\ProjectName\app\build\generated\source\apt\你设置的包名`。

当我们的索引类生成后，我们还需要在初始化 EventBus 时应用我们生成的索引类，代码如下所示：

```java
 EventBus.builder().addIndex(new EventBusIndex()).installDefaultEventBus();
```

之所以要配置索引类，是因为我们需要将我们生成的索引类添加到 `subscriberInfoIndexes` 集合中，这样我们才能从之前讲解的 `getSubscriberInfo（）`找到我们配置的索引类。`addIndex()` 代码如下所示：

```java
 public EventBusBuilder addIndex(SubscriberInfoIndex index) {
        if (subscriberInfoIndexes == null) {
            subscriberInfoIndexes = new ArrayList<>();
        }
        //👇这里添加索引类到 subscriberInfoIndexes 集合中
        subscriberInfoIndexes.add(index);
        return this;
    }
```

#### 索引类实际使用分析

如果你已经配置好了索引类，那么我们看下面的例子，这里我配置的索引类为 `EventBusIndex` 对应包名为： `'com.eventbus.project'` 。我在 EventBusDemo.java 中声明了如下方法：

```java
  public class EventBusDemo {

    @Subscribe(threadMode = ThreadMode.MAIN)
    public void onMessageEventOne(MessageEvent event) {
        System.out.println("hello");
    }

    @Subscribe(threadMode = ThreadMode.MAIN)
    public void onMessageEventTwo(MessageEvent event) {
        System.out.println("world");
    }
}

```

自动生成的索引类，如下所示：

```java
public class EventBusIndex implements SubscriberInfoIndex {
    private static final Map<Class<?>, SubscriberInfo> SUBSCRIBER_INDEX;

    static {
        SUBSCRIBER_INDEX = new HashMap<Class<?>, SubscriberInfo>();

        putIndex(new SimpleSubscriberInfo(EventBusDemo.class, true, new SubscriberMethodInfo[] {
            new SubscriberMethodInfo("onMessageEventOne", MessageEvent.class, ThreadMode.MAIN),
            new SubscriberMethodInfo("onMessageEventTwo", MessageEvent.class, ThreadMode.MAIN),
        }));

    }

    private static void putIndex(SubscriberInfo info) {
        SUBSCRIBER_INDEX.put(info.getSubscriberClass(), info);
    }

    @Override
    public SubscriberInfo getSubscriberInfo(Class<?> subscriberClass) {
        SubscriberInfo info = SUBSCRIBER_INDEX.get(subscriberClass);
        if (info != null) {
            return info;
        } else {
            return null;
        }
    }
}
```

在生成的索引类中我们可以看出：

- 生成的索引类中，维护了一个 key 为 订阅对象 value 为 `SimpleSubscriberInfo` 的 HashMap。
- `SimpleSubscriberInfo` 类中维护了当前订阅者的 class 对象与 `SubscriberMethodInfo[] 数组`。
- HashMap 中的数据添加是放到静态代码块中执行的。

>SubscriberMethodInfo 类中含有 `@Subscribe` 注解的方法信息封装（优先级，是否粘性，线程模式，订阅的事件），以及当前方法的`名称`。

到现在，我们已经知道了我们索引类中的内容，那么现在在回到 `findUsingInfo()` 方法：

```java
    private List<SubscriberMethod> findUsingInfo(Class<?> subscriberClass) {
        //省略部分代码
        while (findState.clazz != null) {
            findState.subscriberInfo = getSubscriberInfo(findState);
            if (findState.subscriberInfo != null) {
                 //👇关键代码，从索引类中获取 SubscriberMethod
                SubscriberMethod[] array = findState.subscriberInfo.getSubscriberMethods();
                for (SubscriberMethod subscriberMethod : array) {
                    if (findState.checkAdd(subscriberMethod.method, subscriberMethod.eventType)) {
                        findState.subscriberMethods.add(subscriberMethod);
                    }
                }
            }
            //省略部分代码
        }
    }
```

当 `subscriberInfo` 不为空时，会通过 `getSubscriberMethods()`方法，去获取索引类中 `SubscriberMethod[]数组` 信息。因为索引类使用的是 `SimpleSubscriberInfo` 类，我们查看该类中该方法的实现：

```java
   @Override
    public synchronized SubscriberMethod[] getSubscriberMethods() {
        int length = methodInfos.length;
        SubscriberMethod[] methods = new SubscriberMethod[length];
        for (int i = 0; i < length; i++) {
            SubscriberMethodInfo info = methodInfos[i];
            methods[i] = createSubscriberMethod(info.methodName, info.eventType, info.threadMode,
                    info.priority, info.sticky);
        }
        return methods;
    }
```

观察该代码，我们发现 SubscriberMethod 对象的创建是通过 `createSubscriberMethod` 方法创建的，我们继续跟踪。

```java
  protected SubscriberMethod createSubscriberMethod(String methodName, Class<?> eventType, ThreadMode threadMode,
                                                      int priority, boolean sticky) {
        try {
            Method method = subscriberClass.getDeclaredMethod(methodName, eventType);
            return new SubscriberMethod(method, eventType, threadMode, priority, sticky);
        } catch (NoSuchMethodException e) {
            throw new EventBusException("Could not find subscriber method in " + subscriberClass +
                    ". Maybe a missing ProGuard rule?", e);
        }
    }
```

从上述代码中，我们可以看出 `SubscriberMethod` 中的 `Method` 对象，其实是调用订阅者的 class 对象并使用 `getDeclaredMethod（）`方法找到的。

现在为止我们已经基本了解，索引类之所以相比传统的通过反射遍历去获取订阅方法效率要更高。是因为在自动生成的索引类中，已经包含了相关订阅者中的订阅方法的名称及注解信息，那么当 EventBus 注册订阅者时，就可以直接通过`方法名称`拿到 Method 对象。这样就减少了通过遍历寻找方法的时间。

### 索引类的生成

那现在我们继续学习 EventBus3 中是如何创建索引类的。索引类的创建是通过 `APT` 技术，如果你不了解这门技术，你可能需要查看文章 {% post_link Android-注解系列之APT工具(三) %}

>`APT(Annotation Processing Tool)`是 javac 中提供的一种编译时扫描和处理注解的工具，它会对源代码文件进行检查，并找出其中的注解，然后根据用户自定义的注解处理方法进行额外的处理。APT工具不仅能解析注解，还能根据注解生成其他的源文件，最终将生成的新的源文件与原来的源文件共同编译（`注意：APT并不能对源文件进行修改操作，只能生成新的文件，例如在已有的类中添加方法`）

使用APT技术需要创建自己的注解处理器，在 EventBus 中也创建了自己的注解处理器，从其源代码中我们就可以看出。

{% asset_img EventBus注解处理器.png EventBus注解处理器 %}

那下面，我们就直接查看源码：

> 以下的代码，都出至于 [EventBusAnnotationProcessor](https://github.com/greenrobot/EventBus/blob/master/EventBusAnnotationProcessor/src/org/greenrobot/eventbus/annotationprocessor/EventBusAnnotationProcessor.java)

查看 EventBusAnnotationProcessor 中的 `process()` 方法：

>`process(Set<? extends TypeElement> annotations, RoundEnvironment roundEnv)`：注解处理器实际处理方法，一般要求子类实现该抽象方法，你可以在在这里写你的扫描与处理注解的代码，以及生成 Java 文件。其中参数 RoundEnvironment ，可以让你查询出包含特定注解的被注解元素.

```java
@SupportedAnnotationTypes("org.greenrobot.eventbus.Subscribe")
@SupportedOptions(value = {"eventBusIndex", "verbose"})
public class EventBusAnnotationProcessor extends AbstractProcessor {
public static final String OPTION_EVENT_BUS_INDEX = "eventBusIndex";

@Override
    public boolean process(Set<? extends TypeElement> annotations, RoundEnvironment env) {
        Messager messager = processingEnv.getMessager();
        try {
            //步骤1：👇获取我们配置的索引类，
            String index = processingEnv.getOptions().get(OPTION_EVENT_BUS_INDEX);
            if (index == null) {
                messager.printMessage(Diagnostic.Kind.ERROR, "No option " + OPTION_EVENT_BUS_INDEX +
                        " passed to annotation processor");
                return false;
            }

            //省略部分代码

            //步骤2:👇收集当前订阅者信息
            collectSubscribers(annotations, env, messager);
            //步骤3：👇创建索引类文件
            if (!methodsByClass.isEmpty()) {
                createInfoIndexFile(index);
            } else {
                messager.printMessage(Diagnostic.Kind.WARNING, "No @Subscribe annotations found");
            }
            writerRoundDone = true;
        } catch (RuntimeException e) {
            //省略部分代码
        }
        return true;
    }
}
```

该方法中主要逻辑为三个逻辑：

- 步骤1：读取我们之前在 APP 中的 build.gradle 设置的索引类对应的包名与类名。
- 步骤2：读取源文件中的包含 `@Subscribe` 注解的方法。并将订阅者与订阅方法进行记录在 `methodsByClass` Map 集合中。
- 步骤3：根据读取的索引类设置，通过 `createInfoIndexFile()` 方法开始创建索引类文件。

> 因为声明了`@SupportedAnnotationTypes("org.greenrobot.eventbus.Subscribe")`  在注解处理器上，那么 APT 只会处理包含该注解的文件。

我们接下来看看步骤2中的方法 `collectSubscribers()` 方法：

```java
    private void collectSubscribers(Set<? extends TypeElement> annotations, RoundEnvironment env, Messager messager) {
        for (TypeElement annotation : annotations) {
            Set<? extends Element> elements = env.getElementsAnnotatedWith(annotation);
            for (Element element : elements) {
                if (element instanceof ExecutableElement) {
                    ExecutableElement method = (ExecutableElement) element;
                    if (checkHasNoErrors(method, messager)) {
                        //获取包含`@Subscribe`类的class对象
                        TypeElement classElement = (TypeElement) method.getEnclosingElement();
                        methodsByClass.putElement(classElement, method);
                    }
                } else {
                    messager.printMessage(Diagnostic.Kind.ERROR, "@Subscribe is only valid for methods", element);
                }
            }
        }
    }
```

>在注解处理过程中，我们需要扫描所有的Java源文件，源代码的每一个部分都是一个特定类型的`Element`，也就是说 Element 代表源文件中的元素，例如包、类、字段、方法等。

在上述方法中，`annotations` 为扫描到包含 `@Subscribe` 注解 的 `Element` 集合。其中 ExecutableElement 表示类或接口的方法、构造函数或初始化器（静态或实例），因为我们可以通过 getEnclosingElement（）方法，拿到当前 `ExecutableElement` 的最近的父 Element，那么我们就能获得当前的类的 element 对象了。那么通过该方法，我们就能知道所有订阅者与其对应的订阅方法了。

我们继续跟踪查看索引类文件的创建：

```java
    private void createInfoIndexFile(String index) {
        BufferedWriter writer = null;
        try {
            JavaFileObject sourceFile = processingEnv.getFiler().createSourceFile(index);
            int period = index.lastIndexOf('.');
            String myPackage = period > 0 ? index.substring(0, period) : null;
            String clazz = index.substring(period + 1);
            writer = new BufferedWriter(sourceFile.openWriter());
            if (myPackage != null) {
                writer.write("package " + myPackage + ";\n\n");
            }
            writer.write("import org.greenrobot.eventbus.meta.SimpleSubscriberInfo;\n");
            writer.write("import org.greenrobot.eventbus.meta.SubscriberMethodInfo;\n");
            writer.write("import org.greenrobot.eventbus.meta.SubscriberInfo;\n");
            writer.write("import org.greenrobot.eventbus.meta.SubscriberInfoIndex;\n\n");
            writer.write("import org.greenrobot.eventbus.ThreadMode;\n\n");
            writer.write("import java.util.HashMap;\n");
            writer.write("import java.util.Map;\n\n");
            writer.write("/** This class is generated by EventBus, do not edit. */\n");
            writer.write("public class " + clazz + " implements SubscriberInfoIndex {\n");
            writer.write("    private static final Map<Class<?>, SubscriberInfo> SUBSCRIBER_INDEX;\n\n");
            writer.write("    static {\n");
            writer.write("        SUBSCRIBER_INDEX = new HashMap<Class<?>, SubscriberInfo>();\n\n");
            //👇这里是关键的代码
            writeIndexLines(writer, myPackage);
            writer.write("    }\n\n");
            writer.write("    private static void putIndex(SubscriberInfo info) {\n");
            writer.write("        SUBSCRIBER_INDEX.put(info.getSubscriberClass(), info);\n");
            writer.write("    }\n\n");
            writer.write("    @Override\n");
            writer.write("    public SubscriberInfo getSubscriberInfo(Class<?> subscriberClass) {\n");
            writer.write("        SubscriberInfo info = SUBSCRIBER_INDEX.get(subscriberClass);\n");
            writer.write("        if (info != null) {\n");
            writer.write("            return info;\n");
            writer.write("        } else {\n");
            writer.write("            return null;\n");
            writer.write("        }\n");
            writer.write("    }\n");
            writer.write("}\n");
        } catch (IOException e) {
            throw new RuntimeException("Could not write source for " + index, e);
        } finally {
            if (writer != null) {
                try {
                    writer.close();
                } catch (IOException e) {
                    //Silent
                }
            }
        }
    }
```

在该方法中，通过 `processingEnv.getFiler().createSourceFile(index)` 拿到我们需要创建的索引类文件对象，然后通过文件IO流向该文件中输入索引类中需要的内容。在该方法中，最为主要的就是 `writeIndexLines()` 方法了。查看该方法：

```java
    private void writeIndexLines(BufferedWriter writer, String myPackage) throws IOException {
        for (TypeElement subscriberTypeElement : methodsByClass.keySet()) {
            if (classesToSkip.contains(subscriberTypeElement)) {
                continue;
            }
            //当前订阅对象的class对象
            String subscriberClass = getClassString(subscriberTypeElement, myPackage);
            if (isVisible(myPackage, subscriberTypeElement)) {
                writeLine(writer, 2,
                        "putIndex(new SimpleSubscriberInfo(" + subscriberClass + ".class,",
                        "true,", "new SubscriberMethodInfo[] {");
                List<ExecutableElement> methods = methodsByClass.get(subscriberTypeElement);
                //👇关键代码
                writeCreateSubscriberMethods(writer, methods, "new SubscriberMethodInfo", myPackage);
                writer.write("        }));\n\n");
            } else {
                writer.write("        // Subscriber not visible to index: " + subscriberClass + "\n");
            }
        }
    }
```

在该方法中，会从 `methodsByClass` Map 中遍历获取我们之前的订阅者，然后获取其所有的订阅方法，并书写模板方法。其中关构造 `SubscriberMethodInfo` 代码的关键方法为 `writeCreateSubscriberMethods()`，跟踪该方法：

```java
 private void writeCreateSubscriberMethods(BufferedWriter writer, List<ExecutableElement> methods,
                                              String callPrefix, String myPackage) throws IOException {
        for (ExecutableElement method : methods) {
            //获取当前方法上的参数
            List<? extends VariableElement> parameters = method.getParameters();
            TypeMirror paramType = getParamTypeMirror(parameters.get(0), null);
            //获取第一个参数的类型
            TypeElement paramElement = (TypeElement) processingEnv.getTypeUtils().asElement(paramType);
            //获取方法的名称
            String methodName = method.getSimpleName().toString();
            //获取订阅的事件class类型字符串信息
            String eventClass = getClassString(paramElement, myPackage) + ".class";
            //获取方法上的注解信息
            Subscribe subscribe = method.getAnnotation(Subscribe.class);
            List<String> parts = new ArrayList<>();
            parts.add(callPrefix + "(\"" + methodName + "\",");
            String lineEnd = "),";
            //设置优先级，是否粘性，线程模式，订阅事件class类型
            if (subscribe.priority() == 0 && !subscribe.sticky()) {
                if (subscribe.threadMode() == ThreadMode.POSTING) {
                    parts.add(eventClass + lineEnd);
                } else {
                    parts.add(eventClass + ",");
                    parts.add("ThreadMode." + subscribe.threadMode().name() + lineEnd);
                }
            } else {
                parts.add(eventClass + ",");
                parts.add("ThreadMode." + subscribe.threadMode().name() + ",");
                parts.add(subscribe.priority() + ",");
                parts.add(subscribe.sticky() + lineEnd);
            }
            writeLine(writer, 3, parts.toArray(new String[parts.size()]));

            if (verbose) {
                processingEnv.getMessager().printMessage(Diagnostic.Kind.NOTE, "Indexed @Subscribe at " +
                        method.getEnclosingElement().getSimpleName() + "." + methodName +
                        "(" + paramElement.getSimpleName() + ")");
            }

        }
    }
```

在该方法中，会获取订阅方法的参数信息，并构建 `SubscriberMethodInfo` 信息。这里就不对该方法进行详细的介绍了，大家可以根据代码中的注释进行理解。

### 混淆相关

在使用 EventBus3 的时候，如果你的项目采用了混淆，需要注意 keep 以下类及方法。官方中已经给出了详细的 keep 规则，如下所示：

```java
-keepattributes *Annotation*
-keepclassmembers class * {
    @org.greenrobot.eventbus.Subscribe <methods>;
}
-keep enum org.greenrobot.eventbus.ThreadMode { *; }

# Only required if you use AsyncExecutor
-keepclassmembers class * extends org.greenrobot.eventbus.util.ThrowableFailureEvent {
    <init>(java.lang.Throwable);
}
```

#### 为什么不能混淆注解

android在打包的时候，应用程序会进行代码优化，优化的过程就把注解给去掉了。为了在程序运行期间读取到注解信息，所以我们需要保存注解信息不被混淆。

#### 为什么不能混淆包含 @Subscribe 注解的方法

因为当我们在使用索引类时，获取相关订阅的方法是通过`方法名称`获取的，那么当代码被混淆过后，订阅者的方法名称将会发生改变，比如原来订阅方法名称为onMessageEvent，混淆后有可能改为a，或b方法。这个时候是找不到相关的订阅者的方法的 ，就会抛出 `Could not find subscriber method in  + subscriberClass + Maybe a missing ProGuard rule?` 的异常，所以在混淆的时候我们需要保留订阅者所有包含 `@Subscribe` 注解的方法。

#### 为什么不能混淆枚举类中的静态变量

如果我们没有在混淆规则中添加如下语句:

```java
-keep public enum org.greenrobot.eventbus.ThreadMode { public static *; }
```

在运行程序的时候，会报`java.lang.NoSuchFieldError: No static field POSTING`。原因是因为在 `SubscriberMethodFinder` 的 `findUsingReflection` 方法中，在调用 `Method.getAnnotation()`时获取 `ThreadMode` 这个 `enum` 失败了。

我们都知道当我们声明枚举类时，编译器会为我们的枚举，自动生成一个继承
`java.lang.Enum` 的 `final` 类。如下所示：

```java
//使用命令 javap ThreadMode.class
public final class com.tian.auto.ThreadMode extends java.lang.Enum<com.tian.auto.ThreadMode> {
  public static final com.tian.auto.ThreadMode POSTING;
  public static final com.tian.auto.ThreadMode MAIN;
  public static final com.tian.auto.ThreadMode MAIN_ORDERED;
  public static final com.tian.auto.ThreadMode BACKGROUND;
  public static final com.tian.auto.ThreadMode ASYNC;
  public static com.tian.auto.ThreadMode[] values();
  public static com.tian.auto.ThreadMode valueOf(java.lang.String);
  static {};
}
```

也就是说，我们在枚举中声明的元素，其实最后对应的是类中的静态公有的常量。

那么在结合在没有添加混淆规则时，程序所提示的错误信息。我们可以确定当我们在注解中包含`枚举类型`的注解元素时且设置了默认值时。该默认值是通过枚举类的 class 对象.getField(String name) 去获取的。因为只有该方法才会抛出该异常。`getField()` 代码如下所示：

```java
    public Field getField(String name)
        throws NoSuchFieldException {
        if (name == null) {
            throw new NullPointerException("name == null");
        }
        Field result = getPublicFieldRecursive(name);
        if (result == null) {
            throw new NoSuchFieldException(name);
        }
        return result;
    }
```

那么也就说如果不添加上述的 keep 规则，就会导致我们编译器自动生成的静态常量名发生变化，又因为注解中的默认枚举值，是通过 `getField(String name)` 获得的。所以就会出现找不到字段的情况。

>其实在很多情况下，我们需要添加 keep 规则，常常是因为代码中是直接拿**混淆前**的方法名称或字段名称去直接寻找**混淆后**的方法与字段名称，我们只要在项目中注意这些情况，添加相应的 keep 规则，就可以避免因为代码被混淆而产生的异常啦。

### 最后

EventBus3 中的索引类及其相关内容到这里就讲完啦！我相应大家已经了解了索引类在性能优化上的重要作用。希望大家在后续使用EventBus3时，一定要使用索引类呦。在接下来的一段时间内，我可能不会继续更新博客啦，因为作者我要去学习 flutter 去啦~ 没有办法，总要保持前进呢。优秀的人还在努力，更何况自己并不聪明呢。哎~伤心
