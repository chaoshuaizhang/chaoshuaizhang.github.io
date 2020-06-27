---
title: Launcher是如何启动Activity的
date: 2018-07-17 14:28:28
tags: activity
categories: android
---

### Launcher的启动
>大致分下面4步

1. Android系统启动的最后一步就是启动Launcher
2. Launcher是由AMS启动的，启动后会通过PMS获取到所有的已安装应用信息
3. Launcher的自动，AMS的startHomeActivityLocked方法
```java
boolean startHomeActivityLocked(int userId, String reason) {
	//通过这个方法启动launcher
	return true;
}

// 启动时intent设置的参数和manifest文件中的一致
<action android:name="android.intent.action.MAIN" />
<category android:name="android.intent.category.HOME" />
```
4. Launcher应用就启动起来了，执行onCreate、onStart...等方法

### Launcher加载其它应用图标到桌面
>大致分下面3步

1. 开启一个线程，通过handler发送执行体
```java
public void startLoader(boolean isLaunching, int synchronousBindPage) {
	synchronized (mLock) {
		mDeferredBindRunnables.clear();
		if (mCallbacks != null && mCallbacks.get() != null) {
			isLaunching = isLaunching || stopLoaderLocked();
			mLoaderTask = new LoaderTask(mApp, isLaunching);
			if (synchronousBindPage > -1 && mAllAppsLoaded && mWorkspaceLoaded) {
				mLoaderTask.runBindSynchronousPage(synchronousBindPage);
			} else {
				sWorkerThread.setPriority(Thread.NORM_PRIORITY);
				//这个sWorker是个handler对象，也就是说 mLoadTask是一个Runnable类型的对象
				sWorker.post(mLoaderTask);
			}
		}
	}
}

private static final HandlerThread sWorkerThread = new HandlerThread("launcher-loader");
//注意：这是一个静态代码块，一进入LauncherModel开启
static {
	//线程开启-开启一个looper
	sWorkerThread.start();
}
//依赖上述创建的looper为创建一个handler
private static final Handler sWorker = new Handler(sWorkerThread.getLooper());
```	

2. 在执行体内部加载所有已安装的应用信息
#### mLoadTask运行

```java
public void run() {
	keep_running: {
		loadAndBindWorkspace();
		//加载并绑定所有的app
		//内部使用的还是handler，回调处理需要在launcher显示的app列表
		loadAndBindAllApps();
	}
	// Update the saved icons if necessary
	//更新桌面的图标(如果需要更新的话)
	updateSavedIcon(mContext, (ShortcutInfo) key, sBgDbIconCache.get(key));
}
```

3. 加载上述的图标（快捷方式），显示在recyclerview上

	
	
	
	
	
	