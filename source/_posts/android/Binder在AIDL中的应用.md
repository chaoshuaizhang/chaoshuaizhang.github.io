---
title: Binder在AIDL中的应用
date: 2018-06-03 17:36:04
tags: [aidl,binder,进程间通信,服务绑定]
categories: android
---

# 先看一张图
>这张图描述了AIDL进程间通信的大致流程。
**同时它也是服务绑定的过程，只不过涉及的一些类：ApplicationThread、IServiceConnection.aidl、InnerConnection、Handler等没有列出来。**

![Alt text](./1564889071567.png)

# 服务端

## 创建服务时构造一个binder
public class BookManageService extends Service {
	
	//在服务端创建一个binder对象，当客户端绑定时，返回这个binder给客户端
	private IBinder mBinder = new IBookManager.Stub() {
        @Override
        public boolean onTransact(int code, Parcel data, Parcel reply, int flags) throws RemoteException {
            return super.onTransact(code, data, reply, flags);
        }
		//具体的实现类
        @Override
        public List<Book> getBookList() throws RemoteException {
            return list;
        }
        @Override
        public void addBook(Book book) throws RemoteException {
            list.add(book);
        }
    };
}

### new Stub()时做了什么
>new Stub时把DESCRIPTOR和具体的实现类做了关联，便于后期找到
```java
public Stub() {
	//这个this就是当前的匿名内部类，后期根据这个DESCRIPTOR查找对应的IBookManager来操作相关方法
	this.attachInterface(this, DESCRIPTOR);
}

//调用父类的attachInterface方法
public void attachInterface(IInterface owner, String descriptor) {
	//把当前的Stub-Binder的具体实现类赋值给mOwner
	mOwner = owner;
	mDescriptor = descriptor;
}
```

### 把服务端构造的Binder返给客户端
```java
public IBinder onBind(Intent intent) {
	//把服务端的Binder对象传给client端
	return mBinder;
}
```

# 客户端

## 绑定服务
```java
bindService(intent, connection, Context.BIND_AUTO_CREATE);
```

```java
private ServiceConnection connection = new ServiceConnection() {
    @Override
    public void onServiceConnected(ComponentName name, IBinder service) {
		//【注意】相同进程时返回的service是服务端创建的那个binder，不同进城时返回的是服务端binder的代理binder-BiderProxy
		//客户端要想和服务端通信，需要把服务端的binder构造成一个服务端的操作实例
		//最终返回一个服务端操作实例的代理类，用代理类来调服务端相应的方法
        bookManager = IBookManager.Stub.asInterface(service);
    }
};
```
>* 客户端调用bookManager的方法时实际调用的是IBookManager.Stub的内部类Proxy相应的方法。
* activity发起绑定服务后，会经过一大圈最终回调到onServiceConnected，返回相应的binder，其中涉及到跨进程，handler消息机制等。

### Stub类的asInterface方法

```java
public static net.shopin.myaidlserver.IBookManager asInterface(android.os.IBinder obj) {
	//String DESCRIPTOR = "net.shopin.myaidlserver.IBookManager"
	//因为是跨进程，所以这里实际执行的是BinderProxy的queryLocalInterface方法
	//所以因为是跨进程，这里返回的一定是null
	android.os.IInterface iin = obj.queryLocalInterface(DESCRIPTOR);
	if (((iin != null) && (iin instanceof net.shopin.myaidlserver.IBookManager))) {
		return ((net.shopin.myaidlserver.IBookManager) iin);
	}
	//不同进程的话，返回一个IBookManager的代理类，并传入服务端相应的binder实例
	return new net.shopin.myaidlserver.IBookManager.Stub.Proxy(obj);
}
```

#### BinderProxy的queryLocalInterface方法
```java
public IInterface queryLocalInterface(String descriptor) {
	//内部没有实现，默认返回null
	return null;
}
```

#### Binder的queryLocalInterface方法
```java
public IInterface queryLocalInterface(String descriptor) {
	if (mDescriptor.equals(descriptor)) {
		//直接返回mOwner，mOwner在上边已经说了
		return mOwner;
	}
	return null;
}
```

#### Proxy代理类
```java
private static class Proxy implements net.shopin.myaidlserver.IBookManager {
	private android.os.IBinder mRemote;

	Proxy(android.os.IBinder remote) {
		//服务端返回的binder实例或者是BinderProxy代理类
		mRemote = remote;
	}
	
	//根据mRemote来调用相应的方法
	getBookList(){
		mRemote.transact(Stub.TRANSACTION_getBookList, _data, _reply, 0);
	}
	addBook(){
		mRemote.transact(Stub.TRANSACTION_addBook, _data, _reply, 0);
	}
}
```
>客户端同样会自动生成一个Stub类，但是客户端不会用这个Stub实例，只会用Stub的静态内部类Proxy，通过这个Proxy对应到服务端的Stub类。

>**一个服务对应的有一个binder实例，服务端曝露出这个binder实例（或者是此binder的代理实例）给客户端，客户端根据这个binder实例生成相应的服务代理类，在服务代理类中通过服务端暴漏的那个binder或代理类调用目标方法，此时这个通信过程会交由内核层处理也就是/dev/binder来桥接，然后再返回到应用层，交由真正的服务端对应的方法处理。**
一次通信完成。











