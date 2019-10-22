---
title: Android 注解系列之 EventBus3 原理（四）
tags:
  - EventBus
categories:
  - 源码分析
date: 2019-10-23 00:17:31
---


{% asset_img bus.jpg bus %}

### 前言

在之前的文章 {% post_link Android-注解系列之APT工具(三) %}中，我们介绍了 APT 技术的及其使用方式，也提到了一些知名的开源框架如 [Dagger2](https://github.com/google/dagger)、[ButterKnife](https://github.com/JakeWharton/butterknife)、[EventBus](https://github.com/greenrobot/EventBus) 都使用了该技术。为了让大家更好的了解 APT 技术的使用，在接下来的文章中我将会着重带领大家来了解 EventBus 中 APT 技术的使用，在了解该知识之前，需要我们对 EventBus 内部原理较为熟悉，如果你已经熟悉其内部机制了，可以跳过该篇文章，直接阅读直接阅读 {% post_link Android-注解系列之EventBus3“加速引擎”（五）%}

阅读该篇文章，我们能够学到如下知识点：

- EventBus3 内部原理
- EventBus3 订阅与发送消息原理
- EventBus3 线程切换的原理
- EventBus3 粘性事件的处理
  
> 整篇文章结合 EventBus 3.1.1 版本进行讲解。

### EventBus 简介

EventBus 对于 Android 程序员来说应该不是很陌生，它是基于观察者模式的事件发布/订阅框架，我们常常用它来实现不同组件的通讯，后台线程通信等。

{% asset_img EventBus-Publish-Subscribe.png EventBus-Publish-Subscribe %}

虽然 EventBus 非常简单好用，但是还是会因为 EventBus 满天飞，使程序代码结构非常混乱，难以测试和追踪。即使 EventBus 有很多诟病，但仍然不影响我们去学习其中的原理与编程思想~

### 大概流程

在了解 EventBus 内部原理之前，我们先了解一下 EventBus 框架的一个大概流程。如下图所示：

{% asset_img EventBus粗暴理解.jpg EventBus粗暴理解 %}

>上图中`绿色`为订阅流程，`红色`为发送事件流程，大家可以结合上图，来理解源码。

在上图中我们在 `A.java` 中订阅了事件 `AEvent`，在 `B.java` 中订阅了事件 `AEvent` 与 `BEvent`，下面我们来分析 EventBus 中注册与事件发送的两个流程，在介绍两个流程之前，先介绍一下 `Subscription` 与 `SubscriberMethod` 中所包含的内容。

`Subscription` 类中包含以下内容：

- 当前注册对象
- 对应订阅方法的封装对象 SubscriberMethod
  
`SubscriberMethod` 类中包含以下内容：

- 包含 `@Subscribe` 注解的方法的 Method (`java.lang.reflect` 包下的对象)。
- `@Subscribe` 注解中设置的线程模式 ThreadMode
- 方法的注册的事件类型的 Class 对象
- `@Subscribe`中设置的优先级 priority
- `@Subscribe`中设置事件是否是粘性事件 sticky

#### 注册流程

当我们通过调用 EventBus.register() 注册 A、B 两个对象时，EventBus 会做以下几件事件：

- 通过内部的 `SubscriberMethodFinder` 来获取 A、B类中含有 `@Subscribe` 注解的方法，并将该注解中的内容与对应方法封装为 `SubscriberMethod` 对象。然后再将当前订阅对象与对应的 `SubscriberMethod` 再封装为 `Subscription` 对象。
- 将所有的 `Subscription` 放在名为 `subscriptionsByEventType` 类型为 `Map<Class<?>, CopyOnWriteArrayList<Subscription>>` 数据结构（key 为事件类型的 Class 对象） 中，因为 `Subscription` 对象内部包含 `SubscriberMethod`， 那么就能知道订阅的事件类型，所以我们可以根据事件类型来区分 `Subscription` ，又因为相同事件可以被不同订阅者中的方法来订阅，所以相同类型的事件也就以对应不同的 `Subscription`。
- 将订阅者中的所有订阅的事件都封装在名为 `typesBySubscriber` 类型为 `Map<Object, List<Class<?>>>`数据结构（key 为订阅对象，value 为该对象订阅的事件类型 Class 对象）。该集合主要用于取消订阅，在下文中我们会进行介绍。

在整个注册流程中，最主要的流程就是 EventBus 通过 `SubscriberMethodFinder` 去获取类中包含 `@Subscribe` 注解的订阅方法。在 EventBus 3.0 之前该流程一直都是通过`反射`的方式去获取。在 3.0 及以后版本，EventBus 采用了 APT 技术，对 `SubscriberMethodFinder` 查找订阅方法流程进行了优化，使其能在 `EventBus.register()` 方法调用之前就能知道相关订阅事件的方法，这样就减少了程序在运行期间使用反射遍历获取方法所带来的时间消耗。在下文中我们也会指出具体的优化点。

#### 事件发送流程

知道了 EventBus 的注册过程，再来了解事件的发生流程就非常简单了。因为我们已经通过 `subscriptionsByEventType` 存储事件对应的 `Subscription`，只要找到了 `Subscription` ，那么我们就能从 `Subscription` 拿到订阅事件的对象 `subscriber` ，以及对应的订阅方法 Method (`java.lang.reflect` 包下的对象)。然后通过反射调用：

> Subscription 内部包含订阅者及 SubscriberMethod（内部包含订阅方法 Method )

