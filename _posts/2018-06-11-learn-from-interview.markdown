---
layout:     post
title:      "项目总结：近期的两个小项目"
subtitle:   "Learn from interview"
date:       2018-06-11 16:25:00
author:     "Damon To"
header-style: text
catalog:    true
tags:
    - NIO
    - Websocket
    - JWT
    - Spring
---

> 近期为了应对校招面试，比较有针对性地做了两个技术项目：一个是基于 Java NIO 实现的简易版非阻塞 Http 服务器，另一个是基于 Spring Boot + Websocket 实现的网络聊天室。与以往的项目总结不同，这次我会忽略一些代码实现细节，把更多的篇幅用来总结开发过程中我遇到的难题以及在面试中面试官提出的相关问题。

### 基于 Java NIO 实现的简易版非阻塞 Http 服务器

首先需要强调的是，这种 [Http 服务器的实现](https://github.com/DamonDu/nio-server)是非常朴素直接的，没有考虑很多具体实践中的问题。最近在看[《Netty In Action》](https://www.amazon.com/Netty-Action-Norman-Maurer/dp/1617291471)才越来越感觉之前做项目时想法上的过于单纯。但项目本身其实十分适合初学者进阶的，因为其中涉及到的知识：Java NIO、网络多路复用、Http 报文解析、缓冲区设计，都是十分重要的基础问题，也是在校招技术面试中面试官喜欢深入考察的问题。在开发过程中，我借鉴了 [Java NIO: Non-blocking Server非阻塞服务器](http://wiki.jikexueyuan.com/project/java-nio-zh/java-nio-non-blocking-server.html)这篇文章的思路，上面提及的几个关键问题文章中也有较为详细的解析。

#### 项目基本架构

![](/img/in-post/2018-06-11-learn-from-interview/nio-aarchitecture.png)

#### Java BIO/NIO/AIO

关于 Java BIO/NIO/AIO 这三者的区别，个人觉得 [Java BIO, NIO, AIO understanding](https://www.programering.com/a/MDM0YzMwATE.html) 这篇文章总结得十分简洁到位，作者分别从编程、原理、底层三方面进行总结。编程方面作者讲解的十分到位，而要理解原理部分可能需要一些 Unix 网络编程的相关知识，以下就做一些网络编程相关的补充：

* Unix 网络编程中将 IO 模型分为五类：阻塞 IO、非阻塞 IO（轮询）、IO 复用、信号驱动式 IO、异步 IO。其中非阻塞 IO（轮询）、IO 复用、信号驱动式 IO 都可以理解为非阻塞 IO 的不同形式实现。（[图解UNIX的I/O模型](https://blog.csdn.net/lihao21/article/details/51620374)一文可以让你快速理解 Unix 中的 IO 模型）
* 无论是属于哪种 IO 模型，其 IO 过程都可以分为两个阶段： IO request（数据请求）和 IO operation（数据复制）两个阶段。
* 阻塞 IO 与非阻塞 IO 的区别在于：使用阻塞 IO 时，线程在 IO request 和 IO operation 阶段都会进入阻塞；而使用非阻塞 IO 时，线程只在 IO operation 阶段进入阻塞，IO request 无需阻塞。
* 同步 IO 与异步 IO 的区别在于：同步 IO 在 IO operation 阶段会进入阻塞，而异步 IO 在 IO request 和 IO operation 阶段都不会进入阻塞。

* 按照 Unix 网络编程中对 IO 模型的分类，Java BIO/NIO/AIO 的实现分别对应了阻塞 IO/IO 多路复用/异步 IO 这三种模型。

#### BIO vs NIO

Java AIO（在 JDK 1.7 中也称为 NIO.2），在性能上相较于 NIO 并没有带来提升，所以在 Netty 4.0.0 中也[移除了对 AIO 的支持](https://github.com/netty/netty/issues/2515)。无论是在实际应用中还是在技术面试中都较少提及 AIO，这里我们重点比较一下 BIO 与 NIO 的区别：

1. 处理的对象：BIO 直接面向 Socket 的字节流，每次从流中读一个或多个字节，直到读取完所有字节；NIO 面向缓冲块（Block），需要时可以在缓冲区中前后移动处理，这增加了处理过程的灵活性。
2. 阻塞：BIO 必须要对线程进行阻塞，NIO 无需阻塞，一个单独的线程可以管理多个输入和输出通道。
3. 选择器：Java NIO的选择器允许一个单独的线程同时监视多个通道，可以注册多个通道到同一个选择器上，然后使用一个单独的线程来“选择”已经就绪的通道。
4. 零拷贝：Java NIO中提供的 `FileChannel` 拥有 `transferTo` 和 `transferFrom` 两个方法，可直接把 `FileChannel` 中的数据拷贝到另外一个 Channel，或者直接把另外一个 Channel 中的数据拷贝到 `FileChannel` 。通过该方法传输数据并不需要将源数据从内核态拷贝到用户态，再从用户态拷贝到目标通道的内核态，同时也**避免了两次用户态和内核态间的上下文切换**，也即使用了“零拷贝”。

*总结自：[Java进阶（五）Java I/O模型从 BIO到 NIO和 Reactor模式](http://www.jasongj.com/java/nio_reactor/)*

#### HTTP 报文及解析

项目中对 HTTP 协议的支持也做了简化处理：只判别了 GET、POST、HEAD、PUT、DELETE 五种方法，主要是基于 HTTP 的 Content-Length 字段来实现 HTTP 报文切割（处理 TCP 粘包问题）。但是在面试中，HTTP 协议几乎是所有面试官会深究的一部分内容。大概涉及的问题如下：

* HTTP 报文格式：

  ![](/img/in-post/2018-06-11-learn-from-interview/Http-format.png)

* HTTP Method 有哪些？

* HTTP 常见状态码有哪些？

* HTTP 建立 TCP 长连接？（Connection：Keep-Alive）

* [HTTP 1.0/1.1/2.0 之间的区别？](https://www.jianshu.com/p/52d86558ca57)

#### Message 缓冲区设计

* 读取不完整的 message：每次 Reader 从 Channel 读取一个数据块后，先**通过一个 Http 报文 parser 来确定是否有一个完整的 message**。若有，则读取一个完整的 message 后将剩余部分缓存起来；若无，则直接将整个数据块缓存起来。缓存的数据块将在下次读取时与下次读取的数据块合并，再进行重新的 parsing。
* message buffer 的大小：MessageBuffer 实现了一个容量可伸缩的 message buffer。它提供三种大小的 buffer，4KB/128KB/1MB。初始时 buffer 默认为 4KB，若空间不足时，MessageBuffer 内部的方法会自动将 buffer 扩容。
* message buffer 的读写：仿写了 `nio.Buffer` 中的 `flip()` 函数（`readPos` 和 `writePos`），在 message buffer 的读写操作中调用 `flip()` 来避免错读、漏读。

#### 生产者-消费者队列

用一个 ArrayBlockingQueue 来存放 socket，创建两个线程，acceptor 和 processor 分别使用其 put / take 操作来进行生产和消费。

> 关于 ArrayBlockingQueue：
>
> 1. 基于数组，直接将对象放入和取出队列。（LinkedBlockingQueue 基于单向链表，放入与取出时操作 Node）
> 2. 基于一个 ReentrantLock 和两个 Condition（notEmpty 和 notFull） 实现。（LinkedBlockingQueue 基于两个 ReentrantLock ：putLock 和 takeLock 和两个 Condition：notEmpty 和 notFull 实现）

### 基于 Spring Boot + Websocket 实现的网络聊天室

[项目实现了一个简单的网络聊天室](https://github.com/DamonDu/websocketchat)，基于 Spring Boot 搭建后台，基于 Websocket 与 STOMP 协议实现即时通讯，使用 Spring Security 与 Spring Oauth 实现用户登录，基于 JWT 实现对单点登录的支持。

#### Oauth 验证流程

![](/img/in-post/2018-06-11-learn-from-interview/springbootwebsocket.png)

#### Websocket 协议

参考[WebSocket协议深入探究](http://www.infoq.com/cn/articles/deep-in-websocket-protocol)

#### JWT

- JWT(Json Web Token) 一个非常轻巧的规范。这个规范允许我们使用JWT在用户和服务器之间传递安全可靠的信息。

- 字符串形式，三部分组成：头部（Header）、载荷（Payload）、签名（Signature）

- 头部：用于描述 JWT

  ```json
  {
    "typ": "JWT", 
    "alg": "HS256" // 指定签名的加密算法
  }
  ```

- 载荷：真正需要传递的内容 + 部分其他信息

  ```json
  {
      "iss": "User JWT",
      "iat": 1441593502,
      "exp": 1441594722,
      "aud": "www.example.com",
      "sub": "sub@example.com",
      "from_user": "B",
      "target_user": "A"
  }
  ```

- 签名：用于在服务端判断 JWT 的内容是否经过篡改，使用 JWT 头部指定的加密算法。

- 格式：header.payload.signature（先通过 base64 将 JSON 转化为字符串）

- 载荷内容可以通过 base64 反编码获得，所以不应用 JWT 传输敏感数据。

- 使用场景：添加好友、创建订单、实现单点登录