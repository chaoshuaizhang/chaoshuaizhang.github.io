---
title: AOP的一些使用与理解
date: 2018-08-15 23:56:01
tags: AOP
categories: java
---

# AOP

[TOC]

>* AOP作用：日志记录，权限验证，事务控制，性能检测，错误信息检测等等，和核心业务没有任何关系。
>* 为什么要使用切面编程：方便，提取公共的非业务代码，避免一些逻辑校验代码与业务代码混合，可以使用分层思想，从上到下，一个一个的切面，剥离一些“无用”的代码，直到抽出核心代码。

>AOP-AspectJ-SpringAOP-JDK动态代理-CGlib动态代理这几个的关系
>![Alt text](./67c610db84fdb4a7d346f0a5d181ca23.png)

### 实现切面的两种思路
#### 横向扩展
* AspectJ
* 代理

#### 纵向扩展
>一般通过继承的方式：A要有个打日志的功能，则B继承A，B重写A的方法，添加日志，A又想有监测的功能，则C继承B，重写B的方法，添加监测功能。

缺点：如上所述，添加的功能越多，代码耦合越大。

### AspectJ分析
>分为两种：
>* 基于AspectJ语法编写，然后使用ajc（默认java使用的是javac，因为要编译AspectJ，需要在IDE中改成ajc，或者是使用命令，依赖于aspectjtool包）编译.aj文件，生成标准的class文件。
>* 基于注解，Java语言+aspectj包的注解，通过AspectJ提供的编织器，编织代码到目标class文件中（**但是我反编译class文件，没有看到被织入的代码**）。

##### 织入的概念
>何为织入？使切面的逻辑代码并入（逻辑上并入或者物理上并入）到业务代码中，这里的逻辑和物理是指：A中调用B称为逻辑织入，A和B融合到一起称为物理织入。

