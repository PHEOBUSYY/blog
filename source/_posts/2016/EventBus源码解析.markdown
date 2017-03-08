layout: "post"
title: "EventBus源码解析"
date: "2016-12-09 14:05"
---

## EventBus3.0源码解析

  使用方法是在工程的任何一个地方都可以通过EventBus对象来post一个Event对象给EventBus，然后在需要处理Event对象的类中register该类来响应处理，那么程序的入口就从EventBus的register方法入手了。
```java
public void register(Object subscriber) {
        Class<?> subscriberClass = subscriber.getClass();
        List<SubscriberMethod> subscriberMethods = subscriberMethodFinder.findSubscriberMethods(subscriberClass);
        synchronized (this) {
            for (SubscriberMethod subscriberMethod : subscriberMethods) {
                subscribe(subscriber, subscriberMethod);
            }
        }
    }
```

可以看到register方法中的核心方法在 *subscriberMethodFinder.findSubscriberMethods* 来返回所有注册的方法对象
那么下一步就进入 *subscriberMethodFinder* 来查看这个 *findSubscriberMethods* 方法

```java
List<SubscriberMethod> findSubscriberMethods(Class<?> subscriberClass) {
       List<SubscriberMethod> subscriberMethods = METHOD_CACHE.get(subscriberClass);
       if (subscriberMethods != null) {
           return subscriberMethods;
       }

       if (ignoreGeneratedIndex) {
           subscriberMethods = findUsingReflection(subscriberClass);
       } else {
           subscriberMethods = findUsingInfo(subscriberClass);
       }
       if (subscriberMethods.isEmpty()) {
           throw new EventBusException("Subscriber " + subscriberClass
                   + " and its super classes have no public methods with the @Subscribe annotation");
       } else {
           METHOD_CACHE.put(subscriberClass, subscriberMethods);
           return subscriberMethods;
       }
   }

```

可以看到 *findSubscriberMethods* 方法中,关键点就在 *findUsingReflection* 和 *findUsingInfo* 这两个方法中.由于 *findUsingReflection* 的代码比较少,所以先从这个方法开始.

```java
private List<SubscriberMethod> findUsingReflection(Class<?> subscriberClass) {
       FindState findState = prepareFindState();
       findState.initForSubscriber(subscriberClass);
       while (findState.clazz != null) {
           findUsingReflectionInSingleClass(findState);
           findState.moveToSuperclass();
       }
       return getMethodsAndRelease(findState);
   }
```
这里用来存放匹配结果的中间对象叫做 *FindState* ,稍后具体来说 *FindState* 的内部实现,现在先看一个生成 *FindState* 对象的方法 *prepareFindState*

```java
private FindState prepareFindState() {
        synchronized (FIND_STATE_POOL) {
            for (int i = 0; i < POOL_SIZE; i++) {
                FindState state = FIND_STATE_POOL[i];
                if (state != null) {
                    FIND_STATE_POOL[i] = null;
                    return state;
                }
            }
        }
        return new FindState();
    }
```

```java
private List<SubscriberMethod> getMethodsAndRelease(FindState findState) {
        List<SubscriberMethod> subscriberMethods = new ArrayList<>(findState.subscriberMethods);
        findState.recycle();
        synchronized (FIND_STATE_POOL) {
            for (int i = 0; i < POOL_SIZE; i++) {
                if (FIND_STATE_POOL[i] == null) {
                    FIND_STATE_POOL[i] = findState;
                    break;
                }
            }
        }
        return subscriberMethods;
    }
```

这里 *FIND_STATE_POOL* 是一个对象池,最大容量4个,可以想象,如果每次都生成一个 *findState* 的方法的话,会生成很多很多的中间对象,对性能造成影响,所以通过一个对象池来解决大量中间计算对象的问题,首先如果在对象池中找到有对象就直接使用,同时把对象池中的对象置空,最后回收的时候把用完了的对象放入对象池中,达到了循环引用的目的.感觉这个设计非常巧妙.

下一步我们来到 *findUsingReflection* 方法中的 *findUsingReflectionInSingleClass* 方法中

