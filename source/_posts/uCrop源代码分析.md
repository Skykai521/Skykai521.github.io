---
title: uCrop源代码分析
date: 2016-03-016 21:35:27
tags:
- uCrop
- 图片剪裁
- 建造者模式
- 源代码分析
---

> 项目地址：[uCrop](https://github.com/Yalantis/uCrop)，本文分析版本: [83b77c0](https://github.com/Yalantis/uCrop/tree/83b77c018d95889f4aabe6fd899b81cfef20f1be2)

### 1.简介

![uCrop.png](https://d13yacurqjgara.cloudfront.net/users/221935/screenshots/2474295/animation.gif)

在项目开发中,我们难免会有一些功能,比如上传用户头像,上传图片等等会使用到图片裁剪的功能,为了节省开发时间,我们一般不会去自己开发一个图片剪裁库,这时候我们会去`github`上寻找各种开源的图片裁剪库,找来找去会发现目前最好用的就要数`Yalantis`公司出的[uCrop](https://github.com/Yalantis/uCrop)了.这也是最近刚刚开源的一个库,`Yalantis`专门写了一篇文章来说明问什么会有`uCrop`这个项目,以及对比了目前主流的几个图片剪裁的项目,地址[在这](https://yalantis.com/blog/introducing-ucrop-our-own-image-cropping-library-for-android/).英文不错的同学可以去看看,话说英文对工程师来说还是很重要的,读英文资料以及逛英文社区可以很好的拓宽我们的技术视野,也能第一时间接触最前沿的技术。好了废话不多说,今天我们就来看看`uCrop`这个目前最好的图片剪裁库是如何使用以及实现的:
<!-- more -->
### 2.使用方法
作为最好用的图片剪裁库,`uCrop`的使用方法当然是相当简单的:
#### 1.AndroidManifest中注册UCropActivity

```java
    <activity
        android:name="com.yalantis.ucrop.UCropActivity"
        android:screenOrientation="portrait"
        android:theme="@style/Theme.AppCompat.Light.NoActionBar"/>
```
#### 2.配置uCrop参数
你可以通过建造者模式来创建一个`uCrop`对象,并且可以通过`UCrop.Options`来设置一些个性化的参数:

```java
    UCrop.of(sourceUri, destinationUri)
        .withAspectRatio(16, 9)
        .withMaxResultSize(maxWidth, maxHeight)
        .start(context);
```
其中`sourceUri`代表选择图片的`Uri`地址,`destinationUri`代表图片裁剪完毕后保存的`Uri`地址,`withAspectRatio(16, 9)`表示设定你需要裁剪的图片的宽高比,这里表示希望`16:9`,当然你也可以选择`useSourceImageAspectRatio()`来确保裁剪后的图片的宽高比和原图相同,如果选择了上面两项,则进入裁剪界面后是无法再改变裁剪的宽高比的。如果你需要动态的改变图片裁剪的比例,那么什么都不设置就好了。`withMaxResultSize(maxWidth, maxHeight)`设置裁剪后的图片的最大宽度和高度,`start(context)`方法即可开启裁剪.

`uCrop`很友善的提供给了我们更多可以自定义方法,我们可以通过`UCrop.Options`来设定:

```java

    UCrop.Options options = new UCrop.Options();
    //设置裁剪图片的保存格式
    options.setCompressionFormat(Bitmap.CompressFormat.PNG);
    //设置裁剪图片的图片质量
    options.setCompressionQuality(90);
    //设置你想要指定的可操作的手势
    options.setAllowedGestures(UCropActivity.SCALE, UCropActivity.ROTATE, UCropActivity.ALL);
    //设置uCropActivity里的UI样式
    options.setMaxScaleMultiplier(5);
    options.setImageToCropBoundsAnimDuration(666);
    options.setDimmedLayerColor(Color.CYAN);
    options.setOvalDimmedLayer(true);
    options.setShowCropFrame(false);
    options.setCropGridStrokeWidth(20);
    options.setCropGridColor(Color.GREEN);
    options.setCropGridColumnCount(2);
    options.setCropGridRowCount(1);
    //最后别忘记调用
    uCrop.withOptions(options);
```

#### 3.在onActivityResult中处理裁剪后的结果
    
```java
    @Override
    public void onActivityResult(int requestCode, int resultCode, Intent data) {
        if (resultCode == RESULT_OK && requestCode == UCrop.REQUEST_CROP) {
            final Uri resultUri = UCrop.getOutput(data);
        } else if (resultCode == UCrop.RESULT_ERROR) {
            final Throwable cropError = UCrop.getError(data);
        }
    }
```

### 3.类关系图

![uCrop-classes-relation.png](http://upload-images.jianshu.io/upload_images/1485091-2dc8413a8b747851.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

`uCrop`的整体设计非常的清晰,这里的类图我省去了`UCrop`类和`UCropActivity`类,我只画了最核心功能的类图,从类图上来看`UCropView`包含了`OverlayView`是用来绘制裁剪页面上的裁剪格子的,整体的裁剪功能都是通过`GestureCropImageView`继承`CropImageView`继承`TransformImageView`然后最终继承自`ImageView`的这三个类来完成了,`GestureCropImageView`负责监听各种手势然后调用父类的方法来完成图片的旋转,方法和位移操作,`CropImageView`是用来完成图片裁剪工作,以及确保图片是处在正确的状态,以及负责完成一些动画.`TransformImageView`则是负责旋转,放大,缩小以及位移操作的。我们先对`uCrop`有一个整体的了解,下面我们就来具体看看`uCrop`是如何实现的:

### 4.源码分析

#### 1.UCrop和UCropActivity的实现
`UCropActivity`就是我们用来裁剪照片的`Activity`了,对于`Activity`大家应该都很熟悉了,我们就不多说了.而`UCrop`类在前面的使用方法中我们也介绍过了,主要是用来提供一个入口以及一系列的调用方法和提供自定义参数的设定,在此类开源项目中很常见,下面我们就主要介绍`uCrop`库核心的裁剪功能是如何实现的。

#### 2.调用流程分析
看到这里,我假设大家已经有使用过`uCrop`或者已经至少把项目`clone`下来`run`了一遍体验了一下了,在`uCrop`里我们可以通过手势来缩放,旋转图片。那么我们就从一次双指缩放的手势来对整个调用流程进行分析:

**(1)GestureCropImageView的实现**

在看这类有UI控件的项目的时候,我们一般直接找到对应控件然后看具体是如何实现的就行了,这里我们看了`UCropActivity`的布局以及代码就能知道我们想知道的就在`GestureCropImageView`中,所以我们直接看看`GestureCropImageView`是如何实现的:

```java

public class GestureCropImageView extends CropImageView {

    private ScaleGestureDetector mScaleDetector;
    private RotationGestureDetector mRotateDetector;
    private GestureDetector mGestureDetector;
    private float mMidPntX, mMidPntY;

    ...
    @Override
    public boolean onTouchEvent(MotionEvent event) {
        if ((event.getAction() & MotionEvent.ACTION_MASK) == MotionEvent.ACTION_DOWN) {
            cancelAllAnimations();
        }
        //如果有多个手指触摸,则计算出两个手指之前的中心点坐标
        if (event.getPointerCount() > 1) {
            mMidPntX = (event.getX(0) + event.getX(1)) / 2;
            mMidPntY = (event.getY(0) + event.getY(1)) / 2;
        }
        //依次将事件传递给mGestureDetector,mIsRotateEnabled,mRotateDetector处理
        mGestureDetector.onTouchEvent(event);

        if (mIsScaleEnabled) {
            mScaleDetector.onTouchEvent(event);
        }

        if (mIsRotateEnabled) {
            mRotateDetector.onTouchEvent(event);
        }
        //如果手指离开,则检测图片是否完全充满裁剪框,如果没有则缩放至完全充满裁剪框
        if ((event.getAction() & MotionEvent.ACTION_MASK) == MotionEvent.ACTION_UP) {
            setImageToWrapCropBounds();
        }
        return true;
    }

    @Override
    protected void init() {
        super.init();
        setupGestureListeners();
    }

    private void setupGestureListeners() {
        mGestureDetector = new GestureDetector(getContext(), new GestureListener(), null, true);
        mScaleDetector = new ScaleGestureDetector(getContext(), new ScaleListener());
        mRotateDetector = new RotationGestureDetector(new RotateListener());
    }
    ...
}
```

上面就是`GestureCropImageView`的实现,这里省略了构造方法以及其余一些代码,总体代码不多,我们可以很清楚的看到首先初始化了`ScaleGestureDetector`,`RotationGestureDetector`和`GestureDetector`三个手势监听的相关类,然后在`onTouchEvent()`方法中依次交给这三个`GestureDetector`来处理触摸事件。如果有指定的触摸事件发生则会回调对应的接口,然后就执行相应的操作了。

在使用`uCrop`中我们发现可以将图片拖动出我们的裁剪框之外,但是松手之后图片都会自动回弹回去并自动适应裁剪框,当缩放或者旋转操作时都有可能触发,那么这个到底是如何实现的呢?实际上就是上面`onTouchEvent()`方法中最后调用的那个方法`setImageToWrapCropBounds();`中实现的,我们下面会详细分析。那么好我们已经知道了`GestureCropImageView`类是如何实现以及它的职责了,那么现在我们假设我们做了一个双指缩放的手势,这将会回调`ScaleListener`的`onScale()`方法,代码如下:

```java

    private class ScaleListener extends ScaleGestureDetector.SimpleOnScaleGestureListener {
        @Override
        public boolean onScale(ScaleGestureDetector detector) {
            postScale(detector.getScaleFactor(), mMidPntX, mMidPntY);
            return true;
        }
    }
```
可以看到调用了`postScale()`方法,其中`detector.getScaleFactor()`是表示当前两个手指之间的距离与上一次移动的手指距离之比,`mMidPntX`和`mMidPntY`是量手指之间的中心点坐标,我们跟进`postScale()`方法,发现它是在`CropImageView`里也就是`GestureCropImageView`的父类中实现的,接下来我们转到`CropImageView`中。

**(2)CropImageView中postScale()的实现**

```java
    public void postScale(float deltaScale, float px, float py) {
        if (deltaScale > 1 && getCurrentScale() * deltaScale <= getMaxScale()) {
            super.postScale(deltaScale, px, py);
        } else if (deltaScale < 1 && getCurrentScale() * deltaScale >= getMinScale()) {
            super.postScale(deltaScale, px, py);
        }
    }
```
很简单,就是判断了还是否可以缩放,然后继续调用了父类的`postScale()`方法,我们知道`CropImageView`父类是`TransformImageView`那我们继续来看:

**(3)TransformImageView中postScale()的实现**

```java

    public void postScale(float deltaScale, float px, float py) {
        if (deltaScale != 0) {
            //变化当前的matrix对象
            mCurrentImageMatrix.postScale(deltaScale, deltaScale, px, py);
            //设置ImageMatrix
            setImageMatrix(mCurrentImageMatrix);
            //回调mTransformImageListener接口
            if (mTransformImageListener != null) {
                mTransformImageListener.onScale(getMatrixScale(mCurrentImageMatrix));
            }
        }
    }
```
可以看到`TransformImageView`中根据设置`ImageView`中的`matrix`对象来使图片进行缩放变化的,`Matrix`在我们进行图像变换处理时经常用到,具体的介绍和详解请参照[这篇文章](http://www.cnblogs.com/qiengo/archive/2012/06/30/2570874.html#code)。如果原理看不懂可以直接看下面代码中是如何使用的即可。

然后我们看到`setImageMatrix(mCurrentImageMatrix);`又调用了`updateCurrentImagePoints();`方法:

```java

    private void updateCurrentImagePoints() {
        mCurrentImageMatrix.mapPoints(mCurrentImageCorners, mInitialImageCorners);
        mCurrentImageMatrix.mapPoints(mCurrentImageCenter, mInitialImageCenter);
    }
```
这里的`mCurrentImageMatrix.mapPoints(float[] dst, float[] src);`方法的意思就是将`src`数组通过这个`matrix`变换赋值给`dst`数组,在这里的意思就是将最初我们保存的四个顶点的数组`mInitialImageCorners`通过这个`matrix`变换后赋值给`mCurrentImageCorners`。同样`mInitialImageCenter`中保存的中点坐标也进行对应的操作,之所以保存这些是因为我们接下来的运算要使用.

其实到这里我们一次缩放的手势就分析完了,这时候如果我们将手指抬起就会调用`GestureCropImageView`中的`setImageToWrapCropBounds();`方法,前面我们已经介绍了这个方法的作用,下面我们就具体来看看它是怎么实现的:

#### 3.setImageToWrapCropBounds()方法的实现

`setImageToWrapCropBounds();`方法是在`CropImageView`里实现的:

```java

    public void setImageToWrapCropBounds() {
        setImageToWrapCropBounds(true);
    }

    public void setImageToWrapCropBounds(boolean animate) {
        if (!isImageWrapCropBounds()) {
            ...
        }
    }
```
直接调用了`setImageToWrapCropBounds(boolean animate);`所以这里传入的`bool`值就是是否做动画,这里是`true`,这里先判断了`isImageWrapCropBounds()`,如果返回`false`才执行具体的代码.这里我们先省略,那么`isImageWrapCropBounds()`方法从名字上看是检测图片当前是不是包裹了裁剪的区域。如果返回是`false`那么里面的逻辑肯定是对图片进行位移或者缩放变换然后充满裁剪区域。我们先来看看`isImageWrapCropBounds()`如何实现的:

```java

    protected boolean isImageWrapCropBounds() {
        //将当前保存的图片的四个顶点数组传入
        return isImageWrapCropBounds(mCurrentImageCorners);
    }

    protected boolean isImageWrapCropBounds(float[] imageCorners) {
        mTempMatrix.reset();
        //利用一个matrix先逆旋转当前旋转的角度.
        mTempMatrix.setRotate(-getCurrentAngle());
        //得到不旋转的图片的顶点数组
        float[] unrotatedImageCorners = Arrays.copyOf(imageCorners, imageCorners.length);
        mTempMatrix.mapPoints(unrotatedImageCorners);
        //先从mCropRect得到四个顶点数组,然后做matrix变换,这里就有逆向的旋转变换了
        float[] unrotatedCropBoundsCorners = RectUtils.getCornersFromRect(mCropRect);
        mTempMatrix.mapPoints(unrotatedCropBoundsCorners);
        //最后比较当前图片所形成的Rect是否包含旋转过后的mCropRect所形成的Rect
        //RectUtils.trapToRect(float[] array)方法是用来获得包含当前所有点的最小矩形
        return RectUtils.trapToRect(unrotatedImageCorners).contains(RectUtils.trapToRect(unrotatedCropBoundsCorners));
    }
```
因为这里也算是一个比较`trick`的做法,先将原图转换成未旋转的状态,然后再旋转我们裁剪的区域,然后获得到这个区域形成的最小矩形,看看是否包含在原图的区域中来判断裁剪区域是否完全在图片中,这里也有一篇`uCrop`的开发者写的文章[我们是如何开发uCrop的](https://yalantis.com/blog/how-we-created-ucrop-our-own-image-cropping-library-for-android/)。里面同样解释了为何这样做。

如果这里返回了`false`就会执行`if`里的内容,那么我们来看看到底是如何实现的:

```java

    public void setImageToWrapCropBounds(boolean animate) {
        if (!isImageWrapCropBounds()) {
            //得到当前图片的中心点坐标
            float currentX = mCurrentImageCenter[0];
            float currentY = mCurrentImageCenter[1];
            //得到当前图片的缩放比例
            float currentScale = getCurrentScale();
            //得到裁剪区域中心和当前图片中心的偏移量
            float deltaX = mCropRect.centerX() - currentX;
            float deltaY = mCropRect.centerY() - currentY;
            float deltaScale = 0;
            //给mTempMatrix 设置偏移量
            mTempMatrix.reset();
            mTempMatrix.setTranslate(deltaX, deltaY);
            //对图片做matrix变换.
            final float[] tempCurrentImageCorners = Arrays.copyOf(mCurrentImageCorners, mCurrentImageCorners.length);
            mTempMatrix.mapPoints(tempCurrentImageCorners);
            //做完变换后再检测是否平移变换之后,图片就充满了裁剪区域
            boolean willImageWrapCropBoundsAfterTranslate = isImageWrapCropBounds(tempCurrentImageCorners);

            if (willImageWrapCropBoundsAfterTranslate) {
                //如果平移转换就可以充满裁剪区域
                //那么就找出最合适的偏移量
                final float[] imageIndents = calculateImageIndents();
                deltaX = -(imageIndents[0] + imageIndents[2]);
                deltaY = -(imageIndents[1] + imageIndents[3]);
            } else {
                //如果不能充满裁剪区域
                RectF tempCropRect = new RectF(mCropRect);
                mTempMatrix.reset();
                mTempMatrix.setRotate(getCurrentAngle());
                mTempMatrix.mapRect(tempCropRect);

                final float[] currentImageSides = RectUtils.getRectSidesFromCorners(mCurrentImageCorners);

                //算出需要缩放的比例差
                deltaScale = Math.max(tempCropRect.width() / currentImageSides[0],
                        tempCropRect.height() / currentImageSides[1]);
                // Ugly but there are always couple pixels that want to hide because of all these calculations
                deltaScale *= 1.01;
                deltaScale = deltaScale * currentScale - currentScale;
            }

            //如果需要动画
            if (animate) {
                post(mWrapCropBoundsRunnable = new WrapCropBoundsRunnable(
                        CropImageView.this, mImageToWrapCropBoundsAnimDuration, currentX, currentY, deltaX, deltaY,
                        currentScale, deltaScale, willImageWrapCropBoundsAfterTranslate));
            } else {
                postTranslate(deltaX, deltaY);
                if (!willImageWrapCropBoundsAfterTranslate) {
                    zoomInImage(currentScale + deltaScale, mCropRect.centerX(), mCropRect.centerY());
                }
            }
        }
    }

```
上面就是如何计算位移以及缩放的代码了,注意在最后的时候如果需要动画的话,则通过一个`Runnable`对象`mWrapCropBoundsRunnable`来进行动画,具体的逻辑大家可以自行去看看也是比较清晰的。

至此我们就大致分析了`uCrop`总体上是如何实现的,关于怎么加载的图片以及如何裁剪的图片我们这篇文章就不分析了,有兴趣的同学可以自行研究。

### 5.个人评价
不愧是目前最优秀的图片剪裁库,无论从整个库产品方面的设计还是从代码的结构上来看,`uCrop`都是值得我们学习与使用的,以前也阅读过其他裁剪项目的源代码,整体对比上来看`uCrop`提供的方法以及定制性和易用性都是很棒的,值得推荐!