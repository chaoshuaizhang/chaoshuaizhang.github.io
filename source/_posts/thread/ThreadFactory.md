---
title: ThreadFactory
date: 2018-07-01 10:46:38
tags: thread
categories: java
---

# ThreadFactory

>好像从没有用过这个ThreadFactory，不过从名字看，应该、是造线程的工厂。如果我们不先设置ThreadFactory的话，它会使用默认的Factory
```java
static class DefaultThreadFactory implements ThreadFactory {
	private static final AtomicInteger poolNumber = new AtomicInteger(1);
	private final ThreadGroup group;
	private final AtomicInteger threadNumber = new AtomicInteger(1);
	private final String namePrefix;

	DefaultThreadFactory() {
		SecurityManager s = System.getSecurityManager();
		//线程组
		group = (s != null) ? s.getThreadGroup() : Thread.currentThread().getThreadGroup();
		//设置线程池中默认的线程名
		namePrefix = "pool-" + poolNumber.getAndIncrement() + "-thread-";
	}

	public Thread newThread(Runnable r) {
		Thread t = new Thread(group, r, namePrefix + threadNumber.getAndIncrement(), 0);
		//恢复为非守护线程
		if (t.isDaemon())
			t.setDaemon(false);
		//恢复为默认的优先级
		if (t.getPriority() != Thread.NORM_PRIORITY)
			t.setPriority(Thread.NORM_PRIORITY);
		return t;
	}
}
```
**注意**为什么要执行恢复为非守护线程，默认优先级？
>因为线程的创建是依赖当前线程的，当前线程创建了子线程，则当前线程就是子线程的Parent，子线程的一些属相会直接使用父线程的，所以要恢复为默认的。

```java
this.daemon = parent.isDaemon();
this.priority = parent.getPriority();
if (security == null || isCCLOverridden(parent.getClass()))
    this.contextClassLoader = parent.getContextClassLoader();
else
    this.contextClassLoader = parent.contextClassLoader;
this.inheritedAccessControlContext =
        acc != null ? acc : AccessController.getContext();
this.target = target;
setPriority(priority);
if (parent.inheritableThreadLocals != null)
    this.inheritableThreadLocals =
        ThreadLocal.createInheritedMap(parent.inheritableThreadLocals);
```
