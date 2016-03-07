---
title: Picasso源代码分析
date: 2016-02-25 09:23:03
tags:
- Picasso
- 源代码分析
- 建造者模式 
- 责任链模式
---

### 1.简介
[Picasso](https://github.com/square/picasso)是Square公司开源的一个Android平台上的图片加载框架,也是大名鼎鼎的[JakeWharton](https://github.com/JakeWharton)的代表作品之一.对于图片加载和缓存框架,优秀的开源作品有不少。比如:[Android-Universal-Image-Loader](https://github.com/nostra13/Android-Universal-Image-Loader),[Glide](https://github.com/bumptech/glide),[fresco](https://github.com/facebook/fresco)等等.我自己有在项目中使用过的有`Picasso`,`U-I-L`,`Glide`.对于一般的应用上面这些图片加载和缓存框架都是能满足使用的,[Trinea](http://www.trinea.cn/)有一篇关于这些框架的对比文章[Android 三大图片缓存原理、特性对比](http://www.trinea.cn/android/android-image-cache-compare/).有兴趣了解的可以去看看,本文我们主要讲`Picasso`的使用方法和源码分析.
<!-- more -->
### 2.使用方法

```java
//加载一张图片
Picasso.with(this).load("url").placeholder(R.mipmap.ic_default).into(imageView);

//加载一张图片并设置一个回调接口
Picasso.with(this).load("url").placeholder(R.mipmap.ic_default).into(imageView, new Callback() {
    @Override
    public void onSuccess() {

    }

    @Override
    public void onError() {

    }
});

//预加载一张图片
Picasso.with(this).load("url").fetch();

//同步加载一张图片,注意只能在子线程中调用并且Bitmap不会被缓存到内存里.
new Thread() {
    @Override
    public void run() {
        try {
            final Bitmap bitmap = Picasso.with(getApplicationContext()).load("url").get();
            mHandler.post(new Runnable() {
                @Override
                public void run() {
                    imageView.setImageBitmap(bitmap);
                }
            });
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}.start();

//加载一张图片并自适应imageView的大小,如果imageView设置了wrap_content,会显示不出来.直到该ImageView的
//LayoutParams被设置而且调用了该View的ViewTreeObserver.OnPreDrawListener回调接口后才会显示.
Picasso.with(this).load("url").priority(Picasso.Priority.HIGH).fit().into(imageView);

//加载一张图片并按照指定尺寸以centerCrop()的形式缩放.
Picasso.with(this).load("url").resize(200,200).centerCrop().into(imageView);

//加载一张图片并按照指定尺寸以centerInside()的形式缩放.并设置加载的优先级为高.注意centerInside()或centerCrop()
//只能同时使用一种,而且必须指定resize()或者resizeDimen();
Picasso.with(this).load("url").resize(400,400).centerInside().priority(Picasso.Priority.HIGH).into(imageView);

//加载一张图片旋转并且添加一个Transformation,可以对图片进行各种变化处理,例如圆形头像.
Picasso.with(this).load("url").rotate(10).transform(new Transformation() {
    @Override
    public Bitmap transform(Bitmap source) {
        //处理Bitmap
        return null;
    }

    @Override
    public String key() {
        return null;
    }
}).into(imageView);

//加载一张图片并设置tag,可以通过tag来暂定或者继续加载,可以用于当ListView滚动是暂定加载.停止滚动恢复加载.
Picasso.with(this).load("url").tag(mContext).into(imageView);
Picasso.with(this).pauseTag(mContext);
Picasso.with(this).resumeTag(mContxt);
        
```

上面我们介绍了`Picasso`的大部分常用的使用方法.此外`Picasso`内部还具有监控功能.可以检测内存数据,缓存命中率等等.而且还会根据网络变化优化并发线程.下面我们就来进行`Picasso`的源码分析。这里说明一下,`Picasso`的源码分析目前在网络上已经有几篇比较好的分析文章,我在分析的时候也借鉴了不少,分别是[RowandJJ的Picasso源码学习](http://blog.csdn.net/chdjj/article/details/49964901)和[闭门造车的Picasso源码分析系列](http://blog.happyhls.me/category/android/picasso/).在这里注明,下文章有部分图是引用自这两个地方的.引用就不做具体标注了.

### 3.类关系图
![picasso](image/Picasso-classes-relation.png)
从类图上我们可以看出`Picasso`的核心类主要包括:`Picasso`,`RequestCreator`,`Action`,`Dispatcher`,`Request`,`RequestHandler`,`BitmapHunter`等等.一张图片加载可以分为以下几步:

	创建->入队->执行->解码->变换->批处理->完成->分发->显示(可选)
	
下面就让我们来通过`Picasso`的调用流程来具体分析它的具体实现.

### 4.源码分析

#### Picasso.with()方法的实现
按照我们的惯例,我们从`Picasso`的调用流程开始分析,我们就从加载一张图片开始看起:

```java
Picasso.with(this).load(url).into(imageView);
```

让我们先来看看`Picasso.with()`做了什么:
```java
	  public static Picasso with(Context context) {
	    if (singleton == null) {
	      synchronized (Picasso.class) {
	        if (singleton == null) {
	          singleton = new Builder(context).build();
	        }
	      }
	    }
	    return singleton;
	  }
```

维护一个`Picasso`的单例,如果还未实例化就通过`new Builder(context).build()`创建一个`singleton`并返回,我们继续看`Builder`类的实现:

```java
	  public static class Builder {
	
	    public Builder(Context context) {
	      if (context == null) {
	        throw new IllegalArgumentException("Context must not be null.");
	      }
	      this.context = context.getApplicationContext();
	    }
	
	    /** Create the {@link Picasso} instance. */
	    public Picasso build() {
	      Context context = this.context;
	
	      if (downloader == null) {
			//创建默认下载器
	        downloader = Utils.createDefaultDownloader(context);
	      }
	      if (cache == null) {
			//创建Lru内存缓存
	        cache = new LruCache(context);
	      }
	      if (service == null) {
			//创建线程池,默认有3个执行线程,会根据网络状况自动切换线程数
	        service = new PicassoExecutorService();
	      }
	      if (transformer == null) {
			//创建默认的transformer,并无实际作用
	        transformer = RequestTransformer.IDENTITY;
	      }
		  //创建stats用于统计缓存,以及缓存命中率,下载数量等等
	      Stats stats = new Stats(cache);
		  //创建dispatcher对象用于任务的调度
	      Dispatcher dispatcher = new Dispatcher(context, service, HANDLER, downloader, cache, stats);
	
	      return new Picasso(context, dispatcher, cache, listener, transformer, requestHandlers, stats,
	          defaultBitmapConfig, indicatorsEnabled, loggingEnabled);
	    }
	  }
```



上面的代码里省去了部分配置的方法,当我们使用`Picasso`默认配置的时候(当然也可以自定义),最后会调用`build()`方法并配置好我们需要的各种对象,最后实例化一个`Picasso`对象并返回。最后在`Picasso`的构造方法里除了对这些对象的赋值以及创建一些新的对象,例如清理线程等等.最重要的是初始化了`requestHandlers`,下面是代码片段:

```java
	List<RequestHandler> allRequestHandlers =
	    new ArrayList<RequestHandler>(builtInHandlers + extraCount);
	
	// ResourceRequestHandler needs to be the first in the list to avoid
	// forcing other RequestHandlers to perform null checks on request.uri
	// to cover the (request.resourceId != 0) case.
	allRequestHandlers.add(new ResourceRequestHandler(context));
	if (extraRequestHandlers != null) {
	  allRequestHandlers.addAll(extraRequestHandlers);
	}
	allRequestHandlers.add(new ContactsPhotoRequestHandler(context));
	allRequestHandlers.add(new MediaStoreRequestHandler(context));
	allRequestHandlers.add(new ContentStreamRequestHandler(context));
	allRequestHandlers.add(new AssetRequestHandler(context));
	allRequestHandlers.add(new FileRequestHandler(context));
	allRequestHandlers.add(new NetworkRequestHandler(dispatcher.downloader, stats));
	requestHandlers = Collections.unmodifiableList(allRequestHandlers);
```

可以看到除了添加我们可以自定义的`extraRequestHandlers`,另外添加了7个`RequestHandler`分别用来处理加载不同来源的资源,可能是`Resource`里的,也可能是`File`也可能是来源于网络的资源.这里使用了一个`ArrayList`来存放这些`RequestHandler`现在先不用了解这么做是为什么,下面我们会分析,到这我们就了解了`Picasso.with()`做了什么,接下来我们去看看`load()`方法.

#### load(),centerInside(),等方法的实现
在`Picasso`的`load()`方法里我们可以传入`String`,`Uri`或者`File`对象,但是其最终都是返回一个`RequestCreator`对象,如下所示:

```java
	public RequestCreator load(Uri uri) {
		return new RequestCreator(this, uri, 0);
	}
```
再来看看`RequestCreator`的构造方法:

```java
  RequestCreator(Picasso picasso, Uri uri, int resourceId) {
    if (picasso.shutdown) {
      throw new IllegalStateException(
          "Picasso instance already shut down. Cannot submit new requests.");
    }
    this.picasso = picasso;
    this.data = new Request.Builder(uri, resourceId, picasso.defaultBitmapConfig);
  }

```

首先是持有一个`Picasso`的对象,然后构建一个`Request`的`Builder`对象,将我们需要加载的图片的信息都保存在`data`里,在我们通过`.centerCrop()`或者`.transform()`等等方法的时候实际上也就是改变`data`内的对应的变量标识,再到处理的阶段根据这些参数来进行对应的操作,所以在我们调用`into()`方法之前,所有的操作都是在设定我们需要处理的参数,真正的操作都是有`into()`方法引起的。

#### into()方法的实现
从上文中我们知道在我们调用了`load()`方法之后会返回一个`RequestCreator`对象,所以`.into(imageView)`方法必然是在`RequestCreator`里:

```java

  public void into(ImageView target) {
    //传入空的callback
    into(target, null);
  }

  public void into(ImageView target, Callback callback) {
    long started = System.nanoTime();
    //检查调用是否在主线程
    checkMain();

    if (target == null) {
      throw new IllegalArgumentException("Target must not be null.");
    }
    //如果没有设置需要加载的uri,或者resourceId
    if (!data.hasImage()) {
      picasso.cancelRequest(target);
      //如果设置占位图片,直接加载并返回
      if (setPlaceholder) {
        setPlaceholder(target, getPlaceholderDrawable());
      }
      return;
    }
    //如果是延时加载,也就是选择了fit()模式
    if (deferred) {
      //fit()模式是适应target的宽高加载,所以并不能手动设置resize,如果设置就抛出异常
      if (data.hasSize()) {
        throw new IllegalStateException("Fit cannot be used with resize.");
      }
      int width = target.getWidth();
      int height = target.getHeight();
      //如果目标ImageView的宽或高现在为0
      if (width == 0 || height == 0) {
        //先设置占位符
        if (setPlaceholder) {
          setPlaceholder(target, getPlaceholderDrawable());
        }
        //监听ImageView的ViewTreeObserver.OnPreDrawListener接口,一旦ImageView
        //的宽高被赋值,就按照ImageView的宽高继续加载.
        picasso.defer(target, new DeferredRequestCreator(this, target, callback));
        return;
      }
      //如果ImageView有宽高就设置设置
      data.resize(width, height);
    }

    //构建Request
    Request request = createRequest(started);
    //构建requestKey
    String requestKey = createKey(request);

    //根据memoryPolicy来决定是否可以从内存里读取
    if (shouldReadFromMemoryCache(memoryPolicy)) {
      //通过LruCache来读取内存里的缓存图片
      Bitmap bitmap = picasso.quickMemoryCacheCheck(requestKey);
      //如果读取到
      if (bitmap != null) {
        //取消target的request
        picasso.cancelRequest(target);
        //设置图片
        setBitmap(target, picasso.context, bitmap, MEMORY, noFade, picasso.indicatorsEnabled);
        if (picasso.loggingEnabled) {
          log(OWNER_MAIN, VERB_COMPLETED, request.plainId(), "from " + MEMORY);
        }
        //如果设置了回调接口就回调接口的方法.
        if (callback != null) {
          callback.onSuccess();
        }
        return;
      }
    }
    //如果缓存里没读到,先根据是否设置了占位图并设置占位
    if (setPlaceholder) {
      setPlaceholder(target, getPlaceholderDrawable());
    }
    //构建一个Action对象,由于我们是往ImageView里加载图片,所以这里创建的是一个ImageViewAction对象.
    Action action =
        new ImageViewAction(picasso, target, request, memoryPolicy, networkPolicy, errorResId,
            errorDrawable, requestKey, tag, callback, noFade);

    //将Action对象入列提交
    picasso.enqueueAndSubmit(action);
  }

```

整个流程看下来应该是比较清晰的,最后是创建了一个`ImageViewAction`对象并通过`picasso`提交,这里简要说明一下`ImageViewAction`,实际上`Picasso`会根据我们调用的不同方式来实例化不同的`Action`对象,当我们需要往`ImageView`里加载图片的时候会创建`ImageViewAction`对象,如果是往实现了`Target`接口的对象里加载图片是则会创建`TargetAction`对象,这些`Action`类的实现类不仅保存了这次加载需要的所有信息,还提供了加载完成后的回调方法.也是由子类实现并用来完成不同的调用的。然后让我们继续去看`picasso.enqueueAndSubmit(action)`方法:

```java
  void enqueueAndSubmit(Action action) {
    Object target = action.getTarget();
    //取消这个target已经有的action.
    if (target != null && targetToAction.get(target) != action) {
      // This will also check we are on the main thread.
      cancelExistingRequest(target);
      targetToAction.put(target, action);
    }
    //提交action
    submit(action);
  }
  //调用dispatcher来派发action
  void submit(Action action) {
    dispatcher.dispatchSubmit(action);
  }

```

很简单,最后是转到了`dispatcher`类来处理,那我们就来看看`dispatcher.dispatchSubmit(action)`方法:

```java
  void dispatchSubmit(Action action) {
    handler.sendMessage(handler.obtainMessage(REQUEST_SUBMIT, action));
  }
```
看到通过一个`handler`对象发送了一个`REQUEST_SUBMIT`的消息,那么这个`handler`是存在与哪个线程的呢?

```java
  Dispatcher(Context context, ExecutorService service, Handler mainThreadHandler,
      Downloader downloader, Cache cache, Stats stats) {
    this.dispatcherThread = new DispatcherThread();
    this.dispatcherThread.start();    
    this.handler = new DispatcherHandler(dispatcherThread.getLooper(), this);
    this.mainThreadHandler = mainThreadHandler;
  }
  
    static class DispatcherThread extends HandlerThread {
    DispatcherThread() {
      super(Utils.THREAD_PREFIX + DISPATCHER_THREAD_NAME, THREAD_PRIORITY_BACKGROUND);
    }
  }
```

上面是`Dispatcher`的构造方法(省略了部分代码),可以看到先是创建了一个`HandlerThread`对象,然后创建了一个`DispatcherHandler`对象,这个`handler`就是刚刚用来发送`REQUEST_SUBMIT`消息的`handler`,这里我们就明白了原来是通过`Dispatcher`类里的一个子线程里的`handler`不断的派发我们的消息,这里是用来派发我们的`REQUEST_SUBMIT`消息,而且最终是调用了 `dispatcher.performSubmit(action);`方法:

```java
  void performSubmit(Action action) {
    performSubmit(action, true);
  }

  void performSubmit(Action action, boolean dismissFailed) {
    //是否该tag的请求被暂停
    if (pausedTags.contains(action.getTag())) {
      pausedActions.put(action.getTarget(), action);
      if (action.getPicasso().loggingEnabled) {
        log(OWNER_DISPATCHER, VERB_PAUSED, action.request.logId(),
            "because tag '" + action.getTag() + "' is paused");
      }
      return;
    }
    //通过action的key来在hunterMap查找是否有相同的hunter,这个key里保存的是我们
    //的uri或者resourceId和一些参数,如果都是一样就将这些action合并到一个
    //BitmapHunter里去.
    BitmapHunter hunter = hunterMap.get(action.getKey());
    if (hunter != null) {
      hunter.attach(action);
      return;
    }

    if (service.isShutdown()) {
      if (action.getPicasso().loggingEnabled) {
        log(OWNER_DISPATCHER, VERB_IGNORED, action.request.logId(), "because shut down");
      }
      return;
    }

    //创建BitmapHunter对象
    hunter = forRequest(action.getPicasso(), this, cache, stats, action);
    //通过service执行hunter并返回一个future对象
    hunter.future = service.submit(hunter);
    //将hunter添加到hunterMap中
    hunterMap.put(action.getKey(), hunter);
    if (dismissFailed) {
      failedActions.remove(action.getTarget());
    }

    if (action.getPicasso().loggingEnabled) {
      log(OWNER_DISPATCHER, VERB_ENQUEUED, action.request.logId());
    }
  }
```

注释很详细,这里我们再分析一下`forRequest()`是如何实现的:

```java
  static BitmapHunter forRequest(Picasso picasso, Dispatcher dispatcher, Cache cache, Stats stats,
      Action action) {
    Request request = action.getRequest();
    List<RequestHandler> requestHandlers = picasso.getRequestHandlers();

    //从requestHandlers中检测哪个RequestHandler可以处理这个request,如果找到就创建
    //BitmapHunter并返回.
    for (int i = 0, count = requestHandlers.size(); i < count; i++) {
      RequestHandler requestHandler = requestHandlers.get(i);
      if (requestHandler.canHandleRequest(request)) {
        return new BitmapHunter(picasso, dispatcher, cache, stats, action, requestHandler);
      }
    }

    return new BitmapHunter(picasso, dispatcher, cache, stats, action, ERRORING_HANDLER);
  }

```

这里就体现出来了**责任链模式**,通过依次调用`requestHandlers`里`RequestHandler`的`canHandleRequest()`方法来确定这个`request`能被哪个`RequestHandler`执行,找到对应的`RequestHandler`后就创建`BitmapHunter`对象并返回.再回到`performSubmit()`方法里,通过`service.submit(hunter);`执行了`hunter`,`hunter`实现了`Runnable`接口,所以`run()`方法就会被执行,所以我们继续看看`BitmapHunter`里`run()`方法的实现:

```java
  @Override public void run() {
    try {
      //更新当前线程的名字
      updateThreadName(data);

      if (picasso.loggingEnabled) {
        log(OWNER_HUNTER, VERB_EXECUTING, getLogIdsForHunter(this));
      }

      //调用hunt()方法并返回Bitmap类型的result对象.
      result = hunt();

      //如果为空,调用dispatcher发送失败的消息,
      //如果不为空则发送完成的消息
      if (result == null) {
        dispatcher.dispatchFailed(this);
      } else {
        dispatcher.dispatchComplete(this);
      }
    //通过不同的异常来进行对应的处理
    } catch (Downloader.ResponseException e) {
      if (!e.localCacheOnly || e.responseCode != 504) {
        exception = e;
      }
      dispatcher.dispatchFailed(this);
    } catch (NetworkRequestHandler.ContentLengthException e) {
      exception = e;
      dispatcher.dispatchRetry(this);
    } catch (IOException e) {
      exception = e;
      dispatcher.dispatchRetry(this);
    } catch (OutOfMemoryError e) {
      StringWriter writer = new StringWriter();
      stats.createSnapshot().dump(new PrintWriter(writer));
      exception = new RuntimeException(writer.toString(), e);
      dispatcher.dispatchFailed(this);
    } catch (Exception e) {
      exception = e;
      dispatcher.dispatchFailed(this);
    } finally {
      Thread.currentThread().setName(Utils.THREAD_IDLE_NAME);
    }
  }

  Bitmap hunt() throws IOException {
    Bitmap bitmap = null;
    //是否可以从内存中读取
    if (shouldReadFromMemoryCache(memoryPolicy)) {
      bitmap = cache.get(key);
      if (bitmap != null) {
        //统计缓存命中率
        stats.dispatchCacheHit();
        loadedFrom = MEMORY;
        if (picasso.loggingEnabled) {
          log(OWNER_HUNTER, VERB_DECODED, data.logId(), "from cache");
        }
        return bitmap;
      }
    }

    //如果未设置networkPolicy并且retryCount为0,则将networkPolicy设置为
    //NetworkPolicy.OFFLINE
    data.networkPolicy = retryCount == 0 ? NetworkPolicy.OFFLINE.index : networkPolicy;
    //通过对应的requestHandler来获取result
    RequestHandler.Result result = requestHandler.load(data, networkPolicy);
    if (result != null) {
      loadedFrom = result.getLoadedFrom();
      exifOrientation = result.getExifOrientation();
      bitmap = result.getBitmap();

      // If there was no Bitmap then we need to decode it from the stream.
      if (bitmap == null) {
        InputStream is = result.getStream();
        try {
          bitmap = decodeStream(is, data);
        } finally {
          Utils.closeQuietly(is);
        }
      }
    }

    if (bitmap != null) {
      if (picasso.loggingEnabled) {
        log(OWNER_HUNTER, VERB_DECODED, data.logId());
      }
      stats.dispatchBitmapDecoded(bitmap);
      //处理Transformation
      if (data.needsTransformation() || exifOrientation != 0) {
        synchronized (DECODE_LOCK) {
          if (data.needsMatrixTransform() || exifOrientation != 0) {
            bitmap = transformResult(data, bitmap, exifOrientation);
            if (picasso.loggingEnabled) {
              log(OWNER_HUNTER, VERB_TRANSFORMED, data.logId());
            }
          }
          if (data.hasCustomTransformations()) {
            bitmap = applyCustomTransformations(data.transformations, bitmap);
            if (picasso.loggingEnabled) {
              log(OWNER_HUNTER, VERB_TRANSFORMED, data.logId(), "from custom transformations");
            }
          }
        }
        if (bitmap != null) {
          stats.dispatchBitmapTransformed(bitmap);
        }
      }
    }
    //返回bitmap
    return bitmap;
  }
```

在`run()`方法里调用了`hunt()`方法来获取`result`然后通知了`dispatcher`来处理结果,并在`try-catch`里通知`dispatcher`来处理相应的异常,在`hunt()`方法里通过前面指定的`requestHandler`来获取相应的`result`,我们是从网络加载图片,自然是调用`NetworkRequestHandler`的`load()`方法来处理我们的`request`,这里我们就不再分析`NetworkRequestHandler`具体的细节.获取到`result`之后就获得我们的`bitmap`然后检测是否需要`Transformation`,这里使用了一个全局锁`DECODE_LOCK`来保证同一个时刻仅仅有一个图片正在处理。我们假设我们的请求被正确处理了,这样我们拿到我们的`result`然后调用了`dispatcher.dispatchComplete(this);`最终也是通过`handler`调用了`dispatcher.performComplete()`方法:

```java
  void performComplete(BitmapHunter hunter) {
    //是否可以放入内存缓存里
    if (shouldWriteToMemoryCache(hunter.getMemoryPolicy())) {
      cache.set(hunter.getKey(), hunter.getResult());
    }
    //从hunterMap移除
    hunterMap.remove(hunter.getKey());
    //处理hunter
    batch(hunter);
    if (hunter.getPicasso().loggingEnabled) {
      log(OWNER_DISPATCHER, VERB_BATCHED, getLogIdsForHunter(hunter), "for completion");
    }
  }
  
  private void batch(BitmapHunter hunter) {
    if (hunter.isCancelled()) {
      return;
    }
    batch.add(hunter);
    if (!handler.hasMessages(HUNTER_DELAY_NEXT_BATCH)) {
      handler.sendEmptyMessageDelayed(HUNTER_DELAY_NEXT_BATCH, BATCH_DELAY);
    }
  }
  
  void performBatchComplete() {
    List<BitmapHunter> copy = new ArrayList<BitmapHunter>(batch);
    batch.clear();
    mainThreadHandler.sendMessage(mainThreadHandler.obtainMessage(HUNTER_BATCH_COMPLETE, copy));
    logBatch(copy);
  }

```

首先是添加到内存缓存中去,然后在发送一个`HUNTER_DELAY_NEXT_BATCH`消息,实际上这个消息最后会出发`performBatchComplete()`方法,`performBatchComplete()`里则是通过`mainThreadHandler`将`BitmapHunter`的`List`发送到主线程处理,所以我们去看看`mainThreadHandler`的`handleMessage()`方法:

```java
  static final Handler HANDLER = new Handler(Looper.getMainLooper()) {
    @Override public void handleMessage(Message msg) {
      switch (msg.what) {
        case HUNTER_BATCH_COMPLETE: {
          @SuppressWarnings("unchecked") List<BitmapHunter> batch = (List<BitmapHunter>) msg.obj;
          //noinspection ForLoopReplaceableByForEach
          for (int i = 0, n = batch.size(); i < n; i++) {
            BitmapHunter hunter = batch.get(i);
            hunter.picasso.complete(hunter);
          }
          break;
        }
        default:
          throw new AssertionError("Unknown handler message received: " + msg.what);
      }
    }
  };
```
很简单,就是依次调用`picasso.complete(hunter)`方法:

```java
 void complete(BitmapHunter hunter) {
    //获取单个Action
    Action single = hunter.getAction();
    //获取被添加进来的Action
    List<Action> joined = hunter.getActions();

    //是否有合并的Action
    boolean hasMultiple = joined != null && !joined.isEmpty();
    //是否需要派发
    boolean shouldDeliver = single != null || hasMultiple;

    if (!shouldDeliver) {
      return;
    }

    Uri uri = hunter.getData().uri;
    Exception exception = hunter.getException();
    Bitmap result = hunter.getResult();
    LoadedFrom from = hunter.getLoadedFrom();

    //派发Action
    if (single != null) {
      deliverAction(result, from, single);
    }

    //派发合并的Action
    if (hasMultiple) {
      //noinspection ForLoopReplaceableByForEach
      for (int i = 0, n = joined.size(); i < n; i++) {
        Action join = joined.get(i);
        deliverAction(result, from, join);
      }
    }

    if (listener != null && exception != null) {
      listener.onImageLoadFailed(this, uri, exception);
    }
  }
  
 private void deliverAction(Bitmap result, LoadedFrom from, Action action) {
    if (action.isCancelled()) {
      return;
    }
    if (!action.willReplay()) {
      targetToAction.remove(action.getTarget());
    }
    if (result != null) {
      if (from == null) {
        throw new AssertionError("LoadedFrom cannot be null.");
      }
      //回调action的complete()方法
      action.complete(result, from);
      if (loggingEnabled) {
        log(OWNER_MAIN, VERB_COMPLETED, action.request.logId(), "from " + from);
      }
    } else {
      //失败则回调error()方法
      action.error();
      if (loggingEnabled) {
        log(OWNER_MAIN, VERB_ERRORED, action.request.logId());
      }
    }
  }

```

可以看出最终是回调了`action`的`complete()`方法,从前文知道我们这里是`ImageViewAction`,所以我们去看看`ImageViewAction`的`complete()`的实现:

```java
  @Override public void complete(Bitmap result, Picasso.LoadedFrom from) {
    if (result == null) {
      throw new AssertionError(
          String.format("Attempted to complete action with no result!\n%s", this));
    }

    //得到target也就是ImageView
    ImageView target = this.target.get();
    if (target == null) {
      return;
    }

    Context context = picasso.context;
    boolean indicatorsEnabled = picasso.indicatorsEnabled;
    //通过PicassoDrawable来将bitmap设置到ImageView上
    PicassoDrawable.setBitmap(target, context, result, from, noFade, indicatorsEnabled);

    //回调callback接口
    if (callback != null) {
      callback.onSuccess();
    }
  }
```

很显然通过了`PicassoDrawable.setBitmap()`将我们的`Bitmap`设置到了我们的`ImageView`上,最后并回调`callback`接口,这里为什么会使用`PicassoDrawabl`来设置`Bitmap`呢?使用过`Picasso`的都知道,`Picasso`自带渐变的加载动画,所以这里就是处理渐变动画的地方,由于篇幅原因我们就不做具体分析了,感兴趣的同学可以自行研究,所以到这里我们的整个`Picasso`的调用流程的源码分析就结束了.

### 5.设计模式

#### 建造者模式 
[建造者模式](https://github.com/simple-android-framework-exchange/android_design_patterns_analysis/tree/master/builder/mr.simple)是指:将一个复杂对象的构建与它的表示分离，使得同样的构建过程可以创建不同的表示。建造者模式应该是我们都比较熟悉的一种模式,在创建`AlertDialog`的时候通过配置不同的参数就可以展示不同的`AlertDialog`,这也正是建造者模式的用途,通过不同的参数配置或者不同的执行顺序来构建不同的对象,在`Picasso`里当构建`RequestCreator`的时候正是使用了这种设计模式,我们通过可选的配置比如`centerInside()`,`placeholder()`等等来分别达到不同的使用方式,在这种使用场景下使用建造者模式是非常合适的.

#### 责任链模式
[责任链模式](https://github.com/simple-android-framework-exchange/android_design_patterns_analysis/tree/master/chain-of-responsibility/AigeStudio)是指:一个请求沿着一条“链”传递，直到该“链”上的某个处理者处理它为止。当我们有一个请求可以被多个处理者处理或处理者未明确指定时。可以选择使用责任链模式,在`Picasso`里当我们通过`BitmapHunter`的`forRequest()`方法构建一个`BitmapHunter`对象时,需要为`Request`指定一个对应的`RequestHandler`来进行处理`Request`.这里就使用了责任链模式,依次调用`requestHandler.canHandleRequest(request)`方法来判断是否该`RequestHandler`能处理该`Request`如果可以就构建一个`RequestHandler`对象返回.这里就是典型的责任链思想的体现。

### 6.个人评价
`Picasso`是我最早接触Android时第一个使用的图片加载库,当时也了解过`U-I-L`,但是最终选择使用了`Picasso`因为对于当时的我来说`Picasso`使用起来足够简单,简单到我根本不需要任何配置一行代码就可以搞定图片加载,所以对于初学者`Picasso`可以算得上最易使用的图片加载缓存库了,而且分析完源代码发现`Picasso`实现的`feature`并不算少,对于一个标准的图片库功能也可谓是应有竟有,虽然比`Glide`和`Fresco`是少一些功能或者性能,但是作为出现时间比较早的图片加载库还是值得推荐学习和使用的。


