---
title: Binder机制分析
date: 2018-04-02 14:29:37
tags: [binder,aidl]
categories: android
---

## 结合AIDL分析Binder机制

### AIDL使用流程

>1. aidl文件--->直接用as生成aidl文件就行，他会默认生成一个aidl文件夹，创建Book.java在java下的包下，而Book.aidl在aidl下的包下，注意这两个包名必须一致--->这点as会帮我们做。
>2. aidl demo server-d:\\myprjs\aidl\client\MyApplication
>3. aidl demo client-d:\\myprjs\aidl\server\MyAidlServer

#### 服务端Service

##### Service
```java
public class BookManageService extends Service {
    private static final String TAG = "CLIENT_ERROR_LOG";
    private CopyOnWriteArrayList<Book> list = new CopyOnWriteArrayList<>();
    
    //下面分析mBinder
    private IBinder mBinder = new IBookManager.Stub() {};
    @Override
    public void onCreate() {
        super.onCreate();
		//service在create时先添加一本书
        list.add(new Book(1001, "谁许传"));
    }
    /**
     * 在客户端与服务端建立连接时传给客户端
     */
    @Override
    public IBinder onBind(Intent intent) {
		//把服务端的Binder对象传给client端
        return mBinder;
    }
}
```
##### mBinder对象
>我们在.aidl中定义的那些操作，真正的执行体在这里
```java
/**
 * 创建这个Binder就是为了暴露给客户端
 * 客户端拿到server端的binder执行add，get方法时，经过一系列转换，最后走的就是这里。
 * IBookManager.aidl的这4个方法都执行在binder线程中
 * 注意mBinder是如何生成的：mBinder指向了一个匿名内部类（注意这个匿名内部类，下边的分析会提到）
 */
private IBinder mBinder = new IBookManager.Stub() {
	@Override
	public boolean onTransact(int code, Parcel data, Parcel reply, int flags) throws RemoteException {
		return super.onTransact(code, data, reply, flags);
	}
	@Override
	public List<Book> getBookList() throws RemoteException {
		return list;
	}
	@Override
	public void addBook(Book book) throws RemoteException {
		//添加是在client的主线程中add的，但是这里走到了binder线程
		list.add(book);
	}
};
```

#### 客户端绑定服务端Service

##### 获得要绑定的connection
```java
//client端定义的connection
private ServiceConnection connection = new ServiceConnection() {
    @Override
    public void onServiceConnected(ComponentName name, IBinder service) {
	    //根据server端的binder对象调用AIDL中的Stub类的asInterface方法
	    //最终返回的是一个Proxy对象-实现了IBookManager接口（因为不是在同一进程，所以会用代理实现跨进程通信）
	    //相当于是本地的Proxy代理了远程的Binder，生成了一个Manager对象(即IInterface的实现类，里边有Binder)，但最终执行时肯定还是要通过被代理对象的。
	    //【注意：】上述的service对象实际是BinderProxy类型的对象
        bookManager = IBookManager.Stub.asInterface(service);
    }
    @Override
    public void onServiceDisconnected(ComponentName name) {
        Log.e(TAG, "onServiceDisconnected: " + name.toString());
    }
};
```

##### 绑定

```java
Intent intent = new Intent();
intent.setAction("net.shopin.aidlservice");
intent.setPackage("net.shopin.myaidlserver");
//与服务端建立连接
//这边有个小问题：在api>21时，上述的intent不仅要加action，还要加package
bindService(intent, connection, Context.BIND_AUTO_CREATE);
```

### 由AIDL自动生成的类

#### Server端的IBookManager接口
```java
public interface IBookManager extends android.os.IInterface {
	public static abstract class Stub extends android.os.Binder implements net.shopin.myaidlserver.IBookManager{
		//下边分析Stub类
	}
	/**
	* 下面的方法就是我们在.aidl文件中定义的。
	* 可以肯定：必须会有一个类来实现这些方法，不然我们定义的操作没有执行体！
	* 那么是谁呢？就看谁实现了IBookManager接口===>Stub类
	*/
	public java.util.List<Book> getBookList();
    public void addBook(Book book);
}
```

#### IBookManager接口中的Stub类

```java
/**
 * Local-side IPC implementation stub class.
 * 注意：Stub继承Binder，实现IBookManager，但它是一个抽象类，并且也是自动生成的-说明我们定义的那些操作的执行体肯定不在这里边-肯定在Stub的子类中-就是我们在Service中为了生成mBinder而定义的那个匿名内部类
 */
public static abstract class Stub extends android.os.Binder implements net.shopin.myaidlserver.IBookManager {
	private static final java.lang.String DESCRIPTOR = "net.shopin.myaidlserver.IBookManager";
	/**
	 * Server端的Service构造Binder对象时会走到这里生成一个Stub()实例
	 */
	public Stub() {
		/*
		注意：开始把当前的this（stub对象）和DESCRIPTOR建立关联
		所以后期：执行binder.queryLocalInterface(DESCRIPTOR)时，
		就是根据这里attach的对应关系来查找的
		*/
		this.attachInterface(this, DESCRIPTOR);
	}
	public static net.shopin.myaidlserver.IBookManager asInterface(android.os.IBinder obj) {
		//具体代码下边分析
	}
	@Override
	public boolean onTransact(int code, Parcel data, Parcel reply, int flags) throws android.os.RemoteException {
		//下边分析
	}
	private static class Proxy implements net.shopin.myaidlserver.IBookManager {
		//下边分析
	}
	static final int TRANSACTION_getBookList = (IBinder.FIRST_CALL_TRANSACTION + 0);
	static final int TRANSACTION_addBook = (IBinder.FIRST_CALL_TRANSACTION + 1);
}
```

