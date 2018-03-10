---
layout:     post
title:      "JDK 源码阅读(一)：理解 ThreadLocal"
subtitle:   "Java 的线程封闭技术"
date:       2018-02-25 22:40:00
author:     "Damon To"
header-img: "img/jdk-threadlocal.jpg"
header-mask: 0.4
catalog:    true
tags:
    - Java
    - JDK
    - ThreadLocal
    - Concurrency
---

> ThreadLocal 是 Java 并发领域里一个十分重要的类，本文将深入 ThreadLocal 这个类的具体含义与细节实现（基于 JDK8）。如果你之前没有接触或使用过 ThreadLocal ，那么推荐你先简单了解一下 ThreadLocal 的使用（推荐 @vigarbuaa 的 [Java ThreadLocal示例及使用方法总结](http://www.cnblogs.com/vigarbuaa/archive/2012/03/01/2375149.html) 这篇博客），以及你需要了解一些关于 Java 并发的基础知识（推荐我的 [Java 并发学习笔记 (一)：并发基础知识](http://splitmusic.cn/2018/02/05/java-concurrency-learning-1/) 一文）。希望本文对你有所帮助。

### (一) ThreadLocal 的定义与作用 

学习 `ThreadLocal` 之前，最好明晰它处于 Java 开发体系的什么位置。`ThreadLocal` 类定义在 java / lang 中，但是其主要的应用是在于并发领域。[《 Java 并发编程实战》](https://book.douban.com/subject/10484692/)中提到：

> 处理线程安全性问题的三种主要方式：
>
> 1. 不在线程之间共享该状态变量
> 2. 将状态变量设置为不可变
> 3. 在访问状态变量时使用同步

`ThreadLocal` 在应该被归类于第一类技术：即通过**线程封闭（Thread Confinement）**技术，使得状态变量与特定的线程”绑定“起来（不在线程之间共享该状态变量），从而保证在多线程下的线程安全性。[《 Java 并发编程实战》](https://book.douban.com/subject/10484692/)将线程封闭技术分为三种：Ad-hoc 线程封闭、栈封闭以及 `ThreadLocal` 。其中 `ThreadLocal` 是由 JDK 提供的、最易实现和维护且最为健壮的线程封闭技术。（具体请看 [Java 并发学习笔记 (一)：并发基础知识](http://splitmusic.cn/2018/02/05/java-concurrency-learning-1/) 一文）

### (二) 对 ThreadLocal 的整体了解与实现猜想

在了解 `ThreadLocal` 的具体实现之前，我们可以先通过对 `ThreadLocal` 源码结构的总体把握来大概了解 `ThreadLocal` 实现线程封闭的基本原理：

<img src="http://ompnv884d.bkt.clouddn.com/ThreadLocal-structure.jpg" width="400px">

 `ThreadLocal` 的作者在注释中如此介绍  `ThreadLocal` 的工作原理：

> This class provides thread-local variables.  These variables differ from their normal counterparts in that each thread that accesses one (via its {@code get} or {@code set} method) has its own, independently initialized copy of the variable.  {@code ThreadLocal} instances are typically private static fields in classes that wish to associate state with a thread (e.g., a user ID or Transaction ID).

大概告诉了我们几点信息：

-  `ThreadLocal` 为线程提供了一种特殊的”线程内部变量“（thread-local variables）。
-  线程通过 `ThreadLocal` 的 `get()` 和 `set()` 方法来获取一个变量副本，这个变量副本为该线程所独有。
-  `ThreadLocal` 实例最好是一个 private static 修饰的字段。

 结合上面 `ThreadLocal`  的结构中，有一个名为 `ThreadLocalMap` 的内部类，**我们可以大胆地对 `ThreadLocal` 的实现进行如下猜想：**

`ThreadLocal` 维护一个 `ThreadLocalMap` 的 key 为每个线程的 ThreadId，value 为我们需要封闭的线程内部变量。每次调用其 `get()` 和 `set()` ，以当前线程的 ThreadId 为 key 从  `ThreadLocalMap` 中定位到其对应的线程内部变量，从而实现不同线程之间的隔离。

这种想法十分直接，事实上 JDK 早期的版本也是如此进行实现的，但是**这种实现方法现在已经被摒弃**。以下，我们尝试通过阅读 jdk8 源码中 `ThreadLocal` 的实现来了解这种实现方式为何被摒弃。

### (三) ThreadLocal 的实现细节

#### Thread、ThreadLocal、ThreadLocalMap 的关系

 `ThreadLocal` 的一般使用步骤为：new 一个 `ThreadLocal` ，调用 `set()` 方法设置线程内部变量，需要变量时调用 `get()` 方法获取变量。那么我们就根据这个使用步骤来看看此时 `ThreadLocal`  内部的运作。

```java
/**
* Creates a thread local variable.
* @see #withInitial(java.util.function.Supplier)
*/
public ThreadLocal() {
}
```

 `ThreadLocal` 只是简单的创建实例，没有做额外的初始化操作。

```java
/**
* Sets the current thread's copy of this thread-local variable
* to the specified value.  Most subclasses will have no need to
* override this method, relying solely on the {@link #initialValue}
* method to set the values of thread-locals.
*
* @param value the value to be stored in the current thread's copy of
*        this thread-local.
*/
public void set(T value) {
    // 获取当前线程
    Thread t = Thread.currentThread();
    // 获取当前线程所维护的 ThreadLocalMap
    ThreadLocalMap map = getMap(t);
    if (map != null)
        // map 非空则取对象
    	map.set(this, value);
    else
       	// map 为空则创建 map 并进行相应的初始化
    	createMap(t, value);
}
```

在 `set()` 方法的实现中我们已经可以看到这和我们之前的猜想不尽相同。在 JDK8 中， `ThreadLocal` 的基本实现思路是：每个 `Thread` 维护一个 `ThreadLocalMap` 实例， `ThreadLocalMap` 的 key 是 `ThreadLocal` 实例，value 是线程内部变量。访问线程内部变量时，先获取当前线程对应的  `ThreadLocalMap` ，再在该 `ThreadLocalMap`  中获取 `ThreadLocal` 对应的内部变量。

**这种实现方式有何优点呢？** 最大的优点应该是：当线程结束时， `ThreadLocalMap` 也会随之销毁，以避免对内存的占用。

上述代码除了说明了 `ThreadLocal` 的基本实现，还说明了一个细节：每个 `Thread` 所对应的 `ThreadLocalMap` 是有可能为空的，也就是说线程不会在创建时就直接创建一个 `ThreadLocalMap` 并与之绑定，而是**采取 JIT 策略，在需要使用 `ThreadLocalMap` 之前才绑定。**

#### ThreadLocalMap 的创建

这里继续看 `createMap()` 的具体实现和 `ThreadLocalMap` 的创建。

```java
/**
* Create the map associated with a ThreadLocal. Overridden in
* InheritableThreadLocal.
*
* @param t the current thread
* @param firstValue value for the initial entry of the map
*/
void createMap(Thread t, T firstValue) {
    // 调用 ThreadLocalMap 的构造函数
	t.threadLocals = new ThreadLocalMap(this, firstValue);
}
```

```java
static class ThreadLocalMap {
    
    //...省略了部分无关代码
    
    /**
    * Construct a new map initially containing (firstKey, firstValue).
    * ThreadLocalMaps are constructed lazily, so we only create
    * one when we have at least one entry to put in it.
    */
    ThreadLocalMap(ThreadLocal<?> firstKey, Object firstValue) {
        // map 的内部实际为一个 Entry 类型的数组，常量 INITIAL_CAPACITY 默认为 16
        table = new Entry[INITIAL_CAPACITY];
        // threadLocalHashCode 是优化后的哈希散列值，减少了碰撞的发生；&运算使得 i 落在 0~15 之间
        int i = firstKey.threadLocalHashCode & (INITIAL_CAPACITY - 1);
        table[i] = new Entry(firstKey, firstValue);
        size = 1;
        setThreshold(INITIAL_CAPACITY);
    }
    
    /**
         * Set the resize threshold to maintain at worst a 2/3 load factor.
         */
    private void setThreshold(int len) {
        threshold = len * 2 / 3;
    }
}
```

**读到这里，我们会有两个疑问：**

1.  `threadLocalHashCode` 是什么？它对哈希表进行了哪些优化？
2.  如果 `size` 大于 `threshold` 时，如何处理？

接下来的两部分我们分别关注这里提出的两个问题。

#### threadLocalHashCode 对哈希表进行的优化

我们看回 `ThreadLocal` ：

```java
/**
     * ThreadLocals rely on per-thread linear-probe hash maps attached
     * to each thread (Thread.threadLocals and
     * inheritableThreadLocals).  The ThreadLocal objects act as keys,
     * searched via threadLocalHashCode.  This is a custom hash code
     * (useful only within ThreadLocalMaps) that eliminates collisions
     * in the common case where consecutively constructed ThreadLocals
     * are used by the same threads, while remaining well-behaved in
     * less common cases.
     */
private final int threadLocalHashCode = nextHashCode();

/**
     * The next hash code to be given out. Updated atomically. Starts at
     * zero.
     */
private static AtomicInteger nextHashCode =
    new AtomicInteger();

/**
     * The difference between successively generated hash codes - turns
     * implicit sequential thread-local IDs into near-optimally spread
     * multiplicative hash values for power-of-two-sized tables.
     */
private static final int HASH_INCREMENT = 0x61c88647;

/**
     * Returns the next hash code.
     */
private static int nextHashCode() {
    return nextHashCode.getAndAdd(HASH_INCREMENT);
}
```

`threadLocalHashCode` 以 `HASH_INCREMENT` 为增量，产生一组从 0、1 * HASH_INCREMENT、2 * HASH_INCREMENT...到 n * HASH_INCREMENT 的哈希散列值。在这里，HASH_INCREMENT 被设置为一个很特殊的十六进制数——「0x61c88647」。以这个数形成的散列称为斐波那契散列（Fibonacci Hashing）。斐波那契散列使得哈希表分布更加均匀，极大地减少了碰撞的发生，在实际运用中往往对效率有很大的提升。

#### ThreadLocalMap 的扩容

如果 `size` 大于 `threshold` 时，最基本的想法是直接对 table 进行扩容，一般的做法就是 new 一个两倍大小的 table 并将原 table deep copy 到新的 table，`ThreadLocalMap`  提供了 `resize()` 方法：

```java
/**
         * Double the capacity of the table.
         */
private void resize() {
    Entry[] oldTab = table;
    int oldLen = oldTab.length;
    int newLen = oldLen * 2;
    Entry[] newTab = new Entry[newLen];
    int count = 0;

    // deep copy
    for (int j = 0; j < oldLen; ++j) {
        Entry e = oldTab[j];
        if (e != null) {
            ThreadLocal<?> k = e.get();
            if (k == null) {
                e.value = null; // Help the GC
            } else {
                int h = k.threadLocalHashCode & (newLen - 1);
                while (newTab[h] != null)
                    h = nextIndex(h, newLen);
                newTab[h] = e;
                count++;
            }
        }
    }

    setThreshold(newLen);
    size = count;
    table = newTab;
}

```

然而，更关键的问题在于：进行 `resize()` 的条件是什么？

相比每次一达到 `threshold` 就进行 `resize()` ，如果我们对 `resize()` 的条件加以限制，尽可能地减少 `resize()` 的次数，则可以尽可能地优化性能。所以我们不妨看看 `set()` 是如何对这一条件进行限制的。

```java
/**
         * Set the value associated with key.
         *
         * @param key the thread local object
         * @param value the value to be set
         */
private void set(ThreadLocal<?> key, Object value) {

    // We don't use a fast path as with get() because it is at
    // least as common to use set() to create new entries as
    // it is to replace existing ones, in which case, a fast
    // path would fail more often than not.

    Entry[] tab = table;
    int len = tab.length;
    int i = key.threadLocalHashCode & (len-1);

    for (Entry e = tab[i];
         e != null;
         // 发生哈希碰撞，则储存在下一个 entry 中
         e = tab[i = nextIndex(i, len)]) {
        ThreadLocal<?> k = e.get();

        if (k == key) {
            e.value = value;
            return;
        }

        if (k == null) {
            replaceStaleEntry(key, value, i);
            return;
        }
    }

    tab[i] = new Entry(key, value);
    int sz = ++size;
    // 需要满足 cleanSomeSlots(i, sz) 为 fales 且 sz >= threshold 才进行 rehash()
    if (!cleanSomeSlots(i, sz) && sz >= threshold)
        // rehash() 方法调用了 resize()
        rehash();
}
```

`cleanSomeSlots(i, n)` 的主要作用是在哈希表中找到过期的项（stale entry）并将其擦除，以避免表空间被这些过期的项占用。stale entry 的产生很自然，当我们所保护的线程内部变量使用完毕，生命周期结束后则会被 GC 掉，这些 value 已被清除的 entry 则为 stale entry。

需要注意的是在 `cleanSomeSlots()` 的实现（如下代码）中，出于对性能和时间复杂度的平衡， `cleanSomeSlots()` 并不是扫描 n 个 entry，而是扫描 log2(n) 个。虽然无法找出所有的 stale entry，但将时间复杂度从O(n) 降为 O(log2(n))。在实践中，这种平衡策略能提高效率，而且往往是够用的。

另外，当 `set()` 调用  `cleanSomeSlots()`  时，参数 n 传入的是 `size` （即 entry 的个数）；如果在扫描过程中找到了 stale entry，n 的值会被 `cleanSomeSlots()` 为修改为 `length` （即 table 的总容量），即此时 `cleanSomeSlots()` 的扫描范围被扩大为整个表。采取这种实现的原因代码作者没有在注释中解释，但是从我的角度理解：这是一种从统计角度的优化。从 JVM 执行 GC 的角度来看，当 `ThreadLocalMap` 中发现了一个项因被 GC 而过期，那么此时往往存在多个项也是过期的，所以在此时扩大扫描范围，有更大的概率找到更多的 stale entry ，从而提高 `cleanSomeSlots()` 的效率。

```java
/**
         * Heuristically scan some cells looking for stale entries.
         * This is invoked when either a new element is added, or
         * another stale one has been expunged. It performs a
         * logarithmic number of scans, as a balance between no
         * scanning (fast but retains garbage) and a number of scans
         * proportional to number of elements, that would find all
         * garbage but would cause some insertions to take O(n) time.
         *
         * @param i a position known NOT to hold a stale entry. The
         * scan starts at the element after i.
         *
         * @param n scan control: {@code log2(n)} cells are scanned,
         * unless a stale entry is found, in which case
         * {@code log2(table.length)-1} additional cells are scanned.
         * When called from insertions, this parameter is the number
         * of elements, but when from replaceStaleEntry, it is the
         * table length. (Note: all this could be changed to be either
         * more or less aggressive by weighting n instead of just
         * using straight log n. But this version is simple, fast, and
         * seems to work well.)
         *
         * @return true if any stale entries have been removed.
         */
private boolean cleanSomeSlots(int i, int n) {
    boolean removed = false;
    Entry[] tab = table;
    int len = tab.length;
    do {
        i = nextIndex(i, len);
        Entry e = tab[i];
        // 符合该 if 条件的则为 stale entry，将其擦除
        if (e != null && e.get() == null) {
            n = len;
            removed = true;
            i = expungeStaleEntry(i);
        }
    // 无符号右移一位并赋值，所以最终循环 log2(n) 次
    } while ( (n >>>= 1) != 0);
    return removed;
}

```

除了在 `set()` 限制进行 `resize()` 的条件，在 `set()` 通过 `rehash()` 调用  `resize()` 过程中， `rehash()` 又做了进一步地限制：

```java
/**
         * Re-pack and/or re-size the table. First scan the entire
         * table removing stale entries. If this doesn't sufficiently
         * shrink the size of the table, double the table size.
         */
private void rehash() {
    // expunge all stale entry
    expungeStaleEntries();

    // Use lower threshold for doubling to avoid hysteresis
    if (size >= threshold - threshold / 4)
        resize();
}
```

`rehash()` 进行限制的原理与 `set()` 的实现异曲同工，但扫描程度更完全。之前说过 `cleanSomeSlots()` 只会扫描范围内的 log2(n) 个 entry，而 `expungeStaleEntries()` 会扫描范围内所有的 entry。**总之，`ThreadLocalMap` 通过种种手段，保证了只有当哈希表中完全没有过期的项时才有可能进行扩容。**

#### 总结 ThreadLocal 的实现

至此，我们从 `TreadLocal.set()` 开始，了解了 `TreadLocalMap()` 、 `TreadLocalMap.set()` 、 `cleanSomeSlots()` 、 `expungeStaleEntries()`  ，基本理清了从创建 `ThreadLocal` 到进行 `set()` 时 `ThreadLocal` 的内部实现，包括了：Thread、TreadLocal、TreadLocalMap 三者的关系；TreadLocalMap 的创建；TreadLocalMap 为减少碰撞对哈希表的优化；TreadLocalMap 的扩容和对应的优化。

至于 `get()` 的理解相信在理解了 `set()` 之后就相对比较简单，所以这里就不再赘述。

###(四) ThreadLocal 会导致内存泄漏？

有一些观点认为，使用 `ThreadLocal` 可能会导致内存泄漏，原因基于以下的事实：

在 `ThreadLocalMap` 的实现中， `Entry` 继承了 `WeakReference`（即弱引用），如下：

```java
/**
         * The entries in this hash map extend WeakReference, using
         * its main ref field as the key (which is always a
         * ThreadLocal object).  Note that null keys (i.e. entry.get()
         * == null) mean that the key is no longer referenced, so the
         * entry can be expunged from table.  Such entries are referred to
         * as "stale entries" in the code that follows.
         */
static class Entry extends WeakReference<ThreadLocal<?>> {
    /** The value associated with this ThreadLocal. */
    Object value;

    Entry(ThreadLocal<?> k, Object v) {
        super(k);
        value = v;
    }
}
```

用实线代表强引用，虚线代表弱引用，那么使用 ThreadLocal 时对象在 JVM 内存中如下分布：

<img src="http://ompnv884d.bkt.clouddn.com/ThreadLocal-WeakRef.png">

假设强引用不存在了，那么 GC 时 `ThreadLocal` 对象由于只有弱引用所以会被回收，那么 key 就会变为 `null` ，意味着其对应的 value 将永远无法被调用到，导致内存泄漏。然而 JDK 设计者也意识到这个问题，所以也采取了一定的措施避免这种问题出现。例如在 `resize()` 中有这样一段代码：

```java
if (k == null) {
    e.value = null; // Help the GC
} 
```

类似的代码不止一处，其实在 `ThreadLocalMap` 中，只要涉及到扫描 Entry 的方法，都会检查其 key 是否为空，如若为空都会及时清除 value 以避免内存泄漏。类似的代码几乎遍布于 `ThreadLocalMap` 的方法实现之中（感兴趣的可以找来源码看看有多少方法加入了类似的代码）。所以说，内存泄漏在实际使用 `ThreadLocal` 时虽有可能出现，但我们也不用对此过于担心。同时，为了最大化的保证安全性，我们也可以在调用 `remove()` 来保证当前线程的所有内部变量都被销毁。

### (五) 总结

了解完 `ThreadLocal` 的实现，其实我们发现：本质上， `ThreadLocal` 类是通过 OOP 的封装思想，实现了变量的线程封闭。虽然思想是简单直接的，但是具体实现中却是有很多细节上的心思：例如 Thread、ThreadLocal、ThreadLocalMap 三个类关系的处理，例如对扩容操作的逐步限制以优化性能，例如对内存泄漏的避免....这些都很值得我们在实际开发中学习与借鉴。

------

*参考资料：*

1. *JAVA 并发 - 自问自答学 ThreadLocal：[https://juejin.im/post/5a0e985df265da430e4ebb92](https://juejin.im/post/5a0e985df265da430e4ebb92)*
2. *ThreadLocal源码分析解密  - By 谢照东：[http://xiezhaodong.me/2016/03/05/ThreadLocal%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90%E8%A7%A3%E5%AF%86/](http://xiezhaodong.me/2016/03/05/ThreadLocal%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90%E8%A7%A3%E5%AF%86/)*
3. *ThreadLocal和 synchronized的区别 ? - 知乎用户的回答 - 知乎：[https://www.zhihu.com/question/23089780/answer/62097840](https://www.zhihu.com/question/23089780/answer/62097840)*
4. *Java ThreadLocal 示例及使用方法总结：[http://www.cnblogs.com/vigarbuaa/archive/2012/03/01/2375149.html](http://www.cnblogs.com/vigarbuaa/archive/2012/03/01/2375149.html)*
5. *Github - lambdalab-mirror/jdk8u-jdk：[https://github.com/lambdalab-mirror/jdk8u-jdk/](https://github.com/lambdalab-mirror/jdk8u-jdk/)*

