---
title: Android-注解系列之EventBus索引类（五）
tags:
- EventBus
categories:
- 源码分析
---

### 前言

在上篇文章 《Android 注解系列之 EventBus 原理（四）》中我们讲解了 EventBus 的内部原理，在该篇文章中我们将讲解 EventBus 中的索引类。阅读该篇文章我们能够学到如下知识点。

- EventBus 索引类出现的原因
- EventBus 索引类的使用
- EventBus APT技术的使用
- EventBus 混淆注意事项
  
>对 APT 技术不熟悉的小伙伴，可以查看文章Android-注解系列之APT工具(三)

### 前景回顾

在 《Android 注解系列之 EventBus 原理（四）》中，我们特别指出 EventBus3 通过 `SubscriberMethodFinder` 去获取类中包含 `@Subscribe` 注解的订阅方法中进行了特别的优化
，使其能在 `EventBus.register()` 方法调用之前就能知道相关订阅事件的方法，这样就减少了程序在运行期间使用反射带来的时间消耗。下图中 `红色虚线框` 所示：

![EventBus3优化.jpg](https://upload-images.jianshu.io/upload_images/2824145-775ec7aee8132189.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

而这里的优化，正是使用了 `APT` 技术所创建的索引类。下面我们就来看一看 EventBus 中是怎么做的。

### 关键代码

在 EventBus 中获取类中包含 `@Subscribe` 注解的方法有两种方式，第一种是直接通过运行时反射获取，另一种就是通过索引类。下面我们来看看使用索引类的关键方法 `getSubscriberInfo()` ，代码如下所示：

```java
    private SubscriberInfo getSubscriberInfo(FindState findState) {
        if (findState.subscriberInfo != null && findState.subscriberInfo.getSuperSubscriberInfo() != null) {
            SubscriberInfo superclassInfo = findState.subscriberInfo.getSuperSubscriberInfo();
            if (findState.clazz == superclassInfo.getSubscriberClass()) {
                return superclassInfo;
            }
        }
        //👇这里是EventBus中优化的关键，索引类
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

从代码逻辑中我们能得出，如果 `subscriberInfoIndexes` 集合不为空的话，那么就会从 `SubscriberInfoIndex（索引类）` 中去获取 `SubscriberInfo`。

>SubscriberMethod 类其实是含有 `@Subscribe` 注解的方法信息封装，包括当前方法的 Method 对象(`java.lang.reflect` 包下的对象)，是否是粘性事件，优先级等。

那么这个 SubscriberInfoIndex 是如何来的呢？

### EventBusAnnotationProcessor

如果不使用EventBus中的索引类我们可以在gradle中添加如下依赖：

```java
compile 'org.greenrobot:eventbus:3.1.1'
```

如果我们需要索引加速的话，就需要添加如下依赖：

```java
annotationProcessor 'org.greenrobot:eventbus-annotation-processor:3.1.1'
```

同时我们我们也需要在App的build.gradle中添加如下：

>[AnnotationProcessorOptions介绍](http://google.github.io/android-gradle-dsl/current/com.android.build.gradle.internal.dsl.AnnotationProcessorOptions.html)

```java
javaCompileOptions {
     annotationProcessorOptions {
          arguments = [eventBusIndex: 'com.jennifer.andy.myprotice.eventbus.EventBusIndex']
            }
        }
```

这里需要注意，如果应用了EventBusAnnotationProcessor却没有设置arguments的话，编译时就会报错：

`No option eventBusIndex passed to annotation processor`

此时需要我们先编译一次，生成索引类。编译成功之后，就会发现在\ProjectName\app\build\generated\source\apt\PakageName\下看到通过注解分析生成的索引类，这样我们便可以在初始化EventBus时应用我们生成的索引了。

```java
 EventBus.builder().addIndex(new EventBusIndex()).installDefaultEventBus();
```

#### 注解处理器源代码

![EventBus注解处理器.png](https://upload-images.jianshu.io/upload_images/2824145-5c698cc9cec0701f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

```java
@Override
    public boolean process(Set<? extends TypeElement> annotations, RoundEnvironment env) {
        Messager messager = processingEnv.getMessager();
        try {
            String index = processingEnv.getOptions().get(OPTION_EVENT_BUS_INDEX);
            if (index == null) {
                messager.printMessage(Diagnostic.Kind.ERROR, "No option " + OPTION_EVENT_BUS_INDEX +
                        " passed to annotation processor");
                return false;
            }
            verbose = Boolean.parseBoolean(processingEnv.getOptions().get(OPTION_VERBOSE));
            int lastPeriod = index.lastIndexOf('.');
            String indexPackage = lastPeriod != -1 ? index.substring(0, lastPeriod) : null;

            round++;
            if (verbose) {
                messager.printMessage(Diagnostic.Kind.NOTE, "Processing round " + round + ", new annotations: " +
                        !annotations.isEmpty() + ", processingOver: " + env.processingOver());
            }
            if (env.processingOver()) {
                if (!annotations.isEmpty()) {
                    messager.printMessage(Diagnostic.Kind.ERROR,
                            "Unexpected processing state: annotations still available after processing over");
                    return false;
                }
            }
            if (annotations.isEmpty()) {
                return false;
            }

            if (writerRoundDone) {
                messager.printMessage(Diagnostic.Kind.ERROR,
                        "Unexpected processing state: annotations still available after writing.");
            }
            collectSubscribers(annotations, env, messager);
            checkForSubscribersToSkip(messager, indexPackage);

            if (!methodsByClass.isEmpty()) {
                createInfoIndexFile(index);
            } else {
                messager.printMessage(Diagnostic.Kind.WARNING, "No @Subscribe annotations found");
            }
            writerRoundDone = true;
        } catch (RuntimeException e) {
            // IntelliJ does not handle exceptions nicely, so log and print a message
            e.printStackTrace();
            messager.printMessage(Diagnostic.Kind.ERROR, "Unexpected error in EventBusAnnotationProcessor: " + e);
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

### 索引类的生成

```java
    @Subscribe(threadMode = ThreadMode.MAIN)
    public void onMessageEvent(MessageEvent event) {
        System.out.println("hello");
    }
```

```java
public class EventBusIndex implements SubscriberInfoIndex {
    private static final Map<Class<?>, SubscriberInfo> SUBSCRIBER_INDEX;

    static {
        SUBSCRIBER_INDEX = new HashMap<Class<?>, SubscriberInfo>();

        putIndex(new SimpleSubscriberInfo(Activity1.class, true, new SubscriberMethodInfo[] {
            new SubscriberMethodInfo("onMessageEvent", MessageEvent.class, ThreadMode.MAIN),
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

```java
    @Subscribe(threadMode = ThreadMode.MAIN)
    public void onMessageEvent(MessageEvent event) {
        System.out.println("hello");
    }
```

```java
public class EventBusIndex implements SubscriberInfoIndex {
    private static final Map<Class<?>, SubscriberInfo> SUBSCRIBER_INDEX;

    static {
        SUBSCRIBER_INDEX = new HashMap<Class<?>, SubscriberInfo>();

        putIndex(new SimpleSubscriberInfo(Activity1.class, true, new SubscriberMethodInfo[] {
            new SubscriberMethodInfo("onMessageEvent", MessageEvent.class, ThreadMode.MAIN),
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

 因为通过APT生成的代码记录的订阅者的回调方方法是在代码混淆之前的名称，如上述代码中的onMessageEvent()方法。当通过混淆后，该方法名称有可能发生改变了，那么它有可能叫a,叫b，叫c。那么通过

```java
public class SimpleSubscriberInfo extends AbstractSubscriberInfo {

    private final SubscriberMethodInfo[] methodInfos;

    public SimpleSubscriberInfo(Class subscriberClass, boolean shouldCheckSuperclass, SubscriberMethodInfo[] methodInfos) {
        super(subscriberClass, null, shouldCheckSuperclass);
        this.methodInfos = methodInfos;
    }

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
}
```

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

因为是通过记录的实际名称来寻找相应的方法的，因为混淆过后，订阅者的方法发生了改变（onMessageEvent有可能改为a()，或b()方法。所以这个时候是找不到相关的订阅者的方法的 ，就会抛出`Could not find subscriber method in  + subscriberClass + Maybe a missing ProGuard rule?`的异常，所以在混淆的时候我们需要保留订阅者所有包含`@Subscribe`注解的方法。




### 最后

为什么枚举类，不能被混淆，因为枚举类，最终会自动生成，而注解也是自动生成的是全限定名称
官方作者：

http://androiddevblog.com/eventbus-3-droidcon/

https://blog.kaush.co/2014/12/24/implementing-an-event-bus-with-rxjava-rxbus/


https://www.jianshu.com/p/61631134498e
站在巨人的肩膀上，才能看的更远~
