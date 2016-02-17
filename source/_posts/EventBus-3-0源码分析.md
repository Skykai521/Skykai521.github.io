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

```
   	//3.0版本的注册
    EventBus.getDefault().register(this);
        
    //2.x版本的注册
    EventBus.getDefault().register(this);
    EventBus.getDefault().register(this, 100);
    EventBus.getDefault().registerSticky(this, 100);
    EventBus.getDefault().registerSticky(this);
```

可以看到2.x版本中有四种注册方法,区分了普通注册和粘性事件注册,并且在注册时可以选择接收事件的优先级,这里我们就不对2.x版本做过多的研究了,如果想研究可以参照[此篇文章](http://kymjs.com/code/2015/12/12/01).由于3.0版本将粘性事件以及订阅事件的优先级换了一种更好的实现方式,所以3.0版本中的注册就变得简单,只有一个`register()`方法即可.

#### 2.2编写响应事件订阅方法
注册之后,我们需要编写响应事件的方法,代码如下:

```
    //3.0版本
    @Subscribe(threadMode = ThreadMode.BACKGROUND, sticky = true, priority = 100)
    public void test(String str) {
        Log.d(TAG, Thread.currentThread().getId() + " BACKGROUND");
    }

    //2.x版本
    public void onEvent(String str) {

    }
    public void onEventMainThread(String str) {

    }
    public void onEventBackgroundThread(String str) {

    }

```

在2.x版本中只有通过onEvent开头的方法会被注册,而且响应事件方法触发的线程通过`onEventMainThread`或`onEventBackgroundThread`这些方法名区分,而在3.0版本中.通过`@Subscribe`注解,来确定运行的线程`threadMode`,是否接受粘性事件`sticky`以及事件优先级`priority`,而且方法名不在需要`onEvent`开头,所以又简洁灵活了不少.

#### 2.3发送事件
我们可以通过`EventBus`的`post()`方法来发送事件,发送之后就会执行注册过这个事件的对应类的方法.或者通过`postSticky()`来发送一个粘性事件.在代码是2.x版本和3.0版本是一样的.

```
    EventBus.getDefault().post("str");
    EventBus.getDefault().postSticky("str");
```

#### 2.4解除注册
当我们不在需要接收事件的时候需要解除注册`unregister`,2.x和3.0的解除注册也是相同的.代码如下:

```
	EventBus.getDefault().unregister(this);
```

### 3.类关系图

### 4.源码分析

### 5.设计模式

### 6.个人评价

