---
title: AsyncTask的原理分析
date: 2016-08-02 14:27:37
tags: [异步任务,线程池]
categories: android
---

### 创建一个AsyncTask对象

```java
public AsyncTask() {
	//一个执行体，在里边执行后台任务、抛出执行结果
	mWorker = new WorkerRunnable<Params, Result>() {
		public Result call() throws Exception {
			mTaskInvoked.set(true);
			Process.setThreadPriority(Process.THREAD_PRIORITY_BACKGROUND);
			//noinspection unchecked
			//后台执行任务
			Result result = doInBackground(mParams);
			Binder.flushPendingCommands();
			//postResult方法中会使用Handler把执行结果抛到主线程
			return postResult(result);
		}
	};
	//把上述执行体mWorker传进来
	mFuture = new FutureTask<Result>(mWorker) {
		protected void done() {
			postResultIfNotInvoked(get());
		}
	};
}
```

>这就是new AsyncTask()所做的操作，可以想到肯定有一个地方执行了mFeture，然后调用了call方法回到WorkRunnable中开始任务的执行。

### 执行execute方法
```java
public final AsyncTask<Params, Progress, Result> execute(Params... params) {
	//
	return executeOnExecutor(sDefaultExecutor, params);
}
```

#### sDefaultExecutor是什么

```java
public static final Executor SERIAL_EXECUTOR = new SerialExecutor();
private static volatile Executor sDefaultExecutor = SERIAL_EXECUTOR;
```

#### SerialExecutor的具体实现

```java
private static class SerialExecutor implements Executor {
	final ArrayDeque<Runnable> mTasks = new ArrayDeque<Runnable>();
	Runnable mActive;
	//右下边的分析可以知道，这里的 r 就是mFuture实例
	public synchronized void execute(final Runnable r) {
		mTasks.offer(new Runnable() {
			//mActive的run方法
			public void run() {
				try {
					//线程会走到这里执行真正的执行体，也就是mFuture的run方法
					//mFuture是一个FutureTask类型，在执行Task的run时会调用call方法
					//result = c.call()这个c就是上述的mWorker实例
					r.run();
				} finally {
					scheduleNext();
				}
			}
		});
		if (mActive == null) {
			scheduleNext();
		}
	}

	protected synchronized void scheduleNext() {
		//执行体出队-把上述的mTasks.offer的那个Runnable赋值为mActive
		if ((mActive = mTasks.poll()) != null) {
			//直到这里才开始真正执行任务，会执行mActive的run方法
			THREAD_POOL_EXECUTOR.execute(mActive);
		}
	}
}
```

```java
public final AsyncTask<Params, Progress, Result> executeOnExecutor(Executor exec, Params... params) {
	if (mStatus != Status.PENDING) {
		switch (mStatus) {
			case RUNNING:
				throw new IllegalStateException("Cannot execute task: the task is already running.");
			case FINISHED:
				throw new IllegalStateException("Cannot execute task: the task has already been executed (a task can be executed only once)");
		}
	}
	//置为运行状态
	mStatus = Status.RUNNING;
	onPreExecute();
	mWorker.mParams = params;
	/**
	* exec就是sDefaultExecutor，这里执行sDefaultExecutor也就是SerialExecutor的execute方法
	* mFeture就是在构造方法里边初始化的那个FutureTask实例
	*/
	exec.execute(mFuture);
	return this;
}
```