```java
private void findUsingReflectionInSingleClass(FindState findState) {
        Method[] methods;
        try {
            // This is faster than getMethods, especially when subscribers are fat classes like Activities
            methods = findState.clazz.getDeclaredMethods();
        } catch (Throwable th) {
            // Workaround for java.lang.NoClassDefFoundError, see https://github.com/greenrobot/EventBus/issues/149
            methods = findState.clazz.getMethods();
            findState.skipSuperClasses = true;
        }
        for (Method method : methods) {
            int modifiers = method.getModifiers();
            if ((modifiers & Modifier.PUBLIC) != 0 && (modifiers & MODIFIERS_IGNORE) == 0) {
                Class<?>[] parameterTypes = method.getParameterTypes();
                if (parameterTypes.length == 1) {
                    Subscribe subscribeAnnotation = method.getAnnotation(Subscribe.class);
                    if (subscribeAnnotation != null) {
                        Class<?> eventType = parameterTypes[0];
                        if (findState.checkAdd(method, eventType)) {
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

核心代码在for循环中的第一个if中,其他的是严格模式下返回异常信息.
首先取到注册类中的所有的声明方法,然后遍历找到其中是public的并且不是静态,抽象,中间编译器生成的方法,下面是这需要忽略类型的并集

```java
 private static final int MODIFIERS_IGNORE = Modifier.ABSTRACT | Modifier.STATIC | BRIDGE | SYNTHETIC;
```

看代码,明确要求注册方法必须只能有一个参数,同时注解必须是Subscribe对象类型,然后获取第一个param的类型,交由 *findState.checkAdd(method,eventType)* 方法去判断是否需要添加到事件绑定中.

```java
boolean checkAdd(Method method, Class<?> eventType) {
            // 2 level check: 1st level with event type only (fast), 2nd level with complete signature when required.
            // Usually a subscriber doesn't have methods listening to the same event type.
            Object existing = anyMethodByEventType.put(eventType, method);
            if (existing == null) {
                return true;
            } else {
                if (existing instanceof Method) {
                    if (!checkAddWithMethodSignature((Method) existing, eventType)) {
                        // Paranoia check
                        throw new IllegalStateException();
                    }
                    // Put any non-Method object to "consume" the existing Method
                    anyMethodByEventType.put(eventType, this);
                }
                return checkAddWithMethodSignature(method, eventType);
            }
        }
```

通过该方法的注释可以看到,有两层快速判断:第一层是通过eventType来判断,这种比较快,第二层是通过完成的签名来判断;通常情况下一个事件接受类不应该有多个方法对同一种类型的事件进行监听;
首先把eventType作为key,method对象作为value放入anyMethodByEventType这个map中,如果返回null说明之前是没有的放过的同样类型的eventType的,如果返回不为Null,说明anyMethodByEventType中已经有了同样的eventType事件监听,就需要更进一步的判断,交给下面的 *checkAddWithMethodSignature* 方法来处理.


```java
private boolean checkAddWithMethodSignature(Method method, Class<?> eventType) {
           methodKeyBuilder.setLength(0);
           methodKeyBuilder.append(method.getName());
           methodKeyBuilder.append('>').append(eventType.getName());

           String methodKey = methodKeyBuilder.toString();
           Class<?> methodClass = method.getDeclaringClass();
           Class<?> methodClassOld = subscriberClassByMethodKey.put(methodKey, methodClass);
           if (methodClassOld == null || methodClassOld.isAssignableFrom(methodClass)) {
               // Only add if not already found in a sub class
               return true;
           } else {
               // Revert the put, old class is further down the class hierarchy
               subscriberClassByMethodKey.put(methodKey, methodClassOld);
               return false;
           }
       }
```

首先是把方法名+接收参数类型名作为key存入map中,如果之前没有放入过,那么直接把返回true,交给上面的checkAdd返回true,说明事件绑定成功
如果已经放入过了,说明有两个同样名称的方法并且监听的居然是同一种eventType,获取方法的声明类class对象,和之前放入的声明class对象做isAssignableFrom判断,如果isAssignableFrom为true,说明已经放入的class是新放入的class本身类或者父类,返回true,可以添加,否则返回false,不可以添加,交给 *checkState* 方法throw一个异常.为什么会出现这种情况呢?个人思考了下应该是首先子类已经注册了一个subscribe方法对某一eventType做监听,同事父类里面有个同名方法对同样的eventType做监听,那么就会出现到底该绑定哪个方法的问题,这个时候优先绑定子类中同名方法,也就是离继承结构更远的方法,越靠近实现优先放入.

当当前类所有的绑定都完成之后, 通过 *findState.moveToSuperclass();* 让findState对象的class属性指向当前类的父类继续完成上面的操作,直到没有父类为止.

```java

