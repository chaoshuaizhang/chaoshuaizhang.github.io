---
title: Binder在启动Activity过程中的应用
date: 2019-01-03 17:35:50
tags: [activity,binder]
categories: android
---

# 启动Activity
```java
ActivityManagerNative.getDefault().startActivity(...);
```

## getDefault()得到的是什么

```java
static public IActivityManager getDefault() {
	//由下边的分析可以知道：get最终返回的是ActivityManagerProxy实例（跨进程时）或者IActivityManager的其它实现类（相同进程时）
	return gDefault.get();
}
```

### gDefault的创建
>gDefault.get最终会返回一个IActivityManager的具体实现类。

* 【注意】以下分为两步：
1. 第一步：获取一个关于Activity的binder
2. 第二步：根据这个binder实例化成一个具体的服务
>这个和AIDL类似，拿到binder，生成实例，调用相应方法。

```java
//ActivityManagerNative.java
private static final Singleton<IActivityManager> gDefault = new Singleton<IActivityManager>() {
	protected IActivityManager create() {
		//【第一步】
		IBinder b = ServiceManager.getService("activity");
		//【第二步】
		//asInterface最终返回ActivityManagerProxy实例
		IActivityManager am = asInterface(b);
		return am;
	}
};
//gDefault的get方法
public final T get() {
	return mInstance == null ? create() : mInstance;
}
```

#### 第一步：获取和Activity相关的那个binder

##### ServiceManager.getService("Activity")
```java
//ServiceManager的getService方法
public static IBinder getService(String name) {
	IBinder service = sCache.get(name);
	if (service != null) {
		return service;
	} else {
		//getIServiceManager()返回的是IServiceManager的具体实现
		//getIServiceManager().getService()返回的是一个Binder对象
		return getIServiceManager().getService(name);
	}
}
```

##### getServiceManager()

```java
//获得全局的ServiceManager管理类
private static IServiceManager getIServiceManager() {
	if (sServiceManager != null) {
		return sServiceManager;
	}
	//注意这里传入BinderInternal.getContextObject()-->返回的实际是BinderProxy实例（Binder代理类，服务端暴漏出来让客户端拿着调的）
	//asInterface最终返回一个IServiceManager的具体实现类
	sServiceManager = ServiceManagerNative.asInterface(BinderInternal.getContextObject());
	return sServiceManager;
}

//是个native方法，返回整个进程间通信时Binder对应的代理类，客户端用这个代理类去调用服务端
public static final native IBinder getContextObject();
```
>

###### ServiceManagerNative.asInterface
>根据传入的binder代理类返回一个具体实例。

```java
//ServiceManagerNative.java
/**
 * Cast a Binder object into a service manager interface, generating a proxy if needed.
 */
static public IServiceManager asInterface(IBinder obj) {
	//descriptor = "android.os.IServiceManager";
	IServiceManager in = (IServiceManager)obj.queryLocalInterface(descriptor);
	if (in != null) {
		return in;
	}
	return new ServiceManagerProxy(obj);
}
```

##### getIServiceManager().getService(name)
>这个ServiceManagerProxy就是服务管理的代理类，根据name拿到对应的binder服务。

```java
//ServiceManagerProxy.java
public ServiceManagerProxy(IBinder remote) {
	mRemote = remote;
}

public IBinder getService(String name) throws RemoteException {
	...
	data.writeString(name);
	//这里的mRemote是上述的binder代理类BinderProxy，调用底层，交由IServiceManager.cpp处理
	mRemote.transact(GET_SERVICE_TRANSACTION, data, reply, 0);
	IBinder binder = reply.readStrongBinder();
	...
	//返回"activity"对应的binder实例
	return binder;
}
```

>至此，第一步完成，activity相关的binder已经得到。

#### 第二步：根据上述binder实例化成一个具体的服务
>这个服务就是控制Activity的服务。

##### asInterface根据binder得到具体服务
```java
//ActivityManagerNative.java
/**
 * Cast a Binder object into an activity manager interface, generating a proxy if needed.
 */
static public IActivityManager asInterface(IBinder obj) {
	//descriptor = "android.app.IActivityManager"
	//根据activity对应的binder获取activity对应的管理实例
	IActivityManager in = (IActivityManager)obj.queryLocalInterface(descriptor);
	if (in != null) {
		return in;
	}
	//根据activity对应的binder实例，返回最终的服务，之后通过这个服务调用startActivity方法
	return new ActivityManagerProxy(obj);
}
```

###### ActivityManagerProxy
>ActivityManagerProxy是ActivityManagerNative的内部类。
```java
class ActivityManagerProxy implements IActivityManager {
    public ActivityManagerProxy(IBinder remote) {
		//第一步获取的binder
        mRemote = remote;
    }
	public int startActivity(){
		//这个mRemote就是对应activity的那个binder（也就是其暴露出来让客户端使用的那个binder）
		mRemote.transact(START_ACTIVITY_TRANSACTION, data, reply, 0);
	}
}
```

###### 又回到了ActivityManagerNative
>最终通过跨进程通信，把启动activity的消息发给了管理Activity的服务。
```java
public boolean onTransact(int code, Parcel data, Parcel reply, int flags) throws RemoteException {
	switch (code) {
		case START_ACTIVITY_TRANSACTION:
			//startActivity最终调到了AMS中
			int result = startActivity(app, callingPackage, intent, resolvedType, resultTo, resultWho, requestCode, startFlags, profilerInfo, options);
	}
}
```

>**一个服务对应的有一个binder实例，服务端曝露出这个binder实例给客户端，客户端根据这个binder实例生成相应的服务代理类，在服务代理类中通过服务端暴漏的那个binder调用目标方法，此时这个通信过程会交由内核层处理也就是/dev/binder来桥接，然后再返回到应用层，交由真正的服务端对应的方法处理。**
一次通信完成。