```java
 method.invoke(subscription.subscriber, event)
````

通过上述方法，就能将对应事件发送到相关订阅者了。当然这里只是简单的介绍了事件是如何发送到相关订阅者的。关于 EventBus 中粘性事件的处理，线程如何切换。会在下文中进行详细介绍。

### 源码分析

在了解了 EventBus 的内部大概流程后，现在我们通过源码来更深层次的了解其内部实现。还是从订阅过程与事件的发送两个过程进行讲解。

#### 订阅过程源码分析

EventBus 的订阅入口为 register() 方法，如下所示：

```java
  public void register(Object subscriber) {
        Class<?> subscriberClass = subscriber.getClass();
        //流程1：获取对应类中所有的订阅方法
        List<SubscriberMethod> subscriberMethods = subscriberMethodFinder.findSubscriberMethods(subscriberClass);
        synchronized (this) {
            //流程2：实际订阅
            for (SubscriberMethod subscriberMethod : subscriberMethods) {
                subscribe(subscriber, subscriberMethod);
            }
        }
    }
```

在该方法中，主要涉及到 SubscriberMethodFinder 查找方法与实际订阅两个流程，下面我们会对这两个流程进行介绍。

#### SubscriberMethodFinder 查找方法流程

在该流程中，主要通过 `SubscriberMethodFinder` 去获取订阅者中所有的 SubscriberMethod ，我们先看 `findSubscriberMethods()` 方法：

```java
    List<SubscriberMethod> findSubscriberMethods(Class<?> subscriberClass) {
        //从缓存中获取订阅者中的订阅方法，如果有则读缓存，如果没有进行查找
        List<SubscriberMethod> subscriberMethods = (List)METHOD_CACHE.get(subscriberClass);
        if (subscriberMethods != null) {
            return subscriberMethods;
        } else {
            if (this.ignoreGeneratedIndex) {//如果忽略索引类，则使用反射。
                subscriberMethods = this.findUsingReflection(subscriberClass);
            } else {//否则使用索引类
                subscriberMethods = this.findUsingInfo(subscriberClass);
            }
            //如果订阅者没有订阅方法，则抛出异常
            if (subscriberMethods.isEmpty()) {
                throw new EventBusException("Subscriber " + subscriberClass + " and its super classes have no public methods with the @Subscribe annotation");
            } else {
                //将对应类中的订阅方法，添加到缓存中，提高效率，方便下次查找
                METHOD_CACHE.put(subscriberClass, subscriberMethods);
                return subscriberMethods;
            }
        }
    }
