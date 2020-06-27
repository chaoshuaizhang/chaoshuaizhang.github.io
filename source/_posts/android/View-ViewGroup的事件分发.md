---
title: View/ViewGroup的事件分发
date: 2018-03-24 11:37:17
tags: 事件分发
categories: android
---

## 先看一张大致的流程图
![Alt text](./1564283863879.png)
>onIntercept、onTouchEvent都是在dispatch中调用的，以dispatch为入口在不同的情况下调用intercept、onTouchEvent，最终Activity根据其返回结果判断是否执行自己的onTouchEvent。

## 执行流程分析
>事件从进入Activity的dispatch方法开始，到出了Activity的dispatch方法结束。

### 事件交给Activity处理
>Activity会先交给自己的view来处理，如果它们都不处理，Activity自己才处理。
```java
//Activity.java
public boolean dispatchTouchEvent(MotionEvent ev) {
	if (ev.getAction() == MotionEvent.ACTION_DOWN) {
		//我的理解：这个方法就是做一些类似beforeDone的事
		onUserInteraction();
	}
	//直接调用window的superdispatch，经历一系列的转发，丢给Decorview，让它开始分发给自己的View，看谁消费。
	if (getWindow().superDispatchTouchEvent(ev)) {
		return true;
	}
	//如果没有View进行消费，那么就交给Activity自己处理！
	return onTouchEvent(ev);
}
```
######  phoneWondow
```java
//PhoneWindow.java
public boolean superDispatchTouchEvent(MotionEvent event) {
	//在attach阶段已经实例化了一个DecorView了，这里调用decorview的superdispatch
	return mDecor.superDispatchTouchEvent(event);
}
```
###### decorView
```java
//DecorView.java
public boolean superDispatchTouchEvent(MotionEvent event) {
	//DecorView是一个ViewGroup，这里会调用VG的dispatch
	return super.dispatchTouchEvent(event);
}
```

>前边墨迹了这么长时间（没有一点业务逻辑的调用），我的理解就是为了把事件传给DecorView，然后让decorView分发给自己的子view。

### 事件分发到ViewGroup中
>ViewGroup的dispatch里有很多逻辑，同时它会和View的dispatch进行交互，它调用View的dispatch的原因是：
>1. 当前ViewGroup要消费事件
>2. 看子View是否消费事件
>具体逻辑可以看下边的`dispatchTransformedTouchEvent`方法

#### dispatchTouchEvent方法
>ViewGroup的dispatch方法太长了，我分为了三步进行讲解
```java
public boolean dispatchTouchEvent(MotionEvent ev) {
	boolean handled = false;
	//第一步：处理down事件，并且得到当前ViewGroup是否拦截的结论
	//第二步：父ViewGroup不拦截，遍历子View，如果子View消费了，就把子View复制给mFirstTouchTarget（这些逻辑也只是在down时才会执行）
	//第三步：根据mFirstTouchTarget是否为null，事件交给当前ViewGroup处理还是子View处理
	return handled;
}
```