* 如何织入？两种方式吧（[参考网络](https://blog.mythsman.com/2017/12/21/1/)）
1. 提供注册机制，通过提供配置文件（注册表）指明那些类需要被切。（**逻辑织入**）
2. 在编译期或者类加载期，优先编译或加载切面代码，然后将切面代码并入到指定的类中，一般都是通过aspectjweaver包实现的。（**物理织入**）

>总感觉这两种方式差不多吧，难道第一种是我使用某一个方法时，先查一下它是否进行了切面注册，如果是的话，先执行切面？第二种就是我常用的方式。

##### 物理织入
>**具体分为三种**
>>使用AspectJ语法的话，默认只能使用第一种方式（因为.aj文件只能通过ajc编译），使用注解的话，下列三种方式都可以实现。
1. 编译时织入
>利用**ajc编译器替代javac编译器**，直接将源文件(java或者aspect文件)编译成class文件并将切面织入进代码。
2. 编译后织入
>利用ajc编译器**向javac编译期编译后的**class文件或jar文件织入切面代码。
3. 加载时织入
>**不使用ajc编译器**，利用aspectjweaver.jar工具，使用java agent代理在类加载期将切面织入进代码。

![Alt text](./8acf8e36d6042f9459df05007a3c803c.png)

#### 第一种、使用AspectJ语法
>这种方式只需借助`aspectjrt.jar、aspectjtools.jar这两个jar包`

##### 使用原生AspectJ语法定义切面
>此文件为TestAspect.aj文件，默认IDE是不能创建.aj文件的，添加spring-aop依赖后就可以创建。

```java
public aspect TestAspect {
    //定义切点
    pointcut webLog(): execution(public * net.shopin.pdaware.common..*.sayHello(..));
    //前置通知
    before(): webLog() {
        System.out.println("前置 入参打印" + thisJoinPoint.getArgs());
    }
    //环绕通知
    around(): webLog(){
        System.out.println("环绕打印");
        //获得参数名
        String[] names = ((CodeSignature) thisJoinPointStaticPart.getSignature()).getParameterNames();
        //获得参数
        Object[] args = thisJoinPoint.getArgs();
    }
    //后置通知
    after(): webLog(JoinPoint point){
        System.out.println("后置 返参打印");
    }
}
```

##### 织入后的文件
>首先，我没有看到织入成功的文件，配置了ajc但是没生效，所以拷贝一段网络的代码吧。

```java
public void sayHello() {
	try {
		//前置通知
		TestAspect.aspectOf().ajc$before$TestAspect$1$682722c();
		//核心业务代码
		System.out.println("Hello");
	} catch (Throwable localThrowable) {
		//后置通知（不清楚为什么这里也有后置通知）
		TestAspect.aspectOf().ajc$after$TestAspect$2$682722c();
		throw localThrowable;
	}
	//明明这里有一个后置通知就行了，为什么上边的catch中还会有一个？
	TestAspect.aspectOf().ajc$after$TestAspect$2$682722c();
}
```
##### 编译后的Aspect文件
* 看到上述的前置后置都是调用TestAspect.aspectOf().XXX来实现打日志，可以联想到可能是个单例
```java
public static TestAspect aspectOf() {
	if(ajc$perSingletonInstance == null) {
		throw new NoAspectBoundException("TestAspect", ajc$initFailureCause);
	}
	return ajc$perSingletonInstance;
}
```
* 什么时候初始化perSingletonInstance呢？
```java
//静态代码块，类加载时就先初始化
static {
	try {
		//实例化我们定义的切面对象
		ajc$postClinit();
	} catch (Throwable localThrowable) {
		ajc$initFailureCause = localThrowable;
	}
}
```

```java
private static void ajc$postClinit() {
	//可以看到在这里实例化了一个TestAspect对象
	ajc$perSingletonInstance = new TestAspect();
}
```

* 通过单例来调用前置、后置、环绕等方法
```java
public void ajc$before$TestAspect$1$682722c() {
	System.out.println("前置 入参打印");
}

public void ajc$after$TestAspect$2$682722c() {
	System.out.println("后置 返参打印");
}
```
>通过看上边.aj文件的编译情况，可以想到为什么说AspectJ是静态编译了，他就是在核心业务代码前后调用了一些方法，和我们手动定义一些Util方法是有些类似的，不同的是，这些AspectJ会自动为我们做好。
>缺点：目前还没


#### 第二种、使用注解
>这种方式不仅需要借助`aspectjrt.jar、aspectjtools.jar这两个jar包`还需要`aspectjweaver.jar`，因为注解的方式需要把目标class文件和切面class文件进行织入。aspectj语法的方式不需要aspectjweaver，不太清楚为什么不需要aspectweaver包，但肯定和它是.aj有关，哈哈。
>>下面将讲三种方式：编译时、后、加载时。我从服务端down的class文件中没有织入代码，难道使用的是加载时的织入方式？

```java
@Aspect
@Order(1)
public class WebLogAspect {
    @Pointcut("execution(public * net.shopin.pdaware..*Controller.*(..))")
    public void webLog() {}

    @Before("webLog()")
    public void doLogBefore(JoinPoint joinPoint) {}
    @After("webLog()")
    public void doLogAfter(JoinPoint joinPoint) {}
}
```

##### 方式一、编译时织入
>所谓编译时织入就是：在使用ajc编译目标.java文件和切面文件时，就进行融合，生成的class文件和上述使用AspectJ语法编写后的织入是一样的。

##### 方式二、编译后织入
>编译后织入分为两步。

* 第一步：使用javac编译目标文件和切面文件，生成.class（和java文件没有区别，看不出什么）。
* 第二步：使用ajc再次处理（编译？）上述class文件，得到新的织入切面代码后的class文件（此时看到的新的class文件和上述AspectJ语法编译后的class是一样的，单例模式）。


* * *
>上述两种aspect.class时如何织入到target.class文件中的？
>>我们在**aspect中定义的有我要切哪些目标类、方法等**，所以编译期会通过这些标识去找到对应的target，然后织入。

##### 方式三、加载时织入（LTW）
>LTW：Load-Time Weaving
>此时就不借助于ajc工具了。

###### 如何实现？
>上述已经说了，编译时、后是通过ajc查看class为标识把目标class和切面class进行关联，那么运行时呢？已经编译过了，咋关联？
>>是否可以通过一种方式：在加载目标类时它知道它要被某个切面进行关联，所以JVM要先加载那个切面，然后再加载它自己。也就是在目标类加载之前，其对应的切面必须被加载完成。

**具体的织入方式需要借助于JavaAgent**
>对于javaagent这里就不做详细介绍了，毕竟不清楚。大致拷贝下它的作用

* 可以在加载 class 文件之前做拦截，对字节码做修改
* 可以在运行期对已加载类的字节码做变更，但是这种情况下会有很多的限制
* 还有其他一些小众的功能：
>* 获取所有已经加载过的类
>* 获取所有已经初始化过的类（执行过 clinit 方法，是上面的一个子集）
>* 获取某个对象的大小
>* 将某个 jar 加入到 bootstrap classpath 里作为高优先级被 bootstrapClassloader 加载
>* 将某个 jar 加入到 classpath 里供 AppClassloard 去加载
>* 设置某些 native 方法的前缀，主要在查找 native 方法的时候做规则匹配

**从agent的作用可以看出来，它可以实现运行时的织入。**

###### 使用LTW方式的要求
>官方给了三个要使用LTW的前置条件。
1. All aspects to be used for weaving **must be defined to the weaver before any types to be woven are loaded**. This avoids types being "missed" by aspects added later, with the result that invariants across types fail.
>在加载需要被织入切面的业务代码前，必须把所有的织入类先加载了。
2. All aspects visible to the weaver are usable. A visible aspect is one defined by the weaving class loader or one of its parent class loaders. All concrete visible aspects are woven and all abstract visible aspects may be extended.
3. **A class loader may only weave classes that it defines.** It may not weave classes loaded by **a delegate or parent class loader**.
>类加载器可能只会织入它定义（加载）的那些类。其它加载器、父加载器定义的类他可能无法织入。

### SpringAOP

##### 疑惑点
>总是不清楚SpringAOP这个概念，都说实现方式是JDK动态代理或者是Cglib，但是我在Spring项目中使用的只是AspectJ，所以就一直在想AspectJ是不是也是SpringAOP的一种实现方式，毕竟SpringAOP也依赖了aspectjweaver的包。

###### pom中的依赖
>可以看到spring-aop中也是依赖aspectjweaver的，所以为啥它的实现方式只是代理而没有AspectJ呢？

* spring-boot-starter-aop
![Alt text](./de2ffb0b869b69dd743b870a970e2830.png)
* spring-aop
![Alt text](./eec369c68e59c64f7144db801613ed47.png)

##### 如何判断你用的是织入方式还是代理方式
>目前没能直接看出来，只能通过打印。

```java
IAnimalEat target = (IAnimalEat) factory.getProxy();
System.out.println("生成的新的代理类：" + target.getClass().getName());
```
* JDK
![Alt text](./6121ca1a8f560bccbdab7df5bb0d8d61.png)
* Cglib
![Alt text](./4ad811775c2ba1fe9a2f72e7011385e6.png)

### JDK动态代理原理
>直接看手写动态代理那块儿就行，重点是为什么非得要接口才能实现。

#### 为什么非要接口呢？
>个人理解，通过调用Proxy$0的xxx方法，映射到真正实例的xxx方法。

### Cglib代理原理

### SpringAOP分析

#### 实现方式
>都知道是两种方式，什么有接口是jdk，没接口是Cglib。

##### 模板实例
>先不用管这个实例会采用什么方式代理，先分析流程。
```java
ProxyFactory factory = new ProxyFactory();
IAnimalEat dog = new Dog();
//第一步：设置目标代理类
factory.setTarget(dog);
//第二步：设置各种通知
factory.addAdvice(new BeforeAdvice());
factory.addAdvice(new AfterReturnAdvice());
//得到代理类（这块儿如果没有接口的话，就直接用实体类接收就行，不影响）
IAnimalEat target = (IAnimalEat) factory.getProxy();
//执行业务方法
target.eat();
```
>这个简单的SpringAOP就实现了，当然可以对其进行封装，详见下边的参考文档。

###### 源码分析
>分析上述模板代码的执行流程。

**从模板可以看出，核心方法应该是getProxy()，so我们先分析它。**

* ProxyFactory.java
```java
public Object getProxy() {
	return this.createAopProxy().getProxy();
}

protected final synchronized AopProxy createAopProxy() {
    //getAopProxyFactory得到的就是一个DefaultAopProxyFactory()对象
	return this.getAopProxyFactory().createAopProxy(this);
}
```
* DefaultAopProxyFactory.java
>核心代码，可以看到具体使用jdk还是cglib全在这里进行判断

```java
public AopProxy createAopProxy(AdvisedSupport config) {
	//【1】此处成立则使用JDK代理
	if (!config.isOptimize() && !config.isProxyTargetClass() && !this.hasNoUserSuppliedProxyInterfaces(config)) {
		return new JdkDynamicAopProxy(config);
	} else {
		Class<?> targetClass = config.getTargetClass();
		//【3】判断targetClass是否是接口，是否是proxy的子类，都是的话用JDK代理，否则Cglib
		return (AopProxy)(!targetClass.isInterface() && !Proxy.isProxyClass(targetClass) ? new ObjenesisCglibAopProxy(config) : new JdkDynamicAopProxy(config));
	}
}

//这个方法的cache这块儿不太懂
public static boolean isProxyClass(Class<?> cl) {
	return Proxy.class.isAssignableFrom(cl) && proxyClassCache.containsValue(cl);
}

private boolean hasNoUserSuppliedProxyInterfaces(AdvisedSupport config) {
	//得到设置的接口类（要么有一个，要么没有）
	Class<?>[] ifcs = config.getProxiedInterfaces();
	//【2】
	return ifcs.length == 0 || ifcs.length == 1 && SpringProxy.class.isAssignableFrom(ifcs[0]);
}
```
【1】：config应该就是一个配置类，isOptimize默认是false，isProxyTargetClass默认也是false，但是这两个属性都提供了set方法供我们修改它的值。
【2】：hasNoUserSuppliedProxyInterfaces根据是否有接口、接口个数是不是1并且接口是不是SpringProxy类型，来返回true、false。
>先不管第三点，**如果我们想用JDK代理，则需要保证**：
```java
1. Optimize = false//是否对产生代理的策略进行优化
2. isProxyTargetClass = false//设置代理的模式jdk/cglib
3. 设置了接口类，并且接口不继承自SpringProxy。
```
>如何设置接口类？

* AdvisedSupport.java
```java
public Class<?>[] getProxiedInterfaces() {
	//interfaces是一个list对象
	return (Class[])this.interfaces.toArray(new Class[this.interfaces.size()]);
}
```
>看一下interfaces是在哪赋值的？

```java
public void setInterfaces(Class... interfaces) {
	this.interfaces.clear();
	Class[] var2 = interfaces;
	int var3 = interfaces.length;
	for(int var4 = 0; var4 < var3; ++var4) {
		Class<?> ifc = var2[var4];
		//调用addInterface方法，往上述的interfaces里添加接口类
		this.addInterface(ifc);
	}
}

public void addInterface(Class<?> intf) {
	if (!this.interfaces.contains(intf)) {
		this.interfaces.add(intf);
		this.adviceChanged();
	}
}
```
>OK，只要设置setInterfaces就行，我们发现**ProxyFactory继承自ProxyCreatorSupport，ProxyCreatorSupport继承自AdvisedSupport**。
>所以上述的三点可以改为：
```java
1. Optimize = false//是否对产生代理的策略进行优化
2. isProxyTargetClass = false//设置代理的模式jdk/cglib
3. 设置setInterfaces，并且接口不继承自SpringProxy。
```
>同时发现设置优化时才会有机会使用Cglib，难道Cglib性能更好？（Cglib：创建慢，执行块，JDK：创建快，执行慢）。
* 所以，最终如果想使用JDK，则需要把模板修改为：

```java
factory.setTarget(dog);
factory.setProxyTargetClass(false);//默认false，可不设置
factory.setOptimize(false);//默认false，可不设置
factory.setInterfaces(IAnimalEat.class);//设置目标类的接口
factory.addAdvice(new BeforeAdvice());
factory.addAdvice(new AfterReturnAdvice());
IAnimalEat target = (IAnimalEat) factory.getProxy();
```
* 运行：
![Alt text](./b1b1ed687d431b10051a5c88bcf30d5a.png)

* 如果我们想使用Cglib代理呢？
>**必要不充分条件**：上述的三个条件只要有一个不成立就行。
>忘记了还有一个重要的判断呢。

【3】：判断targetClass是否是接口类型，是否继承proxy等
* 我们自己定义的代理类不可能是接口，并且一般不会继承自Proxy类，所以当逻辑走到了这里，并且代理类是我们自己定义的时候，一般就会使用Cglib代理。

#### Cglib和JDK比较
>性能问题：由于Cglib代理是利用ASM字节码生成框架**在内存中生成一个需要被代理类的子类完成代理**，而JDK动态代理是**利用反射**原理完成动态代理，所以Cglib创建的动态代理对象性能比JDk动态代理动态创建出来的代理对象新能要好的多，但是对象创建的速度比JDk动态代理要慢，所以，当Spring使用的是单例情况下可以选用Cglib代理，反之使用JDK动态代理更加合适。同时还有一个问题，被final修饰的类只能使用JDK动态代理，因为被final修饰的类不能被继承，而Cglib则是利用的继承原理实现代理的。[摘自](https://my.oschina.net/zicheng/blog/1920892)