void moveToSuperclass() {
           if (skipSuperClasses) {
               clazz = null;
           } else {
               clazz = clazz.getSuperclass();
               String clazzName = clazz.getName();
               /** Skip system classes, this just degrades performance. */
               if (clazzName.startsWith("java.") || clazzName.startsWith("javax.") || clazzName.startsWith("android.")) {
                   clazz = null;
               }
           }
       }
```

这里忽略掉了java和android的公共类,节省性能,系统的类是没有必要继续遍历的.
当当前类包括其自定义父类全部都遍历完成后,调用 *getMethodsAndRelease(findState)* 生成绑定方法的集合

```java
private List<SubscriberMethod> getMethodsAndRelease(FindState findState) {
        List<SubscriberMethod> subscriberMethods = new ArrayList<>(findState.subscriberMethods);
        findState.recycle();
        synchronized (FIND_STATE_POOL) {
            for (int i = 0; i < POOL_SIZE; i++) {
                if (FIND_STATE_POOL[i] == null) {
                    FIND_STATE_POOL[i] = findState;
                    break;
                }
            }
        }
        return subscriberMethods;
    }
```

上面的流程是 *findUsingReflection* 方法的主要实现流程,还记得有两个绑定事件的方法吗?还有一个是 *findUsingInfo* 通过一个变量开关来开启 *ignoreGeneratedIndex* ,刚开始么有看懂是啥意思,上网查了下资料发现这个功能是3.0新加的,个人理解有点像 *预编译* 在编译的时候通过 apt中新增一个AnnotationProcessor来把代码中声明了subscribe的方法提前生成一个index集合,这样可以快速的完成事件的绑定,加快运行速度.

java的annotation如果声明为source的target的时候,可以通过继承一个AbstractProcessor来处理自定义annotation.
具体的实现方式这里先不展开了,感兴趣的话可以上网去看下官方说明[subscriber-index][1818f832]

  [1818f832]: http://greenrobot.org/eventbus/documentation/subscriber-index/ "subscriber-index"

完成了事件绑定之后,就进入EventBus的核心方法 *subscribe* 了,通过它来完成一个观察者模式,在post的时候找到事件绑定的方法然后调用它,这样就完成了一个事件从绑定到响应的主流程.

```java
private void subscribe(Object subscriber, SubscriberMethod subscriberMethod) {
        Class<?> eventType = subscriberMethod.eventType;
        Subscription newSubscription = new Subscription(subscriber, subscriberMethod);
        CopyOnWriteArrayList<Subscription> subscriptions = subscriptionsByEventType.get(eventType);
        if (subscriptions == null) {
            subscriptions = new CopyOnWriteArrayList<>();
            subscriptionsByEventType.put(eventType, subscriptions);
        } else {
            if (subscriptions.contains(newSubscription)) {
                throw new EventBusException("Subscriber " + subscriber.getClass() + " already registered to event "
                        + eventType);
            }
        }

        int size = subscriptions.size();
        for (int i = 0; i <= size; i++) {
            if (i == size || subscriberMethod.priority > subscriptions.get(i).subscriberMethod.priority) {
                subscriptions.add(i, newSubscription);
                break;
            }
        }

        List<Class<?>> subscribedEvents = typesBySubscriber.get(subscriber);
        if (subscribedEvents == null) {
            subscribedEvents = new ArrayList<>();
            typesBySubscriber.put(subscriber, subscribedEvents);
        }
        subscribedEvents.add(eventType);

        if (subscriberMethod.sticky) {
            if (eventInheritance) {
                // Existing sticky events of all subclasses of eventType have to be considered.
                // Note: Iterating over all events may be inefficient with lots of sticky events,
                // thus data structure should be changed to allow a more efficient lookup
                // (e.g. an additional map storing sub classes of super classes: Class -> List<Class>).
                Set<Map.Entry<Class<?>, Object>> entries = stickyEvents.entrySet();
                for (Map.Entry<Class<?>, Object> entry : entries) {
                    Class<?> candidateEventType = entry.getKey();
                    if (eventType.isAssignableFrom(candidateEventType)) {
                        Object stickyEvent = entry.getValue();
                        checkPostStickyEventToSubscription(newSubscription, stickyEvent);
                    }
                }
            } else {
                Object stickyEvent = stickyEvents.get(eventType);
                checkPostStickyEventToSubscription(newSubscription, stickyEvent);
            }
        }
    }