##### 第一步：对down事件进行处理
```java
final int action = ev.getAction();
final int actionMasked = action & MotionEvent.ACTION_MASK;
// Handle an initial down.
if (actionMasked == MotionEvent.ACTION_DOWN) {
	//如果是down事件的话，说明是新一轮的触摸事件来了，清除所有之前的标识。
	//重要的一点：置空mFirstTouchTarget
	cancelAndClearTouchTargets(ev);
	resetTouchState();
}
// Check for interception.
final boolean intercepted;
//如果当前不是down事件，若mFirstTouchTarget==null，说明是ViewGroup自己消费down事件了，如果!=null，说明子view消费down事件了
if (actionMasked == MotionEvent.ACTION_DOWN || mFirstTouchTarget != null) {
	//看下当前ViewGroup是否拦截事件
	intercepted = onInterceptTouchEvent(ev);
} else {
	intercepted = true;
}
```
##### 第二步：看子View是否消费事件
>如果子View消费事件的话，在这里只会消费down事件，其它如move、up事件，在第三步消费
```java
TouchTarget newTouchTarget = null;
boolean alreadyDispatchedToNewTouchTarget = false;
if (!canceled && !intercepted) {
	//如果不进行拦截
	View childWithAccessibilityFocus = ev.isTargetAccessibilityFocus() ? findChildWithAccessibilityFocus() : null;
	//【注意1】这里ACTION_DOWN、ACTION_POINTER_DOWN、ACTION_HOVER_MOVE都会走到这里
	if (actionMasked == MotionEvent.ACTION_DOWN || (split && actionMasked == MotionEvent.ACTION_POINTER_DOWN) || actionMasked == MotionEvent.ACTION_HOVER_MOVE) {
		//如果当前是down事件，说明这是第一次进入dispatch
		final int childrenCount = mChildrenCount;
		if (newTouchTarget == null && childrenCount != 0) {
			for (int i = childrenCount - 1; i >= 0; i--) {
				final int childIndex = getAndVerifyPreorderedIndex(childrenCount, i, customOrder);
				final View child = getAndVerifyPreorderedView(preorderedList, children, childIndex);
				//第一次进来时newTouchTarget为null
				newTouchTarget = getTouchTarget(child);
				if (newTouchTarget != null) {
					newTouchTarget.pointerIdBits |= idBitsToAssign;
					break;
				}
				resetCancelNextUpFlag(child);
				//事件交由当前的child，看其是否处理
				if (dispatchTransformedTouchEvent(ev, false, child, idBitsToAssign)) {
					//如果当前的child消费了事件，就保存mFirstTouchTarget=child，并且此时newTouchTarget也是child
					//【注意2】这里会在mFirstTouchTarget前边（first是一个链表结构）添加一个child
					//只有down、ACTION_POINTER_DOWN、ACTION_HOVER_MOVE事件，这里的newTouchTarget才=null
					newTouchTarget = addTouchTarget(child, idBitsToAssign);
					alreadyDispatchedToNewTouchTarget = true;
					break;
				}
			}
		}
	}
}
```
##### 第三步：决定是谁消费事件
>注意：
>* 如果是子View消费事件的话，走到这里的时候，down事件已经被消费了（因为子View已经执行了`dispatchTransformedTouchEvent`方法），在这里只会再消费move、up等事件
>* 如果是ViewGroup自己消费的话，down、move、up都是第三步执行
```java
if (mFirstTouchTarget == null) {
	// No touch targets so treat this as an ordinary view.
	//走到这里说明上述进行拦截了（如果不拦截的话，mFirstTouchTarget就不为空）
	//可以看到这里的child对象是null，也就是接下来会调用super.dispatch，而不是child.dispatch
	//handled = 当前viewgroup的onTouch或者onTouchEvent的返回值
	handled = dispatchTransformedTouchEvent(ev, canceled, null, TouchTarget.ALL_POINTER_IDS);
} else {
	//走到这里说明上边是子View消费事件了，子View消费down事件后，move、up都会走到这里
	TouchTarget predecessor = null;
	TouchTarget target = mFirstTouchTarget;
	//下边的循环不是很清楚，mFirstTouchTarget就是目标view，为什么还要循环查找呢？
	//难道是因为上述【注意1】的原因？除了DOWN之外的那俩事件也会走到【注意2】中，添加到mFirstTouchTarget，所以这里要循环查找真正消费事件的View？
	while (target != null) {
		final TouchTarget next = target.next;
		if (alreadyDispatchedToNewTouchTarget && target == newTouchTarget) {
			//只有down时newTouchTarget才不为null，所以走到这里的应该是down事件
			handled = true;
		} else {
			final boolean cancelChild = resetCancelNextUpFlag(target.child) || intercepted;
			if (dispatchTransformedTouchEvent(ev, cancelChild, target.child, target.pointerIdBits)) {
				handled = true;
			}
			if (cancelChild) {
				if (predecessor == null) {
					mFirstTouchTarget = next;
				} else {
					predecessor.next = next;
				}
				target.recycle();
				target = next;
				continue;
			}
		}
		predecessor = target;
		target = next;
	}
}
```

#### onInterceptTouchEvent方法
>只有ViewGroup才有intercept方法，View无法对时间进行拦截。
```java
public boolean onInterceptTouchEvent(MotionEvent ev) {
	if (ev.isFromSource(InputDevice.SOURCE_MOUSE)
			&& ev.getAction() == MotionEvent.ACTION_DOWN
			&& ev.isButtonPressed(MotionEvent.BUTTON_PRIMARY)
			&& isOnScrollbarThumb(ev.getX(), ev.getY())) {
		return true;
	}
	//可以认为，VG的intercept默认返回false
	return false;
}
```

