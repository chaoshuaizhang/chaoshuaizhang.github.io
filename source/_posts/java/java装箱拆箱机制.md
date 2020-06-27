---
title: java装箱拆箱机制
date: 2017-06-11 12:10:16
tags: 装箱/拆箱
categories: java
---

默认情况下我们定义一个包装类型的对象，应该是Integer a = new Integer(10);
但是有了装箱机制后，可以直接这么定义：Integer a = 10;
>本文只对Integer做一些简单介绍，其它的可以看源码。

### 装箱
>这个过程java会自动实现装箱机制
```java
Integer num = 10;
```
>可以看下字节码文件，会发现装箱的原理就是`Integer.valueOf(...)`

### 拆箱
```java
int sum = num;
```
>拆箱的原理是：`Integer.intValue(...)`

### 弊端
>自动装箱拆箱会耗费一些性能。避免在for循环里使用类似这样的操作:

```java
Integer i = 0;
for(){
	//这块会先拆箱，再装箱
	i+=1;
}
```

### 包装类型与普通类型的比较
* “==”运算符左右两边的操作数都是包装类型时，比较的是对象，否则比较的是数值
* `equals`作比较时会先看左右两边是不是同一类型，只有同一类型才会做比较
```java
public boolean equals(Object obj) {
	//Long等其它类型类似
	if (obj instanceof Integer) {
		return value == ((Integer)obj).intValue();
	}
	return false;
}
```
