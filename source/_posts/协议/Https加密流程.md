---
title: Https加密流程
date: 2017-09-14 23:43:05
tags: Https
categories: 协议
---

# Https加密流程

## 加密方式
>加密方式大体分为对称加密、非对称加密两种（简单的摘录一下网上的介绍，便于理解上下文）

1. 对称加密算法的优点是**算法公开、计算量小、加密速度快、加密效率高**。对称加密算法的缺点是在数据传送前，发送方和接收方必须商定好秘钥，然后使双方都能保存好秘钥。其次如果一方的秘钥被泄露，那么加密信息也就不安全了。另外，每对用户每次使用对称加密算法时，都需要使用其他人不知道的唯一秘钥，这会使得收、发双方所拥有的钥匙数量巨大，密钥管理成为双方的负担。

2. 非对称加密与对称加密相比，其安全性更好：对称加密的通信双方使用相同的秘钥，如果一方的秘钥遭泄露，那么整个通信就会被破解。而非对称加密使用一对秘钥，一个用来加密，一个用来解密，而且公钥是公开的，私钥是自己保存的，不需要像对称加密那样在通信之前要先同步秘钥。非对称加密的**缺点是加密和解密花费时间长、速度慢，只适合对少量数据进行加密**。

## Https采用的加密方式
>根据上述的优缺点，Https采用了两者结合的方式进行加密。

### 只使用非对称加密
* 如何验证server是真的server？
![Alt text](./1531019257645.png)

* 上述的单项**认证过程**过程最起码是安全的。
* 那么开始正常的数据传输
![Alt text](./1531019240418.png)
* 注意上述的传输过程，client和server的身份都确定了，但是存在一个问题就是：server回复的信息可能被第三者截获，虽然第三者无法修改信息（即使修改了也没用，因为没有私钥，无法对修改后的数据进行加密），但它可以解密（因为公钥是公开的）看到信息，这样数据就无法保密了。
>**实验证明：只用非对称加密是不可靠的，同样从上述非对称加密的缺点来看，它并不适合对大批明文进行加密。**

### 只使用对称加密
>1. 对称的话，秘钥是保密的，所以（只要不泄露）肯定能保证安全。
>2. 那么问题来了：server如何把秘钥给client？这岂不又是一个发送信息而又要信息保密的过程？
>3. 想一下非对称加密：
>server--->client不是安全的（信息无法保密）
>client--->server是安全的（即使被窃取到了，没有私钥也无法解密）
>4. 所以可以让**client**生成一个对称秘钥Key，这个Key经非对称加密由client--->server，这样不就行了？
![Alt text](./1531021694804.png)

>1. 因为私钥只有server拥有，因此client可以通过判断对方是否有私钥来判断对方是否是server。
>2. client通过非对称加密的掩护，安全的和server商量好一个对称加密算法和密钥来保证后面通信过程内容的安全。

---------
上述看起来已经安全了，但有个问题：非对称加密确实保证了对称加密的秘钥的安全传输，那么谁来保证非对称加密的公钥的安全传输呢？
1. ？？？非对称加密的公钥不是公开的吗？
公钥是公开的，那么server如何把公钥发给client呢？
* 把公钥放到互联网的某个地方的一个下载地址，事先给client去下载。
* 每次和client开始通信时，“服务器”把公钥发给server。
2. 对于方法1，client无法确定这个下载地址是不是server发布的，可能是第三者自己生成的Public，然后伪造的假的地址。
对于方法2，也有问题，因为第三者也可以自己生成一对Public和Private，他只要向client发送他自己的Public就可以冒充Server了（**这样的话，相当于后期的所有操作都是在和假的server做交互**）。

### 使用数字证书保证公钥的安全
>1. 为了解决上述【大家都可以生成公钥、私钥对，无法确认公钥对到底是谁的】的这个问题，出现了数字证书。
>2. 那也就是说：数字证书可以保证数字证书里的公钥确实是这个证书的所有者(Subject)的，或者证书可以用来确认对方的身份。

#### 数字证书里边都有什么
>公钥、证书信息、hash算法、SSl版本信息。

![Alt text](./1563795189720.png)
>上述的数字证书发给client后：
>1. client根据本地**CA的公钥**对数字签名解密，得到A
>2. 把公钥等信息进行hash得到B
>3. 对比A和B是否相等，相等说明没人篡改。

## 流程
![Alt text](./1563794147322.png)
* 为什么是三个随机数？
>客户端产生一个Random1、服务端产生一个Random2，但1、2可能是伪随机数，这样的话可能就能猜出加密算法，所以再给一个随机数Premaster secret，这次是真正的"随机数"，是由SSL协议自己产生的。

## 参考文章
[数字证书原理](https://www.cnblogs.com/JeffreySun/archive/2010/06/24/1627247.html)
[关于Https那些事](https://mp.weixin.qq.com/s?__biz=MzA5MzI3NjE2MA==&mid=2650242840&idx=1&sn=8c0baf32761c3caca218a5b33b07a1c1&chksm=88638e77bf140761aa0e63a4be3429a017317e700743ccd8e498ef959bb71265bd95a7500398&mpshare=1&scene=1&srcid=0707HrykBg05naCQGiwIaARl#rd)
[也许，这样理解HTTPS更容易](https://mp.weixin.qq.com/s?__biz=MzI0MjE3OTYwMg==&mid=2649551508&idx=1&sn=74400e80ac0aa575b11604876bd70b1a&chksm=f11819e9c66f90ffc5dfa9c3e7ed761a94a3fcd1b3a20906818f05e5c888fe7211343cb53232&mpshare=1&scene=1&srcid=0503BGQGXsNLVUwAMuJRLL3z#)