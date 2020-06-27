---
title: Activity的setContentView加载机制
date: 2018-07-28 16:13:44
tags: [绘制流程,Activity显示流程]
categories: android
---

>**本篇主要说一下View的显示流程。就要就是setContentView方法。**
setContentView是通过LayoutInflater把传入的资源id转换为一个View并添加到根布局中，所以我们先分析LayoutInflater的执行机制。

### LayoutInflater用法
```java
//构造一个inflate对象
LayoutInflater inflater = LayoutInflater.from(this);
//根据布局等参数信息构造一个view
View mainView = inflater.inflate(R.layout.activity_main, null);
```

#### from的实现
```java
public static LayoutInflater from(Context context) {
	//可以看出来form真正的实现也是context.getSystemService
	LayoutInflater LayoutInflater = (LayoutInflater) context.getSystemService(Context.LAYOUT_INFLATER_SERVICE);
	if (LayoutInflater == null) {
		throw new AssertionError("LayoutInflater not found.");
	}
	return LayoutInflater;
}
```

#### inflate方法的三个参数

```java
/**
* resource：资源文件，最终经过一系列操作，转换为View
* root：父布局，如果attachToRoot是true的话，会把上述的view加在root中
* attachToRoot：就是是否添加到root中成为其子view
*/
public View inflate(@LayoutRes int resource, @Nullable ViewGroup root, boolean attachToRoot) {
    final Resources res = getContext().getResources();
    //根据资源文件生成一个xml解析器（这个解析器已经包含了资源文件所有的内容了）
    final XmlResourceParser parser = res.getLayout(resource);
    try {
        return inflate(parser, root, attachToRoot);
    } finally {
        parser.close();
    }
}
```

#### inflate的执行流程
>inflate的执行流程分为几个步骤：
>1. 把资源文件实例化成为一个xml资源解析器。
>2. 根据资源解析器构造一个View。
>3. 判断view是否设置params，是否添加到root中。
>4. 返回view
```java
public View inflate(XmlPullParser parser, @Nullable ViewGroup root, boolean attachToRoot) {
	synchronized (mConstructorArgs) {
		final Context inflaterContext = mContext;
		//从解析器中取出属性集合
		final AttributeSet attrs = Xml.asAttributeSet(parser);
		Context lastContext = (Context) mConstructorArgs[0];
		mConstructorArgs[0] = inflaterContext;
		View result = root;
		try {
			// Look for the root node.
			int type;
			//查找资源文件的根节点（即start节点）
			while ((type = parser.next()) != XmlPullParser.START_TAG && type != XmlPullParser.END_DOCUMENT) {
				// Empty
			}
			if (type != XmlPullParser.START_TAG) {
				throw new InflateException(parser.getPositionDescription() + ": No start tag found!");
			}
			final String name = parser.getName();
			//merge标签的处理
			if (TAG_MERGE.equals(name)) {
				if (root == null || !attachToRoot) {
					throw new InflateException("<merge /> can be used only with a valid "
							+ "ViewGroup root and attachToRoot=true");
				}
				rInflate(parser, root, inflaterContext, attrs, false);
			} else {
				//根据root(parent)等一些布局参数生成一个新的view（资源文件id最终也是肯定要转为view的）
				final View temp = createViewFromTag(root, name, inflaterContext, attrs);
				/**
				* 注意看LayoutParams的设置：
				* 如果attact是true，说明需要附加在父view中
				* 如果是false：说明不附加。如果root不为null，则使新view的params设置为root的params
				* 否则就不设置params
				*/
				ViewGroup.LayoutParams params = null;
				if (root != null) {
					// Create layout params that match root, if supplied
					params = root.generateLayoutParams(attrs);
					if (!attachToRoot) {
						temp.setLayoutParams(params);
					}
				}
				// Inflate all children under temp against its context.
				rInflateChildren(parser, temp, attrs, true);
				//可以看到当attach=true时，做的操作是把生成的view塞到root中
				if (root != null && attachToRoot) {
					root.addView(temp, params);
				}
				//当root是null或者attach=false时，直接返回生成的temp
				if (root == null || !attachToRoot) {
					result = temp;
				}
			}
		} catch (XmlPullParserException e) {
		}
		//返回结果：要么是添加了view的root，要么是view本身
		return result;
	}
}
```
>OK，inflate过程结束，inflate过程中对merge标签做了处理---merge不能单独使用，必须有接纳它的容器。

