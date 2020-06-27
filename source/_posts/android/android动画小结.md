---
title: android动画小结
date: 2017-07-30 17:13:06
tags: 动画
categories: android
---

## android动画小结

>本篇介绍三种动画
>1. 帧动画
>2. 补间动画(View动画)
>3. 属性动画


### 帧动画
>依赖于完善的UI资源，它的原理就是将一张张单独的图片连贯的进行播放，从而在视觉上产生一种动画的效果。有点类似于某些软件制作gif动画的方式。
如何使用
* 一般是在drawable中新建一个frame_anim.xml
* 在drawable中放置一些图片

#### 在XML中配置
```java
<?xml version="1.0" encoding="utf-8"?>
<animation-list xmlns:android="http://schemas.android.com/apk/res/android">
    <item android:drawable="@drawable/refresh_01" android:duration="1000"/>
    <item android:drawable="@drawable/refresh_08" android:duration="1000"/>
</animation-list>
```

##### 使用
```java
//有三种使用方式
//第一种方式设置
imageView.setImageResource(R.drawable.frame_anim);
animationDrawable = (AnimationDrawable) imageView.getDrawable();

//第二种方式设置
imageView.setBackgroundResource(R.drawable.frame_anim);
animationDrawable = (AnimationDrawable) imageView.getBackground();

//第三种方式设置
//getDrawable建议使用下面的这种方式
animationDrawable = (AnimationDrawable) ContextCompat.getDrawable(getApplicationContext(), R.drawable.frame_anim);
imageView.setBackground(animationDrawable);
```

#### 在代码中设置
```java
animationDrawable = new AnimationDrawable();
animationDrawable.addFrame(ContextCompat.getDrawable(getApplicationContext(), R.drawable.refresh_01), 1000);
......
animationDrawable.addFrame(ContextCompat.getDrawable(getApplicationContext(), R.drawable.refresh_08), 1000);
imageView.setBackground(animationDrawable);
```
##### 其它属性设置
```java
//设置是否只执行一次
animationDrawable.setOneShot(false);
```

### 补间动画（View动画）
>专为View打造的动画。通过对View的内容进行一系列的图形变换 (包括平移、缩放、旋转、改变透明度)来实现动画效果。
如何使用：在res下新建anim文件夹，然后在anim中新建对应的xml
#### XML方式实现
```java
//透明度
<alpha xmlns:android="http://schemas.android.com/apk/res/android"
       android:duration="2000"
       android:fromAlpha="1"
       android:interpolator="@android:anim/accelerate_interpolator"
       android:toAlpha="0"
    />
//旋转
<rotate xmlns:android="http://schemas.android.com/apk/res/android"
        android:duration="2000"
        //起始角度
        android:fromDegrees="0"
        //差值器
        android:interpolator="@android:anim/accelerate_decelerate_interpolator"
        //旋转中心点（xml默认是相对于自身的），如何改为相对于父布局，或者相对于某一个绝对位置点？
        //难道只能使用java代码设置？
        android:pivotX="50%"
        android:pivotY="50%"
        //结束角度（0-360相当于转一周）
        android:toDegrees="-360"
    />

//缩放
<scale xmlns:android="http://schemas.android.com/apk/res/android"
    android:duration="500"
    android:fromXScale="0.0"
    android:fromYScale="0.0"
    android:interpolator="@android:anim/decelerate_interpolator"
    //相对于某个位置（点）缩放，如果是%数，表示相对于view%数的位置，如果是数字
    //则表示旋转点是view的x坐标+pivotX，同理y
    android:pivotX="50%"
    android:pivotY="50%"
    //重复次数
    android:repeatCount="1"
    //动画重复的模式，reverse为反向，当第偶次执行时，动画方向会相反。restart为重新执行，方向不变
    android:repeatMode="reverse"
    //动画多次执行的间隔时间，如果只执行一次，执行前会暂停这段时间，单位毫秒 
    android:startOffset="0"
    android:toXScale="1.5"
    android:toYScale="1.5" />

//平移
<translate xmlns:android="http://schemas.android.com/apk/res/android"
    android:duration="500"
    //起始点
    android:fromXDelta="100"
    android:fromYDelta="0"
    android:interpolator="@android:anim/cycle_interpolator"
    //结束点
    android:toXDelta="0"
    android:toYDelta="0" />

//集合
<set xmlns:android="http://schemas.android.com/apk/res/android">
    <alpha
        android:duration="500"
        android:fromAlpha="1.0"
        android:toAlpha="0.0" />

    <scale
        android:duration="500"
        android:fromXScale="0.0"
        android:fromYScale="0.0"
        android:interpolator="@android:anim/decelerate_interpolator"
        android:pivotX="50%"
        android:pivotY="50%"
        android:repeatCount="1"
        android:repeatMode="reverse"
        android:startOffset="0"
        android:toXScale="1.5"
        android:toYScale="1.5" />
</set>
```

