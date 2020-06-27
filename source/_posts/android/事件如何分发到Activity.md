---
title: 事件如何分发到Activity
date: 2018-03-24 14:59:27
tags: 事件分发
categories: android
---

不管是Activity的绘制还是事件的传输，都离不开PhoneWindow、DecorView、ViewRootImpl这几个类，我们从0开始看他们如何构建、交互。

## Activity的attach方法
>在ActivityThread中的H发送LAUNCH_ACTIVITY消息时，会在performLaunchActivity中执行activity.attach

### attach做了什么
##### 1. 在attach时创建一个window，并让当前Activity与Window建立关联
```java
//传入的this是当前activity
mWindow = new PhoneWindow(this, window, activityConfigCallback);
//this是当前activity（这里很重要，最后会通过这个回调），设置CallBack，实现dispatchTouchEvent方法
mWindow.setCallback(this);
```

##### 2. PhoneWindow的构造方法
```java
public PhoneWindow(Context context, Window preservedWindow, ActivityConfigCallback activityConfigCallback) {
	//Only main activity windows use decor context, all the other 
	//windows depend on whatever context that was given to them.
	if (preservedWindow != null) {
		//得到main传过来的DecorView
		mDecor = (DecorView) preservedWindow.getDecorView();
	}
}
```
>`Only main activity windows use decor context`对于这句注释之前有点疑惑，以为只有Main Activty会构造一个DecorView，其它的拿的都是Main的
>但是想想不是这样的，每构造一个Activity都会创建一个PhoneWindow，都会生成一个新的DecorView。

###### DecorView的构建过程
在Activity的onCreate方法中执行setContentView时，会调用getWindow().setContentView(resId)

