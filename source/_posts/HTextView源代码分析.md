---
title: HTextView源代码分析
date: 2016-01-30 17:49:27
tags:
- HTextView
- 策略模式
- 源代码分析
- 模板方法
---
[HTextView](https://github.com/hanks-zyh/HTextView)是一个用来给TextView里的文字做各种转换动画的开源库,第一次看到这个库的时候就被这些动画吸引了,不仅提供了多种动画选择,而且还有重复字符的位移动画,的确别出心裁,虽然实现起来并不是多么复杂,但是从1700+的star数上还是可以看出它的受欢迎程度,所以今天我们就来分析看看它到底是如何实现的.有哪些值得我们借鉴的地方,又有哪些不完善的地方。

### 使用方法 ###
HTextView的使用方法还是比较简单的,只需要调用`hTextView.setAnimateType();`来设定一种动画的类型,再调用`hTextView.animateText();`将字符串传入就可以执行切换动画了,此外还提供了`hTextView.reset();`方法来重置动画,具体代码如下:

    hTextView.setAnimateType(HTextViewType.SCALE);
	hTextView.animateText(sentences[mCounter]);

<!-- more -->
### 类关系图 ###
![](http://ww2.sinaimg.cn/mw690/7909c3e7gw1f0gr0gba5vj20ra0fv74z.jpg)

当我们去分析一个项目的时候,首先看这个类库的UML类图往往是最直观的,能很清晰的将各个类的关系用图的形式展示的很清楚,这里我是使用的Android Studio的插件[simpleUMLCE](https://plugins.jetbrains.com/plugin/4946?pr=)来自动生成的类图,非常方便推荐给大家,另外如果看不懂UML图可以参照[深入浅出UML类图系列](http://blog.csdn.net/lovelion/article/details/7838679),已经讲得很详细我就不再补充了。

从类图上看我们可以很清晰的看出,首先是定义了一个`IHText`的接口,然后`HText`实现了`IHText`的接口,然后左边的那么多类,可以从名字上大致猜出是各种动画的具体实现他们都是继承了HText类,值得一提的是`PixelateText`和`BurnText`是直接实现`IHText`的,我猜测应该是作者后期重构代码的时候忘记这两个类了,实际上这两个类也是可以继承自`HText`来实现.最后可以看出`IHText`是和`HTextView`相互耦合的.好的,类图就讲到这里,下面我们来看具体实现.

### 源码分析 ###
在开始分析一个开源项目的时候,我们往往先从其定义的接口来看,所以我们先看`IHText`接口是如何定义的:

    public interface IHText {
    	void init(HTextView hTextView, AttributeSet attrs, int defStyle);
    	void animateText(CharSequence text);
    	void onDraw(Canvas canvas);
    	void reset(CharSequence text);
	}
首先`init()`方法,顾名思义应该是进行一些初始化的操作,`annimateText`应该就是让文字开始做动画的方法,`onDraw`这个大家应该都很熟悉了,因为做动画,实际上就是一帧一帧的绘制然后来组成动画,所以`onDraw`方法也是必须的.最后一个`reset`应该就是重置文字以及一些状态等等.

看完了接口定义,不用着急,接下来我们去看`HTextView`,精简后的代码如下:

    public class HTextView extends TextView {
	    private IHText mIHText = new ScaleText();
	    private AttributeSet attrs;
	    private int defStyle;
	
	    public HTextView(Context context, AttributeSet attrs, int defStyle) {
	        super(context, attrs, defStyle);
	        init(attrs, defStyle);
	    }
	
	    private void init(AttributeSet attrs, int defStyle) {
	        this.attrs = attrs;
	        this.defStyle = defStyle;
	        TypedArray typedArray = getContext().obtainStyledAttributes(attrs, R.styleable.HTextView);
	        int animateType = typedArray.getInt(R.styleable.HTextView_animateType, 0);
	        switch (animateType) {
	            case 0:
	                mIHText = new ScaleText();
	                break;
	            case 1:
	                mIHText = new EvaporateText();
	                break;
	            case 2:
	                mIHText = new FallText();
	                break;
	        }
	        typedArray.recycle();
	        initHText(attrs, defStyle);
	    }
	
	    private void initHText(AttributeSet attrs, int defStyle) {
	        mIHText.init(this, attrs, defStyle);
	    }
	
	    public void animateText(CharSequence text) {
	        mIHText.animateText(text);
	    }
	
	    @Override
	    protected void onDraw(Canvas canvas) {
	        mIHText.onDraw(canvas);
	    }
	
	    public void reset(CharSequence text) {
	        mIHText.reset(text);
	    }
	
	    public void setAnimateType(HTextViewType type) {
	        switch (type) {
	            case SCALE:
	                mIHText = new ScaleText();
	                break;
	            case EVAPORATE:
	                mIHText = new EvaporateText();
	                break;
	            case FALL:
	                mIHText = new FallText();
	                break;
	        }
	        initHText(attrs, defStyle);
	    }
	}

从代码中可以看出`HTextView`继承自TextView,并在初始化的时候根据`animateType`来实例化对应的`IHText`,然后再对外暴露的`animateText()`,`onDraw()`,`reset()`这几个方法里都是直接调用`IHText`来进行处理.值得一提的是`HTextView`重写了`onDraw`方法,这样也就意味着一些`TextView`的特性就没法使用了,比如添加drawable,换行等等..

看到这里我们知道了原来就是通过type来实例化对应的动画执行类,然后再做具体的处理.其实这里就是设计模式中的**策略模式**,我们先引出来,文章后面我们再介绍策略模式.这样我们只需要去找一个实例去具体分析它的实现就能明白整个库的原理了,这里我们就拿`ScaleText`类来分析具体动画的实现方式.去到`ScaleText`里发现它继承自`HText`类的,所以我们先来看看`HText`类的代码:

    public abstract class HText implements IHText {
	    protected Paint mPaint, mOldPaint;
	    protected float[] gaps = new float[100];
	    protected float[] oldGaps = new float[100];
	    protected float mTextSize;
	    protected CharSequence mText;
	    protected CharSequence mOldText;
	    protected List<CharacterDiffResult> differentList = new ArrayList<>();
	    protected float oldStartX = 0; // 原来的字符串开始画的x位置
	    protected float startX = 0; // 新的字符串开始画的x位置
	    protected float startY = 0; // 字符串开始画的y, baseline
	    protected HTextView mHTextView;
	
	    @Override
	    public void init(HTextView hTextView, AttributeSet attrs, int defStyle){
	
	        mHTextView = hTextView;
	        mPaint = new Paint(Paint.ANTI_ALIAS_FLAG);
	        mPaint.setColor(mHTextView.getCurrentTextColor());
	        mPaint.setStyle(Paint.Style.FILL);
	
	        mOldPaint = new Paint(Paint.ANTI_ALIAS_FLAG);
	        mOldPaint.setColor(mHTextView.getCurrentTextColor());
	        mOldPaint.setStyle(Paint.Style.FILL);
	
	        mText = mHTextView.getText();
	        mOldText = mHTextView.getText();
	
	        mTextSize = mHTextView.getTextSize();
	
	        initVariables();
	        mHTextView.postDelayed(new Runnable() {
	            @Override
	            public void run() {
	                prepareAnimate();
	            }
	        }, 50);
	
	    }
	
	    @Override
	    public void animateText(CharSequence text) {
	        mHTextView.setText(text);
	        mOldText = mText;
	        mText = text;
	        prepareAnimate();
	        animatePrepare(text);
	        animateStart(text);
	    }
	
	    @Override
	    public void onDraw(Canvas canvas) {
	        drawFrame(canvas);
	    }
	
	    private void prepareAnimate() {
	        mTextSize = mHTextView.getTextSize();
	
	        mPaint.setTextSize(mTextSize);
	        for (int i = 0; i < mText.length(); i++) {
	            gaps[i] = mPaint.measureText(mText.charAt(i) + "");
	        }
	
	        mOldPaint.setTextSize(mTextSize);
	        for (int i = 0; i < mOldText.length(); i++) {
	            oldGaps[i] = mOldPaint.measureText(mOldText.charAt(i) + "");
	        }
	
	        oldStartX = (mHTextView.getMeasuredWidth() - mHTextView.getCompoundPaddingLeft() - mHTextView.getPaddingLeft() - mOldPaint
	                .measureText(mOldText.toString())) / 2f;
	        startX = (mHTextView.getMeasuredWidth() - mHTextView.getCompoundPaddingLeft() - mHTextView.getPaddingLeft() - mPaint
	                .measureText(mText.toString())) / 2f;
	        startY = mHTextView.getBaseline();
	
	        differentList.clear();
	        differentList.addAll(CharacterUtils.diff(mOldText, mText));
	    }
	
	    public void reset(CharSequence text) {
	        animatePrepare(text);
	        mHTextView.invalidate();
	    }
	
	    /**
	     * 类被实例化时初始化
	     */
	    protected abstract void initVariables();
	
	    /**
	     * 具体实现动画
	     *
	     * @param text
	     */
	    protected abstract void animateStart(CharSequence text);
	
	    /**
	     * 每次动画前初始化调用
	     *
	     * @param text
	     */
	    protected abstract void animatePrepare(CharSequence text);
	
	    /**
	     * 动画每次刷新界面时调用
	     *
	     * @param canvas
	     */
	    protected abstract void drawFrame(Canvas canvas);
	}

首先`HText`是一个抽象类,并且在初始化的时候,分别初始化了需要画旧的文字和新的文字的画笔,以及对新旧文字的赋值,最后调用了`prepareAnimate();`方法.在这个方法里,首先设置了两种`Paint`的`TextSize`,然后计算了每一个文字的宽度并保存在了`gaps`和`oldGaps`两个数组里,最后计算了`mOldText`和`mText`里相同字符的位置信息.这里主要是为了做到当两组`Text`中有相同字符时就不执行默认动画,而进行字符的平移动画,使动画更灵动.

看完初始化方法以及`prepareAnimate();`之后,我们留意到类的最后的四个抽象方法.这里作者注释的比较清楚了,就不再过多解释,这里我们就能明白不论是`ScaleText`类还是其他动画类型的类,实际上都是实现了这些方法,然后`HText`中对这些方法进行了调用,从而会执行子类中的相应实现然后来实现具体的某个动画,实际上这里就是设计模式中的**模板方法**.这里同样先不多说,文章最后会做总结.

然后我们再具体看这四个抽象方法分别是在哪里被调用的,我们就能理解实现`HText`的子类实现的这四个抽象方法会在什么时候调用.从上面的代码可以看出来`initVariables();`方法是在`init();`方法里调用用来初始化,`animatePrepare(CharSequence text);`和`animateStart(CharSequence text);`是在`prepareAnimate();`方法里调用用来准备动画和开始动画,最后`drawFrame(Canvas canvas);`会在`onDraw(Canvas canvas);`方法里不断调用。所以从文章开始的使用方法中我们知道是调用`hTextView.animateText(text);`就可以执行动画了,所以最终都是调用继承自`HText`的子类的`animatePrepare(CharSequence text);`和`animateStart(CharSequence text);`方法。然后一定是在这里开始动画,不断的触发`onDraw()`方法来完成动画。

好的,下面我们就来看看`ScaleText`的具体实现,由于篇幅原因,我们只具体分析一个`ScaleText`的实现,其余效果的实现只是绘制方法的不同,可以试着自己去阅读研究。`ScaleText`具体代码如下:

    public class ScaleText extends HText {
	    float mostCount = 20;
	    float charTime = 400;
	    private long duration;
	    private float progress;
	
	    @Override
	    protected void initVariables() {
	
	    }
	
	    @Override
	    protected void animateStart(CharSequence text) {
	        int n = mText.length();
	        n = n <= 0 ? 1 : n;
	        // 计算动画总时间
	        duration = (long) (charTime + charTime / mostCount * (n - 1));
	
	        ValueAnimator valueAnimator = ValueAnimator.ofFloat(0, duration).setDuration(duration);
	        valueAnimator.setInterpolator(new AccelerateDecelerateInterpolator());
	        valueAnimator.addUpdateListener(new ValueAnimator.AnimatorUpdateListener() {
	            @Override
	            public void onAnimationUpdate(ValueAnimator animation) {
	                progress = (float) animation.getAnimatedValue();
	                mHTextView.invalidate();
	            }
	        });
	        valueAnimator.start();
	    }
	
	    @Override
	    protected void animatePrepare(CharSequence text) {
	
	    }
	
	    @Override
	    public void drawFrame(Canvas canvas) {
	        float offset = startX;
	        float oldOffset = oldStartX;
	
	        int maxLength = Math.max(mText.length(), mOldText.length());
	
	        for (int i = 0; i < maxLength; i++) {
	
	            // draw old text
	            if (i < mOldText.length()) {
	
	                float percent = progress / duration;
	                int move = CharacterUtils.needMove(i, differentList);
	                if (move != -1) {
	                    mOldPaint.setTextSize(mTextSize);
	                    mOldPaint.setAlpha(255);
	
	                    float p = percent * 2f;
	                    p = p > 1 ? 1 : p;
	                    float distX = CharacterUtils.getOffset(i, move, p, startX, oldStartX, gaps, oldGaps);
	                    canvas.drawText(mOldText.charAt(i) + "", 0, 1, distX, startY, mOldPaint);
	                } else {
	                    mOldPaint.setAlpha((int) ((1 - percent) * 255));
	                    mOldPaint.setTextSize(mTextSize * (1 - percent));
	                    float width = mOldPaint.measureText(mOldText.charAt(i) + "");
	                    canvas.drawText(mOldText.charAt(i) + "", 0, 1, oldOffset + (oldGaps[i] - width) / 2, startY, mOldPaint);
	                }
	                oldOffset += oldGaps[i];
	            }
	
	            // draw new text
	            if (i < mText.length()) {
	
	                if (!CharacterUtils.stayHere(i, differentList)) {
	
	                    int alpha = (int) (255f / charTime * (progress - charTime * i / mostCount));
	                    if (alpha > 255) alpha = 255;
	                    if (alpha < 0) alpha = 0;
	
	                    float size = mTextSize * 1f / charTime * (progress - charTime * i / mostCount);
	                    if (size > mTextSize) size = mTextSize;
	                    if (size < 0) size = 0;
	
	                    mPaint.setAlpha(alpha);
	                    mPaint.setTextSize(size);
	
	                    float width = mPaint.measureText(mText.charAt(i) + "");
	                    canvas.drawText(mText.charAt(i) + "", 0, 1, offset + (gaps[i] - width) / 2, startY, mPaint);
	                }
	                offset += gaps[i];
	            }
	        }
	    }
	}

先来看`ScaleText`中定义的几个变量`mostCount`是表示最多同时执行动画的字符个数,为了实现顺序的动画执行,`charTime`表示`mostCount`个字符的动画时间,根据字符个数的不同动画时间不同,`duration`表示动画总时间,`progress`显然是表示进度.所以在`animateStart(CharSequence text)；`方法中,是根据字符个数的不同来计算总时间,代码如下:

    int n = mText.length();
	n = n <= 0 ? 1 : n;
	// 计算动画总时间
	duration = (long) (charTime + charTime / mostCount * (n - 1));

然后再通过`ValueAnimator`设置好`progress`的区间以及动画的`duration`,最后在`onAnimationUpdate(ValueAnimator animation)`的回调接口里,不断的拿到当前的`progress`然后调用`mHTextView.invalidate();`来不断更新,我们都知道最终会不断的调用`onDraw();`方法所以流转到最后还是调用`ScaleText`的`drawFrame(Canvas canvas);`方法.所以最终的动画都是在这里实现的。

从`drawFrame(Canvas canvas);`来看,首先是拿到了新旧字符串各自X方向的偏移量,因为看效果我们可以发现`ScaleText`切换的过程中总共有三种动画:

    1.oldText中不重复字符的缩小动画.
	2.oldText中与newText中重复字符的位移动画.
	3.newText中不重复字符的放大动画.

这三种动画都是在`drawFrame(Canvas canvas);`方法里处理的,首先是循环绘制每一个字符,然后先绘制`oldText`并在`oldText`首先判断这个字符是不是需要平移动画通过`CharacterUtils.needMove(i, differentList);`来判断,当不返回`-1`时表示需要进行平移动画,当返回`-1`时就进行缩小和透明动画,然后紧接着绘制`newText`,通过`!CharacterUtils.stayHere(i, differentList);`方法来跳过重复字符的绘制,然后再通过`progress`的值来计算出当前绘制的字符的大小和透明度。所以通过不断增加的`progress`和`onDraw();`方法的调用再配合这一系列算法,最终实现了我们要的动画。讲到这里整个库我们应该整体的理解了。

我想有人会说,为什么这么一个简单的动画写这么类,我用一个类就能写出来,又是定义接口,又是抽象类又是继承烦不烦?可是我们不要忘了,他还有另外9种动画实现,将来还可能有几十种拓展出来的动画.如果都写在一个类里实现,那就毫无拓展性可言了,所以这里我们要聊聊设计模式的好处了。

### 设计模式 ###

此项目中用到两种常用的设计模式分别是**策略模式**和**模板方法**设计模式.

#### 策略模式 ####
[策略模式(点击关于策略模式的详解)](https://github.com/simple-android-framework-exchange/android_design_patterns_analysis/tree/master/strategy/gkerison) 定义了一系列的算法，并将每一个算法封装起来，而且使它们还可以相互替换。策略模式让算法独立于使用它的客户而独立变化。所以在这个库中,每一种动画都相当于一种独立算法,又可以相互替换,所以每一个实现了`HText`的子类相当于组成了策略模式,这样做使得类库的结构清晰明了,拓展方便,耦合度低。缺点就是策略越多实现的子类就会增加。不过相对于策略模式的好处这点也不算什么了。

#### 模板方法 ####
[模板方法(点击关于模板方法的详解)](https://github.com/simple-android-framework-exchange/android_design_patterns_analysis/tree/master/template-method/mr.simple)定义一个操作中的算法的框架，而将一些步骤延迟到子类中。使得子类可以不改变一个算法的结构即可重定义该算法的某些特定步骤。模板方法在此库中的体现是`HText`这个抽象类定义的那四个抽象方法,分别在`HText`中进行调用,将这些步骤延迟到子类中执行,所以子类可以实现各种各样的动画效果,这是很典型的模板方法设计模式。

### 个人评价 ###
至此,我们就算是彻底了解了HTextView,虽然并没有多么复杂,但是它使用的这些典型的设计模式以及各种动画的实现确实可以从中让我们学到不少知识。尤其是各种动画的具体实现,能为我们自己在做相关动画时提供不少思路!
