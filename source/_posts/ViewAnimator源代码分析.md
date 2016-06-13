---
title: ViewAnimator源代码分析
date: 2016-03-23 08:39:32
tags:
- ViewAnimator
- 属性动画
- 源代码分析
---

> 项目地址：[ViewAnimator](https://github.com/florent37/ViewAnimator)，本文分析版本: [dfa45e0](https://github.com/florent37/ViewAnimator/tree/dfa45e000aa5954581d31fc987b1f34ed62594df)

### 1.简介

![ViewAnimator.png](http://upload-images.jianshu.io/upload_images/1485091-4cbf66d069909c83.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
在项目开发中我们应该都接触过动画效果的开发.我们知道在`Andorid`中实现动画大致分为两类,一种是`Tween/Frame`动画,另一种是`Property Animation`也就是属性动画.关于这两种动画的使用方法我们这篇文章就不多做讨论了。[可以从这篇文章了解更多](http://a.codekk.com/detail/Android/lightSky/%E5%85%AC%E5%85%B1%E6%8A%80%E6%9C%AF%E7%82%B9%E4%B9%8B%20Android%20%E5%8A%A8%E7%94%BB%E5%9F%BA%E7%A1%80)。我们这篇文章只涉及属性动画相关知识。我们今天要介绍的`ViewAnimator`就是用来简化我们写属性动画的的代码量的,它可以通过非常简洁的代码通过建造者模式调用来组合各种动画.让我们的代码简洁易读。如果你的`APP`里需要各种动画组合,`ViewAnimator`一定是你的最佳选择。

<!-- more -->
### 2.使用方法
想必大家都使用过属性动画了。我们来做一个最简单的位移动画:

```java

        ObjectAnimator animator = ObjectAnimator.ofFloat(textView, "translationX",0, 500);
        animator.setDuration(2000);
        animator.setRepeatCount(1);
        animator.setInterpolator(new BounceInterpolator());
        animator.start();
```


上面的代码执行之后就可以使`textView`从当前位置水平移动`500px`,整个动画过程是`2s`,并且添加了一个弹性插值器,而且使动画再重复执行一遍。这样看起来整个代码还是很清晰的,使用起来也很方便,但是如果我们要多个`View`相互组合再加上各种动画,可想而知代码量会有多少了。下面我们就用属性动画来写一个下面这张图里的动画:

![ViewAnimatorGif.gif](http://upload-images.jianshu.io/upload_images/1485091-4cf622733ad39113.gif?imageMogr2/auto-orient/strip)

这张图里包含了:1.文字颜色的渐变以及背景的渐变,然后同时又`textView`放大动画,和两张图片的下落动画,第一组动画结束后,圆形的图片开始旋转,然后`textView`不断的显示进度.这就是所有动画,下面是我们实现的代码:

```java

        ObjectAnimator mountainTransY = ObjectAnimator.ofFloat(mountain, "translationY", - dip2px(500), 0);
        ObjectAnimator mountainAlpha = ObjectAnimator.ofFloat(mountain, "alpha", 0, 1);
        ObjectAnimator imageTransY = ObjectAnimator.ofFloat(image, "translationY", - dip2px(500), 0);
        ObjectAnimator imageAlpha = ObjectAnimator.ofFloat(image, "alpha", 0, 1);
        ObjectAnimator percentScaleX = ObjectAnimator.ofFloat(percent, "scaleX", 0, 1);
        ObjectAnimator percentScaleY = ObjectAnimator.ofFloat(percent, "scaleY", 0, 1);
        ObjectAnimator textColorAnimator = ObjectAnimator.ofInt(text, "textColor", Color.BLACK, Color.WHITE, Color.RED);
        textColorAnimator.setEvaluator(new ArgbEvaluator());
        ObjectAnimator textBackgroundAnimator = ObjectAnimator.ofInt(text, "backgroundColor", Color.WHITE, Color.BLACK, Color.YELLOW);
        textBackgroundAnimator.setEvaluator(new ArgbEvaluator());

        ObjectAnimator imageRotation = ObjectAnimator.ofFloat(image, "rotation", 0, 360);
        ValueAnimator valueAnimator = ValueAnimator.ofFloat(0, 1);
        valueAnimator.addUpdateListener(new ValueAnimator.AnimatorUpdateListener() {
            @Override
            public void onAnimationUpdate(ValueAnimator animation) {
                percent.setText(String.format(Locale.US, "%.02f%%", animation.getAnimatedValue()));
            }
        });

        AnimatorSet firstSet = new AnimatorSet();
        firstSet.playTogether(mountainTransY, mountainAlpha, imageTransY, imageAlpha, percentScaleX,
                percentScaleY, textColorAnimator, textBackgroundAnimator);
        firstSet.setInterpolator(new AccelerateDecelerateInterpolator());
        firstSet.setDuration(5000);

        final AnimatorSet secondSet = new AnimatorSet();
        secondSet.playTogether(imageRotation, valueAnimator);
        secondSet.setDuration(5000);

        firstSet.addListener(new Animator.AnimatorListener() {
            @Override
            public void onAnimationStart(Animator animation) {

            }

            @Override
            public void onAnimationEnd(Animator animation) {
                secondSet.start();
            }

            @Override
            public void onAnimationCancel(Animator animation) {

            }

            @Override
            public void onAnimationRepeat(Animator animation) {

            }
        });
        firstSet.start();
```

上面就是实现这个效果的所有代码.借用岳云鹏的一句话就是:"我的天哪"(请脑补配音)。这么一大坨代码。。我整整写了十几分钟。。而且从这么多代码来看上去,如果以后需要调整动画的话,无论如何也得先整个看一遍才能找到怎么调节。这样维护成本就增加了。那么如何解决这个问题呢?这就要用到我们今天要介绍的主角`ViewAnimator`,下面是用`ViewAnimator`来实现相同动画的代码:

```java

        ViewAnimator.animate(mountain, image)
                .dp().translationY(-500, 0)
                .alpha(0, 1)

                .andAnimate(percent)
                .scale(0, 1)

                .andAnimate(text)
                .textColor(Color.BLACK, Color.WHITE)
                .backgroundColor(Color.WHITE, Color.BLACK)
                .interpolator(new AccelerateDecelerateInterpolator())
                .duration(2000)

                .thenAnimate(percent)
                .custom(new AnimationListener.Update<TextView>() {
                    @Override
                    public void update(TextView view, float value) {
                        view.setText(String.format(Locale.US, "%.02f%%", value));
                    }
                }, 0, 1)
                .andAnimate(image)
                .rotation(0, 360)
                .duration(2000)
                .start();
```

真是又简洁又易读。简直"完美"(请再脑补配音)。可以看到从上到下,我们需要先通过`animate(View... view)`方法将我们要进行动画的`View`传入,然后通过建造者模式调用我们需要做的动画,方法名代表我们需要动画的属性,方法参数里直接传入数值即可。`andAnimate(View... view)`表示同时做该`view`的动画但是具体的动画可以不一样。然后通过`thenAnimate(View... view)`方法就可以表示前面的动画执行完毕后再执行的动画.具体每个方法代表的意思也很清楚就是我们需要操作的属性的意思。所以整体来看代码简洁易读又好维护。

此外`ViewAnimator`还封装了不少动画组合让我们拿来即用,例如:`standUp()`,`wave()`,`shake()`等等动画。此外还支持`Path`以及`SVG Path`动画.更多的使用方法可以参照`ViewAnimator`的[README.md](https://github.com/florent37/ViewAnimator/blob/master/README.md)。下面我们就具体来看看如此好用的`ViewAnimator`是如何实现的。

### 3.类关系图

![classes-relation.png](http://upload-images.jianshu.io/upload_images/1485091-df9a187246a53cc3.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 4.源码分析

### 5.个人评价