```  

### subscribe方法
  当通过register方法获得了绑定事件的集合之后,就需要把注册类和绑定事件结合在一起,这样在post方法调用之后,可以快速的找到哪些类的哪些方法可以响应这次post事件,然后通过method对象的反射方法invoke(Object obj,Object... args)来调用绑定事件的方法.其中第二个参数就是post方法传入的eventType对象,第一个参数是register传入的绑定对象.一般是class对象.

  ```java
  // Must be called in synchronized block
   private void subscribe(Object subscriber, SubscriberMethod subscriberMethod) {
       Class<?> eventType = subscriberMethod.eventType;
       Subscription newSubscription = new Subscription(subscriber, subscriberMethod);
       CopyOnWriteArrayList<Subscription> subscriptions = subscriptionsByEventType.get(eventType);
       if (subscriptions == null) {
           subscriptions = new CopyOnWriteArrayList<>();
           subscriptionsByEventType.put(eventType, subscriptions);
       } else {
           if (subscriptions.contains(newSubscription)) {
               throw new EventBusException("Subscriber " + subscriber.getClass() + " already registered to event "
                       + eventType);
           }
       }

       int size = subscriptions.size();
       for (int i = 0; i <= size; i++) {
           if (i == size || subscriberMethod.priority > subscriptions.get(i).subscriberMethod.priority) {
               subscriptions.add(i, newSubscription);
               break;
           }
       }

       List<Class<?>> subscribedEvents = typesBySubscriber.get(subscriber);
       if (subscribedEvents == null) {
           subscribedEvents = new ArrayList<>();
           typesBySubscriber.put(subscriber, subscribedEvents);
       }
       subscribedEvents.add(eventType);

       if (subscriberMethod.sticky) {
           if (eventInheritance) {
               // Existing sticky events of all subclasses of eventType have to be considered.
               // Note: Iterating over all events may be inefficient with lots of sticky events,
               // thus data structure should be changed to allow a more efficient lookup
               // (e.g. an additional map storing sub classes of super classes: Class -> List<Class>).
               Set<Map.Entry<Class<?>, Object>> entries = stickyEvents.entrySet();
               for (Map.Entry<Class<?>, Object> entry : entries) {
                   Class<?> candidateEventType = entry.getKey();
                   if (eventType.isAssignableFrom(candidateEventType)) {
                       Object stickyEvent = entry.getValue();
                       checkPostStickyEventToSubscription(newSubscription, stickyEvent);
                   }
               }
           } else {
               Object stickyEvent = stickyEvents.get(eventType);
               checkPostStickyEventToSubscription(newSubscription, stickyEvent);
           }
       }

  ```  

  抛开该方法最底下的sticky判断那块,代码其实很简单,主要是为两个map完成赋值操作.
  两个map分别是:

  ```java
  private final Map<Class<?>, CopyOnWriteArrayList<Subscription>> subscriptionsByEventType;

  ```

  key是eventType,value是所有的绑定该事件的方法对象,可以理解就是触发post的时候,可以通过这个map快速的找到都有哪些方法绑定了该事件,然后通过遍历调用他们

  ```java
  private final Map<Object, List<Class<?>>> typesBySubscriber;
  ```
  这个map的key是register传入的对象,value是这个对象内部所有的绑定方法,这个map主要是用来unRegister的时候可以把该对象中所有的绑定方法全部移除,节省时间,提升效率

  当完成两个map的赋值工作之后,时间绑定对应关系就确定了,那么剩下逻辑就是看post方法的相关处理了.

  ### post方法

  ```java
  /** Posts the given event to the event bus. */
  public void post(Object event) {
      PostingThreadState postingState = currentPostingThreadState.get();
      List<Object> eventQueue = postingState.eventQueue;
      eventQueue.add(event);

      if (!postingState.isPosting) {
          postingState.isMainThread = Looper.getMainLooper() == Looper.myLooper();
          postingState.isPosting = true;
          if (postingState.canceled) {
              throw new EventBusException("Internal error. Abort state was not reset");
          }
          try {
              while (!eventQueue.isEmpty()) {
                  postSingleEvent(eventQueue.remove(0), postingState);
              }
          } finally {
              postingState.isPosting = false;
              postingState.isMainThread = false;
          }
      }
  }
  ```
这里的 *currentPostingThreadState*用到了ThreadLocal这种概念,ThreadLocal是指每个线程都有自己的独有变量,线程之间不互相干扰,是线程同步的另一种解决方法.ThreadLocal在第一次获取数据的时候如果没有的话就会调用它自己的initialValue方法,现在是返回一个 *PostingThreadState* 对象

```java
/** For ThreadLocal, much faster to set (and get multiple values). */
   final static class PostingThreadState {
       final List<Object> eventQueue = new ArrayList<Object>();
       boolean isPosting;
       boolean isMainThread;
       Subscription subscription;
       Object event;
       boolean canceled;
   }
```
其中的 *eventQueue* 相当于一个事件处理队列,isPosting用来做单条发送的间隔开关,就是每次消化一个事件,isMainThread用来表示是否在主线程发布,对应EventBus的ThreadMode4种模式的处理,后续会讲到.下面进入 *postSingleEvent* 方法

```java

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
                Log.d(TAG, "No subscribers registered for event " + eventClass);
            }
            if (sendNoSubscriberEvent && eventClass != NoSubscriberEvent.class &&
                    eventClass != SubscriberExceptionEvent.class) {
                post(new NoSubscriberEvent(this, event));
            }
        }

