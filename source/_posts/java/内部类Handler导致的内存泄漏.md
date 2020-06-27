---
title: 内部类Handler导致的内存泄漏
date: 2018-08-02 10:40:15
tags: [handler,内部类,内存泄漏]
categories: [android,java]
---

### 关于handler内存泄漏的警告
>使用自定义handler会提示内存泄漏，这个引用链是：`activity->handler->message->queue->looper->sThreadLocal`

```java
if (FIND_POTENTIAL_LEAKS) {
    final Class<? extends Handler> klass = getClass();
    if ((klass.isAnonymousClass() || klass.isMemberClass() || klass.isLocalClass()) && (klass.getModifiers() & Modifier.STATIC) == 0) {
        Log.w(TAG, "The following Handler class should be static or leaks might occur: " + klass.getCanonicalName());
    }
}
```

#### 改进方法

```java
static class MyWeakHandler extends Handler {
	private WeakReference<WangPosCashierActivity> weakReference;
	public MyWeakHandler(WangPosCashierActivity activity) {
		weakReference = new WeakReference<>(activity);
	}
	@Override
	public void handleMessage(Message msg) {
		//调用外部类做一些事
		weakReference.get().doSth();
	}
}
```
>为什么静态内部类不会产生内存泄漏？**因为它没有持有外部类的引用**

### 内部类和外部类的关系

![Alt text](./1564709600346.png)
* 从上图的字节码文件可以看出：
1. 静态内部类不持有外部类的引用
2. 普通内部类持有了外部引用（所以它可以访问外部类的任何信息-通过那个引用访问）
* 同时也可以引申出一些其他的结论：
1. 静态内部类和外部类就相当于是完全独立的，普通内部类和外部类是相互依赖的，只有拿着外部类才能创建内部类。
2. 普通内部类中不能声明静态变量、方法，但是可以声明静态常量。
>个人理解：类中所有的静态变量、方法是在类加载时都要被加载到内存中，所以加载外部类时就会去加载内部类中的静态变量。
**那么问题就来了，普通内部类还没有被加载呢，它里边的静态变量却已经被加载了**，这样是有问题的：类还没加载，但是类的变量却加载了。
同时也可以明白为什么静态内部类可以有静态变量：因为**加载时静态内部类也会被加载，它里边的静态变量也会被加载，两者都被加载了**，所以没什么问题。