```java
//以alpha举例，其它动画设置一样，AnimationSet也是如此
Animation alphaAnimation = AnimationUtils.loadAnimation(getApplicationContext(), R.anim.anim_alpha);
//使用
imageView.startAnimation(alphaAnimation);//使用一种动画
//当使用动画集合时，true表示使用AnimationSet提供的的插值器，false表示使用各自的插值器
AnimationSet animationSet = new AnimationSet(true);
//注意，如果想让一组动画持续执行，对animationSet设置repeat貌似不管用，还要对各个animation单独设置
animationSet.addAnimation(alphaAnimation);//添加动画
```

#### java代码设置
```java
alphaAnimation = new AlphaAnimation(1.0f, 0.0f);

//(fromX, toX, fromY, toY,pivotXType, pivotXValue, pivotYType, pivotYValue)
//pivotXType：旋转点是相对于自身，还是绝对的坐标点，还是相对于父View的位置
/**
ABSOLUTE：具体的坐标值
RELATIVE_TO_SELF：相对于自身
RELATIVE_TO_PARENT：相对于父容器
pivotXValue：This value can either be an absolute number if pivotXType is ABSOLUTE, or a percentage (where 1.0 is 100%) otherwise
*/
scaleAnimation = new ScaleAnimation(0.0f, 1.5f, 0.0f, 1.5f, Animation.RELATIVE_TO_SELF, 0.5f, Animation.RELATIVE_TO_SELF, 0.5f);

rotateAnimation = new RotateAnimation(0, 360, Animation.RELATIVE_TO_SELF, 0.5f, Animation.RELATIVE_TO_SELF, 0.5f);

translateAnimation = new TranslateAnimation(0, 100, 0, 0);
//同样可以使用AnimationSet来设置多个动画。
```

#### 事件监听
```java
alphaAnimation.setAnimationListener(new Animation.AnimationListener() {
    @Override
    public void onAnimationStart(Animation animation) {
        //动画开始时调用
    }

    @Override
    public void onAnimationEnd(Animation animation) {
        //动画结束时调用
    }

    @Override
    public void onAnimationRepeat(Animation animation) {
        //动画重复时调用
    }
    });
```

#### 常用插值器
```java
AccelerateInterpolator 加速，开始时慢中间加速
DecelerateInterpolator 减速，开始时快然后减速
AccelerateDecelerateInterolator 先加速后减速，开始结束时慢，中间加速
AnticipateInterpolator 反向，先向相反方向改变一段再加速播放
AnticipateOvershootInterpolator 反向加超越，先向相反方向改变，再加速播放，会超出目的值然后缓慢移动至目的值
BounceInterpolator 跳跃，快到目的值时值会跳跃，如目的值100，后面的值可能依次为85，77，70，80，90，100
CycleIinterpolator 循环，动画循环一定次数，值的改变为一正弦函数：Math.sin(2* mCycles* Math.PI* input)
LinearInterpolator 线性，线性均匀改变
OvershootInterpolator超越，最后超出目的值然后缓慢改变到目的值
```

#### 注意事项
* 发现View有个startAnimation和setAnimation，startAnimation是立即开启动画。setAnimation这个不太清楚，但是和父容器的invalidate()有关。
* 补间动画有一个致命的缺陷，就是它只是改变了View的显示效果而已，而不会真正去改变View的属性。假设View有点击事件，即使View从左上角移动到右上角，仍然是点击左上角有响应，右上角没有。

### 属性动画

