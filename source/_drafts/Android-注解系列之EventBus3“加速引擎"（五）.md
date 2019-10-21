---
title: Android-注解系列之EventBus3“加速引擎"（五）
tags:
- EventBus
categories:
- 源码分析
---

### 前言

在上篇文章 《Android 注解系列之 EventBus3 原理（四）》中我们讲解了 EventBus3 的内部原理，在该篇文章中我们将讲解 EventBus3 中的 `“加速引擎"`---索引类。阅读该篇文章我们能够学到如下知识点。

- EventBus3 索引类出现的原因
- EventBus3 索引类的使用
- EventBus3 中APT的使用
- EventBus3 混淆注意事项
  
>对 APT 技术不熟悉的小伙伴，可以查看文章 Android-注解系列之APT工具(三)

### 前景回顾

在 《Android 注解系列之 EventBus3 原理（四）》中，我们特别指出在 EventBus3 中优化了 `SubscriberMethodFinder` 获取类中包含 `@Subscribe` 注解的订阅方法的流程。使其能在 `EventBus.register()` 方法调用之前就能知道相关订阅事件的方法，这样就减少了程序在运行期间使用反射遍历获取方法所带来的时间消耗。优化点如下图中 `红色虚线框` 所示：

![EventBus3优化.jpg](https://upload-images.jianshu.io/upload_images/2824145-775ec7aee8132189.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

EventBus 作者 [Markus Junginger](http://androiddevblog.com/eventbus-3-droidcon/) 也给出了使用索引类前后 EventBus 的效率对比，如下图所示：

![eventbus3-registration-perf-nexus9m.png](https://upload-images.jianshu.io/upload_images/2824145-97e436b273ff2a1d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

从上图中，我们可以使用索引类后，EventBus 的效率有着明显的提升，而效率提升的背后，正是使用了 `APT` 技术。那么接下啦我们就来看一看 EventBus3 中是如何结合 `APT` 技术来进行优化的。

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

当 `subscriberInfo` 不为空时，会通过 `getSubscriberMethods()`方法，去获取索引类中 `SubscriberMethod[]数组` 信息。因为索引类使用的是 `SimpleSubscriberInfo` 类，我们查看该方法的实现：

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

从上述代码中，我们可以看出 `SubscriberMethod` 中的 `Method` 对象，其实是通过订阅者中的 class 对象根据索引类中的方法名称找到的。

现在为止，我们已经基本了解了索引类的使用流程。相比传统的通过反射遍历去获取订阅方法。使用索引类确实提高了不少的性能，因为在程序运行之前，EventBus 已经通过索引类知道了那些订阅方法的名称，那么当 EventBus 注册订阅者时，就可以直接通过方法名称拿到 Method 对象。

### 索引类的生成

那现在我们继续学习 EventBus 中是如何创建索引类的。在之前的文章中，我已经介绍了其实是使用了 `APT` 技术。如果你不是了解这门技术，你可能需要查看文章 《Android-注解系列之APT工具(三)》

在 EventBus 中使用了创建了自己的注解处理器，从其源代码中我们就可以看出。

![EventBus注解处理器.png](https://upload-images.jianshu.io/upload_images/2824145-5c698cc9cec0701f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

> 以下的代码，都出至于[EventBusAnnotationProcessor](https://github.com/greenrobot/EventBus/blob/master/EventBusAnnotationProcessor/src/org/greenrobot/eventbus/annotationprocessor/EventBusAnnotationProcessor.java)

那下面，我们就直接查看源码：

```java
@Override
    public boolean process(Set<? extends TypeElement> annotations, RoundEnvironment env) {
        Messager messager = processingEnv.getMessager();
        try {
            //👇获取我们配置的索引类，
            String index = processingEnv.getOptions().get(OPTION_EVENT_BUS_INDEX);
            if (index == null) {
                messager.printMessage(Diagnostic.Kind.ERROR, "No option " + OPTION_EVENT_BUS_INDEX +
                        " passed to annotation processor");
                return false;
            }
            //省略部分代码

            //👇创建索引类文件
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
```

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

```java
    private void writeIndexLines(BufferedWriter writer, String myPackage) throws IOException {
        for (TypeElement subscriberTypeElement : methodsByClass.keySet()) {
            if (classesToSkip.contains(subscriberTypeElement)) {
                continue;
            }

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


```java
 private void writeCreateSubscriberMethods(BufferedWriter writer, List<ExecutableElement> methods,
                                              String callPrefix, String myPackage) throws IOException {
        for (ExecutableElement method : methods) {
            List<? extends VariableElement> parameters = method.getParameters();
            TypeMirror paramType = getParamTypeMirror(parameters.get(0), null);
            TypeElement paramElement = (TypeElement) processingEnv.getTypeUtils().asElement(paramType);
            String methodName = method.getSimpleName().toString();
            String eventClass = getClassString(paramElement, myPackage) + ".class";

            Subscribe subscribe = method.getAnnotation(Subscribe.class);
            List<String> parts = new ArrayList<>();
            parts.add(callPrefix + "(\"" + methodName + "\",");
            String lineEnd = "),";
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

### 混淆相关

在使用EventBus的时候，如果你的项目采用了混淆，需要注意keep以下类及方法。官方中已经给了使用EventBus库中需要keep的类，具体如下所示：

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

首先，因为EventBus 3弃用了反射的方式去寻找回调方法，改用注解的方式。作者的意思是在混淆时就不用再keep住相应的类和方法。但是我们在运行时，却会报java.lang.NoSuchFieldError: No static field POSTING。网上给出的解决办法是keep住所有eventbus相关的代码：

-keep class de.greenrobot.** {*;}
其实我们仔细分析，可以看到是因为在SubscriberMethodFinder的findUsingReflection方法中，在调用Method.getAnnotation()时获取ThreadMode这个enum失败了，所以我们只需要keep住这个enum就可以了（如下）。

-keep public enum org.greenrobot.eventbus.ThreadMode { public static *; }

 因为通过APT生成的代码记录的订阅者的回调方方法是在代码混淆之前的名称，如上述代码中的onMessageEvent()方法。当通过混淆后，该方法名称有可能发生改变了，那么它有可能叫a,叫b，叫c。那么通过

因为是通过记录的实际名称来寻找相应的方法的，因为混淆过后，订阅者的方法发生了改变（onMessageEvent有可能改为a()，或b()方法。所以这个时候是找不到相关的订阅者的方法的 ，就会抛出`Could not find subscriber method in  + subscriberClass + Maybe a missing ProGuard rule?`的异常，所以在混淆的时候我们需要保留订阅者所有包含`@Subscribe`注解的方法。


### 最后

为什么枚举类，不能被混淆，因为枚举类，最终会自动生成，而注解也是自动生成的是全限定名称
官方作者：

http://androiddevblog.com/eventbus-3-droidcon/

https://blog.kaush.co/2014/12/24/implementing-an-event-bus-with-rxjava-rxbus/


https://www.jianshu.com/p/61631134498e
站在巨人的肩膀上，才能看的更远~
