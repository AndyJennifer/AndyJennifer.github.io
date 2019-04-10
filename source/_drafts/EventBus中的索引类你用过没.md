---
title: 'EventBus中的索引类你用过没? '
tags:
- EventBus
categories:
- 源码分析
---

###前言
在上篇文[Android 注解系列之Annotation（三）](https://juejin.im/post/5bc4606c6fb9a05d2469e854)中，我们介绍了APT技术，也间接的提到了一些主流的库，如Dagger2、ButterKnife、EventBus等都使用了该技术。那下面，我将会对EventBus如何使用APT，及内部原理进行分析。

### EventBus简介
EventBus在平时项目开发中，我们主要用于不同组件的通讯，

### 整体结构
- subscriptionsByEventType： 类型为`Map<Class<?>, CopyOnWriteArrayList<Subscription>> `其中对应的key是相应事件类，value为相关订阅者与回调方法按照优先级排序的集合。
- Subscription：为每个订阅者与相应回调方法的关系抽象表达。
- typesBySubscriber ：类型为`Map<Object, List<Class<?>>>`其中的key对应的订阅者，value为该订阅者所有监听的事件集合。
- SubscriberMethod：订阅者中事件回调方法的封装。
- SubscriberMethodFinder：订阅者事件回调方法查找器。


### 订阅事件
```
public void register(Object subscriber) {
        Class<?> subscriberClass = subscriber.getClass();
        //获取当前订阅者中所有的事件回调方法
        List<SubscriberMethod> subscriberMethods = this.subscriberMethodFinder.findSubscriberMethods(subscriberClass);
        synchronized(this) {
            Iterator var5 = subscriberMethods.iterator();
            while(var5.hasNext()) {
                SubscriberMethod subscriberMethod = (SubscriberMethod)var5.next();
                this.subscribe(subscriber, subscriberMethod);
            }

        }
    }
```
#### 获取该订阅者中事件的回调方法
```
    List<SubscriberMethod> findSubscriberMethods(Class<?> subscriberClass) {
        //从缓存中获取是否有缓存，如果有则读缓存，如果没有进行查找
        List<SubscriberMethod> subscriberMethods = (List)METHOD_CACHE.get(subscriberClass);
        if (subscriberMethods != null) {
            return subscriberMethods;
        } else {
            if (this.ignoreGeneratedIndex) {//如果忽略索引类，则使用反射。
                subscriberMethods = this.findUsingReflection(subscriberClass);
            } else {//否则使用索引类，
                subscriberMethods = this.findUsingInfo(subscriberClass);
            }
            //如果订阅者订阅了，但是并没有响应事件的方法，则抛出异常
            if (subscriberMethods.isEmpty()) {
                throw new EventBusException("Subscriber " + subscriberClass + " and its super classes have no public methods with the @Subscribe annotation");
            } else {
                //添加到缓存中
                METHOD_CACHE.put(subscriberClass, subscriberMethods);
                return subscriberMethods;
            }
        }
    }
```
这里我们提到了索引类，这里我们先不作介绍，这里大家先知道这个概念就行了，下文中我们会进行介绍。现在我们看实际的反射过程。

```
    private List<SubscriberMethod> findUsingReflection(Class<?> subscriberClass) {
        //这里最多记录4层的继承关系
        FindState findState = prepareFindState();
        findState.initForSubscriber(subscriberClass);
        while (findState.clazz != null) {
            findUsingReflectionInSingleClass(findState);
            findState.moveToSuperclass();
        }
        return getMethodsAndRelease(findState);
    }

```
这里我们直接看通过反射获取相应订阅的方法
```
  private void findUsingReflectionInSingleClass(FindState findState) {
        Method[] methods;
        //获取当前订阅者中的所有的方法
        try {
            // This is faster than getMethods, especially when subscribers are fat classes like Activities
            methods = findState.clazz.getDeclaredMethods();
        } catch (Throwable th) {
            // Workaround for java.lang.NoClassDefFoundError, see https://github.com/greenrobot/EventBus/issues/149
            methods = findState.clazz.getMethods();
            findState.skipSuperClasses = true;
        }
        //循环遍历所有的方法，通过相关注解找到相应的事件的回调方法
        for (Method method : methods) {
            int modifiers = method.getModifiers();
            if ((modifiers & Modifier.PUBLIC) != 0 && (modifiers & MODIFIERS_IGNORE) == 0) {
                Class<?>[] parameterTypes = method.getParameterTypes();
                //找到参数为1，且该方法包含Subscrile注解的方法
                if (parameterTypes.length == 1) {
                    Subscribe subscribeAnnotation = method.getAnnotation(Subscribe.class);
                    if (subscribeAnnotation != null) {
                        Class<?> eventType = parameterTypes[0];
                        if (findState.checkAdd(method, eventType)) {//判断是否重复添加了
                            ThreadMode threadMode = subscribeAnnotation.threadMode();
                            findState.subscriberMethods.add(new SubscriberMethod(method, eventType, threadMode,
                                    subscribeAnnotation.priority(), subscribeAnnotation.sticky()));
                        }
                    }
                } else if (strictMethodVerification && method.isAnnotationPresent(Subscribe.class)) {
                    String methodName = method.getDeclaringClass().getName() + "." + method.getName();
                    throw new EventBusException("@Subscribe method " + methodName +
                            "must have exactly 1 parameter but has " + parameterTypes.length);
                }
            } else if (strictMethodVerification && method.isAnnotationPresent(Subscribe.class)) {
                String methodName = method.getDeclaringClass().getName() + "." + method.getName();
                throw new EventBusException(methodName +
                        " is a illegal @Subscribe method: must be public, non-static, and non-abstract");
            }
        }
    }
```


#### 实际订阅方法subscribe

当找到订阅者所有的方法集合后，会遍历调用`subscribe()方法`，那么继续往下走

```
private void subscribe(Object subscriber, SubscriberMethod subscriberMethod) {
		
        Class<?> eventType = subscriberMethod.eventType;
        //第一步，将每个回调方法和订阅者分装成Subscription
        Subscription newSubscription = new Subscription(subscriber, subscriberMethod);
        
        //第二步，获取对应事件中所有的回调方法，判断是否重复添加
        CopyOnWriteArrayList<Subscription> subscriptions = (CopyOnWriteArrayList)this.subscriptionsByEventType.get(eventType);
        if (subscriptions == null) {
            subscriptions = new CopyOnWriteArrayList();
            this.subscriptionsByEventType.put(eventType, subscriptions);
        } else if (subscriptions.contains(newSubscription)) {
            throw new EventBusException("Subscriber " + subscriber.getClass() + " already registered to event " + eventType);
        }

		 //第三步，根据优先级，将当前新封装的Subscription添加到subscriptionsByEventType中对应的list中
        int size = subscriptions.size();
        for(int i = 0; i <= size; ++i) {
            if (i == size || subscriberMethod.priority > ((Subscription)subscriptions.get(i)).subscriberMethod.priority) {
                subscriptions.add(i, newSubscription);
                break;
            }
        }
		
		 //第四步，将订阅者中订阅的事件类型，添加到typesBySubscriber中的list中
        List<Class<?>> subscribedEvents = (List)this.typesBySubscriber.get(subscriber);
        if (subscribedEvents == null) {
            subscribedEvents = new ArrayList();
            this.typesBySubscriber.put(subscriber, subscribedEvents);
        }

        ((List)subscribedEvents).add(eventType);
        
        //第五步，如果该方法有注册有粘性事件，则从stickyEvents中获取相应粘性事件，并发送
        if (subscriberMethod.sticky) {
            if (eventInheritance) {
                Set<Map.Entry<Class<?>, Object>> entries = stickyEvents.entrySet();
                for (Map.Entry<Class<?>, Object> entry : entries) {
                    Class<?> candidateEventType = entry.getKey();
                    if (eventType.isAssignableFrom(candidateEventType)) {
                        Object stickyEvent = entry.getValue();
                        checkPostStickyEventToSubscription(newSubscription, stickyEvent);
                    }
                }
            } else {
                Object stickyEvent = this.stickyEvents.get(eventType);
                this.checkPostStickyEventToSubscription(newSubscription, stickyEvent);
            }
        }

    }
```

整体流程

- 通过subscriberMethodFinder调用findSubscriberMethods方法，获取当前订阅者所有回调方法
- 遍历该订阅者的所有回调方法
- 把每个回调方法和订阅者封装成Subscription
- 获取对应事件中所有的回调方法，判断是否重复添加
- 根据优先级将当前新封装的Subscription添加到subscriptionsByEventType中对应的list中
- 将订阅者中订阅的事件类型，添加到typesBySubscriber中的list中
- 如果该方法有注册有粘性事件，则从stickyEvents中获取相应粘性事件，并发送


### 发送事件
发送事件，主要调用post方法，具体代码如下:
```
  public void post(Object event) {
  		 //第一步，获取当前线程信息，并获取到对应的事件队列eventQueue，并将该事件添加到当前线程的事件队列中
        EventBus.PostingThreadState postingState = (EventBus.PostingThreadState)this.currentPostingThreadState.get();
        List<Object> eventQueue = postingState.eventQueue;
        eventQueue.add(event);
        
        //判断当前线程是否正在分发事件，如果不是则循环遍历事件队列中的事件，并将事件分发出去，直到当前事件队列空为止
        if (!postingState.isPosting) {
            postingState.isMainThread = this.isMainThread();
            postingState.isPosting = true;
            //如果当前分发状态为取消则抛出异常
            if (postingState.canceled) {
                throw new EventBusException("Internal error. Abort state was not reset");
            }
			  //循环遍历事件队列，并将消息发送出去
            try {
                while(!eventQueue.isEmpty()) {
                    this.postSingleEvent(eventQueue.remove(0), postingState);
                }
            } finally {
                postingState.isPosting = false;
                postingState.isMainThread = false;
            }
        }

    }

```
在post方法内部调用了postSingleEvent（）方法，我们继续查看该方法。

```
  private void postSingleEvent(Object event, EventBus.PostingThreadState postingState) throws Error {
        Class<?> eventClass = event.getClass();
        boolean subscriptionFound = false;
        if (this.eventInheritance) {
            List<Class<?>> eventTypes = lookupAllEventTypes(eventClass);
            int countTypes = eventTypes.size();

            for(int h = 0; h < countTypes; ++h) {
                Class<?> clazz = (Class)eventTypes.get(h);
                subscriptionFound |= this.postSingleEventForEventType(event, postingState, clazz);
            }
        } else {
            subscriptionFound = this.postSingleEventForEventType(event, postingState, eventClass);
        }

        if (!subscriptionFound) {
            if (this.logNoSubscriberMessages) {
                this.logger.log(Level.FINE, "No subscribers registered for event " + eventClass);
            }

            if (this.sendNoSubscriberEvent && eventClass != NoSubscriberEvent.class && eventClass != SubscriberExceptionEvent.class) {
                this.post(new NoSubscriberEvent(this, event));
            }
        }

    }
```

```
/** Looks up all Class objects including super classes and interfaces. Should also work for interfaces. */
    private static List<Class<?>> lookupAllEventTypes(Class<?> eventClass) {
        synchronized (eventTypesCache) {
            List<Class<?>> eventTypes = eventTypesCache.get(eventClass);
            if (eventTypes == null) {
                eventTypes = new ArrayList<>();
                Class<?> clazz = eventClass;
                while (clazz != null) {
                    eventTypes.add(clazz);
                    addInterfaces(eventTypes, clazz.getInterfaces());
                    clazz = clazz.getSuperclass();
                }
                eventTypesCache.put(eventClass, eventTypes);
            }
            return eventTypes;
        }
    }

    /** Recurses through super interfaces. */
    static void addInterfaces(List<Class<?>> eventTypes, Class<?>[] interfaces) {
        for (Class<?> interfaceClass : interfaces) {
            if (!eventTypes.contains(interfaceClass)) {
                eventTypes.add(interfaceClass);
                addInterfaces(eventTypes, interfaceClass.getInterfaces());
            }
        }
    }

```
通过lookupAllEventTypes获得事件的所有父类，并遍历。然后在subscriptionsByEventType中获得每个事件类的订阅者的回调方法。遍历回调方法。根据ThreadMode,通过不同的poster在对应的线程中通过反射该invoke该方法

```
    private void postSingleEvent(Object event, PostingThreadState postingState) throws Error {
        Class<?> eventClass = event.getClass();
        boolean subscriptionFound = false;
        if (eventInheritance) {
            List<Class<?>> eventTypes = lookupAllEventTypes(eventClass);
            int countTypes = eventTypes.size();
            for (int h = 0; h < countTypes; h++) {
                Class<?> clazz = eventTypes.get(h);
                subscriptionFound |= postSingleEventForEventType(event, postingState, clazz);
            }
        } else {
            subscriptionFound = postSingleEventForEventType(event, postingState, eventClass);
        }
        if (!subscriptionFound) {
            if (logNoSubscriberMessages) {
                logger.log(Level.FINE, "No subscribers registered for event " + eventClass);
            }
            if (sendNoSubscriberEvent && eventClass != NoSubscriberEvent.class &&
                    eventClass != SubscriberExceptionEvent.class) {
                post(new NoSubscriberEvent(this, event));
            }
        }
    }
```

```
    private boolean postSingleEventForEventType(Object event, PostingThreadState postingState, Class<?> eventClass) {
        CopyOnWriteArrayList<Subscription> subscriptions;
        synchronized (this) {
            subscriptions = subscriptionsByEventType.get(eventClass);
        }
        if (subscriptions != null && !subscriptions.isEmpty()) {
            for (Subscription subscription : subscriptions) {
                postingState.event = event;
                postingState.subscription = subscription;
                boolean aborted = false;
                try {
                    postToSubscription(subscription, event, postingState.isMainThread);
                    aborted = postingState.canceled;
                } finally {
                    postingState.event = null;
                    postingState.subscription = null;
                    postingState.canceled = false;
                }
                if (aborted) {
                    break;
                }
            }
            return true;
        }
        return false;
    }

```

### 发送粘性事件


### 不同线程的切换

```
    private final Poster mainThreadPoster;
    private final BackgroundPoster backgroundPoster;
    private final AsyncPoster asyncPoster;
```

```
    private void postToSubscription(Subscription subscription, Object event, boolean isMainThread) {
        switch (subscription.subscriberMethod.threadMode) {
            case POSTING:
                invokeSubscriber(subscription, event);
                break;
            case MAIN:
                if (isMainThread) {
                    invokeSubscriber(subscription, event);
                } else {
                    mainThreadPoster.enqueue(subscription, event);
                }
                break;
            case MAIN_ORDERED:
                if (mainThreadPoster != null) {
                    mainThreadPoster.enqueue(subscription, event);
                } else {
                    // temporary: technically not correct as poster not decoupled from subscriber
                    invokeSubscriber(subscription, event);
                }
                break;
            case BACKGROUND:
                if (isMainThread) {
                    backgroundPoster.enqueue(subscription, event);
                } else {
                    invokeSubscriber(subscription, event);
                }
                break;
            case ASYNC:
                asyncPoster.enqueue(subscription, event);
                break;
            default:
                throw new IllegalStateException("Unknown thread mode: " + subscription.subscriberMethod.threadMode);
        }
    }
```
### EventBusAnnotationProcessor

如果不使用EventBus中的索引类我们可以在gradle中添加如下依赖：
```
compile 'org.greenrobot:eventbus:3.1.1'
```
如果我们需要索引加速的话，就需要添加如下依赖：
```
annotationProcessor 'org.greenrobot:eventbus-annotation-processor:3.1.1'
```
同时我们我们也需要在App的build.gradle中添加如下：[AnnotationProcessorOptions介绍](http://google.github.io/android-gradle-dsl/current/com.android.build.gradle.internal.dsl.AnnotationProcessorOptions.html)

```
javaCompileOptions {
     annotationProcessorOptions {
          arguments = [eventBusIndex: 'com.jennifer.andy.myprotice.eventbus.EventBusIndex']
            }
        }
```
这里需要注意，如果应用了EventBusAnnotationProcessor却没有设置arguments的话，编译时就会报错： 
`No option eventBusIndex passed to annotation processor`

此时需要我们先编译一次，生成索引类。编译成功之后，就会发现在\ProjectName\app\build\generated\source\apt\PakageName\下看到通过注解分析生成的索引类，这样我们便可以在初始化EventBus时应用我们生成的索引了。

```
 EventBus.builder().addIndex(new EventBusIndex()).installDefaultEventBus();
```


### 混淆相关
在使用EventBus的时候，如果你的项目采用了混淆，需要注意keep以下类及方法。官方中已经给了使用EventBus库中需要keep的类，具体如下所示：

```
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

```

    @Subscribe(threadMode = ThreadMode.MAIN)
    public void onMessageEvent(MessageEvent event) {
        System.out.println("hello");
    }
```

```
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
```
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
```

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
因为是通过记录的实际名称来寻找相应的方法的，因为混淆过后，订阅者的方法发生了改变（onMessageEvent有可能改为a()，或b()方法。所以这个时候是找不到相关的订阅者的方法的 ，就会抛出`Could not find subscriber method in  + subscriberClass + Maybe a missing ProGuard rule?`的异常，所以在混淆的时候我们需要保留订阅者所有包含`@Subscribe `注解的方法。
### 最后
