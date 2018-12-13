---
layout:     post
title:      "Java 沉思录(一)：自动装箱实现细节"
subtitle:   "装箱时 Java 做了什么？"
date:       2017-11-02 02:50:00
author:     "Damon To"
header-img: "img/ruminations-java.jpg"
header-mask: 0.6
catalog:    true
tags:
    - Java
---

> 这是\<Java 沉思录>系列的第一篇文章。之所以叫\<Java 沉思录>，其实是仿照\<C++沉思录>的命名。后者几乎是我第一次读到的能够令人激动和恍然大悟的技术书籍，让当时还是编程语言初学者的我印象深刻。虽然本系列大概无法匹及那种高度和深度，但主要动机是相似的，即：记录开发过程中可能会遇到但经常被忽略的一些技术细节与语言陷阱。第一篇文章，从基础的自动装箱谈起。

### (一) 自动装箱

Java 的自动装箱为我们提供从基本值类型到引用类型的便捷转化方法：

```java
Integer a = new Integer(1000);//无自动装箱
Integer b = 1000;//自动装箱
```

### (二) 一个陷阱

考虑如下代码：

```java
public class Test {
    public static void main(String[] args) {
        Integer a = 10; //自动装箱生成对象a
        Integer b = 10; //自动装箱生成对象b
        System.out.println("a==b : " + String.valueOf(a==b));
        System.out.println("a.equals(b) : " + String.valueOf(a.equals(b)));
    }
}
```

**这段代码的输出会是什么？**

Java 初学者应该都知道如下的事实：

> 比较两个对象时：
>
> 使用`==`：比较**内存地址**是否相等
>
> 使用`equals()`：比较**值**是否相等

这也是我们在进行引用类型对象之间的比较时，常常使用`equals()`而不是`==`的原因，我们一般关注两个对象的值之间的关系而非内存地址之间的比较。

按照这个事实，我们认为 a 与 b 是两个不同的对象，所以他们有不同的内存地址；又由于他们的值相同，所以我们自然而然地预期如下输出：

```
a==b : false
a.equals(b) : true
```

然而运行代码，**真正的输出有些出乎意料**：

```
a==b : true
a.equals(b) : true
```

**这是为什么呢？**

### (三) 查看源码：自动装箱时 Java 做了什么？

对于以上的输出，我们可以有一个直接的猜想：**在进行自动装箱时 Java 通过某种机制，让 a 与 b 这两个引用实例指向了相同的一段内存空间。**然而这仅仅是我们的猜测，我们可以从 Java 源码中找到真正的原因。

在这之前，首先我们需要知道一个事实：自动装箱是基于函数`valueOf()`实现的，即：

```java
//下面两个语句等价：
Integer a = 10;
Integer a = Integer.valueOf(10);
```

因此我们查看 JDK 中`Integer.valueOf()`的代码实现：

```java
//jdk edition：6-b14
public static Integer valueOf(int i) {
	final int offset = 128;
	if (i >= -128 && i <= 127) { // must cache
		return IntegerCache.cache[i + offset];
	}
	return new Integer(i);
}
```

这段代码的意思很好理解：当参数 i 落在区间[-128, 127] 时，返回值为`IntegerCache.cache[i + offset]`，否则才返回`Integer(i)`。**这意味着并不是每次自动装箱都会创建一个新的`Integer`实例。**但是，Java 如何实现两个引用示例指向相同地址空间呢？这与`IntegerCache`类有关：

```java
//jdk edition：6-b14
private static class IntegerCache {
	private IntegerCache(){}

	static final Integer cache[] = new Integer[-(-128) + 127 + 1];

	static {
		for(int i = 0; i < cache.length; i++)
		cache[i] = new Integer(i - 128);
	}
}
```

可以看出`IntegerCache`内部维护了一个静态数组作为缓存，当我们自动装箱一个值在[-128, 127]之间的`int`时，它将直接返回静态数组里的对应值，这也就是为什么上面的 a 与 b 会共用相同的内存地址，因为从底层看，他们其实是**同一个对象**。

### (四) 进一步思考

**为什么 Java 要这样设计？带来的好处是什么？**

`IntegerCache`及其背后的缓存机制，是从 Java 5 开始引入的一种缓存策略。显然，[-128, 127]区间之间的值是在编程中高频出现的值，引入这种缓存机制意味着在多次使用中避免了多次重复的创建与销毁对象所带来的开销，本质上是**通过“复用”来节省内存，提高运行速度。**

**缺陷与后续版本中的优化**

考虑上述代码中的这部分代码：

```java
if (i >= -128 && i <= 127) { // must cache
	return IntegerCache.cache[i + offset];
}
```

这样的代码直接其实存在一个 Magic Number 的问题，即直接将 -128，127这样的数字嵌入代码，这样的代码其实是 bad smell 的。其实在 Java 7 以后的版本以及解决了这个问题：

```java
private static class IntegerCache {
	static final int low = -128;
	static final int high;
	static final Integer cache[];
 
	static {
	// high value may be configured by property
	int h = 127;
	String integerCacheHighPropValue =
           sun.misc.VM.getSavedProperty("java.lang.Integer.IntegerCache.high");
	if (integerCacheHighPropValue != null) {
		try {
			int i = parseInt(integerCacheHighPropValue);
			i = Math.max(i, 127);
			// Maximum array size is Integer.MAX_VALUE
			h = Math.min(i, Integer.MAX_VALUE - (-low) -1);
        } catch( NumberFormatException nfe) {
		// If the property cannot be parsed into an int, ignore it.
		}
	}
	high = h;
 
	cache = new Integer[(high - low) + 1];
	int j = low;
	for(int k = 0; k &amp;lt; cache.length; k++)
		cache[k] = new Integer(j++);
 
		// range [-128, 127] must be interned (JLS7 5.1.7)
		assert IntegerCache.high &amp;gt;= 127;
	}
	private IntegerCache() {}
}
```

需要注意的是默认区间仍然是[-128, 127]，但是最大值可以通过设定 JVM 的启动参数来修改，以达到修改缓存空间大小的目的。

**另外的一些类的自动装箱也存在缓存机制**

在自动装箱的过程中，除了`Integer`，`Byte`，`Short`，`Long`，`Character`也存在缓存机制(缓存区间及大小不尽相同)，而`Double`，`Float`则没有对应的缓存机制。

### (五) 小结

对于严格的对象处理(例如要维护一个`Integer`的哈希映射时)，我们要谨慎使用自动装箱，因为使用自动装箱往往会导致我们会从字面上认为我们创建了多个实例，而实际上他们使用同一个实例。在这种情况下，更好的选择是使用传统的构造器来创建对象。

------

*参考网站：*

1. *Grepcode：http://grepcode.com/*
2. *理解 Java Integer的缓存策略：http://www.importnew.com/18884.html*
3. *How large is the Integer cache? - Stackoverflow：https://stackoverflow.com/questions/15052216/how-large-is-the-integer-cache*