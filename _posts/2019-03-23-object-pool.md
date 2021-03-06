---
layout: post
title:  "由sftp连接问题到对象池"
syntax: base16.monokai
tag: 对象池
---

<br>
## 前言
---

&ensp; &ensp; &ensp; 之前做过一个利用sftp上传文件的项目，做法是定时任务遍历要上传的文件，然后通过jsch jar包获取到sftp连接，然后上传，上传完了之后关闭sftp会话与连接。因为涉及到别的业务操作，所以是一个一个循环去做的。

## 问题及初步分析
---

&ensp; &ensp; &ensp; 前几天任务在跑的时候出现了一个问题，时不时地会在获取sftp连接时抛出一个异常。
`com.jcraft.jsch.jschexception: connection reset` 在网上查了一下该异常，发现该异常在服务端和客户端均有可能出现，有2个原因:

> 1. 如果一端的Socket被关闭（或主动关闭，或因为异常退出而引起的关闭），另一端仍发送数据，发送的第一个数据包引发该异常(Connect reset by peer)。
> 2. 一端退出，但退出时并未关闭该连接，另一端如果在从连接中读数据则抛出该异常（Connection reset）。

简单地说，就是在客户端与服务端通信过程中由一方断开了连接，而另一方继续写或者继续读引起的。

**分析**:
&ensp;因为我们是客户端，而且抛出了`connection reset`的异常，所以应该是sftp服务器主动断开了连接，而我们客户端继续读数据导致的。由于异常并不是一直在打印，而是时不时在打印，即也会有成功获取到sftp的情况，所以猜测应该是服务端做了连接数的限制，而我们这边由于是循环去做的，每次都会创建连接，用完关闭，但是我们只是掉了关闭的接口，服务端未必及时关闭，所以导致在之后获取连接时，超出了服务端的连接限制，导致服务端直接关闭了连接，我们这边也就抛出了异常。

## 解决方案：对象池
---

&ensp; &ensp; &ensp; 因为可能是sftp连接频繁创建与关闭导致的，所以打算复用连接测试一下。经过同事的点拨，可以通过对象池来自己实现一个sftp连接池。这样，就开始学习对象池。

#### 1. 什么是对象池
---

&ensp; &ensp; &ensp; 对象池其实就是我们要使用对象的一个集合，类似水池。我们需要使用对象时，直接从对象池中获取，使用完毕后重新放回池中，便于下次使用。

#### 2. 对象池的目的
---

&ensp; &ensp; &ensp; 减少频繁创建与销毁对象的成本，实现对象的缓存与复用。

#### 3. 对象池应具备的功能
---

- 如果有可用的对象，对象池应当能返回给客户端。
- 客户端把对象放回池里后，可以对这些对象进行重用。
- 对象池能够创建新的对象来满足客户端不断增长的需求。
- 需要有一个正确关闭池的机制来确保关闭后不会发生内存泄露。

#### 4. 对象池如何实现
---

实现对象池，需要实现以下几个组件：
- 可重用对象。这是我们最终要使用的东西，除了具备我们正常使用的功能之外，还应至少实现一个方法：

``` java
//可通过实现接口的方式。增加这个方法的目的是为了在对象使用完毕归还对象池
//的时候过滤掉一些不可重用的对象。
//判断自身的可重用状态
boolean isValid();
```

- 对象工厂。用于创建可重用对象，最好使用静态工厂方法。
- 对象池。维护一个保存对象的并发集合，用于返回可重用对象和接收归还对象。对象池最好是保持单例，便于统一管理。集合类型和初始化的策略以具体场景来选择。除此之外，对象池应保证在池对象被销毁时，正常释放所有资源，防止内存泄漏。
- 使用者。使用对象时向对象池申请，使用完毕后归还对象。

#### 5. 对象池返回对象时，可能的情况
---

- 池中存在可重用对象，直接返回
- 池中不存在可重用对象，若允许创建新对象，可创建新的对象，然后返回
- 池中不存在可重用对象，不允许创建新对象，则阻塞使用者线程，等待可重用对象归还，然后返回（可设置超时处理）
- 池中不存在可重用对象，不允许创建新对象，直接返回null，通知使用者。

&ensp; &ensp; &ensp; 学习完理论，自己开始实践，不过因为项目的关系，暂时还没能进行验证，只能等待之后再行验证。

---
摘自:
- [一个通用并发对象池的实现](http://www.importnew.com/20804.html)
- [java 对象池技术](https://www.jianshu.com/p/38c5bccf892f)
- [关于对象池的一些分析](https://droidyue.com/blog/2016/12/12/dive-into-object-pool/)