#### Stub类中的asInterface方法

```java
/**
 * Cast an IBinder object into an net.shopin.myaidlserver.IBookManager interface, generating a proxy if needed.
 */
public static net.shopin.myaidlserver.IBookManager asInterface(android.os.IBinder obj) {
       if ((obj == null)) {
           return null;
       }
       //DESCRIPTOR就是包名+类名（string类型）
       IInterface iin = obj.queryLocalInterface(DESCRIPTOR);
       //那么问题是：什么时候if条件不成立？
       //注意queryLocalInterface方法：在本地（指的就是当前进程）查找是否有DESCRIPTOR对应的IInterface
       //在Server端时，obj.queryLocalInterface是可以找到对应的DESCRIPTOR，因为此时Binder对象所处的进程和调用者所处的是同一个进程（这里的调用者指的就是bookManager.addBook操作所处的进程），但是当把server端的service设置为remote进程时，调用者和binder对象所处的就不是同一个进程了(此时，Binder就成为一个远程对象了)！
       if (((iin != null) && (iin instanceof net.shopin.myaidlserver.IBookManager))) {
           return ((net.shopin.myaidlserver.IBookManager) iin);
       }
       //既然不在同一个进程，那如何获取到远程的(binder)对象实例来操作呢？
       //通过代理（和java的代理实现方式不一样）返回对应的IBookManager实现类（也就是Proxy类），也就是说以后操作IBookManager类的各个方法都是先操作Proxy，最后才到被代理对象了
       return new net.shopin.myaidlserver.IBookManager.Stub.Proxy(obj);
}
```

#### Stub的onTransact方法

```java
/*
* 这是Stub类中的onTransact方法，不同进程通信时才会走这里！
* 同一进程通信时（比如当前应用注册当前的Service(没有设置remote)，使用AIDL通信，直接就会走到Service中的匿名内部类中，不会经过这一系列的操作）
* */
public boolean onTransact(int code, android.os.Parcel data, android.os.Parcel reply, int flags) throws android.os.RemoteException {
	//方法中的参数就是client调用transact方法时传过来的
	switch (code) {
		case INTERFACE_TRANSACTION: {
			reply.writeString(DESCRIPTOR);
			return true;
		}
		case TRANSACTION_getBookList: {
			data.enforceInterface(DESCRIPTOR);
			List<Book> _result = this.getBookList();
			reply.writeNoException();
			reply.writeTypedList(_result);
			return true;
		}
		case TRANSACTION_addBook: {
			data.enforceInterface(DESCRIPTOR);
			net.shopin.myaidlserver.Book _arg0;
			if ((0 != data.readInt())) {
				_arg0 = net.shopin.myaidlserver.Book.CREATOR.createFromParcel(data);
			} else {
				_arg0 = null;
			}
			//注意下边的this.getBookList()，调的就是子类（那个匿名内部类）的getBookList方法
			this.addBook(_arg0);
			reply.writeNoException();
			//注意这里直接返回了，并没有再调到Binder的onTransact
			return true;
		}
	}
	return super.onTransact(code, data, reply, flags);
}
```

#### 代理类Proxy

```java
private static class Proxy implements net.shopin.myaidlserver.IBookManager {
    private android.os.IBinder mRemote;
    //这个remote是在client中构建ServiceConnection时，onServiceConnected方法传来的，实际上还是Server的Service传来的Binder对象--->也就是IBookManager.Stub()
    Proxy(android.os.IBinder remote) {
	    //当执行this.asBinder()时，会把这个binder对象返回
        mRemote = remote;
    }
    @Override
    public android.os.IBinder asBinder() {
        return mRemote;
    }
    @Override
    public void addBook(net.shopin.myaidlserver.Book book) throws android.os.RemoteException {
        android.os.Parcel _data = android.os.Parcel.obtain();
        android.os.Parcel _reply = android.os.Parcel.obtain();
        try {
	        //写入类名，类似于标记吧，只要类名一致都可以反序列化
            _data.writeInterfaceToken(DESCRIPTOR);
            if ((book != null)) {
                _data.writeInt(1);
                book.writeToParcel(_data, 0);
            } else {
                _data.writeInt(0);
            }
            //此处执行IBookManager.Stub()的transact方法
            //可以发现：此处真正执行的addBook方法，最终还是通过一个转换
            //mRemote.transact(Stub.TRANSACTION_addBook来实现的。
            //那么mRemote是谁呢？就是onServiceConnected传回来的Server端的IBinder，但是transact是父类Binder的方法，下边分析
            mRemote.transact(Stub.TRANSACTION_addBook, _data, _reply, 0);
            _reply.readException();
        } finally {
            _reply.recycle();
            _data.recycle();
        }
    }
}
```

