---
layout:     post
title:      " Java 并发学习笔记(一)"
subtitle:   "并发基础知识"
date:       2018-2-05 16:00:00
author:     "Damon To"
header-img: "img/java-concurrency.jpg"
header-mask: 0.3
catalog:    true
tags:
    - Java
    - Concurrency
---

> 这是我阅读[《 Java 并发编程实战》](https://book.douban.com/subject/10484692/)整理出的知识大纲，第一部分主要是一些关于并发与多线程的基本概念和 Java 中并发编程的基础。

### Before Reading

1. 本篇文章对应原书 １~5 章的内容。
2. 建议通读过原书的对应内容后再使用这份知识大纲。
3. 大部分内容为原书的摘抄，少部分是我自己的归纳。
4. 文章中的代码是从原书 example 中摘抄得来，你可以从[原书官网](http://jcip.net/listings.html)获取完整的代码。

### 多线程简介

* （以**线程**为对象）研究并发的**动机**：由于同一个进程中的所有线程都将共享进程的内存地址空间，因此这些线程都能访问相同的变量并在同一个堆上分配对象，这就需要实现一种**比在进程间共享数据粒度更细**的数据共享机制。

* **多线程的优势**

  1. 发挥多处理器的优势：提高系统**吞吐率**（其实也能提升单处理器系统的吞吐率，主要是减少了等待 I/O 的时间）

  2. 简化建模：使用线程可以将复杂并且异步的工作流进一步分解为一组简单并且同步的工作流，每个工作流在一个单独的线程中运行，并在特定的同步位置进行交互。

  3. 简化异步事件的处理：单线程程序中对异步事件的处理：若使用阻塞 I/O 则会造成性能下降；若使用非阻塞 I/O 则复杂度高且易出错。在多线程中，则可简单地使用阻塞 I/O 处理。

     > “异步”（asynchronous）在中文语境里的使用较为混乱，在各种中文译本和技术文章里常常不够统一，常见的理解如下：一、指**异步事件（Asynchronous Event）**，即事件结果无法在执行后立即获得，需要在一段较长的时间后才能获得；二、指**[异步传输/异步通信（Asynchronous Communication）](https://en.wikipedia.org/wiki/Asynchronous_communication)**，即数据的接收方和传输方不需要使用一个同步的时钟，在计算机领域可以理解为遇到异步事件时不阻塞而继续执行后续代码，典型的例子就是异步 I/O。

  4. 响应更灵敏的用户界面（GUI）

* **多线程带来的问题**

  1. 安全性问题：多个线程执行顺序的不可预测带来的运行结果确性上的不可预测。

     > 认为一段代码是非线程安全的（not thread safe），并不是说其运行结果必然是错的，而是指其运行结果在多线程环境下无法保证绝对的正确性。

  2. 活跃性问题：当某个操作无法继续执行下去，就会发生活跃性问题（例如死循环）。

  3. 性能问题：线程切换时的上下文切换操作（Context Switch）造成极大的性能开销。

* **线程安全性需求的蔓延**：当某个框架在应用程序中引入并发性时，通常不可能将并发性仅局限于框架代码，因为框架本身会回调（Callback）应用程序的代码，而这些代码将访问应用程序的状态。

### **线程安全性**

* 编写线程安全的代码的**核心在于要对状态访问操作进行管理**，更具体地，是对**共享的（Shared）**和**可变的（Mutable）**状态的访问管理。“共享”意味着变量可以由多个线程同时访问，而“可变”则意味着变量的值在其生命周期内可以发生变化。

* **处理安全性问题的三种方式**
  1. 不在线程之间共享该状态变量
  2. 将状态变量设置为不可变
  3. 在访问状态变量时适用同步

* **线程安全性的定义：**当多个线程访问某个类时，这个类始终都能表现出正确的行为，那么就称这个类是线程安全的。

  * 在线程安全性的定义中，最核心的概念就是**正确性**。 正确性的含义是，某个类的行为与其**规范**完全一致。
  * 在良好的**规范**中通常会定义各种**不变性条件（Invariant）**来约束对象的状态，以及定义各种**后验条件（Postcondition）**来描述对象操作的结果。

  > 注意区分不可变（immutable）和不变性（Invariant）。

* 无状态对象一定是线程安全的。

#### 原子性

* **竞态问题（Race Condition）**：两个或多个进程对共享的数据进行读或写的操作时，**最终的结果取决于这些进程的执行顺序**。可以简单地理解为：会引发线程安全性问题的情况。

  * 大多数竞态条件的**本质**——基于一种可能失效的观察结果来做出判断或者执行某个计算。
  * 典型例子：“先检查后执行”/“读取—修改—写入”。
  * 竞态条件并不总是会产生错误，还需要某种不恰当的执行时序。
* **原子操作**：对于访问同一个状态的所有操作（包括该操作本身）来说，这个操作是以一个原子的方式执行的操作。[Package java.util.concurrent.atomic](https://docs.oracle.com/javase/8/docs/api/?java/util/concurrent/atomic/package-summary.html)

#### 加锁机制

* **同步代码块（Synchronized Block）**：线程在进入同步代码块之前会自动获得锁，并且在退出同步代码块时自动释放锁

  ```java
  synchronized (lock) {
  	//acomic operation 
  }
  ```

* **内置锁**：每个Java对象都可以用做一个实现同步的锁，这些锁被称为内置锁**（lntrinsic Lock）**或**监视器锁（Monitor Lock）**。

  * Java 的内置锁相当于一种**互斥体（或互斥锁）**，最多只有一个线程能持有这种锁。

* 锁的**可重入性**：可重入意味着线程可以进入任何一个它已经拥有的锁所同步着的代码块。“重入”意味着锁的操作粒度是“线程”而不是“调用”。**Java 内置锁是可重入的。**

* 每个共享的和可变的变量都应该**只由一个锁来保护**。当不变性条件涉及多个状态变量时，还需保证不变性条件中的每一个变量都有同一个锁保护。

* **不良并发（Poor Concurrency）应用程序**：可同时调用的数量，不仅受到可用处理资源的限制，还受到应用程序本身结构的限制。

* **同步代码块的设置策略**：应该尽量将不影响共享状态且执行时间较长的操作从同步代码块中分离出去，但也不能将同步代码块分解得过细，因为获取和释放锁等操作是有一定性能开销的。

  ```java
  /**
   * 缓存最近执行因数分解的数值及其计算结果的 Servlet
   * 完整代码：http://jcip.net/listings/CachedFactorizer.java
   */
  @ThreadSafe
  public class CachedFactorizer extends GenericServlet implements Servlet {
      @GuardedBy("this") private BigInteger lastNumber;
      @GuardedBy("this") private BigInteger[] lastFactors;
      @GuardedBy("this") private long hits;
      @GuardedBy("this") private long cacheHits;

      public synchronized long getHits() {
          return hits;
      }

      public synchronized double getCacheHitRatio() {
          return (double) cacheHits / (double) hits;
      }

      public void service(ServletRequest req, ServletResponse resp) {
          BigInteger i = extractFromRequest(req);
          BigInteger[] factors = null;
          synchronized (this) {
              ++hits;
              if (i.equals(lastNumber)) {
                  ++cacheHits;
                  factors = lastFactors.clone();
              }
          }
          if (factors == null) {
              //不影响共享状态且执行时间较长的操作
              factors = factor(i);
              synchronized (this) {
                  lastNumber = i;
                  lastFactors = factors.clone();
              }
          }
          encodeIntoResponse(resp, factors);
      }
  }
  ```

### 对象的共享

#### 可见性

* **可见性（VIsibility）**：当一个线程修改了对象状态后，其他线程可以看到发生的状态变化。

* 在缺乏足够同步的多线程程序中，由于代码**重排序（Reordering）**的存在，内存操作的执行顺序几乎是无法确定的。

* 在缺乏同步的程序中可能产生错误结果的一种情况：**失效数据**。更糟糕的是，失效值可能不会同时出现：一个线程可能获得某个变量的最新值，而获得另一个变量的失效值。

* **最低安全性（out-of-thin-airsafety）**：当线程在没有同步的情况下读取变量时，可能会得到一个失效值，但至少这个值是由之前某个线程设置的值，而不是一个随机值。这种安全性保证也被称为最低安全性（out-of-thin-airsafety）。

* **加锁与可见性**：加锁可以用于确保某个线程以一种可预测的方式来查看另一个线程的执行结果。

* `volatile` **变量**：一种稍弱的同步机制，确保将变量的更新操作通知到其他线程。（编译器和运行时都不会将该变量上的操作进行重排序）。

  * 仅当 `volatile` 变量能简化代码的实现以及对同步策略的验证时才应该适用它，因为通过 `volatile` 来控制状态的可见性比使用加锁机制脆弱且难以理解。

* **当且仅当满足以下所有条件时，才应该使用** `volatile` **变量**

  * 对变量的写入操作不依赖变量的当前值，或者你能确保只有单个线程更新变量的值。
  * 该变量不会与其他状态变量一起纳入不变性条件中。
  * 在访问变量时不需要加锁。


#### 发布与逸出

* **发布（Publish）**：使对象能够在当前作用域之外的代码中使用。

* **逸出（Escape）**：发布内部状态可能会破坏封装性，并使得程序难以维持不变性条件。当某个不应该发布的对象被发布时，这种情况就被称为逸出（Escape）。

* **当发布某个对象时，可能会间接地发布其他对象。**当发布一个对象时，在该对象的非私有域中引用的所有对象同样会被发布。

* **当把一个对象传递给某个外部方法时，就相当于发布了这个对象。**

* **外部（Alien）方法**：假定有一个类 C，外部（Alien）方法指行为并不完全由 C 来规定的方法，包括其他类中定义的方法以及类C中可以被改写的方法。

* **不要在构造过程中使用 this 引用逸出。**例如：在构造函数中启动一个线程。可以通过延迟启动，或者使用一个私有构造函数与公共工厂方法来避免逸出。

  ```java
  /**
   * 使用工厂方法来防止 this 引用在构造过程中逸出
   * 完整代码：http://jcip.net/listings/SafeListener.java
   */
  public class SafeListener {
      private final EventListener listener;

      private SafeListener() {
          listener = new EventListener() {
              public void onEvent(Event e) {
                  doSomething(e);
              }
          };
      }

      public static SafeListener newInstance(EventSource source) {
          SafeListener safe = new SafeListener();
          source.registerListener(safe.listener);
          return safe;
      }
  }

  ```

* **线程封闭（Thread Confinement）**：一种避免使用同步的方式就是不共享数据。如果仅在单线程内访问数据，就不需要同步。这种技术被称为**线程封闭（Thread Confinement）**，它是实现线程安全性的最简单方式之一。

  * **Ad-hoc 线程封闭**：维护线程封闭性的职责**完全**由程序实现来承担。Ad-hoc线程封闭是**非常脆弱**的。当决定使用线程封闭技术时，通常是因为要将某个特定的子系统实现为一个**单线程子系统**。在某些情况下，单线程子系统提供的简便性要胜过Ad-hoc线程封闭技术的脆弱性。
  * **栈封闭**：线程封闭的一种特例。在栈封闭中，**只能通过局部变量**才能访问对象。栈封闭（也被称为线程内部使用或者线程局部使用）比Ad-hoc线程封闭更**易于维护**，也**更加健壮**。
  * `ThreadLocal` **类**：这个类能使线程中的某个值与保存值的对象关联起来。`ThreadLocal` 提供了 `get` 与 `set` 等访问接口或方法，这些方法为每个使用该变量的线程都存有一份独立的副本，因此 `get` 总是返回由当前执行线程在调用 `set` 时设置的最新值。`ThreadLocal` 通常用于防止对**可变的单实例变量（Singleton）或全局变量**进行共享。[Class ThreadLocal](https://docs.oracle.com/javase/8/docs/api/java/lang/ThreadLocal.html)


#### 不变性

* **不可变对象（Immutable Object）**：如果某个对象在被创建后其状态就不能被修改，那么这个对象就称为不可变对象。

  * 不可变对象一定是线程安全的。
  * 不可变性并不等于将对象中所有的域都声明为 `final` 类型，即使对象中所有的域都是 `final` 类型的，这个对象也仍然是可变的，因为在 `final` 类型的域中可以保存对可变对象的引用。
  * “不可变的对象”与“不可变的对象引用”之间存在着差异。保存在不可变对象中的程序状态仍然可以更新，即通过**将一个保存新状态的实例来“替换”原有的不可变对象。**

* **当满足以下条件时，对象才是不可变的**

  * 对象创建以后其状态就不能修改。
  * 对象的所有域都是 `final` 类型。
  * 对象是正确创建的（在对象的创建期间，`this` 引用没有逸出）。

* 即使对象是可变的，通过将对象的某些域声明为 `final` 类型，仍然可以简化对状态的判断，因此限制对象的可变性也就相当于**限制了该对象可能的状态集合**。

* 使用 `volatile` 类型发布不可变对象来保证线程安全

  ```java
  /**
   * Immutable holder for caching a number and its factors
   * 完整代码：http://jcip.net/listings/OneValueCache.java
   */
  @Immutable
  public class OneValueCache {
      private final BigInteger lastNumber;
      private final BigInteger[] lastFactors;

      public OneValueCache(BigInteger i,
                           BigInteger[] factors) {
          lastNumber = i;
          lastFactors = Arrays.copyOf(factors, factors.length);
      }

      public BigInteger[] getFactors(BigInteger i) {
          if (lastNumber == null || !lastNumber.equals(i))
              return null;
          else
              return Arrays.copyOf(lastFactors, lastFactors.length);
      }
  }
  ```

  ```java
  /**
   * Caching the last result using a volatile reference to an immutable holder object
   * 完整代码：http://jcip.net/listings/VolatileCachedFactorizer.java
   */
  @ThreadSafe
  public class VolatileCachedFactorizer extends GenericServlet implements Servlet {
      private volatile OneValueCache cache = new OneValueCache(null, null);

      public void service(ServletRequest req, ServletResponse resp) {
          BigInteger i = extractFromRequest(req);
          BigInteger[] factors = cache.getFactors(i);
          if (factors == null) {
              factors = factor(i);
              cache = new OneValueCache(i, factors);
          }
          encodeIntoResponse(resp, factors);
      }
  }

  ```

* 即使在发布不可变对象的引用时没有使用同步，也仍然可以安全地访问该对象。

#### 安全发布

* **一个正确构造的对象可以通过以下方式来安全地发布**

  * 在静态初始化函数中初始化一个对象引用。
  * 将对象的引用保存到 `volatile` 类型的域或者 `AtomicReference` 对象中。
  * 将对象的引用保存到某个正确构造对象的 `final` 类型域中。
  * 将对象的引用保存到一个由锁保护的域中。
* **事实不可变对象（Effectively Immutable Object）**：如果对象从技术上来看是可变的，但其状态在发布后不会再改变，那么把这种对象称为事实不可变对象（Effectively Immutable Object）。通过使用事实不可变对象，不仅可以简化开发过程，而且还能由于减少了同步而提高性能。
* **可变对象的安全发布**：如果对象在构造后可以修改，那么安全发布只能确保**“发布当时”**状态的可见性。对于可变对象，不仅在发布对象时需要使用同步，而且在**每次**对象访问时同样需要使用同步来确保后续修改操作的可见性。
* **对象的发布需求取决于它的可变性**

  * 不可变对象可以通过任意机制来发布。
  * 事实不可变对象必须通过安全方式来发布。
  * 可变对象必须通过安全方式发布，且必须是线程安全的或者由某个锁保护起来。

### 对象的组合

#### 实例封闭

* **实例封闭（Instance Confinement）**：也称封闭，将数据封装在对象内部，将数据的访问限制在对象的方法上，从而更容易确保线程在访问数据时能持有正确的锁。

  * 实例封闭是构建线程安全类的一个最简单的方式，具有灵活性。
  * 通过装饰器（Decorator）实现线程安全本质上就是一种实例封闭。

  ```java
  /**
   * Using confinement to ensure thread safety
   * 完整代码：http://jcip.net/listings/VolatileCachedFactorizer.java
   */
  @ThreadSafe
  public class PersonSet {
      @GuardedBy("this") private final Set<Person> mySet = new HashSet<Person>();

      public synchronized void addPerson(Person p) {
          mySet.add(p);
      }

      public synchronized boolean containsPerson(Person p) {
          return mySet.contains(p);
      }

      interface Person {
      }
  }
  ```

* **Java 监视器模式**：把对象的所有可变状态都封装起来，并由对象自己的内置锁来保护。

* **使用私有锁而不是置锁（或任何其他可通过公有方式访问的锁）有许多优点。**私有的锁对象可以将锁封装起来，使客户代码无法得到锁，但客户代码可以通过公有方法来访问锁，以便（正确或者不正确地）参与到它的同步策略中。

#### 委托

* **线程安全性的委托**：向一个无状态的类 A 中添加一个线程安全的类型 B 的属性，所得组合对象仍然是线程安全的，因为A的状态就是线程安全类B的状态，并且A没有对B的状态增加额外的有效性的约束，则可以说A将它的线程安全性**委托**给了B。

* 可以将线程安全性委托给**多个**底层的状态变量，只要满足：状态变量之间是**相互独立且线程安全**的，并且不**存在将状态变量转化为无效状态的操作**即可。（然而状态之间并非相互独立）

* 修改接口以发布底层的可变状态

  ```java
  /**
   * 安全发布底层状态的车辆追踪器
   * 完整代码：http://jcip.net/listings/PublishingVehicleTracker.java
   */
  @ThreadSafe
  public class PublishingVehicleTracker {
      private final Map<String, SafePoint> locations;
      private final Map<String, SafePoint> unmodifiableMap;

      public PublishingVehicleTracker(Map<String, SafePoint> locations) {
          this.locations = new ConcurrentHashMap<String, SafePoint>(locations);
          this.unmodifiableMap = Collections.unmodifiableMap(this.locations);
      }

      public Map<String, SafePoint> getLocations() {
          return unmodifiableMap;
      }

      public SafePoint getLocation(String id) {
          return locations.get(id);
      }

      public void setLocation(String id, int x, int y) {
          if (!locations.containsKey(id))
              throw new IllegalArgumentException("invalid vehicle name: " + id);
          locations.get(id).set(x, y);
      }
  }
  ```

* **在现有的线程安全类中添加功能的方法**

  * **修改**原有的类：安全且容易理解与维护，但往往无法做到。
  * **扩展**原有的类：并非所有的类都将所有状态向子类公开；脆弱，不易维护。
  * **客户端加锁**：对于使用某个对象 x 的客户端代码，使用 x 本身用于保护其状态的锁来保护这段客户代码；更加脆弱、高耦合、不易维护。
  * **组合（Composition）**：通过委托实现，虽然增加额外的同步层带来了轻微的性能损失但更为健壮。

  ```java
  /**
   * 通过客户端加锁实现“若没有则增加”
   * 完整代码：http://jcip.net/listings/ListHelpers.java
   */
  @ThreadSafe
  class GoodListHelper <E> {
      public List<E> list = Collections.synchronizedList(new ArrayList<E>());

      public boolean putIfAbsent(E x) {
          synchronized (list) {
              boolean absent = !list.contains(x);
              if (absent)
                  list.add(x);
              return absent;
          }
      }
  }
  ```

  ```java
  /**
   * 通过组合加锁实现“若没有则增加”
   * 完整代码：http://jcip.net/listings/ImprovedList.java
   */
  @ThreadSafe
  public class ImprovedList<T> implements List<T> {
      private final List<T> list;

      public ImprovedList(List<T> list) { this.list = list; }

      public synchronized boolean putIfAbsent(T x) {
          boolean contains = list.contains(x);
          if (contains)
              list.add(x);
          return !contains;
      }
  }
  ```

### 基础构建模块

#### 同步容器

* **同步容器**包括 [Vector](https://docs.oracle.com/javase/8/docs/api/java/util/Vector.html) 和 [Hashtable](https://docs.oracle.com/javase/8/docs/api/java/util/Hashtable.html) 。JDK1.2 提供了Collections.synchronizedXxx等工程方法，该方法将指定集合对象的状态封装起来，并对每个公有方法都进行
  同步，并返回指定集合对象对应的同步对象。
  * **同步容器类的问题**：虽然在单个方法被使用时可以保证线程安全，但**复合操作**则需要额外的客户端加锁来保护。
  * 由于“及时失败”（fail-first）机制，当迭代器发现容器在迭代过程中被修改时，就会抛出一个 `ConcurrentModificationException` 异常。
  * 解决 `ConcurrentModificationException` 异常的方法：一种方法是对迭代行为加锁，但是降低了并发性且可能会产生死锁；另一种方法是“克隆”整个容器并让迭代在克隆容器上进行，缺点是克隆容器时的有性能开销。


#### 并发容器

* **并发容器**： [ConcurrentHashMap](https://docs.oracle.com/javase/7/docs/api/java/util/concurrent/ConcurrentHashMap.html) / [CopyOnWriteArrayList](https://docs.oracle.com/javase/7/docs/api/java/util/concurrent/CopyOnWriteArrayList.html) / [Queue](https://docs.oracle.com/javase/7/docs/api/java/util/Queue.html) / [BlockingQueue](https://docs.oracle.com/javase/7/docs/api/java/util/concurrent/BlockingQueue.html)
  * 并发容器的迭代器不会抛出 `ConcurrentModificationException` 异常且具有弱一致性（Weakly Consistent）。弱一致性的迭代器可以容忍并发的修改，当创建迭代器时会遍历已有的元素，并可以（但是不保证）在迭代器被构造后将修改操作反映给容器。
  * 并发容器的一些方法（如 `size` 和 `isEmpty`）的语义被略微减弱以反映并发特性。例如：允许 `size` 返回一个近似值而非精确值。
  * 一些常见的复合操作在容器中实现为原子操作。  
* 阻塞队列支持**生产者—消费者模式**，这种模式消除了生产者与消费者之间的代码依赖性，简化工作负载的管理。常见的生产者—消费者模式：线程池和工作队列的组合。
* 双端队列（Deque）适用于工**作密取模式（Work Stealing）**。


#### 同步工具类

* **同步工具类**根据自身的状态来协调线程的控制类。
* **闭锁（Latch）**：延迟线程的进度直到其到达终止状态，用来确保某些活动知道其他活动都完成后才继续执行。
  * [CountDownLatch](https://docs.oracle.com/javase/7/docs/api/java/util/concurrent/CountDownLatch.html)
  * 打开闭锁使得主线程能同时释放所有工作线程，而光笔闭锁则使得主线程能够等待最后一个线程执行完成。
  * [FutureTask](https://docs.oracle.com/javase/7/docs/api/java/util/concurrent/FutureTask.html) 也可用于实现闭锁，它可以处于三种状态：等待运行、正在运行、运行完成。它可以用于使计算在使用计算结果之前启动，减少等待计算的时间。
* **计数信号量（Couting Semaphore）**：用来控制同时访问某个特定资源的操作量，或者同时执行某个指定操作的数量。还可以用来实现某种资源池，或者对容器施加边界。
* **栅栏（Barrier）**：类似于闭锁，它能阻塞一组线程直到某个事件发生。与闭锁的关键区别在于，所有线程必须同时到达栅栏位置，才能继续执行。（即闭锁用于等待事件，而栅栏用于等待其他线程）
  * [CyclicBarrier](https://docs.oracle.com/javase/7/docs/api/java/util/concurrent/CyclicBarrier.html)
  * 两方栅栏 [Exchanger](https://docs.oracle.com/javase/7/docs/api/java/util/concurrent/Exchanger.html)