```

该方法的逻辑也非常简单，为如下几个步骤：

- 步骤1：先从缓存( `METHOD_CACHE` )中获取订阅者对应的 `SubscriberMethod（订阅方法)` ，如果有则从缓存中取。
- 步骤2：如果缓存中没有，则通过布尔变量 `ignoreGeneratedIndex`，来判断是直接使用反射获取订阅方法，还是通过`索引类`(EventBus 3.0 使用APT 增加的类）来获取。因为 `ignoreGeneratedIndex` 默认值为 false ，则默认会走 `findUsingInfo()` 方法
- 步骤3：将步骤2中获得的订阅方法集合，存储到缓存中，方便下一次获取，提高效率。

因为默认会走 `findUsingInfo()` 方法，我们继续查看该方法：

``` java
    private List<SubscriberMethod> findUsingInfo(Class<?> subscriberClass) {
        //步骤1：构建了查询状态缓存池，最多缓存4个类的查询状态
        FindState findState = prepareFindState();
        findState.initForSubscriber(subscriberClass);
        while (findState.clazz != null) {
            //步骤2，获取查找状态对应的订阅信息，👇这里EventBus 3.0 使用了索引类，
            findState.subscriberInfo = getSubscriberInfo(findState);
            if (findState.subscriberInfo != null) {
                SubscriberMethod[] array = findState.subscriberInfo.getSubscriberMethods();
                for (SubscriberMethod subscriberMethod : array) {
                    //将订阅者的所有的订阅方法添加到FindState的集合中
                    if (findState.checkAdd(subscriberMethod.method, subscriberMethod.eventType)) {
                        findState.subscriberMethods.add(subscriberMethod);
                    }
                }
            } else {//步骤3：如果订阅信息为null，则通过反射来获取类中所有的方法
                findUsingReflectionInSingleClass(findState);
            }// 继续查找父类的方法
            findState.moveToSuperclass();
        }
        //步骤4，获取findState中的所有方法，并清空对象池
        return getMethodsAndRelease(findState);
    }