### Binder中用到的方法

#### transact方法

```java
/**
* 上述Proxy中的mRemote.transact方法最终执行Binder中的transact方法（这点通
* 过debug还未发现！），在该方法中依旧会调用onTransact方法-因为在Stub（Binder
* 的子类）中重写了(自动生成)onTransact方法，所以会先执行Stub类的onTransact
*/
public final boolean transact(int code, Parcel data, Parcel reply,
        int flags) throws RemoteException {
    if (data != null) {
        data.setDataPosition(0);
    }
    //执行onTransact方法（先执行我们在service中重写的onTransact方法[根据需要选择是否重写]，再执行Stub中自动生成的onTransact方法===>完成跨进程通信）
    boolean r = onTransact(code, data, reply, flags);
    if (reply != null) {
        reply.setDataPosition(0);
    }
    return r;
}
```

### 上面分析完成后，开始执行add方法
bookManager.addBook(book);
客户端用manager add了book，那么真正的执行体在哪呢？
![Alt text](./1546689948194.png)
1. 回想上边：manager如何生成的？根据Proxy和远程的IBinder生成的
2. 当执行add方法时，执行的是Proxy中的add方法
![Alt text](./1546690064092.png)

3. 观察上边client中Proxy的add方法...最终执行的还是IBinder
的transact方法--->也就是Binder（注意：这个Binder指的是remote）的transact方法--->最终会映射到Binder进程（dev/binder）
>第三步有些疑惑：通过debug并没有看到client执行mRemote时，server端有执行Binder的transact方法（难道是远程调用的原因？不清楚），直接执行的是onTransact！这点很奇怪，难道不是：<font color='#AA8800'>client执行transact--->server端的Binder.java的transact--->server端实现IBinder.java的子类的onTransact？？？</font>
><font color='#FF0000'>事实不太清楚，但大致的执行流程肯定是：
>remote.transact执行后会通过binder机制（和dev/binder有关了）来远程映射到server端的onTransact方法，远程的onTransact通过其参数（data、code）找到对应的方法，进行相应的操作。
></font>
![Alt text](./1546697701352.png)

![Alt text](./1546697857391.png)

5. OK，一次读/写完成

### 工作原理
为什么可以实现跨进程通信？原理是什么？

![Alt text](./1546618047950.png)
![Alt text](./1546618062864.png)
如上图所示：server的service打印出的线程名是 9692_2，但可以看出进程是9692（是myaidlserver应用的），那么到底是Binder进程是哪个进程号？难道是因为没有设置service的remote？因为设置之后就表示是单独进程了！
![Alt text](./1546620099505.png)
**如上图所示，确实是这样，设置其为单独的服务，并且设置名字后，发现果然成为了一个单独的服务！**

那么到底什么是Binder进程？client与server通信时，说的是靠Binder进程从中调解，但事实是 靠的是 开启的那个service所在的那个进程？
><font color='#FF0000'>**注意**：不要说Binder进程，Binder就是Binder，它是进程间通信时创建的一个对象，是一个桥梁，它是系统层面的一个进程！Binder内部维护了一个Binder线程池</font>


### AIDL编写时遇到的问题

1. 关于找不到Book.java文件的问题
>把Book.aidl放在了aidl包下，Book.java也放在了aidl包下就找不到了，原因是as只会去java下边找java文件，解决办法：

* 方案1：build文件修改
```java
sourceSets {
    main {
        java.srcDirs = ['src/main/java', 'src/main/aidl']
    }
}
```

* 方案2：把aidl放在as创建的aidl包下，把java放在java包下

| 第一种方案	 									 |第二种方案 										| 
| :--------:												 | :--------:
| ![Alt text](./1536638215799.png)     |   ![Alt text](./1536641847499.png) |

2. 对于往客户端放置aidl文件时有些问题
	>在as中，aidl文件只能放在系统生成的aidl文件夹下(应该是这样的)，而系统生成的aidl文件夹的包名是和java文件夹下的包名是一样的，所以直接把服务端的aidl文件放在aidl文件夹下肯定不行---因为包名不一致，如果尝试把aidl下的包名改成和服务端一致的，也不行，会导致java下面的包也会随之改变。其实解决办法就是在net.shopin下新建一个myaidlserver包，如下图，新建一个包A，这样就保证了server和client的aid包名一直。让然此时也可以把myaidlclient包删除了。至于B，如果是按照上述第二种方案解决的，则也要在java下新建同样的包来存放java文件了。
![Alt text](./1536648684435.png)

3. 连接service时报错
>Service Intent must be explicit: Intent { act=net.shopin.aidlservice }，可以看下ContextImpl源码，看下service和contextimpl什么关系

### AIDL权限问题
控制client调用权限
1. 在onBind方法中控制
2. 在onTransact方法中控制