```
下面来解释一下 *eventInheritance* 这个开关,通过字面理解是 "事件继承" ,通过查阅文档说明,解释如下
> 默认情况下,EventBus认为绑定事件Event Class是具有继承性的,也就是说绑定了Event Class的父类方法在post Event class的时候也会触发.
> 关闭这项功能,可以加快事件传递的速度.如果关闭的话,大概事件post速度大概会提高20%,如果是复杂的Event继承关系的话,速度会提升大于20%

默认是开启 *事件继承* 功能的,所以我们进入 *lookupAllEventTypes* 方法

```java
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
```
```java
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
通过代码很容易理解,获取当前post的Event class的所有父类和接口类,然后放入 *eventTypes* 集合中,分别发送一次,这样当你post一个事件的时候,相当于该事件对象的所有父类和接口类都post了一次.
直观一点举例,比如一个叫CommonEvent的类,父类是BaseEvent,接口是IEvent,那么当你post CommonEvent的时候,所有接受BaseEvent 和IEvent的绑定方法也会触发.

这里为了加快速度,把类的关系链存入了一个集合作为缓存使用.

```java
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

```
下面马上进入post之后的回调处理,首先通过Event找到所有绑定该事件的方法集合,这里返回的是一个 *CopyOnWriteArrayList<Subscription>* ,然后遍历之,逐条调用绑定方法.进入 *postToSubscription* ,根据ThreadMode来做不同的回调处理.

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
*postToSubscription* 这个方法就是回调的核心逻辑所在,通过threadMode的不同来对应不同的处理关系.
先翻译一下关于ThreadMode的文档说明:
1. ThreadMode: POSTING
  接收者会在与post方法的同一个线程中同步执行,这种方式的开销最小,主要是避免了线程切换,这种模式推荐使用在可以快速完成的简单任务并且不需要阻塞主线程.
  ```java
    // Called in the same thread (default)
    // ThreadMode is optional here
    @Subscribe(threadMode = ThreadMode.POSTING)
    public void onMessage(MessageEvent event) {
        log(event.message);
    }
  ```
2. ThreadMode: MAIN
  接收者会在android的主线程中被调用,如果post的事件是在主线程中,那么这时调用方法会直接被调用和上面的Posting一样,这种模式推荐处理很快并且不会阻塞主线程的逻辑
  ```java
  // Called in Android UI's main thread
  @Subscribe(threadMode = ThreadMode.MAIN)
  public void onMessage(MessageEvent event) {
  textField.setText(event.message);
  }
  ```
