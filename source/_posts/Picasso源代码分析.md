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

上面我们介绍了`Picasso`的大部分常用的使用方法.此外`Picasso`内部还具有监控功能.可以检测内存数据,缓存命中率等等.而且还会根据网络变化优化并发线程.
下面我们就来进行`Picasso`的源码分析。这里说明一下,`Picasso`的源码分析目前在网络上已经有几篇比较好的分析文章,我在分析的时候也借鉴了不少,分别是
[RowandJJ的Picasso源码学习](http://blog.csdn.net/chdjj/article/details/49964901)和[闭门造车的Picasso源码分析系列](http://blog.happyhls.me/category/android/picasso/).在这里注明,下文章有部分图是引用自这两个地方的.引用就不做具体标注了.

### 3.类关系图
![picasso](image/Picasso-classes-relation.png)
从类图上我们可以看出`Picasso`的核心类主要包括:`Picasso`,`RequestCreator`,`Action`,`Dispatcher`,`Request`,`RequestHandler`,`BitmapHunter`等等.一张图片加载可以分为以下几步:

	创建->入队->执行->解码->变换->批处理->完成->分发->显示(可选)
	
下面就让我们来通过`Picasso`的调用流程来具体分析它的具体实现.

### 4.源码分析

### 5.设计模式

### 6.个人评价





















