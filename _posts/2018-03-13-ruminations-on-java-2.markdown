---
layout:     post
title:      "Java 沉思录(二)：关于 String"
subtitle:   "从源码角度分析"
date:       2018-03-13 03:25:00
author:     "Damon To"
header-img: "img/ruminations-java.jpg"
header-mask: 0.6
catalog:    true
tags:
    - Java
    - JDK
---

> 很长时间没有写「Java 沉思录」这个系列。前段时间学习一些比较进阶一点的 Java 知识，但回头看看最基础的 String 类，越发觉得往往越基础的知识反而越重要。这篇文章基于 JDK 源码出发，希望能纠正我们对 String 类的一些误解。同时强烈地推荐来自 ImportNew 社区的这篇[《探秘 Java 中 String、StringBuilder以及StringBuffer》](http://www.importnew.com/18167.html)，文章由浅入深，并且结合了一些面试中和应用中的实际问题，是一篇很好的技术博客。

### (一) 为什么说 String 是不可变的？

Java 中 String 是不可变的。这句话几乎是每个 Java 初学者第一句深刻记忆的句子，但是其背后的原因是什么。我们尝试从 JDK 源码出发进行分析，JDK 中 String 的部分源码如下：

```java
public final class String
    implements java.io.Serializable, Comparable<String>, CharSequence {
    /** The value is used for character storage. */
    private final char value[];

    /** Cache the hash code for the string */
    private int hash; // Default to 0

    /** use serialVersionUID from JDK 1.0.2 for interoperability */
    private static final long serialVersionUID = -6849794470754667710L;

    /** Class String is special cased within the Serialization Stream Protocol. */
    private static final ObjectStreamField[] serialPersistentFields =
        new ObjectStreamField[0];
        
    ...

}
```

“String 是不可变的” 这个观点是正确的，但**关于其原因的解释却容易有以下这些误区**：

1. 错误一：因为 String 是 `final` 的。`final`  修饰类时代表该类不可以被继承，而非不可变。
2. 错误二：因为 `value` 字段是 `fina` 的。`final` 修饰字段时，若字段是基础类型则意味着字段不可变；而在这里修饰的 `value` 字段是引用类型，这意味着引用是固定的，但其引用的具体内容（即数组内容）仍是可变的。

其实如果**我们只从 String 的字段角度看，是无法确定它是不可变的。我们需要结合 String 的方法来看。**我们挑出 String 其中一个方法的实现：

```java
public String replace(char oldChar, char newChar) {
    if (oldChar != newChar) {
        int len = value.length;
        int i = -1;
        char[] val = value; /* avoid getfield opcode */

        while (++i < len) {
            if (val[i] == oldChar) {
                break;
            }
        }
        if (i < len) {
            char buf[] = new char[len];
            for (int j = 0; j < i; j++) {
                buf[j] = val[j];
            }
            while (i < len) {
                char c = val[i];
                buf[i] = (c == oldChar) ? newChar : c;
                i++;
            }
            // 返回一个新的字符串
            return new String(buf, true);
        }
    }
    return this;
}
```

`replace` 方法是新手最容易错误使用的方法之一。它的作用是返回一个将原字符串中所有为 `oldChar` 的字符替换为 `newChar` 的字符。新手如果忘记 String 是不可变的，往往会认为 `replace` 方法会对原字符串修改。然而实际上 `replace` 并不修改原字符串，它只是**返回了一个进行了字符替换的新的字符串**。

无独有偶，其实 **String 中的每个方法都不修改原字符串，而是返回一个新的字符串。**

既然 **String 类本身没有提供修改 `value` 的方法，而我们又无法继承和改写 String，所以 String 自然是不可变的。**（这里我们不考虑通过反射来进行修改的情况）

### (二) String 与字符串常量重用

String 的不可变性使得字符串重用的实现变得简单，而字符串重用带来的是减少对象创建带来的性能上的优化。例如下面代码：

```java
String str1 = "str";
String str2 = "str";
```

Java 字符串常量的重用，与之前我们在 [深入理解Java虚拟机(二)：虚拟机执行子系统](http://splitmusic.cn/2017/12/17/java-virtual-machine-2/) 中讨论的类加载机制密不可分：

>1. Class 文件的常量池（称为 Class 文件常量池）存放着字面量与符号引用
>2. 运行时 JVM 方法区的常量池（称为运行时常量池）存放对应的字面量与符号引用
>3. 类加载时，JVM 将字节流所代表的静态存储结构转化为方法区的运行时数据结构。其中就包括了将 Class 文件常量池的字面量和符号引用加载到方法区的运行时常量池中。

JVM 加载上面的两个语句所在的类时，会有如下现象：

1. Class 文件常量池存放两个 ”str“ 字面量
2. 运行时，在创建 `str1` 时，JVM 检查**运行时常量池**，没有找到 ”str“ 这个字面量，于是创建该字面量并将其引用赋值给 `str1` 
3. 在创建 `str2` 时，JVM 再次检查**运行时常量池**，这次由于运行时常量池已有 ”str“ 字面量，根据字符串重用的机制，JVM 不重复创建字面量，而是将原有的 ”str“ 的引用直接赋值给 `str2` 

我们可以进行如下测试：

```
System.out.println(str1 == str2);
```

结果输出必然为 `true` ，因为 `str1` 与 `str2` 本就是相同的引用。 

这里的代码存在另一种需要区分的情况：

```
String str3 = new String("str");
String str4 = new String("str");
System.out.println(str3 == str4);
```

这里的输出为 `false` 。通过 new 来创建 String，将会在堆区创建不同的对象，对象引用自然也就不同。

### (三) String、StringBuilder、StringBuffer

String 的不可变性带来了便捷，但也带来了一些不便。不便在于，当我们需要多次地修改某个 String 对象时，直观地看，由于 String 的不可变，似乎每次修改都需要重新 new 一个 String 实例，这将带来极大地空间浪费和 GC 压力。JDK 设计者为了规避这种不便，在**底层的实现中采用可变的 StringBuilder 来避免频繁地对象创建**。

我们以 String 中实现字符串拼接的 `join()` 方法为例：

```java
public static String join(CharSequence delimiter, CharSequence... elements) {
    Objects.requireNonNull(delimiter);
    Objects.requireNonNull(elements);
    // Number of elements not likely worth Arrays.stream overhead.
    StringJoiner joiner = new StringJoiner(delimiter);
    for (CharSequence cs: elements) {
        joiner.add(cs);
    }
    return joiner.toString();
}
```

 `join()` 本身并不实现具体的字符串拼接，而是交由 `joiner` 来实现。

```java
public StringJoiner add(CharSequence newElement) {
    prepareBuilder().append(newElement);
    return this;
}
```

`prepareBuilder()` 顾名思义，创建并初始化了一个 StringBuilder 对象。在这里可以看出，String 中的 `join()`方法在实现中是基于 StringBuilder 的 `append()` 方法实现。也正是由于这个优化，才提高了在使用 ”+“ 对多个字符串进行拼接时的性能。

而至于 StringBuffer，除了和 StringBuilder 一样是可变的之外，还增加了多线程下的线程安全保证。所以在这三者之间进行选择时，一是我们要**根据字符串修改是否频繁来决定使用 String 还是 StringBuilder 和 StringBuffer**；二是要**根据是否处于多线程环境来决定使用 StringBuilder 或是 StringBuffer**。

------

*参考资料：*

1. *探秘 Java中 String、StringBuilder以及 StringBuffer： [http://www.importnew.com/18167.html](http://www.importnew.com/18167.html)*
2. *JDK 之 String 源码阅读笔记：[https://emacsist.github.io/2017/07/01/JDK-%E4%B9%8B-String-%E6%BA%90%E7%A0%81%E9%98%85%E8%AF%BB%E7%AC%94%E8%AE%B0/](https://emacsist.github.io/2017/07/01/JDK-%E4%B9%8B-String-%E6%BA%90%E7%A0%81%E9%98%85%E8%AF%BB%E7%AC%94%E8%AE%B0/)*