>属性动画机制已经不再是针对于View来设计的了，也不限定于只能实现移动、缩放、旋转和淡入淡出这几种动画操作，同时也不再只是一种视觉上的动画效果了。它实际上是一种不断地对值进行操作的机制，并将值赋值到指定对象的指定属性上，可以是任意对象的任意属性。所以我们仍然可以将一个View进行移动或者缩放，但同时也可以对自定义View中的Point对象进行动画操作了。我们只需要告诉系统动画的运行时长，需要执行哪种类型的动画，以及动画的初始值和结束值，剩下的工作就可以全部交给系统去完成了。
>XML和java的对应关系：
>* <animator>  对应代码中的ValueAnimator
* <objectAnimator>  对应代码中的ObjectAnimator
* <set>  对应代码中的AnimatorSet

#### ValueAnimator
>ValueAnimator貌似不能对View进行设置，它操作的就是一个值，在这个值变化的过程中会不断的回调，在回掉方法里拿到值设置View。

##### java代码设置
```java
//在10s内，从100平滑过渡到0
ValueAnimator valueAnimator = ValueAnimator.ofInt(100, 0);
//多次过渡：从100->0->10->20->0
ValueAnimator floadValueAnimator = ValueAnimator.ofFloat(100, 0, 10, 20, 0);
//颜色的过度
ValueAnimator colorValueAnimator = ValueAnimator.ofArgb(Color.BLACK, Color.BLUE, Color.RED, Color.GRAY, Color.GREEN);
colorValueAnimator.setFrameDelay(2000);
colorValueAnimator.setDuration(10 * 1000);
colorValueAnimator.addUpdateListener(new ValueAnimator.AnimatorUpdateListener() {
    @Override
    public void onAnimationUpdate(ValueAnimator animation) {
        Object animatedValue = animation.getAnimatedValue();
        Log.i(TAG, "onAnimationUpdate: " + animatedValue);
        //【注意】在这里得到设置的值（根据时间平滑改变的），然后设置view
        (findViewById(R.id.activity_value)).setBackgroundColor((int) animatedValue);
    }
});
colorValueAnimator.setRepeatCount(ValueAnimator.INFINITE);
colorValueAnimator.setRepeatMode(ValueAnimator.REVERSE);
colorValueAnimator.addListener(new AnimatorListenerAdapter() {
		//可以重写自己需要的方法，而不用重写其它用不到的方法
	});
colorValueAnimator.start();
```

##### XML设置
```java
<animator xmlns:android="http://schemas.android.com/apk/res/android"
          android:duration="10000"
          android:valueFrom="100"
          android:valueTo="0"
          android:valueType="intType"
    />

//最好使用getApplicationContext()，否则会内存泄漏（当关闭Activity后，动画还在执行）
ValueAnimator valueAnimator = (ValueAnimator) 
AnimatorInflater.loadAnimator(getApplicationContext(), R.animator.value_animator);
valueAnimator.setTarget((findViewById(R.id.tv_obj_animator)));
valueAnimator.addUpdateListener(new ValueAnimator.AnimatorUpdateListener() {
    @Override
    public void onAnimationUpdate(ValueAnimator animation) {
        Log.i(TAG, "onAnimationUpdate: " + animation.getAnimatedValue());
    }
});
valueAnimator.start();
```

#### ObjectAnimator
>常用的就是ObjectAnimator，ValueAnimator只不过是对值进行了一个平滑的动画过渡，ObjectAnimator则是可以直接对任意对象的任意属性（应该只是那些具有set方法的属性，属性xxx对应setXxx）行动画操作的，比如说View的alpha属性。

