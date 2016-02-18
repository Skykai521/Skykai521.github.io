---
title: EventBus 3.0 源代码分析
---
> 项目地址：[EventBus](https://github.com/greenrobot/EventBus)，本文分析版本: [513f466](https://github.com/greenrobot/EventBus/tree/513f466fee9eec849d4c6a900b7fc1bf6bdc8fba)

### 1.简介

想必每个入了门的Android开发者都多少对EventBus有过了解,EventBus是一个Android事件发布/订阅框架，通过解耦发布者和订阅者简化 Android 事件传递。EventBus使用简单,并将事件发布和订阅充分解耦,从而使代码更简洁。一直以来很受开发者的欢迎,截止到目前EventBus的安装量已经超过一亿次。足以看出EventBus有多么的优秀。

目前网上已经有不少优秀的EventBus的源码分析文章,我也一直在犹豫要不要再写一次,一方面是因为最近EventBus刚好更新了3.0版本,事件的订阅已经从方法名换成了注解的方式,而且整体还是有不少变化。另外一方面也是为了自己学习。毕竟写出来会有更深层次的理解。好了,下面让我们看看3.0版本EventBus的使用方法.

### 2.使用方法
#### 2.1注册订阅者
首先我们需要将我们希望订阅事件的类,通过EventBus类注册,注册代码如下:


	//3.0版本的注册
	EventBus.getDefault().register(this);
	   
    //2.x版本的注册
    EventBus.getDefault().register(this);
    EventBus.getDefault().register(this, 100);
    EventBus.getDefault().registerSticky(this, 100);
    EventBus.getDefault().registerSticky(this);

可以看到2.x版本中有四种注册方法,区分了普通注册和粘性事件注册,并且在注册时可以选择接收事件的优先级,这里我们就不对2.x版本做过多的研究了,如果想研究可以参照[此篇文章](http://kymjs.com/code/2015/12/12/01).由于3.0版本将粘性事件以及订阅事件的优先级换了一种更好的实现方式,所以3.0版本中的注册就变得简单,只有一个`register()`方法即可.

#### 2.2编写响应事件订阅方法
注册之后,我们需要编写响应事件的方法,代码如下:

	//3.0版本
    @Subscribe(threadMode = ThreadMode.BACKGROUND, sticky = true, priority = 100)
    public void test(String str) {
    
    }

    //2.x版本
    public void onEvent(String str) {

    }
    public void onEventMainThread(String str) {

    }
    public void onEventBackgroundThread(String str) {

    }

在2.x版本中只有通过onEvent开头的方法会被注册,而且响应事件方法触发的线程通过`onEventMainThread`或`onEventBackgroundThread`这些方法名区分,而在3.0版本中.通过`@Subscribe`注解,来确定运行的线程`threadMode`,是否接受粘性事件`sticky`以及事件优先级`priority`,而且方法名不在需要`onEvent`开头,所以又简洁灵活了不少.

#### 2.3发送事件
我们可以通过`EventBus`的`post()`方法来发送事件,发送之后就会执行注册过这个事件的对应类的方法.或者通过`postSticky()`来发送一个粘性事件.在代码是2.x版本和3.0版本是一样的.

	EventBus.getDefault().post("str");
    EventBus.getDefault().postSticky("str");

#### 2.4解除注册
当我们不在需要接收事件的时候需要解除注册`unregister`,2.x和3.0的解除注册也是相同的.代码如下:

	EventBus.getDefault().unregister(this);

### 3.类关系图


### 4.源码分析

这一节我们通过`EventBus`的使用流程来分析它的调用流程,通过我们熟悉的使用方法来深入到`EventBus`的实现内部并理解它的实现原理.
#### 4.1创建EventBus
一般情况下我们都是通过`EventBus.getDefault()`获取到`EventBus`对象,从而在进行`register()`或者`post()`等等,所以我们看看`getDefault()`方法的实现:

    public static EventBus getDefault() {
        if (defaultInstance == null) {
            synchronized (EventBus.class) {
                if (defaultInstance == null) {
                    defaultInstance = new EventBus();
                }
            }
        }
        return defaultInstance;
    }
这里就是设计模式里我们常用了**单例模式**了,为了保证`getDefault()`得到的都是同一个实例,然后调用了`EventBus`的构造方法:

	private static final EventBusBuilder DEFAULT_BUILDER = new EventBusBuilder();
	
    public EventBus() {
        this(DEFAULT_BUILDER);
    }

    EventBus(EventBusBuilder builder) {
        //key:订阅的事件,value:订阅这个事件的所有订阅者集合
        //private final Map<Class<?>, CopyOnWriteArrayList<Subscription>> subscriptionsByEventType;
        subscriptionsByEventType = new HashMap<>();
        //key:订阅者对象,value:这个订阅者订阅的事件集合
        //private final Map<Object, List<Class<?>>> typesBySubscriber;
        typesBySubscriber = new HashMap<>();
        //粘性事件 key:粘性事件的class对象, value:事件对象
        //private final Map<Class<?>, Object> stickyEvents;
        stickyEvents = new ConcurrentHashMap<>();
        //事件主线程处理
        mainThreadPoster = new HandlerPoster(this, Looper.getMainLooper(), 10);
        //事件 Background 处理
        backgroundPoster = new BackgroundPoster(this);
        //事件异步线程处理
        asyncPoster = new AsyncPoster(this);
        indexCount = builder.subscriberInfoIndexes != null ? builder.subscriberInfoIndexes.size() : 0;
        //订阅者响应函数信息存储和查找类
        subscriberMethodFinder = new SubscriberMethodFinder(builder.subscriberInfoIndexes,
                builder.strictMethodVerification, builder.ignoreGeneratedIndex);
        logSubscriberExceptions = builder.logSubscriberExceptions;
        logNoSubscriberMessages = builder.logNoSubscriberMessages;
        sendSubscriberExceptionEvent = builder.sendSubscriberExceptionEvent;
        sendNoSubscriberEvent = builder.sendNoSubscriberEvent;
        throwSubscriberException = builder.throwSubscriberException;
        //是否支持事件继承
        eventInheritance = builder.eventInheritance;
        executorService = builder.executorService;
    }
可以看出是通过初始化了一个`EventBusBuilder()`对象来分别初始化`EventBus`的一些配置,当我们在写一个需要自定义配置的框架的时候,这种实现方法非常普遍,将配置解耦出去,使我们的代码结构更清晰.注释里我标注了大部分比较重要的对象,这里没必要记住,看下面的文章时如果对某个对象不了解,可以再回来看看.

#### 4.2注册过程源码分析
##### 4.2.1 register()方法的实现
3.0的注册只提供一个`register()`方法了,所以我们先来看看`register()`方法做了什么:

    public void register(Object subscriber) {
        //首先获得订阅者的class对象
        Class<?> subscriberClass = subscriber.getClass();
        //通过subscriberMethodFinder来找到订阅者订阅了哪些事件.返回一个SubscriberMethod对象的List,SubscriberMethod
        //里包含了这个方法的Method对象,以及将来响应订阅是在哪个线程的ThreadMode,以及订阅的事件类型eventType,以及订阅的优
        //先级priority,以及是否接收粘性sticky事件的boolean值.
        List<SubscriberMethod> subscriberMethods = subscriberMethodFinder.findSubscriberMethods(subscriberClass);
        synchronized (this) {
            for (SubscriberMethod subscriberMethod : subscriberMethods) {
                //订阅
                subscribe(subscriber, subscriberMethod);
            }
        }
    }
可以看到`register()`方法很简洁,代码里的注释也很清楚了,我们可以看出通过`subscriberMethodFinder.findSubscriberMethods(subscriberClass)`方法就能返回一个`SubscriberMethod`的对象,而`SubscriberMethod`里包含了所有我们需要的接下来执行`subscribe()`的信息.所以我们先去看看`findSubscriberMethods()`是怎么实现的,然后我们再去关注`subscribe()`。
##### 4.2.2 SubscriberMethodFinder的实现

##### 4.2.3 subscribe()方法的实现

#### 4.3事件分发过程源码分析

#### 4.4解除注册源码分析

### 5.设计模式

### 6.个人评价







































