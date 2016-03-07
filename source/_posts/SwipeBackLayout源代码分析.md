---
title: SwipeBackLayout源代码分析
date: 2016-03-04 17:49:27
tags:
- SwipeBackLayout
- 滑动返回
- 源代码分析
- ViewDragHelper
---

> 项目地址：[SwipeBackLayout](https://github.com/ikew0ng/SwipeBackLayout)，本文分析版本: [e4ddae6d](https://github.com/ikew0ng/SwipeBackLayout/tree/e4ddae6d2b8af9b606493cba36faef8beba94be2)

### 1.简介
[SwipeBackLayout](https://github.com/ikew0ng/SwipeBackLayout)是一个在`Android`平台上实现了`Activity`滑动返回的库.实现了左,右,上,下四种手势返回的功能,在`ios`里滑动返回是系统自带可以配置的功能,而在我们`Android`上并没有系统级别的提供,但是主流应用比如微信就带有滑动返回功能,而且滑动返回是一个非常容易培养用户使用习惯的操作,用惯了滑动返回再用没有滑动返回的应用简直不能好好用了。。我自己正是滑动返回这个手势的重度使用者,我非常喜欢用滑动返回,所以在我开发过的应用里我都尽量集成了滑动返回这个功能,当然我相信是有很多人有滑动返回的使用习惯的。`SwipeBackLayout`应该算是使用范围最广的滑动返回的库了,我一直也是这个库的使用者,今天我们就来分析一下这个库是如何实现的:
<!-- more -->
### 2.使用方法
#### 1.设置需要滑动返回Activity的Theme
首先需要在需要滑动返回的`Activity`的`Theme`里添加:

```xml
<item name="android:windowIsTranslucent">true</item>
```
#### 2.将Activity继承`SwipeBackActivity`
然后将我们的`Activity`继承`SwipeBackActivity`就集成完毕,我们的`Activity`就默认带有左划返回的功能了,当然我们也可以在`onCreate()`方法的`super.onCreate(savedInstanceState);`执行之后,调用下面的方法做一些自定义设置:

```java

//是否允许滑动返回
setSwipeBackEnable(false);
//滑动并关闭activity
scrollToFinishActivity();
//获得SwipeBackLayout对象
getSwipeBackLayout();

```
在获取到`SwipeBackLayout`对象之后,可以设定从哪个方向可以滑动`mSwipeBackLayout.setEdgeTrackingEnabled(edgeFlag);`可以使用`setScrimColor()`来设置滑动返回的背景色,` mSwipeBackLayout.setEdgeSize(200);`来设置滑动触发的范围等等。

### 3.类关系图
![img](image/swipeback_class_relation.jpg)

`SwipeBackLayout`的类关系图非常的清晰简单,`SwipeBackActivity`继承自`Activity`并且实现了`SwipeBackActivityBase`接口,`SwipeBackActivity`,`SwipeBackActivityHelper`和`SwipeBackLayout`相互引用,类图上来看还是比较简单的,下面我们来看具体实现:

### 4.源码分析

一句话概括`SwipeBackLayout`的实现原理就是:通过在`DecorView`和其包含的子`View`之间添加一个`ViewGroup`也就是`SwipeBackLayout`,通过在`SwipeBackLayout`里处理触摸事件与位移来实现滑动返回的效果。
#### 1.SwipeBackActivityBase的实现

我们以前说过阅读一个框架的时候,先从它定义的一些接口开始,如果是小项目其实也没有太多的规定,因为代码量本身就不多,所以也可以整体来看,不过我们还是先来看看`SwipeBackActivityBase`接口是怎么定义的:

```java
public interface SwipeBackActivityBase {
    
    //得到SwipeBackLayout对象
    public abstract SwipeBackLayout getSwipeBackLayout();

    //设置是否可以滑动返回
    public abstract void setSwipeBackEnable(boolean enable);

    //自动滑动返回并关闭Activity
    public abstract void scrollToFinishActivity();
}
```
#### 2.SwipeBackActivity,SwipeBackActivityHelper的实现 ####
以上就是`SwipeBackActivityBase`接口定义的内容,`SwipeBackActivity`类实现了这个接口,也就是我们要继承的这个类，具体实现如下:

```java
public class SwipeBackActivity extends AppCompatActivity implements SwipeBackActivityBase {
    private SwipeBackActivityHelper mHelper;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
		//初始化mHelper
        mHelper = new SwipeBackActivityHelper(this);
        mHelper.onActivityCreate();
    }

    @Override
    protected void onPostCreate(Bundle savedInstanceState) {
        super.onPostCreate(savedInstanceState);
        mHelper.onPostCreate();
    }

    @Override
    public SwipeBackLayout getSwipeBackLayout() {
        return mHelper.getSwipeBackLayout();
    }

    @Override
    public void setSwipeBackEnable(boolean enable) {
        getSwipeBackLayout().setEnableGesture(enable);
    }

    @Override
    public void scrollToFinishActivity() {
        Utils.convertActivityToTranslucent(this);
        getSwipeBackLayout().scrollToFinishActivity();
    }
}

```
可以看到在`onCreate()`里创建了一个`SwipeBackActivityHelper`对象,而在`getSwipeBackLayout()`方法里通过`mHelper`得到了对应的`SwipeBackLayout`对象,所以`mHelper`应该是负责创建`SwipeBackLayout`并将`SwipeBackLayout`添加到`Activity`里的。
在`mHelper`的`onActivityCreate()`方法里创建了`SwipeBackLayout`对象,并在`onPostCreate()`方法里将`SwipeBackLayout`添加到`Activity`里,逻辑很简单,这里就不贴代码了,最终是调用了`SwipeBackLayout`的`attachToActivity()`方法：

```java

    public void attachToActivity(Activity activity) {
        //获得activity对象
        mActivity = activity;
        TypedArray a = activity.getTheme().obtainStyledAttributes(new int[]{
                android.R.attr.windowBackground
        });
        //得到窗口背景
        int background = a.getResourceId(0, 0);
        a.recycle();
        //得到DecorView对象,并先将decorChild移除并添加到
        //SwipeBackLayout里,再添加进DecorView
        ViewGroup decor = (ViewGroup) activity.getWindow().getDecorView();
        ViewGroup decorChild = (ViewGroup) decor.getChildAt(0);
        decorChild.setBackgroundResource(background);
        decor.removeView(decorChild);
        addView(decorChild);
        setContentView(decorChild);
        decor.addView(this);
    }

```
这就是整个添加过程,至此我们的`Activity`里就包含了`SwipeBackLayout`了,下面我们就来看看`SwipeBackLayout`的具体实现。

#### 3.SwipeBackLayout的实现

现在就只剩下一个`SwipeBackLayout`类了,它是继承自`FrameLayout`的,我们从它的构造方法开始看:

```
    public SwipeBackLayout(Context context, AttributeSet attrs, int defStyle) {
        super(context, attrs);
        mDragHelper = ViewDragHelper.create(this, new ViewDragCallback());

        TypedArray a = context.obtainStyledAttributes(attrs, R.styleable.SwipeBackLayout, defStyle,
                R.style.SwipeBackLayout);

        int edgeSize = a.getDimensionPixelSize(R.styleable.SwipeBackLayout_edge_size, -1);
		...
        a.recycle();
        final float density = getResources().getDisplayMetrics().density;
        final float minVel = MIN_FLING_VELOCITY * density;
        mDragHelper.setMinVelocity(minVel);
        mDragHelper.setMaxVelocity(minVel * 2f);
    }
```
除了从`AttributeSet`获取一些预先设定的属性外,最重要的是初始化了`ViewDragHelper`对象,关于`ViewDragHelper`:

```
ViewDragHelper是谷歌在2013年的I/O大会上推出的一个用于View拖拽操作的帮助类，借助于该类谷歌同时推出了两个用于侧滑的布局SlidingPaneLayout和DrawerLayout，现在市场上的很多带有侧滑菜单的应用都是基于这两种布局。

ViewDragHelper极大的简化了View的拖拽操作，在它没有出现之前很多侧滑都是采用的第三方库，为了实现侧滑效果需要对touch事件进行自定义拦截和处理，代码量大，实现逻辑也需要很大的耐心才能看懂。如果每个开发人员都从这么原始的步奏开始做起，那对于安卓生态是相当不利的。所以说ViewDragHelper等的出现反映了安卓开发框架已经开始向成熟的方向迈进。
```
我们下一篇文章会具体讨论`ViewDragHelper`的使用以及原理,这篇文章就不过多讨论了,有需要的可以先看这篇文章[ViewDragHelper解析](http://www.sunnyang.com/358.html),所以在这里`ViewDragHelper`的触摸事件就直接交给`ViewDragHelper`和`SwipeBackLayout`的`ViewDragCallback`类来处理了,以下为`SwipeBackLayout`的处理触摸事件的方法:

```java
    @Override
    public boolean onInterceptTouchEvent(MotionEvent event) {
        if (!mEnable) {
            return false;
        }
        try {
            return mDragHelper.shouldInterceptTouchEvent(event);
        } catch (ArrayIndexOutOfBoundsException e) {
            // FIXME: handle exception
            // issues #9
            return false;
        }
    }

    @Override
    public boolean onTouchEvent(MotionEvent event) {
        if (!mEnable) {
            return false;
        }
        mDragHelper.processTouchEvent(event);
        return true;
    }
```
可以看到是完全委托`ViewDragHelper`来处理,我们可以看到`SwipeBackLayout`中的`ViewDragHelper`并不是直接引用的`support`包中的`ViewDragHelper`而是将代码拷贝出来,是因为需要添加一些`support`中`ViewDragHelper`并不存在的方法,例如`mDragHelper.setEdgeSize(size);`,`mDragHelper.setMaxVelocity(minVel * 2f);`等,这篇文章我们就讨论更多关于`ViewDragHelper`的内容了,我们下篇会详细解释(其实是我偷懒=。=,想分两篇写。。不过`ViewDragHelper`还是确实值得一写的)。

这里我们需要知道当每次拖动发生时都会回调`ViewDragCallback`的`onViewPositionChanged()`方法,在这个方法里会调用`invalidate();`方法,最终会调用`SwipeBackLayout`的`drawChild()`方法:

```java
    @Override
    protected boolean drawChild(Canvas canvas, View child, long drawingTime) {
        Log.d(TAG, "drawChild");
        final boolean drawContent = child == mContentView;

        boolean ret = super.drawChild(canvas, child, drawingTime);
        if (mScrimOpacity > 0 && drawContent
                && mDragHelper.getViewDragState() != ViewDragHelper.STATE_IDLE) {
            //绘制边缘垂直阴影
            drawShadow(canvas, child);
            //绘制背景遮罩
            drawScrim(canvas, child);
        }
        return ret;
    }
```
可以看到阴影和遮罩就是在这个方法里绘制的了。除去`ViewDragHelper`的一些逻辑,`SwipeBackLayout`的实现还是非常好理解的,下周我会继续写关于`ViewDragHelper`的文章,到时候我们继续分析。

### 5.存在的问题
#### 1.部分Android版本不兼容:
在实际使用的时候发现当一个`Activity`栈内的所有`Activity`的`Theme`里都添加了`<item name="android:windowIsTranslucent">true</item>`参数时,在某些`Android 4.4.x`的手机上或者`4.4.x的MIUI`上,滑动返回的时候不会看到前面一个`Activity`而会把桌面显示出来,这应该是这些系统的bug.所以为了避免这些问题,我们需要在最底层的`Activity`的`Theme`里设置`<item name="android:windowIsTranslucent">false</item>`来避免这个问题,通常都是我们的`MainActivity`而且一般这个`Activity`我们并不需要左划返回的功能,所以这个问题也算是找到了解决办法。
#### 2.性能问题:
由于被设置了`<item name="android:windowIsTranslucent">true</item>`的`Activity`无法进入`onStop()`生命周期,所以导致`Activity`的`Window`无法回收,所以在多个`Activity`叠加时会出现明显的卡顿现象,目前并没有特别好的解决办法。

### 6.个人评价
虽然有上面的这些问题,但是我了解的目前开源社区中好像也并没有更好的滑动返回库了,但是一定有更好的实现方式可以解决这些问题,例如微信。虽然并不知道微信滑动返回的具体技术实现方式,但是一定是避免或者解决了上面的一些问题的。总体来说,如果我们的项目里需要滑动返回,`SwipeBackLayout`还是值得推荐的.