##### java实现
```java
ObjectAnimator alphaAnimator = ObjectAnimator.ofFloat(findViewById(R.id.tv_obj_animator), "alpha", 1f, 0.5f, 1f, 0f, 1f);
ObjectAnimator colorAnimator = ObjectAnimator.ofArgb(findViewById(R.id.tv_obj_animator), "textColor", Color.BLUE, Color.GREEN, Color.RED);
//这个size不起作用
ObjectAnimator sizeAnimator = ObjectAnimator.ofInt(findViewById(R.id.tv_obj_animator), "textSize", 20, 40, 20, 60, 100, 50);
//动画集合
AnimatorSet animatorSet = new AnimatorSet();
//当设置的after和之前设置的动画有重复时，则动画就不执行了，很奇怪，可以看下源码
//比如：animatorSet.play(alphaAnimator).with(colorAnimator).after(colorAnimator);
animatorSet.play(alphaAnimator).with(colorAnimator).with(sizeAnimator);
//当设置了set的时间后，单独的动画设置的时间就没用了
animatorSet.setDuration(3000);
alphaAnimator.setDuration(10000);
alphaAnimator.setRepeatCount(1);
alphaAnimator.setRepeatMode(ValueAnimator.REVERSE);
animatorSet.start();
```

##### XML实现
```java
<objectAnimator xmlns:android="http://schemas.android.com/apk/res/android"
	//属性名
	android:propertyName="alpha"
	android:repeatCount="infinite"
	android:repeatMode="reverse"
	android:valueFrom="1"
	android:valueTo="0"
	android:duration="10000"
	android:valueType="floatType"
/>

ObjectAnimator objectAnimator = (ObjectAnimator) AnimatorInflater.loadAnimator(getApplicationContext(), R.animator.obj_animator);
objectAnimator.setTarget((findViewById(R.id.tv_obj_animator)));
objectAnimator.start();
```

###### Set的XML实现
>Set的java实现可以看上边（注意一点：after不能添加和前边一致的动画，before应该也是，可以看下源码），XML的实现会涉及到顺序的问题。

```java
<set xmlns:android="http://schemas.android.com/apk/res/android"
     android:ordering="together">
    <!--together和sequentially：
        当A集合为together时，说明这个集合里边的动画一块儿执行，
        当一块执行的也是个集合B时：如果B是sequentially时，说明B会和A一块儿执行，但是B里边的会按顺序依次执行
    -->
    <objectAnimator
        android:duration="10000"
        android:propertyName="translationX"
        android:valueFrom="100"
        android:valueTo="200"
        android:valueType="floatType"
        />
    <set android:ordering="sequentially">
        <objectAnimator
            android:duration="10000"
            android:propertyName="alpha"
            android:valueFrom="1"
            android:valueTo="0.5"
            />
        <set
            android:ordering="sequentially">
            <!--貌似不能同时在一个objectAnimator中设置x、y缩放-->
            <objectAnimator
                android:duration="5000"
                android:propertyName="scaleX"
                android:valueFrom="1"
                android:valueTo="2"
                android:valueType="floatType"
                />
            <objectAnimator
                android:duration="5000"
                android:propertyName="scaleY"
                android:valueFrom="1"
                android:valueTo="2"
                android:valueType="floatType"
                />
        </set>
    </set>
</set>
```

##### 属性动画进阶

>最主要的是通过自定义TypeEvaluator来制作一些复杂的动画，主要是生成运动轨迹：
>ValueAnimator ofObject(TypeEvaluator evaluator, Object... values)
>ObjectAnimator ofObject(Object target, String propertyName,TypeEvaluator evaluator, Object... values)

[详见AnimationTest项目](D:\myprjs\AnimationTest)


##### 属性动画高阶
>1. 插值器，上边写的有默认的一些插值器
>2. ViewPropertyAnimator的用法

###### 插值器
>概念不用介绍了（重要的就是自定义），比如定义一个先减速后加速的插值器，就可以类比 y=x^2这个函数，如果先加速后减速，则可以类比为y=1/x。

```java
RequiresApi(api = Build.VERSION_CODES.LOLLIPOP_MR1)
public class MyInterpolator extends BaseInterpolator {
    @Override
    public float getInterpolation(float input) {
        if (input == 0) {
            return Float.MAX_VALUE;
        } else {
            return 1 / input;
        }
		//return input * input;
    }
}
```

###### ViewPropertyAnimator
* 普通写法：
```java
ObjectAnimator animator = ObjectAnimator.ofFloat(textview, "alpha", 0f);  
animator.start();  
```

* 使用ViewPropertyAnimator写法：
```java
ViewPropertyAnimator animator = textview.animate();
//有点类似流式写法
animator.alpha(1.0f).rotation(1.0f).setDuration(1000).start();
```