### setContentView
>我们这里只分析set资源id的情况，它和setContentView(view...)的区别在于它需要使用layoutInflater把id变为View实例。

>流程：
>第一步：执行setContentView
>第二步：设置decorView
>第三步：设置主页面内容，从（mContentParent）
>>设置主布局内容是挺复杂的，根据我们设置的各个属性去配置我们需要的布局

#### 布局层级
>可能不是很清晰，但是可以看个大概

![Alt text](./1552469205539.png)

#### setContentView(id)

```java
@Override
public void setContentView(int layoutResID) {
	if (mContentParent == null) {
		//安装DecorView
		installDecor();
	} else if (!hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
		mContentParent.removeAllViews();
	}
	if (hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
		final Scene newScene = Scene.getSceneForLayout(mContentParent, layoutResID,
				getContext());
		transitionTo(newScene);
	} else {
		//对于这块儿，为什么没有拿返回值呢？
		//从上边的inflate的源码可以看出：当mContentParent不为空时，attach会
		//设置为true，然后会把对应的布局文件添加到mContentParent中，
		//所以这里不需要拿到view
		mLayoutInflater.inflate(layoutResID, mContentParent);
	}
	mContentParent.requestApplyInsets();
	final Callback cb = getCallback();
	if (cb != null && !isDestroyed()) {
		cb.onContentChanged();
	}
}
```
>`mLayoutInflater.inflate(layoutResID, mContentParent)`由上面inflate的分析知道，最终生成了一个view并加载到了mContentParent中

### 下面对DecorView、ContentParent、resId的联系进行分析

#### installDecor()
```java
private void installDecor() {
	if (mDecor == null) {
		//此处就是简单生成一个DecorView(context)，由事
		//件分发机制可以知道这个Decor是主Activity的。
		mDecor = generateDecor();
	}
	if (mContentParent == null) {
		//得到主内容布局
		mContentParent = generateLayout(mDecor);
	}
}
```

#### generateLayout(decorview)
>根据一些参数设置不同的布局风格，并添加到DecorView中，但是DecorView的层级有很多，这块儿只是在Decor中加了一个View

```java
protected ViewGroup generateLayout(DecorView decor) {
	// Apply data from current theme.
	//得到一些主题资源信息，下边开始设置
	TypedArray a = getWindowStyle();
	final Context context = getContext();
	// Inflate the window decor.
	int layoutResource;
	//获得一些给Activity设置的特性
	int features = getLocalFeatures();
	//根据这些特性生成设置对应的layoutResource
	mDecor.startChanging();
	View in = mLayoutInflater.inflate(layoutResource, null);
	//把上述的view添加到decor中
	decor.addView(in, new ViewGroup.LayoutParams(MATCH_PARENT, MATCH_PARENT));
	//设置根布局（根布局是DecorView的子View）
	mContentRoot = (ViewGroup) in;
	//得到跟布局中的“主内容布局---我们添加的所有布局都会在这里”
	//下面这个findViewById通过看源码发现：是在decorview的布局文件中找的，也就是
	//从上述的mContentRoot中找的。（下边看这个详细的代码实现）
	ViewGroup contentParent = (ViewGroup)findViewById(ID_ANDROID_CONTENT);
	// Remaining setup -- of background and title -- that only applies
	// to top-level windows.
	if (getContainer() == null) {
		//设置DecorView的一些属性
		mDecor.setXXX();
	}
	mDecor.finishChanging();
	//返回decorview中的content---即framelayout
	return contentParent;
}
```
##### 可以发现，contentParent是从DecorView中根据id=content获得的
```java
public <T extends View> T findViewById(@IdRes int id) {
	// id = ID_ANDROID_CONTENT = com.android.internal.R.id.content
    return getDecorView().findViewById(id);
}
```