3. ThreadMode: BACKGROUND
  接收者会在一个后台线程中别调用,如果post事件不是在主线程中,那么调用事件就会直接在post的线程中执行,如果是在主线程中post的话,EventBus会启用一个后台线程来处理所有这类event事件,事件回调应该迅速并且不会阻塞后台线程
  ```java
  // Called in the background thread
  @Subscribe(threadMode = ThreadMode.BACKGROUND)
  public void onMessage(MessageEvent event){
      saveToDisk(event.message);
  }
  ```  
4. ThreadMode: ASYNC
  事件回调会在一个单独的子线程中完成,这种情况下事件回调的线程是独立于主线程和post所在的线程的,是一个单独的线程,事件的post是不会等待回调方法的执行完成与否的,这种模式应该用于事件回调需要执行一段事件,比如网络请求等,要避免触发类似大量的长时间运行的异步线程.EventBus使用一个线程池来管理和回收这种线程.
  ```java
  // Called in a separate thread
  @Subscribe(threadMode = ThreadMode.ASYNC)
  public void onMessage(MessageEvent event){
      backend.send(event.message);
  }
  ```  
[官方ThreadMode说明][88ce7be7]

  [88ce7be7]: http://greenrobot.org/eventbus/documentation/delivery-threads-threadmode/ "官方ThreadMode说明"

  看完了官方说明之后,我们再来看代码具体实现,这个时候就已经很清晰了

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
  switch对应4中不同的ThreadMode,默认的模式使用的是POSTING,就是post在哪个线程那么回调就在哪个线程中.
  ```java
  void invokeSubscriber(Subscription subscription, Object event) {
      try {
          subscription.subscriberMethod.method.invoke(subscription.subscriber, event);
      } catch (InvocationTargetException e) {
          handleSubscriberException(subscription, event, e.getCause());
      } catch (IllegalAccessException e) {
          throw new IllegalStateException("Unexpected exception", e);
      }
  }
  ```
  *invokeSubscriber* 方法就是通过反射来回调绑定方法,结果非常简单.

  如果是 *MAIN* 模式下,判断是否在主线程,如果在主线程直接调用,如果在子线程中,通过 *mainThreadPoster* 来回到主线程中.

  来看 *mainThreadPoster* 的内部实现,实际上就是一个在主线程中的Handler.

  ```java
   mainThreadPoster = new HandlerPoster(this, Looper.getMainLooper(), 10);
  ```
  ```java
  final class HandlerPoster extends Handler {

    private final PendingPostQueue queue;
    private final int maxMillisInsideHandleMessage;
    private final EventBus eventBus;
    private boolean handlerActive;

    HandlerPoster(EventBus eventBus, Looper looper, int maxMillisInsideHandleMessage) {
        super(looper);
        this.eventBus = eventBus;
        this.maxMillisInsideHandleMessage = maxMillisInsideHandleMessage;
        queue = new PendingPostQueue();
    }

    void enqueue(Subscription subscription, Object event) {
        PendingPost pendingPost = PendingPost.obtainPendingPost(subscription, event);
        synchronized (this) {
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
  handler内部的handleMessage是一个死循环,每次通过enqueue方法来吧回调事件放入队列中,第一次的情况下发现handlerActive为否,说明没有启动,通过放一个空的message给handleMessage方法来启动死循环,如果已经启动了,handleMessage内部在不断的poll队列中的事件,依次执行回调方法,直到里面没有事件为止.同时在最底下还有个时间差判断,如果执行了十秒还没有执行完成的话,断开循环抛出异常,算是一个复位操作(个人理解)

  再看ThreadMode的backGround模式,如果在主线程中,那么就交给backgroundPoster处理,如果在子线程中就直接处理了.来看backgroundPoster的逻辑

  ```java
  final class BackgroundPoster implements Runnable {

    private final PendingPostQueue queue;
    private final EventBus eventBus;

    private volatile boolean executorRunning;

    BackgroundPoster(EventBus eventBus) {
        this.eventBus = eventBus;
        queue = new PendingPostQueue();
    }

    public void enqueue(Subscription subscription, Object event) {
        PendingPost pendingPost = PendingPost.obtainPendingPost(subscription, event);
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
                Log.w("Event", Thread.currentThread().getName() + " was interruppted", e);
            }
        } finally {
            executorRunning = false;
        }
    }

  }
  ```
  *BackgroundPoster* 就是一个简单的线程,当触发之后把自己交给EventBus的线程词来调用,自己的run方法内部维护一个死循环,不断从队列里面poll事件,直到没有为止,和上面的mainThreadPoster的做法类似.

  最后是ThreadMode的ASYNC模式,直接交给 *asyncPoster* 来处理了,可以猜想应该也是一个线程只不过是每次触发都是一个新的线程,放入线程池中.

  ```java
  class AsyncPoster implements Runnable {

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
  代码非常简单,每次 *enqueue* 把自己交给线程池去调用.

  最后就是注销绑定事件的unregister方法,也很简单,通过map的对应关系,把所有相关的事件移除就可以了
  ```java
  /** Unregisters the given subscriber from all event classes. */
   public synchronized void unregister(Object subscriber) {
       List<Class<?>> subscribedTypes = typesBySubscriber.get(subscriber);
       if (subscribedTypes != null) {
           for (Class<?> eventType : subscribedTypes) {
               unsubscribeByEventType(subscriber, eventType);
           }
           typesBySubscriber.remove(subscriber);
       } else {
           Log.w(TAG, "Subscriber to unregister was not registered before: " + subscriber.getClass());
       }
   }

  ```
  还剩余sticky事件的处理,下面重点讲一下
  sticky事件讲的是当前事件在post的时候还没有相应的绑定事件,当register相应的event的时候就会触发,那么可以想象在register的时候肯定回去存放sticky时间短额地方去查找有没有对应的事件,然后post一下去回调响应.

  首先看postSticky方法:
  ```java
  /**
     * Posts the given event to the event bus and holds on to the event (because it is sticky). The most recent sticky
     * event of an event's type is kept in memory for future access by subscribers using {@link Subscribe#sticky()}.
     */
    public void postSticky(Object event) {
        synchronized (stickyEvents) {
            stickyEvents.put(event.getClass(), event);
        }
        // Should be posted after it is putted, in case the subscriber wants to remove immediately
        post(event);
    }

  ```
  先放入 *stickEvents* 容器,尝试post一下
  然后在register的subscribe方法中,遍历stickEvents容器,如果找到了对应的event就post一下.
  ```java
  // Must be called in synchronized block
   private void subscribe(Object subscriber, SubscriberMethod subscriberMethod) {

      ...

       if (subscriberMethod.sticky) {
           if (eventInheritance) {
               // Existing sticky events of all subclasses of eventType have to be considered.
               // Note: Iterating over all events may be inefficient with lots of sticky events,
               // thus data structure should be changed to allow a more efficient lookup
               // (e.g. an additional map storing sub classes of super classes: Class -> List<Class>).
               Set<Map.Entry<Class<?>, Object>> entries = stickyEvents.entrySet();
               for (Map.Entry<Class<?>, Object> entry : entries) {
                   Class<?> candidateEventType = entry.getKey();
                   if (eventType.isAssignableFrom(candidateEventType)) {
                       Object stickyEvent = entry.getValue();
                       checkPostStickyEventToSubscription(newSubscription, stickyEvent);
                   }
               }
           } else {
               Object stickyEvent = stickyEvents.get(eventType);
               checkPostStickyEventToSubscription(newSubscription, stickyEvent);
           }
       }
   }

  ```
  然后看checkPostStickyEventToSubscription方法
  ```java
  private void checkPostStickyEventToSubscription(Subscription newSubscription, Object stickyEvent) {
      if (stickyEvent != null) {
          // If the subscriber is trying to abort the event, it will fail (event is not tracked in posting state)
          // --> Strange corner case, which we don't take care of here.
          postToSubscription(newSubscription, stickyEvent, Looper.getMainLooper() == Looper.myLooper());
      }
  }
  ```
  如果找到了对应的sticky事件就post到回调方法,非常清晰.

  ### 内部集合PendingPostQueue分析
  
  这里 *PendingPostQueue* 是一个双端链表的简单实现,链表的好处在于插入和删除的效率要比数组更高,遍历是慢于数组的,这里运用场景就是不断的从里面取最后一个event事件,那么用链表是非常合适的.

  ##总结
  EventBus这个库,通过读源码学到了很多东西,其内部构成就是一个简单观察者模式的实现,但是需要明白java的反射原理,多线程处理,以及各种线程同步的知识,受益匪浅.
  内部代码量也不是很多,核心代码仅仅不到千行却实现了这么小巧方便的代码库,非常高端.