```
// Activity.java
public void setContentView(@LayoutRes int layoutResID) {
	getWindow().setContentView(layoutResID);
	initWindowDecorActionBar();
}

// PhoneWindow.java
public void setContentView(int layoutResID) {
	// Note: FEATURE_CONTENT_TRANSITIONS may be set in the process of installing the window
	// decor, when theme attributes and the like are crystalized. Do not check the feature
	// before this happens.
	if (mContentParent == null) {
	    // 构建装载DecorView
	    installDecor();
	} else if (!hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
	    mContentParent.removeAllViews();
	}

	if (hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
	    final Scene newScene = Scene.getSceneForLayout(mContentParent, layoutResID, getContext());
	    transitionTo(newScene);
	} else {
	    mLayoutInflater.inflate(layoutResID, mContentParent);
	}
}

    private void installDecor() {
        mForceDecorInstall = false;
        if (mDecor == null) {
            mDecor = generateDecor(-1);
            mDecor.setDescendantFocusability(ViewGroup.FOCUS_AFTER_DESCENDANTS);
            mDecor.setIsRootNamespace(true);
            if (!mInvalidatePanelMenuPosted && mInvalidatePanelMenuFeatures != 0) {
                mDecor.postOnAnimation(mInvalidatePanelMenuRunnable);
            }
        } else {
            mDecor.setWindow(this);
        }
        if (mContentParent == null) {
            mContentParent = generateLayout(mDecor);

            // Set up decor part of UI to ignore fitsSystemWindows if appropriate.
            mDecor.makeOptionalFitsSystemWindows();

            final DecorContentParent decorContentParent = (DecorContentParent) mDecor.findViewById(
                    R.id.decor_content_parent);

            if (decorContentParent != null) {
                mDecorContentParent = decorContentParent;
                mDecorContentParent.setWindowCallback(getCallback());
                if (mDecorContentParent.getTitle() == null) {
                    mDecorContentParent.setWindowTitle(mTitle);
                }

                final int localFeatures = getLocalFeatures();
                for (int i = 0; i < FEATURE_MAX; i++) {
                    if ((localFeatures & (1 << i)) != 0) {
                        mDecorContentParent.initFeature(i);
                    }
                }

                mDecorContentParent.setUiOptions(mUiOptions);

                if ((mResourcesSetFlags & FLAG_RESOURCE_SET_ICON) != 0 ||
                        (mIconRes != 0 && !mDecorContentParent.hasIcon())) {
                    mDecorContentParent.setIcon(mIconRes);
                } else if ((mResourcesSetFlags & FLAG_RESOURCE_SET_ICON) == 0 &&
                        mIconRes == 0 && !mDecorContentParent.hasIcon()) {
                    mDecorContentParent.setIcon(
                            getContext().getPackageManager().getDefaultActivityIcon());
                    mResourcesSetFlags |= FLAG_RESOURCE_SET_ICON_FALLBACK;
                }
                if ((mResourcesSetFlags & FLAG_RESOURCE_SET_LOGO) != 0 ||
                        (mLogoRes != 0 && !mDecorContentParent.hasLogo())) {
                    mDecorContentParent.setLogo(mLogoRes);
                }

                // Invalidate if the panel menu hasn't been created before this.
                // Panel menu invalidation is deferred avoiding application onCreateOptionsMenu
                // being called in the middle of onCreate or similar.
                // A pending invalidation will typically be resolved before the posted message
                // would run normally in order to satisfy instance state restoration.
                PanelFeatureState st = getPanelState(FEATURE_OPTIONS_PANEL, false);
                if (!isDestroyed() && (st == null || st.menu == null) && !mIsStartingWindow) {
                    invalidatePanelMenu(FEATURE_ACTION_BAR);
                }
            } else {
                mTitleView = findViewById(R.id.title);
                if (mTitleView != null) {
                    if ((getLocalFeatures() & (1 << FEATURE_NO_TITLE)) != 0) {
                        final View titleContainer = findViewById(R.id.title_container);
                        if (titleContainer != null) {
                            titleContainer.setVisibility(View.GONE);
                        } else {
                            mTitleView.setVisibility(View.GONE);
                        }
                        mContentParent.setForeground(null);
                    } else {
                        mTitleView.setText(mTitle);
                    }
                }
            }

            if (mDecor.getBackground() == null && mBackgroundFallbackResource != 0) {
                mDecor.setBackgroundFallback(mBackgroundFallbackResource);
            }

            // Only inflate or create a new TransitionManager if the caller hasn't
            // already set a custom one.
            if (hasFeature(FEATURE_ACTIVITY_TRANSITIONS)) {
                if (mTransitionManager == null) {
                    final int transitionRes = getWindowStyle().getResourceId(
                            R.styleable.Window_windowContentTransitionManager,
                            0);
                    if (transitionRes != 0) {
                        final TransitionInflater inflater = TransitionInflater.from(getContext());
                        mTransitionManager = inflater.inflateTransitionManager(transitionRes,
                                mContentParent);
                    } else {
                        mTransitionManager = new TransitionManager();
                    }
                }

                mEnterTransition = getTransition(mEnterTransition, null,
                        R.styleable.Window_windowEnterTransition);
                mReturnTransition = getTransition(mReturnTransition, USE_DEFAULT_TRANSITION,
                        R.styleable.Window_windowReturnTransition);
                mExitTransition = getTransition(mExitTransition, null,
                        R.styleable.Window_windowExitTransition);
                mReenterTransition = getTransition(mReenterTransition, USE_DEFAULT_TRANSITION,
                        R.styleable.Window_windowReenterTransition);
                mSharedElementEnterTransition = getTransition(mSharedElementEnterTransition, null,
                        R.styleable.Window_windowSharedElementEnterTransition);
                mSharedElementReturnTransition = getTransition(mSharedElementReturnTransition,
                        USE_DEFAULT_TRANSITION,
                        R.styleable.Window_windowSharedElementReturnTransition);
                mSharedElementExitTransition = getTransition(mSharedElementExitTransition, null,
                        R.styleable.Window_windowSharedElementExitTransition);
                mSharedElementReenterTransition = getTransition(mSharedElementReenterTransition,
                        USE_DEFAULT_TRANSITION,
                        R.styleable.Window_windowSharedElementReenterTransition);
                if (mAllowEnterTransitionOverlap == null) {
                    mAllowEnterTransitionOverlap = getWindowStyle().getBoolean(
                            R.styleable.Window_windowAllowEnterTransitionOverlap, true);
                }
                if (mAllowReturnTransitionOverlap == null) {
                    mAllowReturnTransitionOverlap = getWindowStyle().getBoolean(
                            R.styleable.Window_windowAllowReturnTransitionOverlap, true);
                }
                if (mBackgroundFadeDurationMillis < 0) {
                    mBackgroundFadeDurationMillis = getWindowStyle().getInteger(
                            R.styleable.Window_windowTransitionBackgroundFadeDuration,
                            DEFAULT_BACKGROUND_FADE_DURATION_MS);
                }
                if (mSharedElementsUseOverlay == null) {
                    mSharedElementsUseOverlay = getWindowStyle().getBoolean(
                            R.styleable.Window_windowSharedElementsUseOverlay, true);
                }
            }
        }
    }

```


##### 3. 接下来开始执行performResumeActivity
>执行完performResumeActivity后，会继续在handleResumeActivity方法中调用activity的makeVisible方法
```java
//Activity.java
void makeVisible() {
	if (!mWindowAdded) {
		//追踪getWM最终看到得到的是一个
		ViewManager wm = getWindowManager();
		//往当前窗口添加DecorView
		wm.addView(mDecor, getWindow().getAttributes());
		mWindowAdded = true;
	}
	//显示decorView，也就是contentView也显示了
	mDecor.setVisibility(View.VISIBLE);
}
```
>所以这个时候contentView才是显示状态，也就是在onResume过程中才显示，才可交互。

##### 4. 看下addView方法
```java
//WindowManagerImpl.java
public void addView(View view, ViewGroup.LayoutParams params) {
	mGlobal.addView(view, params, mContext.getDisplay(), mParentWindow);
}
```

