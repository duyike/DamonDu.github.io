---
layout:     post
title:      " Java 并发学习笔记(二)"
subtitle:   "结构化并发应用程序"
date:       2018-2-08 19:00:00
author:     "Damon To"
header-style: text
catalog:    true
tags:
    - Java
    - Concurrency
---

> 这是我阅读[《 Java 并发编程实战》](https://book.douban.com/subject/10484692/)整理出的知识大纲，第二部分主要是关于 Java 中如何使用 Executor 框架和线程池等技术来结构化地管理多线程应用程序。

### Before Reading

1. 本篇文章对应原书 6~9 章的内容。
2. 建议通读过原书的对应内容后再使用这份知识大纲。
3. 大部分内容为原书的摘抄，少部分是我自己的归纳。
4. 文章中的代码是从原书 example 中摘抄得来，你可以从[原书官网](http://jcip.net/listings.html)获取完整的代码。

### 任务执行

* **服务器应用程序（典型的并发应用程序）设计的目标**
  * **正常**的负载下：应该能同时表现出**良好的吞吐量和快速的响应性**。
  * 当**负荷过载**时：应用程序的**性能应该是逐渐降低**，而不是直接失败。
* 要实现并发应用程序的设计目标，应该
  * 选择清晰的**任务边界**
  * 选择明确的任务**执行策略**
* **任务边界**：大多数服务器应用程序都提供了一种自然的任务边界选择方式：**以独立的客户请求为边界**。
* **执行策略**
  * **串行执行**：简单，性能糟糕（服务器资源利用率、吞吐量、响应性）
  * **显式地为任务创建线程**：在正常负载下性能较好，但简单地为任务创建线程存在缺陷（尤其是线程数量大时）：线程生命周期的开销高、资源消耗高、稳定性差，导致在**高负载时性能差**（也即是可用性低）。
  * **使用** `Executor` **框架**：基于**生产者 — 消费者模式**，提供了一种标准的方法将任务的提交过程与执行过程解耦开来，并用 `Runnable` 来表示任务；提供了对生命周期的支持，以及统计信息收集、应用程序管理机制和性能监视等机制。[Executors](https://docs.oracle.com/javase/7/docs/api/java/util/concurrent/Executors.html) ；[Executor](https://docs.oracle.com/javase/7/docs/api/java/util/concurrent/Executor.html) ；

#### 使用 Executor 框架

* **线程池**：指管理一组同构工作线程的资源池。
  * **工作原理**：由工作者线程（Worker Thread）从工作队列中获取一个任务，执行任务，然后返回线程池并等待下一个任务。
  * **使用线程池的好处**：
    1. 通过重用现有的线程而不是创建新线程，减少线程创建和销毁线程产生的巨大开销。
    2. 当请求到达时，工作线程通常已经存在，因此不会由于等待创建线程而延迟任务的执行，提高了响应性。
    3. 基于线程池，应用程序的稳定性提高：在高负载时不会直接失败，而是平缓地降低性能。
    * [newFixedThreadPool](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/Executors.html#newFixedThreadPool-int-java.util.concurrent.ThreadFactory-) / [newCachedThreadPool](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/Executors.html#newCachedThreadPool--) / [newSingleThreadExecutor](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/Executors.html#newSingleThreadExecutor--) / [newScheduledThreadPool](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/Executors.html#newScheduledThreadPool-int-)
* 使用 [Interface ExecutorService](https://docs.oracle.com/javase/7/docs/api/java/util/concurrent/ExecutorService.html) 管理 `Executor` 的生命周期
  * 三种生命状态：运行、关闭、已终止。
  * `shutdown` 方法执行平缓的关闭过程；`shutdownNow` 方法执行强制关闭过程。
  * 在 `ExecutorService` 关闭后提交的任务将由**“拒绝执行处理器（Rejected Execution Handler）”**来处理。
  * `awaitTermination` 等待 `ExecutorService` 到达终止状态；`isTerminated` 轮询 `ExecutorService` 是否已经终止。
* 使用 `ScheduledThreadPoolExecutor` 创建延迟任务和周期任务。

#### 进一步的分解任务以提高并行性

* [**区分 Runnable、Callable、Future、FutureTask**](http://blog.csdn.net/bboyfeiyu/article/details/24851847)

  * Executor 中包含了一些辅助方法能将其他类型的任务封装为一个 `Callable`
  * `Future` 表示一个任务的生命周期，并提供了相应的方法来判断是否已经完成或取消，以及获取任务的结果和取消任务等。`get` 方法的行为**取决于任务的状态**（尚未开始、正在运行、已完成）。如果任务已经完成，那么   `get` 会立即返回或者抛出一个 `Exception` ，如果任务没有完成，那么 `get` 将阻塞并直到任务完成。

  ```java
  /**
   * 使用 Future 等待图片下载，实现异构任务（文字/图片）并行化
   * 完整代码：http://jcip.net/listings/FutureRenderer.java
   */
  public abstract class FutureRenderer {
      private final ExecutorService executor = Executors.newCachedThreadPool();

      void renderPage(CharSequence source) {
          final List<ImageInfo> imageInfos = scanForImageInfo(source);
          //建立任务
          Callable<List<ImageData>> task =
                  new Callable<List<ImageData>>() {
                      public List<ImageData> call() {
                          List<ImageData> result = new ArrayList<ImageData>();
                          for (ImageInfo imageInfo : imageInfos)
                              result.add(imageInfo.downloadImage());
                          return result;
                      }
                  };
  	    //将任务封装为 Future
          Future<List<ImageData>> future = executor.submit(task);
          renderText(source);

        	//尝试 Future.get()
          try {
              List<ImageData> imageData = future.get();
              for (ImageData data : imageData)
                  renderImage(data);
          } catch (InterruptedException e) {
              // 重新设置线程的中断状态
              Thread.currentThread().interrupt();
              // 因为不需要结果，所以取消任务
              future.cancel(true);
          } catch (ExecutionException e) {
              throw launderThrowable(e.getCause());
          }
      }
  }
  ```

* `Future.get()` 拥有“状态依赖”的内在特性，因而**调用者不需要知道任务的状态**，此外在任务提交和获得结果中包含的安全发布属性也确保了**这个方法是线程安全的**。

* 使用**完成服务（CompletionService）**来在计算完成后获取结果：

  * **工作原理**：整合了 `Executor` 和 `BlockingQueue` 的功能，可以将 `Callable` 任务提交给它来执行，然后使用类似于队列操作的 `take` 和 `poll` 等方法来获得已完成的结果。
  * `CompletionService` 可以**从两个方面**来**提高页面渲染器的性能**：**缩短总运行时间**：每个图片的下载从串行变为并行。**提高响应性**：图片在下载完成后立刻显示出来，能使用户获得一个更加动态和更高响应性的用户界面。

* 为 `Future.get()` 方法设置时间限制可以实现**为任务设定时限**。

### 取消与关闭

#### 任务取消

* Java 没有提供任何机制来安全地终止线程，但它提供了**中断（lnterruption）**，这是一种协作机制，能够使一个线程终止另一个线程的当前工作。
* **线程中断**：
  * 每个线程都有一个 `boolean` 类型的中断状态。当中断线程时，这个线程的中断状态将被设置为 `true`。
  * `interrupt` 方法能中断目标线程。
  * `islnterrupted` 方法能返回目标线程的中断状态。
  * 静态的 `interrupted` 方法将清除当前线程的中断状态，并返回它之前的值，这也是**清除中断状态的唯一方法。**
* 中断操作**并不会真正地中断**一个正在运行的线程，而只是发出中断请求，然后由线程在下一个合适的时刻中断自己。（这些时刻也被称为取消点）
* 使用静态的 `interrupted` 时会清除当前线程的中断状态，除非你想屏蔽这个中断，否则必须对它进行处理。
* 通常，中断是实现取消的最合理方式。
* **中断策略**：中断策略规定线程如何解释某个中断请求。当发现中断请求时，应该做哪些工作（如果需要的话），哪些工作单元对于中断来说是原子操作，以及以多快的速度来响应中断。
* 最合理的中断策略是某种形式的**线程级（Thread-Level）取消操作**或**服务级（Service-Level）取消操作**：尽快退出，在必要时进行清理，通知某个所有者该线程已经退出。

#### 停止一个基于线程的服务

* 任务不会在其自己拥有的线程中执行，而是在某个服务（例如线程池）拥有的线程中执行。对于**非线程所有者**的代码来说，**应该小心地保存中断状态**，这样拥有线程的代码才能对中断做出响应。

* **规则：若一个线程的中断策略是未知的，那么不可以中断该线程**。

* 两种策略处理 `InterruptedException` ：传递异常和恢复中断状态。

* 对于**不支持取消但仍可以调用可中断阻塞方法的操作**：它们必须**在循环中调用**这些方法，并在发现中断后重新尝试。它们应该在本地保存中断状态，并在**返回前**（即 `finally` 中）恢复状态而不是在**捕获 `InterruptedException` 时**恢复状态。如果过早地设置中断状态，就可能引起无限循环。

* **在外部线程中断**

  ```java
  // 完整代码：http://jcip.net/listings/TimedRun1.java
  public class TimedRun1 {
      private static final ScheduledExecutorService cancelExec = Executors.newScheduledThreadPool(1);

      public static void timedRun(Runnable r, long timeout, TimeUnit unit) {
          final Thread taskThread = Thread.currentThread();
          // 创建一个 Runnable 来中断 taskThread
          cancelExec.schedule(new Runnable() {
              public void run() {
                  taskThread.interrupt();
              }
          }, timeout, unit);
          r.run();
      }
  }
  ```

  * 缺点：违反规则——若一个线程的中断策略是未知的，那么不可以中断该线程。

* **在专门的线程中中断**

  ```java
  // 完整代码：http://jcip.net/listings/TimedRun2.java
  public class TimedRun2 {
      private static final ScheduledExecutorService cancelExec = newScheduledThreadPool(1);

      public static void timedRun(final Runnable r, long timeout, TimeUnit unit)
              throws InterruptedException {
        	// 为 Runnable 拓展中断策略
          class RethrowableTask implements Runnable {
              private volatile Throwable t;

              public void run() {
                  try {
                      r.run();
                  } catch (Throwable t) {
                      this.t = t;
                  }
              }
  		   // 实现了中断策略：若出现异常则再次抛出该异常
              void rethrow() {
                  if (t != null)
                      throw launderThrowable(t);
              }
          }
  		// 创建已知中断策略的线程
          RethrowableTask task = new RethrowableTask();
          final Thread taskThread = new Thread(task);
        
          taskThread.start();
          cancelExec.schedule(new Runnable() {
              public void run() {
                  taskThread.interrupt();
              }
          }, timeout, unit);
          // join 等待并检查任务中是否抛出异常
          taskThread.join(unit.toMillis(timeout));
          task.rethrow();
      }
  }
  ```

  * [thread.join](https://docs.oracle.com/javase/7/docs/api/java/lang/Thread.html#join(long)) 的不足是：无法知道执行控制是因为线程正常退出而返回还是因为 `join` 超时而返回。

* **通过 `Future` 处理中断实现取消**

  ```java
  public class TimedRun {
      private static final ExecutorService taskExec = Executors.newCachedThreadPool();

      public static void timedRun(Runnable r, long timeout, TimeUnit unit)
              throws InterruptedException {
          Future<?> task = taskExec.submit(r);
          try {
              task.get(timeout, unit);
          } catch (TimeoutException e) {
              // task will be cancelled below
          } catch (ExecutionException e) {
              // exception thrown in task; rethrow
              throw launderThrowable(e.getCause());
          } finally {
              // Harmless if task already completed
              task.cancel(true); // interrupt if running
          }
      }
  }
  ```

* 对于持有线程的服务，只要服务的存在时间大于创建线程的方法的存在时间，那么就应该提供生命周期方法。

* **在日志服务中添加可靠的取消操作**

  * 日志服务本身是个生产者 — 消费者问题，所以取消日志服务时要考虑生产者线程和消费者线程的同步取消。

  ```java
  // 完整代码：http://jcip.net/listings/LogService.java
  public class LogService {
      private final BlockingQueue<String> queue;
      private final LoggerThread loggerThread;
      private final PrintWriter writer;
      @GuardedBy("this") private boolean isShutdown;
      @GuardedBy("this") private int reservations;

      public void start() {
          loggerThread.start();
      }

      public void stop() {
          synchronized (this) {
              isShutdown = true;
          }
          loggerThread.interrupt();
      }
  	// 生产者
      public void log(String msg) throws InterruptedException {
         	// 内置锁同步
          synchronized (this) {
              if (isShutdown)
                  throw new IllegalStateException(/*...*/);
              ++reservations;
          }
          queue.put(msg);
      }
  	// 消费者
      private class LoggerThread extends Thread {
          public void run() {
              try {
                  while (true) {
                      try {
                        	// 内置锁同步
                          synchronized (LogService.this) {
                          	// 被通知中断且没有需要处理的消息
                              if (isShutdown && reservations == 0)
                                  break;
                          }
                          String msg = queue.take();
                          synchronized (LogService.this) {
                              --reservations;
                          }
                          writer.println(msg);
                      } catch (InterruptedException e) { /* retry */
                      }
                  }
              } finally {
                  writer.close();
              }
          }
      }
  }
  ```

* **毒丸（Poison Pill）对象**：另一种关闭生产者 一 消费者服务的方式就是使用毒丸对象。毒丸是指一个放在队列上的对象，其含义是：**当得到这个对象时，立即停止**。

  * 毒丸对象将确保消费者在关闭之前首先完成队列中的所有工作；而生产者在提交了“毒丸”对象后，将不会再提交任何工作。
  * 只有在生产者和消费者的数量都已知的情况下，才可以使用毒丸对象。

* 实现跟踪**被 `shutdownNow` 取消的正在执行**的任务

  ```java
  // 完整代码：http://jcip.net/listings/TrackingExecutor.java
  public void execute(final Runnable runnable) {
          exec.execute(new Runnable() {
              public void run() {
                  try {
                      runnable.run();
                  } finally {
                    	// 这里存在以一个竞态条件，但对于幂等任务来说不会导致正确性问题
                      if (isShutdown() && Thread.currentThread().isInterrupted())
                          tasksCancelledAtShutdown.add(runnable);
                  }
              }
          });
      }
  ```

* 并发程序中一些异常（如 `RuntimeException`）不会被捕捉到，要处理这些异常，一般通过结合“**在工作者线程中主动捕捉异常”**和**“使用 `UncaughtExceptionHandler` ”**这两种方法来实现。

  * `execute` 提交的任务抛出的异常才会交给 `UncaughtExceptionHandler` ，而 `submit `提交的任务则不会。

#### JVM 关闭

* 在**正常关闭**中，JVM 首先调用所有已注册的**关闭钩子（Shutdown Hook）**。当所有的关闭钩子都执行结束时，如果 `runFinalizersOnExit` 为 `true` ，那么 JVM 将运行**终结器**，然后再停止。
* **关闭钩子**：通过 `Runtime.addShutdownHook` 注册的但尚未开始的线程。
  * 关闭钩子应该是线程安全的.
  * 关闭钩子可以用于实现服务或应用程序的清理工作。
  * 关闭钩子不应该依赖那些可能被应用程序或其他关闭钩子关闭的服务。
* **守护线程（Daemon Thread）** 
  * 当希望创建一个线程来执行一些辅助工作，但又不希望这个线程阻碍 JVM的关闭，这种情况下就需要使用守护线程。
  * 当创建一个新线程时，新线程将继承创建它的线程的守护状态，因此在默认情况下，主线程创建的所有线程都是普通线程。
  * 普通线程与守护线程之间的**差异仅在于当线程退出时发生的操作**：当 JVM 停止时，所有仍然存在的守护线程都将被抛弃——既不会执行finally代码块，也不会执行回卷栈，而 JVM 只是直接退出。
  * **应尽可能少地使用守护线程**。
* **终结器**
  * 对于其他一些资源，例如文件句柄或套接字句柄，当不再需要它们时，必须**显式地**交还给操作系统。
  * 在回收器释放它们后，调用它们的 `finalize` 方法，从而保证一些持久化的资源被释放。
  * **避免使用终结器**。

### 线程池的使用

#### 任务和执行策略之间可能存在隐形耦合

* 在使用任务执行框架时，由于在**任务和执行策略之间可能存在隐形耦合**，所以可能存在一些危险，主要考虑这几种任务：
  * 依赖性任务。
  * 使用线程封闭机制的任务。
  * 对响应时间敏感的任务。
  * 使用 `ThreadLocal` 的任务。
* 如果任务依赖于其他任务，那么可能产生**线程饥饿死锁（Thread Starvation Deadlock）**，即任务需要无限期地等待一些必须由其他任务才能提供的资源或条件。
  * 可以通过使用无界的线程池来解决。

#### 设置线程池的大小

* 通常不要固定线程池的大小，而应该通过某种配置机制来提供，或者根据 `Runtime.availableProcessors` 来动态计算。

* **计算线程池的最优大小**

  * 对于 CPU 资源的限制

  ![](../img/in-post/2018-02-08-java-concurrency-learning-2/CPU 资源的限制.png)

  * 对于其他资源的限制：计算每个任务对该资源的需求量，然后用该资源的可用总量除以每个任务的需求量，所得结果就是线程池大小的上限。
  * 线程池和资源池（如：数据连接池）的大小将会相互影响。

#### 使用 ThreadPoolExecutor

* `ThreadPoolExecutor` 是一个灵活的、稳定的线程池，允许各种定制。

  * `corePoolSize` ：**基本大小**，也就是线程池的目标大小，即在没有任务执行时线程池的大小，并且只有在工作队列满了的情况下才会创建超出这个数量的线程。
  * `maximumPoolSize` ：**最大大小**，可同时活动的线程数量的上限。

* **队列任务**：如果新请求的到达速率超过了线程池的处理速率，那么新到来的请求将累积起来。在线程池中，这些请求会在一个由 `Executor` 管理的 `Runnable` 队列中等待，不会竞争CPU资源。

* `ThreadPooIExecutor` 允许提供一个 `BlockingQueue` 来保存等待执行的任务，基本的任务排队方法有3种：**无界队列、有界队列和同步移交（Synchronous Handoff）**。

* 对于有界队列，如果**线程池较小而队列较大**，那么有助于减少内存使用量，降低 CPU 的使用率，同时还可以减少上下文切换，但付出的代价是可能会限制吞吐量。

* 通过使用 `SynchronousQueue` ，可以**避免任务排队**，以及直接将任务从生产者移交给工作者线程。`SynchronousQueue` **不是一个真正的队列**，而是一种在线程之间进行移交的机制。

* **饱和策略**：当有界队列被填满后，饱和策略开始发挥作用。 `ThreadPooIExecutor` 的饱和策略可以通过调用 `setRejectedExecutionHandler` 来修改。

  * `AbortPolicy`：中止，默认的饱和策略，抛出未检查的 `RejectedExecutionException` 。
  * `DiscardPolicy`：抛弃，且不抛出异常。
  * `DiscardOldestPolicy`：抛弃最旧，且不抛出异常。
  * `CallerRunsPolicy`：调用者运行，不会抛弃任务，也不会抛出异常，而是将某些任务回退到调用者，从而降低新任务的流量。

* **使用调用者运行策略的优点**：当服务器过载，这种过载情况会逐渐**向外蔓延**开来——从线程池到工作队列到应用程序再到TCP层，最终达到客户端，导致服务器在高负载下**实现一种平缓的性能降低**。

* 通过指定一个线程工厂方法，可以定制线程池的配置信息。

  ```java
  // 完整代码：http://jcip.net/listings/MyThreadFactory.java
  public class MyThreadFactory implements ThreadFactory {
      private final String poolName;

      public MyThreadFactory(String poolName) {
          this.poolName = poolName;
      }

      public Thread newThread(Runnable runnable) {
          return new MyAppThread(runnable, poolName);
      }
  }
  ```

* 在调用完 `ThreadPooIExecutor` 的构造函数后，仍然可以通过**设置函数（Setter）**来修改大多数传递给它的构造函数的参数。

  * 使用 `unconfigurableExecutorService` 来包装可以防止通过 Setter 修改。

* 通过改写 `beforeExecute` 和 `afterExecute` 方法，可以添加日志、计时、监视或统计信息收集的功能；改写 `terminated` 可以用来释放 `Executor` 在其生命周期里分配的各种资源，以及实现执行发送通知、记录日志或者收集finalize统计信息等操作。

#### 迭代算法的并行化

* **迭代并行化的条件**：在循环体中，每一次迭代是独立的；更进一步，只有当每个迭代操作执行的工作量大于管理一个新线程的开销，并行化才有意义。

* **谜题框架**

  * “谜题”**定义**为：包含了一个初始位置，一个目标位置，以及用于判断是否是有效移动的规则集。
  * 并发的谜题解答器：

  ```java
  // 完整代码：http://jcip.net/listings/ConcurrentPuzzleSolver.java
  public class ConcurrentPuzzleSolver <P, M> {
      private final Puzzle<P, M> puzzle;
      private final ExecutorService exec;
      private final ConcurrentMap<P, Boolean> seen;
      protected final ValueLatch<PuzzleNode<P, M>> solution = new ValueLatch<PuzzleNode<P, M>>();
      
      public List<M> solve() throws InterruptedException {
          try {
              P p = puzzle.initialPosition();
              exec.execute(newTask(p, null, null));
              // 阻塞直到找到解答
              PuzzleNode<P, M> solnPuzzleNode = solution.getValue();
              return (solnPuzzleNode == null) ? null : solnPuzzleNode.asMoveList();
          } finally {
              exec.shutdown();
          }
      }

      protected Runnable newTask(P p, M m, PuzzleNode<P, M> n) {
          return new SolverTask(p, m, n);
      }

      protected class SolverTask extends PuzzleNode<P, M> implements Runnable {
          public void run() {
              if (solution.isSet()
                      || seen.putIfAbsent(pos, true) != null)
                  return; // 找到解答或者已完全遍历
              if (puzzle.isGoal(pos))
                  solution.setValue(this);
              else
                  for (M m : puzzle.legalMoves(pos))
                      exec.execute(newTask(puzzle.move(pos, m), m, this));
          }
      }
  }
  ```

  * 使用闭锁，可以检测是否有线程已经找到一个解答。

### 用户图形界面应用程序

* **GUI 是单线程的原因**：多线程的 GUI 容易发生死锁问题。
  1. 多线程下，输入事件的处理过程（气泡上升）与GUI组件的面向对象模型（气泡下降）之间会存在错误的交互。
  2. MVC 设计模式进一步增加了出现不一致锁定顺序的风险。
* 单线程的 GUI 框架通过**线程封闭机制**来实现线程安全性。

#### 处理 GUI 任务

* 对于**短时间且只访问 GUI 对象**的 GUI 任务，基本上所有操作都可以直接在事件线程中进行。
* 对于**长时间以及复杂**的 GUI 任务，这些任务必须在另外的线程中运行以保存 GUI 的高响应性。同时，可能还有一些另外的实现需求，例如：
  1. [取消任务](http://jcip.net/listings/ListenerExamples.java)
  2. [进度标识、完成标识](http://jcip.net/listings/BackgroundTask.java)

#### 共享数据模型

* 简单情况下，其他线程无法访问数据模型，GUI 应用程序只需简单地静态加载数据模型；但在一些情况下，数据在 GUI 程序进出时由多个线程共享。这种情况下，有两种解决数据同步性的方法：
  1. 实现线程安全的数据模型：但要构建一个既能提供高效的并发访问又能在旧数据无效后不再维护它们的数据结构却并不容易。
  2. 采用**分解数据模型的设计模式**。
* 分解模型的设计模式中，**表现模型**被封闭在事件线程中，而其他模型，即**共享模型**，是线程安全的。
  * **表现模型可以在共享模型中得到更新**：通过将相关状态的快照嵌入到更新消息中，或者由表现模型在收到更新事件时直接从共享模型中获取数据。
  * 通过快照实现更新，虽然简单，但是当数据模型很大或更新频率高时效率低下。采取**增量更新**可以解决这个问题。并且，增量更新还能带来细粒度的变化信息，从而**优化 GUI 的视觉效果**。
* 当某个工具必须被实现为单线程子系统是，可以借鉴 GUI 的实现，采取线程封闭来实现。