```

- 步骤1：创建与订阅者相关的 FindState 对象。会从 FinState 对象缓存池(最大为4个)中获取，一个订阅者对象对应一个FindState，一个订阅者对象对应一个或多个订阅方法。
- 步骤2：通过 FindState 对象 调用 `getSubscriberInfo()` 方法去获取订阅者相关的订阅方法信息。该方法使用了 APT 技术，构建了EventBus的索引类。关于具体的优化，会在下篇文章中 {% post_link Android-注解系列之EventBus3“加速引擎”（五）%} 进行描述，大家这里有个印象就好了。
- 步骤3：如果通过步骤2获取不到订阅方法信息，则通过`反射`来获取类中的所有的订阅方法。并将获取的方法，封装到 FindState 中的 subscriberMethods 集合中去。
- 步骤4：将 FindState 对象中的 subscriberMethods 集合返回。

在上述方法中，我们需要注意的是，如果当前订阅着没有相关的订阅方法，那么会依次遍历其父类的订阅方法。还有一个知识点，就是该方法中 FindState 使用了 `对象缓存池`，不会每次注册一个订阅者就创建 一个FindState 对象。这样就节约了内存的使用。

关于索引类的知识点，会在下篇文章中进行介绍，这里我们直接查看 `findUsingReflectionInSingleClass()` 方法：

```java
  private void findUsingReflectionInSingleClass(FindState findState) {
        Method[] methods;
        try {
             //获取当前订阅者中的所有的方法
            methods = findState.clazz.getDeclaredMethods();
        } catch (Throwable th) {
            //获取该类的所有public 方法 包括继承的公有方法
            methods = findState.clazz.getMethods();
            findState.skipSuperClasses = true;
        }
        //循环遍历所有的方法，通过相关注解找到相应的订阅方法。
        for (Method method : methods) {
            int modifiers = method.getModifiers();
            //满足修饰符为 public 并且非抽象、非静态
            if ((modifiers & Modifier.PUBLIC) != 0 && (modifiers & MODIFIERS_IGNORE) == 0) {
                Class<?>[] parameterTypes = method.getParameterTypes();
                //找到参数为1，且该方法包含Subscrile注解的方法
                if (parameterTypes.length == 1) {
                    Subscribe subscribeAnnotation = method.getAnnotation(Subscribe.class);
                    if (subscribeAnnotation != null) {
                        Class<?> eventType = parameterTypes[0];
                        if (findState.checkAdd(method, eventType)) {
                            // 创建订阅方法对象，并将对应方法对象，事件类型，线程模式，优先级，粘性事件封装到SubscriberMethod对象中。
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

该方法的逻辑也非常简单，通过获取 FindState 中的订阅者的 Class 对象，然后通过反射获取所有包含 `@Subscribe` 注解且参数为 `1` 的 Method 对象，并读取到该参数的类型`EventType`，接着读取注解中的 `thredMode`、`priority`、`sticy`，最后将这些数据都统一分装到新建的`SubscriberMethod` 对象中，最后将该对象添加到 FindState 中的 subscriberMethods 集合中去。

#### 实际订阅方法 subscribe

当找到订阅者所有的方法集合后，最终会遍历调用 `subscribe()` 方法，查看该方法：

```java
private void subscribe(Object subscriber, SubscriberMethod subscriberMethod) {
        Class<?> eventType = subscriberMethod.eventType;

        //步骤1，将每个订阅方法和订阅者封装成Subscription
        Subscription newSubscription = new Subscription(subscriber, subscriberMethod);

        //步骤2，获取对应事件中所有的 Subscription，判断是否重复添加
        CopyOnWriteArrayList<Subscription> subscriptions = (CopyOnWriteArrayList)this.subscriptionsByEventType.get(eventType);
        if (subscriptions == null) {
            subscriptions = new CopyOnWriteArrayList();
            this.subscriptionsByEventType.put(eventType, subscriptions);
        } else if (subscriptions.contains(newSubscription)) {
            throw new EventBusException("Subscriber " + subscriber.getClass() + " already registered to event " + eventType);
        }

        //步骤3，根据优先级，将当前新封装的Subscription对象添加到subscriptionsByEventType中去
        int size = subscriptions.size();
        for(int i = 0; i <= size; ++i) {
            if (i == size || subscriberMethod.priority > ((Subscription)subscriptions.get(i)).subscriberMethod.priority) {
                subscriptions.add(i, newSubscription);
                break;
            }
        }

       //步骤4，将当前订阅者中与当前订阅者所订阅的事件类型，添加到typesBySubscriber中去
        List<Class<?>> subscribedEvents = (List)this.typesBySubscriber.get(subscriber);
        if (subscribedEvents == null) {
            subscribedEvents = new ArrayList();
            this.typesBySubscriber.put(subscriber, subscribedEvents);
        }
        subscribedEvents.add(eventType);

        //步骤5，如果该方法有订阅了粘性事件，则从stickyEvents中获取相应粘性事件，并发送
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

在上述方法中主要流程如下：

- 步骤1，将每个订阅方法和订阅者封装成 Subscription。
- 步骤2，获取对应事件中所有的 Subscription ，判断是否重复添加。
- 步骤3，根据 `优先级`，将当前新封装的 Subscription 对象添加到 subscriptionsByEventType 中去。(设置了优先级后，EvenBus 就可以按照优先级顺序，将事件发送给订阅者)
- 步骤4，将当前订阅者中与当前订阅者所订阅的事件类型，添加到 typesBySubscriber 中去。
- 步骤5，如果该方法有订阅了粘性事件，则从 stickyEvents 中获取相应粘性事件，并发送。

再结合我们最开始所画的 EventBus 大致流程，该方法其实就做了下图`红色虚线框`中的事：

{% asset_img subscribe()实际做的事.jpg subscribe()实际做的事 %}

关于粘性事件的知识点，需要我们了解事件的发送流程，我们会在下文进行详细介绍。

### 事件发送流程源码分析

事件的发送，主要分为`简单事件`与`粘性事件`，分别对应方法为  `post()` 与 `postSticky()` 两个方法。这里我们先看简单事件的发送，代码如下:

#### 简单事件的发送

```java
  public void post(Object event) {
     //步骤1，获取当前线程中独立拥有的PostingThreadState，并从中获取事件队列（eventQueue），将发送的事件添加到该队列中
        EventBus.PostingThreadState postingState = (EventBus.PostingThreadState)this.currentPostingThreadState.get();
        List<Object> eventQueue = postingState.eventQueue;
        eventQueue.add(event);

        //步骤2：判断当前线程是否正在分发事件，如果不是，则循环遍历事件队列中的事件，并将事件分发出去，直到当前事件队列空为止
        if (!postingState.isPosting) {
            postingState.isMainThread = this.isMainThread();
            postingState.isPosting = true;
            //如果当前分发事件状态为取消，则抛出异常
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

在 EventBus 中会为个每调用 post() 方法的线程都会创建一个唯一的 `PostingThreadState` 对象，用于记录当前线程存储发送消息与发送的状态，其内部结构如下所示：

{% asset_img PositingThreadState与线程的关系.jpg PositingThreadState与线程的关系 %}

> PostingThreadState 使用了 ThreadLocal 不熟悉 ThreadLocal 的小伙伴，可以查看该篇文章：Android Handler机制之ThreadLocal

也就是说当我们调用 `EventBus.post()` 方法，其实是从 EventQueue 队列中取出消息，然后通过调用 postSingleEvent（）方法 来实际发送消息，该方法代码如下所示：

```java
  private void postSingleEvent(Object event, EventBus.PostingThreadState postingState) throws Error {
        Class<?> eventClass = event.getClass();
        boolean subscriptionFound = false;
        //步骤1：👇判断否事件传递发送
        if (this.eventInheritance) {
            List<Class<?>> eventTypes = lookupAllEventTypes(eventClass);
            int countTypes = eventTypes.size();
            for(int h = 0; h < countTypes; ++h) {
                Class<?> clazz = (Class)eventTypes.get(h);
                //👇循环遍历遍历事件并发送
                subscriptionFound |= this.postSingleEventForEventType(event, postingState, clazz);
            }
        } else {
            //步骤2：👇如果不支持事件的传递，那么这里开始发送事件。
            subscriptionFound = this.postSingleEventForEventType(event, postingState, eventClass);
        }
        //步骤3：如果没有找到订阅的方式，提示用户
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

该方法主要为如下三个步骤：

- 步骤1：通过布尔变量 `eventInheritance` 判断是否支持事件是否传递发送，如果支持，那么通过`lookupAllEventTypes()` 方法获得发送事件祖先类及其接口。然后通过 `postSingleEventForEventType()`方法，将它们都发送出去，
- 步骤2：步骤1返回 false 那么就直接使用 `postSingleEventForEventType()` 方法发送事件。
- 步骤3：如果没有找到相关的订阅方法，那么就提示用户没有相关的订阅方法。

> 布尔变量 `eventInheritance` 默认为 `false` ,我们可以通过 EventBusBuilder 来配置该变量的值。

那什么是事件的传递发送呢？我们来查看 `lookupAllEventTypes()`方法：

```java

    private static List<Class<?>> lookupAllEventTypes(Class<?> eventClass) {
        synchronized (eventTypesCache) {
            List<Class<?>> eventTypes = eventTypesCache.get(eventClass);
            if (eventTypes == null) {
                eventTypes = new ArrayList<>();
                Class<?> clazz = eventClass;
                //👇获取该类所有祖先类及其接口
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

    //将接口添加到集合中
    static void addInterfaces(List<Class<?>> eventTypes, Class<?>[] interfaces) {
        for (Class<?> interfaceClass : interfaces) {
            if (!eventTypes.contains(interfaceClass)) {
                eventTypes.add(interfaceClass);
                addInterfaces(eventTypes, interfaceClass.getInterfaces());
            }
        }
    }
```

在该方法中，会获取发送事件的所有的祖先类及其接口，最后将他们以集合的方式返回，在 `postSingleEvent` 方法中拿到这个集合之后，那么就会将集合中所有的数据都发送出去。这样做会造成什么效果呢？如果当前我们的继承体系为 Aevent -> Bevent -> Cevent ( `->` 表示继承)，那么通过发送 Aevent，那么其他所有订阅过 Bevent 及 Cevent 的订阅者都会收到消息。

我们继续查看 `postSingleEventForEventType()` 方法，代码如下所示：

```java
    private boolean postSingleEventForEventType(Object event, PostingThreadState postingState, Class<?> eventClass) {
        CopyOnWriteArrayList<Subscription> subscriptions;
        //👇从缓存中拿取之前存取的 Subscription
        synchronized (this) {
            subscriptions = subscriptionsByEventType.get(eventClass);
        }
        if (subscriptions != null && !subscriptions.isEmpty()) {
            for (Subscription subscription : subscriptions) {
                postingState.event = event;
                postingState.subscription = subscription;
                boolean aborted = false;
                try {
                    //👇这里找到相应的方法后，开始切换线程了。
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

该方法的逻辑非常简单，就是从我们之前的 `subscriptionsByEventType` 集合中拿到存储的 `Subscription`，并根据当前线程状态设置关联的 `PostingState` 中 `canceled` 、`subscription` 、`isMainThread` 等属性值，然后通过 `postToSubscription()` 方法来真正的执行事件的传递。

到目前为止整个流程如下所示：

{% asset_img 简单事件的发送.jpg 简单事件的发送 %}

#### postToSubscription()

postToSubscription() 方法是真正实际将事件传递到订阅者的代码。查看该方法：

```java
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

从上述方法中，我们拿到 `Subscription` 中成员变量 `SubscriberMethod` 中的线程模式 `threadMode` 来判断订阅方法需要执行的线程。如果当前线程模式是 `POSTING` ，那么默认就直接调用 `invokeSubscriber()` 方法。具体代码如下所示：

```java
    void invokeSubscriber(Subscription subscription, Object event) {
        try {
            //👇直接通过反射调用订阅方法。
            subscription.subscriberMethod.method.
            invoke(subscription.subscriber, event);
        }
        //省略部分代码
    }
```

如果为其他模式，那么会根据相应的 `poster` 调用 `enqueue()` 方法来控制执行订阅方法所在的线程。在 EventBus 中提供了如下三个 Poster 来控制订阅方法的所运行的线程。

- HandlerPoster （切换到主线程）
- BackgroundPoster （切换到后台线程）
- AsyncPoster （切换到后台线程）

以上三个 Poster 都实现了 Poster 接口，且内部都维护了一个名为 `PendingPostQueue` 的队列，该队列以 `PendingPost` 为存储单元，其中 `PendingPost` 中存储内容为我们根据当前事件所找到的 `Subscription` 与当前所发生的事件。

那么结合整个流程，我们能得到下图：

{% asset_img 简单事件发送的整个流程.jpg 简单事件发送的整个流程 %}

针对上图，再进行一下简单的说明。

- 当我们调用 `EventBus.post()` 发送简单事件时，会将该事件放入与线程相关的 `PostingThreadState` 的 `EventQueue` 中。
- 接着会从之前在 `subscriptionsByEventType` 集合中找到与该事件相关的 `Subscription`。
- 接着将找到的 `Subscription` 与当前所发送的事件都封装为 `PendingPost` 并添加到对应 `Poster` 中的 `PendingPostQueue` 队列中。
- 最后对应的 `Poster` 从队列中取出相应的 `PendingPost`,通过反射调用订阅者的订阅方法。

其中订阅方法执行线程的规则，如下所示：

{% asset_img 订阅方法规则.png 订阅方法规则 %}

#### 线程的切换

在上节中，订阅者的订阅方法执行的所在线程，是由 EventBus 中内部的三个 `Poster`来实现的。那下面我们就来看看这三个 `Poster` 的实现。

##### HandlerPoster

```java
public class HandlerPoster extends Handler implements Poster {

    private final PendingPostQueue queue;
    private final int maxMillisInsideHandleMessage;
    private final EventBus eventBus;
    private boolean handlerActive;

    //默认会传递主线程的Looper
    protected HandlerPoster(EventBus eventBus, Looper looper, int maxMillisInsideHandleMessage) {
        super(looper);
        this.eventBus = eventBus;
        this.maxMillisInsideHandleMessage = maxMillisInsideHandleMessage;
        queue = new PendingPostQueue();
    }

    public void enqueue(Subscription subscription, Object event) {
        PendingPost pendingPost = PendingPost.obtainPendingPost(subscription, event);
        synchronized (this) {
            //👇这里将PedingPost放入PendingPostQueue中，然后发送消息
            queue.enqueue(pendingPost);
            if (!handlerActive) {
                handlerActive = true;
                if (!sendMessage(obtainMessage())) {
                    throw new EventBusException("Could not send handler message");
                }
            }
        }
    }

    @Override
    public void handleMessage(Message msg) {
        boolean rescheduled = false;
        try {
            long started = SystemClock.uptimeMillis();
            while (true) {
                //👇从队列中取出最近的PendingPost
                PendingPost pendingPost = queue.poll();
                if (pendingPost == null) {
                    synchronized (this) {
                        // Check again, this time in synchronized
                        pendingPost = queue.poll();
                        if (pendingPost == null) {
                            handlerActive = false;
                            return;
                        }
                    }
                }
                //👇直接通过反射，调用订阅者的订阅方法。
                eventBus.invokeSubscriber(pendingPost);
                long timeInMethod = SystemClock.uptimeMillis() - started;
                if (timeInMethod >= maxMillisInsideHandleMessage) {
                    if (!sendMessage(obtainMessage())) {
                        throw new EventBusException("Could not send handler message");
                    }
                    rescheduled = true;
                    return;
                }
            }
        } finally {
            handlerActive = rescheduled;
        }
    }
}
```

HanderPoster 中的逻辑非常容易理解，继承 Handler，并在初始化的时候默认会关联 `主线程` 的 Looper，这样该 Handler 所发送的消息将会在主线程中被处理。

分析一下 HanderPoster 中主要的步骤：

- 在调用 `enqueue()` 方法时，会将之前我们封装好的 `PendingPost` 放入 `PendingPostQueue` 队列中，同时发送消息。
- 在 `handleMessage()` 方法中，从 `PendingPostQueue` 队列中取出最近的 `PendingPost`，然后直接通过 `eventBus.invokeSubscriber()` 反射执行订阅者的订阅方法。

##### BackgroundPoster

```java
final class BackgroundPoster implements Runnable, Poster {

    private final PendingPostQueue queue;
    private final EventBus eventBus;

    private volatile boolean executorRunning;

    BackgroundPoster(EventBus eventBus) {
        this.eventBus = eventBus;
        queue = new PendingPostQueue();
    }

    public void enqueue(Subscription subscription, Object event) {
        PendingPost pendingPost = PendingPost.obtainPendingPost(subscription, event);
        //使用线程池来提交任务，该方法是线程安全的。
        synchronized (this) {
            queue.enqueue(pendingPost);
            if (!executorRunning) {
                executorRunning = true;
                eventBus.getExecutorService().execute(this);
            }
        }
    }

    @Override
    public void run() {
        try {
            try {
                while (true) {
                    PendingPost pendingPost = queue.poll(1000);
                    if (pendingPost == null) {
                        synchronized (this) {
                            // Check again, this time in synchronized
                            pendingPost = queue.poll();
                            if (pendingPost == null) {
                                executorRunning = false;
                                return;
                            }
                        }
                    }
                    eventBus.invokeSubscriber(pendingPost);
                }
            } catch (InterruptedException e) {
                eventBus.getLogger().log(Level.WARNING, Thread.currentThread().getName() + " was interruppted", e);
            }
        } finally {
            executorRunning = false;
        }
    }
}
```

BackgroundPoster 与 HandlerPoster 最大的不同是其内部使用了线程池，并且该类也实现了 Runnable 接口。

在 BackgroundPoster 中的 `enqueue()` 方法中，默认会使用 EventBus 中默认的线程池 `DEFAULT_EXECUTOR_SERVICE`来提交任务 ，该线程池的声明如下：

```java
 private final static ExecutorService DEFAULT_EXECUTOR_SERVICE = Executors.newCachedThreadPool();
```

> CachedThreadPool 适用于大量的且耗时较少的任务

同样的，BackgroundPoster 也就是通过反射调用订阅者的订阅方法，只不过不同的是它是放入线程池中的非主线程中进行执行。

需要注意的是不管是在任何线程中发送消息，EventBus 总是线程安全的。从 BackgroundPoster 的代码中我们就可以看出。

##### AsyncPoster

```java
class AsyncPoster implements Runnable, Poster {

    private final PendingPostQueue queue;
    private final EventBus eventBus;

    AsyncPoster(EventBus eventBus) {
        this.eventBus = eventBus;
        queue = new PendingPostQueue();
    }

    public void enqueue(Subscription subscription, Object event) {
        PendingPost pendingPost = PendingPost.obtainPendingPost(subscription, event);
        queue.enqueue(pendingPost);
        eventBus.getExecutorService().execute(this);
    }

    @Override
    public void run() {
        PendingPost pendingPost = queue.poll();
        if(pendingPost == null) {
            throw new IllegalStateException("No pending post available");
        }
        eventBus.invokeSubscriber(pendingPost);
    }
}
```

这里就不对 AsyncPoster 进行讲解了，相信大家根据之前的内容也能理解。

#### 粘性事件的发送

现在我们还剩最后一个知识点了，就是粘性事件的发送。在 EventBus 中发送粘性事件，我们需要调用方法 `postSticky()` 方法，代码如下所示：

```java
  public void postSticky(Object event) {
        synchronized (stickyEvents) {
            stickyEvents.put(event.getClass(), event);
        }
        post(event);
    }
```

从代码中，我们不难看出，粘性的事件发送与简单事件的发送唯一的区别就是将发送的事件添加到 `stickyEvents` 集合中去了。那为什么要这么做呢？在了解具体的原因之前，我们需要了解粘性事件的概念。

粘性事件的概念：当订阅者还没有订阅相关事件 `A` 时，程序已经发送了一些事件 `A`，按照正常的逻辑，当订阅者开始订阅事件 `A` 时，是接受不到程序已经发送过的事件 `A` ,但是我们希望接受到那些已经发送过的消息。这种已经过时，但又被重新接受的事件，我们称之为粘性事件。

那么根据粘性事件的思想，我们需要将已经发送的事件存储下来，并在粘性事件的订阅的过程中进行特别的处理，也就是在 `EventBus.register()` 方法中进行处理。还记得之前注册过程中的 `subscribe()` 方法吗？该方法内部对粘性事件进行了特殊的处理，代码如下所示：

```java
 private void subscribe(Object subscriber, SubscriberMethod subscriberMethod) {
        //省略部分代码
        //判断是否是粘性事件
        if (subscriberMethod.sticky) {
            //👇支持事件传递的粘性事件
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
                //👇开始执行订阅方法。
                Object stickyEvent = stickyEvents.get(eventType);
                checkPostStickyEventToSubscription(newSubscription, stickyEvent);
            }
        }
    }
```

在上述逻辑中，会从 `stickyEvents` 中获取之前发送的事件，然后调用 `checkPostStickyEventToSubscription()`。该方法代码如下所示：

```java
 private void checkPostStickyEventToSubscription(Subscription newSubscription, Object stickyEvent) {
        if (stickyEvent != null) {
            postToSubscription(newSubscription, stickyEvent, isMainThread());
        }
    }
```

又因为`checkPostStickyEventToSubscription()` 方法内部会调用 `postToSubscription()` 方法。那么最终订阅者就能接受到之前发送的事件，并执行相应的订阅方法啦。

### 最后

EventBus 主要的流程到现在已经讲完了。从实际的代码中，我们不仅能看到其良好的代码规范以及封装思想。还能看到该框架对性能的优化，尤其是添加了一些必要的缓存。我相信以上的这些点，都是值得我们借鉴与参考的。在接下来的文章中我们会讲解 EventBus 中的 `“加速引擎"`  索引类。有兴趣的小伙伴可以继续关注。