###### 构造一个ViewRootImpl，并把root、decor、window建立关联
```java
//WindowManagerGlobal.java
//参数view就是传过来的DecorView
public void addView(View view, ViewGroup.LayoutParams params, Display display, Window parentWindow) {
	final WindowManager.LayoutParams wparams = (WindowManager.LayoutParams) params;
	ViewRootImpl root;
	synchronized (mLock) {
		//构造一个ViewRootImpl实例
		root = new ViewRootImpl(view.getContext(), display);
		view.setLayoutParams(wparams);
		mViews.add(view);
		mRoots.add(root);
		mParams.add(wparams);
		//ViewRootImpl实例与DecorView实例建立关联，下面会用到这个view
		root.setView(view, wparams, panelParentView);
	}
}
```
>WindowManagerGlobal是什么？就像是一个操作Window的工具类，里边提供了获取WMS、WindowSession、根View的方法。没有继承任何类，可以当成一个桥接类。

##### 5. 在ViewRootImpl的setView方法中会构造一个WindowInputEventReceiver实例，用于接收input事件
事件输入涉及InputManagerService、WindowManagerService这些类，下边这个receiver需要注册到IMS或者WMS中，才能接收到事件。
```java
mInputEventReceiver = new WindowInputEventReceiver(mInputChannel, Looper.myLooper());
```
>WindowInputEventReceiver是ViewRootImpl的内部类，从类名可以看出它就是接收输入事件的。

```java
final class WindowInputEventReceiver extends InputEventReceiver {
	//当有事件输入时，父类会回调给onInputEvent方法
	public void onInputEvent(InputEvent event){
		enqueueInputEvent(event, this, 0, true);
	}
	public void onBatchedInputEventPending()...
	public void dispose()...
}
```
>其父类InputEventReceiver有一个dispatchInputEvent方法用于分发接收到的事件

```java
//InputEventReceiver.java
private void dispatchInputEvent(int seq, InputEvent event) {
	mSeqMap.put(event.getSequenceNumber(), seq);
	//会继续调用子类也就是上边WindowInputEventReceiver的onInputEvent方法
	onInputEvent(event);
}
```

```java
void enqueueInputEvent(InputEvent event, InputEventReceiver receiver, int flags, boolean processImmediately) {
	//从这里可以看出：Input事件也是一个一个的消息，是一个链表结构，一个一个发
	...
	if (processImmediately) {
		//立即发消息
		doProcessInputEvents();
	} else {
		//应该是延迟发消息（内部会调用handler在“合适的时间发送”）
		scheduleProcessInputEvents();
	}
}
```

```java
void doProcessInputEvents() {
	// Deliver all pending input events in the queue.
	//循环发出队列中的全部消息
	while (mPendingInputEventHead != null) {
		QueuedInputEvent q = mPendingInputEventHead;
		mPendingInputEventHead = q.mNext;
		//通过这个方法分发事件
		deliverInputEvent(q);
	}
}
```

```java
//交付（有点分发的意思）事件
private void deliverInputEvent(QueuedInputEvent q) {
	InputStage stage;
	if (q.shouldSendToSynthesizer()) {
		stage = mSyntheticInputStage;
	} else {
		stage = q.shouldSkipIme() ? mFirstPostImeInputStage : mFirstInputStage;
	}
	stage.deliver(q);
}
```

```java
public final void deliver(QueuedInputEvent q) {
	//注意看这里有个onProcess方法，它是InputStage中的方法，但是InputStage是个抽象类，所以这里可能是调其子类的onProcess方法
	apply(q, onProcess(q));
}
```

>InputStage的实现类：
* ViewPreImeInputStage
* EarlyPostImeInputStage
* ViewPostImeInputStage
* SyntheticInputStage
**我们的触摸事件走的应该是ViewPostImeInputStage这个**

```java
protected int onProcess(QueuedInputEvent q) {
	if (q.mEvent instanceof KeyEvent) {
		//输入事件（如键盘）
		return processKeyEvent(q);
	} else {
		//手指触摸事件
		return processPointerEvent(q);
	}
}
```

##### 6. 最终会调用decorView的dispatchPointerEvent分发
```java
//ViewRootImpl.java
private int processPointerEvent(QueuedInputEvent q) {
	final MotionEvent event = (MotionEvent)q.mEvent;
	//这个mView就是上边ViewRootImpl.setView设置的
	boolean handled = mView.dispatchPointerEvent(event);
	return handled ? FINISH_HANDLED : FORWARD;
}
```

##### 7. dispatchPointerEvent又会调用自己的dispatchTouchEvent，也就是decorview的dispatchTouchEvent
```java
//View.java
public final boolean dispatchPointerEvent(MotionEvent event) {
	if (event.isTouchEvent()) {
		//执行DecorView的dispatchTouchEvent方法
		return dispatchTouchEvent(event);
	}
}
```

##### 8. 终于，在decorView的dispatch中拿到上边构造window时传的Activity实例回调其dispatch
```java
public boolean dispatchTouchEvent(MotionEvent ev) {
	//这个cb就是Activity实例
	final Window.Callback cb = mWindow.getCallback();
	//调用activity的dispatchTouchEvent方法
	return cb != null && !mWindow.isDestroyed() && mFeatureId < 0? cb.dispatchTouchEvent(ev) : super.dispatchTouchEvent(ev);
}
```























