---
title: 关于AMS的创建、启动
date: 2018-03-18 11:17:31
tags: ams
categories: android
---

# AMS介绍
>AMS是android中最核心的服务，主要负责四大组件的启动、切换、调度以及应用进程的管理和调度，其职责和操作系统中的进程管理和调度模块相类似。


## AMS的启动流程
>由SystemServer的启动流程看以看出：AMS是在启动SystemServer的过程中启动的。

* 在SystemServer的run方法中开启了一些服务：引导服务、核心服务等，其中在引导服务中就启动了ActivityManagerService

```java
private void startBootstrapServices() {
	//开启AMS服务（Lifecycle内部创建了AMS实例）
	mActivityManagerService = mSystemServiceManager.startService(ActivityManagerService.Lifecycle.class).getService();
	mActivityManagerService.setSystemServiceManager(mSystemServiceManager);
	mActivityManagerService.setInstaller(installer);
	mPowerManagerService = mSystemServiceManager.startService(PowerManagerService.class);
	//开启电源管理
	mActivityManagerService.initPowerManagement();
	//亮度服务
	mSystemServiceManager.startService(LightsService.class);
	//开启管理显示的服务
	mDisplayManagerService = mSystemServiceManager.startService(DisplayManagerService.class);
	//包管理服务
	mPackageManagerService = PackageManagerService.main(mSystemContext, installer, mFactoryTestMode != FactoryTest.FACTORY_TEST_OFF, mOnlyCore);
	mSystemServiceManager.startService(UserManagerService.LifeCycle.class);
	//把AMS设置成系统进程并启动
	mActivityManagerService.setSystemProcess();
}
```
>可以看出，AMS的启动是在bootStrap（引导服务）中启动的，通过AMS又启动了一些其它的服务：电源、亮度、屏幕管理等服务，最后把AMS设置成系统进程。

* 看下AMS的Lifecycle类，这里最终创建了AMS实例，并开启了AMS服务
```java
public static final class Lifecycle extends SystemService {
	private final ActivityManagerService mService;
	public Lifecycle(Context context) {
		super(context);
		//创建一个AMS实例
		mService = new ActivityManagerService(context);
	}
	@Override
	public void onStart() {
		//服务开启
		mService.start();
	}
	public ActivityManagerService getService() {
		return mService;
	}
}
```

* AMS的start方法，内部跑了一个线程，和CPU执行状态有关，定时去更新CPU状态
```java
private void start() {
	Process.removeAllProcessGroups();
	mProcessCpuThread.start();
	mBatteryStatsService.publish(mContext);
	mAppOpsService.publish(mContext);
	Slog.d("AppOps", "AppOpsService published");
	LocalServices.addService(ActivityManagerInternal.class, new LocalService());
}

mProcessCpuThread = new Thread("CpuTracker") {
	@Override
	public void run() {
		while (true) {
			synchronized(this) {
				final long now = SystemClock.uptimeMillis();
				long nextCpuDelay = (mLastCpuTime.get()+MONITOR_CPU_MAX_TIME)-now;
				long nextWriteDelay = (mLastWriteTime+BATTERY_STATS_TIME)-now;
				if (nextWriteDelay < nextCpuDelay) {
					nextCpuDelay = nextWriteDelay;
				}
				if (nextCpuDelay > 0) {
					mProcessCpuMutexFree.set(true);
					this.wait(nextCpuDelay);
				}
			}
			//更改CPU状态
			updateCpuStatsNow();
		}
	}
}
```

## AMS与其它进程
>Zygote启动时会去监听一个socket端口，等待AMS来请求创建新的进程。要启动一个
应用程序，则该应用必须得有一个进程，如果要启动的应用进程不存在则AMS就会请
求Zygote进程将需要的应用进程启动。

## ActivityManager
>感觉它就像一个桥梁，处于应用程序与AMS的中间，AM内部调用AMS的方法对Activity进行管理，比如：在启动新的Activity时会在Instrumentation中通过AM`AM.getService().startActivityXXX`去启动新的Activity

* 调用AM的getService方法获得AMS实例
```java
//Instrumatation.java
int result = ActivityManager.getService()
	.startActivity(whoThread, who.getBasePackageName(), intent,
	intent.resolveTypeIfNeeded(who.getContentResolver()),
	token, target != null ? target.mEmbeddedID : null,
	requestCode, 0, null, options);
```

拿到实例
```java
public static IActivityManager getService() {
	return IActivityManagerSingleton.get();
}
```

* 最后走到AMS的startActivity方法中，按照正常启动Activity的流程进行启动。

#### 参考
[理解AMS](https://blog.csdn.net/innost/article/details/47254381)



















