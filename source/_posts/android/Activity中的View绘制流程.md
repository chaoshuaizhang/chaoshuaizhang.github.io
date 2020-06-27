---
title: Activity中的View绘制流程
date: 2018-09-28 19:48:28
tags: 绘制流程
categories: android
---

## 起源
>记得分析事件是如何传递到Activity时有一个点：构造一个ViewRootImpl，并且将root、decor、window建立关联。
`root.setView(view, wparams, panelParentView)`

```java
public void setView(View view, WindowManager.LayoutParams attrs, View panelParentView) {
	...
	// Schedule the first layout -before- adding to the window
	// manager, to make sure we do the relayout before receiving
	// any other events from the system.
	//【注意】注释中有一句很重要的话：要保证在接收到事件前，先布置好布局
	requestLayout();
	...
}
```

#### 请求绘制
```java
public void requestLayout() {
	...
	mLayoutRequested = true;
	scheduleTraversals();
}
```
>最终会在scheduleTraversals方法中调用performTraversals完成绘制**这里的绘制是从顶级View发起的，然后遍历子View循环绘制**

## 绘制流程

#### 核心方法（从顶级开始遍历）
>在performTraversals中依次调用measure、layout、draw。
```java
//ViewRootImpl.java
private void performTraversals() {
	//mWindowAttributes是MATCH_PARENT的
	WindowManager.LayoutParams lp = mWindowAttributes;
	mWidth = frame.width();
	mHeight = frame.height();
	//默认布局的宽高就是屏幕的宽高
	int childWidthMeasureSpec = getRootMeasureSpec(mWidth, lp.width);
	int childHeightMeasureSpec = getRootMeasureSpec(mHeight, lp.height);	
	//执行measure测量
	performMeasure(childWidthMeasureSpec, childHeightMeasureSpec);
	//放置子View
	performLayout(lp, mWidth, mHeight);
	//draw绘制
	performDraw();
}
```

#### 测量measure
```java
//这里调用当前View的measure来进行测量，最后通过onMeasure完成测量
private void performMeasure(int childWidthMeasureSpec, int childHeightMeasureSpec) {
	mView.measure(childWidthMeasureSpec, childHeightMeasureSpec);
}
```

```java
public final void measure(int widthMeasureSpec, int heightMeasureSpec) {
	//这里会执行view自身的onMeasure方法，一般每个子View都会重写onMeasure，依据自己布局的特点来测量
	onMeasure(widthMeasureSpec, heightMeasureSpec);
}
```
>对于最初的布局，mView就是DecorView。在测量时是根据自身的大小和LayoutParams一起计算出来最终大小的。

#### 布局layout
```java
//测量完成后调用layout完成对自己子View的布局
private void performLayout(WindowManager.LayoutParams lp, int desiredWindowWidth, int desiredWindowHeight) {
	//在layout中调用自己的onLayout完成对自己view的摆放
	host.layout(0, 0, host.getMeasuredWidth(), host.getMeasuredHeight());
}
```
```java
protected void onLayout(boolean changed, int left, int top, int right, int bottom) {
	//空的
}
```
>onLayout是一个空方法，和onMeasure一样里边没有逻辑，摆放由自己决定，不同的View的摆放逻辑不一样。我们会在此方法中根据measure的结果把View摆在对应的位置上。

#### 绘制draw
>绘制这一块儿感觉比较复杂。

```java
private void performDraw() {
	draw(fullRedrawNeeded);
}
```

```java
private void draw(boolean fullRedrawNeeded) {
	//在dispatchOnDraw方法中会遍历各个子View执行其自己的onDraw方法
	mAttachInfo.mTreeObserver.dispatchOnDraw();
	...
	mAttachInfo.mThreadedRenderer.draw(mView, mAttachInfo, this);
	//注意这里也可能会调用scheduleTraversals来进行重绘
	if (animating) {
		mFullRedrawNeeded = true;
		scheduleTraversals();
	}
}
```

### 注意
>* 对于measure、layout、draw三个方法，View和ViewGroup的执行逻辑是不一样的。当对一个子布局进行绘制时需要判断其是View还是ViewGroup，如果是ViewGroup的话，继续遍历。。。
* 对于View的invalidate方法也会执行到scheduleTraversals，也可以联想到：ViewGroup中任一view有变更，都会导致ViewRootImpl去重新绘制。
* ViewRootImpl控制着View的刷新等操作，它相当于是介于Activity与View之间的一个桥。

## 总结
>大致的绘制流程就是这样（并没有分析什么特别有用的东西）：在一个方法中先后执行measure、layout、draw完成绘制。布局的变化会导致ViewRootImpl进行重绘。