#### dispatchTransformedTouchEvent方法
>不管父view拦不拦截事件，都会走到此方法：
>父VG拦截：传入的child是null
>父VG不拦截：传入的child是子view
>**这个方法很重要：VG的dispatch通过这个方法来和View的dispatch交互**
```java
private boolean dispatchTransformedTouchEvent(MotionEvent event, boolean cancel, View child, int desiredPointerIdBits) {
	// Perform any necessary transformations and dispatch.
	//大致的代码就是：子View是null了，说明是ViewGroup拦截了，所以调用父View的dispatch
	if (child == null) {
		handled = super.dispatchTouchEvent(transformedEvent);
	} else {
		//父ViewGroup不拦截，则继续分发，交由子View处理
		handled = child.dispatchTouchEvent(transformedEvent);
	}
	return handled;
}
```

#### View的dispatchTouchEvent方法
>会在这个方法中检验是否执行onTouch、onTouchEvent、onClick、longClick

```java
public boolean dispatchTouchEvent(MotionEvent event) {
	boolean result = false;
	final int actionMasked = event.getActionMasked();
	if (actionMasked == MotionEvent.ACTION_DOWN) {
		//down事件之前如果在滑动，则立马停止
		stopNestedScroll();
	}
	if (onFilterTouchEventForSecurity(event)) {
		...
		//noinspection SimplifiableIfStatement
		ListenerInfo li = mListenerInfo;
		//1. 这里会判断是否设置了onTouchListener监听
		//执行onTouch方法，如果onTouch返回为true，则不再执行onTouchEvent，并且dispatch的最终结果也是返回true
		if (li != null && li.mOnTouchListener != null && (mViewFlags & ENABLED_MASK) == ENABLED && li.mOnTouchListener.onTouch(this, event)) {
			result = true;
		}
		//如果onTouch返回false，则才会执行onTouchevent，并且dispatchTouchEvent的返回值和onTouchEvent一致
		if (!result && onTouchEvent(event)) {
			result = true;
		}
	}
	...
	return result;
}
```

#### onTouchEvent方法
>click在up事件中触发，longClick在down事件中触发
```java
public boolean onTouchEvent(MotionEvent event) {
	if (mTouchDelegate != null) {
		//如果有代理view的话，交给代理view执行onTouchEvent
		if (mTouchDelegate.onTouchEvent(event)) {
			return true;
		}
	}
	if (clickable || (viewFlags & TOOLTIP) == TOOLTIP) {
		switch (action) {
			case MotionEvent.ACTION_UP:
				boolean prepressed = (mPrivateFlags & PFLAG_PREPRESSED) != 0;
				if ((mPrivateFlags & PFLAG_PRESSED) != 0 || prepressed) {
					boolean focusTaken = false;
					if (!mHasPerformedLongPress && !mIgnoreNextUpEvent) {
						// Only perform take click actions if we were in the pressed state
						if (!focusTaken) {
							if (mPerformClick == null) {
								//PerformClick实现自Runnable接口
								mPerformClick = new PerformClick();
							}
							//先通过handler发送消息，失败的话，再通过接口回调
							if (!post(mPerformClick)) {
								performClick();
							}
						}
					}
				}
				break;
			case MotionEvent.ACTION_DOWN:
				//longClick事件在这里执行
				break;
			case MotionEvent.ACTION_MOVE:
				//从这里我们可以知道为什么点击按钮，但是滑动到其它View上时，click事件不触发的原因。
				if (!pointInView(x, y, mTouchSlop)) {
					// Outside button
					// Remove any future long press/tap checks
					removeTapCallback();
					removeLongPressCallback();
					if ((mPrivateFlags & PFLAG_PRESSED) != 0) {
						setPressed(false);
					}
					mPrivateFlags3 &= ~PFLAG3_FINGER_DOWN;
				}
				break;
		}

		return true;
	}
	return false;
}
```
#### dispatch分发完成
>Activity会根据view是否消费了事件来决定是否执行自己的onTouchEvent方法